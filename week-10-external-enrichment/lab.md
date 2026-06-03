# Week 10: Lab — Enrich a Customer Table

You'll geocode addresses through Nominatim with proper rate limiting + cache, look up Census tracts, attach demographics, enrich with weather + holidays, and end with a fully enriched customer table.

## Setup

```bash
uv add httpx tenacity polars holidays
```

```python
import httpx
import hashlib
import json
import sqlite3
import time
from pathlib import Path
import polars as pl
from tenacity import retry, wait_exponential, stop_after_attempt
import holidays
import datetime as dt

USER_AGENT = "data-mentorship-lab/1.0 (your.email@example.com)"   # PUT YOUR REAL CONTACT
```

If you want to use the **US Census API** in Exercise 10.4, register for a free key at api.census.gov/data/key_signup.html (instant). Otherwise that exercise is a worked example.

---

## Exercise 10.1 — Build the SQLite-backed enrichment cache

```python
class APICache:
    def __init__(self, path="data/api_cache.db"):
        Path(path).parent.mkdir(parents=True, exist_ok=True)
        self.con = sqlite3.connect(path)
        self.con.execute("""
            CREATE TABLE IF NOT EXISTS cache (
                hash TEXT PRIMARY KEY,
                provider TEXT,
                input TEXT,
                result TEXT,
                cached_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)

    def _key(self, provider: str, input_str: str) -> str:
        return hashlib.sha256(f"{provider}::{input_str}".encode()).hexdigest()

    def get_or_compute(self, provider: str, input_str: str, compute_fn):
        h = self._key(provider, input_str)
        row = self.con.execute("SELECT result FROM cache WHERE hash=?", (h,)).fetchone()
        if row:
            return json.loads(row[0])
        result = compute_fn(input_str)
        self.con.execute(
            "INSERT INTO cache (hash, provider, input, result) VALUES (?, ?, ?, ?)",
            (h, provider, input_str, json.dumps(result, default=str)),
        )
        self.con.commit()
        return result

    def stats(self):
        rows = self.con.execute("SELECT provider, COUNT(*) FROM cache GROUP BY provider").fetchall()
        return dict(rows)


cache = APICache()
print(cache.stats())
```

You should see an empty dict (no entries yet).

---

## Exercise 10.2 — Geocode via Nominatim (with rate limiting)

```python
@retry(wait=wait_exponential(multiplier=1, min=1, max=30), stop=stop_after_attempt(5), reraise=True)
def nominatim_geocode_raw(query: str) -> dict:
    """Single Nominatim call. Rate limit enforced by caller."""
    resp = httpx.get(
        "https://nominatim.openstreetmap.org/search",
        params={"q": query, "format": "json", "limit": 1, "addressdetails": 1},
        headers={"User-Agent": USER_AGENT},
        timeout=15,
    )
    resp.raise_for_status()
    results = resp.json()
    if not results:
        return {"found": False}
    r = results[0]
    return {
        "found": True,
        "lat": float(r["lat"]),
        "lng": float(r["lon"]),
        "display_name": r["display_name"],
        "city": (r.get("address") or {}).get("city") or (r.get("address") or {}).get("town"),
        "state": (r.get("address") or {}).get("state"),
        "country": (r.get("address") or {}).get("country"),
        "postal_code": (r.get("address") or {}).get("postcode"),
    }


def geocode(address: str) -> dict:
    """Cached geocode. First call: API hit. Repeat calls: free."""
    return cache.get_or_compute("nominatim", address, _nominatim_call)


def _nominatim_call(addr: str) -> dict:
    result = nominatim_geocode_raw(addr)
    time.sleep(1.1)  # mandatory rate limit
    return result


# Test on 5 famous NYC addresses
addresses = [
    "Empire State Building, New York",
    "Statue of Liberty, New York",
    "Times Square, New York",
    "Yankee Stadium, New York",
    "Coney Island, Brooklyn, New York",
]

results = []
for addr in addresses:
    t0 = time.perf_counter()
    geo = geocode(addr)
    print(f"{addr:<40s} → ({geo.get('lat'):.4f}, {geo.get('lng'):.4f})  ({time.perf_counter()-t0:.2f}s)")
    results.append({"input": addr, **geo})

print(f"\ncache contents: {cache.stats()}")
```

