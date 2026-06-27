# Week 01: Theory - SQL Deep

Most engineers know enough SQL to be dangerous and not enough to be fast. The basic dialect - `SELECT`, `WHERE`, `JOIN`, `GROUP BY` - is taught everywhere. The senior dialect - **window functions, CTEs, qualify, lateral, recursive, gaps-and-islands, semi/anti** - is taught in scattered blog posts and learned by accident.

This is the missing chapter. By the end you should be able to replace a 200-line pandas pipeline with 20 lines of SQL and not feel like you're losing anything.

---

## Part 1: The logical execution order

A SQL query is written in one order and executed in another. Memorize this:

```
1.  FROM        ← pick the source tables, apply joins
2.  WHERE       ← row-level filter on raw rows
3.  GROUP BY    ← collapse rows into groups
4.  HAVING      ← filter on groups
5.  SELECT      ← compute output columns (including window functions)
6.  DISTINCT    ← dedup
7.  ORDER BY    ← sort
8.  LIMIT       ← cut
```

**Why this matters:**

- You can't use a `SELECT` alias in `WHERE` because `SELECT` runs after `WHERE`. (DuckDB and BigQuery actually let you. Postgres doesn't.)
- Window functions run during `SELECT`, so you can't filter on them in `WHERE`. **Use `QUALIFY`** (Part 5) or wrap in a subquery.
- `HAVING` is for filtering aggregates; `WHERE` is for filtering rows. They're not interchangeable.

---

## Part 2: CTEs (Common Table Expressions)

CTEs are named subqueries declared at the top of a statement with `WITH`. They turn a tangled nested mess into a top-to-bottom story.

```sql
WITH
trips_in_january AS (
    SELECT *
    FROM read_parquet('data/yellow_tripdata_2024-01.parquet')
    WHERE tpep_pickup_datetime >= '2024-01-01'
      AND tpep_pickup_datetime <  '2024-02-01'
),
long_trips AS (
    SELECT * FROM trips_in_january
    WHERE trip_distance > 10
),
fare_per_mile AS (
    SELECT
        trip_distance,
        fare_amount,
        fare_amount / trip_distance AS dollars_per_mile
    FROM long_trips
)
SELECT * FROM fare_per_mile ORDER BY dollars_per_mile DESC LIMIT 10;
```

Read top-to-bottom. Each CTE is a *named intermediate result*. Compare to the nested-subquery version, which reads inside-out.

### When CTEs help

- Multi-step analyses where each step has a name
- Reusing the same intermediate result twice in one query
- Recursive queries (the only way to express them)

### When CTEs don't help

- Many CTEs and then one final `SELECT *` from the last - could just be a view
- Two-line queries where the CTE wrapping is more verbose than nested

### Materialized vs not

Most warehouses (Postgres ≥12, DuckDB, Snowflake, BigQuery) **inline** CTEs by default - they're just named expressions, not materialized. You can force materialization with `WITH foo AS MATERIALIZED (...)` in Postgres / DuckDB.

For performance-sensitive queries: don't assume CTEs are free. Profile.

---

## Part 3: JOIN types

Memorize the four core join types (already known, included for completeness):

| Join | Keeps |
|---|---|
| `INNER JOIN` | Only rows matching on both sides |
| `LEFT JOIN`  | All left rows + matching right rows (nulls if no match) |
| `RIGHT JOIN` | Symmetric of LEFT - rarely used; left-flip instead |
| `FULL OUTER JOIN` | All rows from both sides |
| `CROSS JOIN` | Cartesian product (every row × every row) |

The two **less common but high-value** ones:

### Semi-join - "rows in A that have a match in B"

```sql
-- The textbook way (subquery)
SELECT * FROM customers c WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- DuckDB / Snowflake / BigQuery shortcut:
SELECT * FROM customers SEMI JOIN orders USING (customer_id);
```

Notice no duplication: even if a customer has 100 orders, `SEMI JOIN` returns the customer once. Unlike `INNER JOIN` which fan-outs.

### Anti-join - "rows in A with NO match in B"

```sql
-- The textbook way
SELECT * FROM customers c WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- DuckDB / Snowflake / BigQuery shortcut:
SELECT * FROM customers ANTI JOIN orders USING (customer_id);
```

