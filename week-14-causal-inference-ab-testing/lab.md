# Week 14: Lab — Run an A/B Test Without Lying to Yourself

You'll design and analyze a simulated A/B test, demonstrate peeking inflation by simulation, detect SRM, do propensity score matching on observational data, and finish with a 1-page A/B result template.

## Setup

```bash
uv add statsmodels econml dowhy scipy polars matplotlib
```

```python
import numpy as np
import pandas as pd
import polars as pl
import matplotlib.pyplot as plt
from scipy import stats
from statsmodels.stats.power import NormalIndPower
from sklearn.linear_model import LogisticRegression
from sklearn.neighbors import NearestNeighbors

np.random.seed(42)
```

---

## Exercise 14.1 — Plan the experiment (power analysis)

Baseline conversion is 10%. We want to detect a 2% relative lift.

```python
baseline_conv = 0.10
mde_relative = 0.02
treatment_conv = baseline_conv * (1 + mde_relative)

def cohens_h(p1, p2):
    return 2 * (np.arcsin(np.sqrt(p1)) - np.arcsin(np.sqrt(p2)))

effect_size = cohens_h(baseline_conv, treatment_conv)
analysis = NormalIndPower()
n_per_group = analysis.solve_power(effect_size=effect_size, alpha=0.05, power=0.80, alternative="two-sided")
print(f"sample size per group for 2% relative lift detection: {int(n_per_group):,}")

# What if MDE is 5%?
mde_5 = 0.05
n_5 = analysis.solve_power(
    effect_size=cohens_h(baseline_conv, baseline_conv * (1 + mde_5)),
    alpha=0.05, power=0.80, alternative="two-sided",
)
print(f"sample size per group for 5% relative lift detection: {int(n_5):,}")
```

**Run it.** At baseline 10% with a relative MDE of 2% (absolute lift 10.0% → 10.2%), `cohens_h ≈ 0.0066`, and the math gives roughly **n ≈ 180,000 per group**, not 80,000 — a 2% relative lift on a 10% baseline is a tiny absolute effect and demands a lot of samples. At 5% relative MDE you need around **n ≈ 30,000 per group**. Trust the number `solve_power` prints, not a memorized one — the MDE relationship is highly non-linear.

---

## Exercise 14.2 — Run a clean A/B test (with simulation)

```python
N_PER_GROUP = 80_000
TRUE_LIFT_RELATIVE = 0.02   # let's say the treatment actually works

# Simulate
control = np.random.binomial(1, baseline_conv, N_PER_GROUP)
treatment = np.random.binomial(1, baseline_conv * (1 + TRUE_LIFT_RELATIVE), N_PER_GROUP)

# Analyze
n_c, n_t = N_PER_GROUP, N_PER_GROUP
conv_c, conv_t = control.mean(), treatment.mean()
lift_abs = conv_t - conv_c
lift_rel = lift_abs / conv_c

p_pooled = (control.sum() + treatment.sum()) / (n_c + n_t)
se = np.sqrt(p_pooled * (1 - p_pooled) * (1/n_c + 1/n_t))
z = lift_abs / se
p_value = 2 * (1 - stats.norm.cdf(abs(z)))

ci_lo = lift_abs - 1.96 * se
ci_hi = lift_abs + 1.96 * se

print(f"control conv:  {conv_c:.4f}")
print(f"treatment conv: {conv_t:.4f}")
print(f"absolute lift: {lift_abs:+.4f} (95% CI: [{ci_lo:+.4f}, {ci_hi:+.4f}])")
print(f"relative lift: {lift_rel:+.2%}")
print(f"p-value: {p_value:.4f}")
```

You should see a positive lift with p < 0.05 most runs (you have 80% power for a real 2% effect).

---

## Exercise 14.3 — Demonstrate peeking inflation

What happens if you check daily and stop when p < 0.05?

