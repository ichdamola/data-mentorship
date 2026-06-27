# Week 15: Theory - Dashboards and Storytelling

A dashboard that ships isn't a feat of plotting. It's a feat of **judgment about what to show**. Most dashboards fail not because they're ugly but because they show too much, don't answer a clear question, and bury the headline.

This week is the BI side: hands-on **Metabase** and **Superset**, the **visualization rules** that work, the **rule of 5**, the **headline + bullets pattern**, and how to write the **half-page narrative** that turns a chart into a decision.

---

## Part 1: What a dashboard is for

Three honest goals, in priority order:

1. **Answer one or two questions** clearly and quickly
2. **Surface anomalies** that need attention
3. **Let the curious explore** without breaking the answer above

Most dashboards skip 1 and 2, going straight to 3 - "let people explore." The result: 50 charts, no answer, nobody uses it.

The senior framing: **a dashboard exists to make a decision.** What decision, by whom, how often? If you can't answer that, you're not designing a dashboard; you're decorating a notebook.

---

## Part 2: Categories of dashboard

Four common archetypes, with different design constraints:

| Type | Purpose | Update freq | Example |
|---|---|---|---|
| **Executive** | High-level performance; trend over time | Daily/weekly | KPI scorecard |
| **Operational** | Real-time monitoring; alerts | Minutes | Order pipeline status |
| **Analytical** | Deep-dive exploration | On demand | Ad-hoc cohort analysis |
| **Reporting** | Periodic snapshot for stakeholders | Weekly/monthly | Board pack |

Don't mix. An executive dashboard is not the place to put a 50-row drill-down. An analytical workbench shouldn't try to be an exec scorecard.

---

## Part 3: The rule of 5

**Show no more than 5 things on a single page.** Five charts, five rows, five KPIs.

Why? Human working memory holds 5-7 items. A dashboard with 20 charts isn't surfacing information; it's burying it. Stakeholders skim the top, see "a lot of stuff," and leave with a vague impression.

Better: 5 carefully-chosen, large, well-labeled charts. The user can scan them in 30 seconds and walk away knowing the state of the business.

For deeper drill-down, **link to a separate page** - don't cram it on the main view.

---

## Part 4: The headline + bullets pattern

Every dashboard should open with text:

```
=== Q2 PERFORMANCE - JUNE 2026 ===

📈 Revenue: $4.2M (+8% vs Q1, +12% vs Q2 last year)

· iOS users drove ~60% of growth
· Enterprise plan up 22%; Free flat
· Churn ticked up to 4.1% (vs 3.6% target)
```

Two sentences and three bullets above the charts. Reasons:

1. **Saves time** - a stakeholder reads 3 lines and has the gist
2. **Sets context** - the charts below confirm or detail; they don't surprise
3. **Forces the dashboard author to pick a headline** - the discipline of "if this dashboard had one sentence, what would it be?" is the entire design exercise

Tools to use:
- **Metabase**: native text cards
- **Superset**: markdown components
- **Streamlit / Dash**: `st.markdown(...)`
- **Notion / Confluence pages** with embedded queries

---

## Part 5: Visualization rules

The chart-type-to-question matching:

| Question | Best chart |
|---|---|
| What's the trend over time? | Line chart |
| How do groups compare? | Bar chart (horizontal if many) |
| What's the distribution of one variable? | Histogram |
| Two variables - relationship? | Scatter plot |
| Proportion of a whole | Stacked bar (NOT pie) |
| Geographic distribution | Map / choropleth |
| Sequential / hierarchical | Sankey / tree |
| Pivot / heatmap of metric | Heatmap |

Mistakes to avoid:

- **Pie charts >5 slices**: humans can't compare areas; use horizontal bar
- **3D anything**: hides data; confuses perspective
- **Dual-axis charts**: silently lies; readers can't compare
- **Rainbow color scales**: misrepresents ordering
- **Tiny multiples without alignment**: makes comparison impossible

### Pre-attentive attributes

Some visual properties pop instantly without conscious effort:
- Position
- Size
- Color (hue, saturation)
- Orientation

Use them for what's important. Don't use them for decoration. **The big red number is your headline; everything else is supporting cast.**

---

## Part 6: The dashboard layout

A working layout for an executive dashboard:

```
┌───────────────────────────────────────────────────────────┐
│  HEADLINE - one sentence, big font                         │
│  3-4 bullets - quick context                               │
└───────────────────────────────────────────────────────────┘
┌─────────────────┬─────────────────┬─────────────────────┐
│  KPI 1          │  KPI 2          │  KPI 3              │
│  (big number)   │  (big number)   │  (big number)       │
│  trend sparkline │  trend sparkline │  trend sparkline   │
└─────────────────┴─────────────────┴─────────────────────┘
┌─────────────────────────┬──────────────────────────────┐
│  Trend chart (revenue)   │  Top categories bar         │
│  monthly, last 12 months │  this period                │
└─────────────────────────┴──────────────────────────────┘
┌───────────────────────────────────────────────────────────┐
│  Geographic / segment breakdown                            │
└───────────────────────────────────────────────────────────┘
```

Top to bottom: **narrative → numbers → details**. The eye scans this naturally.

---

