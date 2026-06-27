# Week 06: Lab - Missing Data and Outliers

You'll generate a dataset with three flavors of missingness, see why the heatmap matters, try 4 imputation strategies and measure their downstream impact, then run outlier detection 4 ways on the same data.

## Setup

```bash
uv add scikit-learn missingno pyod matplotlib seaborn polars
```

```python
import numpy as np
import polars as pl
import pandas as pd
import matplotlib.pyplot as plt
import missingno as msno
from sklearn.impute import SimpleImputer, KNNImputer
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer
from sklearn.linear_model import LinearRegression
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor

np.random.seed(42)
```

---

## Exercise 6.1 - Generate data with the three missingness types

We'll create a synthetic "survey" dataset where we know the ground truth, deliberately introduce three kinds of missingness, then try to recover.

```python
N = 5000
age = np.random.randint(18, 80, N)
income = 25000 + (age - 18) * 1500 + np.random.normal(0, 10000, N)
income = np.clip(income, 0, None)
satisfaction = np.clip(7 - (income / 30000) + np.random.normal(0, 1, N), 1, 10)
spending = income * 0.4 + np.random.normal(0, 3000, N)

df_truth = pd.DataFrame({
    "age": age,
    "income": income,
    "satisfaction": satisfaction,
    "spending": spending,
})

# (1) MCAR - sensor noise on satisfaction
mcar_mask = np.random.random(N) < 0.10
df_truth.loc[mcar_mask, "satisfaction"] = np.nan

# (2) MAR - older people skip income; missingness depends on AGE (observed)
mar_prob = np.clip((df_truth["age"] - 40) / 80, 0, 0.7)
mar_mask = np.random.random(N) < mar_prob
df_truth.loc[mar_mask, "income"] = np.nan

# (3) MNAR - high spenders skip spending; missingness depends on VALUE ITSELF
mnar_prob = np.clip(df_truth["spending"] / df_truth["spending"].max(), 0, 0.5)
mnar_mask = np.random.random(N) < mnar_prob
df_truth.loc[mnar_mask, "spending"] = np.nan

print(f"null rates:")
print(df_truth.isna().mean())
# satisfaction ~10% (MCAR)
# income ~25-35% (MAR - depends on age)
# spending ~15-25% (MNAR - depends on value)
```

This is your test bed: three columns with three different missingness mechanisms, and you have the ground truth to compare against.

---

## Exercise 6.2 - Diagnose with missingno

```python
fig, axes = plt.subplots(1, 3, figsize=(18, 4))
msno.matrix(df_truth, ax=axes[0])
axes[0].set_title("matrix - visual pattern")
msno.heatmap(df_truth, ax=axes[1])
axes[1].set_title("heatmap - null correlations")
msno.dendrogram(df_truth, ax=axes[2])
axes[2].set_title("dendrogram - group nulls together")
plt.tight_layout(); plt.show()
```

You should see:
- `satisfaction` nulls scattered evenly (MCAR)
- `income` nulls concentrated where `age` is older (visible in matrix)
- `spending` nulls correlated with high values (less visible - that's MNAR's signature)

**MAR is detectable from the data; MNAR usually isn't.** You diagnose MNAR by knowing the data-generating process, not by looking at the file.

---

## Exercise 6.3 - Four imputation strategies, compared

For each strategy, fill nulls; then fit a regression of `satisfaction ~ age + income + spending` and compare coefficients to the ground truth.

```python
# Ground truth model (no missing)
df_clean = df_truth.dropna()
print(f"clean dataset size: {len(df_clean)}")
X_truth = df_clean[["age", "income", "spending"]]
y_truth = df_clean["satisfaction"]
model_truth = LinearRegression().fit(X_truth, y_truth)
coef_truth = dict(zip(X_truth.columns, model_truth.coef_))
print(f"truth coefficients: {coef_truth}")
```

Now the strategies:

```python
df_with_nulls = df_truth.copy()
X = df_with_nulls[["age", "income", "spending"]]
y = df_with_nulls["satisfaction"]

# Strategy 1: drop rows with any null
X1 = X.dropna()
y1 = y.loc[X1.index].dropna()
X1 = X1.loc[y1.index]
m1 = LinearRegression().fit(X1, y1)

# Strategy 2: mean fill
m_imp = SimpleImputer(strategy="mean")
X2 = m_imp.fit_transform(X)
y2 = y.fillna(y.mean())
m2 = LinearRegression().fit(X2, y2)

# Strategy 3: KNN
knn_imp = KNNImputer(n_neighbors=5)
X3 = knn_imp.fit_transform(X)
y3 = y.fillna(y.mean())
m3 = LinearRegression().fit(X3, y3)

# Strategy 4: IterativeImputer (MICE-like)
it_imp = IterativeImputer(max_iter=10, random_state=42)
X4 = it_imp.fit_transform(X)
y4 = y.fillna(y.mean())
m4 = LinearRegression().fit(X4, y4)

print(f"{'strategy':<25s}  age      income    spending")
print(f"{'truth':<25s}  {coef_truth['age']:>+7.4f}  {coef_truth['income']:>+8.6f}  {coef_truth['spending']:>+8.6f}")
for name, m in [("dropna", m1), ("mean fill", m2), ("KNN", m3), ("Iterative (MICE)", m4)]:
    a, i, s = m.coef_
    print(f"{name:<25s}  {a:>+7.4f}  {i:>+8.6f}  {s:>+8.6f}")
```