```python
def simulate_with_peeking(n_total, baseline_conv, true_lift, n_checks=10, n_sims=1000):
    """Returns fraction of sims where we 'find' significance via peeking at any checkpoint."""
    n_per_check = n_total // n_checks
    false_positives = 0   # treatment_conv = baseline_conv * (1 + true_lift)
    treatment_conv = baseline_conv * (1 + true_lift)

    for _ in range(n_sims):
        control = np.random.binomial(1, baseline_conv, n_total)
        treatment = np.random.binomial(1, treatment_conv, n_total)
        for check_n in range(n_per_check, n_total + 1, n_per_check):
            c_subset = control[:check_n]
            t_subset = treatment[:check_n]
            p_pooled = (c_subset.sum() + t_subset.sum()) / (2 * check_n)
            se = np.sqrt(p_pooled * (1 - p_pooled) * (2 / check_n))
            if se > 0:
                z = (t_subset.mean() - c_subset.mean()) / se
                p = 2 * (1 - stats.norm.cdf(abs(z)))
                if p < 0.05:
                    false_positives += 1
                    break
    return false_positives / n_sims

# Side-by-side comparison: NO peeking (test only at the end) vs peeking 10×.
# Without the baseline you have no calibration point — you'd just see one
# number and trust the prose.

def simulate_no_peeking(n_total, baseline_conv, true_lift, n_sims=1000):
    """Test only once, at the end. Should sit at ~5% under the null."""
    treatment_conv = baseline_conv * (1 + true_lift)
    rejections = 0
    for _ in range(n_sims):
        control = np.random.binomial(1, baseline_conv, n_total)
        treatment = np.random.binomial(1, treatment_conv, n_total)
        p_pooled = (control.sum() + treatment.sum()) / (2 * n_total)
        se = np.sqrt(p_pooled * (1 - p_pooled) * (2 / n_total))
        if se > 0:
            z = (treatment.mean() - control.mean()) / se
            p = 2 * (1 - stats.norm.cdf(abs(z)))
            if p < 0.05:
                rejections += 1
    return rejections / n_sims

# Under the TRUE null effect:
fp_no_peek = simulate_no_peeking(n_total=20_000, baseline_conv=0.10, true_lift=0.0, n_sims=500)
fp_peek    = simulate_with_peeking(n_total=20_000, baseline_conv=0.10, true_lift=0.0, n_checks=10, n_sims=500)
print(f"NO peeking, test once at end:  'significant' in {fp_no_peek:.1%} of sims  ← should be ~5%")
print(f"peeking 10 times, stop at first p<0.05:  {fp_peek:.1%} of sims  ← inflated")
print(f"\nThe difference IS alpha inflation. Without sequential correction,")
print(f"every extra peek bleeds false positives into your 'significant' bucket.")
```

You should see the no-peeking FPR sit near 5% and the peeking FPR climb to 15-30%. **The gap is alpha inflation made concrete.**

---

## Exercise 14.4 — Detect SRM

```python
def srm_test(control_n, treatment_n, expected_ratio=0.5):
    result = stats.binomtest(k=control_n, n=control_n + treatment_n, p=expected_ratio)
    return result.pvalue

# Good — both groups close to 50/50
good_p = srm_test(40_000, 40_000)
print(f"clean A/B (40k vs 40k): SRM p = {good_p:.4f}")

# Bad — 47/53 split (just 3pp off)
bad_p = srm_test(47_000, 53_000)
print(f"SRM A/B (47k vs 53k):  SRM p = {bad_p:.2e}")

# When SRM is detected, the lift result is suspect
if bad_p < 0.001:
    print("\n⚠️ SRM detected. DO NOT trust the conversion lift result.")
    print("    Possible causes: biased random assignment, downstream filtering")
    print("    Action: debug the experiment pipeline before reanalysis.")
```

Run SRM test first on every real A/B. **No exception.**

---

## Exercise 14.5 — Segment analysis

Even when overall result is clear, segments can flip.

```python
# Simulate: overall lift is +2%, but Android users see no lift; iOS users see +5%
N = 100_000
is_ios = np.random.binomial(1, 0.5, N)
treated = np.random.binomial(1, 0.5, N)

control_rate = 0.10
ios_effect = 0.05
android_effect = 0.0

p_conv = control_rate * np.where(
    treated == 0, 1, 1 + np.where(is_ios == 1, ios_effect, android_effect)
)
converted = np.random.binomial(1, p_conv, N)

# Overall
ctrl = converted[treated == 0]
trt = converted[treated == 1]
print(f"OVERALL lift: {trt.mean() - ctrl.mean():+.4f} ({(trt.mean()/ctrl.mean() - 1):+.2%})")

# By segment
for segment_name, segment_mask in [("iOS", is_ios == 1), ("Android", is_ios == 0)]:
    c_seg = converted[(treated == 0) & segment_mask]
    t_seg = converted[(treated == 1) & segment_mask]
    if len(c_seg) > 100 and len(t_seg) > 100:
        print(f"  {segment_name}: {t_seg.mean() - c_seg.mean():+.4f} ({(t_seg.mean()/c_seg.mean() - 1):+.2%}, n_c={len(c_seg)}, n_t={len(t_seg)})")
```

You'll see the overall ~+2.5% obscures iOS-only +5% and Android-flat. **Always segment-analyze top categories — typically OS, region, plan, country.**

---

## Exercise 14.6 — Bayesian A/B as alternative

