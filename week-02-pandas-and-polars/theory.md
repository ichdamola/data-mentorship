# Week 02: Theory - pandas and Polars

Two dataframe libraries dominate Python data work. **pandas** is the incumbent - 15 years old, in every job description, has 90% of the StackOverflow answers. **Polars** is the modern alternative - multi-threaded by default, expression-based, ~10× faster on typical workloads, increasingly the choice for new projects.

You'll use both. This week teaches you to **think in expressions** (which Polars insists on and pandas grudgingly allows), to recognize the four pandas footguns that bite everyone, and to translate fluently between them.

---

## Part 1: The mental model - DataFrames vs Series vs expressions

### What a DataFrame actually is

A DataFrame is a **collection of named columns of equal length**. Each column has a single type. Under the hood (for both libs in 2026):

- **Backed by Apache Arrow** - a columnar in-memory format. Same data layout as Parquet. Zero-copy with each other.
- Some legacy pandas operations still copy to NumPy arrays internally; Polars stays in Arrow throughout.

The columnar layout matters for performance. Operations like "sum this column" load only that column's bytes from RAM - much less data than scanning row-by-row.

### Series (pandas) vs Column (Polars)

A pandas **Series** is a 1-D array with an **index** (a separate set of labels). The index is pandas's signature feature and its most consequential design choice. It enables `.loc['Alice']` lookups but introduces every alignment surprise you'll meet.

A Polars **Column** is just a named array. No index. Lookups are by position or by filter expression. **Simpler. Faster. Less expressive in two specific cases (time-series alignment, multi-level indexing) - but worth it.**

### The "expressions over columns" framing

The single biggest mental shift between pandas and Polars:

**pandas thinks imperatively:** "loop over rows; or apply a function to each cell; or compute this aggregate."

**Polars thinks declaratively:** "this is an *expression* that, given a column, produces a value. Apply it within this context."

```python
# pandas style (imperative)
df["fare_per_mile"] = df["fare_amount"] / df["trip_distance"]
df = df[df["fare_per_mile"] < 100]
result = df.groupby("payment_type")["fare_per_mile"].mean()

# Polars style (expression-based)
import polars as pl
result = (
    df
    .with_columns(fare_per_mile=(pl.col("fare_amount") / pl.col("trip_distance")))
    .filter(pl.col("fare_per_mile") < 100)
    .group_by("payment_type")
    .agg(pl.col("fare_per_mile").mean())
)
```

In Polars, `pl.col("fare_amount") / pl.col("trip_distance")` doesn't compute anything immediately. It builds an **expression** that knows it wants column A divided by column B. The expression gets executed when a *context* (with_columns, filter, group_by, etc.) consumes it.

**Why this matters:** expressions can be:
- **Combined** in unlimited ways without intermediate materialization
- **Lazy-evaluated** - Polars's killer feature (Part 5)
- **Parallelized** automatically - Polars knows the data flow and schedules work across threads
- **Optimized** by a query planner - column pruning, predicate pushdown, just like SQL

pandas can do most of these things eventually, but you have to know the right incantation. Polars makes them the default.

---

## Part 2: pandas idioms - the modern way

Tom Augspurger's "Modern Pandas" series is the reference. The headline patterns:

### Method chaining over reassignment

```python
# BAD - repeated reassignment makes diffs noisy
df = pd.read_parquet("trips.parquet")
df["fare_per_mile"] = df["fare_amount"] / df["trip_distance"]
df = df[df["fare_per_mile"] < 100]
df = df.groupby("payment_type")["fare_per_mile"].mean().reset_index()
result = df.sort_values("fare_per_mile", ascending=False)

# GOOD - method chain, reads top to bottom like SQL
result = (
    pd.read_parquet("trips.parquet")
    .assign(fare_per_mile=lambda d: d["fare_amount"] / d["trip_distance"])
    .query("fare_per_mile < 100")
    .groupby("payment_type")["fare_per_mile"]
    .mean()
    .reset_index()
    .sort_values("fare_per_mile", ascending=False)
)
```

