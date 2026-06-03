# Week 06: Missing Data and Outliers

## 🎯 What you'll learn

How to handle **missing values** without quietly biasing your analysis, and how to handle **outliers** without quietly destroying your signal. Most courses say "drop nulls; clip the top 1%." Both can be terrible defaults.

By the end of this week you'll be able to:

- Diagnose missingness as **MCAR, MAR, or MNAR** — and explain why it matters
- Choose between drop, fill, model-based imputation, and "encode-as-feature"
- Use sklearn's `IterativeImputer` (MICE-style) and know when it's worth the complexity
- Detect outliers with multiple methods (IQR, z-score, isolation forest, DBSCAN) and reason about which is right for the distribution
- **Leave outliers alone** when they're the most important rows in the table
- Document every imputation/clip decision so downstream consumers don't get burned

## 🧰 Lab setup

```bash
uv add scikit-learn missingno pyod
```

## ✅ Your job

1. Read [theory.md](theory.md). The MCAR/MAR/MNAR section is the single most under-taught idea in tabular data work.
2. Work through [lab.md](lab.md). Apply 4 imputation strategies to the same dataset and compare downstream impact.
3. Build a "missingness report" you'd attach to any dataset you ship.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Rubin (1976) — Inference and missing data](https://academic.oup.com/biomet/article-abstract/63/3/581/270932) | The MCAR/MAR/MNAR paper | 60 min |
| [scikit-learn imputation guide](https://scikit-learn.org/stable/modules/impute.html) | Practical | 30 min |
| [van Buuren — Flexible Imputation of Missing Data](https://stefvanbuuren.name/fimd/) | The free textbook | 2 hours skim |
| [Aggarwal — Outlier Analysis (ch. 1-3)](https://link.springer.com/book/10.1007/978-3-319-47578-3) | The reference | 90 min skim |

## 💡 What you should already know

- Weeks 01-05
- Probability basics from [ml-mentorship week 02](https://github.com/ichdamola/ml-mentorship/tree/main/week-02-probability-stats) if you've taken that

---

> 🚧 **Scaffolded.** Theory + lab fully fleshed in the next pass.

**Next**: [Week 07: Normalization + Schema Enforcement →](../week-07-normalization-schema-enforcement/readme.md)