The first 5 calls each take ~1 second (rate limit). Now re-run the loop:

```python
print("\n=== second run (should be cached, fast) ===")
for addr in addresses:
    t0 = time.perf_counter()
    geo = geocode(addr)
    print(f"{addr:<40s} → ({time.perf_counter()-t0*1000:.0f}ms)")
```

Each call should now be sub-millisecond. **That's the cache earning its keep.**

---

## Exercise 10.3 — Reverse geocoding (lat/lng → address)

The reverse of geocoding — given coordinates, get the address. Same Nominatim endpoint, different path.

```python
@retry(wait=wait_exponential(multiplier=1, min=1, max=30), stop=stop_after_attempt(5), reraise=True)
def nominatim_reverse(lat: float, lng: float) -> dict:
    resp = httpx.get(
        "https://nominatim.openstreetmap.org/reverse",
        params={"lat": lat, "lon": lng, "format": "json", "addressdetails": 1},
        headers={"User-Agent": USER_AGENT},
        timeout=15,
    )
    resp.raise_for_status()
    r = resp.json()
    return {
        "display_name": r.get("display_name"),
        "city": (r.get("address") or {}).get("city") or (r.get("address") or {}).get("town"),
        "state": (r.get("address") or {}).get("state"),
        "neighborhood": (r.get("address") or {}).get("neighbourhood"),
        "country": (r.get("address") or {}).get("country"),
    }


def reverse_geocode(lat: float, lng: float) -> dict:
    return cache.get_or_compute("nominatim_reverse", f"{lat},{lng}", lambda key: _reverse_call(key))


def _reverse_call(latlng: str) -> dict:
    lat, lng = [float(x) for x in latlng.split(",")]
    result = nominatim_reverse(lat, lng)
    time.sleep(1.1)
    return result


# Test
print(reverse_geocode(40.7484, -73.9857))   # Empire State Building area
```

---

## Exercise 10.4 — Census tract from coordinates (worked)

The US Census Geocoder converts lat/lng to a tract GEOID. Two endpoints:

```python
@retry(wait=wait_exponential(multiplier=1, min=1, max=10), stop=stop_after_attempt(3))
def census_tract_from_latlng(lat: float, lng: float) -> dict:
    """Get tract GEOID for a point."""
    resp = httpx.get(
        "https://geocoding.geo.census.gov/geocoder/geographies/coordinates",
        params={
            "x": lng, "y": lat,
            "benchmark": "Public_AR_Current",
            "vintage": "Current_Current",
            "format": "json",
        },
        timeout=30,
    )
    resp.raise_for_status()
    data = resp.json()
    tracts = data["result"]["geographies"].get("Census Tracts", [])
    if not tracts:
        return {"geoid": None}
    t = tracts[0]
    return {
        "geoid": t["GEOID"],
        "state_fips": t["STATE"],
        "county_fips": t["COUNTY"],
        "tract": t["TRACT"],
    }


def census_tract(lat: float, lng: float) -> dict:
    return cache.get_or_compute("census_tract", f"{lat},{lng}", lambda k: _census_tract_call(k))


def _census_tract_call(latlng: str) -> dict:
    lat, lng = [float(x) for x in latlng.split(",")]
    result = census_tract_from_latlng(lat, lng)
    time.sleep(0.3)
    return result


# Test: Empire State Building
tract_info = census_tract(40.7484, -73.9857)
print(tract_info)
# {'geoid': '36061007800', 'state_fips': '36', 'county_fips': '061', 'tract': '007800'}
```

Now you have the tract GEOID. Next: demographics from ACS.

---

## Exercise 10.5 — Census ACS demographics (worked)

For one tract, fetch median household income, education, age distribution.

