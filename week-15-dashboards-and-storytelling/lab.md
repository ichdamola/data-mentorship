# Week 15: Lab - Stand Up a Dashboard

You'll spin up Metabase (or Superset, your choice) against DuckDB, build a clean 5-chart executive dashboard for the NYC taxi data, iterate on three "stakeholder asks," critique three published dashboards, then build the same layout in Streamlit as a code-driven alternative.

## Setup

We'll use **Metabase** because it's the lightest to install. Superset is mentioned where the workflow differs.

```bash
# Pull and run Metabase
docker run -d -p 3000:3000 --name metabase metabase/metabase
```

Open `http://localhost:3000`. Follow the setup wizard (admin email, password).

If you prefer code-driven dashboards, also install Streamlit:

```bash
uv add streamlit plotly
```

For the data source we'll use a DuckDB file:

```python
import duckdb
con = duckdb.connect("data/taxi.duckdb")
con.execute("""
    CREATE OR REPLACE TABLE trips AS
    SELECT *
    FROM read_parquet('data/yellow_tripdata_2024-01.parquet')
    WHERE tpep_pickup_datetime BETWEEN '2024-01-01' AND '2024-02-01'
      AND fare_amount BETWEEN 0 AND 500
      AND trip_distance BETWEEN 0 AND 100
""")
con.execute("SELECT COUNT(*) FROM trips").fetchone()
```

You should see ~3M rows in `data/taxi.duckdb`.

---

## Exercise 15.1 - Connect Metabase to DuckDB

Metabase doesn't ship native DuckDB support; you have two options:

**Option A (simplest)**: export your DuckDB tables to Parquet, load into PostgreSQL/SQLite, point Metabase at that.

