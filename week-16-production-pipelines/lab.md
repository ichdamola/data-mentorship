# Week 16: Lab — Build the Capstone Pipeline

The final lab. You'll build a complete bronze → silver → gold pipeline with dbt + Dagster + DuckDB, schedule it, wire a dashboard, and ship a `CAPSTONE.md` documenting it all. This is your portfolio piece.

## Setup

```bash
uv add dagster dagster-webserver dagster-dbt dbt-core dbt-duckdb polars httpx tenacity duckdb
```

Project layout:

```
data-mentorship-capstone/
├── pyproject.toml
├── README.md
├── CAPSTONE.md            ← your deliverable
├── data/
│   ├── bronze/             ← raw, sacred
│   ├── silver/             ← typed, validated
│   └── warehouse.duckdb
├── ingestion/
│   └── nyc_open_data.py    ← Python ingest from week 03
├── dbt/
│   ├── dbt_project.yml
│   ├── profiles.yml
│   └── models/
│       ├── staging/
│       │   ├── stg_trees.sql
│       │   └── sources.yml
│       └── marts/
│           ├── fct_trees_by_borough.sql
│           └── dim_species.sql
└── orchestrator/
    └── definitions.py      ← Dagster pipeline
```

Start by initializing the project:

```bash
mkdir data-mentorship-capstone && cd data-mentorship-capstone
uv init
```

---

## Exercise 16.1 — Bronze ingest (Python)

Reuse the week 03 pattern. Save as `ingestion/nyc_open_data.py`:

```python
import gzip
import json
import os
import time
from pathlib import Path

import httpx
import orjson
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception_type

BASE_URL = "https://data.cityofnewyork.us/resource/uvpi-gqnh.json"
BRONZE_DIR = Path("data/bronze/nyc_trees")
BRONZE_DIR.mkdir(parents=True, exist_ok=True)


class TransientError(Exception):
    pass


@retry(
    retry=retry_if_exception_type((TransientError, httpx.HTTPError)),
    wait=wait_exponential(min=1, max=60),
    stop=stop_after_attempt(8),
    reraise=True,
)
def fetch_page(client, offset, limit=1000):
    resp = client.get(BASE_URL, params={"$limit": limit, "$offset": offset, "$order": "tree_id"})
    if resp.status_code == 429:
        time.sleep(int(resp.headers.get("Retry-After", "10")))
        raise TransientError("rate limited")
    if 500 <= resp.status_code < 600:
        raise TransientError(f"server {resp.status_code}")
    resp.raise_for_status()
    return resp.json()


def ingest_bronze(max_rows: int = 10_000) -> int:
    """Pulls a window of trees data. Returns rows fetched."""
    with httpx.Client(timeout=30, headers={"User-Agent": "data-mentorship-lab/1.0"}) as client:
        rows_written = 0
        offset = 0
        while rows_written < max_rows:
            page = fetch_page(client, offset)
            if not page:
                break
            path = BRONZE_DIR / f"page_{offset:07d}.jsonl.gz"
            with gzip.open(path, "wb") as f:
                for row in page:
                    f.write(orjson.dumps(row) + b"\n")
            rows_written += len(page)
            offset += len(page)
            time.sleep(0.1)
        return rows_written


if __name__ == "__main__":
    n = ingest_bronze()
    print(f"ingested {n} rows to {BRONZE_DIR}")
```

Run it once:

```bash
uv run python ingestion/nyc_open_data.py
```

You should see `data/bronze/nyc_trees/page_*.jsonl.gz` files.

---

## Exercise 16.2 — Bronze → DuckDB

Build a small Python step that loads the bronze JSONLs into a DuckDB table.

