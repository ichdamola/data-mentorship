# Week 13: Lab — EDA, Honestly

You'll do a full first-hour EDA, diagnose distribution shapes, compute bootstrap CIs and permutation tests, demonstrate Simpson's paradox, and produce a 1-page summary you'd send to a stakeholder.

## Setup

```bash
uv add matplotlib seaborn statsmodels scipy polars duckdb
```

```python
import polars as pl
import pandas as pd
import duckdb
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats
from statsmodels.stats.power import TTestIndPower

np.random.seed(42)
plt.style.use("seaborn-v0_8-darkgrid")
```

We'll use the NYC taxi parquet you already have. If not:

```bash
mkdir -p data
curl -o data/yellow_tripdata_2024-01.parquet \
    https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet
```

---

## Exercise 13.1 — First-hour scan

```python
PATH = "data/yellow_tripdata_2024-01.parquet"
con = duckdb.connect()

# Size
size = con.sql(f"SELECT COUNT(*) FROM read_parquet('{PATH}')").fetchone()[0]
print(f"rows: {size:,}")

# Schema
print("\nschema:")
print(con.sql(f"DESCRIBE SELECT * FROM read_parquet('{PATH}')").pl())

# Quick stats
print("\nSUMMARIZE (top 10 columns):")
summary = con.sql(f"SUMMARIZE FROM read_parquet('{PATH}')").pl()
print(summary.select("column_name", "column_type", "min", "max", "avg", "null_percentage").head(10))
```

You should see ~3M rows, 19 columns, types, ranges. Note any null_percentage that surprises you.

---

## Exercise 13.2 — Distribution shapes

Pick four columns with different shapes and plot them.

```python
df = pl.read_parquet(PATH).filter(
    (pl.col("fare_amount") > 0) & (pl.col("trip_distance") > 0)
).head(100_000)

cols_to_plot = ["fare_amount", "trip_distance", "tip_amount", "passenger_count"]
fig, axes = plt.subplots(2, 4, figsize=(20, 8))

for i, col in enumerate(cols_to_plot):
    x = df[col].to_numpy()
    # linear-scale histogram
    axes[0, i].hist(x, bins=100)
    axes[0, i].set_title(f"{col} (linear)")
    axes[0, i].set_yscale("log")   # log y always; many tail counts get suppressed otherwise
    # log-scale x for heavy-tailed
    axes[1, i].hist(np.log1p(x[x > 0]), bins=100)
    axes[1, i].set_title(f"log({col} + 1)")
plt.tight_layout(); plt.show()
```

You should see:

- `fare_amount`: heavy right tail; log makes it look ~normal
- `trip_distance`: also heavy right tail
- `tip_amount`: bimodal — many zeros (cash payers; week 1 lesson) + continuous positive
- `passenger_count`: discrete with a few modes (1, 2, then long tail)

**Mean ± std lies about all of these.** Always plot before summarizing.

### Q-Q plot

```python
fig, axes = plt.subplots(1, 2, figsize=(12, 4))
stats.probplot(df["fare_amount"].to_numpy(), dist="norm", plot=axes[0])
axes[0].set_title("fare_amount Q-Q vs Normal")
log_fare = np.log1p(df["fare_amount"].to_numpy())
stats.probplot(log_fare, dist="norm", plot=axes[1])
axes[1].set_title("log(fare_amount + 1) Q-Q vs Normal")
plt.show()
```

The raw is curved (not normal); the log is much closer to the diagonal. **`fare_amount` is approximately log-normal.**

---

## Exercise 13.3 — Bootstrap CIs

Compute a 95% CI for the median trip_distance using the bootstrap.

```python
def bootstrap_ci(data, statistic, n_iter=5000, alpha=0.05, rng=None):
    rng = rng or np.random.default_rng(42)
    boot = np.empty(n_iter)
    n = len(data)
    for i in range(n_iter):
        sample = rng.choice(data, size=n, replace=True)
        boot[i] = statistic(sample)
    return statistic(data), np.percentile(boot, alpha/2 * 100), np.percentile(boot, (1-alpha/2) * 100)

# Median trip distance with 95% CI
dist = df["trip_distance"].to_numpy()
point, lo, hi = bootstrap_ci(dist[:10000], np.median)
print(f"median trip distance: {point:.3f} miles  (95% CI: [{lo:.3f}, {hi:.3f}])")

# 90th percentile
point, lo, hi = bootstrap_ci(dist[:10000], lambda x: np.percentile(x, 90))
print(f"P90 trip distance:    {point:.3f} miles  (95% CI: [{lo:.3f}, {hi:.3f}])")

# Mean (compare to t-based CI for sanity)
point, lo, hi = bootstrap_ci(dist[:10000], np.mean)
print(f"mean trip distance:   {point:.3f} miles  (95% CI: [{lo:.3f}, {hi:.3f}])")

# Compare to t-based CI on mean
from scipy.stats import t
m = np.mean(dist[:10000])
s = np.std(dist[:10000], ddof=1)
n = 10000
ci = t.ppf([0.025, 0.975], df=n-1) * s / np.sqrt(n) + m
print(f"t-based CI on mean:   {m:.3f} miles  (95% CI: [{ci[0]:.3f}, {ci[1]:.3f}])")
```