**Use these.** They're clearer, faster, and avoid the duplication footgun of `LEFT JOIN ... WHERE right.id IS NULL`.

### The `LEFT JOIN ... WHERE NULL` antipattern

```sql
-- DON'T:
SELECT c.* FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;

-- DO:
SELECT * FROM customers ANTI JOIN orders USING (customer_id);
```

Same result, half the lines, and the planner generates a better plan.

---

## Part 4: Window functions (the headline content)

A **window function** computes a value for each row using a "window" of related rows - without collapsing the rows into groups.

The shape:

```sql
window_fn() OVER (
    PARTITION BY ...    -- group rows
    ORDER BY ...        -- order within group
    ROWS BETWEEN ... AND ...  -- which rows in the group count
)
```

### Ranking functions

```sql
SELECT
    customer_id,
    order_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)        AS nth_order,
    RANK()       OVER (PARTITION BY customer_id ORDER BY order_amount DESC) AS amount_rank,
    DENSE_RANK() OVER (PARTITION BY customer_id ORDER BY order_amount DESC) AS amount_dense_rank
FROM orders;
```

Three near-twins; the differences matter:

| Function | Behavior on ties |
|---|---|
| `ROW_NUMBER` | Arbitrary tiebreak; result is always 1, 2, 3, 4, ... |
| `RANK` | Ties get same rank; next rank skips: 1, 2, 2, 4, ... |
| `DENSE_RANK` | Ties get same rank; next rank doesn't skip: 1, 2, 2, 3, ... |

The single most common application: **"give me the latest record per customer"** - `ROW_NUMBER` with `ORDER BY date DESC`, then `WHERE rn = 1`.

### Aggregate window functions

```sql
SELECT
    order_id,
    customer_id,
    order_amount,
    SUM(order_amount) OVER (PARTITION BY customer_id)                                    AS total_per_customer,
    AVG(order_amount) OVER (PARTITION BY customer_id)                                    AS avg_per_customer,
    order_amount / SUM(order_amount) OVER (PARTITION BY customer_id) * 100               AS pct_of_customer_total
FROM orders;
```

The same row data; aggregated context added as columns. No collapsing.

### Running totals (the frame clause)

```sql
SELECT
    order_id, customer_id, order_date, order_amount,
    SUM(order_amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM orders;
```

The `ROWS BETWEEN ... AND ...` clause defines the **frame** - which rows in the partition count for this row's computation:

| Frame spec | Meaning |
|---|---|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | Running total |
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | 7-day rolling sum (if daily) |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | Full partition (same as no frame) |
| `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` | Time-based 7-day window (RANGE not ROWS) |

**Tip:** `ROWS` counts rows; `RANGE` counts logical distance (calendar days, dollar amounts). For time-series with daily snapshots, both behave the same. For irregular data, use `RANGE`.

### LAG and LEAD - neighbor access

```sql
SELECT
    order_id, customer_id, order_date,
    LAG(order_date, 1)  OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_date,
    LEAD(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_date,
    DATE_DIFF('day',
        LAG(order_date, 1) OVER (PARTITION BY customer_id ORDER BY order_date),
        order_date
    ) AS days_since_last_order
FROM orders;
```

`LAG(col, n)` gets the value `n` rows before the current row in the partition. `LEAD` the same, forward.

**Useful for:**
- Time between events (sessions, retention)
- "Did this column change between rows?" (change detection)
- Gaps-and-islands (Part 8)

### `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`

```sql
SELECT
    customer_id, order_date, order_amount,
    FIRST_VALUE(order_amount) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_order_amount,
    LAST_VALUE(order_amount)  OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_amount
FROM orders;
```

**Critical gotcha:** `LAST_VALUE` with the default frame (`UNBOUNDED PRECEDING AND CURRENT ROW`) returns the current row's value, not the last in the partition. You must specify `UNBOUNDED FOLLOWING`. Everyone forgets this.

---

## Part 5: QUALIFY - filter on window functions

You can't put a window function in `WHERE` (logical-order reasons). The classical fix is a subquery; the modern fix is `QUALIFY`.

```sql
-- Classical: wrap in a subquery
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) WHERE rn = 1;

-- Modern (DuckDB / Snowflake / BigQuery / Databricks):
SELECT *,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
FROM orders
QUALIFY rn = 1;

-- Even cleaner - inline the window function:
SELECT *
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) = 1;
```

