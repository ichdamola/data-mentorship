# Week 11: Feature Engineering

## 🎯 What you'll learn

For tabular ML, **feature engineering still beats fancy models**. XGBoost on well-crafted features routinely outperforms a neural network on raw inputs. This week covers the high-leverage patterns: categorical encoding, target encoding (without leakage), time-based features, aggregations, and the discipline that prevents production-time surprises.

By the end of this week you'll be able to:

- Encode categoricals correctly: one-hot, ordinal, target, hashing, embeddings (the last as preview for week 12)
- Build **time-window features** ("user's avg purchase in last 30 days") without time-leakage
- Build **aggregation features** at multiple grains (customer / customer-month / customer-category)
- Detect and prevent **data leakage** — the bug that makes your val accuracy 99% in dev and 60% in prod
- Use sklearn `Pipeline` and feature-store patterns for reproducibility
- Recognize when to stop engineering and let the model handle it

## 🧰 Lab setup

```bash
uv add scikit-learn category-encoders xgboost lightgbm
```

## ✅ Your job

1. Read [theory.md](theory.md). The leakage section is non-negotiable.
2. Work through [lab.md](lab.md). Engineer features on a synthetic e-commerce dataset; train XGBoost; compare to a baseline with no engineering.
3. Build at least one feature that introduces leakage on purpose; watch it inflate val accuracy then crash in production.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Kaggle — Feature Engineering Bookcamp](https://www.kaggle.com/learn/feature-engineering) | The fast track | 90 min |
| [scikit-learn Pipelines](https://scikit-learn.org/stable/modules/compose.html) | The right way to ship features | 45 min |
| [Chip Huyen — Designing ML Systems (ch. 5)](https://huyenchip.com/books/) | The leakage chapter | 60 min |
| [Tecton — point-in-time correctness](https://www.tecton.ai/blog/time-travel-in-ml-the-key-to-avoiding-data-leakage/) | The feature-store framing | 30 min |

## 💡 What you should already know

- Weeks 01-10
- Optional: [ml-mentorship Week 04](https://github.com/ichdamola/ml-mentorship/tree/main/week-04-classical-ml) if you've taken it

---

**Next**: [Week 12: Vector Embeddings + Semantic Enrichment →](../week-12-vector-embeddings-semantic/readme.md)
