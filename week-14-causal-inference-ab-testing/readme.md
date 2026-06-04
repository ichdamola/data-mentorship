# Week 14: Causal Inference + A/B Testing

## 🎯 What you'll learn

**Correlation is not causation.** "Users who used feature X had 20% higher retention" — was it the feature, or the kind of user who chose it? This week is the toolkit for answering "what caused what" — both via randomized experiments (the easy mode) and via observational adjustment (the hard but common case).

By the end of this week you'll be able to:

- Design a clean A/B test: sample size, MDE, power, randomization unit
- Read an A/B test result honestly: confidence intervals, p-values, the danger of peeking
- Apply **propensity score matching**, **inverse-propensity weighting**, **difference-in-differences** when you can't randomize
- Recognize **confounders, mediators, and colliders** — and why mishandling each breaks your analysis
- Build a Bayesian A/B test variant (for small samples where p-values mislead)
- Avoid the canonical mistakes: SRM (sample ratio mismatch), Simpson's paradox, novelty/primacy effects, network spillover

## 🧰 Lab setup

```bash
uv add statsmodels econml dowhy scipy
```

## ✅ Your job

1. Read [theory.md](theory.md). The "confounder vs mediator vs collider" section is the single highest-leverage page.
2. Work through [lab.md](lab.md). Run a real A/B analysis on synthetic data; then try the same with observational adjustment.
3. Pick an observational claim from a recent paper or blog and re-analyze it with adjustment.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Kohavi, Tang, Xu — Trustworthy Online Controlled Experiments](https://www.cambridge.org/core/books/trustworthy-online-controlled-experiments/D97B26382EB0EB2DC2019A7A7B518F59) | The A/B testing reference | 2 hours skim |
| [Pearl — The Book of Why (selected chapters)](https://www.basicbooks.com/titles/judea-pearl/the-book-of-why/9780465097616/) | Causal intuition | 90 min |
| [Stitch Fix engineering blog on causal ML](https://multithreaded.stitchfix.com/algorithms/blog/) | Practical applications | 60 min |
| [Microsoft EconML docs](https://econml.azurewebsites.net/) | The library | 30 min |

## 💡 What you should already know

- Weeks 01-13

---

**Next**: [Week 15: Dashboards + Storytelling →](../week-15-dashboards-and-storytelling/readme.md)
