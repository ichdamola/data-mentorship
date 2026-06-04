# Week 14: Theory — Causal Inference and A/B Testing

**Correlation is not causation.** You've heard it. You'll spend this week internalizing what to do instead.

"Users who used feature X had 20% higher retention" — was it the feature, or the kind of user who chose it? "Revenue went up after the redesign" — was it the redesign, or the holiday? "Customers in tier A churn less" — is tier causing retention, or are loyal customers self-selecting into tier A?

This week is the toolkit for actually answering "what caused what":

- **Randomized experiments (A/B tests)** — the gold standard
- **Observational adjustment** — when you can't randomize: propensity matching, IPW, diff-in-differences
- **The vocabulary** — confounder, mediator, collider, why mishandling each breaks your analysis

By the end you can design a defensible A/B test, read an A/B result honestly, and apply observational adjustment when randomization isn't possible — without falling for the standard mistakes.

---

## Part 1: The fundamental problem of causal inference

For any unit (user, order, store), there are two **potential outcomes**:

- `Y(1)`: what would have happened if treated
- `Y(0)`: what would have happened if not treated

The causal effect for that unit: `Y(1) - Y(0)`.

**You can only observe one of the two.** This is the fundamental problem. You can't roll the dice for the same person twice.

Causal inference is a collection of methods for **estimating the average causal effect across the population** despite this constraint.

Two strategies:

1. **Randomize** the treatment, so the two groups are statistically equivalent on everything else, then compare their averages. (A/B testing.)
2. **Adjust** observational data, modeling the differences between treated and untreated to mimic randomization.

Randomization is much more reliable. Adjustment is harder but sometimes the only option.

---

## Part 2: Random assignment is everything

In an A/B test, you flip a coin per user (or per session, or per device — the **randomization unit**) to decide treatment or control.

If randomization is real, the two groups have **the same distribution of every other variable**: age, geography, prior behavior, device type, everything observable AND unobservable. The only systematic difference is the treatment.

The average difference in outcomes between groups, then, is causal — caused by treatment.

This is why A/B tests are powerful: **they eliminate confounding by design**, not by guess. You don't need to know what variables matter; randomization handles all of them.

---

## Part 3: Designing an A/B test

The minimum-viable A/B design has six components:

