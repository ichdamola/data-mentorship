# Week 10: Theory - External Enrichment

Your data is more valuable when joined with the world's data. **Geocoding** an address turns "123 Main St" into a lat/lng that joins to neighborhood demographics. **Census attaches** demographic features to any US zip / tract. **Weather, FX rates, holiday calendars** provide context that turns a flat fact table into a richly featured one.

This week covers the high-leverage open and cheap-commercial enrichment sources, the **ethics and rate-limit reality**, and the **caching pattern** that keeps your re-runs free.

---

## Part 1: What "enrichment" actually means

A dataset has rows of *facts* (events, customers, orders). Enrichment adds *context*:

| Original column | Enriched columns |
|---|---|
| `address` | `lat`, `lng`, `tract_geoid`, `borough` |
| `tract_geoid` | `median_household_income`, `pct_college`, `population_density` |
| `pickup_ts` | `temperature_f`, `precip_inches`, `is_holiday` |
| `currency`, `ts` | `usd_equivalent`, `fx_rate_used` |
| `company_name` | `industry`, `headcount_band`, `founded_year` |

The **enrichment columns drive downstream model accuracy and dashboard depth**. A dashboard showing "trips by location" is fine; "trips by neighborhood income" is a story.

---

## Part 2: Geocoding

The most common enrichment. The options:

| Tool | Cost | Accuracy | Rate limit | Notes |
|---|---|---|---|---|
| **Nominatim** (OpenStreetMap) | Free | Variable | **1 req/sec**; bulk requires self-host | Open data; the polite default |
| **Pelias / Photon** | Free, self-host | Comparable | None (your own infra) | Run as Docker; harder ops |
| **US Census Geocoder** | Free | High in US | 10k/day per source | US addresses only |
| **Google Maps Geocoding API** | $5 / 1k requests | Highest | High; quotas configurable | Pay; ToS forbids cache > 30 days |
| **Mapbox Geocoding** | $0.75 / 1k req | High | High | Pay; more lenient ToS |
| **HERE Geocoding** | Tiered free; paid above | High | Tiered | Enterprise-friendly |

### Nominatim usage policy (read this!)

The free public Nominatim instance has hard rules:

- **Maximum 1 request per second**
- **Set a User-Agent that identifies you** (with contact)
- **Cache aggressively** - re-running the same query is bad faith
- **For bulk jobs (>1k addresses)**: self-host Nominatim via Docker

```python
import httpx
import time

response = httpx.get(
    "https://nominatim.openstreetmap.org/search",
    params={"q": "Times Square, New York", "format": "json", "limit": 1},
    headers={"User-Agent": "data-mentorship-lab/1.0 (contact@example.com)"},
)
time.sleep(1)   # mandatory!
```

If you ignore the policy, you get rate-limited or IP-blocked, and you've contributed to the funding burden that maintains the free service.

For a small one-shot job (< 1000 addresses), Nominatim with `time.sleep(1)` is fine. For more, either self-host or use a commercial API.

---

## Part 3: Census demographics

The US Census provides one of the world's richest free demographic datasets - household income, education, race, age distribution, etc. - at varying granularities:

| Granularity | Population | Use |
|---|---|---|
| State | 4-40M | Country-level analysis |
| County | 10k-10M | Regional analysis |
| Tract (~4k pop) | 1k-8k | Neighborhood analysis (most ML uses) |
| Block group | 600-3000 | Hyperlocal |
| Block | 0-600 | Extreme local; often suppressed for privacy |

Available via the **Census Data API** (free, with key registration at api.census.gov/data/key_signup.html).

Common Census products:

| Product | What | Granularity |
|---|---|---|
| **ACS 5-year** | American Community Survey, 5-year averages | Down to tract |
| **ACS 1-year** | More recent but only for areas ≥ 65k pop | County and above |
| **Decennial** | The 10-year full census | Block-level historical |

For attaching demographics to events:

```
1. Each event has lat/lng → geocode the address if needed
2. Each lat/lng → look up tract GEOID via Census Geocoder
3. tract GEOID → demographic columns from ACS
```

Step 2-3 require one round-trip each but produce 50+ enrichment columns per row.

---

## Part 4: Other rich public sources

The catalog of "free useful public data" is huge. The hits:

| Source | Provides |
|---|---|
| **NOAA / NWS** | Hourly weather, historical climate |
| **FRED (St Louis Fed)** | Economic indicators, FX rates, interest rates |
| **OpenSky** | Real-time flight traffic |
| **OpenStreetMap** | Roads, buildings, points of interest |
| **EU Open Data Portal** | Comparable to data.gov for EU |
| **World Bank Open Data** | International economic / demographic |
| **GDELT** | Global news events at high granularity |
| **Wikipedia API** | Encyclopedic content; Wikidata for structured |
| **Yahoo Finance / yfinance** | Stock prices (free; ToS-bound) |
| **Common Crawl** | Petabytes of crawled web pages, free |

For commercial-grade alternatives that aggregate many of these into clean APIs, **Clearbit**, **People Data Labs**, **ZoomInfo** (B2B); **Apollo.io**, **Crunchbase API** (companies). All paid; all faster to integrate than rolling your own.

---

## Part 5: Scraping - the ethical and legal landscape

If a vendor doesn't have an API, you're scraping. The senior engineer's checklist:

1. **Check `robots.txt`** at `<site>/robots.txt`. It's a request, not a contract - but ignoring it is bad faith and weakens any "we acted in good faith" defense.
2. **Rate limit yourself**. 1 request/second is universally polite; some sites publish lower limits in robots.txt.
3. **Identify yourself**. Set a `User-Agent: contact@yourorg.com (...)`. Site operators can ask you to stop; if they can't reach you, they'll just block your IPs.
4. **Cache aggressively**. The first scrape is necessary; the 50th re-run is hostile.
5. **Read the ToS**. Many sites prohibit scraping in their ToS. The legal force of ToS-only prohibition varies by jurisdiction (look up *hiQ v. LinkedIn* and *Van Buren v. United States* if curious), but you don't want to be the case that decides it.
6. **Personal data**: anything involving PII (names + locations, faces, IDs) crosses into privacy regulation. GDPR, CCPA, and Nigeria's NDPR all have something to say.

The bright lines:

- **OK**: scraping public data that's already aggregated for research (Common Crawl exists; use it)
- **OK (usually)**: scraping a single page occasionally with attribution
- **OK with care**: scraping a company's product catalog for price comparison (still ToS-relative)
- **Don't**: scraping personal profiles at scale, scraping a competitor's user base, scraping anything behind a login
- **Don't**: ignoring CAPTCHAs, rotating IPs to evade rate limits

