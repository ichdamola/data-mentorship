# Week 01: SQL Deep — the patterns nobody teaches

## 🎯 What you'll learn

The SQL most engineers stop at: `SELECT ... FROM ... WHERE ... JOIN ... GROUP BY`. That's table stakes. This week pushes into the operators that turn SQL from "query a table" into "shape any analysis": **window functions, CTEs, lateral joins, recursive queries, qualify, semi/anti joins, pivots**.

By the end of this week you'll be able to:

- Pick the right window function (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG/LEAD`, `SUM() OVER`, running totals) for any analytics question
- Write CTEs that read top-to-bottom like a story, not a riddle
- Use `QUALIFY` to filter on window results without subqueries
- Do gaps-and-islands, sessionization, and cohort retention with no Python in sight
- Replace 200-line pandas pipelines with 20-line SQL
- Recognize when SQL stops paying — and when it's still the right hammer

## ⚠️ Why DuckDB

DuckDB is Postgres-flavored SQL with one quality-of-life win after another (`QUALIFY`, `EXCLUDE`, `*COLUMNS()`, `GROUPING SETS`, native Parquet, Arrow zero-copy). Runs as a Python library, a CLI, or embedded in your editor. Free. Fast. The right teaching tool for 2026.

Everything you'll write in this week is portable to Postgres, Snowflake, BigQuery, and Databricks with minor dialect tweaks. Where DuckDB-specific shortcuts exist, the readme notes them.

## 🧰 Lab setup

```bash
uv init data-mentorship-work && cd data-mentorship-work
uv add duckdb pandas pyarrow
uv run python -c "import duckdb; print(duckdb.__version__)"
```

We'll use the NYC Yellow Taxi trip data — one of the canonical clean-and-realistic datasets for analytics work.

```bash
mkdir -p data && cd data
curl -O https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet
ls -lh yellow_tripdata_2024-01.parquet
# ~50 MB; 3M rows
```

If that URL changes (NYC TLC publishes new files monthly), grab any month from [the TLC site](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page).

## ✅ Your job

1. Read [theory.md](theory.md). Spend extra time on Part 4 (window functions). It's the single highest-leverage SQL skill.
2. Work through [lab.md](lab.md). Every query should run cold against the taxi data.
3. The stretch exercise (lab.md, Exercise 1.10) builds cohort retention with one SQL statement. Senior interviewers love asking this.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Mode SQL Window Functions](https://mode.com/sql-tutorial/sql-window-functions/) | The clearest free tutorial | 60 min |
| [DuckDB SQL reference (SELECT)](https://duckdb.org/docs/sql/statements/select) | The whole language on one page | 45 min skim |
| [Markus Winand — Modern SQL](https://modern-sql.com/) | The "what's actually in the standard since 2003" companion | 60 min skim |
| [Joe Celko's SQL puzzles](https://www.amazon.com/Joe-Celkos-Puzzles-Answers-Kaufmann/dp/0123735963) (selected) | The classic problem-solving collection — optional but recommended | as much as you want |

## 💡 What you should already know

- Basic SQL: `SELECT`, `WHERE`, `JOIN`, `GROUP BY`, `ORDER BY`
- The difference between `WHERE` and `HAVING`
- That a query has a *logical* execution order (FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT) different from its written order

If any of that is fuzzy, read [Mode's SQL basics](https://mode.com/sql-tutorial/) first.

---

**Next**: [Week 02: pandas + Polars →](../week-02-pandas-and-polars/readme.md)
