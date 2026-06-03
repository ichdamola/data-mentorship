# Week 04: Lab — Profile, Validate, Quarantine

You'll profile the NYC trees data from week 03, write expectations in two tools (pandera and Great Expectations), watch them fail when reality diverges, and build a quarantine pipeline. By the end you'll have a validation harness you'd run in CI.

## Setup

```bash
uv add pandera great-expectations soda-core ydata-profiling duckdb polars matplotlib
```

```python
import polars as pl
import pandas as pd
import duckdb
import pandera as pa
from pandera.typing import Series
from pathlib import Path
import time

SILVER_PATH = Path("data/silver/nyc_trees.parquet")
assert SILVER_PATH.exists(), "Run week 03 lab first, or use the NYC taxi parquet"
```

If you didn't do the week-03 ingest, substitute the taxi parquet:

```python
SILVER_PATH = Path("data/yellow_tripdata_2024-01.parquet")
```

The exercises below are written for the NYC trees data; the patterns transfer.

---

## Exercise 4.1 — DuckDB SUMMARIZE (the 5-second profile)

```python
con = duckdb.connect()
summary = con.sql(f"SUMMARIZE FROM read_parquet('{SILVER_PATH}')").fetchdf()
print(summary[["column_name", "column_type", "min", "max", "avg", "null_percentage", "count"]])
```

