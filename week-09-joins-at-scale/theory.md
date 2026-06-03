# Week 09: Theory — Joins at Scale

Joins look easy until **both tables are big**, the keys **don't match exactly**, the timestamps are **off by half a second**, or you're trying to join "anything within 500 meters of this point." This week is the practical reference for the join patterns nobody teaches you in SQL 101.

By the end you'll be able to do **temporal**, **geographic**, and **range** joins fluently; reason about which side of a join gets broadcast vs shuffled; and spot Cartesian disasters before they take down your warehouse.

---

## Part 1: The four classic joins (recap)

Already covered in week 01. The reference:

```
A INNER JOIN B    → only matching rows
A LEFT JOIN B     → all A rows + B (nulls where no match)
A RIGHT JOIN B    → all B rows + A (avoid; flip to LEFT)
A FULL OUTER B    → all rows from both
A SEMI JOIN B     → A rows that have a match in B (no duplication)
A ANTI JOIN B     → A rows with NO match in B
```

**Always normalize keys before joining.** Whitespace, case, trailing periods break exact joins silently. From week 08:

```sql
ON LOWER(TRIM(a.email)) = LOWER(TRIM(b.email))
```

If you find yourself doing this often, materialize a normalized column rather than re-computing in every join.

---

## Part 2: Broadcast vs shuffle — what's happening underneath

When you write `A JOIN B ON a.x = b.x`, the engine has two strategies:

### Broadcast (hash) join

If **B is small** (fits in memory): send a full copy to every worker, then each worker locally joins its slice of A.

- **O(|A|)** time
- **Memory: |B| per worker**
- **No network shuffle**
- Used when one side is < some threshold (Spark default: 10MB; DuckDB: a few hundred MB)

### Shuffle (sort-merge) join

If **both sides are large**: hash each row by the join key, send each hash bucket to one worker. Each worker has all rows for some keys, joins locally.

- **O(|A|+|B|)** time
- **Heavy network shuffle**
- The default for large × large joins

The takeaway: **the engine picks for you, but the smaller side wins.** For interactive queries, putting the small dimension table as the *right* side of a `LEFT JOIN` hints to the planner. For DuckDB / Polars on a laptop, this rarely matters; on Spark / Snowflake / BigQuery, it does.

### When joins explode

A subtle case: A has 1M rows; B has 1M rows; but the join key has **5 distinct values** with **200k rows each**. Each match produces `200k × 200k = 40B` rows per key — Cartesian per key. The right-shaped data plus the wrong-shaped join key kills you.

