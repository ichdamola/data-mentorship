# Week 04: Theory - Data Quality Fundamentals

You ingested data in week 03. Now: is it any good?

"Looks fine" doesn't scale. The senior practice is **measure quality explicitly, encode expectations as code, fail loudly when reality drifts.** That practice is called **data quality engineering**, and the modern tooling - Great Expectations, Soda, pandera - exists to make it routine.

This week is the framework: profile to *understand* the data, validate to *guard* it, and contract to *negotiate* with the upstream team that produced it.

---

## Part 1: The six dimensions of data quality

The standard taxonomy (every framework agrees on these, with minor wording differences):

| Dimension | Definition | Example fail |
|---|---|---|
| **Completeness** | All expected records present; no unexpected nulls | "Half of yesterday's orders are missing" |
| **Validity** | Each value conforms to its type and constraints | "country = 'XX' isn't in our ISO list" |
| **Accuracy** | Values reflect reality | "the order amount is wrong" |
| **Consistency** | Same fact, same answer everywhere | "user count differs between two reports" |
| **Uniqueness** | No unintended duplicates | "two rows for one order" |
| **Timeliness** | Data is fresh enough for the use | "the dashboard shows last week's numbers" |

Memorize them. Every quality issue you'll diagnose maps to at least one.

**Most failures are completeness or validity.** Accuracy is hardest to check (you need ground truth). Consistency and timeliness are systemic and often political. Uniqueness is the easy win - week 05's dedup work handles most cases.

---

## Part 2: Profiling - understand before you validate

Profiling is the diagnostic phase: scan a dataset and produce a quality report. **You always profile first.** You can't write meaningful expectations about data you haven't characterized.

A first-pass profile of any new dataset should answer:

```
For each column:
  - Type (declared vs actual)
  - Null %
  - Distinct count
  - Min / max / mean / median / stddev (numeric)
  - Min / max length (string)
  - Top 5 values + their frequency
  - Histogram (numeric)
  - Sample of 5 rows

Pair-wise:
  - Correlation matrix (numeric)
  - Foreign-key-like relationships ("does column A's distinct values match column B?")

Whole-table:
  - Row count
  - Approximate size on disk
  - Date range (any datetime columns)
  - Duplicate row count
```

Three tools for this in Python (use the right one for the situation):

| Tool | Speed | Output | When |
|---|---|---|---|
| **ydata-profiling** (formerly pandas-profiling) | Slow | Big HTML report | First look at unfamiliar data, share with stakeholders |
| **`df.describe(include='all')` + a few custom calls** | Fast | Console | Daily-driver quick scan |
| **DuckDB SUMMARIZE** | Fast | Console | When data is in Parquet/SQL form |

DuckDB's `SUMMARIZE` is the modern hidden gem:

```sql
SUMMARIZE FROM read_parquet('data/silver/nyc_trees.parquet');
```

Returns one row per column with: name, type, min, max, average, std, q25, q50, q75, count, null %. Sub-second on 50 GB files. Use it.

---

## Part 3: Validation - encode expectations as code

Profiling tells you what *is*. Validation tells you what *should be*.

The senior pattern: **after profiling, write expectations as code, run them in CI / pipeline / on every load.** When reality drifts from expectations, the pipeline fails loudly.

A minimal expectation suite for the NYC trees data might look like:

```
tree_id              should be non-null, unique, positive integer
spc_common           should be non-null, in a known list of ~150 species
boroname             should be in {'Manhattan', 'Bronx', 'Brooklyn', 'Queens', 'Staten Island'}
status               should be in {'Alive', 'Stump', 'Dead'}
health               should be in {'Good', 'Fair', 'Poor', null}
latitude             should be between 40.4 and 40.9
longitude            should be between -74.3 and -73.6
created_at           should be after 2014-01-01 and before today
tree_dbh             should be ≥ 0 and ≤ 200
```

Each line is a contract. When a load brings in `boroname = 'Yonkers'`, validation fires. Either upstream changed (talk to them), or there's a real data corruption (investigate).

### Where to validate

| Layer | What to check |
|---|---|
| **Bronze → Silver** | Schema (types, columns); basic null counts; row count > 0 |
| **Silver → Gold** | Business rules; foreign-key integrity; row counts within expected bounds |
| **Pre-publish** | Freshness (last load < N hours ago); no critical failures |

The "**heavy** check at bronze, **business-rule** check at silver, **freshness** check at publish" pattern fits most pipelines.

---

## Part 4: The Python data-quality tooling

Three serious tools, with overlapping but distinct sweet spots.

### Great Expectations

The oldest and biggest. JSON-based "expectation suites" + a CLI + a data docs HTML site.

