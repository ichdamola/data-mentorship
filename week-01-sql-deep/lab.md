# Week 01: Lab — SQL Deep

Ten questions of increasing difficulty against the NYC Yellow Taxi data. **Solve each in SQL only** — no pandas allowed until you've got the SQL working. Then you can verify in pandas if you want.

## Setup

```bash
# In your data-mentorship-work dir:
uv add duckdb pyarrow
mkdir -p data
cd data
curl -O https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet
cd ..
```

DuckDB CLI (the most fun way to do SQL):

```bash
uv run python -m duckdb        # or install the duckdb CLI directly
```

In the prompt:

```sql
-- Inspect the schema
DESCRIBE SELECT * FROM read_parquet('data/yellow_tripdata_2024-01.parquet');

-- Quick row count
SELECT COUNT(*) FROM read_parquet('data/yellow_tripdata_2024-01.parquet');
-- Should be ~3M rows
```

Or use DuckDB from Python:

```python
import duckdb
con = duckdb.connect()
con.sql("SELECT COUNT(*) FROM read_parquet('data/yellow_tripdata_2024-01.parquet')").show()
```

For convenience, register the file as a view:

```sql
CREATE VIEW trips AS SELECT * FROM read_parquet('data/yellow_tripdata_2024-01.parquet');
```

Now you can use `FROM trips` everywhere.

---

## Exercise 1.1 — Basic exploration

Compute: total trips, total revenue, average trip distance, median trip distance, average tip percentage. **Use one query.**

<details>
<summary>Solution</summary>

```sql
SELECT
    COUNT(*)                                                AS total_trips,
    ROUND(SUM(total_amount), 2)                              AS total_revenue,
    ROUND(AVG(trip_distance), 2)                             AS avg_distance,
    ROUND(MEDIAN(trip_distance), 2)                          AS median_distance,
    ROUND(AVG(tip_amount / NULLIF(fare_amount, 0)) * 100, 2) AS avg_tip_pct
FROM trips
WHERE fare_amount > 0;
```

Note `NULLIF(fare_amount, 0)` — divides safely; `0 / 0` would be NaN.

`MEDIAN` is a DuckDB extension. In Postgres it's `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ...)`.
</details>

---

## Exercise 1.2 — Top 10 most expensive trips per day

Return, for each day in January 2024, the top 10 trips by `total_amount`.

<details>
<summary>Solution — window function + QUALIFY</summary>

```sql
SELECT
    DATE(tpep_pickup_datetime) AS trip_date,
    tpep_pickup_datetime,
    trip_distance,
    total_amount,
    ROW_NUMBER() OVER (PARTITION BY DATE(tpep_pickup_datetime)
                       ORDER BY total_amount DESC) AS rank_in_day
FROM trips
WHERE tpep_pickup_datetime >= '2024-01-01'
  AND tpep_pickup_datetime <  '2024-02-01'
QUALIFY rank_in_day <= 10
ORDER BY trip_date, rank_in_day;
```

You'll see the top 10 trips per day — likely $300+ amounts each. Some are airport runs; some are credit-card refund corrections (negative trip distance).
</details>

---

## Exercise 1.3 — Trip duration percentiles by hour-of-day

For each hour of the day (0-23), compute: median, 90th, 95th, and 99th percentile of trip duration in minutes.

<details>
<summary>Solution</summary>

```sql
WITH durations AS (
    SELECT
        EXTRACT(hour FROM tpep_pickup_datetime) AS hour_of_day,
        DATE_DIFF('minute', tpep_pickup_datetime, tpep_dropoff_datetime) AS duration_min
    FROM trips
    WHERE tpep_dropoff_datetime > tpep_pickup_datetime
      AND DATE_DIFF('minute', tpep_pickup_datetime, tpep_dropoff_datetime) BETWEEN 1 AND 240
)
SELECT
    hour_of_day,
    ROUND(QUANTILE_CONT(duration_min, 0.5),  1) AS p50,
    ROUND(QUANTILE_CONT(duration_min, 0.9),  1) AS p90,
    ROUND(QUANTILE_CONT(duration_min, 0.95), 1) AS p95,
    ROUND(QUANTILE_CONT(duration_min, 0.99), 1) AS p99,
    COUNT(*) AS trips
FROM durations
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

Observe: late-night trips have higher variance (long tail of airport runs and DUIs); rush-hour trips have higher median but lower variance.
</details>

---

## Exercise 1.4 — Compare each trip to the day's average

For each trip, return its `fare_amount` and the average `fare_amount` for that day, and the ratio. **Don't collapse rows** — keep one row per trip.

<details>
<summary>Solution — aggregate window function</summary>

```sql
SELECT
    tpep_pickup_datetime,
    fare_amount,
    AVG(fare_amount) OVER (PARTITION BY DATE(tpep_pickup_datetime)) AS day_avg,
    ROUND(fare_amount / AVG(fare_amount) OVER (PARTITION BY DATE(tpep_pickup_datetime)), 2) AS ratio_to_day_avg