You should see:
- Bootstrap CI for mean ≈ t-based CI (mean has CLT working for it at n=10k)
- Bootstrap CI for median/P90 is computable without distributional assumptions

**Bootstrap is the universal tool when you can't justify the t-based formula.**

---

## Exercise 13.4 — Permutation test

Is the average tip percentage on credit-card trips different between weekdays and weekends?

> ⚠️ **Polars version pin**: `dt.weekday()` returns **1=Mon..7=Sun in Polars ≥ 0.19**; older Polars returned 0..6 and the filter `< 6` would silently include Saturday. Use `polars >= 1.0` in this lab. The `is_in([6, 7])` form is more explicit than the inequality and self-documents which days are "weekend."

```python
trips = df.filter(pl.col("payment_type") == 1).with_columns(
    pickup_dow=pl.col("tpep_pickup_datetime").dt.weekday(),  # 1=Mon .. 7=Sun
    tip_pct=pl.col("tip_amount") / pl.col("fare_amount").clip(0.01, None),
)

weekday = trips.filter(pl.col("pickup_dow").is_in([1, 2, 3, 4, 5]))["tip_pct"].to_numpy()
weekend = trips.filter(pl.col("pickup_dow").is_in([6, 7]))["tip_pct"].to_numpy()

print(f"n weekday: {len(weekday):,}   n weekend: {len(weekend):,}")
print(f"weekday mean tip %: {weekday.mean()*100:.2f}")
print(f"weekend mean tip %: {weekend.mean()*100:.2f}")
print(f"observed difference: {(weekday.mean() - weekend.mean())*100:.3f} percentage points")

def permutation_test(a, b, n_iter=2000):
    """Permutation test on the FULL samples. A permutation test's whole
    point is that it uses every observation — sub-sampling here would
    throw away power for no reason (the inner loop is O(n) per iter,
    seconds on millions of rows)."""
    observed = a.mean() - b.mean()
    combined = np.concatenate([a, b])
    n_a = len(a)
    rng = np.random.default_rng(42)
    extreme_count = 0
    for _ in range(n_iter):
        shuffled = rng.permutation(combined)
        diff = shuffled[:n_a].mean() - shuffled[n_a:].mean()
        if abs(diff) >= abs(observed):
            extreme_count += 1
    return extreme_count / n_iter, observed

p, observed = permutation_test(weekday, weekend)
print(f"\npermutation test p-value: {p:.4f}")
print(f"observed difference: {observed*100:.3f} pp")

# Effect size — Cohen's d (sample-weighted pooled SD; works for any n_a, n_b)
n_a, n_b = len(weekday), len(weekend)
pooled_var = ((n_a - 1) * weekday.var(ddof=1) + (n_b - 1) * weekend.var(ddof=1)) / (n_a + n_b - 2)
pooled_sd = np.sqrt(pooled_var)
cohens_d = (weekday.mean() - weekend.mean()) / pooled_sd
print(f"Cohen's d: {cohens_d:.4f}")
```

You'll likely see p < 0.05 (statistical significance) but Cohen's d ≈ 0.05 (tiny effect). **Statistically detectable, practically irrelevant.** That's the difference between p-value and effect size, made concrete.

---

## Exercise 13.5 — Demonstrate Simpson's paradox

Build a synthetic example to feel the trap.

