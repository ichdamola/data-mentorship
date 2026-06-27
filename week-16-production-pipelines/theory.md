# Week 16: Theory - Production Data Pipelines (Capstone)

End of the curriculum. Sixteen weeks of cleaning, enriching, modeling, dashboarding. **This week stitches it all together** into a pipeline that runs on a schedule, survives failures, alerts when things break, and produces output the org trusts.

The modern data stack - dbt, Dagster (or Airflow), the medallion architecture, CDC - has converged on a set of patterns that work. This week makes them concrete with a capstone deliverable: a running pipeline you'd actually deploy.

---

## Part 1: What "production" actually means

A pipeline is in production when:

1. **It runs on a schedule** without manual intervention
2. **It fails loudly** when something is wrong - alerts to a person who can fix it
3. **It's reproducible** - same inputs produce same outputs
4. **It's observable** - you can answer "what ran when, was it healthy, what changed"
5. **It's documented** - both how it works and what the outputs mean

A notebook that runs once is not production. A Cron-driven Python script with no monitoring is barely production. Production is **the whole loop**: orchestration + tests + alerts + docs + a person on call.

---

## Part 2: The modern data stack (2026 edition)

What every mid-sized data team uses:

```
┌───────────────────────────────────────────────────────────────┐
│  SOURCES                                                       │
│  REST APIs · Postgres CDC · SaaS exports · Kafka topics        │
└──────────────┬────────────────────────────────────────────────┘
               ▼
┌───────────────────────────────────────────────────────────────┐
│  INGESTION                                                     │
│  Airbyte · Fivetran · Stitch · dlt · custom Python             │
└──────────────┬────────────────────────────────────────────────┘
               ▼
┌───────────────────────────────────────────────────────────────┐
│  WAREHOUSE / LAKEHOUSE (the bronze landing)                    │
│  Snowflake · BigQuery · Databricks · Redshift · DuckDB (local) │
└──────────────┬────────────────────────────────────────────────┘
               ▼
┌───────────────────────────────────────────────────────────────┐
│  TRANSFORMATION                                                │
│  dbt (the universal default)                                   │
│  → silver: typed, validated                                    │
│  → gold: business semantics, dashboard-ready                   │
└──────────────┬────────────────────────────────────────────────┘
               ▼
┌───────────────────────────────────────────────────────────────┐
│  ORCHESTRATION                                                 │
│  Dagster · Airflow · Prefect · Mage                            │
└──────────────┬────────────────────────────────────────────────┘
               ▼
┌───────────────────────────────────────────────────────────────┐
│  CONSUMERS                                                     │
│  Metabase · Looker · Hex · Streamlit · ML feature stores       │
└───────────────────────────────────────────────────────────────┘
```

The standard 2026 small-team stack: **Python ingestion + DuckDB warehouse + dbt + Dagster + Metabase**. Free, open source, runs on a laptop or a $30/month VPS, scales to maybe 100GB before you need to upgrade.

For larger teams: replace DuckDB with Snowflake/BigQuery. Everything else stays the same.

---

## Part 3: The medallion architecture

**Bronze → Silver → Gold.** The framing from week 03; here applied as the structure of the whole pipeline.

| Layer | What | Schema strictness | Audience |
|---|---|---|---|
| **Bronze** | Raw landed data (Parquet, JSONL, CSV) | Lenient - type everything as string if needed | Engineers only |
| **Silver** | Cleaned, typed, validated, deduped | Strict - enforce schema; quarantine bad rows | Analysts |
| **Gold** | Modeled business entities; aggregated | Strict + semantic | Stakeholders, dashboards |

In dbt, this maps to:

```
dbt/models/
  staging/        # bronze → silver: typing, basic cleanup
  intermediate/   # silver: business logic, joins
  marts/          # silver → gold: business entities
    core/          # the source of truth tables
    finance/       # finance-team aggregates
    marketing/     # marketing-team aggregates
```