```python
# Same data as 14.2
control_successes = control.sum()
control_failures = len(control) - control_successes
treatment_successes = treatment.sum()
treatment_failures = len(treatment) - treatment_successes

# Beta priors. Beta(1,1) is the uniform prior (equivalent to 2 pseudo-obs);
# Beta(0.5, 0.5) is the Jeffreys prior — the conventional default for
# proportions and what most A/B platforms use. At sample sizes >1000 they're
# indistinguishable, but for very small studies pick deliberately.
PRIOR_A, PRIOR_B = 1, 1   # uniform; switch to 0.5, 0.5 for Jeffreys
post_control = stats.beta(PRIOR_A + control_successes, PRIOR_B + control_failures)
post_treatment = stats.beta(PRIOR_A + treatment_successes, PRIOR_B + treatment_failures)

# Monte Carlo
samples = 100_000
c_samples = post_control.rvs(samples)
t_samples = post_treatment.rvs(samples)

prob_treatment_better = (t_samples > c_samples).mean()
median_lift = np.median(t_samples - c_samples)
ci_lo, ci_hi = np.percentile(t_samples - c_samples, [2.5, 97.5])

print(f"P(treatment > control) = {prob_treatment_better:.2%}")
print(f"median lift: {median_lift:+.4f} (95% credible interval: [{ci_lo:+.4f}, {ci_hi:+.4f}])")
```

You'll see something like "P(treatment > control) = 95%" — directly answering "is treatment better?" without invoking p-values.

For most stakeholders, this language ("there's a 95% probability treatment beats control") is more intuitive than frequentist p-values.

---

## Exercise 14.7 — Propensity score matching on observational data

You have historical data with "users who chose the new feature" vs "users who didn't" — no randomization.

```python
# Simulate: users self-selected
N = 5000
# Covariates that affect both treatment and outcome
age = np.random.normal(40, 12, N)
income = np.random.exponential(50000, N)
prior_engagement = np.random.exponential(10, N)

# Confounded treatment assignment — high-engagement users tend to opt-in
treatment_prob = stats.expit(-2 + 0.05 * prior_engagement + 0.01 * income / 10000)
treated = np.random.binomial(1, treatment_prob)

# True causal effect on outcome
TRUE_EFFECT = 0.10
outcome_baseline = 0.50 + 0.01 * prior_engagement - 0.005 * age + np.random.normal(0, 0.5, N)
outcome = outcome_baseline + TRUE_EFFECT * treated

df = pd.DataFrame({
    "age": age, "income": income, "prior_engagement": prior_engagement,
    "treated": treated, "outcome": outcome,
})

# Naive comparison (will overestimate the effect due to confounding)
naive_effect = df.loc[df["treated"] == 1, "outcome"].mean() - df.loc[df["treated"] == 0, "outcome"].mean()
print(f"naive effect estimate: {naive_effect:+.4f}")
print(f"TRUE effect (we know):  {TRUE_EFFECT:+.4f}")
print(f"(naive overestimates because of confounding)")
```

You should see naive effect substantially > TRUE_EFFECT. Now adjust:

