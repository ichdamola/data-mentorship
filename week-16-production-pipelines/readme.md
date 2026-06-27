# Week 16: Production Data Pipelines - Capstone

## 🎯 What you'll learn

Stitch everything together into a pipeline that **runs on a schedule, survives failures, alerts when things break, and produces output the org trusts**. The full modern data stack assembled from open-source pieces.

By the end of this week you'll be able to:

- Build the **medallion architecture**: bronze (raw) → silver (cleaned) → gold (modeled)
- Author transformations as **dbt models** with tests and docs
- Orchestrate runs with **Dagster** (newer, declarative) or **Airflow** (incumbent)
- Stand up **CDC ingestion** (Postgres → DuckDB / Iceberg) for near-real-time
- Wire **lineage**: which downstream model breaks if this column is renamed?
- Add **observability**: freshness, schema drift, row count anomalies
- Cost-estimate the deployment

## 🧰 Lab setup

```bash
uv add dagster dagster-webserver dagster-dbt dbt-core dbt-duckdb
```

For the optional Airflow path:

```bash
uv add apache-airflow
```

## ✅ Your job

1. Read [theory.md](theory.md). The "medallion architecture" section is the modern default; the dbt + orchestrator pattern is the production shape.
2. Work through [lab.md](lab.md). Build a Dagster + dbt + DuckDB pipeline end to end.
3. **Capstone deliverable:** a running pipeline on a schedule (your laptop's `cron`, a GitHub Action, or a free Modal/Render instance) that ingests, cleans, enriches, models, and dashboards a real dataset. Push the repo and link to a live dashboard or a screenshot. Write `CAPSTONE.md` with: arch diagram, schedule, lineage, observability dashboard, what you'd add for true production.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [dbt Learn](https://www.getdbt.com/learn) | The transformation framework that ate the world | 2 hours |
| [Dagster docs - Getting started](https://docs.dagster.io/getting-started) | Modern orchestrator with assets-as-first-class | 90 min |
| [Reis + Housley - Fundamentals of Data Engineering](https://www.oreilly.com/library/view/fundamentals-of-data/9781098108298/) | The reference for the modern stack | 3 hours skim |
| [Bauplan / Tobiko on the lakehouse pattern](https://tobikodata.com/blog) | What's coming next | 30 min |

## 💡 What you should already know

- Weeks 01-15

---

**Curriculum complete** - you've gone from raw CSV to a live production pipeline.