**`QUALIFY` is to window functions what `HAVING` is to aggregates.** Use it; your queries will be shorter.

Not in standard Postgres (yet, as of 17), but in DuckDB and most modern warehouses. Worth using even if you have to fall back to subqueries on Postgres.

---

## Part 6: GROUP BY tricks

### `GROUPING SETS`

Compute multiple groupings in one query:

```sql
SELECT
    payment_type,
    EXTRACT(month FROM tpep_pickup_datetime) AS month,
    COUNT(*) AS trips,
    SUM(total_amount) AS revenue
FROM read_parquet('data/yellow_tripdata_2024-01.parquet')
GROUP BY GROUPING SETS (
    (payment_type, month),  -- one row per (payment_type, month)
    (payment_type),         -- subtotals by payment_type alone
    (month),                -- subtotals by month alone
    ()                      -- grand total
);
```

Equivalent to four `UNION ALL`'d queries with different `GROUP BY`s. The planner does it in one pass.

### `ROLLUP` and `CUBE`

```sql
-- ROLLUP: hierarchical subtotals
GROUP BY ROLLUP (region, country, city)
-- Produces: (region, country, city), (region, country), (region), ()

-- CUBE: every combination
GROUP BY CUBE (region, country, city)
-- Produces all 2^3 = 8 groupings
```

`ROLLUP` is what you want for typical reporting (Region → Country → City subtotals). `CUBE` is rarely worth it; explodes combinatorially.

### `GROUPING()` to identify the grouping level

In a `GROUPING SETS` query, how do you tell which row is which subtotal? The `GROUPING(col)` function returns 1 if that column was rolled-up in this row, 0 if it was preserved:

```sql
SELECT
    CASE WHEN GROUPING(month) = 1 THEN 'all months' ELSE month::TEXT END AS month_label,
    payment_type,
    COUNT(*) AS trips
FROM trips
GROUP BY GROUPING SETS ((payment_type, month), (payment_type));
```

---

## Part 7: LATERAL joins

A `LATERAL` join lets the right side reference columns from the left side. Useful for "for each row in A, compute something using A's values".

```sql
SELECT
    c.customer_id,
    c.name,
    last3.order_date,
    last3.order_amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT order_date, order_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 3
) AS last3 ON true;
```

Returns each customer's 3 most-recent orders. **Without LATERAL** you'd need a window function + filter, or a correlated subquery with array_agg, or a complex CTE.

Supported in Postgres, DuckDB, BigQuery (as `UNNEST`), Snowflake (via flatten). Not in MySQL ≤ 8.x.

---

## Part 8: Gaps-and-islands

The canonical SQL puzzle: **find runs of consecutive rows where some condition holds**. Examples:

- Login streaks (consecutive days a user logged in)
- Sessionization (clicks within 30 min of each other are one session)
- Detecting status changes

The trick: use `ROW_NUMBER()` twice and subtract.

```sql
-- "Group consecutive dates into streaks"
WITH numbered AS (
    SELECT
        user_id, login_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn
    FROM logins
),
groups AS (
    SELECT *,
        -- `login_date - INTERVAL (rn) DAY` is NOT valid SQL. Portable forms:
        --   DuckDB:   login_date - INTERVAL 1 DAY * rn
        --   Postgres: login_date - (rn || ' days')::INTERVAL
        --   Both:     login_date - rn        -- DATE minus INT works on DuckDB and Postgres
        login_date - INTERVAL 1 DAY * rn AS streak_anchor
    FROM numbered
)
SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS streak_length
FROM groups
GROUP BY user_id, streak_anchor;
```

The magic: if logins are on consecutive dates, `login_date - rn` is the same constant for the whole streak. As soon as a gap appears, the anchor changes.

Variations: **sessionization** uses `LAG(timestamp)` and a threshold to mark session starts, then `SUM(is_session_start) OVER (...)` as the session ID.

Senior interviewers love this pattern. Knowing it cold is worth memorizing.

---

## Part 9: Recursive CTEs

The only way to express "walk a tree" or "compute a transitive closure" in SQL.

