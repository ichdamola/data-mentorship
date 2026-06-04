# Week 13: Theory — EDA and Statistical Foundations

You've cleaned data (04-08), enriched it (09-12), and now someone wants insights. This is where most data work breaks down: **the analyst opens a notebook, runs `df.describe()`, makes a histogram, and ships a number.** That number is wrong as often as not — distributions misread, confidence intervals ignored, Simpson's paradox lurking.

This week is **EDA done seriously** — the moves that separate the engineers who know what their data says from the ones who guess. Plus the statistical foundations to back honest claims: bootstrap (when parametric assumptions fail), permutation tests (when t-tests don't apply), and the common sins (p-hacking, multiple comparisons, Simpson's paradox).

---

## Part 1: What EDA is for

Three goals, in order of importance:

1. **Understand the data well enough to ask the right question.** Most "the model is wrong" diagnoses turn out to be "the question was wrong."
2. **Detect quality issues before they show up downstream.** Reuses everything from week 04.
3. **Build hypotheses worth testing.** EDA generates; later analysis confirms.

The classic Tukey framing: **"Far better an approximate answer to the right question than an exact answer to the wrong question."** EDA is about getting to the right question.

What EDA isn't:

- A statistical test (those come *after* EDA)
- A causal claim ("X caused Y" — that's week 14)
- A production deliverable on its own ("here's a chart" without the question or method is a story, not analysis)

---

## Part 2: The first-hour scan

For any new dataset, an opening scan should answer:

| Question | How |
|---|---|
| How big? | `len(df)`, file size, time span |
| What columns and types? | `df.dtypes`, `df.schema` |
| How clean? | null % per column (week 04) |
| What's the grain? | What does one row represent? |
| What distributions? | Histogram per numeric column; bar chart per categorical |
| What relationships? | Correlation matrix; scatter matrix |
| What's weird? | Outliers; high-null columns; suspicious categoricals |

For DuckDB on Parquet:

```sql
SUMMARIZE FROM read_parquet('data/silver/orders.parquet');
```

For Polars on anything:

```python
df.describe()                   # numeric summary
df.null_count()                  # nulls per column
df["category"].value_counts()    # top categoricals
```

The first hour produces **questions, not answers**. Write them down.

---

## Part 3: Distribution shapes — read them visually

`mean` and `std` are summaries; **the histogram is the truth**. Five shapes to recognize:

| Shape | Recognize by | Implications |
|---|---|---|
| **Normal (Gaussian)** | Bell-shaped, symmetric | Mean = median = mode; t-tests work; std is a meaningful spread |
| **Log-normal** | Right-skewed; log(x) is normal | Income, prices, time-to-event; report median + IQR |
| **Bimodal** | Two peaks | Two underlying populations mixed together; **stratify** |
| **Uniform** | Flat | Often a generated / synthetic field; or a constraint |
| **Heavy-tailed (power law)** | Most rows small; tail of huge values | Network degrees, file sizes, popularity; **mean is misleading; median + percentiles** |

Plot the **log scale** when in doubt; many "normal-looking" distributions are actually log-normal viewed on a linear scale.

### The Q-Q plot

When you need to know "is this *actually* normal?" — a quantile-quantile plot is faster than a Shapiro-Wilk test and more interpretable:

```python
import scipy.stats as stats
stats.probplot(x, dist="norm", plot=plt)
```

Points on the diagonal → normal. Curve away from the line → tells you exactly how it deviates (heavy tails, skew).

---

## Part 4: The bootstrap — confidence intervals when assumptions fail

For most parametric tests (t-test, ANOVA, linear regression CIs), the assumption is: **the sampling distribution of the statistic is approximately normal**. This is often false in practice — heavy tails, small samples, non-linear estimators.

The bootstrap fixes this in a way that works for almost any statistic:

```
1. Sample N rows from the data WITH REPLACEMENT
2. Compute the statistic of interest (mean, median, correlation, whatever)
3. Repeat 10,000 times → distribution of the statistic
4. Confidence interval = percentiles of that distribution (e.g., 2.5th and 97.5th for 95% CI)
```

```python
import numpy as np

def bootstrap_ci(data, statistic, n_iter=10_000, alpha=0.05):
    """Returns (point_estimate, lower, upper) for a 1-alpha CI."""
    boot_estimates = [
        statistic(np.random.choice(data, size=len(data), replace=True))
        for _ in range(n_iter)
    ]
    return statistic(data), np.percentile(boot_estimates, alpha/2 * 100), np.percentile(boot_estimates, (1-alpha/2) * 100)
```

For a 95% CI on the median of a heavy-tailed distribution where a t-test would be wrong: bootstrap is right.

**Computational cost**: 10k iterations × a few microseconds each → seconds. Worth it for any non-trivial estimate you're going to report.

Efron's 1979 paper introduced this. It's one of the most consequential statistical methods of the modern era because it democratizes valid inference.

---

## Part 5: Permutation tests — non-parametric hypothesis testing

When you want to test "is the difference between group A and group B significant?" without assuming normality:

```
1. Compute the observed difference: diff_observed = mean(A) - mean(B)
2. Pool A and B together
3. Randomly shuffle the labels (which row is A, which is B)
4. Compute the difference with shuffled labels
5. Repeat 10,000 times
6. p-value = fraction of shuffles where |shuffled_diff| >= |observed_diff|
```

```python
def permutation_test(a, b, n_iter=10_000):
    observed = np.mean(a) - np.mean(b)
    combined = np.concatenate([a, b])
    n_a = len(a)
    count_extreme = 0
    for _ in range(n_iter):
        np.random.shuffle(combined)
        shuffled_diff = np.mean(combined[:n_a]) - np.mean(combined[n_a:])
        if abs(shuffled_diff) >= abs(observed):
            count_extreme += 1
    return count_extreme / n_iter
```

**Distribution-free**. Works for any statistic, any sample size, any underlying distribution. The standard t-test relies on assumptions; the permutation test relies on the data.

For 2026 work: **use permutation tests as the default for any "is A different from B" question**. Parametric tests are a backup when the data is huge and the parametric assumptions clearly hold.

---

## Part 6: The common statistical sins

The patterns to recognize (and avoid):

### p-hacking

You ran 10 tests; the 1 that came up significant at p < 0.05 is "the result." Statistically: you'd expect ~0.5 false positives among 10 tests by chance alone.

The fix: **pre-register** what you're testing before looking. Or correct for multiple comparisons.

### Multiple comparisons

Running 20 t-tests at α=0.05: probability of at least one false positive ≈ 1 - 0.95²⁰ ≈ 64%.

Corrections:
- **Bonferroni** (conservative): α_each = α / n_tests
- **Benjamini-Hochberg** (less conservative, controls FDR): more involved; standard for genomics

In practice for analytics work: if you're hunting through 20 categoricals for "significant" effects, you're not testing hypotheses; you're prospecting. **Treat findings as hypotheses for the next experiment**, not as conclusions.

### Simpson's paradox

A relationship reverses when you stratify by a third variable:

```
Overall:      Treatment A is better than Treatment B
Stratified:   Treatment A is worse than B in subgroup X
              Treatment A is worse than B in subgroup Y
```

This happens when the subgroups have very different baseline rates AND the treatments are unevenly distributed across them. The classic example: UC Berkeley admissions in the 1970s — overall favored men, but within each department, favored women.

The fix:

- **Always stratify** when you have a strong demographic / segment / region variable
- **Look at subgroup conditional means**, not just overall
- **Suspect Simpson's** whenever the overall result conflicts with intuition

### Selection bias

Your sample isn't representative of the population you're trying to make claims about. Classic: surveying customers about your product (your unhappy customers churned and aren't there to respond).

The fix: **think carefully about who's in your sample and who isn't.** Often there's no fix; just label the bias explicitly.

### Regression to the mean

If you select on a high or low value, the next observation is closer to the mean. "The top 10% of last month's reps are doing worse this month!" — they were probably lucky last month.

The fix: **select once, measure independently.** Or use difference-in-differences (week 14).

---

## Part 7: Effect size — beyond p-values

A p-value answers "is the effect zero?" — almost always, no. At enough sample size, anything is significantly non-zero.

The better question: **how big is the effect?**

| Effect size measure | Use |
|---|---|
| Cohen's d | Two-group difference in std-units |
| % difference | Business interpretable; "20% lift" |
| Confidence interval | "Lift is 18-22%, with 95% confidence" |
| Practical significance threshold | "Below 5% lift is below our deployment threshold" |

Report effect size **always**. Report p-value sometimes. The latter without the former is meaningless.

---

## Part 8: Power and sample size

If your test has low power, you'll miss real effects. If you only have 30 samples per group, you can probably detect a 50% lift, not a 5% lift.

Power is "probability of detecting an effect of size X if one exists." Standard target: 80%.

```python
from statsmodels.stats.power import TTestIndPower
analysis = TTestIndPower()
sample_size = analysis.solve_power(effect_size=0.2, alpha=0.05, power=0.8, alternative="two-sided")
# Sample size needed PER GROUP to detect a Cohen's d = 0.2 effect at 80% power
```

For A/B testing (week 14), power analysis sizes the experiment **before** launching. For exploratory EDA, power analysis tells you which findings you should trust (high-power) vs treat as hypotheses (low-power).

---

## Part 9: Visual conventions that matter

The visualizations that age well:

- **Axes start at 0** for bar charts (always). For line charts: depends on whether the question is about magnitude or change.
- **Show the underlying data** when reasonable: scatter > smoothed line; box plot > only-the-mean
- **Label units explicitly**: "$10K" not "10K"; "%" not "0.1"
- **Color signals meaning**: ordinal data → sequential colormap; categorical → distinct hues
- **Avoid 3D**: 3D charts hide data and confuse comparison

For week 15 we'll build dashboards. The visual rules in this part are the foundation.

---

## Part 10: When to stop

EDA can go forever. Stop when:

- The question is answered (with appropriate uncertainty)
- The next question requires a real experiment, not more inspection
- You've found a quality issue that needs upstream fixing first
- You've spent your time budget (set one before starting)

A common failure: **the analyst keeps exploring**, the stakeholder keeps waiting, both lose the thread. Set a 1-day or 2-day EDA budget; produce a 1-page summary; iterate.

---

## Part 11: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| `df.describe()` only; never plotting | Mean / std lie about non-Gaussian distributions |
| Using t-test on heavily skewed data | Wrong p-values |
| Computing CIs without stating the method | "± something" — what something? |
| Reporting p-value without effect size | Statistical noise looks meaningful |
| Stratification ignored | Simpson's paradox unnoticed |
| "Eyeballing" significance from charts | Even experts are fooled by random noise |
| Publishing the first finding without checks | Most published findings don't replicate |

---

## Part 12: Connect to the rest of the curriculum

- **Week 04 (quality)**: EDA reuses profiling tools
- **Week 06 (missing data)**: missingness analysis is part of every EDA
- **Week 11 (features)**: EDA before feature engineering catches data issues
- **Week 14 (causal/A/B)**: EDA generates the hypotheses; testing confirms

---

## What's next

In [lab.md](lab.md) you'll:

- Run a full first-hour EDA on a real dataset
- Diagnose distribution shapes via histograms + Q-Q plots
- Compute bootstrap CIs for non-normal estimates
- Run a permutation test
- Demonstrate Simpson's paradox with a worked example
- Build a 1-page EDA summary template

By end of week 13 you can pick up any new dataset and produce an honest, defensible understanding of what it says — without the common analytic sins.
