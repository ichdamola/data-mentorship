# Week 03: Ingestion — from anywhere into your stack

## 🎯 What you'll learn

Where data actually comes from in real jobs: **REST APIs, scraping, file dumps, CDC streams, partner SFTP, Kafka topics, webhooks**. And the honest tradeoffs between file formats: **CSV, JSON, JSONL, Parquet, Avro, Arrow**.

By the end of this week you'll be able to:

- Pull from a paginated REST API with retries, rate limiting, and resumability
- Scrape responsibly (robots.txt, rate limiting, the ethical bright lines from [appsec-mentorship](https://github.com/ichdamola/appsec-mentorship))
- Pick the right file format for the job — and explain why CSV is the wrong default
- Read 50 GB of Parquet on a laptop with predicate pushdown
- Set up a tiny CDC ingestion (Postgres logical replication → DuckDB)

## 🧰 Lab setup

```bash
uv add httpx tenacity duckdb pyarrow polars
```

For the optional CDC exercise: Docker + Postgres.

## ✅ Your job

1. Read [theory.md](theory.md) — the file-format trade-off table is the headline content
2. Work through [lab.md](lab.md) — build a real API puller and a Parquet conversion pipeline
3. Convert at least one CSV in your own life to Parquet and measure the size + read-speed wins

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Apache Arrow: A modern columnar standard](https://arrow.apache.org/overview/) | The format underneath everything | 30 min |
| [DuckDB Parquet docs](https://duckdb.org/docs/data/parquet/overview) | How predicate pushdown actually works | 20 min |
| [Wes McKinney — "Apache Arrow and the future of data frames"](https://wesmckinney.com/blog/apache-arrow-pandas-internals/) | History + direction | 30 min |
| [tenacity docs](https://tenacity.readthedocs.io/) | The right Python retry library | 15 min |

## 💡 What you should already know

- Weeks 01-02
- HTTP basics (status codes, headers, JSON)

---

**Next**: [Week 04: Data Quality Fundamentals →](../week-04-data-quality-fundamentals/readme.md)