```python
# ingestion/bronze_to_duckdb.py
import duckdb
import gzip
import orjson
import polars as pl
from pathlib import Path

BRONZE_DIR = Path("data/bronze/nyc_trees")

def load_to_duckdb(db_path: str = "data/warehouse.duckdb"):
    con = duckdb.connect(db_path)

    rows = []
    for f in sorted(BRONZE_DIR.glob("*.jsonl.gz")):
        with gzip.open(f, "rb") as h:
            for line in h:
                rows.append(orjson.loads(line))

    if not rows:
        print("no bronze rows found")
        return

    df = pl.DataFrame(rows)
    con.execute("CREATE SCHEMA IF NOT EXISTS bronze")
    con.execute("DROP TABLE IF EXISTS bronze.nyc_trees_raw")
    con.execute("CREATE TABLE bronze.nyc_trees_raw AS SELECT * FROM df")

    count = con.execute("SELECT COUNT(*) FROM bronze.nyc_trees_raw").fetchone()[0]
    print(f"loaded {count} rows into bronze.nyc_trees_raw")
    con.close()


if __name__ == "__main__":
    load_to_duckdb()
```

```bash
uv run python ingestion/bronze_to_duckdb.py
```

---

## Exercise 16.3 — dbt project

Create the dbt project:

```bash
mkdir -p dbt/models/staging dbt/models/marts
```

`dbt/dbt_project.yml`:

```yaml
name: data_mentorship_capstone
version: 1.0.0
config-version: 2

profile: capstone

model-paths: ["models"]

models:
  data_mentorship_capstone:
    staging:
      materialized: view
      schema: silver
    marts:
      materialized: table
      schema: gold
```

`dbt/profiles.yml`:

```yaml
capstone:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: ../data/warehouse.duckdb
      threads: 4
```

Source declaration `dbt/models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: bronze
    schema: bronze
    tables:
      - name: nyc_trees_raw
        description: "Raw NYC street tree data, landed daily."
```

Staging model `dbt/models/staging/stg_trees.sql`:

```sql
SELECT
    CAST(tree_id AS BIGINT) AS tree_id,
    spc_common,
    spc_latin,
    boroname AS borough,
    nta_name AS neighborhood,
    status,
    health,
    CAST(tree_dbh AS INTEGER) AS tree_dbh_inches,
    CAST(latitude AS DOUBLE) AS latitude,
    CAST(longitude AS DOUBLE) AS longitude,
    CAST(created_at AS TIMESTAMP) AS observed_at
FROM {{ source('bronze', 'nyc_trees_raw') }}
WHERE tree_id IS NOT NULL
  AND boroname IS NOT NULL
```

Schema YAML with tests `dbt/models/staging/schema.yml`:

```yaml
version: 2

models:
  - name: stg_trees
    description: "Typed and basic-cleaned tree census."
    columns:
      - name: tree_id
        description: "Unique tree identifier"
        tests:
          - not_null
          - unique
      - name: borough
        description: "NYC borough"
        tests:
          - not_null
          - accepted_values:
              values: [Manhattan, Bronx, Brooklyn, Queens, Staten Island]
      - name: status
        tests:
          - accepted_values:
              values: [Alive, Stump, Dead]
```

Mart `dbt/models/marts/fct_trees_by_borough.sql`:

```sql
SELECT
    borough,
    COUNT(*) AS total_trees,
    COUNT(*) FILTER (WHERE status = 'Alive') AS alive_trees,
    COUNT(*) FILTER (WHERE status = 'Dead') AS dead_trees,
    ROUND(COUNT(*) FILTER (WHERE health = 'Good') * 100.0 / NULLIF(COUNT(*) FILTER (WHERE status = 'Alive'), 0), 1) AS pct_good_health,
    ROUND(AVG(tree_dbh_inches) FILTER (WHERE status = 'Alive'), 1) AS avg_dbh_inches,
FROM {{ ref('stg_trees') }}
GROUP BY borough
ORDER BY total_trees DESC
```

Run it:

```bash
cd dbt
dbt run --profiles-dir .
dbt test --profiles-dir .
dbt docs generate --profiles-dir .
dbt docs serve --profiles-dir .   # opens browser to lineage docs
```

You should see:
- `silver.stg_trees` view created
- `gold.fct_trees_by_borough` table created
- All tests pass
- Documentation at http://localhost:8080

---

## Exercise 16.4 — Dagster orchestrator

Create `orchestrator/definitions.py`:

```python
from dagster import asset, Definitions, ScheduleDefinition, define_asset_job, AssetExecutionContext
from dagster_dbt import DbtCliResource, dbt_assets
from pathlib import Path

DBT_PROJECT_DIR = Path(__file__).parent.parent / "dbt"

# Python ingestion as a Dagster asset
@asset
def bronze_nyc_trees(context: AssetExecutionContext):
    """Pull NYC trees from API to bronze JSONL."""
    from ingestion.nyc_open_data import ingest_bronze
    n = ingest_bronze(max_rows=5000)
    context.log.info(f"ingested {n} rows")
    return n


@asset(deps=[bronze_nyc_trees])
def bronze_loaded_to_duckdb(context: AssetExecutionContext):
    """Load bronze JSONL files into DuckDB."""
    from ingestion.bronze_to_duckdb import load_to_duckdb
    load_to_duckdb()
    return True


# dbt models as Dagster assets
@dbt_assets(manifest=DBT_PROJECT_DIR / "target" / "manifest.json")
def dbt_models(context: AssetExecutionContext, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()


# Schedule: run daily at 02:00 UTC
daily_job = define_asset_job(name="daily_pipeline", selection="*")
daily_schedule = ScheduleDefinition(job=daily_job, cron_schedule="0 2 * * *")


defs = Definitions(
    assets=[bronze_nyc_trees, bronze_loaded_to_duckdb, dbt_models],
    schedules=[daily_schedule],
    resources={
        "dbt": DbtCliResource(project_dir=str(DBT_PROJECT_DIR), profiles_dir=str(DBT_PROJECT_DIR)),
    },
)
```

Generate the dbt manifest first:

```bash
cd dbt
dbt parse --profiles-dir .
cd ..
```

Run Dagster:

```bash
dagster dev -f orchestrator/definitions.py
```

Open http://localhost:3000. You should see the asset graph:

```
bronze_nyc_trees → bronze_loaded_to_duckdb → stg_trees → fct_trees_by_borough
                                          → dim_species
```

Click "Materialize all" — the whole pipeline runs end-to-end. Each asset shows green when complete.

---

## Exercise 16.5 — Wire Metabase to the warehouse

```bash
docker run -d -p 3000:3000 -v ${PWD}/data:/data --name capstone-metabase metabase/metabase
```

In Metabase, add a SQLite-compatible database (or use the DuckDB plugin if installed). Point at `/data/warehouse.duckdb`.

Build a dashboard with:

1. **Headline**: total trees count, pct alive
2. Line chart: trees per neighborhood
3. Bar chart: `fct_trees_by_borough` aggregates
4. Map: trees by lat/lng (Metabase has a map widget)

Apply the week 15 rules: headline + bullets, Y-axes starting at 0, no pie charts.

---

## Exercise 16.6 — Test and observability

Add freshness checks to your dbt sources:

```yaml
# dbt/models/staging/sources.yml
version: 2
sources:
  - name: bronze
    schema: bronze
    tables:
      - name: nyc_trees_raw
        loaded_at_field: created_at
        freshness:
          warn_after: {count: 25, period: hour}
          error_after: {count: 48, period: hour}
```

Run:

```bash
cd dbt
dbt source freshness --profiles-dir .
```

If the bronze source is over 25 hours old, dbt warns. Over 48 hours, dbt errors. In Dagster, you'd wire that error into a Slack notification.

Add `pip install dbt-elementary` and integrate with Elementary for an HTML report of test results and anomalies. (Stretch.)

---

## Exercise 16.7 — Schedule for real

Two options:

### Option A — Cron on your machine

```bash
# Edit crontab
crontab -e

# Add: every day at 2am
0 2 * * * cd /path/to/data-mentorship-capstone && /path/to/uv run dagster job execute -f orchestrator/definitions.py -j daily_pipeline
```

### Option B — Dagster Cloud

Free tier handles small workloads:

```bash
dagster-cloud config setup
dagster-cloud deployment add-location  ...
```

### Option C — GitHub Actions