FROM trips
WHERE fare_amount > 0
LIMIT 20;
```

This is the canonical "compare each row to its group average without losing row granularity" pattern.
</details>

---

## Exercise 1.5 — 7-day rolling revenue

Compute daily revenue and a 7-day rolling average of daily revenue.

<details>
<summary>Solution</summary>

```sql
WITH daily AS (
    SELECT
        DATE(tpep_pickup_datetime) AS day,
        SUM(total_amount)          AS revenue
    FROM trips
    GROUP BY day
)
SELECT
    day,
    revenue,
    ROUND(
        AVG(revenue) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW),
        2
    ) AS rolling_7d_avg
FROM daily
ORDER BY day;
```

The first 6 days have rolling averages of fewer than 7 days — they're not invalid, just averaging fewer values.
</details>

---

## Exercise 1.6 — LAG: time between trips per driver?

Skip — the dataset doesn't have a driver ID. Use **pickup location** as a stand-in. **For each pickup location, what's the median time between consecutive trips?**

<details>
<summary>Solution</summary>

```sql
WITH ordered AS (
    SELECT
        PULocationID,
        tpep_pickup_datetime,
        LAG(tpep_pickup_datetime) OVER (
            PARTITION BY PULocationID ORDER BY tpep_pickup_datetime
        ) AS prev_pickup
    FROM trips
),
gaps AS (
    SELECT
        PULocationID,
        DATE_DIFF('minute', prev_pickup, tpep_pickup_datetime) AS gap_min
    FROM ordered
    WHERE prev_pickup IS NOT NULL
)
SELECT
    PULocationID,
    COUNT(*)                                AS n_pairs,
    ROUND(QUANTILE_CONT(gap_min, 0.5),  2)  AS median_gap_min,
    ROUND(QUANTILE_CONT(gap_min, 0.9),  2)  AS p90_gap_min
FROM gaps
WHERE gap_min BETWEEN 0 AND 240
GROUP BY PULocationID
HAVING COUNT(*) > 100
ORDER BY median_gap_min DESC
LIMIT 20;
```

You'll see suburban / airport locations with median gaps of hours; midtown locations with median gaps measured in seconds.
</details>

---

## Exercise 1.7 — Bucket by tip percentage

Classify each trip into a tip bucket: `none` (0%), `low` (0-10%), `standard` (10-20%), `high` (20%+). Compute: count, average fare, average distance, per bucket.

<details>
<summary>Solution</summary>

```sql
WITH classified AS (
    SELECT
        fare_amount,
        trip_distance,
        tip_amount,
        CASE
            WHEN fare_amount = 0                                 THEN 'invalid'
            WHEN tip_amount = 0                                  THEN 'none'
            WHEN tip_amount / fare_amount BETWEEN 0   AND 0.10   THEN 'low'
            WHEN tip_amount / fare_amount BETWEEN 0.10 AND 0.20  THEN 'standard'
            WHEN tip_amount / fare_amount > 0.20                 THEN 'high'
        END AS tip_bucket
    FROM trips
    WHERE fare_amount > 0
      AND payment_type = 1   -- credit card only; cash tips aren't tracked
)
SELECT
    tip_bucket,
    COUNT(*)               AS trips,
    ROUND(AVG(fare_amount), 2)    AS avg_fare,
    ROUND(AVG(trip_distance), 2)  AS avg_dist,
    ROUND(AVG(tip_amount), 2)     AS avg_tip
FROM classified
WHERE tip_bucket IS NOT NULL
GROUP BY tip_bucket
ORDER BY trips DESC;
```

The `payment_type = 1` filter is critical: the NYC taxi data only records tips for credit-card transactions. Cash tips are invisible. **The "low/none" buckets are full of cash payers, not bad tippers.** Good lesson in always reading the data dictionary.
</details>

---

## Exercise 1.8 — Day-over-day percent change

For each day, the revenue and the percent change vs the previous day.

<details>
<summary>Solution</summary>

```sql
WITH daily AS (
    SELECT DATE(tpep_pickup_datetime) AS day, SUM(total_amount) AS revenue
    FROM trips
    GROUP BY day
)
SELECT
    day,
    revenue,
    LAG(revenue) OVER (ORDER BY day) AS prev_day_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY day))
        / LAG(revenue) OVER (ORDER BY day) * 100,
        2
    ) AS pct_change
FROM daily
ORDER BY day;
```

You'll see large jumps around January 1 (low) → first business day (high), and dips during the snowstorm week.
</details>

---

## Exercise 1.9 — Gaps and islands: which days had no trips?

Find any day in the dataset's range with zero trips.

<details>
<summary>Solution</summary>

```sql
WITH date_range AS (
    SELECT
        MIN(DATE(tpep_pickup_datetime)) AS min_date,
        MAX(DATE(tpep_pickup_datetime)) AS max_date
    FROM trips
),
all_days AS (
    SELECT UNNEST(GENERATE_SERIES(min_date, max_date, INTERVAL '1 day'))::DATE AS day
    FROM date_range
),
trips_per_day AS (
    SELECT DATE(tpep_pickup_datetime) AS day, COUNT(*) AS trips
    FROM trips
    GROUP BY day
)
SELECT
    d.day,
    COALESCE(t.trips, 0) AS trips