```python
# Two restaurants serving customers in two cities
# City A is busy; city B is sleepy
# Within each city, Restaurant 1 has lower satisfaction than Restaurant 2
# But Restaurant 1 has many more A customers...
# Overall, Restaurant 1 looks BETTER

n_per_cell = 1000

# Restaurant 1 — mostly in busy city A; satisfaction is ~7 in A, ~8 in B
r1_city_a = pd.DataFrame({
    "restaurant": "R1", "city": "A",
    "satisfaction": np.random.normal(7.0, 1.0, size=n_per_cell*9),  # 9000 customers
})
r1_city_b = pd.DataFrame({
    "restaurant": "R1", "city": "B",
    "satisfaction": np.random.normal(8.0, 1.0, size=n_per_cell),     # 1000 customers
})

# Restaurant 2 — mostly in sleepy city B; satisfaction is ~7.5 in A, ~8.5 in B
r2_city_a = pd.DataFrame({
    "restaurant": "R2", "city": "A",
    "satisfaction": np.random.normal(7.5, 1.0, size=n_per_cell),
})
r2_city_b = pd.DataFrame({
    "restaurant": "R2", "city": "B",
    "satisfaction": np.random.normal(8.5, 1.0, size=n_per_cell*9),
})

df_simpson = pd.concat([r1_city_a, r1_city_b, r2_city_a, r2_city_b], ignore_index=True)

# Overall mean
print("OVERALL satisfaction by restaurant:")
print(df_simpson.groupby("restaurant")["satisfaction"].mean())

print("\nSTRATIFIED by city:")
print(df_simpson.groupby(["city", "restaurant"])["satisfaction"].mean().unstack())
```

You should see:

```
OVERALL:
R1 ≈ 7.10
R2 ≈ 8.40
```

Looks like R2 is much better!

```
STRATIFIED:
city A: R1 ≈ 7.00, R2 ≈ 7.50    ← R2 wins in A
city B: R1 ≈ 8.00, R2 ≈ 8.50    ← R2 wins in B
```

R2 wins in both cities (consistent with overall). **No paradox here.** Now let's swap the populations to create a paradox:

```python
# Modified: R1 has more low-rated city-A customers; R2 has more high-rated city-B customers
# But within each city, R1 is BETTER

r1_city_a_v2 = pd.DataFrame({"restaurant": "R1", "city": "A",
    "satisfaction": np.random.normal(7.5, 1.0, size=9000)})
r1_city_b_v2 = pd.DataFrame({"restaurant": "R1", "city": "B",
    "satisfaction": np.random.normal(8.5, 1.0, size=1000)})
r2_city_a_v2 = pd.DataFrame({"restaurant": "R2", "city": "A",
    "satisfaction": np.random.normal(7.0, 1.0, size=1000)})
r2_city_b_v2 = pd.DataFrame({"restaurant": "R2", "city": "B",
    "satisfaction": np.random.normal(8.0, 1.0, size=9000)})

df_paradox = pd.concat([r1_city_a_v2, r1_city_b_v2, r2_city_a_v2, r2_city_b_v2], ignore_index=True)

print("OVERALL:")
print(df_paradox.groupby("restaurant")["satisfaction"].mean())

print("\nSTRATIFIED:")
print(df_paradox.groupby(["city", "restaurant"])["satisfaction"].mean().unstack())
```

Now you should see:

```
OVERALL:    R1 ≈ 7.60, R2 ≈ 7.90    ← R2 looks better overall
STRATIFIED:
  A:  R1 ≈ 7.5, R2 ≈ 7.0           ← R1 better in A
  B:  R1 ≈ 8.5, R2 ≈ 8.0           ← R1 better in B
```

**R1 is better in every city, but R2 looks better overall.** That's Simpson's paradox.

The fix: stratify; report subgroup means; flag the city/restaurant imbalance.

---

## Exercise 13.6 — Power analysis

For an A/B test, how many samples do you need to detect a 5% effect at 80% power?

```python
analysis = TTestIndPower()

# Cohen's d = 0.1 (small effect)
n_small = analysis.solve_power(effect_size=0.1, alpha=0.05, power=0.8, alternative="two-sided")
print(f"detect Cohen's d = 0.1: n = {int(n_small):,} per group")

# Cohen's d = 0.2 (small-medium)
n_med = analysis.solve_power(effect_size=0.2, alpha=0.05, power=0.8, alternative="two-sided")
print(f"detect Cohen's d = 0.2: n = {int(n_med):,} per group")

# Cohen's d = 0.5 (medium)
n_big = analysis.solve_power(effect_size=0.5, alpha=0.05, power=0.8, alternative="two-sided")
print(f"detect Cohen's d = 0.5: n = {int(n_big):,} per group")
```

You should see:
- d = 0.1: ~1500 per group
- d = 0.2: ~395 per group
- d = 0.5: ~64 per group