**Option B (DuckDB plugin)**: install the [metabase-duckdb plugin](https://github.com/AlexR2D2/metabase_duckdb_driver) - drop the JAR in Metabase's `plugins/` folder, restart, connect.

For the lab, the cleanest path is Option A with SQLite:

```python
import sqlite3
import duckdb

con_duck = duckdb.connect("data/taxi.duckdb")
con_sql = sqlite3.connect("data/taxi.sqlite")

# Export a manageable summary table
con_duck.execute("""
    CREATE OR REPLACE TABLE daily_summary AS
    SELECT
        DATE(tpep_pickup_datetime) AS day,
        COUNT(*) AS trips,
        SUM(total_amount) AS revenue,
        AVG(trip_distance) AS avg_distance,
        AVG(total_amount) AS avg_fare,
        SUM(CASE WHEN payment_type = 1 THEN tip_amount ELSE 0 END) AS total_tips,
        SUM(CASE WHEN payment_type = 1 THEN 1 ELSE 0 END) AS cc_trips
    FROM trips
    GROUP BY day
    ORDER BY day
""")

# Export to SQLite
con_duck.execute("INSTALL sqlite; LOAD sqlite;")
con_duck.execute("ATTACH 'data/taxi.sqlite' AS sqlitedb (TYPE SQLITE);")
con_duck.execute("CREATE TABLE sqlitedb.daily_summary AS SELECT * FROM daily_summary")
con_duck.execute("DETACH sqlitedb")
print("exported to data/taxi.sqlite")
```

In Metabase: **Admin → Databases → Add database → SQLite**, point at `data/taxi.sqlite`. Save.

---

## Exercise 15.2 - Build the 5-chart dashboard

Plan: **a "January 2024 Taxi Performance" dashboard**. Five charts, headline, bullets.

Build these in Metabase via "+ Ask a question":

### 1. Headline KPI: total revenue + trip count

```sql
SELECT SUM(revenue) AS total_revenue, SUM(trips) AS total_trips
FROM daily_summary
```

Display: number, big font. Two side-by-side cards.

### 2. Revenue by day (line chart)

```sql
SELECT day, revenue FROM daily_summary ORDER BY day
```

Display: line chart.

### 3. Trip volume by day (line chart)

```sql
SELECT day, trips FROM daily_summary ORDER BY day
```

Display: line chart. Y-axis starts at 0.

### 4. Average fare trend

```sql
SELECT day, avg_fare FROM daily_summary ORDER BY day
```

Display: line chart.

### 5. Day-of-week breakdown

```sql
SELECT
    CASE strftime('%w', day)
        WHEN '0' THEN 'Sun' WHEN '1' THEN 'Mon' WHEN '2' THEN 'Tue'
        WHEN '3' THEN 'Wed' WHEN '4' THEN 'Thu' WHEN '5' THEN 'Fri'
        WHEN '6' THEN 'Sat'
    END AS dow,
    AVG(trips) AS avg_trips,
    AVG(revenue) AS avg_revenue
FROM daily_summary
GROUP BY strftime('%w', day)
ORDER BY strftime('%w', day)
```

Display: horizontal bar chart.

### Dashboard composition

In Metabase: **+ New → Dashboard**. Add a text card at the top:

```markdown
# January 2024 Taxi Performance

**Headline**: 3M trips generating $48M revenue (+5% vs December estimated).

- Weekday volumes peak Wed-Fri; weekend is slowest
- Avg fare stable at ~$16/trip across the month
- One drop on Jan 7 - investigate (snowstorm?)
```

Then drag the 5 charts in below. Set the layout: 2 KPIs across the top, 3 charts in a row below. Save.

Open the dashboard. Read the top in 5 seconds. **Did you get the gist? If not, the headline isn't sharp enough.**

---

## Exercise 15.3 - Iterate on stakeholder asks

A stakeholder Slack messages you three asks. Handle each.

### Ask 1: "Can we add a card showing the busiest day?"

You: ✅ valid - adds context. SQL:

```sql
SELECT day, trips FROM daily_summary ORDER BY trips DESC LIMIT 1
```

Add as a "biggest day" number card. Small text annotation: "Busiest day was Friday Jan 19 with 110k trips."

### Ask 2: "Can we add a borough breakdown? And trip-type pie chart?"

You: borough breakdown is reasonable. **Pie chart NO** - replace with horizontal bar.

```sql
-- Borough breakdown (assuming we joined PULocationID to a borough table)
-- For now, use payment type breakdown as proxy
SELECT
    CASE payment_type
        WHEN 1 THEN 'Credit card'
        WHEN 2 THEN 'Cash'
        WHEN 3 THEN 'No charge'
        WHEN 4 THEN 'Dispute'
        ELSE 'Other'
    END AS payment_method,
    COUNT(*) AS trips,
    SUM(total_amount) AS revenue
FROM trips
GROUP BY payment_type
ORDER BY trips DESC
```

Add a horizontal bar chart. Push back politely on the pie request: "Bar chart makes comparison easier - humans can compare bar lengths but struggle with pie slice areas."

### Ask 3: "Can we add 20 more dimensions for drill-down?"

You: **NO** - propose linking to a separate "deep-dive" dashboard.

Build a second dashboard "Taxi - Deep Dive" with the drill-downs they want. Link to it from the main dashboard via text card. Keep the main dashboard at 5-6 charts.

**Push back on bloat. The dashboard's job is to make a decision possible.**

---

## Exercise 15.4 - Critique published dashboards

Pick 3 publicly-viewable dashboards. For each, write 3 lines:

- **What it does well**
- **What it does poorly**
- **What you'd change**

Good sources:
- Public state COVID dashboards (varied quality)
- Strava heatmaps
- Federal Reserve FRED dashboards
- COVID Tracking Project archives
- Tableau Public's featured dashboards

Example critique:

> **NYC TLC public dashboard**: Does well - clear navigation, embedded narrative. Poorly - bar charts with non-zero Y, pie charts with 8 slices, no last-updated timestamp. I'd: replace pies with horizontal bars, add timestamps, simplify color palette.

The discipline of explicit critique sharpens your own design instincts.

---

## Exercise 15.5 - Streamlit alternative

For code-driven dashboards, build the same view in Streamlit.

```python
# app.py
import streamlit as st
import duckdb
import plotly.express as px
import pandas as pd

st.set_page_config(page_title="NYC Taxi - Jan 2024", layout="wide")

con = duckdb.connect("data/taxi.duckdb", read_only=True)

# Headline
st.title("January 2024 Taxi Performance")
st.markdown("""
**Headline**: 3M trips generating $48M revenue.

- Weekday volumes peak Wed-Fri; weekend is slowest
- Avg fare stable at ~$16/trip across the month
- One drop on Jan 7 - investigate (snowstorm?)
""")

# KPIs
total = con.sql("SELECT SUM(revenue) AS r, SUM(trips) AS t FROM daily_summary").fetchone()
col1, col2 = st.columns(2)
col1.metric("Total Revenue", f"${total[0]/1e6:.1f}M")
col2.metric("Total Trips", f"{total[1]/1e6:.2f}M")

# Charts
daily = con.sql("SELECT * FROM daily_summary ORDER BY day").pl().to_pandas()

col1, col2 = st.columns(2)
with col1:
    st.subheader("Daily revenue")
    fig = px.line(daily, x="day", y="revenue")
    fig.update_yaxes(range=[0, daily["revenue"].max() * 1.1])  # start at 0
    st.plotly_chart(fig, use_container_width=True)

with col2:
    st.subheader("Daily trips")
    fig = px.line(daily, x="day", y="trips")
    fig.update_yaxes(range=[0, daily["trips"].max() * 1.1])
    st.plotly_chart(fig, use_container_width=True)

# Dow
dow_data = con.sql("""
    SELECT
        CASE EXTRACT(dow FROM day)
            WHEN 0 THEN 'Sun' WHEN 1 THEN 'Mon' WHEN 2 THEN 'Tue'
            WHEN 3 THEN 'Wed' WHEN 4 THEN 'Thu' WHEN 5 THEN 'Fri'
            WHEN 6 THEN 'Sat'
        END AS dow_label,
        EXTRACT(dow FROM day) AS dow_n,
        AVG(trips) AS avg_trips
    FROM daily_summary
    GROUP BY dow_label, dow_n
    ORDER BY dow_n
""").pl().to_pandas()

st.subheader("Avg trips by day-of-week")
fig = px.bar(dow_data, x="dow_label", y="avg_trips")
fig.update_yaxes(range=[0, dow_data["avg_trips"].max() * 1.1])
st.plotly_chart(fig, use_container_width=True)

st.markdown("---")
st.caption(f"Data: NYC TLC Yellow Taxi Jan 2024. Last updated: 2026-06-03")
```

Run:

```bash
streamlit run app.py
```

You should see your dashboard at http://localhost:8501. Code-driven dashboards win when you need custom logic, multi-page navigation, or embeds.

---

## Exercise 15.6 - Build a memo (the senior deliverable)

Sometimes the dashboard isn't the answer; a memo is. Write a 1-page Markdown memo for the question "should we invest in increasing weekend taxi service?"

```markdown
# Memo: Weekend taxi service expansion

**Date**: 2026-06-03
**Author**: data-mentorship-lab
**Audience**: Ops VP

## TL;DR
**Yes, modestly.** Weekend revenue per shift is ~15% lower than weekday, suggesting under-supply. Expansion to 20% more weekend driver hours would likely capture $2M/year in additional revenue at NYC scale. Risk: oversupply on Sun mornings, where current utilization is already low.

## Question
Should NYC increase weekend driver supply to capture demand?

## Evidence
- Trip volume: Saturday is the 2nd-busiest day (95k avg trips, vs 110k Fri)
- Fare per trip is 8% higher on weekends (avg $17.30 vs $16.00 weekday)
- Wait-time data (not in this report) suggests excess demand from 6pm-2am Fri-Sat

[See dashboard](http://localhost:3000/dashboard/1) for charts.

## What I can't conclude
- This is one month of data; seasonal patterns unknown
- No data on driver availability vs demand mismatch
- No data on rider satisfaction / abandonment

## Recommendation
Run a 2-week pilot in 2 boroughs (Manhattan + Brooklyn) with 20% more weekend driver slots; measure trip volume, fare, and rider wait time. Report back in 6 weeks.
```

The memo is what gets read; the dashboard is the evidence. **Both are deliverables.**

---

## Exercise 15.7 - Set up a Slack digest (alternative to dashboard)

For periodic reporting, Slack messages often beat dashboards:

```python
import requests

webhook_url = "https://hooks.slack.com/services/YOUR/WEBHOOK/HERE"

# Compute daily stats
con = duckdb.connect("data/taxi.duckdb", read_only=True)
total = con.sql("SELECT SUM(trips), SUM(revenue) FROM daily_summary WHERE day = '2024-01-15'").fetchone()
prev = con.sql("SELECT SUM(trips), SUM(revenue) FROM daily_summary WHERE day = '2024-01-14'").fetchone()

revenue_change = (total[1] - prev[1]) / prev[1] * 100
trip_change = (total[0] - prev[0]) / prev[0] * 100

message = f"""
📊 *Daily Taxi Update - Jan 15, 2024*
• Trips: {total[0]:,} ({trip_change:+.1f}% vs yesterday)
• Revenue: ${total[1]:,.0f} ({revenue_change:+.1f}% vs yesterday)
"""

# requests.post(webhook_url, json={"text": message})   # uncomment in real run
print(message)
```

In production, schedule this in Dagster/Airflow to run daily. **Stakeholders read Slack; many dashboards collect dust.**

---

## Submission checklist

- [ ] Metabase or Superset running and connected to DuckDB-derived data
- [ ] 5-chart dashboard built with headline + bullets text card
- [ ] All bar charts have Y-axis starting at 0
- [ ] No pie charts
- [ ] Stakeholder iteration: pushed back on pie chart and on 20-dimension drill-down
- [ ] Critique of 3 public dashboards written
- [ ] Streamlit alternative built and runs locally
- [ ] 1-page memo written with dashboard link
- [ ] (Optional) Slack digest script tested

---

## What you just did

You can stand up a dashboard in 30 minutes, fit it on 5 charts, write the headline + bullets that the stakeholder actually needs, resist the pull toward sprawl, and produce a memo or Slack digest when those are better deliverables. You can critique any dashboard and articulate why it works or doesn't.

Week 16 closes the curriculum: **the capstone**. dbt + Dagster + DuckDB pipeline that runs on a schedule, ingests + cleans + enriches + dashboards a real dataset. End-to-end production.

---

**Next**: [Week 16: Production Data Pipelines (Capstone) →](../week-16-production-pipelines/readme.md)