1. **Hypothesis**: "Adding a recommendation widget increases purchase rate."
2. **Primary metric**: purchase rate (binary: did/didn't purchase)
3. **Randomization unit**: user (not session — users may revisit)
4. **MDE (minimum detectable effect)**: smallest effect worth knowing about, e.g., 2% relative lift
5. **Sample size**: from power analysis given MDE and current baseline
6. **Duration / stopping rule**: pre-specified, NOT "I'll stop when it's significant"

### Sample size calculation

```python
from statsmodels.stats.power import NormalIndPower
analysis = NormalIndPower()

baseline_conversion = 0.10
mde_relative = 0.02   # 2% relative lift → 0.10 → 0.102
treatment_conversion = baseline_conversion * (1 + mde_relative)

# Convert to effect size (Cohen's h for proportions)
import numpy as np
def cohens_h(p1, p2):
    return 2 * (np.arcsin(np.sqrt(p1)) - np.arcsin(np.sqrt(p2)))

effect_size = cohens_h(baseline_conversion, treatment_conversion)
n_per_group = analysis.solve_power(
    effect_size=effect_size, alpha=0.05, power=0.80, alternative="two-sided",
)
print(f"need {int(n_per_group):,} users per group")
```

For 2% relative lift on a 10% baseline: ~80k users per group. Your traffic determines how long that takes.

### Randomization unit choice

| Unit | When |
|---|---|
| **User** | The default; users return |
| **Session** | One-off interactions, no repeat visits |
| **Device** | Anonymous users; web without login |
| **Account / Company** | B2B; effects spread to all teammates |
| **Geography** | When randomization-by-user has network effects (e.g., marketplaces) |

The wrong unit produces wrong p-values. **Always randomize at the level where you'll measure independence.**

---

## Part 4: The cardinal A/B sins

The list of things that go wrong in real A/B tests:

### Peeking

You check the test daily; when it shows p < 0.05, you stop. Problem: the **probability of randomness producing p < 0.05 at some point during the test is vastly higher than 5%**. Naive p-values inflate from 5% to 25%+ in typical setups.

Fixes:

- **Don't peek** — pre-specify duration; analyze once at the end
- **Sequential testing** — methods (Sequential probability ratio, mSPRT, bayesian-with-priors) that explicitly correct for repeated looks
- **Group-sequential designs** — pre-specified interim analyses with adjusted thresholds

Most modern experimentation platforms (Optimizely, GrowthBook, Statsig, Eppo) handle this with sequential testing built in. **If you're rolling your own, learn the methods or pre-commit to a fixed window.**

### Sample Ratio Mismatch (SRM)

You assigned 50/50, but the data shows 47/53. **That's a bug** — your randomization didn't work as intended, or there's biased filtering downstream. ANY A/B test result with SRM is suspect; the difference might be entirely the SRM, not the treatment.

```python
from scipy.stats import binomtest
result = binomtest(k=control_count, n=control_count + treatment_count, p=0.5)
if result.pvalue < 0.001:
    print("⚠️ SAMPLE RATIO MISMATCH — debug randomization before trusting results")
```

Test for SRM **first** before analyzing the primary metric.

### Novelty / primacy effects

Users react to anything new ("hey, what's that?"); old habits stick ("ugh, where's the button?"). Both fade in ~2-4 weeks. **Run tests at least 2 weeks, ideally a full business cycle.**

### Cherry-picking metrics

You measured 5 metrics; one is significant. Multiple comparisons (week 13) — **pre-specify the primary metric** before launching.

### Spillover / network effects

A user in treatment influences a user in control (marketplaces, social networks, B2B teams). Standard A/B math fails. Use **geo-experiments** or **switchback testing** instead.

### Simpson's paradox lurking

The overall result is a fake summary of opposite-direction segment results. Always look at the top 3-5 segments separately.

---

## Part 5: Analyzing an A/B test

Once the data is in:

```python
# Compare conversion rates
n_c = len(control)
n_t = len(treatment)
conv_c = control["converted"].mean()
conv_t = treatment["converted"].mean()
lift_absolute = conv_t - conv_c
lift_relative = lift_absolute / conv_c

# Pooled standard error
p = (control["converted"].sum() + treatment["converted"].sum()) / (n_c + n_t)
se = np.sqrt(p * (1-p) * (1/n_c + 1/n_t))
z = lift_absolute / se
from scipy.stats import norm
p_value = 2 * (1 - norm.cdf(abs(z)))

# 95% CI on absolute lift
ci_lo = lift_absolute - 1.96 * se
ci_hi = lift_absolute + 1.96 * se

print(f"control conv: {conv_c:.4f}, treatment conv: {conv_t:.4f}")
print(f"absolute lift: {lift_absolute:+.4f} (95% CI: [{ci_lo:+.4f}, {ci_hi:+.4f}])")
print(f"relative lift: {lift_relative:+.2%}")
print(f"p-value: {p_value:.4f}")
```

The CI is more useful than the p-value. A CI of `[+1%, +3%]` is much more informative than "p = 0.02" — it tells you the effect could be small but is unlikely to be zero.

### Bayesian A/B

Frequentist A/B testing answers "is the effect non-zero?" Bayesian A/B answers **"what's the probability treatment is better?"** — usually more decision-useful.

```python
from scipy import stats
# Beta posteriors with uniform priors
posterior_c = stats.beta(1 + control["converted"].sum(), 1 + n_c - control["converted"].sum())
posterior_t = stats.beta(1 + treatment["converted"].sum(), 1 + n_t - treatment["converted"].sum())

# Probability treatment > control via Monte Carlo
samples_c = posterior_c.rvs(100_000)
samples_t = posterior_t.rvs(100_000)
prob_treatment_better = (samples_t > samples_c).mean()
print(f"P(treatment > control) = {prob_treatment_better:.2%}")
```

For small experiments where frequentist p-values mislead, Bayesian gives more honest answers. Modern platforms (Statsig, Eppo) often offer Bayesian analysis alongside frequentist.

---

## Part 6: Observational adjustment — when you can't randomize

Sometimes you can't run an A/B test:

- You're studying historical data
- Treatment can't ethically be randomized (you can't randomize people to smoke)
- The treatment was self-selected
- The intervention already happened

The toolkit for causal claims without randomization:

### Propensity Score Matching (PSM)

For each treated unit, find an untreated unit with similar covariates. Compare outcomes within matched pairs.

```
1. Fit a model: P(treated | X) — the propensity score
2. For each treated unit, find untreated unit(s) with similar propensity
3. Compute average outcome difference within matches
```

Works when:
- You have rich enough covariates to make treated and untreated look the same after matching
- Unobserved confounders are minimal

```python
from sklearn.linear_model import LogisticRegression
# Fit propensity
propensity_model = LogisticRegression().fit(X_covariates, treatment_indicator)
propensity_scores = propensity_model.predict_proba(X_covariates)[:, 1]
# Then match treated to untreated with similar scores
```

### Inverse Propensity Weighting (IPW)

Instead of matching, **reweight** the data so treated and untreated look balanced.

