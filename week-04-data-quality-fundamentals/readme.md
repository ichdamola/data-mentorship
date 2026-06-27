# Week 04: Data Quality Fundamentals

## 🎯 What you'll learn

How to *measure* whether your data is good, not just stare at it. Profiling, validation, the contract pattern between data producers and consumers, and the open-source tooling: **Great Expectations**, **Soda**, **pandera**.

By the end of this week you'll be able to:

- Profile a new dataset and produce a "quality report" in one command
- Write **expectations** (this column should be non-null; this is a foreign key to that table; this value should be between X and Y)
- Wire those into a pipeline so bad data fails loudly *before* it pollutes downstream
- Pick between Great Expectations (heavy, full-featured), Soda (lighter, SQL-native), and pandera (pythonic, in-process)
- Negotiate a data contract with an upstream producer without it becoming a slap fight

## 🧰 Lab setup

```bash
uv add pandera great-expectations soda-core ydata-profiling
```

## ✅ Your job

1. Read [theory.md](theory.md). The "data contract" section is the senior framing.
2. Work through [lab.md](lab.md). Profile the NYC taxi data; write 5 expectations; make them fail intentionally and see the report.
3. Compare pandera vs Great Expectations on the same dataset.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Great Expectations - Getting started](https://docs.greatexpectations.io/docs/oss/tutorials/getting_started_tutorial/) | The biggest tool in this space | 45 min |
| [pandera docs](https://pandera.readthedocs.io/) | The pythonic alternative | 30 min |
| [Chad Sanderson - Data Contracts 101](https://dataproducts.substack.com/p/data-contracts-101) | The framing | 30 min |
| [Soda Core docs](https://docs.soda.io/soda-core/overview.html) | SQL-native alternative | 30 min |

## 💡 What you should already know

- Weeks 01-03

---

**Next**: [Week 05: Deduplication + Entity Resolution →](../week-05-deduplication-entity-resolution/readme.md)
