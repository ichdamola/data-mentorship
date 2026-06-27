# Week 02: Lab - pandas and Polars side by side

Same 8 analyses, in both libraries. By the end you'll have written every common operation twice and will know which idiom you prefer.

## Setup

```bash
uv add pandas polars pyarrow matplotlib
```

We'll reuse the NYC taxi parquet from week 01. If you don't have it:

```bash
mkdir -p data
curl -o data/yellow_tripdata_2024-01.parquet \
    https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet
```

In a notebook `02_pandas_polars.ipynb`:

```python
import pandas as pd
import polars as pl
import numpy as np
import matplotlib.pyplot as plt
import time

# Modern pandas: CoW is the default in pandas 3.0; the toggle is deprecated.
# (Leave this commented; on pandas 3.x it FutureWarns; on 2.2+ it's already on.)
# pd.options.mode.copy_on_write = True

PATH = "data/yellow_tripdata_2024-01.parquet"
```

---

## Exercise 2.1 - Load + inspect

```python
# pandas
df_pd = pd.read_parquet(PATH, dtype_backend="pyarrow")
print(df_pd.shape)
print(df_pd.dtypes)
print(df_pd.head(3))

# polars
df_pl = pl.read_parquet(PATH)
print(df_pl.shape)
print(df_pl.schema)
print(df_pl.head(3))
```

Notice: pandas with `dtype_backend="pyarrow"` gives you proper `string`, `datetime`, `int64` dtypes - no `object` columns. **Use this whenever loading messy data.**

---

## Exercise 2.2 - Filter + add column + aggregate (both styles)

The headline pattern: clean rows, derive a column, summarize.

### pandas - method chain version

```python
result_pd = (
    df_pd
    .query("fare_amount > 0 and trip_distance > 0")
    .assign(fare_per_mile=lambda d: d["fare_amount"] / d["trip_distance"])
    .query("fare_per_mile < 100")
    .groupby("payment_type")
    .agg(
        n_trips=("fare_per_mile", "size"),
        avg_fare_per_mile=("fare_per_mile", "mean"),
        median_fare_per_mile=("fare_per_mile", "median"),
    )
    .sort_values("avg_fare_per_mile", ascending=False)
    .reset_index()
)
print(result_pd)
```

### Polars - expression chain version

```python
result_pl = (
    df_pl
    .filter(
        (pl.col("fare_amount") > 0)
        & (pl.col("trip_distance") > 0)
    )
    .with_columns(
        fare_per_mile=pl.col("fare_amount") / pl.col("trip_distance"),
    )
    .filter(pl.col("fare_per_mile") < 100)
    .group_by("payment_type")
    .agg(
        n_trips=pl.len(),
        avg_fare_per_mile=pl.col("fare_per_mile").mean(),
        median_fare_per_mile=pl.col("fare_per_mile").median(),
    )
    .sort("avg_fare_per_mile", descending=True)
)
print(result_pl)
```

Both produce the same answer. Which felt more readable? **Most developers find Polars's version cleaner once they get used to `pl.col(...)`.**

---

## Exercise 2.3 - Benchmark them

```python
def bench(fn, n_iter=5):
    times = []
    for _ in range(n_iter):
        t0 = time.perf_counter()
        result = fn()
        times.append(time.perf_counter() - t0)
    return min(times)   # use min: more stable than mean

def pandas_pipeline():
    return (
        df_pd
        .query("fare_amount > 0 and trip_distance > 0")
        .assign(fare_per_mile=lambda d: d["fare_amount"] / d["trip_distance"])
        .query("fare_per_mile < 100")
        .groupby("payment_type")
        .agg(avg=("fare_per_mile", "mean"))
    )

def polars_eager_pipeline():
    return (
        df_pl
        .filter((pl.col("fare_amount") > 0) & (pl.col("trip_distance") > 0))
        .with_columns(fare_per_mile=pl.col("fare_amount") / pl.col("trip_distance"))
        .filter(pl.col("fare_per_mile") < 100)
        .group_by("payment_type")
        .agg(avg=pl.col("fare_per_mile").mean())
    )

def polars_lazy_pipeline():
    return (
        pl.scan_parquet(PATH)
        .filter((pl.col("fare_amount") > 0) & (pl.col("trip_distance") > 0))
        .with_columns(fare_per_mile=pl.col("fare_amount") / pl.col("trip_distance"))
        .filter(pl.col("fare_per_mile") < 100)
        .group_by("payment_type")
        .agg(avg=pl.col("fare_per_mile").mean())
        .collect()
    )

print(f"pandas:        {bench(pandas_pipeline)*1000:.1f} ms")
print(f"polars eager:  {bench(polars_eager_pipeline)*1000:.1f} ms")
print(f"polars lazy:   {bench(polars_lazy_pipeline)*1000:.1f} ms")
```