```python
weight = treatment / propensity_score + (1 - treatment) / (1 - propensity_score)
```

Each unit's contribution is inversely proportional to its assignment probability. Treated units in low-propensity strata get up-weighted; this rebalances.

IPW is more flexible than matching but can have high variance if propensities are extreme (near 0 or 1).

### Difference-in-Differences (DiD)

When you have a pre-period and a post-period for both treated and untreated:

```
DiD = (treated_post - treated_pre) - (control_post - control_pre)
```

Assumes the two groups would have followed parallel trends in the absence of treatment. Powerful when this is plausible (policy changes, geographic rollouts).

### Regression Discontinuity (RDD)

When treatment is assigned based on a threshold (income < $50k = qualify; otherwise not), compare units just above vs just below the threshold. Local randomization-ish.

### Synthetic Control

Build a synthetic "untreated" version of the treated unit by weighting other untreated units. Most useful for policy evaluation (one country gets policy; build a synthetic country from others).

### The library to know

**EconML** (Microsoft) and **DoWhy** (Microsoft) implement most of these in Python with a sensible API. **CausalPy** (PyMC) for Bayesian DiD and RDD.

For most analyst work: PSM and DiD cover 80% of needs. Master those two.

---

## Part 7: Confounders, mediators, colliders

The most important conceptual page in causal inference. Three relationships between a third variable Z and the treatment T → outcome Y arrow:

### Confounder: Z → T AND Z → Y

Z affects both. Not adjusting → biased estimate.

Example: ice cream sales (T) and shark attacks (Y). Confounder Z = summer.

**Always adjust for confounders.**

### Mediator: T → Z → Y

T affects Y through Z. Adjusting Z removes part of the causal effect.

Example: T = "click ad", Z = "visited landing page", Y = "purchased". Adjusting for Z would tell you "what's the effect of the ad NOT through the landing page" — usually not what you want.

**Don't adjust for mediators when estimating total effect.** Do adjust when decomposing direct vs indirect.

### Collider: T → Z AND Y → Z

Z is affected by both. Adjusting Z **creates a spurious correlation** between T and Y.

Example: T = talent, Y = beauty, Z = "is a successful actor." Among successful actors, talent and beauty are negatively correlated — because you needed at least one to succeed. Adjusting for "successful actor" creates a false negative correlation.

**Never adjust for colliders.**

### The decision

When in doubt: **draw the directed acyclic graph (DAG)**. Decide which arrows are causal. Adjust for confounders, leave mediators alone (or decompose), avoid colliders.

For a 30-min introduction: Judea Pearl's *The Book of Why*.

---

## Part 8: A/B test communication

Frame results in terms of:

| What | How |
|---|---|
| Effect size, not just p-value | "+2% lift (95% CI: [+0.5%, +3.5%])" |
| Practical significance | "Below our 5% threshold for deployment" |
| Segments | "Effect concentrated in iOS users; no effect on Android" |
| Confidence | "Strong evidence" (p < 0.001, large effect) vs "Suggestive" (p < 0.05, small effect) |
| What we're not claiming | "Did not measure long-term effects" |

The instinct of "p < 0.05 = ship it" is wrong. A statistically significant 0.1% lift in revenue is probably not worth the deployment cost. A non-significant 3% lift in a 50k sample might be worth re-testing.

---

## Part 9: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| Peeking and stopping when significant | False positive rate is 25%+, not 5% |
| Trusting frequentist A/B with SRM | Result might be entirely the bias |
| Adjusting for a collider in observational data | Creates fake effect; reports it as real |
| Reporting p-value without effect size | Statistical noise looks meaningful |
| Running 1-week A/B tests | Novelty effects haven't worn off |
| Not measuring guardrail metrics | You optimize one number; another breaks |
| Frequentist Bayesian frequentist switching | Pick one framework; commit |

---

## Part 10: Connect to the rest of the curriculum

- **Week 11 (features)**: Point-in-time correctness is causal reasoning applied to feature engineering
- **Week 13 (EDA)**: Bootstrap CIs and permutation tests are the statistical foundation
- **Week 15 (dashboards)**: A/B results often go on a dashboard; the visualization rules matter
- **ml-mentorship**: Causal ML is a deeper rabbit hole; this week is the practical analyst level

---

## What's next

In [lab.md](lab.md) you'll:

- Design and analyze a simulated A/B test end-to-end
- Detect SRM and what it does to results
- Demonstrate peeking inflation via simulation
- Run propensity score matching on observational data
- Build a difference-in-differences analysis
- Identify confounders, mediators, colliders in a worked example
- Write the A/B test result the way it should be communicated

By end of week 14 you can run defensible experiments and adjust observational data when randomization isn't possible — without committing the standard sins.