FROM all_days d
LEFT JOIN trips_per_day t USING (day)
WHERE COALESCE(t.trips, 0) = 0
ORDER BY d.day;
```

On the January 2024 file you should find zero zero-trip days — taxis run every day. But the technique is the standard one for sales / inventory / events analysis where missing days matter.
</details>

---

## Exercise 1.10 — Cohort retention (the stretch)

Group trips by the pickup-location ID. For each pickup location's "cohort week" (the first week we see trips from there), compute: how many distinct days within the next 4 weeks did we see trips from there? **In one query.**

This is the **cohort retention** pattern that interviewers love asking.

<details>
<summary>Solution</summary>

```sql
WITH first_appearance AS (
    SELECT
        PULocationID,
        MIN(DATE_TRUNC('week', tpep_pickup_datetime))::DATE AS cohort_week
    FROM trips
    GROUP BY PULocationID
),
trips_with_cohort AS (
    SELECT
        t.PULocationID,
        f.cohort_week,
        DATE(t.tpep_pickup_datetime) AS trip_date,
        DATE_DIFF('day', f.cohort_week, DATE(t.tpep_pickup_datetime)) AS days_since_cohort_start
    FROM trips t
    JOIN first_appearance f USING (PULocationID)
)
SELECT
    cohort_week,
    COUNT(DISTINCT PULocationID) AS cohort_size,
    COUNT(DISTINCT CASE WHEN days_since_cohort_start <  7  THEN trip_date END) AS week_0_active_days,
    COUNT(DISTINCT CASE WHEN days_since_cohort_start BETWEEN 7  AND 13 THEN trip_date END) AS week_1_active_days,
    COUNT(DISTINCT CASE WHEN days_since_cohort_start BETWEEN 14 AND 20 THEN trip_date END) AS week_2_active_days,
    COUNT(DISTINCT CASE WHEN days_since_cohort_start BETWEEN 21 AND 27 THEN trip_date END) AS week_3_active_days
FROM trips_with_cohort
GROUP BY cohort_week
ORDER BY cohort_week;
```

For a single month of data the retention picture is partial — only one cohort has 4 weeks observable. The structure is the point.

**This same query shape works for:** user retention, store reactivation, product re-purchase, anything-cohort-anything. Memorize it.
</details>

---

## Exercise 1.11 — Compare query plans

DuckDB's `EXPLAIN` shows the plan. Run the same logical query two ways and compare:

```sql
-- Way A: window function + QUALIFY
EXPLAIN
SELECT * FROM trips
QUALIFY ROW_NUMBER() OVER (PARTITION BY VendorID ORDER BY tpep_pickup_datetime DESC) <= 3;

-- Way B: subquery
EXPLAIN
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY VendorID ORDER BY tpep_pickup_datetime DESC) AS rn
    FROM trips
) WHERE rn <= 3;
```

You'll usually find the plans are identical — modern optimizers rewrite QUALIFY to a subquery. **Readability is the point of QUALIFY**, not raw speed.

For more dramatic comparisons, try comparing a `LEFT JOIN ... WHERE NULL` anti-pattern vs `ANTI JOIN` on two real tables — the plans will differ visibly.

---

## Submission checklist

- [ ] DuckDB installed; taxi parquet downloaded; `trips` view created
- [ ] Exercises 1.1-1.10 solved in SQL only (no pandas crutch)
- [ ] You used `QUALIFY` at least three times
- [ ] You used `LAG` or `LEAD` at least twice
- [ ] You used a frame clause (`ROWS BETWEEN ...`) at least once
- [ ] You used `CASE WHEN` to bucket
- [ ] You understand why "low tip" buckets are full of cash payers
- [ ] You can explain the gaps-and-islands trick (Part 8 of theory) cold
- [ ] You ran `EXPLAIN` on at least one query

---

## What you just did

You replaced what would be ~500 lines of pandas across 10 exercises with ~150 lines of SQL. Each query is auditable, readable top-to-bottom, and runs against 3M-row data in milliseconds on a laptop.

Window functions, CTEs, and `QUALIFY` are the SQL features most engineers never reach for and then look surprised when they see them in someone else's code. After this week they should feel as natural as `GROUP BY`.

Week 02 brings pandas and Polars — for when SQL stops paying. But carry this rule into the rest of the curriculum: **if it's expressible in SQL, write it in SQL.** Reproducible. Auditable. Fast. Boring. All the things you want in production data work.

---

**Next**: [Week 02: pandas + Polars →](../week-02-pandas-and-polars/readme.md)