## Part 7: Color, scale, and respecting your audience

### Color

| Use | Choice |
|---|---|
| Categorical | Distinct, color-blind-friendly (viridis, tab10) |
| Sequential (ordered) | Single-hue gradient (Blues, Greens) |
| Diverging (around a center) | Red-blue or similar diverging palette |
| Highlight | One bright color against grayed-out background |

The **Color Brewer** palettes are good defaults. Avoid traffic-light red/yellow/green for categorical (it adds spurious ranking).

### Scale

Bar chart Y-axis: **starts at 0**. Always. Otherwise small differences look huge.

Line chart Y-axis: **starts at 0 if magnitude matters; starts at min value if change matters**. Stock charts don't start at 0; CO2 charts shouldn't either. State the choice.

Log scale: useful when range is orders of magnitude. **Label clearly** - readers default to linear.

### Annotations

When a number is the headline, annotate it directly on the chart. Don't make the reader look at a legend, then back at the chart, then back at the legend.

```
Revenue: $4.2M (▲8% vs Q1)
```

Inline, large, clear.

---

## Part 8: Metabase vs Superset vs Tableau vs Streamlit

| Tool | Strengths | When |
|---|---|---|
| **Metabase** | Easy to install (Docker); auto-detects schemas; great for non-technical users | Small-to-medium teams starting out |
| **Superset** | More flexible; SQL Lab; lots of viz types | Larger teams; willing to invest in setup |
| **Tableau / Power BI** | Polished commercial; enterprise integrations | Existing org investment |
| **Metabase Cloud / Hex / Looker** | Hosted; SOC2 / governance | Compliance / scale |
| **Streamlit / Dash / Panel** | Python-first; great for embedded apps | When you need code-driven custom views |
| **Notebook + matplotlib** | Total control | One-off analyses; portfolio pieces |

For 2026: **Metabase or Superset for stakeholder dashboards; Streamlit for embedded data apps; matplotlib + Markdown for one-off reports.**

---

## Part 9: Dashboard sprawl

The natural decay path:

```
Week 1:  3 charts, 1 stakeholder, 100% engagement
Week 6:  10 charts, 2 stakeholders, "can you add this?"
Month 4: 30 charts, 4 stakeholders, nobody reads it
Year 1: 100 charts, undocumented, broken queries, "we have a dashboard for that"
```

Prevention:

- **Date columns** on every chart's title: "last updated 2026-04-15"
- **Owner per chart**: contact for questions
- **Automatic decay**: if no one queries a chart for 90 days, archive it
- **Periodic audit**: every quarter, walk through every dashboard with stakeholders; kill what's not useful

---

## Part 10: Storytelling - the dashboard isn't the deliverable

For high-stakes findings, the dashboard is the **evidence**. The **narrative** is the deliverable.

Format that works:

```
1. The question (one sentence)
2. The answer (one sentence)
3. The reasoning (3 bullets)
4. The dashboard / chart (link or embed)
5. What you can't conclude (caveats)
6. What you recommend
```

A 5-paragraph memo + a dashboard. The dashboard alone is decorative; the narrative + dashboard is decision-ready.

Cole Nussbaumer Knaflic's *Storytelling with Data* is the canonical resource. The Edward Tufte books are the older / more academic reference.

---

## Part 11: When a dashboard isn't the right deliverable

Other options:

| Need | Better than a dashboard |
|---|---|
| One-off answer | Markdown memo with embedded chart |
| Recurring report | Automated email / Slack digest |
| Alerting | Monitor + page on threshold |
| Exploration by a few specific analysts | Notebook share; or SQL lab |
| Public-facing share | Static HTML or PDF |
| Embedded in another app | Streamlit / API + custom UI |

Default to a dashboard is lazy. Pick the right form factor for the question.

---

## Part 12: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| Dashboard with 30 charts | Nobody knows what to look at |
| No update timestamp | Readers don't know if data is fresh |
| No clear owner / contact | Questions go unanswered |
| Pie charts everywhere | Unreadable; comparison hard |
| Y-axis doesn't start at 0 | Bars lie about magnitude |
| Default Tableau / Metabase colors with no thought | Bad color choices |
| Dashboard with no headline | Reader has to figure out the takeaway |
| Saving the same chart to 5 different dashboards | Maintenance nightmare |

---

## Part 13: Connect to the rest of the curriculum

- **Week 04 (quality)**: Dashboard data quality directly visible to stakeholders; failures are public
- **Week 13 (EDA)**: Dashboard prep is EDA at the visualization layer
- **Week 14 (A/B)**: A/B results often land on dashboards; the visualization rules matter
- **Week 16 (pipelines)**: Dashboards consume from gold-layer tables; production pipeline serves them

---

## What's next

In [lab.md](lab.md) you'll:

- Spin up Metabase against your DuckDB warehouse in under 30 minutes
- Build the headline + 5-chart dashboard for the NYC taxi data
- Iterate with 3 (made-up) stakeholder requests
- Critique 3 published dashboards
- Compare to the same layout in Streamlit (code-driven option)
- Build a 1-page memo + embedded chart deliverable

By end of week 15 you can stand up a stakeholder-grade dashboard and resist the natural pull toward dashboard sprawl.