You should see:
- `dropna` biases because the dropped rows aren't random (especially MNAR)
- `mean fill` shrinks coefficients toward zero (it removes variance)
- `KNN` and `IterativeImputer` are closer to truth on MAR columns; still biased on MNAR

**The lesson: imputation reduces but doesn't eliminate bias on MAR; on MNAR, no method recovers the truth.** That's why understanding *why* values are missing is critical.

---

## Exercise 6.4 - The "encode missingness as feature" pattern

For tree-based models or any model that can use a categorical/boolean:

```python
df_with_indicators = df_with_nulls.copy()
for col in ["income", "satisfaction", "spending"]:
    df_with_indicators[f"{col}_was_null"] = df_with_indicators[col].isna().astype(int)
    df_with_indicators[col] = df_with_indicators[col].fillna(df_with_indicators[col].median())

print(df_with_indicators.head())
print(f"shape: {df_with_indicators.shape}")
```

This pattern is the production default for ML - you keep the missingness information (often informative) AND a usable numeric column.

For week 11 (feature engineering) you'll combine this with target encoding and time-window features.

---

## Exercise 6.5 - Outlier detection 4 ways

Now switch to outliers. Use the income column from the synthetic data.

```python
income_observed = df_truth["income"].dropna().values.reshape(-1, 1)

# Method 1: IQR
q1, q3 = np.percentile(income_observed, [25, 75])
iqr = q3 - q1
iqr_outliers = (income_observed < q1 - 1.5*iqr) | (income_observed > q3 + 1.5*iqr)

# Method 2: Z-score
z = np.abs((income_observed - income_observed.mean()) / income_observed.std())
z_outliers = z > 3

# Method 3: Modified Z-score (MAD-based; robust)
from scipy import stats
mad = stats.median_abs_deviation(income_observed.flatten())
modz = 0.6745 * (income_observed - np.median(income_observed)) / mad
modz_outliers = np.abs(modz) > 3.5

# Method 4: Isolation Forest
iso = IsolationForest(contamination=0.05, random_state=42)
iso_outliers = iso.fit_predict(income_observed) == -1

print(f"{'method':<25s}  outliers")
print(f"{'IQR (1.5)':<25s}  {iqr_outliers.sum()}")
print(f"{'Z-score (>3)':<25s}  {z_outliers.sum()}")
print(f"{'Modified Z (MAD)':<25s}  {modz_outliers.sum()}")
print(f"{'Isolation Forest 5%':<25s}  {iso_outliers.sum()}")
```

Different methods catch different rows. Plot them:

```python
fig, axes = plt.subplots(1, 4, figsize=(20, 4))
masks = {
    "IQR": iqr_outliers.flatten(),
    "Z-score": z_outliers.flatten(),
    "Mod Z (MAD)": modz_outliers.flatten(),
    "Isolation Forest": iso_outliers,
}
for ax, (name, mask) in zip(axes, masks.items()):
    ax.hist(income_observed.flatten(), bins=50, alpha=0.5, color="blue")
    ax.scatter(income_observed[mask].flatten(),
               np.zeros(mask.sum()) + 5,
               color="red", s=10)
    ax.set_title(name)
plt.tight_layout(); plt.show()
```

**You should see that each method has a different aggressiveness.** Pick based on the cost of false positives vs missing real anomalies.

---

## Exercise 6.6 - Multivariate outliers

Some rows are normal on every column individually but weird in combination.

```python
df_multi = df_truth.dropna().copy()
X = df_multi[["age", "income", "spending"]].values

# LOF - finds locally anomalous points
lof = LocalOutlierFactor(n_neighbors=20, contamination=0.05)
lof_outliers = lof.fit_predict(X) == -1
print(f"LOF found {lof_outliers.sum()} multivariate outliers")

# Inspect - are they univariately outliers, or only multivariately?
df_multi["lof_outlier"] = lof_outliers
print("\nLOF outliers vs typical:")
print(df_multi.groupby("lof_outlier")[["age", "income", "spending"]].mean())
```

Look at the means: LOF-flagged rows might have average age and income but unusual *spending given their income* - a pattern that's invisible univariately.

This is the kind of detection fraud teams care about.

---

## Exercise 6.7 - The missingness + outliers report

Combine everything into a per-load report.

