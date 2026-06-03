# Week 11: Lab — Engineer Features Without Leakage

You'll build a synthetic e-commerce dataset, engineer features for "predict whether the user will churn in the next 30 days," train XGBoost, compare against a no-features baseline, and finally introduce leakage on purpose to feel its consequences.

## Setup

```bash
uv add scikit-learn category-encoders xgboost polars
```

```python
import polars as pl
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split, cross_val_score, KFold
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.metrics import roc_auc_score, classification_report
from xgboost import XGBClassifier
import category_encoders as ce
import datetime as dt
import random

np.random.seed(42)
random.seed(42)
```

---

## Exercise 11.1 — Synthesize an e-commerce event log

```python
N_USERS = 5000
N_DAYS = 180

# Generate user characteristics
users = pd.DataFrame({
    "user_id": range(N_USERS),
    "signup_date": pd.date_range(start="2023-07-01", periods=N_USERS, freq="3h").date,
    "country": np.random.choice(["US", "UK", "DE", "FR", "JP", "BR"], N_USERS, p=[0.4, 0.15, 0.15, 0.1, 0.1, 0.1]),
    "plan": np.random.choice(["free", "pro", "enterprise"], N_USERS, p=[0.6, 0.3, 0.1]),
})

# Generate per-user activity — high-spend users buy more
purchases = []
for _, u in users.iterrows():
    if u["plan"] == "free":      n_purchases = np.random.poisson(2)
    elif u["plan"] == "pro":     n_purchases = np.random.poisson(8)
    else:                        n_purchases = np.random.poisson(20)
    for _ in range(n_purchases):
        days_after = random.randint(1, N_DAYS)
        ts = pd.Timestamp(u["signup_date"]) + pd.Timedelta(days=days_after, hours=random.randint(0, 23))
        purchases.append({
            "user_id": u["user_id"],
            "purchase_ts": ts,
            "amount_cents": int(np.random.exponential(5000)),
            "category": np.random.choice(["food", "electronics", "books", "clothing", "home"]),
        })
purchases_df = pd.DataFrame(purchases)

# Generate churn labels: users who haven't purchased in the last 30 days are "at risk"
# For training: take a snapshot at 2024-04-01; label = will_purchase_in_30d (May 2024 is "future")
SNAPSHOT_DATE = pd.Timestamp("2024-04-01")

# For each user, compute: did they purchase between SNAPSHOT and SNAPSHOT + 30d?
future_purchases = purchases_df[
    (purchases_df["purchase_ts"] >= SNAPSHOT_DATE) &
    (purchases_df["purchase_ts"] < SNAPSHOT_DATE + pd.Timedelta(days=30))
].groupby("user_id").size()
labels = (~users["user_id"].isin(future_purchases.index)).astype(int)
users["churned_30d"] = labels.values

print(f"churn rate: {users['churned_30d'].mean():.2%}")
print(f"users: {len(users)}; purchases: {len(purchases_df):,}")
print(users.head())
```

You should see ~50-70% churn (most users in a synthetic e-commerce are casual). That's our prediction target.

---

## Exercise 11.2 — Baseline: features = none

Train XGBoost on just `country` and `plan`.

```python
baseline_features = pd.get_dummies(users[["country", "plan"]], drop_first=False)
y = users["churned_30d"]

X_train, X_test, y_train, y_test = train_test_split(
    baseline_features, y, test_size=0.2, random_state=42, stratify=y,
)

baseline_model = XGBClassifier(n_estimators=200, max_depth=4, random_state=42, eval_metric="logloss")
baseline_model.fit(X_train, y_train)
baseline_pred = baseline_model.predict_proba(X_test)[:, 1]
baseline_auc = roc_auc_score(y_test, baseline_pred)
print(f"baseline AUC: {baseline_auc:.4f}")
```

You should see AUC around 0.55-0.65 — barely better than random. That's the bar to beat.

---

## Exercise 11.3 — Engineer time-window aggregation features (no leakage)

For each user, compute trailing 30-day and 90-day stats **using only purchases before the SNAPSHOT_DATE**.

