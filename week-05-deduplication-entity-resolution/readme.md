# Week 05: Deduplication and Entity Resolution

## 🎯 What you'll learn

The same person registers as "John Smith" and "j. smith" and "John A. Smith" with three different phone numbers. The same product shows up in your catalog with 14 variations. **Entity resolution** - figuring out which records refer to the same real-world thing - is one of the dirtiest, most consequential parts of data work.

By the end of this week you'll be able to:

- Do exact, deterministic deduplication in SQL (the easy 60%)
- Do **fuzzy deduplication** with rapidfuzz + sensible blocking
- Use **splink** - the modern open-source record linkage tool - at scale
- Choose a **survivorship** strategy: which record wins, which fields are merged
- Build a **golden record** the rest of the org can trust
- Measure your dedup pipeline's precision and recall against a labeled sample

## 🧰 Lab setup

```bash
uv add splink rapidfuzz duckdb pandas polars
```

## ✅ Your job

1. Read [theory.md](theory.md). The "blocking" section is the secret to making fuzzy matching tractable at scale.
2. Work through [lab.md](lab.md). Dedup Open Food Facts product names (real, messy, ~2M rows).
3. Build a labeled sample of 200 pairs; measure your pipeline's precision and recall.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Splink documentation - Tutorial](https://moj-analytical-services.github.io/splink/) | The tool + the concepts taught together | 60 min |
| [Peter Christen - Data Matching (book, ch. 1-3)](https://link.springer.com/book/10.1007/978-3-642-31164-2) | Academic reference | 90 min |
| [Fellegi & Sunter (1969)](https://courses.cs.washington.edu/courses/cse590q/04au/papers/Felligi69.pdf) | The original probabilistic record linkage paper | 60 min skim |

## 💡 What you should already know

- Weeks 01-04
- Some Bayesian intuition (it's the math behind splink)

---

**Next**: [Week 06: Missing Data + Outliers →](../week-06-missing-data-outliers/readme.md)