```yaml
# .github/workflows/daily.yml
on:
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:

jobs:
  run-pipeline:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install uv && uv sync
      # Persistence: each Actions runner is a fresh ephemeral VM. Without
      # this restore step, `warehouse.duckdb` from yesterday is gone and
      # every "incremental" load actually does a full rebuild.
      - uses: actions/cache@v4
        with:
          path: data/warehouse.duckdb
          key: duckdb-warehouse-${{ github.run_id }}
          restore-keys: |
            duckdb-warehouse-
      - run: uv run python ingestion/nyc_open_data.py
      - run: uv run python ingestion/bronze_to_duckdb.py
      - run: cd dbt && uv run dbt source freshness --profiles-dir .   # fail if sources stale
      - run: cd dbt && uv run dbt build --profiles-dir .
      - run: |
          if [ $? -ne 0 ]; then
            curl -X POST $SLACK_WEBHOOK -d "{\"text\":\"Pipeline failed\"}"
          fi
```

> ⚠️ **Real persistence for DuckDB**: the `actions/cache` step above keeps state across runs but caches are best-effort (GitHub evicts entries over 10 GB / unused). For a durable warehouse, options in increasing order of robustness: (a) sync `warehouse.duckdb` to S3 at end of run + restore at start, (b) move to **Motherduck** (hosted DuckDB; SDK identical), (c) move to **BigQuery / Snowflake / Redshift**. The capstone is fine with `actions/cache` for the assignment; production wants Motherduck or a real warehouse.

For the capstone: any of these is fine. Cron is the simplest; GitHub Actions is free and observable.

---

## Exercise 16.8 — Write CAPSTONE.md

```markdown
# Data Mentorship Capstone — NYC Trees Pipeline

**Live pipeline**: GitHub Actions running daily at 02:00 UTC ([workflow](https://github.com/YOUR/repo/actions))
**Dashboard**: http://your-domain.com/dashboard or screenshots/dashboard.png
**Code**: https://github.com/YOUR/data-mentorship-capstone

## What it does

A production-shape pipeline that:

1. Pulls NYC street tree census data from NYC Open Data via Socrata API
2. Lands bronze JSONL.gz with content-hash caching
3. Transforms via dbt: staging → marts in DuckDB
4. Runs tests on every refresh
5. Serves a Metabase dashboard for stakeholders
6. Notifies Slack on failure

## Architecture

```
┌─────────────────────────────────┐
│  NYC Open Data API              │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│  Python ingestion (tenacity)    │
│  → data/bronze/nyc_trees/       │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│  Bronze loader → DuckDB         │
│  → bronze.nyc_trees_raw         │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│  dbt staging (typing, tests)    │
│  → silver.stg_trees             │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│  dbt marts (aggregations)       │
│  → gold.fct_trees_by_borough    │
└──────────────┬──────────────────┘
               ▼
┌─────────────────────────────────┐
│  Metabase dashboard             │
└─────────────────────────────────┘
```

Orchestration: Dagster (dev) / GitHub Actions (prod).
Schedule: daily at 02:00 UTC.

## Key numbers

- ~683k trees census records (full data)
- Pipeline runs in ~30 seconds end-to-end
- 11 dbt tests pass on every refresh
- 5 charts on the dashboard

## What I'd add for true production

- Real CDC from a Postgres source (not just static API pull)
- Elementary integration for richer test/anomaly reporting
- Backfill plan for the marts (currently full re-snapshot every run)
- Real Slack alerting on failure (now just printed)
- A data catalog (DataHub or OpenMetadata) for column-level lineage and discovery
- Quarantine table for rows that fail silver validation

## What I learned

(Your reflection — three paragraphs on the hardest part, the surprising part, and what you'd do differently.)

## Sources & credit

- NYC Open Data — tree census API
- dbt, Dagster, DuckDB, Metabase — open source tooling
- Data Mentorship curriculum (weeks 01-16)
```

Push to GitHub. **Send the link in your portfolio.**

---

## Exercise 16.9 — Final cleanup