```python
# Critical: only use purchases BEFORE the snapshot
pre_snapshot = purchases_df[purchases_df["purchase_ts"] < SNAPSHOT_DATE]
print(f"pre-snapshot purchases: {len(pre_snapshot):,}")

# 30-day window
window_30d = pre_snapshot[
    pre_snapshot["purchase_ts"] >= SNAPSHOT_DATE - pd.Timedelta(days=30)
]
agg_30d = window_30d.groupby("user_id").agg(
    n_purchases_30d=("purchase_ts", "count"),
    total_spend_30d=("amount_cents", "sum"),
    avg_purchase_30d=("amount_cents", "mean"),
).reset_index()

# 90-day window
window_90d = pre_snapshot[
    pre_snapshot["purchase_ts"] >= SNAPSHOT_DATE - pd.Timedelta(days=90)
]
agg_90d = window_90d.groupby("user_id").agg(
    n_purchases_90d=("purchase_ts", "count"),
    total_spend_90d=("amount_cents", "sum"),
).reset_index()

# All-time pre-snapshot
all_time = pre_snapshot.groupby("user_id").agg(
    n_purchases_lifetime=("purchase_ts", "count"),
    last_purchase_ts=("purchase_ts", "max"),
).reset_index()
all_time["days_since_last_purchase"] = (SNAPSHOT_DATE - all_time["last_purchase_ts"]).dt.days
all_time = all_time.drop(columns="last_purchase_ts")

# Date features
users["signup_date_dt"] = pd.to_datetime(users["signup_date"])
users["days_since_signup"] = (SNAPSHOT_DATE - users["signup_date_dt"]).dt.days
users["signup_dow"] = users["signup_date_dt"].dt.dayofweek
users["signup_month"] = users["signup_date_dt"].dt.month

# Combine
features = (
    users[["user_id", "country", "plan", "days_since_signup", "signup_dow", "signup_month", "churned_30d"]]
    .merge(agg_30d, on="user_id", how="left")
    .merge(agg_90d, on="user_id", how="left")
    .merge(all_time, on="user_id", how="left")
    .fillna(0)  # users with no purchases get 0 for all aggregates
)
print(features.head())
print(features.columns.tolist())
```

You should see 12+ engineered features per user, all computed from data **strictly before** SNAPSHOT_DATE.

---

## Exercise 11.4 — Train with engineered features

```python
feature_cols = [
    "country", "plan", "days_since_signup", "signup_dow", "signup_month",
    "n_purchases_30d", "total_spend_30d", "avg_purchase_30d",
    "n_purchases_90d", "total_spend_90d",
    "n_purchases_lifetime", "days_since_last_purchase",
]

X = pd.get_dummies(features[feature_cols], columns=["country", "plan"], drop_first=False)
y = features["churned_30d"]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

model = XGBClassifier(n_estimators=300, max_depth=5, learning_rate=0.05, random_state=42, eval_metric="logloss")
model.fit(X_train, y_train)
pred = model.predict_proba(X_test)[:, 1]
auc = roc_auc_score(y_test, pred)
print(f"engineered features AUC: {auc:.4f}")
print(f"baseline:                {baseline_auc:.4f}")
print(f"improvement:             {auc - baseline_auc:+.4f}")

# Feature importance
import polars as pl
importances = pd.Series(model.feature_importances_, index=X.columns).sort_values(ascending=False)
print("\ntop 10 features:")
print(importances.head(10))
```

You should see AUC jump to 0.85+. **The features added ~25 AUC points; the model only added a few.** Standard tabular ML lesson.

---

## Exercise 11.5 — K-fold target encoding

Demonstrate target encoding done right. For `country`, replace it with the leak-free target mean.

```python
def kfold_target_encode(df, cat_col, target_col, n_splits=5):
    enc = pd.Series(df[target_col].mean(), index=df.index, dtype=float)
    kf = KFold(n_splits=n_splits, shuffle=True, random_state=42)
    for train_idx, val_idx in kf.split(df):
        target_means = df.iloc[train_idx].groupby(cat_col)[target_col].mean()
        global_mean = df.iloc[train_idx][target_col].mean()
        enc.iloc[val_idx] = df.iloc[val_idx][cat_col].map(target_means).fillna(global_mean).values
    return enc

# Apply to country, plan
features["country_target_encoded"] = kfold_target_encode(features, "country", "churned_30d")
features["plan_target_encoded"] = kfold_target_encode(features, "plan", "churned_30d")

print(features.groupby("country")["country_target_encoded"].mean())   # leak-free means
print(features.groupby("plan")["plan_target_encoded"].mean())
```

