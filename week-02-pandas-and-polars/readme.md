# Week 02: pandas and Polars

## 🎯 What you'll learn

The two daily-driver dataframe libraries: **pandas** (the incumbent - everywhere) and **Polars** (the modern alternative - multi-threaded, expression-based, ~10× faster on typical workloads). When each wins, how to think in **expressions** instead of imperative loops, the gotchas that bite everyone in pandas, and how to translate one to the other.

By the end of this week you'll be able to:

- Write idiomatic pandas: `assign`, `pipe`, method chaining, `query`, multi-index where it actually helps
- Write idiomatic Polars: the expression API, lazy mode, `group_by` semantics
- Pick between them: "small data, broad ecosystem" → pandas; "tabular ML feature pipelines, repeated runs" → Polars
- Translate any pandas pipeline to Polars (and vice versa)
- Recognize the four pandas footguns: `SettingWithCopyWarning`, `apply` slowness, mixed dtypes, `inplace` lies

## 🧰 Lab setup

```bash
uv add pandas polars pyarrow matplotlib
```

We'll reuse the NYC taxi parquet from week 01.

## ✅ Your job

1. Read [theory.md](theory.md). Focus on the "expressions vs columns" framing.
2. Work through [lab.md](lab.md). The same analysis is built in both libraries side-by-side.
3. Solve the assigned exercises.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [pandas user guide - Essential basic functionality](https://pandas.pydata.org/docs/user_guide/basics.html) | The reference; skim and bookmark | 45 min |
| [Polars Modern Polars book](https://kevinheavey.github.io/modern-polars/) | The best free Polars-vs-pandas walk-through | 90 min |
| [Tom Augspurger's "Modern Pandas"](https://tomaugspurger.net/posts/modern-1-intro/) (parts 1-3) | Idiomatic pandas the right way | 60 min |

## 💡 What you should already know

- Week 01 (SQL - many pandas patterns mirror SQL clauses)
- Python: dicts, lists, list comprehensions

---

**Next**: [Week 03: Ingestion →](../week-03-ingestion/readme.md)
