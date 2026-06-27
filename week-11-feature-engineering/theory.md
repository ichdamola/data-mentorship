# Week 11: Theory - Feature Engineering

For tabular ML, **feature engineering still beats architecture**. XGBoost on well-engineered features routinely outperforms a deep neural network on raw inputs. Even in the LLM era, tabular fraud / churn / propensity models live and die by the features.

This week is the catalog of high-leverage patterns: **categorical encoding** (one-hot, ordinal, target, hashing), **time-window features** ("user's avg purchase over last 30 days") **without time leakage**, **aggregation features** at multiple grains, and the discipline that prevents production-time surprises.

---

## Part 1: The first principle - leakage is the enemy

The single most common bug in feature engineering: **using information that wouldn't have been available at prediction time**.

Classic examples:

- Computing "user's average lifetime spend" using post-event data
- Using post-aggregation labels (e.g., feature = `n_orders_total`; label = `churned_in_next_30_days`; an "order" *after* the cutoff sneaks into the feature)
- Encoding a categorical via target mean computed on the full dataset before train/val split
- Using next-week's price as today's feature

The pattern is always the same: a value that would not have been known **at the moment you'd need to predict** appears in the training data. The model learns to "predict" using future information; the offline metric looks great; **production deployment is a disaster.**

Senior moves:

- **Always set a `cutoff_ts` per row** - "what is the timestamp at which this prediction would have been made?" - and ensure every feature is derived from data at or before that cutoff.
- **Compute aggregates over a defined `[cutoff - lookback, cutoff)` window** - not the full dataset.
- **Use point-in-time joins (week 09) for dimensional features**, never the current dimension table.

This is one of the patterns the feature-store industry (Tecton, Feast, Hopsworks) exists to enforce.

---

## Part 2: Categorical encoding - the four mainstream options

| Encoding | What | When |
|---|---|---|
| **One-hot** | One binary column per category | Low cardinality (< 50); linear models / NNs |
| **Ordinal / label** | Map categories to integers 0..N-1 | Tree models that don't care about order |
| **Target / mean** | Replace category with the mean target for that category | High cardinality; mind the leakage |
| **Hashing** | Hash category to integer mod K | Streaming; very high cardinality where collisions are OK |
| **Embedding** | Learned dense vector per category | Deep nets; very high cardinality (week 12) |

### One-hot

```python
import polars as pl
import pandas as pd

# pandas
df_pd = pd.get_dummies(df_pd, columns=["country"], prefix="country")

# Polars
df_pl = df_pl.to_dummies("country")
```

Gotchas:

- **Drop one column** to avoid multicollinearity in linear models (use `drop_first=True` in pandas).
- **High-cardinality columns explode**: a column with 10,000 distinct values becomes 10,000 columns. Use hashing or target encoding instead.
- **Test-set surprise**: if the test set has a category that wasn't in train, one-hot encoding has no column for it. Pre-define the categorical levels.

### Ordinal

```python
ranks = {"low": 0, "medium": 1, "high": 2}
df = df.with_columns(priority_ord=pl.col("priority").replace(ranks))
```

For tree models (XGBoost, LightGBM, CatBoost), ordinal is often as good as one-hot and trains faster. Trees split on `< value`; the relative ordering of integer codes mostly doesn't matter.

### Target encoding (a.k.a. mean encoding)

```
For each category c, replace it with: mean(target | category = c)
```

The win: one column instead of N. Captures category-target relationships cleanly.

**The big footgun**: if computed on the full dataset including the row's own target, you've leaked the label into the feature. The fix: **leave-one-out** or **K-fold** target encoding:

```python
from sklearn.model_selection import KFold

def target_encode_kfold(df, cat_col, target_col, n_splits=5):
    """K-fold target encoding to prevent leakage."""
    enc = df[cat_col].astype(object).copy()
    enc[:] = df[target_col].mean()  # default fill = global mean

    kf = KFold(n_splits=n_splits, shuffle=True, random_state=42)
    for train_idx, val_idx in kf.split(df):
        target_means = df.iloc[train_idx].groupby(cat_col)[target_col].mean()
        enc.iloc[val_idx] = df.iloc[val_idx][cat_col].map(target_means).fillna(df[target_col].mean())
    return enc
```

For each validation fold, target stats are computed from the training fold only. The resulting feature is leak-free.

The **`category_encoders`** library implements this and a dozen variants (CatBoost encoder, James-Stein, etc.).

### Hashing trick

