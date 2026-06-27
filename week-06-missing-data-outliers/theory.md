# Week 06: Theory - Missing Data and Outliers

Two operations every analyst defaults to and most do badly:

- **Missing values:** "drop NA" by reflex, occasionally biasing the entire analysis.
- **Outliers:** "clip the top 1%" by habit, occasionally deleting the only rows that matter.

This week is the careful version. After it, the question stops being *"how do I get rid of this?"* and starts being *"what is this telling me, and what's the cost of each response?"*

---

## Part 1: Why missing data isn't random

The first move when you see nulls: ask **why are they null?**

Rubin's 1976 framework gives three categories. Memorize them.

| Type | Meaning | Example | Safe to drop? |
|---|---|---|---|
| **MCAR** - Missing Completely At Random | Missingness is unrelated to *any* variable (observed or unobserved) | Sensor failed randomly | Yes, but inefficient |
| **MAR** - Missing At Random | Missingness depends only on *observed* variables | Older respondents skip income question more often, and age is recorded | Yes, with proper imputation |
| **MNAR** - Missing Not At Random | Missingness depends on the *unobserved* value itself | High earners skip the income question | **No.** Dropping or imputing biases the analysis. |

### The 60-second diagnosis

For each column with nulls:

1. **What rate?** 1%, 10%, 80%? (Drives the strategy.)
2. **Is there a pattern by row?** All nulls concentrated in one source / time / segment?
3. **Is the missingness predictable from other columns?**
4. **Could the *value itself* be the cause?**

If question 4 has any answer except "no, definitely not," you're in **MNAR** territory and you should be careful.

The plot to make: a **missingness heatmap** - rows × columns, black where missing. Patterns jump out instantly. The library is **missingno**:

```python
import missingno as msno
msno.matrix(df)         # binary pattern
msno.heatmap(df)        # correlation of missingness between columns
msno.dendrogram(df)     # which columns go missing together
```

If two columns are always missing together, they're probably from the same source - and the missingness is structural, not random.

---

## Part 2: Strategies for missing values

| Strategy | What | When | Cost |
|---|---|---|---|
| **Drop rows (listwise)** | Remove rows with any null in selected columns | MCAR + few rows affected | Loses data; biases if MAR/MNAR |
| **Drop columns** | Remove columns above some null % threshold | Column is mostly null and not critical | Loses signal if remaining nulls were informative |
| **Constant fill** | Replace with 0 / 'unknown' / a sentinel | The "missingness is a category" case | Distorts numerical aggregates |
| **Mean / median / mode fill** | Replace with a summary statistic | MCAR; few nulls; no important downstream | Shrinks variance; doesn't propagate uncertainty |
| **Last-observation-carried-forward (LOCF)** | Time series; use previous value | Sensor blips; status that rarely changes | Hides slow drifts |
| **Model-based imputation (KNN, IterativeImputer, MICE)** | Predict from other columns | MAR with structure | Computationally heavier; can leak |
| **Encode-as-feature** | Add a `_was_null` boolean column; fill the original with any value | Whenever nullness might be informative | Doubles columns; usually worth it |
| **Multiple imputation** | Create N imputed versions; analyze each; pool results | Statistical inference; uncertainty matters | More complex; correct answer for many use cases |

### The senior moves

1. **Always create an `_was_null` indicator** when you fill. The fact of nullness is often itself a feature.
2. **Fit imputers on the train set only.** Imputing using statistics from the full dataset (train + val + test) is a form of leakage.
3. **Document the imputation strategy.** It's a modeling choice; downstream consumers need to know.

```python
from sklearn.impute import SimpleImputer, IterativeImputer

# IterativeImputer = sklearn's MICE-like
imp = IterativeImputer(max_iter=10, random_state=42)
X_train_imputed = imp.fit_transform(X_train)
X_test_imputed = imp.transform(X_test)    # uses statistics from train
```

### When to ignore the framework and just leave nulls

Modern tree-based models (XGBoost, LightGBM, CatBoost) **handle nulls natively** - they learn the best split direction for null values per node. Imputing before XGBoost is often a slight loss of signal.

For these models: leave nulls; let the model decide. For linear models, neural nets, sklearn classics: impute first.

---

## Part 3: Outliers - what an outlier actually is

The lazy definition: "a row whose value is far from the typical range."

The careful definition: **a row that doesn't belong to the distribution the rest of the data is sampled from.**

These two definitions disagree exactly when it matters most:

- The fraud detection use case: the rare expensive transaction *is* the signal, not noise
- The cancer screening use case: the rare elevated marker *is* the signal
- The user behavior use case: a "power user" with 100× engagement *is* a real customer

**Always ask: am I trying to detect outliers, or to remove them?** If you're removing, you'd better know what you're losing.

---

## Part 4: Detection methods

| Method | How | Strengths | Weaknesses |
|---|---|---|---|
| **IQR fence** | Flag values outside `[Q1 - 1.5·IQR, Q3 + 1.5·IQR]` | Simple, distribution-free, no parameters | Misses multivariate outliers; assumes roughly symmetric data |
| **Z-score** | Flag `\|z\| > 3` | Simple; familiar | Assumes ~normal; sensitive to true distribution shape |
| **Modified Z-score (MAD-based)** | Use median + MAD instead of mean + std | Robust to outliers itself | Same shape assumption (heavy-tail-naive) |
| **Isolation Forest** | Tree-based; outliers separated by few splits | Handles multivariate; scales well | Black-box; needs `contamination` parameter |
| **DBSCAN** | Density clustering; outliers are noise points | Multivariate; no distribution assumption | Slow; sensitive to `eps` / `min_samples` |
| **Local Outlier Factor (LOF)** | Compare local density to neighbors | Catches local outliers in non-uniform data | Computationally heavier |
| **One-class SVM** | Learn boundary around normal data | Flexible | Slow; opaque |