```python
CENSUS_API_KEY = ""  # ← put your key here

@retry(wait=wait_exponential(multiplier=1, min=1, max=10), stop=stop_after_attempt(3))
def census_acs_for_tract(geoid: str) -> dict:
    """Pull a handful of useful ACS 5-year columns for a tract."""
    if not CENSUS_API_KEY:
        # Worked example without API key
        return {
            "geoid": geoid,
            "median_household_income": None,
            "pct_college_or_higher": None,
            "median_age": None,
            "_note": "Set CENSUS_API_KEY to fetch real data",
        }

    state_fips = geoid[:2]
    county_fips = geoid[2:5]
    tract = geoid[5:]
    resp = httpx.get(
        f"https://api.census.gov/data/2022/acs/acs5",
        params={
            "get": "B19013_001E,B15003_022E,B15003_001E,B01002_001E",   # median income, bachelor's, total, median age
            "for": f"tract:{tract}",
            "in": f"state:{state_fips}+county:{county_fips}",
            "key": CENSUS_API_KEY,
        },
        timeout=30,
    )
    resp.raise_for_status()
    rows = resp.json()
    # rows[0] is column headers; rows[1] is the data
    cols, data = rows[0], rows[1]
    record = dict(zip(cols, data))
    bachelor = int(record.get("B15003_022E", 0))
    total_25plus = int(record.get("B15003_001E", 1))
    return {
        "geoid": geoid,
        "median_household_income": int(record.get("B19013_001E", -1)),
        "pct_college_or_higher": round(bachelor / total_25plus * 100, 1) if total_25plus else None,
        "median_age": float(record.get("B01002_001E", 0)),
    }


# Test
print(census_acs_for_tract("36061007800"))
```

For real data you'd need the API key. The pattern: tract GEOID → ACS columns → enrich any customer/order/event sitting in that tract.

---

## Exercise 10.6 — Holidays enrichment

```python
us_holidays = holidays.US(years=range(2023, 2026))

# Add NY state holidays
ny_holidays = holidays.US(state="NY", years=range(2023, 2026))

def enrich_with_holiday(date_str: str) -> dict:
    d = dt.date.fromisoformat(date_str)
    return {
        "is_federal_holiday": d in us_holidays,
        "federal_holiday_name": us_holidays.get(d),
        "is_state_holiday": d in ny_holidays,
        "day_of_week": d.strftime("%A"),
        "is_weekend": d.weekday() >= 5,
        "is_business_day": d.weekday() < 5 and d not in ny_holidays,
    }


for date_str in ["2024-01-01", "2024-07-04", "2024-11-28", "2024-03-15", "2024-12-25"]:
    info = enrich_with_holiday(date_str)
    print(f"{date_str}: {info}")
```

You should see Independence Day, Thanksgiving, Christmas Day flagged as holidays, with weekend / business-day computed correctly.

---

## Exercise 10.7 — Build the enriched customer table

Combine everything: synthesize a customer dataset, geocode it, look up tracts, attach demographics (mock if no Census key), add holiday context to their signup date.

```python
import random
random.seed(42)

# Synthetic customers with NYC-ish addresses
customers = pl.DataFrame({
    "customer_id": list(range(1, 11)),
    "address": [
        "Empire State Building, New York",
        "Statue of Liberty, New York",
        "Times Square, New York",
        "Yankee Stadium, New York",
        "Coney Island, Brooklyn, New York",
        "Central Park, New York",
        "Brooklyn Bridge, New York",
        "Wall Street, New York",
        "JFK Airport, New York",
        "Flushing Meadows, Queens, New York",
    ],
    "signup_date": [
        "2024-01-01", "2024-03-15", "2024-07-04", "2024-11-28",
        "2024-05-15", "2024-08-20", "2024-12-25", "2024-02-14",
        "2024-09-01", "2024-10-31",
    ],
})

# Enrich each customer
enriched_rows = []
for row in customers.iter_rows(named=True):
    # Geocode
    geo = geocode(row["address"])
    if not geo.get("found", True):
        enriched_rows.append({**row, "lat": None, "lng": None})
        continue

    # Tract
    tract_info = census_tract(geo["lat"], geo["lng"])

    # Demographics (mock if no Census key)
    demo = census_acs_for_tract(tract_info.get("geoid") or "")

    # Holiday on signup_date
    hol = enrich_with_holiday(row["signup_date"])

    enriched_rows.append({
        **row,
        "lat": geo.get("lat"),
        "lng": geo.get("lng"),
        "city": geo.get("city"),
        "tract_geoid": tract_info.get("geoid"),
        "median_household_income": demo.get("median_household_income"),
        "pct_college_or_higher": demo.get("pct_college_or_higher"),
        "signed_up_on_holiday": hol["is_federal_holiday"],
        "holiday_name": hol["federal_holiday_name"],
        "day_of_week": hol["day_of_week"],
    })

enriched = pl.DataFrame(enriched_rows)
print(enriched)

# Save
Path("data/silver").mkdir(parents=True, exist_ok=True)
enriched.write_parquet("data/silver/customers_enriched.parquet")
print(f"\nwrote {len(enriched)} enriched rows")
```