Strengths:
- Rich expectation library (~200 built-in)
- Generates docs automatically
- Big community, many connectors

Weaknesses:
- Heavy. Steep learning curve.
- The 1.x rewrite is meaningful churn.
- Verbose for small projects.

When: large org, many datasets, want a managed catalog of expectations.

```python
import great_expectations as gx
context = gx.get_context()
batch = context.sources.pandas_default.read_parquet("data/silver/nyc_trees.parquet")
batch.expect_column_values_to_not_be_null("tree_id")
batch.expect_column_values_to_be_between("latitude", 40.4, 40.9)
result = batch.validate()
```

### pandera

The pythonic alternative. Schema-defined in Python with type hints; runs inline.

Strengths:
- Minimal ceremony
- Native pandas + Polars support
- Strong typing, plays nicely with mypy

Weaknesses:
- Less rich than Great Expectations
- No built-in docs site
- Smaller ecosystem

When: small-to-medium project; you want validation inline with code; you like type hints.

```python
import pandera as pa
from pandera.typing import Series

class TreeSchema(pa.DataFrameModel):
    tree_id: Series[int] = pa.Field(unique=True, ge=0)
    spc_common: Series[str] = pa.Field(nullable=True)
    boroname: Series[str] = pa.Field(isin=["Manhattan", "Bronx", "Brooklyn", "Queens", "Staten Island"])
    latitude: Series[float] = pa.Field(in_range={"min_value": 40.4, "max_value": 40.9})
    longitude: Series[float] = pa.Field(in_range={"min_value": -74.3, "max_value": -73.6})

TreeSchema.validate(df)
```

### Soda

SQL-native. Define checks in YAML; runs against any warehouse.

Strengths:
- Lives close to the warehouse
- Checks are SQL - auditable by anyone who reads SQL
- Excellent freshness/SLA features

Weaknesses:
- YAML-based config can feel constraining
- Open-source version is smaller than cloud version

When: warehouse-centric stack (Snowflake, BigQuery, Redshift); checks belong with the data, not the code.

```yaml
# checks/nyc_trees.yml
checks for nyc_trees:
  - row_count > 100000
  - duplicate_count(tree_id) = 0
  - missing_count(latitude) = 0
  - invalid_count(boroname) = 0:
      valid values: [Manhattan, Bronx, Brooklyn, Queens, Staten Island]
  - freshness(created_at) < 24h
```

### Picking

For this curriculum we'll do hands-on with **pandera** (lightest, runs in-process) and **Great Expectations** (the incumbent - worth seeing). Soda is mentioned for context.

---

## Part 5: Data contracts

A **data contract** is an explicit agreement between data producer and data consumer about schema, semantics, quality, and SLA.

The historical default is **no contract** - producer changes a column type silently; consumer's pipeline breaks; meetings happen. The modern alternative:

```yaml
# orders_contract.yml
name: orders
owner: orders-platform-team
schema:
  - name: order_id
    type: string
    description: Stripe charge ID; format `ch_*`
    nullable: false
    unique: true
  - name: amount_cents
    type: integer
    description: Order total in cents (never dollars)
    nullable: false
    constraint: ">= 0"
  - name: currency
    type: string
    description: ISO 4217 currency code (e.g. 'USD', 'EUR')
    nullable: false
    allowed_values: [USD, EUR, GBP, JPY, NGN]
  - name: created_at
    type: timestamptz
    nullable: false

quality:
  - row_count_per_day > 1000
  - amount_cents.mean between 1000 and 50000   # smell check

sla:
  - freshness: 1 hour
  - completeness: 99.9% of expected rows present
  - schema_change: 14-day notice
```

The contract is **version controlled, reviewed, tested**. Breaking changes get a deprecation window. Consumers know what to expect; producers know what they're on the hook for.

This is the org-political part of data work. **Contracts don't magically appear** - someone has to sit down with the upstream team and negotiate. The book to read here is Chad Sanderson's writing on the topic.

### Why this pattern is gaining adoption

Without contracts, the data team plays whack-a-mole with breakages. With contracts, breakages are caught at code review (the producer changes the contract), validated in CI, and dealt with on a schedule.

The economics: a 10-engineer data team in 2026 cannot afford to be reactive. Contracts are the cheap fix.

---

## Part 6: Schema evolution

Real pipelines run for years. Schemas evolve. The three things that can change:

| Change | Severity | How to handle |
|---|---|---|
| **Add nullable column** | Safe | Just absorb it; old code keeps working |
| **Add required column** | Breaking - backfill needed | Default value or backfill |
| **Remove column** | Breaking | Deprecate first, remove later |
| **Rename column** | Breaking | Alias the old name during transition |
| **Change type (widen)** | Mostly safe | int32 → int64 fine; string → date risky |
| **Change type (narrow)** | Breaking | varchar(50) → varchar(10) may truncate |
| **Change semantics (same name, different meaning)** | Disaster | The reason "version your contract" exists |