```python
from sklearn.feature_extraction import FeatureHasher

hasher = FeatureHasher(n_features=2**10, input_type="string")
hashed = hasher.transform(df["user_id"].astype(str).tolist())
```

Lossy (collisions happen) but extremely fast, fixed memory, supports streaming. Use when categorical cardinality is in millions+ (think `user_id`, `ip_address`).

---

## Part 3: Numeric features - the workhorse transformations

| Transform | What | When |
|---|---|---|
| **Log** | `log(x + 1)` | Heavy-tailed distributions (prices, counts) |
| **Square root** | `sqrt(x)` | Mild heavy tails (counts) |
| **Standardization** | `(x - mean) / std` | Linear models, neural nets |
| **Min-max scaling** | `(x - min) / (max - min)` | Bounded output (image data) |
| **Quantile binning** | Discretize into N buckets | Capture non-monotone relationships |
| **Piecewise / splines** | Multiple linear segments | Linear models needing non-linearity |

For **tree models**, scaling doesn't matter - splits are scale-invariant. For **linear models** / **NNs**, standardization matters a lot.

### When transforms fight back

Log-transforming a column with zeros becomes `-inf`. Fix: `log(x + 1)`. Or use `np.log1p`.

```python
df = df.with_columns(
    log_revenue=(pl.col("revenue_cents") + 1).log(),
    sqrt_revenue=pl.col("revenue_cents").sqrt(),
)
```

For ML pipelines, **fit the transformer on train, apply to val/test**:

```python
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_val_scaled = scaler.transform(X_val)
```

Using `fit_transform` on the full dataset leaks test-set statistics into train.

---

## Part 4: Time-window aggregation features

The single highest-leverage feature class for any "predict the next thing" problem:

> "For each event, compute aggregations over the user's history in the last 30 / 7 / 1 days."

Pattern:

```sql
-- For each event, compute "trailing 30-day total orders" for that user
WITH events AS (
    SELECT user_id, event_ts, label FROM training_events
),
order_history AS (
    SELECT user_id, order_ts, amount FROM orders
)
SELECT
    e.user_id,
    e.event_ts,
    e.label,
    COUNT(o.order_ts) AS n_orders_30d,
    SUM(o.amount) AS total_30d,
    MAX(o.order_ts) AS last_order_ts,
FROM events e
LEFT JOIN order_history o
    ON o.user_id = e.user_id
    AND o.order_ts < e.event_ts                          -- before event
    AND o.order_ts >= e.event_ts - INTERVAL '30 days'   -- within 30d
GROUP BY e.user_id, e.event_ts, e.label;
```

The critical constraint: `o.order_ts < e.event_ts`. **Without this, you're using future orders to predict past events.** Production blow-up.

### Multiple time windows

It's standard to compute features at multiple windows (1d, 7d, 30d, 90d) and let the model decide which signal is strongest. The trade-off: feature explosion. 10 aggregates × 4 windows × 10 categories = 400 features.

### As-of joins handle this

If your dimension table is point-in-time (week 09), you can do:

```python
features = events.join_asof(
    customer_history,
    left_on="event_ts",
    right_on="valid_from",
    by="customer_id",
    strategy="backward",
)
```

Returns the **state of the customer as known at event_ts** - no leakage by construction.

---

## Part 5: Cross-source aggregations

Combining features from multiple sources at multiple grains:

```
- customer_features:
  - account_age_days  (from CRM)
  - plan_tier         (from billing)
  - last_email_open   (from marketing)
  - n_support_tickets (from support)

- customer × product features:
  - n_purchases_this_product
  - last_purchase_of_product_ts
  - days_since_first_purchase
```

For 1 user × 100 products = 100 feature rows per user. The cardinality explodes, but the signal is enormous for recommendations / propensity / next-best-action.

The feature-store pattern materializes these grain-by-grain:

```
features_customer_daily       (key: customer_id × date)
features_customer_product_daily (key: customer_id × product_id × date)
features_session             (key: session_id)
```

Models join the grains they need at training time.

---

## Part 6: Date / time features

A timestamp column hides 15+ features:

```python
df = df.with_columns(
    year=pl.col("ts").dt.year(),
    month=pl.col("ts").dt.month(),
    day_of_week=pl.col("ts").dt.weekday(),
    hour=pl.col("ts").dt.hour(),
    day_of_year=pl.col("ts").dt.ordinal_day(),
    week_of_year=pl.col("ts").dt.week(),
    is_weekend=pl.col("ts").dt.weekday() >= 6,
    quarter=pl.col("ts").dt.quarter(),
    days_to_next_holiday=...,  # week 10 enrichment
    days_since_last_holiday=...,
)
```