The `assign` + `lambda` pattern is the most-underused pandas method. It lets you reference the in-flight DataFrame instead of needing a variable.

### `pipe` for custom steps

```python
def add_geo_features(df, geo_lookup):
    return df.merge(geo_lookup, on="PULocationID", how="left")

result = (
    pd.read_parquet("trips.parquet")
    .pipe(add_geo_features, geo_lookup=zones)
    .query("borough == 'Manhattan'")
)
```

`pipe(fn, *args)` calls `fn(self, *args)`. Useful for breaking up long chains across functions.

### `query` for string-based filters

```python
# Without query
df[(df["fare_amount"] > 10) & (df["trip_distance"] > 1) & (df["payment_type"] == 1)]

# With query
df.query("fare_amount > 10 and trip_distance > 1 and payment_type == 1")
```

Faster (uses numexpr where available), more readable, supports variable interpolation:

```python
min_fare = 10
df.query("fare_amount > @min_fare")
```

### `.loc` and `.iloc` - the two ways to index

| Method | What it uses |
|---|---|
| `df.loc[label]`  | Label-based - uses the index values |
| `df.iloc[position]` | Position-based - uses 0-based integer positions |
| `df.at[label, col]` | Scalar lookup by label (fast for single value) |
| `df.iat[i, j]` | Scalar lookup by position |

**Never use `df[i]` for a row.** It's ambiguous - column or position depending on dtype. Always `.loc` or `.iloc`.

---

## Part 3: The four pandas footguns

### Footgun 1: `SettingWithCopyWarning`

```python
filtered = df[df["fare_amount"] > 10]
filtered["fare_amount"] = filtered["fare_amount"] * 2
# SettingWithCopyWarning - did you set on the copy or the original?
```

pandas doesn't always know whether `filtered` is a view or a copy of `df`. The warning means "I'm uncertain; your assignment may not propagate where you think."

**Fixes:**

1. Use `.copy()` explicitly: `filtered = df[df["fare_amount"] > 10].copy()`
2. Use `.loc` for the modification: `df.loc[df["fare_amount"] > 10, "fare_amount"] *= 2`
3. Use `.assign()` in a chain - produces a guaranteed new DataFrame

**As of pandas 3.0 (released 2025), Copy-on-Write is always on** - the `pd.options.mode.copy_on_write` toggle is deprecated and removed. Setting it on 3.x raises a `FutureWarning`; on 2.2+ it's already the default behavior. **Don't disable it.** If you want to see the historical footgun, pin `pandas==2.0` in a scratch env - on modern pandas the whole class of bug above quietly went away.

### Footgun 2: `apply` is slow

```python
df["distance_km"] = df["trip_distance"].apply(lambda x: x * 1.609344)   # SLOW
df["distance_km"] = df["trip_distance"] * 1.609344                        # FAST
```

`.apply` falls back to Python-level iteration. Vectorized arithmetic uses NumPy underneath - typically 10-100× faster.

When you genuinely need element-wise logic that's not vectorizable, you have three options:

1. **Vectorize anyway** with `np.where`, `np.select`, `pd.cut`, etc.
2. **Use Polars** - its `pl.when().then().otherwise()` expressions JIT-compile to multi-threaded code
3. **Numba `@njit`** for the truly arbitrary case

The instinct to reach for `.apply` is what makes pandas slow for new users. **Resist.**

### Footgun 3: Mixed dtypes (`object` columns)

```python
df = pd.read_csv("messy.csv")
df.dtypes
# id                int64
# name              object   ← strings, but pandas calls them "object"
# created_at        object   ← actually dates, but pandas didn't parse them
# amount            object   ← actually floats, but some are "12.50" and some are "N/A"
```

`object` is pandas's name for "Python objects" - a column of arbitrary Python references. Slow, memory-hungry, breaks vectorization. The most common source of "why is this 100× slower than I expected" surprises.

**Fixes:**

1. Use `dtype=` in `read_csv` to set types up front
2. Use `parse_dates=` to coerce date columns
3. Convert to proper dtypes after load: `df["amount"] = pd.to_numeric(df["amount"], errors="coerce")`
4. Use pandas 2.0+ **PyArrow-backed dtypes** (`dtype_backend="pyarrow"`) - first-class strings, dates, dictionaries; no object fallback

