# Week 13: EDA + Statistical Foundations

## 🎯 What you'll learn

**Exploratory Data Analysis done seriously.** Not just `df.describe()` and a histogram. The mental moves analysts make to actually understand a dataset: visual checks for distribution shape, missingness patterns, leakage, anomalies; bootstrap and permutation-test approaches; honest hypothesis tests.

By the end of this week you'll be able to:

- Produce a **first-pass EDA notebook** for any dataset in under an hour
- Read distributions visually - recognize log-normal, bimodal, heavy-tailed
- Use **bootstrap** for confidence intervals when parametric assumptions fail
- Use **permutation tests** instead of t-tests when the assumptions don't hold
- Recognize and avoid the standard sins: p-hacking, multiple comparisons, Simpson's paradox
- Use **DuckDB + Polars + matplotlib** for fast, scriptable EDA - no slow notebooks

## 🧰 Lab setup

```bash
uv add matplotlib seaborn statsmodels scipy
```

## ✅ Your job

1. Read [theory.md](theory.md). The "common statistical sins" section is the one to take to interviews.
2. Work through [lab.md](lab.md). Run an EDA pass on a real dataset; produce a 1-page summary.
3. Replicate one statistical claim from a news article and verify or refute it.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Tukey - The Future of Data Analysis (1962)](https://projecteuclid.org/journals/annals-of-mathematical-statistics/volume-33/issue-1/The-Future-of-Data-Analysis/10.1214/aoms/1177704711.full) | The paper that named EDA | 45 min |
| [Efron - Bootstrap Methods (1979)](https://projecteuclid.org/journals/annals-of-statistics/volume-7/issue-1/Bootstrap-Methods-Another-Look-at-the-Jackknife/10.1214/aos/1176344552.full) | The paper that made stats accessible | 45 min |
| [Allen Downey - Think Stats](https://greenteapress.com/thinkstats2/) | Free book, very practical | 3 hours skim |
| [Ioannidis - Why Most Published Research Findings Are False](https://journals.plos.org/plosmedicine/article?id=10.1371/journal.pmed.0020124) | The healthy skepticism dose | 30 min |

## 💡 What you should already know

- Weeks 01-12
- Optional: [ml-mentorship Week 02 (probability + stats)](https://github.com/ichdamola/ml-mentorship/tree/main/week-02-probability-stats)

---

**Next**: [Week 14: Causal Inference + A/B Testing →](../week-14-causal-inference-ab-testing/readme.md)