The discipline: **gold consumers never touch bronze.** Schema changes in bronze should be absorbed at silver - gold stays stable for years.

---

## Part 4: dbt - transformation as code

**dbt** (data build tool) is the universal SQL transformation framework. Its primitives:

| Primitive | What |
|---|---|
| **Model** | A SQL `SELECT` query; dbt wraps it in `CREATE TABLE AS` or `CREATE VIEW AS` |
| **Source** | Reference to raw bronze tables |
| **Ref** | `{{ ref('stg_orders') }}` - declares a dependency |
| **Test** | Assertion on a column (`not_null`, `unique`, `accepted_values`) |
| **Macro** | Reusable SQL function |
| **Seed** | Static CSV → table |
| **Snapshot** | SCD2 (slowly changing dimension) on a source table |

The minimal dbt project:

```
dbt_project/
  dbt_project.yml
  models/
    staging/
      stg_orders.sql
      sources.yml
    marts/
      fct_revenue_daily.sql
  tests/
  macros/
  seeds/
```

A staging model:

```sql
-- models/staging/stg_orders.sql
SELECT
    order_id,
    customer_id,
    CAST(amount_cents AS BIGINT) AS amount_cents,
    UPPER(currency) AS currency,
    CAST(created_at AS TIMESTAMP) AS created_at,
FROM {{ source('raw', 'orders') }}
WHERE amount_cents > 0
```

A mart that depends on it:

```sql
-- models/marts/fct_revenue_daily.sql
SELECT
    DATE(created_at) AS day,
    currency,
    COUNT(*) AS orders,
    SUM(amount_cents) AS revenue_cents
FROM {{ ref('stg_orders') }}
GROUP BY 1, 2
```

Run: `dbt run`. Test: `dbt test`. Document: `dbt docs generate`. The same SQL runs on DuckDB, Snowflake, BigQuery, Databricks, Postgres.

### Tests as code

```yaml
# models/staging/sources.yml
sources:
  - name: raw
    tables:
      - name: orders
        columns:
          - name: order_id
            tests:
              - unique
              - not_null
          - name: currency
            tests:
              - accepted_values:
                  values: [USD, EUR, GBP, JPY]
```

`dbt test` runs all of these against the actual data. Tests live with models, get reviewed in PR, and run on every refresh.

### dbt vs notebook scripts

| | dbt | Python script |
|---|---|---|
| SQL transformation logic | ✅ | ⚠️ awkward |
| Dependency management | ✅ DAG-aware | ❌ |
| Tests with the code | ✅ | partial |
| Auto docs | ✅ | manual |
| Reusable across environments | ✅ | manual |
| Anyone-who-reads-SQL can review | ✅ | depends |

For analytics transformations, **dbt is the default.** Python scripts for ingestion + Python ML models for non-SQL logic; everything else in dbt.

---

## Part 5: Orchestration - Dagster vs Airflow

You need something that:

- Schedules runs (every hour, every day, every push to a CDC topic)
- Manages dependencies between tasks
- Retries on failure
- Logs everything
- Provides observability (dashboard of past runs)

The big two:

| Tool | Strength | Weakness |
|---|---|---|
| **Airflow** | The incumbent; everywhere; huge community | Aging; task-centric; harder to test |
| **Dagster** | Modern; asset-centric; easier dev experience | Smaller community; some pain points still |
| **Prefect** | Cleaner Python-first | Dropped market share recently |
| **Mage** | All-in-one with built-in notebook UI | Niche |

For a **new project in 2026**: **Dagster.** Asset-centric modeling matches how analytics data flows; the dev loop is much faster than Airflow.

For an **existing org with Airflow**: stay with Airflow. Migration is rarely worth it.

### Dagster's asset-centric model

In Dagster, you declare assets (tables, files, models) and their dependencies. Dagster handles the scheduling, lineage, and materialization.

