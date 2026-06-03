# Week 15: Dashboards and Storytelling

## 🎯 What you'll learn

A dashboard that ships isn't a feat of plotting. It's a feat of **judgment about what to show**. This week is the BI side: Metabase and Superset hands-on, visualization rules that work, and how to write the **half-page narrative** that turns a chart into a decision.

By the end of this week you'll be able to:

- Spin up **Metabase** or **Superset** against your DuckDB warehouse in under 30 minutes
- Apply the visualization rules: **pre-attentive attributes, chart-type-to-question matching, the rule of 5**
- Avoid the canonical sins: dual-axis charts that lie, 3D pies, rainbow color scales, axes that don't start at zero (or that should)
- Write the **2-line headline + 3-bullet narrative** every dashboard needs at the top
- Build a self-serve dashboard the team uses — not one that decays
- Reason about **dashboard sprawl**: 200 charts nobody reads ≠ value

## 🧰 Lab setup

Docker for Metabase + Superset:

```bash
# Metabase
docker run -d -p 3000:3000 --name metabase metabase/metabase
# Superset
docker run -d -p 8088:8088 --name superset apache/superset
```

Or use **Streamlit** for code-driven dashboards:

```bash
uv add streamlit plotly altair
```

## ✅ Your job

1. Read [theory.md](theory.md). The "rule of 5" and the "headline + bullets" pattern are the takeaways.
2. Work through [lab.md](lab.md). Build a dashboard on the NYC taxi data; iterate to satisfy three (made-up) stakeholder requests.
3. Critique three published dashboards and identify what they're each doing wrong.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Cole Nussbaumer Knaflic — Storytelling with Data](https://www.storytellingwithdata.com/) | The reference | 2 hours skim |
| [Few — Information Dashboard Design](https://www.amazon.com/Information-Dashboard-Design-At-Glance/dp/1938377001) | The canonical book | 90 min skim |
| [Metabase Learn](https://www.metabase.com/learn/) | Vendor docs that teach the concept | 60 min |
| [Edward Tufte — The Visual Display of Quantitative Information](https://www.edwardtufte.com/tufte/books_vdqi) | The classic | as much as you want |

## 💡 What you should already know

- Weeks 01-14

---

> 🚧 **Scaffolded.** Theory + lab fully fleshed in the next pass.

**Next**: [Week 16: Production Pipelines →](../week-16-production-pipelines/readme.md)
