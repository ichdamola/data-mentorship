# Week 03: Theory — Ingestion

Data has to get into your stack from somewhere. **Vendor APIs, partner SFTP, file dumps, change-data-capture (CDC) streams, scrapes, webhooks, Kafka topics, message queues.** Each has its own failure modes, rate limits, schema-stability surprises, and political dynamics.

By the end of this week you'll have a working mental model for the trade-offs (file format, transport, sync vs async, batch vs stream), and you'll have built a real puller with retries, rate limiting, resumability, and a destination Parquet file.

---

## Part 1: The seven sources of data

In rough order of "how often you'll see this":

| Source | Mechanism | Failure modes |
|---|---|---|
| **REST API** | HTTP pull, JSON / CSV | Rate limits, pagination, undocumented schema changes |
| **File dump** | SFTP / S3 / GCS / email attachment (yes) | Late files, schema drift, encoding hell |
| **Database CDC** | Logical replication / change streams | Replication lag, schema-evolution mismatches |
| **Webhook** | They POST to you | Auth, replay attacks, ordering |
| **Streaming (Kafka, Kinesis, PubSub)** | Continuous event delivery | At-least-once vs exactly-once, partition skew |
| **Scrape** | HTML parsing | Site changes, anti-scrape defenses, ethics |
| **Manual entry / Google Sheets** | Humans | Everything |

You'll mostly write REST API pullers and file-dump processors. **Master those two and you've covered ~80% of real ingestion jobs.**

---

## Part 2: File format trade-offs

The single most consequential ingestion decision. The honest table:

| Format | Size | Read speed | Write speed | Schema | Compression | Good for |
|---|---|---|---|---|---|---|
| **CSV** | Largest (10×) | Slowest | Fastest write | Implicit, lies | None (gz optional) | Human readable, lowest-common-denominator |
| **JSON / JSONL** | Big (5-8×) | Slow | Fast | Implicit | Optional gzip | Nested data, API responses |
| **Parquet** | Smallest (1×) | Fastest | Medium | Strong, embedded | Built-in (snappy/zstd) | Analytics, columnar reads, the right default |
| **Avro** | Small (1-2×) | Fast | Fast | Strong, separate `.avsc` | Built-in | Streaming, Kafka |
| **ORC** | Smallest | Fastest | Slow | Strong | Built-in | Hadoop ecosystem, less common in Python |
| **Arrow IPC / Feather** | Small | Fastest read | Fast write | Strong | Optional | Inter-process / in-memory exchange |
| **Excel** | Largest (15×) | Slowest | Slowest | Visual disaster | None | "Send it to the team" |

**Defaults to adopt:**

- **For analytics storage**: Parquet with snappy compression. Period.
- **For nested API responses**: JSONL gzipped (one JSON object per line). Easy to parse incrementally.
- **For "send to a stakeholder"**: CSV — it'll open in Excel and nobody will fight you.
- **For inter-pipeline exchange**: Arrow IPC if both sides are Polars/pandas; otherwise Parquet.

### Why Parquet wins for analytics

Three properties:

1. **Columnar layout** — analytical queries read a few columns of many rows. Columnar storage reads only what's asked.
2. **Compression by column** — each column compresses independently. A column of timestamps compresses much tighter than a row of mixed types.
3. **Statistics and predicate pushdown** — Parquet stores per-row-group min/max/null counts. Query engines (DuckDB, Polars, Spark) use these to skip row groups entirely.

For a 50 GB CSV with monthly date column, a query for "October 2024 only" reads ~50 GB. Same data in Parquet partitioned by year+month: ~500 MB.

The win is enormous. Always Parquet for analytics.

### Why CSV is the wrong default