You should see one row per column with stats. **Skim the null_percentage column** — anything above 0% deserves a question. For the trees data:
- `spc_common` should be ~0%
- `health` will be ~5-10% (some trees haven't been assessed)
- `tree_dbh` should be ~0%

This 5-second scan flags the columns to look at more carefully.

---

## Exercise 4.2 — ydata-profiling (the 30-second-but-thorough profile)

```python
from ydata_profiling import ProfileReport

# Convert to pandas for ydata-profiling
df_pd = pl.read_parquet(SILVER_PATH).to_pandas()

profile = ProfileReport(df_pd, title="NYC Trees — Quality Profile", minimal=True)
profile.to_file("reports/nyc_trees_profile.html")
print("open reports/nyc_trees_profile.html in a browser")
```

The full report is a 5-MB HTML file with:
- Per-column distributions, top values, distinct counts, missing patterns
- Correlation matrix
- Sample rows
- Warnings (high correlation, high cardinality, etc.)

**Skim it.** Note three observations about the data you didn't have before this report. Send that report to a hypothetical stakeholder — that's the artifact that gets people aligned on what the data actually is.

---

## Exercise 4.3 — Manual quick profile

For dataset you query repeatedly, build a tiny manual profile function:

```python
def quick_profile(df: pl.DataFrame, max_unique_show: int = 5) -> None:
    print(f"shape: {df.shape}")
    print(f"size:  {df.estimated_size() / 1024 / 1024:.2f} MB\n")
    for col in df.columns:
        s = df[col]
        nulls = s.null_count()
        null_pct = nulls / len(s) * 100
        try:
            unique = s.n_unique()
        except Exception:
            unique = "?"
        line = f"  {col:<25s} type={str(s.dtype):<15s} nulls={null_pct:>5.1f}%  unique={unique}"
        if s.dtype.is_numeric():
            line += f"  min={s.min()}  max={s.max()}"
        print(line)

df = pl.read_parquet(SILVER_PATH)
quick_profile(df)
```

This is the function you copy-paste into every notebook. Customize as you like.

---

## Exercise 4.4 — A pandera schema for the trees data

```python
import pandera.polars as ppa
from pandera.polars import DataFrameSchema, Column, Check

schema = DataFrameSchema(
    columns={
        "tree_id": Column(int, checks=[
            Check.greater_than_or_equal_to(0),
            Check.unique_values_eq_count("tree_id"),  # all unique
        ], nullable=False),
        "spc_common": Column(str, nullable=True),
        "boroname": Column(str, checks=[
            Check.isin(["Manhattan", "Bronx", "Brooklyn", "Queens", "Staten Island"]),
        ], nullable=False),
        "status": Column(str, checks=[
            Check.isin(["Alive", "Stump", "Dead"]),
        ], nullable=True),
        "health": Column(str, checks=[
            Check.isin(["Good", "Fair", "Poor"]),
        ], nullable=True),
        "latitude": Column(float, checks=[
            Check.in_range(40.4, 40.95),
        ], nullable=False),
        "longitude": Column(float, checks=[
            Check.in_range(-74.3, -73.6),
        ], nullable=False),
        "tree_dbh": Column(int, checks=[
            Check.greater_than_or_equal_to(0),
            Check.less_than_or_equal_to(200),
        ], nullable=True),
    },
    strict=False,   # allow extra columns
)

# Validate
try:
    validated = schema.validate(df, lazy=True)   # lazy=True: collect all errors
    print(f"✓ all {len(df)} rows pass validation")
except pa.errors.SchemaErrors as e:
    print(f"✗ validation failures:")
    print(e.failure_cases.head(20))
```

If your data is clean, you'll see all pass. **Now break it on purpose:**

```python
# Inject a bad row
df_dirty = df.head(100).clone()
df_dirty = pl.concat([df_dirty, pl.DataFrame({
    "tree_id": [-1],
    "spc_common": ["NoSuchSpecies"],
    "boroname": ["Hoboken"],     # invalid
    "status": ["Unknown"],        # invalid
    "health": ["Excellent"],      # invalid
    "latitude": [42.0],           # out of range
    "longitude": [-80.0],         # out of range
    "tree_dbh": [500],            # out of range
})], how="diagonal_relaxed")

try:
    schema.validate(df_dirty, lazy=True)
except pa.errors.SchemaErrors as e:
    print("Caught these failures:")
    print(e.failure_cases.head(20))
```

You should see 5+ failures listed — boroname, status, health, latitude/longitude, tree_dbh. Each row, each rule, captured.

---

## Exercise 4.5 — Great Expectations (the heavyweight)

GX 1.x is heavier. Initialize a project:

```python
import great_expectations as gx

context = gx.get_context(mode="ephemeral")  # in-memory, no on-disk project

# Load data
data_source = context.data_sources.add_pandas("nyc_trees_pd")
data_asset = data_source.add_dataframe_asset(name="trees")
batch_def = data_asset.add_batch_definition_whole_dataframe("batch1")
batch = batch_def.get_batch(batch_parameters={"dataframe": df_pd})

# Expectations
batch.expect_column_values_to_not_be_null("tree_id")
batch.expect_column_values_to_be_unique("tree_id")
batch.expect_column_values_to_be_in_set("boroname", ["Manhattan", "Bronx", "Brooklyn", "Queens", "Staten Island"])
batch.expect_column_values_to_be_between("latitude", 40.4, 40.95)
batch.expect_column_values_to_be_between("longitude", -74.3, -73.6)

# Validate
result = batch.validate()
print(f"success: {result.success}")
for r in result.results:
    if not r.success:
        print(f"  ✗ {r.expectation_config.expectation_type}: {r.result}")
```

**Compare experiences.** GX is more verbose and forces more ceremony than pandera; in exchange you get an HTML data-docs site, a CLI, and a richer expectation library. **For an ad-hoc check, pandera wins. For a fleet of 30 datasets with shared definitions, GX wins.**

---

## Exercise 4.6 — Quarantine pattern

```python
def validate_and_quarantine(df: pl.DataFrame, schema: DataFrameSchema, out_good: str, out_bad: str):
    """Split good rows from bad. Returns (n_good, n_bad)."""
    try:
        schema.validate(df, lazy=True)
        # All pass
        df.write_parquet(out_good)
        return len(df), 0
    except pa.errors.SchemaErrors as e:
        # Identify bad rows from failure_cases
        bad_indices = set()
        for case in e.failure_cases.iter_rows(named=True):
            if "index" in case and case["index"] is not None:
                bad_indices.add(case["index"])

        if not bad_indices:
            # The failure is column-level (e.g., column type) — bail out
            raise

        df_with_idx = df.with_row_index("__idx")
        df_good = df_with_idx.filter(~pl.col("__idx").is_in(list(bad_indices))).drop("__idx")
        df_bad = df_with_idx.filter(pl.col("__idx").is_in(list(bad_indices))).drop("__idx")

        Path(out_good).parent.mkdir(parents=True, exist_ok=True)
        Path(out_bad).parent.mkdir(parents=True, exist_ok=True)
        df_good.write_parquet(out_good)
        df_bad.write_parquet(out_bad)
        return len(df_good), len(df_bad)

n_good, n_bad = validate_and_quarantine(
    df_dirty,
    schema,
    "data/silver/nyc_trees_clean.parquet",
    "data/quarantine/nyc_trees_bad.parquet",
)
print(f"good: {n_good}  quarantined: {n_bad}")

# Set a threshold
QUARANTINE_THRESHOLD = 0.05
total = n_good + n_bad
if n_bad / total > QUARANTINE_THRESHOLD:
    raise RuntimeError(f"too many quarantined: {n_bad}/{total} = {n_bad/total:.2%}")
```

This is the senior pattern: bad rows aren't blocking, but they're tracked, and the threshold catches systemic breakage.

---

## Exercise 4.7 — Tiny observability table

Track quality metrics over time. Each run appends to a `_metrics` table.

```python
import datetime as dt

METRICS_PATH = Path("data/_metrics.parquet")

def compute_metrics(df: pl.DataFrame, dataset: str, run_at: dt.datetime) -> pl.DataFrame:
    rows = []
    rows.append({
        "dataset": dataset, "metric": "row_count", "value": float(len(df)),
        "run_at": run_at,
    })
    for col in df.columns:
        nulls = df[col].null_count()
        rows.append({
            "dataset": dataset, "metric": f"null_pct.{col}",
            "value": nulls / len(df), "run_at": run_at,
        })
    return pl.DataFrame(rows)

run_at = dt.datetime.utcnow()
new_metrics = compute_metrics(df, "nyc_trees", run_at)

# Append to history
if METRICS_PATH.exists():
    existing = pl.read_parquet(METRICS_PATH)
    combined = pl.concat([existing, new_metrics])
else:
    combined = new_metrics
combined.write_parquet(METRICS_PATH)

print(f"logged {len(new_metrics)} metrics @ {run_at}")
```

Run this twice (the second time after deliberately corrupting some rows). Then query the change:

```python
metrics = pl.read_parquet(METRICS_PATH)
print(
    metrics
    .filter(pl.col("metric").str.starts_with("null_pct."))
    .sort("run_at")
)
```

A daily dashboard fed from this table is your minimum-viable observability. Plot `null_pct.health` over time; you'll see degradation as upstream rot creeps in.

---

## Exercise 4.8 — Write a contract YAML

For one of your datasets, write the contract in YAML:

```yaml
# contracts/nyc_trees.yml
name: nyc_trees
owner: data-mentorship-lab
version: 1.0.0
description: NYC street tree census, from 2015 NYC Forestry data
source: https://data.cityofnewyork.us/resource/uvpi-gqnh.json

schema:
  - name: tree_id
    type: integer
    description: Unique tree identifier (NYC Forestry ID)
    nullable: false
    unique: true
  - name: spc_common
    type: string
    description: Common name of species (e.g., 'red maple')
    nullable: true   # ~1% missing
  - name: boroname
    type: string
    description: NYC borough name
    nullable: false
    allowed_values: [Manhattan, Bronx, Brooklyn, Queens, Staten Island]
  - name: latitude
    type: float
    description: Tree location, WGS84
    nullable: false
    constraint: "between 40.4 and 40.95"
  - name: longitude
    type: float
    description: Tree location, WGS84
    nullable: false
    constraint: "between -74.3 and -73.6"

quality:
  - row_count > 100000
  - duplicate_count(tree_id) = 0
  - null_pct(boroname) = 0
  - null_pct(latitude) < 0.001
  - boroname's distinct values are exactly the 5 NYC boroughs

sla:
  freshness: 24 hours          # we re-pull once a day
  schema_change: 14-day notice
  completeness: 99.5%

deprecation:
  v0_changes:
    - field 'created_at' was nullable; v1 makes it required
```

In a real org this YAML lives in the producer team's repo, is reviewed by consumers, gets a CI check that runs the validation suite, and is the source of truth for "what does the data look like."

---

## Exercise 4.9 (stretch) — Same checks in Soda

```yaml
# checks/nyc_trees.yml
checks for nyc_trees:
  - row_count > 100000
  - duplicate_count(tree_id) = 0
  - missing_count(latitude) = 0
  - invalid_count(boroname) = 0:
      valid values: [Manhattan, Bronx, Brooklyn, Queens, Staten Island]
  - schema:
      warn:
        when forbidden column present: [_internal_only]
  - freshness using created_at < 30d
```

Run with:

```bash
soda scan -d local -c configuration.yml checks/nyc_trees.yml
```

The Soda experience: lighter than GX, less code than pandera, sits closest to the warehouse. **Worth knowing exists.** For a warehouse-heavy stack it's the right tool; in a Python-first workflow pandera is usually easier.

---

## Submission checklist

- [ ] DuckDB SUMMARIZE run on your silver Parquet; null_percentage column scanned
- [ ] ydata-profiling HTML report generated and opened
- [ ] Custom `quick_profile()` function written and saved for reuse
- [ ] pandera schema with ≥ 10 checks; lazy validation collects all errors
- [ ] Same checks expressed in Great Expectations; comparison made
- [ ] Quarantine pipeline splits good vs bad with a configurable threshold
- [ ] Observability `_metrics.parquet` accumulates row count + null % per run
- [ ] Contract YAML written for one dataset
- [ ] (Stretch) Soda checks for same dataset

---

## What you just did

You now have the three legs of data quality: **profiling** (understand), **validation** (guard), **contracts** (negotiate). Plus the quarantine pattern that makes failures non-blocking and the observability scaffold that catches drift.

From week 05 onward, every cleaning operation you do will end with the same template: write expectations for what the cleaning is supposed to achieve, validate post-clean, quarantine the rows that don't match. **Cleaning without validation is just hopeful editing.**

---

**Next**: [Week 05: Deduplication + Entity Resolution →](../week-05-deduplication-entity-resolution/readme.md)