The **medallion architecture** (week 03 Part 7) gives you space to handle this gracefully:

- Bronze accepts whatever comes in (typed permissively as strings if needed)
- Silver enforces the *current* schema; explicitly fail or quarantine on drift
- Gold is downstream-safe - silver's enforcement protects gold consumers

For data formats:

- **Avro** has built-in schema evolution rules (default values, aliases)
- **Parquet** silently allows column additions; everything else is your problem
- **Iceberg / Delta** add full evolution support on top of Parquet - this is why lakehouse formats are gaining

---

## Part 7: Quarantine vs hard-fail

When validation fails, what do you do?

| Strategy | When |
|---|---|
| **Hard fail** - refuse to publish | Critical pipelines; better to be late than wrong (finance, billing) |
| **Quarantine** - set bad rows aside; publish the rest | Most pipelines; bad rows go to `_quarantine/` for review |
| **Repair** - fix the bad rows automatically | Only when you trust the repair rule completely (e.g., uppercase a country code) |
| **Alert + publish** - log the problem, publish anyway | Dashboards / reports where stale data is worse than slightly-wrong data |

A good default: **quarantine** + alert. Don't poison downstream with bad rows; don't block publishing when 0.01% of rows are weird.

```python
# quarantine pattern
validated = schema.validate(df, lazy=True)   # don't stop at first error
bad_rows_mask = ~validated.success_indices
df_good = df[~bad_rows_mask]
df_bad = df[bad_rows_mask]

df_good.write_parquet("data/silver/orders.parquet")
df_bad.write_parquet(f"data/quarantine/orders_{today}.parquet")

if len(df_bad) > len(df) * 0.05:  # > 5% bad
    raise QualityError("too many quarantined; investigate before re-running")
```

---

## Part 8: Observability - beyond expectations

Expectations catch **anticipated** problems. **Observability** catches the rest - unexpected drifts, slow leaks, anomalies you didn't think to write a test for.

The observability stack at minimum tracks:

- **Row count over time** - sudden drops or spikes are usually breakage
- **Null rate per column over time** - gradual upticks signal upstream rot
- **Distinct count over time** - a categorical going from 10 values to 100 is a smell
- **Numeric distribution drift** - KL divergence or PSI between today's mean/std and last week's
- **Freshness** - time since last successful load

The modern tools here: **Monte Carlo**, **Datafold**, **Bigeye** (commercial); **Re_data**, **Elementary** (open). For this curriculum we'll roll a tiny version - append metric rows to a `_metrics` table on each run.

---

## Part 9: The political dimension

Some data quality issues are technical (a NULL where there shouldn't be one). Most are political. Examples:

- The marketing team's "customer count" doesn't match the finance team's because they define "customer" differently
- The upstream platform team renames a column without warning because their JIRA didn't mention you as an affected party
- A dashboard's "revenue" includes refunds; the executive's mental model doesn't

Technical tools (Great Expectations, contracts) help, but the real fix is **process**:

- A **data catalog** with owned definitions (DataHub, Atlan, Amundsen, OpenMetadata)
- **Office hours** where producer + consumer teams sit down
- **Pre-launch reviews** for breaking changes
- A **data steward** role that owns the definitions

These are the unsexy management bits that decide whether a quality program succeeds. Tools help; they don't substitute.

---

## Part 10: Things to defer

| Topic | When |
|---|---|
| Statistical anomaly detection (ARIMA, isolation forest) | After you've nailed the deterministic checks |
| ML-driven data observability | Vendor tools handle this; learn the principles, buy if needed |
| Data lineage (column-level) | Useful but a project of its own; lives in DataHub / OpenMetadata |
| Specific vendor stacks (Monte Carlo, Bigeye) | When the budget exists; principles transfer |

---

## What's next

In [lab.md](lab.md) you'll:

- Profile the NYC trees data from week 03 in two ways (DuckDB SUMMARIZE, ydata-profiling)
- Write a pandera schema with at least 10 checks; make 3 of them fail intentionally
- Write the same checks in Great Expectations; compare experience
- Build a quarantine pattern that splits good vs bad rows
- Add a tiny observability table that tracks row count and null rate per run
- (Stretch) Write a Soda YAML spec for the same dataset

By end of week 04 you have the validation muscle to trust your own pipelines. Weeks 05-08 build on top - every cleaning operation will be paired with a validation check that says "I cleaned what I expected to clean."