- **No types.** Everything is a string. Dates, numbers, booleans — all up to the reader's guess.
- **No nulls.** Empty string ≠ null in CSV.
- **No schema.** Add a column and downstream parsers may crash.
- **No nesting.** Arrays and objects don't exist.
- **No compression** built in (you can gzip it but you've lost everything-format-y about CSV).
- **No quoting consensus.** Embedded quotes, commas, newlines — every parser handles them differently.

The only legitimate reasons to choose CSV: human readability, Excel-friendliness, or you're forced to. Otherwise, Parquet.

---

## Part 3: REST API patterns

### Pagination — the three styles

Almost every API paginates. The three common styles:

```python
# Style 1: offset/limit
GET /v1/things?offset=0&limit=100
GET /v1/things?offset=100&limit=100
GET /v1/things?offset=200&limit=100
# Drawback: results shift if items are inserted between requests
```

```python
# Style 2: page number
GET /v1/things?page=1&per_page=100
GET /v1/things?page=2&per_page=100
# Same shift problem as offset
```

```python
# Style 3: cursor / opaque token (modern best practice)
GET /v1/things?limit=100
# Response: { "data": [...], "next_cursor": "abc123" }
GET /v1/things?limit=100&cursor=abc123
# Response: { "data": [...], "next_cursor": "def456" }
GET /v1/things?limit=100&cursor=def456
# Response: { "data": [...], "next_cursor": null }
```

Cursor pagination is **stable under inserts** — each cursor is bound to a specific position in the result. Stripe, GitHub, modern Twitter/X clones use this. Prefer it when offered.

### Rate limits and retries — `tenacity`

```python
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception_type
import httpx

class TransientError(Exception):
    pass

@retry(
    retry=retry_if_exception_type(TransientError),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    stop=stop_after_attempt(8),
    reraise=True,
)
def fetch_page(client: httpx.Client, url: str, params: dict):
    resp = client.get(url, params=params, timeout=30)
    if resp.status_code == 429:
        # Rate limited — backoff
        raise TransientError(f"rate limited: {resp.headers.get('Retry-After')}")
    if 500 <= resp.status_code < 600:
        raise TransientError(f"server error: {resp.status_code}")
    resp.raise_for_status()
    return resp.json()
```

What this gives you:

- **Exponential backoff** (1s, 2s, 4s, 8s, 16s, 32s, 60s, 60s) — gentle retries that don't pound the upstream
- **Retry on transient errors only** (429, 5xx) — don't retry 401, 403, 404
- **Max 8 attempts** then give up — don't loop forever
- **Reraise** — the caller still sees the failure for tracking

For APIs that publish rate limits in headers (e.g., `X-RateLimit-Remaining`), use a **token bucket** to pre-emptively slow down rather than waiting for 429:

```python
import time

class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate    # tokens per second
        self.last_refill = time.monotonic()

    def acquire(self, n=1):
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
        self.last_refill = now

        if self.tokens >= n:
            self.tokens -= n
            return 0
        wait = (n - self.tokens) / self.refill_rate
        time.sleep(wait)
        self.tokens = 0
        self.last_refill = time.monotonic()
        return wait
```

### Resumability

For long-running pulls (12-hour backfill, etc.), **persist progress**. The naive approach loses everything when the process crashes:

```python
# BAD — losing all progress on a crash
for cursor in walk_api():
    save_to_parquet(cursor)
```

The senior pattern: **checkpoint after each batch**.

```python
# State file holds: last completed cursor, count, schema hash
import json
STATE_PATH = "ingest_state.json"

def load_state():
    try:
        with open(STATE_PATH) as f:
            return json.load(f)
    except FileNotFoundError:
        return {"cursor": None, "rows": 0}

def save_state(state):
    with open(STATE_PATH + ".tmp", "w") as f:
        json.dump(state, f)
    os.replace(STATE_PATH + ".tmp", STATE_PATH)   # atomic

state = load_state()
while True:
    page = fetch_page(client, url, {"cursor": state["cursor"]})
    if not page["data"]:
        break
    save_batch_to_parquet(page["data"])
    state["cursor"] = page["next_cursor"]
    state["rows"] += len(page["data"])
    save_state(state)
```

When this process crashes 9 hours in, restart resumes from where it stopped. **Atomic file replace (`os.replace`) is critical** — partial state writes are an entire class of bug.

---

## Part 4: Scraping ethics and the bright line

If you're not pulling from a vendor's official API, you're **scraping**. The legal and ethical landscape varies by jurisdiction, but the universal rules:

1. **Check `robots.txt`**. It's not legally binding everywhere but ignoring it is bad faith.
2. **Rate limit yourself**. 1 request/sec is polite for most sites; some explicitly publish a target.
3. **Set a `User-Agent`** that identifies you and provides a contact email.
4. **Respect explicit anti-scrape signals** — CAPTCHAs, IP blocks, terms of service rejections.
5. **Don't scrape personal data** without affirmative consent or a clear public-interest justification.
6. **Cache aggressively** — re-running shouldn't multiply load on the source.

The appsec-curriculum version of this conversation lives at [appsec-mentorship week 14](https://github.com/ichdamola/appsec-mentorship/tree/main/week-14-security-misconfig) (the "shodan.io" sidebar). The short version: **authorized = good; unauthorized = the law's lottery, and you're the prize.**

For legitimate uses:

- **Wayback Machine** is a great proxy for historical pages (and has its own API)
- **Common Crawl** is petabytes of pre-crawled web data, free
- **Open data portals** (data.gov, EU Open Data, World Bank) are vast and explicitly meant to be downloaded

---

## Part 5: CDC (Change Data Capture)

When you need a downstream table to **stay in sync** with an upstream OLTP database (Postgres, MySQL), you want CDC.

The three families:

| Approach | How | When |
|---|---|---|
| **Periodic full snapshots** | Dump the whole table every N hours | Small tables (<1M rows), low-criticality |
| **Incremental snapshots with `updated_at`** | Pull rows where `updated_at > last_sync` | Medium tables, you control the source schema |
| **Logical replication / WAL streaming** | Subscribe to row-level change events | Large tables, real-time, modern stack |

For Postgres, **logical replication** (since 10.x) and tools like **Debezium** ([Kafka Connect](https://debezium.io/)) or **PeerDB** make this turnkey. You set up a replication slot; events flow to the destination as they happen.

For DuckDB-as-destination on a laptop, the practical pattern is **incremental snapshots** with `updated_at` watermarks — described in the lab.

### The append-only data warehouse pattern

Even with CDC, most modern warehouses **don't update rows in place**. They append. Each row gets a `(event_time, op, ...)` tuple — INSERT, UPDATE, DELETE all become inserts of new rows with appropriate operation flags. A view materializes the "current" state.

Why: appends are cheap; updates on columnar storage are expensive; the audit log is a feature, not a cost.

This is the model dbt's "incremental" materialization plays well with, and the foundation of the lakehouse pattern (Iceberg, Delta, Hudi).

---

## Part 6: Streaming — the lightest possible mention

Streaming ingestion (Kafka, Kinesis, PubSub) is a whole world. For this curriculum, the things to know:

- **At-least-once vs exactly-once delivery** — most streams promise at-least-once; exactly-once is hard and slow
- **Consumer offsets** — you commit "I read up to position X"; on restart you resume from there
- **Partitions and ordering** — ordering is guaranteed within a partition, not across
- **Schema registry** (Confluent Schema Registry, AWS Glue) — the contract layer between producer and consumer

You probably won't write streaming code in this curriculum. You'll see it in [system-design-mentorship week 09](https://github.com/ichdamola/system-design-mentorship/tree/main/week-09-chat) and [week 11](https://github.com/ichdamola/system-design-mentorship/tree/main/week-11-distributed-counter) (chat and distributed counter). For data work in 2026, knowing the vocabulary is enough; building streaming is its own track.

---

## Part 7: Ingestion architecture — bronze / silver / gold

The modern best practice for organizing ingested data is the **medallion architecture** (Databricks's terminology, but everyone uses it):

```
┌────────────────────────────────────────────────────────────────┐
│  BRONZE — raw, append-only, never modified                     │
│  ────────                                                       │
│  data/raw/api_source/2024-10-15/page_001.jsonl.gz              │
│  data/raw/sftp_dump/customer_export_2024-10-15.csv             │
│                                                                 │
│  ↓                                                              │
│  SILVER — cleaned, normalized, schema-enforced                  │
│  ────────                                                       │
│  data/silver/customers.parquet  ← typed, validated              │
│                                                                 │
│  ↓                                                              │
│  GOLD — modeled, aggregated, business-ready                     │
│  ─────                                                          │
│  data/gold/fct_customer_revenue_daily.parquet                   │
└────────────────────────────────────────────────────────────────┘
```

**Bronze is sacred.** You never overwrite or edit it. If a downstream bug corrupts silver, you re-derive silver from bronze. If you discover the cleaning logic was wrong six months ago, you re-clean from bronze.

In practice this means:

- Bronze paths are **partitioned by ingestion date** (and source if multiple)
- Bronze files are **immutable** — once written, never changed
- Bronze includes a **`_ingested_at`** column (or filename) so you can detect late-arriving data
- Bronze should **never feed dashboards directly** — always through silver

Week 16's capstone builds exactly this structure with dbt + Dagster.

---

## Part 8: Things you can defer

Topics that you'll encounter eventually but don't need to master now:

| Topic | When to revisit |
|---|---|
| Apache Beam / Dataflow | Streaming-heavy environments; replaced by Spark Streaming / Flink in most orgs |
| Singer taps | Older "extract" framework; mostly replaced by Airbyte / Fivetran-as-SaaS |
| Stitch / Fivetran (SaaS) | When time is more valuable than control — start with Airbyte (open) or buy |
| dlt (data load tool) | Promising modern Python ingestion lib; worth a look once you've built your own |
| Iceberg / Delta / Hudi (lakehouse formats) | Production data warehousing at scale; preview in week 16 |

---

## What's next

In [lab.md](lab.md) you'll:

- Pull from a real paginated API (NYC Open Data) with `httpx` + `tenacity` retries
- Store the raw responses as gzipped JSONL (bronze)
- Convert that bronze JSONL to a typed Parquet (silver) and benchmark the size win
- Build resumability with a checkpoint file
- Try predicate pushdown on Parquet — measure the read-speed savings
- (Stretch) Set up Postgres logical replication into DuckDB

By end of week 03 you can move data from anywhere into a clean, validated, queryable form on your laptop. That's the foundation everything else this curriculum builds on.