You should see each customer with 10+ added columns. **This is the format every downstream dashboard and model wants.**

---

## Exercise 10.8 — Column-level provenance metadata

```python
provenance = pl.DataFrame({
    "column": [
        "lat", "lng", "city",
        "tract_geoid",
        "median_household_income", "pct_college_or_higher",
        "signed_up_on_holiday", "holiday_name",
    ],
    "source": [
        "Nominatim", "Nominatim", "Nominatim",
        "US Census Geocoder",
        "ACS 5-year", "ACS 5-year",
        "python holidays library", "python holidays library",
    ],
    "version_or_vintage": [
        "Public 2024", "Public 2024", "Public 2024",
        "Public_AR_Current",
        "2022", "2022",
        "library v0.45", "library v0.45",
    ],
    "ttl_hint": [
        "forever (addresses don't move)", "forever", "forever",
        "forever",
        "5 years", "5 years",
        "until law changes", "until law changes",
    ],
})
print(provenance)

provenance.write_parquet("data/silver/customers_enriched_provenance.parquet")
```

Ship this alongside the enriched data. When a stakeholder asks "where does median_household_income come from?", the answer is in the metadata, not your head.

---

## Exercise 10.9 (stretch) — Self-hosted Nominatim

For real production at 1M+ addresses, you'd run Nominatim yourself.

```bash
# In a separate terminal
docker run -d \
    --name nominatim \
    -p 8080:8080 \
    -e PBF_URL=https://download.geofabrik.de/north-america/us/new-york-latest.osm.pbf \
    -e REPLICATION_URL=https://download.geofabrik.de/north-america/us/new-york-updates/ \
    mediagis/nominatim:4.4
```

After ~20 minutes (initial OSM load), you have your own Nominatim at `http://localhost:8080`. No rate limit (you set it). For NYC-only addresses this consumes ~10 GB disk and ~2 GB RAM.

The pattern: prototype with hosted Nominatim, scale up by self-hosting. Same code, different `httpx.get` URL.

---

## Submission checklist

- [ ] SQLite cache populated; second-run is sub-millisecond per address
- [ ] 5 addresses geocoded through Nominatim with `time.sleep(1.1)`
- [ ] User-Agent set with real contact
- [ ] Reverse-geocoding works
- [ ] Census tract GEOID looked up for at least 1 point
- [ ] Holidays enrichment categorizes signup dates correctly (4th of July, Thanksgiving, etc.)
- [ ] Enriched customers table written with 10+ enrichment columns
- [ ] Column-level provenance metadata table written
- [ ] (Stretch) Self-hosted Nominatim Docker container running

---

## What you just did

You can enrich any address with lat/lng, tract, demographics. Any timestamp with holiday + business-day context. The cache pattern keeps re-runs free; the provenance table keeps stakeholders happy. The Nominatim usage policy stays respected.

Week 11 will use this enriched data for **feature engineering** — turning the columns into ML-ready features without leakage. Week 12 introduces embeddings for the cases where structured enrichment falls short.

---

**Next**: [Week 11: Feature Engineering →](../week-11-feature-engineering/readme.md)