```sql
WITH RECURSIVE org AS (
    -- Base case: top of hierarchy
    SELECT employee_id, manager_id, name, 1 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: each employee, joined to org via their manager
    SELECT e.employee_id, e.manager_id, e.name, o.depth + 1
    FROM employees e
    JOIN org o ON e.manager_id = o.employee_id
)
SELECT * FROM org ORDER BY depth, name;
```

The `UNION ALL` is mandatory; the recursive arm references the CTE's own name. The recursion stops when the joined-from set produces no new rows.

Useful for:
- Org charts and parent/child hierarchies
- Bill of materials (parts → subparts)
- Friend-of-friend / shortest path (with limits)
- Generating sequences (`WITH RECURSIVE t AS (SELECT 1 AS n UNION ALL SELECT n+1 FROM t WHERE n<100)`)

DuckDB, Postgres, and most warehouses support these. Set a depth limit (`WHERE o.depth < 10`) on real data so a cycle doesn't infinite-loop.

---

## Part 10: Pivots and unpivots

### Pivot (long → wide)

```sql
-- Long data: rows of (customer, category, amount)
-- Want: one row per customer with a column per category

-- Classic: CASE WHEN per column
SELECT
    customer_id,
    SUM(CASE WHEN category = 'food'     THEN amount END) AS food_spend,
    SUM(CASE WHEN category = 'clothes'  THEN amount END) AS clothes_spend,
    SUM(CASE WHEN category = 'tech'     THEN amount END) AS tech_spend
FROM orders
GROUP BY customer_id;

-- DuckDB / Snowflake: PIVOT keyword
PIVOT orders ON category USING SUM(amount) GROUP BY customer_id;
```

The `PIVOT` syntax is a huge win when you don't know the columns up front - it learns them from the data.

### Unpivot (wide → long)

```sql
UNPIVOT wide_table ON food_spend, clothes_spend, tech_spend
INTO NAME category VALUE amount;
```

For long-format-friendly analysis (plotting, joining to dimension tables).

---

## Part 11: Common-case patterns you'll reach for weekly

### "Latest row per group"

```sql
SELECT * FROM t
QUALIFY ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY ts DESC) = 1;
```

### "Top N per group"

```sql
SELECT * FROM t
QUALIFY ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY metric DESC) <= 5;
```

### "Rank with ties allowed"

```sql
SELECT * FROM t
QUALIFY RANK() OVER (ORDER BY metric DESC) <= 10;
```

### "Pct of total"

```sql
SELECT
    id,
    amount,
    amount / SUM(amount) OVER () * 100 AS pct_of_total
FROM t;
```

### "7-day rolling avg"

```sql
SELECT
    date,
    metric,
    AVG(metric) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7
FROM daily_metrics;
```

### "First and last row per group"

```sql
SELECT DISTINCT
    customer_id,
    FIRST_VALUE(order_amount) OVER w AS first_order,
    LAST_VALUE(order_amount)  OVER w AS last_order
FROM orders
WINDOW w AS (
    PARTITION BY customer_id
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

Note the `WINDOW` clause - define a window once, reference it by name. Cleaner than repeating the spec inline.

---

## Part 12: When SQL stops paying

SQL isn't always the right tool. Drop to pandas/Polars (week 02) when:

- You need iteration with state that's awkward in SQL (some statistical fits, recurrent algorithms)
- The pipeline is exploratory and you'll write 30 small steps that change minute-to-minute
- You're calling Python ML libraries that need NumPy arrays anyway

Stay in SQL when:

- The data is bigger than RAM (DuckDB + Parquet handles 50 GB on a laptop)
- The transformations are declarative - filtering, joining, aggregating, ranking
- You want the result auditable months later by anyone, not just by the person who wrote it
- It'll run on a schedule against a real warehouse

The senior move: **prototype in pandas, productionize in SQL.** Or: write the dbt model in SQL, debug the rough edges in pandas.

---

## What's next

In [lab.md](lab.md) you'll:

- Set up DuckDB against the NYC taxi parquet
- Solve 10 progressively harder analytics questions using window functions, CTEs, qualify, and gaps-and-islands
- Build a cohort retention query (the stretch - and the one interviewers ask)
- Compare query-plan output between a naive query and a window-function rewrite

If Part 4 (window functions) was the densest section, that's by design. Reread it before opening the lab.