Typical results on a laptop:

```
pandas:        180 ms
polars eager:   40 ms      ~5x
polars lazy:    25 ms      ~7x
```

Lazy beats eager because it includes the file read and only loads the 4 columns it needs - projection pushdown in action.

---

## Exercise 2.4 - `pl.scan_parquet` projection pushdown

Watch Polars only read the columns the query uses.

```python
plan = (
    pl.scan_parquet(PATH)
    .filter(pl.col("payment_type") == 1)
    .select(["fare_amount", "tip_amount"])
    .with_columns(tip_pct=pl.col("tip_amount") / pl.col("fare_amount"))
)

# Show the optimized plan
print(plan.explain())
```

You'll see something like:

```
WITH_COLUMNS:
  [[(col("tip_amount")) / (col("fare_amount"))].alias("tip_pct")]
   FILTER [(col("payment_type")) == (1)] FROM
    PARQUET SCAN [data/yellow_tripdata_2024-01.parquet]
    PROJECT 3/19 COLUMNS    ← only 3 of 19 columns loaded
    SELECTION: [(col("payment_type")) == (1)]   ← filter pushed down
```

**`PROJECT 3/19 COLUMNS` is the magic.** Without projection pushdown you'd be reading 19 columns × 3M rows. With it, 3 columns × 3M rows + filter applied during scan. That's why lazy mode is faster.

---

## Exercise 2.5 - Window functions in both libraries

Replicate the SQL window-function patterns from week 01 in pandas and Polars.

### "Top 3 trips per day"

```python
# pandas - use groupby + apply (slow but works)
top3_pd = (
    df_pd
    .assign(pickup_date=lambda d: d["tpep_pickup_datetime"].dt.date)
    .sort_values(["pickup_date", "total_amount"], ascending=[True, False])
    .groupby("pickup_date")
    .head(3)
)

# pandas - use rank() (faster than apply, looks more SQL-like)
top3_pd_v2 = (
    df_pd
    .assign(
        pickup_date=lambda d: d["tpep_pickup_datetime"].dt.date,
        rn=lambda d: d.groupby(d["tpep_pickup_datetime"].dt.date)["total_amount"]
                       .rank(method="first", ascending=False)
    )
    .query("rn <= 3")
)

# Polars - clean window function via .over()
top3_pl = (
    df_pl
    .with_columns(pickup_date=pl.col("tpep_pickup_datetime").dt.date())
    .with_columns(
        rn=pl.col("total_amount").rank("ordinal", descending=True).over("pickup_date")
    )
    .filter(pl.col("rn") <= 3)
)
```

The Polars `.over()` is the cleanest of the three - closest to the SQL spec.

### Running total

```python
# pandas
df_pd_sample = df_pd.head(10000).sort_values("tpep_pickup_datetime")
df_pd_sample["running_total"] = df_pd_sample["total_amount"].cumsum()

# Polars
df_pl_sample = df_pl.head(10000).sort("tpep_pickup_datetime")
df_pl_sample = df_pl_sample.with_columns(running_total=pl.col("total_amount").cum_sum())
```

Both are one line. Polars uses `cum_sum` (with underscore); pandas uses `cumsum` (without). Trivial difference; just memorize both.

### Lag / lead

```python
# pandas
df_pd_sample["prev_fare"] = df_pd_sample["fare_amount"].shift(1)

# Polars
df_pl_sample = df_pl_sample.with_columns(prev_fare=pl.col("fare_amount").shift(1))
```

Same operation; Polars uses `shift`, pandas uses `shift`. Match.

---

