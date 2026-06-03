# Week 09: Lab — Real Joins, Real Data

You'll do an as-of join (taxi × weather), a geographic join (taxi × neighborhoods), a Cartesian-detection check, and a point-in-time ML feature join.

## Setup

```bash
uv add duckdb polars geopandas shapely httpx
```

```python
import duckdb
import polars as pl
import time
from pathlib import Path

con = duckdb.connect(":memory:")
con.execute("INSTALL spatial; LOAD spatial;")
```

Make sure you have the NYC taxi parquet from week 01-03. We'll add weather and neighborhood data.

---

## Exercise 9.1 — Setup the reference tables

### Hourly weather (synthetic, plausible)

```python
import datetime as dt
import random
random.seed(42)

start = dt.datetime(2024, 1, 1)
hours = [(start + dt.timedelta(hours=i)) for i in range(31 * 24)]
weather = pl.DataFrame({
    "ts": hours,
    "temp_f": [30 + 10 * (random.random() - 0.5) + (3 if 10 <= h.hour <= 16 else 0) for h in hours],
    "precip_inches": [random.choices([0, 0, 0, 0.1, 0.5], weights=[40, 30, 15, 10, 5])[0] for _ in hours],
    "wind_mph": [random.uniform(0, 20) for _ in hours],
})
print(weather.head())
```

### NYC neighborhoods (synthetic 4-polygon GeoJSON; replace with real data later)

For real work you'd download `nyc_neighborhoods.geojson` from NYC Open Data. For the lab we'll use 4 bounding boxes that cover Manhattan, Brooklyn, Queens, Bronx.

```python
neighborhoods_geojson = {
    "type": "FeatureCollection",
    "features": [
        {"type": "Feature", "properties": {"borough": "Manhattan"},
         "geometry": {"type": "Polygon", "coordinates": [[
             [-74.02, 40.70], [-73.93, 40.70], [-73.93, 40.88], [-74.02, 40.88], [-74.02, 40.70]
         ]]}},
        {"type": "Feature", "properties": {"borough": "Brooklyn"},
         "geometry": {"type": "Polygon", "coordinates": [[
             [-74.05, 40.55], [-73.85, 40.55], [-73.85, 40.74], [-74.05, 40.74], [-74.05, 40.55]
         ]]}},
        {"type": "Feature", "properties": {"borough": "Queens"},
         "geometry": {"type": "Polygon", "coordinates": [[
             [-73.96, 40.60], [-73.70, 40.60], [-73.70, 40.80], [-73.96, 40.80], [-73.96, 40.60]
         ]]}},
        {"type": "Feature", "properties": {"borough": "Bronx"},
         "geometry": {"type": "Polygon", "coordinates": [[
             [-73.94, 40.79], [-73.76, 40.79], [-73.76, 40.92], [-73.94, 40.92], [-73.94, 40.79]
         ]]}},
    ]
}

Path("data").mkdir(exist_ok=True)
import json
with open("data/neighborhoods.geojson", "w") as f:
    json.dump(neighborhoods_geojson, f)
```

---

## Exercise 9.2 — As-of join: taxi × weather

Each taxi trip happens at some timestamp; you want the **most recent** hourly weather reading at that moment.

```python
TAXI_PATH = "data/yellow_tripdata_2024-01.parquet"

# Load trips, limit to a manageable sample
trips = (
    pl.scan_parquet(TAXI_PATH)
    .filter(pl.col("tpep_pickup_datetime").is_between(
        dt.datetime(2024, 1, 1), dt.datetime(2024, 1, 8)
    ))
    .select("tpep_pickup_datetime", "trip_distance", "fare_amount")
    .head(100_000)
    .collect()
    .rename({"tpep_pickup_datetime": "pickup_ts"})
    .sort("pickup_ts")
)
print(f"trips: {len(trips):,}")

# Polars as-of join
weather_sorted = weather.sort("ts")
joined = trips.join_asof(
    weather_sorted,
    left_on="pickup_ts",
    right_on="ts",
    strategy="backward",
)
print(joined.head())
print(f"joined: {len(joined):,}")
```

