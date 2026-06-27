# Week 07: Normalization and Schema Enforcement

## 🎯 What you'll learn

Once data is clean *enough*, it has to be **shaped** before downstream consumers can trust it. That means consistent types, normalized representations (dates as ISO 8601, money as decimals not floats, phone numbers in E.164), and **schemas** enforced at every boundary.

By the end of this week you'll be able to:

- Coerce types safely: when to upcast, when to error, when to quarantine
- Normalize the canonical messy fields: dates/times, phone numbers, addresses, currency, units
- Define schemas with **pandera** (for DataFrames), **pyarrow** (for Parquet), **Pydantic** (for records/rows)
- Reason about **schema evolution** - what to do when an upstream producer adds, removes, or renames a column
- Build a "schema firewall" between ingestion and your modeled layer

## 🧰 Lab setup

```bash
uv add pandera pyarrow pydantic phonenumbers babel
```

## ✅ Your job

1. Read [theory.md](theory.md). The "schema evolution" section is the part teams keep underestimating until production breaks.
2. Work through [lab.md](lab.md). Normalize a deliberately messy CSV (with mixed dtypes, half-baked dates, units in three different formats) into a canonical Parquet.
3. Build a pandera schema with at least 10 checks and run it as a CI gate.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [pandera docs](https://pandera.readthedocs.io/) | The tool | 45 min |
| [Pydantic v2 docs](https://docs.pydantic.dev/) | For row-level validation | 30 min |
| [Apache Iceberg - Schema evolution](https://iceberg.apache.org/docs/latest/evolution/) | The reference for how to do it right | 30 min |
| [Kimball - Slowly Changing Dimensions](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/slowly-changing-dimension/) | The classical pattern | 20 min |

## 💡 What you should already know

- Weeks 01-06

---

**Next**: [Week 08: Text Cleaning + Fuzzy Matching →](../week-08-text-cleaning-fuzzy-matching/readme.md)