## Exercise 2.6 - Group-by + multiple aggregations

The patterns differ more here.

### pandas

```python
agg_pd = (
    df_pd
    .query("fare_amount > 0")
    .groupby("PULocationID")
    .agg(
        n_trips=("fare_amount", "size"),
        avg_fare=("fare_amount", "mean"),
        p99_fare=("fare_amount", lambda s: s.quantile(0.99)),
        sum_revenue=("total_amount", "sum"),
    )
    .sort_values("sum_revenue", ascending=False)
    .head(20)
)
```

Note the lambda for p99 - pandas's named-agg syntax doesn't accept quantile directly.

### Polars

```python
agg_pl = (
    df_pl
    .filter(pl.col("fare_amount") > 0)
    .group_by("PULocationID")
    .agg(
        n_trips=pl.len(),
        avg_fare=pl.col("fare_amount").mean(),
        p99_fare=pl.col("fare_amount").quantile(0.99),
        sum_revenue=pl.col("total_amount").sum(),
    )
    .sort("sum_revenue", descending=True)
    .head(20)
)
```

Polars's expression API treats `quantile(0.99)` like any other method - no lambda gymnastics. **For multi-aggregate group-bys, Polars is consistently nicer.**

---

## Exercise 2.7 - The four pandas footguns, observed

Trigger each one deliberately.

### Footgun 1: `SettingWithCopyWarning` (in pre-CoW mode)

```python
# Historical footgun demo. On pandas 3.x this toggle is gone (FutureWarning);
# CoW is always on. The pre-CoW behavior described below could only be
# reproduced on pandas <2.2. Read for context; don't try to disable.
# pd.options.mode.copy_on_write = False

df_copy = df_pd.copy()
filtered = df_copy[df_copy["fare_amount"] > 10]
filtered["fare_amount"] = filtered["fare_amount"] * 2
# Likely SettingWithCopyWarning

# On modern pandas (3.x), CoW is always on - the warning doesn't fire
# regardless of any toggle, because pandas tracks copies internally.
filtered = df_copy[df_copy["fare_amount"] > 10]
filtered["fare_amount"] = filtered["fare_amount"] * 2   # clean
```

**CoW is always-on in pandas 3.x.** On 2.2 it was opt-in; on 3.0 the toggle is deprecated and removed. The class of bug above is silently fixed for everyone now.

### Footgun 2: `.apply` is slow

```python
def slow_pipeline():
    return df_pd["trip_distance"].apply(lambda x: x * 1.609344)

def fast_pipeline():
    return df_pd["trip_distance"] * 1.609344

print(f"slow apply:  {bench(slow_pipeline)*1000:.1f} ms")
print(f"vectorized:  {bench(fast_pipeline)*1000:.1f} ms")
```

You should see vectorized ~50-200× faster. The lesson: anytime you reach for `.apply`, try to vectorize first. Most things can be - `np.where`, `pd.cut`, simple arithmetic.

### Footgun 3: `object` columns sneak in

```python
# Load with the default backend - note dtypes
df_default = pd.read_parquet(PATH)
print(df_default.dtypes)

# vs PyArrow backend - proper string/date types
df_pa = pd.read_parquet(PATH, dtype_backend="pyarrow")
print(df_pa.dtypes)
```

For Parquet you usually won't see this (Parquet has its own schema). For CSV you absolutely will.

### Footgun 4: `inplace` is a lie

```python
import sys

# inplace=True doesn't save memory
df_sample = df_pd.head(10000)
before = sys.getsizeof(df_sample)
df_sample.sort_values("fare_amount", inplace=True)
after = sys.getsizeof(df_sample)
print(f"size before/after inplace: {before} / {after}")
# Same size - no memory was saved.

# And it broke method chaining, hides the result.
# Use: df = df.sort_values("fare_amount")
```

---

## Exercise 2.8 - Migrate a pandas script to Polars

Take this contrived pandas script:

```python
def pandas_messy():
    df = pd.read_parquet(PATH, dtype_backend="pyarrow")
    df = df[df["fare_amount"] > 0]
    df["pickup_hour"] = df["tpep_pickup_datetime"].dt.hour
    df["fare_per_mile"] = df["fare_amount"] / df["trip_distance"]
    df = df[df["fare_per_mile"] < 100]
    result = df.groupby(["pickup_hour", "payment_type"]).agg(
        n_trips=("fare_per_mile", "size"),
        avg_fpm=("fare_per_mile", "mean"),
    ).reset_index()
    result = result.sort_values(["pickup_hour", "avg_fpm"], ascending=[True, False])
    return result
```

**Your job:** rewrite as a single Polars expression chain in lazy mode. Aim for ≤ 12 lines.

<details>
<summary>Reference solution</summary>

```python
def polars_clean():
    return (
        pl.scan_parquet(PATH)
        .filter(pl.col("fare_amount") > 0)
        .with_columns(
            pickup_hour=pl.col("tpep_pickup_datetime").dt.hour(),
            fare_per_mile=pl.col("fare_amount") / pl.col("trip_distance"),
        )
        .filter(pl.col("fare_per_mile") < 100)
        .group_by(["pickup_hour", "payment_type"])
        .agg(
            n_trips=pl.len(),
            avg_fpm=pl.col("fare_per_mile").mean(),
        )
        .sort(["pickup_hour", "avg_fpm"], descending=[False, True])
        .collect()
    )
```

Verify they match:

```python
res_pd = pandas_messy()
res_pl = polars_clean()
print(f"pandas:   {bench(pandas_messy)*1000:.1f} ms")
print(f"polars:   {bench(polars_clean)*1000:.1f} ms")
print(f"speedup:  {bench(pandas_messy)/bench(polars_clean):.1f}x")
```

You should see ~5-10× speedup, same result.
</details>

---

## Exercise 2.9 - Round-trip conversion

Show pandas ↔ Polars zero-copy round-trip.

```python
# Polars → pandas
pd_from_pl = df_pl.to_pandas()
print(type(pd_from_pl), pd_from_pl.shape)

# pandas → Polars
pl_from_pd = pl.from_pandas(df_pd)
print(type(pl_from_pd), pl_from_pd.shape)

# Both share Arrow buffers underneath - conversion is cheap
```

Use this at boundaries:
- Polars internals → pandas for matplotlib / seaborn / sklearn
- pandas → Polars for the hot path of your pipeline
- Polars → pandas → Polars to grab a single missing pandas idiom

The cost is real but minimal. **Don't agonize over which library to use for the whole pipeline; convert at the edge of the function.**

---

## Exercise 2.10 - Pick your style and lock it in

In your own notebook from this lab, pick the version (pandas or Polars) for each of the 9 exercises above that you found clearer or faster. Save those into a file `02_solutions.py`.

You're building muscle memory. Next week you'll do ingestion in whichever style felt right; week 04 onwards you'll mostly use Polars (it's faster, and the lab exercises are smoother in it) - but you should be **able** to write either fluently.

---

## Submission checklist

- [ ] On pandas 3.x - CoW is always on; no toggle to set
- [ ] Parquet loaded in both pandas (with `dtype_backend="pyarrow"`) and Polars
- [ ] Same group-by + agg analysis done in both libs; results identical
- [ ] Benchmark shows Polars at least 3× faster than pandas on the multi-step pipeline
- [ ] `pl.scan_parquet(...).explain()` output read; you understand `PROJECT 3/19 COLUMNS`
- [ ] Window function exercises completed in both libraries
- [ ] Multi-aggregate group-by completed in both
- [ ] The four pandas footguns observed; vectorized version of `.apply` measured at ≥ 50× speedup
- [ ] pandas script migrated to Polars lazy chain; result and speedup verified
- [ ] `02_solutions.py` saved with your preferred version of each exercise

---

## What you just did

You can now write the same operation idiomatically in both pandas and Polars, you understand the **expressions-vs-imperatives** mental model that makes Polars feel different, and you've felt the four pandas footguns bite (so you'll avoid them in real code).

For the rest of this curriculum we'll lean toward **Polars + lazy mode** as the default, dropping to pandas when an ecosystem library demands it. By week 16 you'll have a production pipeline that uses Polars at the hot path and SQL for the materialized models - the senior 2026 stack.

---

**Next**: [Week 03: Ingestion →](../week-03-ingestion/readme.md)