Detect by checking **key cardinality** before joining (week 04's profiling helps).

---

## Part 3: Temporal joins — the "as-of" pattern

The most-asked SQL question that classic SQL doesn't have a clean answer for:

> "For each event, give me the most recent reference value as of that event's timestamp."

Examples:
- "For each trade, what was the FX rate when it happened?"
- "For each ad impression, what was the user's plan at the time?"
- "For each session, what was the model version live then?"

The naive approach:

```sql
SELECT
    e.*,
    r.rate
FROM events e
LEFT JOIN rates r
    ON r.currency = e.currency
    AND r.effective_from <= e.ts
    AND (r.effective_to > e.ts OR r.effective_to IS NULL)
```

This **works** for tables with explicit `[from, to)` intervals but produces a row explosion if multiple intervals overlap, and there's no rule like that for sparse rate updates.

### The as-of join

The modern syntax (Polars, DuckDB, kdb+, Spark):

```python
# Polars
events.join_asof(
    rates,
    left_on="ts", right_on="effective_from",
    by="currency",
    strategy="backward",   # use the latest rate <= event ts
)
```

```sql
-- DuckDB
SELECT e.*, r.rate
FROM events e ASOF LEFT JOIN rates r
    ON e.currency = r.currency
    AND e.ts >= r.effective_from
```

Internally: the engine sorts both tables by `(by_keys, ts)` and walks them in O(n+m) — extremely fast.

### Backward vs forward vs nearest

| Strategy | Returns the rates row where |
|---|---|
| **Backward** | `r.ts <= e.ts` and is the largest such (default; "latest known value at event time") |
| **Forward** | `r.ts >= e.ts` and is the smallest such ("next scheduled value") |
| **Nearest** | minimal `|r.ts - e.ts|` |

For most ML / analytics work: **backward** is what you want. "What did we know at event time?" That's also point-in-time correctness for ML training (Part 5).

---

## Part 4: Point-in-time correctness — the ML feature pitfall

When building training data for ML, you want features **as they would have been known at prediction time** — not their current values.

Example: predicting churn. For an event at `2024-01-15`, you want:
- `customer.plan` as of 2024-01-15, NOT customer's current plan
- `customer.lifetime_orders_count` as of 2024-01-15
- `account.last_login` as of 2024-01-15

If you naively join the current dimension table, you've **leaked the future** into the training set. The model learns "users who churn have plan=Cancelled" — well, of course, that's the current state.

### The fix: as-of joins on snapshots

```python
# customer_history is an SCD2-style table with [valid_from, valid_to]
training_data = events.join_asof(
    customer_history,
    left_on="event_ts",
    right_on="valid_from",
    by="customer_id",
    strategy="backward",
)
```

For the lab you'll do this end-to-end. This is **the** feature-store pattern; week 11 builds on it.

---

## Part 5: Geographic joins — point-in-polygon

The two common geographic joins:

| Join | Question |
|---|---|
| **Point-in-polygon** | "Which neighborhood is this lat/lng in?" |
| **Nearest-neighbor / within-radius** | "Which 3 trees are nearest this address?" |

### Point-in-polygon

```sql
-- DuckDB Spatial extension
LOAD spatial;

SELECT
    t.tree_id,
    n.borough,
    n.neighborhood
FROM trees t
JOIN neighborhoods n
    ON ST_Contains(n.geometry, ST_Point(t.longitude, t.latitude));
```

For 100k points × 100 polygons: a few seconds with the spatial index. Without spatial indexing, it's `points × polygons` — fine at this scale; crippling at 1M × 10k.

### Nearest-neighbor

```sql
-- "Find nearest 5 neighborhoods to each event"
SELECT
    e.event_id,
    n.neighborhood,
    ST_Distance(ST_Point(e.lon, e.lat), n.centroid) AS distance_m
FROM events e
CROSS JOIN neighborhoods n
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY e.event_id
    ORDER BY ST_Distance(ST_Point(e.lon, e.lat), n.centroid)
) <= 5;
```

This is `events × neighborhoods` — fine if neighborhoods are small. For larger reference tables, use a **spatial index**:

```sql
CREATE INDEX nbh_idx ON neighborhoods USING RTREE (geometry);
```

PostGIS, DuckDB Spatial, BigQuery's `ST_*` all support R-tree indexes. **Always index geometries you'll join against.**

### Geographic coordinate systems

A subtle gotcha: GPS uses **WGS84** (lat, lng in degrees). Distance computations naively use Euclidean math, which is wrong at any reasonable scale.

Two ways to get correct distances:

1. **`ST_Distance_Sphere`** or **`ST_DistanceSphere`** — spherical math, meters
2. **Project to UTM** for short distances — meters, very fast

For NYC-scale work (<50km), spherical math is plenty accurate. For Trans-Atlantic distances, you might want a projected coordinate system.

### geohash and H3 as join helpers

Instead of comparing all polygons against all points, **discretize space into cells**, then join on the cell ID.

- **Geohash**: base-32 string; 7 chars ≈ 153m × 153m
- **H3** (Uber's hexagonal grid): integer; level 9 ≈ 174m sides

Workflow:
1. Compute cell ID for points and polygons (polygons → multiple cells)
2. Join on cell ID (string equality — fast)
3. Re-check with actual ST_Contains for boundary cells

This trades exactness for speed; for ~1M-row geographic joins, often the only tractable approach.

---

## Part 6: Range and interval joins

Common in finance, scheduling, validity tracking:

```sql
-- "Each transaction's tax rate at the time"
SELECT t.*, r.tax_rate
FROM transactions t
JOIN tax_rates r
    ON t.country = r.country
    AND t.ts >= r.valid_from
    AND t.ts <  r.valid_until;
```

The fancy version of as-of joins. Many engines (DuckDB, Postgres, BigQuery) optimize range joins specifically with **interval trees**.

For pandas/Polars: there's no native range join. You either:

```python
# Polars — use join_asof + filter
result = (
    transactions
    .sort("ts")
    .join_asof(tax_rates.sort("valid_from"), left_on="ts", right_on="valid_from", by="country")
    .filter(pl.col("ts") < pl.col("valid_until"))
)
```

Or drop to DuckDB for the range part. DuckDB embedded in a Polars pipeline is a common 2026 pattern.

---

## Part 7: Cartesian joins — the silent killer

Symptoms:

- "My 1M-row join produced 100B rows"
- "The query has been running for 4 hours"
- "Memory exploded"

Causes:

| Cause | Detection |
|---|---|
| **Forgot a join condition** | `JOIN` with no `ON` (some dialects allow this) |
| **Many-to-many on the wrong key** | Both sides have the same key value many times |
| **Implicit Cartesian via comma-join** | `FROM a, b WHERE ...` — same as cross join if WHERE filters are weak |

Defensive habit: **always check row counts** before and after joins.

```sql
-- Sanity check pattern
WITH left_count AS (SELECT COUNT(*) AS n FROM a),
     right_count AS (SELECT COUNT(*) AS n FROM b),
     join_count AS (SELECT COUNT(*) AS n FROM a JOIN b USING (key))
SELECT * FROM left_count, right_count, join_count;
```

If join_count > left_count, you have **fan-out**. Check whether that's intentional.

### The "expected fan-out" sanity check

Some joins legitimately fan out (a customer with many orders). The pattern:

```sql
-- Expected: join_count / left_count = average orders per customer
-- If it's 1.0, no fan-out
-- If it's 5.0, average 5 orders per customer (probably fine)
-- If it's 5,000.0, something is wrong
```

Decide what fan-out factor is reasonable for your data; alert when reality diverges.

### Pre-aggregating to avoid fan-out

Instead of:

```sql
SELECT c.*, COUNT(o.id) AS n_orders, SUM(o.amount) AS total
FROM customers c LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id;
```

(Fan-out then collapse — wastes work.) Better:

```sql
WITH order_agg AS (
    SELECT customer_id, COUNT(*) AS n_orders, SUM(amount) AS total
    FROM orders GROUP BY customer_id
)
SELECT c.*, o.n_orders, o.total
FROM customers c LEFT JOIN order_agg o ON c.id = o.customer_id;
```

Aggregate first, join second. **No fan-out; faster; cleaner.**

---

## Part 8: Cross-source joins (no shared key)

The hardest case: two tables that conceptually overlap but **share no key**.

| Source A | Source B |
|---|---|
| CRM customers (name, email, phone, company) | Stripe charges (email, name, ip) |
| Salesforce leads (name, company, title) | LinkedIn scrape (name, current_company) |

The pattern: **entity resolution** (week 05). Build a probabilistic match between the two; join on the resolved cluster ID.

```
A ─────┐
       ├── ER pipeline ──── golden_records (cluster_id per source row)
B ─────┘

Then:
SELECT a.*, b.*
FROM A a
JOIN clusters ca USING (a_id)
JOIN clusters cb USING (cluster_id)
JOIN B b ON cb.b_id = b.id;
```

For a 1-shot join, even fuzzy joins with rapidfuzz work. For ongoing pipelines, **invest in the ER infrastructure**.

---

## Part 9: SQL vs DataFrame engine tradeoffs

A practical question: for joining, where do you compute?

| Engine | Strengths | Weaknesses |
|---|---|---|
| **DuckDB** | Fast on a laptop; SQL; spatial extension; predicate pushdown on Parquet | Single-machine |
| **Polars** | Lazy mode; great with Arrow; expression API for complex joins | Some advanced SQL features missing |
| **pandas** | Familiar; ecosystem | Slow on large joins; no spatial native |
| **Spark / Snowflake / BigQuery** | Distributed; scales | Cost; ops overhead; harder to test |

**For this curriculum**: DuckDB + Polars handles everything up to ~50 GB on a laptop. Beyond that, week 16 (production) touches the distributed stack.

---

## Part 10: Anti-patterns

| Anti-pattern | Why bad |
|---|---|
| `LEFT JOIN ... WHERE right.id IS NULL` instead of `ANTI JOIN` | Slower plan; same intent |
| Joining on un-normalized strings | Misses matches silently |
| Joining without checking key cardinality | Fan-out surprises |
| Geographic join without spatial index | `O(n×m)` instead of `O(n log m)` |
| Range join expressed as as-of + filter without sorting | Slow and confusing |
| Comma-join (`FROM a, b WHERE ...`) | Easy to forget a condition → Cartesian |

The senior version: **always EXPLAIN, always check row counts**.

---

## Part 11: Connect to the rest of the curriculum

- **Week 04 (quality)**: Validation on the joined table — pandera check that row count matches expectation (no surprise fan-out).
- **Week 10 (external enrichment)**: All joins; geographic and temporal patterns from this week.
- **Week 11 (features)**: Point-in-time correctness via as-of joins.
- **Week 16 (pipelines)**: Joins materialized as dbt models.

---

## What's next

In [lab.md](lab.md) you'll:

- Build the canonical as-of join: NYC taxi trips × hourly weather
- Do point-in-polygon: NYC taxi pickups × neighborhood polygons
- Compute Cartesian-join detection: row-count sanity check function
- Aggregate-then-join to avoid fan-out
- Use the DuckDB Spatial extension for a real geo join
- (Stretch) Time-window join: events × user-state-snapshot for ML training

By end of week 09 you can join any two datasets correctly, even when the keys are timestamps, polygons, or messy text.