```python
# Fit propensity
covariates = df[["age", "income", "prior_engagement"]].values
propensity_model = LogisticRegression(max_iter=1000).fit(covariates, df["treated"])
df["propensity"] = propensity_model.predict_proba(covariates)[:, 1]

# Nearest-neighbor matching: each treated unit → 1 control with closest propensity.
# This is 1-NN matching WITH REPLACEMENT (a single high-propensity control can
# be matched to many treated units). Documented method, but it loses the
# independence assumption needed for a naive matched-pair t-test — for SEs,
# bootstrap the whole pipeline.
treated_df = df[df["treated"] == 1]
control_df = df[df["treated"] == 0]

# Common-support check: PSM is unidentified when the propensity ranges don't
# overlap (e.g., some treated have propensity 0.95 but no control exceeds 0.5).
print(f"propensity ranges — treated: [{treated_df['propensity'].min():.3f}, {treated_df['propensity'].max():.3f}]"
      f"  control: [{control_df['propensity'].min():.3f}, {control_df['propensity'].max():.3f}]")
assert treated_df["propensity"].min() >= control_df["propensity"].min() and \
       treated_df["propensity"].max() <= control_df["propensity"].max(), \
       "common-support violated — drop the off-support treated units before matching"

nbrs = NearestNeighbors(n_neighbors=1).fit(control_df[["propensity"]])
distances, indices = nbrs.kneighbors(treated_df[["propensity"]])

# CALIPER: drop matches where the propensities differ by more than 0.2 of an SD
# of the propensity (standard rule of thumb, Stuart 2010). Without a caliper
# you sometimes match propensity-0.9 treated to propensity-0.3 control.
caliper = 0.2 * df["propensity"].std()
keep = distances.flatten() <= caliper
print(f"kept {keep.sum()} / {len(keep)} matches within caliper {caliper:.3f}")

matched_treated_outcomes = treated_df["outcome"].values[keep]
matched_control_outcomes = control_df["outcome"].iloc[indices.flatten()].values[keep]

psm_effect = (matched_treated_outcomes - matched_control_outcomes).mean()

# Bootstrap SE — the matched pairs aren't independent (with-replacement
# matching + caliper); naive t-test SEs are wrong. Bootstrap the whole pipeline.
rng = np.random.default_rng(42)
boot = []
for _ in range(500):
    idx = rng.choice(len(df), size=len(df), replace=True)
    boot_df = df.iloc[idx].reset_index(drop=True)
    boot_t = boot_df[boot_df["treated"] == 1]
    boot_c = boot_df[boot_df["treated"] == 0]
    if len(boot_t) < 10 or len(boot_c) < 10:
        continue
    b_nbrs = NearestNeighbors(n_neighbors=1).fit(boot_c[["propensity"]])
    b_d, b_i = b_nbrs.kneighbors(boot_t[["propensity"]])
    b_keep = b_d.flatten() <= caliper
    if b_keep.sum() < 10:
        continue
    boot.append(
        (boot_t["outcome"].values[b_keep] -
         boot_c["outcome"].iloc[b_i.flatten()].values[b_keep]).mean()
    )
boot_lo, boot_hi = np.percentile(boot, [2.5, 97.5])

print(f"\nPSM-adjusted effect: {psm_effect:+.4f}  (bootstrap 95% CI: [{boot_lo:+.4f}, {boot_hi:+.4f}])")
print(f"TRUE effect:          {TRUE_EFFECT:+.4f}")
```

PSM should produce an estimate much closer to TRUE_EFFECT than naive, and the bootstrap CI should cover TRUE_EFFECT. **Adjustment works when confounders are observed and the model captures them.**

> 💡 **In production**, prefer `econml.dml.LinearDML` (double-ML) or `dowhy.CausalModel.estimate_effect(..., method_name="backdoor.propensity_score_matching")` — they handle common support, calipers, SEs, and sensitivity to propensity-model misspecification properly. The roll-your-own version above is for understanding the mechanics.

---

## Exercise 14.8 — Difference-in-Differences

A retailer launches a new feature in NYC only. Compare NYC pre/post to other-city pre/post.

```python
# Simulate
N_NYC = 1000
N_OTHER = 1000
months = [1, 2, 3, 4]   # 1-2 = pre, 3-4 = post (treatment in NYC at month 3)

data = []
for m in months:
    is_post = m >= 3
    nyc_baseline = 100 + m * 2   # NYC grows
    other_baseline = 100 + m * 2   # other cities grow (parallel trends)

    # Add treatment effect only to NYC, only post
    if is_post:
        nyc_effect = 10   # +10 units lift
    else:
        nyc_effect = 0

    for _ in range(N_NYC):
        data.append({
            "city": "NYC", "month": m, "post": is_post,
            "sales": nyc_baseline + nyc_effect + np.random.normal(0, 5),
        })
    for _ in range(N_OTHER):
        data.append({
            "city": "Other", "month": m, "post": is_post,
            "sales": other_baseline + np.random.normal(0, 5),
        })

did_df = pd.DataFrame(data)

# DiD computation
groups = did_df.groupby(["city", "post"])["sales"].mean().unstack()
print("Mean sales by group/period:")
print(groups)
print()

# DiD = (NYC_post - NYC_pre) - (Other_post - Other_pre)
did = (groups.loc["NYC", True] - groups.loc["NYC", False]) - (groups.loc["Other", True] - groups.loc["Other", False])
print(f"DiD estimate of treatment effect: {did:+.2f}")
print(f"(true effect was +10)")
```

You should see the DiD recover ~+10. The parallel trends assumption was satisfied by design; in real DiD, check it visually first.

---

## Exercise 14.9 — Confounder vs collider example