```python
from dagster import asset, AssetIn

@asset
def raw_orders():
    """Pull raw orders from the API."""
    return pull_from_api()

@asset(ins={"raw_orders": AssetIn()})
def stg_orders(raw_orders):
    """Clean and type the raw orders."""
    return clean_orders(raw_orders)

@asset(ins={"stg_orders": AssetIn()})
def revenue_daily(stg_orders):
    return stg_orders.groupby("date").sum()
```

Dagster computes the DAG. `dagster asset materialize --select revenue_daily` runs the chain. Schedules and sensors trigger materialization automatically.

The **dagster-dbt** integration treats every dbt model as a Dagster asset, giving you unified lineage across Python + SQL.

---

## Part 6: Scheduling, alerting, observability

### Scheduling

Three styles:

| Style | When |
|---|---|
| **Cron** (e.g., daily at 02:00 UTC) | Most batch pipelines |
| **Sensor** (when an upstream changes) | Reactive; new file lands → process it |
| **Auto-materialize** (Dagster) | Based on upstream change AND staleness |

For most pipelines, daily cron is fine. For data that needs to be fresh (operational dashboards), every-15-minutes or sensor-triggered.

### Alerting

When a pipeline fails, **someone needs to know**. Channels:

- Slack / Teams / Discord webhook
- Email (less ideal - easy to ignore)
- PagerDuty / Opsgenie (for critical pipelines)

Alert on:
- Pipeline failure (any step errored)
- Pipeline latency above threshold
- Quality test failure (dbt test red)
- Data freshness violated (last update > N hours ago)
- Anomaly (row count outside expected range)

Don't alert on every step every run - that's noise. Alert on **violations of an SLA**.

### Observability

Modern data observability tools (Monte Carlo, Datafold, Bigeye for commercial; Elementary, re_data for open) automate:

- Lineage (which table feeds what)
- Freshness monitoring (last update time)
- Schema drift detection
- Volume anomaly detection

For a small team: **Elementary** (open source, dbt-native) gets you 80% of these for free.

---

## Part 7: CDC and incremental loads

For tables that grow, doing a full snapshot every run is wasteful. **Incremental loads** process only the new rows.

In dbt:

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='incremental', unique_key='order_id') }}

SELECT * FROM {{ source('raw', 'orders') }}

{% if is_incremental() %}
  WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

On first run: full load. On subsequent runs: only new rows. dbt handles the deduplication via `unique_key`.

For sources without a clean watermark (`updated_at`), you need real CDC:
- **Debezium** (Postgres / MySQL CDC → Kafka)
- **PeerDB** (Postgres → analytics warehouse direct)
- **Singer / Airbyte / Fivetran connectors**

For most teams the incremental-with-watermark pattern handles 80% of cases.

---

## Part 8: Backfills and reprocessing

Inevitable: a bug shipped a week ago, and you need to reprocess. The pipeline must support:

- **Backfill a date range**: re-run for 2024-01-01 to 2024-01-15
- **Reset and re-run**: drop downstream tables; reproduce from scratch
- **Idempotent runs**: re-running the same date twice produces identical output

The medallion architecture supports this naturally: re-run silver from bronze (immutable); re-run gold from silver. Bronze never re-derives - it's the immutable record.

In dbt:

```bash
# Reprocess a specific model
dbt run --select fct_revenue_daily --full-refresh

# Reprocess everything downstream of a source
dbt run --select source:raw+ --full-refresh
```

In Dagster:

```bash
dagster asset materialize --select revenue_daily --partition 2024-01-15
```

---

## Part 9: Cost-aware production

Pipelines cost money. The dominant cost drivers:

| Component | Cost lever |
|---|---|
| Warehouse compute | Query optimization; partition pruning; right-sized warehouses |
| Storage | Hot vs warm vs cold tiering |
| Ingestion vendor | Per-row pricing (Fivetran) vs flat (self-hosted Airbyte) |
| Orchestrator hosting | Self-hosted vs managed |
| Dashboards | Per-user pricing (Looker) vs flat (Metabase) |