```python
df = pd.read_csv("messy.csv", dtype_backend="pyarrow")
```

This gets you Polars-like performance and dtype hygiene without leaving pandas.

### Footgun 4: `inplace` is a lie

```python
df.sort_values("amount", inplace=True)   # appears to save memory; doesn't
```

`inplace=True` doesn't avoid the copy under the hood. It just hides the return value, breaking method chaining. The pandas team has been quietly deprecating it for years.

**Always:** `df = df.sort_values("amount")` and chain instead.

---

## Part 4: Polars expressions - the actual mental model

Polars's expression API is the part most pandas users find foreign at first. Once it clicks it's hard to go back.

### The four contexts

An expression has no value on its own - it's a recipe. It executes when consumed by a **context**:

| Context | What it does |
|---|---|
| `df.select(...)` | Compute expressions and return only those |
| `df.with_columns(...)` | Compute expressions and add to existing columns |
| `df.filter(...)` | Drop rows where expression is `False` |
| `df.group_by(...).agg(...)` | Aggregate per group |

Example walking each:

```python
import polars as pl

df = pl.read_parquet("data/yellow_tripdata_2024-01.parquet")

# select - keep only computed columns
df.select(
    pl.col("fare_amount"),
    fare_per_mile=pl.col("fare_amount") / pl.col("trip_distance"),
)

# with_columns - keep all, add new
df.with_columns(
    fare_per_mile=pl.col("fare_amount") / pl.col("trip_distance"),
)

# filter - keep rows where expression is true
df.filter(pl.col("fare_amount") > 10)

# group_by + agg - collapse per group
df.group_by("payment_type").agg(
    avg_fare=pl.col("fare_amount").mean(),
    n_trips=pl.len(),
    p99_fare=pl.col("fare_amount").quantile(0.99),
)
```

### Expressions compose

The cleanest Polars feature: expressions chain endlessly without intermediate variables.

```python
(
    df
    .with_columns(
        # Multiple new columns from one expression chain each
        fare_per_mile=pl.col("fare_amount") / pl.col("trip_distance"),
        is_big_tip=pl.col("tip_amount") / pl.col("fare_amount") > 0.25,
        pickup_hour=pl.col("tpep_pickup_datetime").dt.hour(),
    )
    .filter(pl.col("fare_amount").is_between(1, 1000))
    .group_by("pickup_hour")
    .agg(
        n_trips=pl.len(),
        avg_fare=pl.col("fare_amount").mean(),
        big_tip_rate=pl.col("is_big_tip").mean(),
    )
    .sort("pickup_hour")
)
```

Note `pl.col("is_big_tip").mean()` - averaging a boolean column gives the proportion. Idiomatic across Polars.

### `pl.when().then().otherwise()` - the CASE WHEN

```python
df.with_columns(
    tip_bucket=(
        pl.when(pl.col("fare_amount") == 0).then(pl.lit("invalid"))
        .when(pl.col("tip_amount") == 0).then(pl.lit("none"))
        .when((pl.col("tip_amount") / pl.col("fare_amount")) < 0.10).then(pl.lit("low"))
        .when((pl.col("tip_amount") / pl.col("fare_amount")) < 0.20).then(pl.lit("standard"))
        .otherwise(pl.lit("high"))
    )
)
```

Pure expression, fully vectorized, no Python loop. Compare to pandas `np.select` - same effect, more verbose syntax.

### Window functions: `.over()`

```python
# "fare amount as percent of customer's total" - same as SQL Part 4 / Part 11
df.with_columns(
    pct_of_day_total=pl.col("fare_amount") / pl.col("fare_amount").sum().over(pl.col("pickup_date"))
)

# "rank within day"
df.with_columns(
    rank_in_day=pl.col("fare_amount").rank("ordinal").over("pickup_date")
)
```

The `.over(col)` modifier is Polars's `OVER (PARTITION BY col)`. Same semantics; cleaner syntax than pandas `df.groupby(col).transform`.