You should see different countries/plans encoded to different values reflecting their true target rates. **No specific row contributed to its own encoding** — that's the leakage prevention.

---

## Exercise 11.6 — sklearn Pipeline with ColumnTransformer

The reproducible pattern:

```python
numeric_features = [
    "days_since_signup", "signup_dow", "signup_month",
    "n_purchases_30d", "total_spend_30d", "avg_purchase_30d",
    "n_purchases_90d", "total_spend_90d",
    "n_purchases_lifetime", "days_since_last_purchase",
]
categorical_features = ["country", "plan"]

preprocessor = ColumnTransformer([
    ("num", Pipeline([
        ("imputer", SimpleImputer(strategy="median")),
        ("scaler", StandardScaler()),
    ]), numeric_features),
    ("cat", OneHotEncoder(handle_unknown="ignore", sparse_output=False), categorical_features),
])

pipe = Pipeline([
    ("preprocessor", preprocessor),
    ("clf", XGBClassifier(n_estimators=300, max_depth=5, learning_rate=0.05, random_state=42, eval_metric="logloss")),
])

# Split and fit
X_full = features[numeric_features + categorical_features]
y_full = features["churned_30d"]
X_train, X_test, y_train, y_test = train_test_split(X_full, y_full, test_size=0.2, random_state=42, stratify=y_full)

pipe.fit(X_train, y_train)
pred = pipe.predict_proba(X_test)[:, 1]
print(f"pipeline AUC: {roc_auc_score(y_test, pred):.4f}")

# Cross-validate
cv_scores = cross_val_score(pipe, X_full, y_full, cv=5, scoring="roc_auc")
print(f"5-fold CV AUC: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")
```

The pipeline:
1. Imputes missing numerics with median
2. Standardizes numerics
3. One-hot encodes categoricals (handles unknown categories at inference time)
4. Trains XGBoost

`cross_val_score` re-fits the entire pipeline (including scaler stats) per fold — **no leakage between train and val statistics**. This is the right shape.

---

## Exercise 11.7 — Cyclical encoding for time

```python
# Add cyclical features for signup_dow (day of week) and signup_month
def cyclical_encode(s, period):
    radians = 2 * np.pi * s / period
    return np.sin(radians), np.cos(radians)

features["signup_dow_sin"], features["signup_dow_cos"] = cyclical_encode(features["signup_dow"], 7)
features["signup_month_sin"], features["signup_month_cos"] = cyclical_encode(features["signup_month"], 12)

# Plot to confirm
import matplotlib.pyplot as plt
fig, axes = plt.subplots(1, 2, figsize=(12, 5))
axes[0].scatter(features["signup_dow"], features["signup_dow_sin"])
axes[0].set_title("dow_sin"); axes[0].set_xlabel("day of week")
axes[1].scatter(features["signup_dow"], features["signup_dow_cos"])
axes[1].set_title("dow_cos"); axes[1].set_xlabel("day of week")
plt.tight_layout(); plt.show()
```

You should see smooth sin/cos curves. **Day 6 (Saturday) and day 0 (Monday) are now close to each other in the encoded space.** For tree models this matters less; for linear models / NNs, it's the right pattern.

---

## Exercise 11.8 — Deliberately introduce leakage

Train with the "all-future-purchases" feature — what XGBoost would do if you didn't enforce the cutoff.