```python
# Re-seed for reproducibility — prior exercises consume the global RNG.
np.random.seed(42)

# Confounder: Z affects both T and Y
# If we adjust for Z, we get the true effect
n = 5000
Z = np.random.normal(0, 1, n)               # the confounder
T = (Z + np.random.normal(0, 1, n)) > 0     # treatment depends on Z
Y = 2 * T + 3 * Z + np.random.normal(0, 1, n)  # outcome depends on T and Z

# Naive estimate
print(f"NAIVE (no adjustment): {Y[T == 1].mean() - Y[T == 0].mean():+.3f}")
# Will be biased — overestimates due to Z confounding

# Adjusted via linear regression
import statsmodels.formula.api as smf
df_conf = pd.DataFrame({"T": T.astype(int), "Y": Y, "Z": Z})
m1 = smf.ols("Y ~ T + Z", data=df_conf).fit()
print(f"adjusted (correctly):  {m1.params['T']:+.3f}")
print(f"true effect:            +2.000\n")

# Re-seed for reproducibility.
np.random.seed(43)

# Collider: T → C, Y → C; adjusting C creates fake correlation
T2 = (np.random.normal(0, 1, n)) > 0
Y2 = np.random.normal(0, 1, n)
C = T2 + Y2 + np.random.normal(0, 0.5, n)   # both T2 and Y2 cause C

# Without conditioning on C: no correlation
df_coll = pd.DataFrame({"T2": T2.astype(int), "Y2": Y2, "C": C})
m2 = smf.ols("Y2 ~ T2", data=df_coll).fit()
print(f"Without adjustment for collider C: {m2.params['T2']:+.3f}")

# With conditioning on C: FAKE correlation appears
m3 = smf.ols("Y2 ~ T2 + C", data=df_coll).fit()
print(f"WITH adjustment for collider C:    {m3.params['T2']:+.3f}")
print("← Adjusting for the collider created a spurious effect")
```

You'll see:
- Confounder case: naive is biased; adjustment recovers truth
- Collider case: no adjustment gives null (correct); adjustment creates fake effect

**The wrong adjustment is worse than no adjustment.** Always draw the DAG first.

---

## Exercise 14.10 — 1-page A/B result template

```markdown
# A/B test result: Add product recommendation widget

## TL;DR
**Ship.** Recommendation widget lifts purchase conversion by **+2.3%** relative
(95% CI: +1.1% to +3.6%). Strong evidence; effect concentrated in mobile users.

## Experiment design
- **Hypothesis**: Recommendation widget → higher purchase conversion
- **Primary metric**: purchase rate within session
- **Randomization unit**: user
- **Duration**: 14 days (April 1-14, 2026)
- **Sample size**: 162k users (80k control, 82k treatment)
- **Pre-registered MDE**: 2% relative lift

## Results
| Metric | Control | Treatment | Lift | 95% CI | p-value |
|---|---|---|---|---|---|
| Purchase rate (primary) | 10.2% | 10.5% | **+2.3% rel** | [+1.1%, +3.6%] | 0.003 |
| Revenue per user | $4.20 | $4.30 | +2.4% rel | [+0.5%, +4.3%] | 0.02 |
| Time on site (guardrail) | 4.2 min | 4.3 min | +0.1 min | — | — |

## SRM check
80,124 / 81,991 split → p = 0.42. ✓ No SRM detected.

## Segments
| Segment | Lift | n_c | n_t |
|---|---|---|---|
| Mobile users | +4.1% | 45k | 46k |
| Desktop users | +0.5% | 35k | 36k |

Mobile effect drives the overall lift; desktop is flat.

## Caveats
- 2-week test; long-term effects (>30d) unknown
- Effect concentrated in mobile; verify before full desktop rollout
- Novelty effects partially controlled by 14-day duration

## Recommendation
Ship to all mobile users. Run a follow-up A/B for desktop tuning.
```

This is the artifact stakeholders need. Notebook code lives behind it; the page is the decision.

---

## Submission checklist

- [ ] Power analysis sample-size calculation
- [ ] Clean A/B simulation analyzed with CI, lift, p-value
- [ ] Peeking inflation simulation shows FPR > 5%
- [ ] SRM test implemented and demonstrated
- [ ] Segment analysis surfaces heterogeneity hidden in overall
- [ ] Bayesian A/B alternative computed
- [ ] PSM applied to observational data; recovers closer-to-true effect than naive
- [ ] DiD analysis on the city / pre-post simulation
- [ ] Confounder vs collider demonstration with statsmodels
- [ ] 1-page A/B result template written

---

## What you just did

You can design an A/B test with appropriate sample size, analyze it without committing the cardinal sins (peeking, SRM-ignoring, cherry-picking), and communicate the result in a way that lands. You can adjust observational data using PSM and DiD when randomization isn't an option, and you understand why colliders are not safe to adjust.

Week 15 takes the results and puts them in front of stakeholders — dashboards, visualizations, and the narrative that turns a chart into a decision.

---

**Next**: [Week 15: Dashboards + Storytelling →](../week-15-dashboards-and-storytelling/readme.md)