The appsec-mentorship version of this is at [appsec-mentorship week 14](https://github.com/ichdamola/appsec-mentorship/tree/main/week-14-security-misconfig). The short version: **authorized = good; unauthorized = a court's discretion, with you on the wrong side**.

---

## Part 6: Holidays, calendars, business days

A common analytic gotcha: "revenue is down on Tuesday" - but it's Tuesday after a 3-day weekend. The fix: enrich with holiday context.

The **`holidays`** library covers ~100 countries; one line per attribute:

```python
import holidays
us = holidays.US()
"2024-01-01" in us       # True (New Year)
"2024-11-28" in us       # True (Thanksgiving)
us.get("2024-12-25")     # 'Christmas Day'
```

For NYC-specific: **NYC government holidays** differ from federal. For UK: `holidays.UK()` includes bank holidays. For Nigeria: `holidays.Nigeria()`.

Enrich with: `is_holiday`, `holiday_name`, `days_since_last_holiday`, `is_business_day`.

---

## Part 7: The enrichment cache pattern

External APIs are the slow, expensive, rate-limited part of your pipeline. Re-running the pipeline shouldn't hit the API again for inputs you've already processed.

The pattern:

```
1. Compute a content hash of the input (e.g., the address string)
2. Check the cache: does this hash exist?
   - Yes → return the cached enrichment
   - No → call the API, store result with hash as key, return
```

A tiny SQLite-backed cache:

```python
import sqlite3
import hashlib
import json

class APICache:
    def __init__(self, path="api_cache.db"):
        self.con = sqlite3.connect(path)
        self.con.execute("""
            CREATE TABLE IF NOT EXISTS cache (
                hash TEXT PRIMARY KEY,
                fn_name TEXT NOT NULL,
                fn_version TEXT NOT NULL,
                input TEXT,
                result TEXT,
                cached_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)

    def get_or_compute(self, input_str: str, compute_fn, *, fn_name: str, fn_version: str = "v1"):
        # Key by (input, fn_name, fn_version) - otherwise calls to the same
        # cache from two different enrichment functions (e.g. geocoding via
        # Nominatim vs Google Maps) collide on identical inputs and return
        # the wrong cached result. Bump fn_version when the upstream
        # response shape changes.
        key = f"{fn_name}:{fn_version}:{input_str}"
        h = hashlib.sha256(key.encode()).hexdigest()
        row = self.con.execute("SELECT result FROM cache WHERE hash=?", (h,)).fetchone()
        if row:
            return json.loads(row[0])
        result = compute_fn(input_str)
        self.con.execute(
            "INSERT INTO cache (hash, fn_name, fn_version, input, result) VALUES (?, ?, ?, ?, ?)",
            (h, fn_name, fn_version, input_str, json.dumps(result)),
        )
        self.con.commit()
        return result
```

Now re-running is free for any already-seen input. Combine with rate limiting and the cache becomes the difference between "the pipeline takes 12 hours" and "the pipeline takes 12 minutes after the first run."

### Cache expiry

When does cached data go stale? Depends on the source:

| Data type | Cache lifetime |
|---|---|
| Geocoded address | Forever (addresses don't move) |
| Census tract demographics | 5 years (ACS update cadence) |
| Stock price | Seconds to minutes |
| Weather (historical) | Forever |
| Weather (forecast) | Hours |
| Holiday calendar | Until the holiday system changes (decades) |

For long-lived caches (geocoding), no expiry needed. For short-lived (live prices), include a timestamp and check freshness.

---

## Part 8: Privacy: when enrichment becomes regulated

Enrichment often combines low-privacy facts into high-privacy profiles:

- Order amount → fine
- Order amount + address → traceable to a household
- Order amount + address + demographic income → individually identifiable

Crossing thresholds like this means the enriched table may now be subject to GDPR / CCPA / etc.

**Senior moves:**

- Document which columns were enrichment-derived and the source
- Keep a column-level lineage in metadata: "median_household_income was derived from Census ACS 5yr on tract_geoid which was derived from address via Nominatim on 2024-01-15"
- Pseudonymize identifiers before enrichment if possible
- Have a data deletion playbook that handles enriched rows specifically

The principle: enrichment can take you from un-regulated to regulated data without anyone noticing. Be deliberate.

---

## Part 9: When to enrich vs when to embed

For the next decade or so, two enrichment styles coexist:

| Approach | Strengths | When |
|---|---|---|
| **Structured enrichment** (this week) | Auditable; explainable; integer columns; works with linear models | Tabular ML; reporting; anything where stakeholders need to understand |
| **Embedding-based enrichment** (week 12) | Captures semantics; handles long-tail; no schema work | RAG; semantic search; recommendation; deep learning |

For most analytic and reporting work, structured enrichment is the right default. For text-heavy or rec-style problems, embeddings.

The right answer in 2026 is often **both**: structured columns + embedding columns side by side, with downstream model deciding which signals help.

---

## Part 10: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| Re-running pipeline without caching | API costs explode; you hit rate limits |
| Using free Nominatim for 1M-address geocoding | IP-blocked; OSM Foundation displeased |
| Caching commercial API results past their ToS-allowed retention | Legal risk |
| Joining enrichment-derived columns without tracking provenance | Future-you doesn't know if the column is fresh |
| Enriching PII data into a table without noting it became regulated | Compliance surprise |
| Trusting one source as ground truth for addresses | Even Google has typos |

---

## What's next

In [lab.md](lab.md) you'll:

- Geocode a small batch of NYC addresses via Nominatim with proper rate limiting + cache
- Look up Census tract GEOIDs from coordinates
- Attach ACS demographic columns to a customer table
- Build a tiny historical weather puller (from NOAA's free archives)
- Run the holidays library; enrich a transactions table with `is_holiday`
- Build the SQLite enrichment cache pattern
- (Stretch) Set up self-hosted Nominatim via Docker for high-volume

By end of week 10 you can take any clean table with addresses or timestamps and produce a richly enriched version downstream consumers will love.