```python
# LEAKED feature: total purchases in next 30 days (which is the LABEL!)
future_n_purchases = purchases_df[
    (purchases_df["purchase_ts"] >= SNAPSHOT_DATE) &
    (purchases_df["purchase_ts"] < SNAPSHOT_DATE + pd.Timedelta(days=30))
].groupby("user_id").size().to_dict()

features["n_purchases_FUTURE_30d"] = features["user_id"].map(future_n_purchases).fillna(0)

# Train with the leak
X_leak = features[numeric_features + ["n_purchases_FUTURE_30d"] + categorical_features]
y = features["churned_30d"]
X_train_l, X_test_l, y_train_l, y_test_l = train_test_split(X_leak, y, test_size=0.2, random_state=42, stratify=y)

pipe_leak = Pipeline([
    ("preprocessor", ColumnTransformer([
        ("num", StandardScaler(), numeric_features + ["n_purchases_FUTURE_30d"]),
        ("cat", OneHotEncoder(handle_unknown="ignore", sparse_output=False), categorical_features),
    ])),
    ("clf", XGBClassifier(n_estimators=300, max_depth=5, learning_rate=0.05, random_state=42)),
])
pipe_leak.fit(X_train_l, y_train_l)
leak_pred = pipe_leak.predict_proba(X_test_l)[:, 1]
print(f"WITH LEAK AUC: {roc_auc_score(y_test_l, leak_pred):.4f}")
print(f"clean AUC:     {roc_auc_score(y_test, pred):.4f}")
print(f"leakage gain (FALSE):  {roc_auc_score(y_test_l, leak_pred) - roc_auc_score(y_test, pred):+.4f}")
```

You should see the AUC jump to ~0.95+ with leakage. **That's the inflated metric that would have looked great in dev and crashed in prod** — because the production system has no way to know the future.

The lesson: **the offline metric is meaningless if leakage is possible.** Always verify time-window constraints with the team.

---

## Exercise 11.9 — Save the pipeline + features metadata

```python
import joblib

# Save the pipeline (reproducible inference)
joblib.dump(pipe, "data/silver/churn_pipeline.joblib")

# Save feature metadata
metadata = pl.DataFrame({
    "column": ["country", "plan", "days_since_signup", "n_purchases_30d", "total_spend_30d",
               "avg_purchase_30d", "n_purchases_90d", "total_spend_90d", "n_purchases_lifetime",
               "days_since_last_purchase"],
    "source": ["users.country", "users.plan", "computed from users.signup_date",
               "purchases windowed by SNAPSHOT - 30d", "purchases windowed by SNAPSHOT - 30d",
               "purchases windowed by SNAPSHOT - 30d", "purchases windowed by SNAPSHOT - 90d",
               "purchases windowed by SNAPSHOT - 90d", "all pre-SNAPSHOT purchases",
               "all pre-SNAPSHOT purchases"],
    "type": ["categorical", "categorical", "numeric"] + ["numeric"] * 7,
    "cutoff_ts": ["snapshot"] * 10,
    "lookback_days": [None, None, None, 30, 30, 30, 90, 90, None, None],
})
metadata.write_parquet("data/silver/churn_features_metadata.parquet")
print(metadata)
```

Now in 6 months when someone asks "what's `n_purchases_30d`?", the metadata answers without anyone's memory being involved.

---

## Submission checklist

- [ ] Synthetic e-commerce dataset generated with 5000 users + thousands of purchases
- [ ] Baseline model with just country + plan; AUC computed
- [ ] Time-window aggregations computed with explicit `<` constraint vs snapshot
- [ ] Engineered features lift AUC by 0.15+ over baseline
- [ ] K-fold target encoding implemented; demonstrated no leakage
- [ ] sklearn Pipeline with ColumnTransformer for reproducible inference
- [ ] Cyclical features for date columns
- [ ] Deliberate leakage experiment shows AUC jump (0.95+) followed by understanding-of-prod-blowup
- [ ] Pipeline + features metadata saved

---

## What you just did

You can take a clean event log and produce a feature matrix with proper cutoffs. You can encode high-cardinality categoricals without leakage. You can wrap it all in an sklearn Pipeline that re-fits on each CV fold. You've felt the dramatic AUC inflation from a leaky feature firsthand.

Week 12 introduces embeddings — the modern technique for very-high-cardinality text features that classical encoding can't handle.

---

**Next**: [Week 12: Vector Embeddings + Semantic Enrichment →](../week-12-vector-embeddings-semantic/readme.md)
