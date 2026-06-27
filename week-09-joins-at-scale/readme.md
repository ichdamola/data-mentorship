# Week 09: Joins at Scale

## 🎯 What you'll learn

Joins look easy until both tables are too big to fit in RAM, the join keys don't match exactly, the timestamps are off by half a second, or you're trying to join "anything within 500m of this point." This week is the practical reference.

By the end of this key you'll be able to:

- Reason about **broadcast vs shuffle** joins and why one OOMs while the other doesn't
- Do **temporal** joins (as-of joins, point-in-time correctness for ML training)
- Do **geographic** joins (point-in-polygon, nearest-neighbor with PostGIS / DuckDB Spatial)
- Use **range** and **interval** joins (which customer was on which plan when?)
- Use **semi-joins and anti-joins** instead of subquery tricks
- Diagnose why your join exploded from 10M rows to 100B (Cartesian disaster)

## 🧰 Lab setup

```bash
uv add duckdb polars geopandas shapely
# DuckDB Spatial extension
uv run python -c "import duckdb; duckdb.connect().install_extension('spatial')"
```

## ✅ Your job

1. Read [theory.md](theory.md). The as-of join section is the one ML engineers wish they'd seen sooner.
2. Work through [lab.md](lab.md). Join NYC taxi trips against weather data (temporal) and against neighborhood polygons (geographic).
3. Build a query that detects accidentally-Cartesian joins before they run.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Polars as-of joins](https://docs.pola.rs/user-guide/transformations/joins/) | The modern API | 30 min |
| [DuckDB Spatial extension](https://duckdb.org/docs/extensions/spatial) | The new way to do geo joins locally | 30 min |
| [PostGIS - Geographic information for PostgreSQL](https://postgis.net/workshops/postgis-intro/) | The classical reference | 60 min |
| [Feature stores - Tecton blog on point-in-time correctness](https://www.tecton.ai/blog/time-travel-in-ml-the-key-to-avoiding-data-leakage/) | Why this matters for ML | 30 min |

## 💡 What you should already know

- Weeks 01-08

---

**Next**: [Week 10: External Enrichment →](../week-10-external-enrichment/readme.md)