### Univariate vs multivariate

A row with `age=200` is an outlier on age alone - univariate. A row with `age=25, income=$2M, zip=00000` might have no single field outside normal range - but the combination is implausible.

For high-stakes work (fraud, healthcare anomalies), multivariate detection (Isolation Forest, LOF) finds patterns simple z-scores miss.

### The thresholds people use

- IQR: `1.5` for "outliers", `3.0` for "far outliers"
- Z-score: 2, 2.5, 3
- Isolation Forest `contamination`: 0.01 (1% of data is outlier) is typical default

**All of these are tunable.** The right value depends on the cost of false-positive vs false-negative - same as any classification problem.

---

## Part 5: When to leave outliers alone

A non-exhaustive list:

| Situation | Why |
|---|---|
| You're modeling rare events | The outliers ARE the events |
| You're computing min/max for monitoring | Outliers are the alarm |
| Outliers are concentrated in one subgroup | They're a real subgroup, not noise |
| The "outlier" rule was determined by looking at the data | Circular; you'll always find what you went looking for |
| You don't know what generated the value | Removing changes the conclusion silently |

The plot to make: **conditional histogram**. Plot the column's distribution split by some category (e.g., user segment, channel, source system). If "outliers" cluster in one category, they're not noise - they're a real subpopulation.

---

## Part 6: Repair strategies

When you do decide to handle outliers:

| Strategy | When |
|---|---|
| **Drop** | The row is a confirmed error (data entry mistake) |
| **Clip / winsorize** | Numeric column; cap at 1st / 99th percentile | 
| **Log-transform** | Heavy-tailed distribution where you care about ratios more than absolute values |
| **Flag as feature** | Like missing - add `_is_outlier` column; let downstream model decide |
| **Bucket** | Replace with categorical bins (low / mid / high) |

**Winsorize** is the production-friendly default - clip values to the 1st/99th percentile (or 5th/95th for heavier-tailed data). Less destructive than dropping; usually invisible to downstream stats.

```python
import numpy as np
def winsorize(s, lower_pct=0.01, upper_pct=0.99):
    lower = s.quantile(lower_pct)
    upper = s.quantile(upper_pct)
    return s.clip(lower, upper)
```

---

## Part 7: The missingness report

A best practice: ship a **per-load missingness + outlier report** alongside the data.

```
=== Missing data report - orders (load 2026-06-03) ===

Total rows: 1,247,883 (vs 1,251,401 yesterday, -0.3%)

Null counts:
  customer_id        0          (0.00%)   - required
  amount_cents       0          (0.00%)
  currency          83          (0.01%)   - OK, within tolerance (0.1%)
  shipping_zip      29,847      (2.39%)   - UP from 0.5% yesterday  ⚠ INVESTIGATE
  notes             892,184     (71.5%)   - expected, optional field
  
Outlier candidates:
  amount_cents      247 trips flagged (>99.9th percentile = $5,000)
                    breakdown: 230 in 'enterprise' channel, 17 in 'self_serve' (unusual)
```

The point: **changes over time** are the real signal. A column going from 0.5% to 2.4% null overnight is a problem; either way the per-load number is the same.

For the lab you'll build a tiny version of this report.

---

## Part 8: Anti-patterns

Things that look reasonable but aren't:

| Anti-pattern | Why it's bad |
|---|---|
| Drop all rows with any null | Loses 70%+ of data on wide tables |
| Fill nulls in test set using train-set mean computed before split | Leakage if there's any pattern |
| Always clip outliers to 99th percentile | Hides actual extreme values that matter |
| Run outlier detection on output of imputation | The imputed values are smooth; you'll always find the originals are "outliers" |
| Use z-score on data you haven't checked is normal | Z-score assumptions are violated; thresholds are meaningless |

The senior version of all of these: **separate detection from action**. Always flag first; decide what to do based on downstream cost.

---

## Part 9: Where this connects to the rest of the curriculum

- **Week 04 (data quality)**: Imputation strategy decisions become part of the data contract. "We impute nulls in `shipping_zip` with the customer's billing zip" is a documented behavior.
- **Week 11 (feature engineering)**: Missingness indicators become features. Outlier flags become features.
- **Week 13 (EDA)**: You'll re-encounter outlier detection as a *signal-finding* technique, not a *removal* technique.
- **Week 14 (causal / A/B)**: Outliers in conversion rates / revenue per user are the whole game - you absolutely don't want to clip them.

The thread: **missing values and outliers are data; treat them with the same care as any other data.**

---

## What's next

In [lab.md](lab.md) you'll:

- Generate a dataset with deliberate MCAR, MAR, and MNAR missingness; build the heatmap that distinguishes them
- Try 4 imputation strategies; compare downstream impact
- Detect outliers with IQR, z-score, isolation forest, LOF; compare what each catches
- Build the missingness-and-outlier report you'd attach to every load
- Run a tiny experiment: how much does aggressive imputation bias a downstream regression?

By end of week 06 you'll never reflexively `dropna()` again.