You should see each trip enriched with the weather from the most recent hourly reading prior to the pickup. **One pass; no cartesian explosion.**

### Compare to the naive approach

```python
import time

# Naive: filter the cross join (slow!)
t0 = time.perf_counter()
con.register("trips", trips)
con.register("weather", weather)
naive = con.sql("""
    SELECT t.*, w.temp_f
    FROM trips t
    JOIN weather w
        ON w.ts = (
            SELECT MAX(w2.ts)
            FROM weather w2
            WHERE w2.ts <= t.pickup_ts
        )
""").pl()
t_naive = time.perf_counter() - t0
print(f"naive correlated subquery: {t_naive:.2f}s")

# DuckDB ASOF JOIN
t0 = time.perf_counter()
duckdb_asof = con.sql("""
    SELECT t.*, w.temp_f
    FROM trips t
    ASOF LEFT JOIN weather w
        ON t.pickup_ts >= w.ts
""").pl()
t_asof = time.perf_counter() - t0
print(f"ASOF join: {t_asof:.2f}s")

# Polars (already timed above)
t0 = time.perf_counter()
polars_asof = trips.join_asof(weather_sorted, left_on="pickup_ts", right_on="ts", strategy="backward")
t_polars = time.perf_counter() - t0
print(f"Polars join_asof: {t_polars:.2f}s")
```

Polars and DuckDB ASOF should be **10-100× faster** than the correlated subquery.

---

## Exercise 9.3 — Geographic join: point-in-polygon

For each taxi pickup point, which borough is it in?

```python
# Load trips with lat/lng — NYC taxi data doesn't have it in recent years, so we'll fake it from location IDs
# (For real work you'd use the actual GPS columns from the 2014 vintage of TLC data.)

# Simulate: for each pickup location ID, generate a random plausible lat/lng
random.seed(42)
def fake_latlng(location_id):
    # Distribute trips across the bounding box
    lat = 40.65 + (location_id % 100) * 0.0025
    lng = -74.02 + (location_id // 100) * 0.0035 % 0.4
    return lat, lng

trip_sample = trips.head(10000).with_columns(
    pickup_lat=pl.col("trip_distance").map_elements(lambda x: 40.7 + random.random() * 0.2, return_dtype=pl.Float64),
    pickup_lng=pl.col("trip_distance").map_elements(lambda x: -73.95 + random.random() * 0.1, return_dtype=pl.Float64),
)

con.register("trips_geo", trip_sample.to_arrow())
con.execute("""
    CREATE OR REPLACE TABLE neighborhoods AS
    SELECT * FROM ST_Read('data/neighborhoods.geojson');
""")

result = con.sql("""
    SELECT
        t.pickup_ts,
        n.borough,
        COUNT(*) AS trips
    FROM trips_geo t
    JOIN neighborhoods n
        ON ST_Contains(n.geom, ST_Point(t.pickup_lng, t.pickup_lat))
    GROUP BY t.pickup_ts, n.borough
    LIMIT 10
""").pl()
print(result)

# Borough distribution
borough_counts = con.sql("""
    SELECT n.borough, COUNT(*) AS trip_count
    FROM trips_geo t
    LEFT JOIN neighborhoods n
        ON ST_Contains(n.geom, ST_Point(t.pickup_lng, t.pickup_lat))
    GROUP BY n.borough
""").pl()
print(borough_counts)
```

You should see trips distributed across the four boroughs (and some `null` for points outside any polygon — they fell outside the bounding boxes). **Real production uses actual polygons; the join syntax stays the same.**

---

## Exercise 9.4 — Cartesian detection

Build a sanity-check function: given a join plan, verify row counts make sense.