```python
def quality_report(df, dataset_name, prev_metrics=None):
    rows = []
    n = len(df)
    rows.append(("row_count", n, ""))

    for col in df.columns:
        null_pct = df[col].isna().sum() / n
        rows.append((f"null_pct.{col}", null_pct, ""))

    for col in df.select_dtypes(include=np.number).columns:
        s = df[col].dropna()
        if len(s) == 0: continue
        q1, q3 = np.percentile(s, [25, 75])
        iqr = q3 - q1
        outliers = ((s < q1 - 1.5*iqr) | (s > q3 + 1.5*iqr)).sum()
        rows.append((f"outlier_pct.{col}", outliers / n, ""))

    rep = pd.DataFrame(rows, columns=["metric", "value", "delta"])

    if prev_metrics is not None:
        prev_lookup = dict(zip(prev_metrics["metric"], prev_metrics["value"]))
        rep["delta"] = rep.apply(
            lambda r: f"{r['value'] - prev_lookup.get(r['metric'], r['value']):+.4f}",
            axis=1,
        )
    return rep

print("=== nyc_trees quality report ===")
print(quality_report(df_truth, "synthetic"))
```

Save this; run it on every load. Then compare today's report to yesterday's - that's how you catch drift before stakeholders do.

---

## Exercise 6.8 - Decide what to do about each column

Apply the framework to your synthetic data:

```python
recommendations = pd.DataFrame([
    {
        "column": "satisfaction",
        "null_pct": "10%",
        "type": "MCAR (sensor failure)",
        "strategy": "median fill + indicator",
        "reasoning": "Random missingness; median preserves rank stats; indicator preserves info."
    },
    {
        "column": "income",
        "null_pct": "25%",
        "type": "MAR (older respondents skip)",
        "strategy": "IterativeImputer (using age) + indicator",
        "reasoning": "Pattern is predictable from age; MICE-style recovers most signal."
    },
    {
        "column": "spending",
        "null_pct": "20%",
        "type": "MNAR (high spenders skip)",
        "strategy": "Encode-as-feature + downstream caveat",
        "reasoning": "No imputation recovers MNAR; document the bias; consider downstream model that handles nulls natively (XGBoost)."
    },
])
print(recommendations.to_string(index=False))
```

This recommendation table is the artifact that gets reviewed with stakeholders. The reasoning column is where senior judgment lives.

---

## Exercise 6.9 (stretch) - Bias from over-aggressive cleaning

A tiny experiment: if you naively clip everything to 99th percentile, what happens to a regression of revenue ~ purchases?

```python
# Synthetic e-commerce
N = 10000
purchases = np.random.exponential(scale=5, size=N) + 1
revenue = 20 * purchases + np.random.normal(0, 30, N)
revenue = np.clip(revenue, 0, None)

# True model
m_true = LinearRegression().fit(purchases.reshape(-1, 1), revenue)
print(f"truth slope: {m_true.coef_[0]:.3f}")

# Now clip both axes at 99th percentile
def clip99(s): return np.clip(s, np.quantile(s, 0.01), np.quantile(s, 0.99))
p_clipped = clip99(purchases)
r_clipped = clip99(revenue)
m_clipped = LinearRegression().fit(p_clipped.reshape(-1, 1), r_clipped)
print(f"clipped slope: {m_clipped.coef_[0]:.3f}")

# Visualize
fig, axes = plt.subplots(1, 2, figsize=(12, 5))
axes[0].scatter(purchases, revenue, alpha=0.3, s=5)
axes[0].set_title(f"truth - slope {m_true.coef_[0]:.2f}")
axes[1].scatter(p_clipped, r_clipped, alpha=0.3, s=5)
axes[1].set_title(f"clipped 1%/99% - slope {m_clipped.coef_[0]:.2f}")
plt.show()
```

Clipping flattens the slope - you've destroyed the relationship at the extremes. **For models that care about the relationship at the extremes (anything customer-LTV-related), this is a silent disaster.**

---

## Submission checklist

- [ ] Synthetic dataset with MCAR / MAR / MNAR columns generated
- [ ] missingno heatmap + dendrogram inspected
- [ ] 4 imputation strategies compared against ground-truth regression coefficients
- [ ] `_was_null` indicator columns added
- [ ] Outlier detection done 4 ways; results compared
- [ ] LOF multivariate detection shows rows univariately normal but jointly anomalous
- [ ] `quality_report()` function produces metrics + deltas vs previous run
- [ ] Per-column recommendation table written with reasoning
- [ ] (Stretch) Clipping-bias experiment shows slope flattening

---

## What you just did

You stopped defaulting to `df.dropna()` and `df[col].clip(...)`. You can diagnose missingness as MCAR/MAR/MNAR, pick an imputation strategy that fits, and articulate the trade-off when MNAR makes recovery impossible. You can detect outliers four ways and explain when each is appropriate.

This is the muscle that prevents the most common silent bug in analytics: **the result was wrong because the cleaning was too aggressive, and nobody noticed because the pipeline ran.**

---

**Next**: [Week 07: Normalization + Schema Enforcement →](../week-07-normalization-schema-enforcement/readme.md)