Cyclical features matter for some models:

```python
import numpy as np
df = df.with_columns(
    hour_sin=(2 * np.pi * pl.col("hour") / 24).sin(),
    hour_cos=(2 * np.pi * pl.col("hour") / 24).cos(),
)
```

This handles the fact that hour 23 and hour 0 are adjacent - not maximally far apart as integer encoding suggests. For linear models / NNs, cyclical encoding helps. For trees, integer hour is fine.

---

## Part 7: Interaction features

Features that combine columns multiplicatively:

```python
df = df.with_columns(
    spend_per_session=pl.col("total_spend") / pl.col("sessions").clip(1, None),
    high_value_active=(pl.col("revenue") > 1000) & (pl.col("days_since_last_visit") < 7),
)
```

For linear models, hand-crafted interactions matter enormously (XGBoost discovers them automatically). For tree models, they speed up training and often modestly improve quality.

---

## Part 8: The sklearn Pipeline pattern

Wrap your full feature-engineering process in a sklearn `Pipeline`. Three wins:

1. **Reproducibility**: applying the exact same transforms to inference data
2. **Leak prevention**: `fit` only on train; `transform` on test
3. **Cross-validation friendliness**: CV refits the pipeline each fold

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.ensemble import GradientBoostingClassifier

numeric_features = ["amount", "trip_distance", "fare_per_mile"]
categorical_features = ["payment_type", "borough"]

preprocessor = ColumnTransformer([
    ("num", Pipeline([
        ("imputer", SimpleImputer(strategy="median")),
        ("scaler", StandardScaler()),
    ]), numeric_features),
    ("cat", OneHotEncoder(handle_unknown="ignore"), categorical_features),
])

model = Pipeline([
    ("preprocessor", preprocessor),
    ("clf", GradientBoostingClassifier()),
])

model.fit(X_train, y_train)
score = model.score(X_test, y_test)
```

For more sophisticated pipelines, `sklearn-pandas` and `featuretools` exist but you can do 90% of real work with vanilla sklearn Pipeline + ColumnTransformer.

---

## Part 9: Feature store basics (a peek)

For production ML at scale, **feature stores** materialize and serve features:

| Pattern | What |
|---|---|
| **Offline store** | Bulk features for training; usually a data warehouse (Parquet, Snowflake) |
| **Online store** | Low-latency features for inference; usually Redis / DynamoDB |
| **Materialization job** | Runs nightly/hourly to refresh both stores |
| **Point-in-time training** | Get features as they were at historical event timestamps |

Open-source: **Feast** (LFAI/CNCF), **Hopsworks**.
Commercial: **Tecton**, **Vertex AI Feature Store**, **AWS SageMaker Feature Store**.

For a team of 1-5 building tabular models, feature stores are usually overkill. The pattern of "engineer features in dbt + Parquet + sklearn Pipeline" handles the same problem at smaller scale.

---

## Part 10: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| Target-encoding before train/test split | Leakage; offline metric inflated 5-30 points |
| Using post-cutoff data in time-window features | Same as above |
| Standardizing on full dataset before split | Mild leakage; subtle but real |
| One-hot encoding 10k-cardinality categorical | Memory + sparsity problems |
| `fit_transform` on test data | Statistical leak |
| Storing features only as code, not as artifacts | Can't reproduce 3 months later |
| Multiplying numerical features for "interactions" without basis | Multicollinearity; harder to debug |

---

## Part 11: Connect to the rest of the curriculum

- **Week 09 (joins)**: Point-in-time joins for dimensional features.
- **Week 10 (enrichment)**: Enriched columns become features.
- **Week 12 (embeddings)**: Embedding-as-feature for high-cardinality text columns.
- **Week 16 (production)**: Feature pipelines as dbt models with backfill.
- **ml-mentorship week 04 + 05**: The classical ML / NN side of the same conversation.

---

## What's next

In [lab.md](lab.md) you'll:

- Build a synthetic e-commerce dataset with timestamps, users, products
- Engineer time-window aggregation features with no leakage
- Implement K-fold target encoding to prevent leakage
- Encode dates with cyclical features
- Build the canonical sklearn Pipeline
- Train XGBoost; compare against a baseline with no feature engineering
- (Stretch) Deliberately introduce leakage and watch val accuracy soar in dev, crash in prod

By end of week 11 you can take a clean event table and produce a fully ML-ready feature matrix that won't betray you in production.