```python
def check_join(con, left_table, right_table, join_clause, expected_fan_out=1.0, tolerance=2.0):
    """Run join, return (left_count, right_count, join_count, ratio, status)."""
    left_n = con.sql(f"SELECT COUNT(*) FROM {left_table}").fetchone()[0]
    right_n = con.sql(f"SELECT COUNT(*) FROM {right_table}").fetchone()[0]
    join_n = con.sql(f"SELECT COUNT(*) FROM {left_table} l JOIN {right_table} r {join_clause}").fetchone()[0]
    ratio = join_n / left_n if left_n else 0
    status = "OK" if ratio <= expected_fan_out * tolerance else "CARTESIAN_RISK"
    return left_n, right_n, join_n, ratio, status

# Build a test scenario
con.execute("""
    CREATE OR REPLACE TABLE customers AS SELECT i AS id, 'name_' || i AS name FROM range(1000) t(i);
    CREATE OR REPLACE TABLE orders AS SELECT i AS id, (i % 1000) AS customer_id, 99.99 AS amount FROM range(10000) t(i);
""")

# Good: each customer has avg 10 orders → fan-out ratio 10
l, r, j, ratio, status = check_join(
    con, "customers", "orders", "ON l.id = r.customer_id",
    expected_fan_out=10.0,
)
print(f"Expected 10× fan-out: {l} × {r} → {j} rows (ratio {ratio:.1f}, {status})")

# Bad: missing join condition
l, r, j, ratio, status = check_join(
    con, "customers", "orders", "ON 1 = 1",
    expected_fan_out=10.0,
)
print(f"No join condition:    {l} × {r} → {j} rows (ratio {ratio:.0f}, {status})")
```

Wire this into your pipeline at every join step. If `status = CARTESIAN_RISK`, fail loud.

---

## Exercise 9.5 — Aggregate-then-join

Compare the slow pattern to the fast one.

```python
import time

# Slow: join, then aggregate
con.execute("""
    CREATE OR REPLACE TABLE big_orders AS
    SELECT i AS id, (i % 10000) AS customer_id, 99.99 + (i % 100) AS amount, '2024-01-01'::DATE AS dt
    FROM range(1000000) t(i);
""")

t0 = time.perf_counter()
slow = con.sql("""
    SELECT c.id, COUNT(o.id) AS n_orders, SUM(o.amount) AS total
    FROM customers c
    LEFT JOIN big_orders o ON c.id = o.customer_id
    GROUP BY c.id
""").pl()
t_slow = time.perf_counter() - t0
print(f"join-then-aggregate: {t_slow:.2f}s")

# Fast: aggregate, then join
t0 = time.perf_counter()
fast = con.sql("""
    WITH order_agg AS (
        SELECT customer_id, COUNT(*) AS n_orders, SUM(amount) AS total
        FROM big_orders GROUP BY customer_id
    )
    SELECT c.id, o.n_orders, o.total
    FROM customers c
    LEFT JOIN order_agg o ON c.id = o.customer_id
""").pl()
t_fast = time.perf_counter() - t0
print(f"aggregate-then-join: {t_fast:.2f}s")
print(f"speedup: {t_slow/t_fast:.1f}x")
```

On larger datasets the speedup is dramatic — sometimes the difference between "runs" and "OOMs."

---

## Exercise 9.6 — Point-in-time correctness for ML

Build a tiny example: predicting customer churn based on features as they were known **at the prediction date**.