```bash
# Ensure everything works end-to-end from a fresh clone
git clone YOUR/data-mentorship-capstone /tmp/fresh
cd /tmp/fresh
uv sync
uv run python ingestion/nyc_open_data.py
uv run python ingestion/bronze_to_duckdb.py
cd dbt && uv run dbt build --profiles-dir .
```

If this works cleanly, your pipeline is reproducible. **Reproducibility is the bar.**

---

## Submission checklist

- [ ] Bronze ingestion working with cache
- [ ] DuckDB warehouse populated
- [ ] dbt project with 2+ staging models, 2+ mart models
- [ ] All dbt tests passing
- [ ] dbt docs generated and viewable
- [ ] Dagster orchestrator showing the asset graph
- [ ] Metabase dashboard with headline + 5 charts
- [ ] Pipeline scheduled (Cron / Dagster Cloud / GitHub Actions)
- [ ] `CAPSTONE.md` written with architecture diagram, live URL, key numbers
- [ ] Code pushed to public GitHub repo

---

## What you just did

You shipped a complete data pipeline: ingestion + warehousing + transformation + tests + orchestration + dashboard. The artifact that proves the past 16 weeks. **Add the GitHub link to your résumé.**

---

# 🎓 You finished the curriculum.

Sixteen weeks. Raw CSVs to a running production pipeline. **Look at what you've built**:

- SQL fluency at a level most engineers never reach (week 01)
- pandas + Polars dual-track for tabular work (week 02)
- Resilient API ingestion with retries and caching (week 03)
- Data quality validation as code (week 04)
- Entity resolution with splink (week 05)
- Missing-data discipline beyond `dropna` (week 06)
- Schema enforcement with pandera + pyarrow (week 07)
- Text + Unicode + fuzzy matching that actually works (week 08)
- As-of and geographic joins (week 09)
- Geocoding + Census enrichment with rate limits + cache (week 10)
- Feature engineering without leakage (week 11)
- Embeddings + ANN search (week 12)
- EDA + statistics done honestly (week 13)
- A/B testing without the cardinal sins (week 14)
- Dashboards stakeholders actually use (week 15)
- The production capstone (week 16)

That portfolio is exactly what a 2026 data engineer / analytics engineer / senior analyst should be able to do. Most can't. **You're in rare company.**

## Where to go next

Pick one direction for the next 6-12 months and go deep:

| If you want to | Add |
|---|---|
| **Be an analytics engineer** | More dbt; semantic layer (Cube, MetricFlow); LookML if Looker-shop |
| **Be a data engineer** | Streaming (Kafka, Flink); Iceberg/Delta; Spark; cloud certs |
| **Be a data scientist / ML engineer** | Tabular ML; the [ml-mentorship](https://github.com/ichdamola/ml-mentorship) curriculum end-to-end |
| **Be an analyst / decision support** | Causal inference deeper (Pearl, Imbens); experimentation platforms; storytelling |
| **Build LLM apps** | RAG, evaluation harnesses, agent frameworks |
| **Work in privacy / governance** | DSAR tooling, lineage at column level (DataHub/OpenMetadata), policy engines |

## Read these next

- [Sebastian Raschka's blog](https://magazine.sebastianraschka.com/) — practical ML
- [Chip Huyen's blog](https://huyenchip.com/blog/) — ML systems & data in industry
- [Tobias Macey's "Data Engineering Podcast"](https://www.dataengineeringpodcast.com/) — weekly industry trends
- [The Locally Optimistic blog](https://locallyoptimistic.com/) — analytics engineering culture
- [Benn Stancil's Substack](https://benn.substack.com/) — sharp takes on the modern data stack

## A final note

Hand-rolling everything you've hand-rolled this curriculum is **not** what production looks like. In production you'll mostly use libraries — dbt, Dagster, Snowflake, Metabase, the rest. **The point was never to make you implement these forever.** The point was to make you a person who understands what each of them does.

After 16 weeks at this depth, you're hard to surprise. New tools, new vendors, new patterns — they'll fit into your mental model instead of feeling like magic. Keep that bar. Keep reading source. Keep measuring. Keep shipping.

Welcome to data engineering at the inside.

— end of curriculum —