**Plan the experiment size before launching.** Underpowered experiments are common — and produce both false positives (random noise) and false negatives (real effects missed).

---

## Exercise 13.7 — A 1-page summary

For your stakeholder, produce a tight one-page Markdown document:

```markdown
# NYC taxi trip analysis — Jan 2024

## Headline
On credit-card trips, **weekend tippers leave 0.15 percentage points lower tips on average** (statistically significant; effect size trivial). No actionable difference.

## Dataset
- Source: NYC TLC Yellow Taxi Jan 2024 (Parquet, ~50MB)
- Rows: 2.97M trips
- Quality: 0.1% null on `fare_amount`; cash payers don't record tips (filtered out)

## Key distributions
- `fare_amount`: log-normal (median $11, P90 $24, P99 $58)
- `trip_distance`: heavy right tail (median 1.7 mi, P99 18 mi)
- `tip_pct` on CC trips: bimodal (a peak at 0% and a peak at ~20%)

## Key finding (with method)
- Weekday tip %: 18.7%; weekend: 18.55%; difference: −0.15 pp
- Permutation test p ≈ 0.02 (statistically significant)
- Cohen's d ≈ 0.05 (negligible effect; practical impact ≈ zero)
- 95% bootstrap CI for difference: [−0.27, −0.03] pp

## Notes & caveats
- Cash trips (~25% of total) excluded — they don't record tips
- "Tip %" here = tip_amount / fare_amount (not including tolls / surcharges)
- January 2024 only — seasonal patterns not assessed

## Recommended next step
- This signal isn't worth acting on
- Consider broader hourly/weekly patterns (proper EDA in next iteration)
```

Save it as `eda_report.md` in your project. **This is the artifact stakeholders want, not the notebook.**

---

## Exercise 13.8 (stretch) — Multiple comparisons

You run 20 t-tests at α=0.05. How many false positives do you expect by chance?

```python
n_tests = 20
alpha = 0.05
p_at_least_one_false_positive = 1 - (1 - alpha) ** n_tests
print(f"P(≥1 false positive in {n_tests} tests at α={alpha}): {p_at_least_one_false_positive:.2%}")
# ~64%

# Bonferroni correction
bonf_alpha = alpha / n_tests
p_bonf = 1 - (1 - bonf_alpha) ** n_tests
print(f"After Bonferroni (α' = {bonf_alpha:.4f}): P(≥1 FP) = {p_bonf:.2%}")
```

Now simulate: run 20 t-tests on random data; count false positives over many simulations.

```python
def simulate_tests(n_tests, n_per_group=50, n_sims=1000):
    fps = []
    for _ in range(n_sims):
        n_significant = 0
        for _ in range(n_tests):
            a = np.random.normal(0, 1, n_per_group)
            b = np.random.normal(0, 1, n_per_group)
            _, p = stats.ttest_ind(a, b)
            if p < 0.05:
                n_significant += 1
        fps.append(n_significant)
    return np.mean(fps), np.percentile(fps, 95)

mean, p95 = simulate_tests(20)
print(f"\nSimulated: average {mean:.2f} false positives in 20 tests (p95 = {int(p95)})")
```

You should see ~1 false positive on average — exactly what 20 × 5% predicts. **This is why "I found something significant on the 20th test" should never be the headline.**

---

## Submission checklist

- [ ] DuckDB SUMMARIZE on the silver Parquet
- [ ] Distribution histograms + Q-Q plot identifying log-normal for at least one column
- [ ] Bootstrap CI implemented; produces sensible 95% CIs for median and P90
- [ ] Permutation test on tip percentages; effect size (Cohen's d) reported alongside p-value
- [ ] Simpson's paradox example demonstrated with subgroup vs overall reversal
- [ ] Power analysis showing sample-size requirements for d = 0.1, 0.2, 0.5
- [ ] 1-page Markdown summary written for a hypothetical stakeholder
- [ ] (Stretch) Multiple-comparisons simulation showing ~1 false positive in 20 tests

---

## What you just did

You can read distributions visually, compute uncertainty without parametric assumptions, run hypothesis tests honestly with effect-size reporting, and recognize Simpson's paradox before it embarrasses you. You can produce a 1-page summary that lands the finding without burying the caveats.

Week 14 takes the next step — from "we found a correlation" to "we measured a cause." A/B testing and observational causal inference.

---

**Next**: [Week 14: Causal Inference + A/B Testing →](../week-14-causal-inference-ab-testing/readme.md)