```python
# Customer dimension as snapshots
customer_history = pl.DataFrame({
    "customer_id": [1,   1,   1,   2,   2,   3],
    "valid_from":  [
        dt.datetime(2023, 1, 1),
        dt.datetime(2023, 6, 1),
        dt.datetime(2024, 1, 1),
        dt.datetime(2023, 1, 1),
        dt.datetime(2023, 11, 1),
        dt.datetime(2023, 1, 1),
    ],
    "plan":        ["free", "pro",      "enterprise", "pro",   "pro",   "free"],
    "monthly_revenue_cents": [0, 9900, 29900, 9900, 9900, 0],
})

# Events
events = pl.DataFrame({
    "customer_id": [1, 1, 2, 2, 3],
    "event_ts": [
        dt.datetime(2023, 3, 15),
        dt.datetime(2023, 12, 15),
        dt.datetime(2023, 5, 15),
        dt.datetime(2024, 2, 15),
        dt.datetime(2024, 1, 15),
    ],
    "label": [0, 1, 0, 1, 1],   # 1 = churned within next 30 days
}).sort("event_ts")

# Point-in-time join — only use customer state KNOWN at event_ts
ml_features = events.join_asof(
    customer_history.sort("valid_from"),
    left_on="event_ts",
    right_on="valid_from",
    by="customer_id",
    strategy="backward",
)
print(ml_features)
```

You should see each event paired with the customer's plan **at that time**, not their current plan.

**This is the point-in-time join that prevents the #1 ML leakage bug.** Feature stores (Tecton, Feast, Hopsworks) automate this; for a small team, this Polars pattern is enough.

---

## Exercise 9.7 — Geographic join with spatial index

```python
con.execute("CREATE INDEX nbh_rtree ON neighborhoods USING RTREE (geom);")

t0 = time.perf_counter()
with_idx = con.sql("""
    SELECT n.borough, COUNT(*) AS n_trips
    FROM trips_geo t
    JOIN neighborhoods n ON ST_Contains(n.geom, ST_Point(t.pickup_lng, t.pickup_lat))
    GROUP BY n.borough
""").pl()
print(f"with rtree index: {time.perf_counter() - t0:.2f}s")
print(with_idx)
```

For our 4 polygons the speedup is minimal. For real NYC's ~300 neighborhoods × 1M trips, the rtree index turns minutes into seconds.

---

## Exercise 9.8 (stretch) — Window-based time join

When the "as of" join isn't enough — you need ALL events within a window:

```python
# "For each event, get all events from same customer within the past 7 days"
events_2 = events.with_columns(event_id=pl.int_range(len(events)))

window_join = con.sql("""
    SELECT
        a.event_id AS a_id,
        b.event_id AS b_id,
        a.event_ts,
        b.event_ts AS b_event_ts,
        b.label AS b_label,
    FROM events_2 a
    JOIN events_2 b ON a.customer_id = b.customer_id
        AND b.event_ts < a.event_ts
        AND b.event_ts >= a.event_ts - INTERVAL '7 days'
""", connection=con).pl()
print(window_join)
```

This is the pattern for "user's behavior over the past week" features. Common in recsys / fraud / personalization.

---

## Submission checklist

- [ ] As-of join taxi × weather working
- [ ] Polars `join_asof` and DuckDB `ASOF JOIN` both used
- [ ] Speed comparison shows ASOF ≥ 10× faster than correlated subquery
- [ ] Point-in-polygon join using DuckDB Spatial
- [ ] Cartesian-detection function flags missing-join-condition as CARTESIAN_RISK
- [ ] Aggregate-then-join shows speedup over join-then-aggregate on 1M rows
- [ ] Point-in-time join produces customer state as of event_ts (not current state)
- [ ] Spatial rtree index created
- [ ] (Stretch) Window-based time join for "behavior over last N days" features

---

## What you just did

You can now do every kind of join that matters in practice: temporal (as-of), geographic (point-in-polygon, nearest-neighbor), range (interval validity), and point-in-time (for ML training without leakage). You can detect Cartesian disasters with a one-line check and rewrite slow joins as fast aggregate-then-join.

Week 10 uses these joins to **bring external data in** — geocoding, Census demographics, weather, holidays. The output of week 09 (clean joinable tables) plus the output of week 10 (rich enrichment) is what makes downstream ML and dashboards 5-10× more valuable.

---

**Next**: [Week 10: External Enrichment →](../week-10-external-enrichment/readme.md)
