# Week 10: External Enrichment

## 🎯 What you'll learn

Your data is more valuable when it's joined with the world's data. **Geocoding** addresses, attaching **demographics**, hitting **public APIs** (Census, weather, holidays, FX rates), responsibly **scraping** the web. Plus the legal/ethical bright lines from [appsec-mentorship](https://github.com/ichdamola/appsec-mentorship).

By the end of this week you'll be able to:

- Geocode at scale with **Nominatim** (open) and **Google/Mapbox** (paid) - and reason about cost
- Attach US Census demographics to any zip / tract / block group
- Pull holiday calendars, weather, FX rates, public-company financials
- Scrape with rate limits, retries, and robots.txt awareness
- Recognize when enrichment is privacy-sensitive (and what to do about it)
- Build a small **enrichment cache** so you don't hit the same API twice

## 🧰 Lab setup

```bash
uv add httpx tenacity geopy pgeocode pandas polars beautifulsoup4 lxml
```

For Census: register for a free [Census API key](https://api.census.gov/data/key_signup.html).

## ✅ Your job

1. Read [theory.md](theory.md). The ethics section is short, mandatory, and serves as a callback to appsec-mentorship Week 12.
2. Work through [lab.md](lab.md). Geocode addresses, attach Census tract demographics, join weather data to NYC taxi trips.
3. Build a tiny SQLite-backed enrichment cache so re-runs are free.

## 📚 Required reading

| Resource | Why | Time |
|---|---|---|
| [Nominatim usage policy](https://operations.osmfoundation.org/policies/nominatim/) | The free geocoder; respect the limits | 15 min |
| [US Census API docs](https://www.census.gov/data/developers/guidance.html) | The richest free demographic source | 30 min |
| [robots.txt and scraping ethics](https://www.robotstxt.org/) | The bright line | 15 min |
| [tenacity docs](https://tenacity.readthedocs.io/) | Retry patterns | 15 min |

## 💡 What you should already know

- Weeks 01-09
- Week 03 (ingestion patterns)

---

**Next**: [Week 11: Feature Engineering →](../week-11-feature-engineering/readme.md)