For a small team: $300-1000/month for the whole stack. For a mid-sized team: $5-20k/month. For an enterprise: $100k+/month.

The biggest accidental costs:

- **Unused dashboards refreshing nightly**
- **Tests that scan billions of rows** for trivial checks
- **Full re-snapshots when incremental would do**

A monthly cost audit catches these.

---

## Part 10: Documentation that survives

Three layers:

| Layer | Tool |
|---|---|
| **Code-level**: what does this dbt model compute? | dbt YAML descriptions + `dbt docs` |
| **Lineage**: which downstream models break if this changes? | dbt docs, DataHub, OpenMetadata |
| **Business**: what does "monthly active user" mean? | A data dictionary, owned by a person |

The business layer is the hardest. It requires a data steward role and a willingness to litigate definitions. Tools help; org buy-in determines whether it works.

---

## Part 11: The deployment patterns

### Local-first

DuckDB + dbt + Dagster running on a laptop or VPS. Free. Scales to ~100GB. Perfect for prototypes and small teams.

### Single-cloud

Snowflake/BigQuery + dbt Cloud + Looker. ~$5k/month entry. The "obvious" stack for ages 1-50 of a startup.

### Multi-cloud

Plus Spark/Databricks for huge data. ~$30k+/month. For when scale really matters.

For this curriculum: **local-first** in the capstone. The same patterns apply at every scale.

---

## Part 12: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| Cron job with no monitoring | Failures unnoticed; data quietly stale |
| Mixed Python + SQL transformation in one script | Hard to test; SQL should be dbt |
| No tests on data | Bad data propagates downstream |
| One giant dbt model | Untestable; hard to debug |
| Direct dashboard queries against bronze | Performance + schema risk |
| Re-doing full snapshot when incremental would do | Cost + time |
| No backfill story | Bug = chaos to recover |
| Cargo-cult Snowflake when DuckDB would do | Premature scale |

---

## Part 13: The capstone

You'll build a complete pipeline:

```
┌────────────────────────────────────────┐
│  Ingest (Python): API → bronze Parquet  │
└────────────────────┬───────────────────┘
                     ▼
┌────────────────────────────────────────┐
│  dbt staging (bronze → silver):         │
│    types, validation, quarantine        │
└────────────────────┬───────────────────┘
                     ▼
┌────────────────────────────────────────┐
│  dbt marts (silver → gold):             │
│    business entities, aggregates        │
└────────────────────┬───────────────────┘
                     ▼
┌────────────────────────────────────────┐
│  Dashboard (Metabase or Streamlit)      │
└────────────────────────────────────────┘

Orchestration: Dagster, runs daily at 02:00 UTC.
Observability: dbt test + Dagster failure notifications.
Backfill: supported via dbt --full-refresh and Dagster partitions.
```

Capstone deliverable: a working pipeline on a schedule, with a live dashboard, a `CAPSTONE.md` write-up, and code in a GitHub repo. The artifact that proves you can ship.

---

## Part 14: Connect to the whole curriculum

Every prior week shows up here:

- Week 01 (SQL): every dbt model is SQL
- Week 02 (pandas/Polars): the Python ingestion bits
- Week 03 (ingestion): bronze layer ingestion
- Week 04 (quality): dbt tests
- Weeks 05-08 (cleaning): silver layer transformations
- Weeks 09-12 (enrichment): joins, external data, features, embeddings
- Weeks 13-14 (insights): the analytical output
- Week 15 (dashboards): the consumer layer

The capstone is the final exam where you wire it all together.

---

## What's next

In [lab.md](lab.md) you'll build the capstone pipeline:

- Set up the dbt + Dagster + DuckDB stack
- Build a real bronze → silver → gold flow
- Add tests and freshness checks
- Schedule it to run daily
- Wire Metabase to gold
- Write `CAPSTONE.md` documenting it all
- Push to GitHub as your portfolio piece

By end of week 16 you have a running production pipeline you'd point a hiring manager at.