---

## Part 5: Lazy mode - Polars's killer feature

Eager mode (the default, what we've been writing) executes each operation immediately. Lazy mode builds a query plan, optimizes it, then executes it all at once when you call `.collect()`.

```python
import polars as pl

# Lazy - note .scan_parquet (returns LazyFrame), not .read_parquet
lazy = pl.scan_parquet("data/yellow_tripdata_2024-01.parquet")

plan = (
    lazy
    .filter(pl.col("fare_amount") > 10)
    .filter(pl.col("trip_distance") > 1)
    .group_by("payment_type")
    .agg(pl.col("total_amount").mean())
)

print(plan.explain())   # show the query plan - even before running
result = plan.collect()  # NOW execute
```

What Polars optimizes:

| Optimization | What it does |
|---|---|
| **Predicate pushdown** | The two `.filter()` calls happen during file read, before any other op |
| **Projection pushdown** | Only the columns actually used (fare_amount, trip_distance, payment_type, total_amount) get loaded from disk |
| **Common subexpression elimination** | If you compute `pl.col("a") + pl.col("b")` twice, it's computed once |
| **Slice pushdown** | If your query ends with `.limit(1000)`, Polars stops reading after enough rows |

For Parquet files specifically, projection + predicate pushdown is **the** win - you can query 100 GB of data on a 16 GB laptop because Polars never loads what it doesn't need.

```python
# This actually works on a laptop against a 50 GB parquet
pl.scan_parquet("huge.parquet").filter(pl.col("date") == "2024-01-15").collect()
```

Habit to adopt: **default to `.scan_parquet` / lazy mode**. Drop to eager only when debugging.

---

## Part 6: Side-by-side - pandas → Polars Rosetta Stone

| Operation | pandas | Polars |
|---|---|---|
| Read CSV | `pd.read_csv("f.csv")` | `pl.read_csv("f.csv")` |
| Read Parquet (lazy) | n/a (always eager) | `pl.scan_parquet("f.parquet")` |
| Select columns | `df[["a", "b"]]` | `df.select("a", "b")` |
| Filter | `df[df["a"] > 5]` or `df.query("a > 5")` | `df.filter(pl.col("a") > 5)` |
| Add column | `df["c"] = df["a"] + df["b"]` | `df.with_columns(c=pl.col("a") + pl.col("b"))` |
| Drop column | `df.drop(columns=["a"])` | `df.drop("a")` |
| Rename | `df.rename(columns={"a": "x"})` | `df.rename({"a": "x"})` |
| Sort | `df.sort_values("a", ascending=False)` | `df.sort("a", descending=True)` |
| Group + aggregate | `df.groupby("a")["b"].mean()` | `df.group_by("a").agg(pl.col("b").mean())` |
| Join | `pd.merge(a, b, on="id", how="left")` | `a.join(b, on="id", how="left")` |
| Concat (rows) | `pd.concat([a, b])` | `pl.concat([a, b])` |
| Pivot | `df.pivot_table(...)` | `df.pivot(...)` |
| Melt (unpivot) | `df.melt(...)` | `df.unpivot(...)` |
| First/last per group | `df.groupby("a").head(1)` | `df.group_by("a").first()` |
| Window function | `df.groupby("a")["b"].transform("sum")` | `df.with_columns(pl.col("b").sum().over("a"))` |
| Day of week | `df["d"].dt.dayofweek` *(0=Mon..6=Sun)* | `pl.col("d").dt.weekday()` *(**1=Mon..7=Sun**)* |
| Hour of day | `df["d"].dt.hour` | `pl.col("d").dt.hour()` |
| Week of year | `df["d"].dt.isocalendar().week` | `pl.col("d").dt.week()` |

> ⚠️ **Day-of-week trap.** pandas uses 0-6 (Mon=0); Polars uses 1-7 (Mon=1) since 0.19. The *same code* `dow < 6` means Mon–Fri in pandas and Mon–Sat in Polars. Use `is_in([1..5])` / `is_in([0..4])` explicitly when porting between the two.

The mental compiler: pandas mostly speaks "verbs on DataFrames", Polars speaks "expressions in contexts". After 30 minutes of practice the translation is automatic.

---

## Part 7: Performance - when each wins

### Where Polars clearly wins

- Files > 1 GB (lazy + projection pushdown)
- group_by aggregations with millions of groups
- Repetitive transformations (it parallelizes; pandas is single-threaded)
- Anything where you'd be tempted to reach for `.apply` in pandas

Real numbers for the NYC taxi 3M-row file on a M2 laptop:

| Operation | pandas | Polars (eager) | Polars (lazy) |
|---|---|---|---|
| Read full file | 1.3 s | 0.9 s | 0.02 s* |
| group_by 1 col | 180 ms | 35 ms | 30 ms |
| group_by + 5 aggs | 350 ms | 60 ms | 50 ms |
| Filter + group_by | 280 ms | 50 ms | 25 ms* |

*Lazy doesn't actually read until `.collect()`; "read" cost is the planning only.

### Where pandas still wins

- Tiny data (< 100k rows) - Polars's planning overhead doesn't amortize
- Ecosystem integration: matplotlib, seaborn, sklearn all expect DataFrames-with-index
- Time-series operations with hierarchical indexes (Polars handles time series fine but doesn't replicate every pandas idiom)
- Anything that lives in a notebook and reads from a CSV every cell

### The sane production stance (2026)

- **New project**: Polars, lazy mode by default.
- **Existing pandas project**: incremental migration. Replace hot paths first.
- **Sklearn pipelines / matplotlib charts**: convert at the boundary - `df.to_pandas()` is cheap (Arrow zero-copy on most operations).
- **dbt models**: SQL (week 01), not either of these.

---

## Part 8: When to drop to NumPy

Both libs are built on top of arrays. Sometimes you want to skip the DataFrame ceremony:

```python
import numpy as np

# Polars → NumPy (zero-copy for numeric types)
arr = df.select("fare_amount").to_numpy()

# pandas → NumPy
arr = df["fare_amount"].values   # legacy
arr = df["fare_amount"].to_numpy()  # preferred
```

Use NumPy directly when:

- Doing linear algebra (`@`, `linalg.solve`)
- Writing functions that just need an array
- Calling sklearn / scipy / numba

The DataFrame is structure for the *boundaries* of your pipeline (input data, output data). NumPy is for the inside of compute kernels.

---

## Part 9: PyArrow as the new lingua franca

In 2026, both pandas and Polars are migrating to **Arrow as the canonical in-memory format**. Two consequences:

- **Zero-copy conversion**: `pd.read_parquet` → DataFrame → `df.to_polars()` → LazyFrame is bytes-cheap. No serialization.
- **Better dtype support in pandas**: `pd.read_csv("f.csv", dtype_backend="pyarrow")` gives you proper string, date, and missing-value support - no more `object` columns.

A practical mental model: **Arrow is the data format. pandas and Polars are two different APIs over the same data.** Pick whichever API is more ergonomic for the immediate task.

---

## Part 10: Things you can safely defer

Topics that look adjacent but you can skip for now:

| Topic | When to revisit |
|---|---|
| pandas MultiIndex (hierarchical indexing) | Time-series / panel data work specifically |
| Dask, Modin | When data > RAM. Polars handles up to ~5× RAM via streaming; beyond that, Dask. |
| Vaex, cuDF (GPU) | Specialized - GPU shines on huge data, niche otherwise |
| `pd.Categorical` | When you have low-cardinality strings repeated millions of times |
| `numexpr`, `Bottleneck` | Auto-enabled in pandas. You rarely call them directly. |

---

## What's next

In [lab.md](lab.md) you'll:
- Set up a project with `uv add pandas polars pyarrow`
- Do the same 8 analyses in both pandas and Polars, side by side
- Benchmark the two
- Try the four pandas footguns deliberately and feel them bite
- Convert a CSV to a Parquet and measure the win
- Migrate one pandas script to Polars from scratch

By end of week 02 you'll be fluent in both; you'll know which one to reach for; and you'll write less of the kind of pandas that everyone secretly hates.
