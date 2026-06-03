# Week 03: Lab — Build a Real API Puller

You'll build a complete ingestion pipeline: paginated pull from NYC Open Data (the same source the taxi data comes from), with retries, rate limiting, resumability, and bronze → silver conversion. By the end, you'll have a script you could run on a schedule.

## Setup

```bash
uv add httpx tenacity duckdb polars pyarrow orjson
```

```python
import gzip
import json
import os
import time
from pathlib import Path

import httpx
import polars as pl
import orjson
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception_type
```

We'll use the [Socrata Open Data API](https://dev.socrata.com/) which is what NYC Open Data uses. No API key needed for low rates.

```python
BASE_URL = "https://data.cityofnewyork.us/resource/uvpi-gqnh.json"   # NYC Forestry tree census
BRONZE_DIR = Path("data/bronze/nyc_trees")
SILVER_DIR = Path("data/silver")
STATE_PATH = Path("ingest_state.json")
BRONZE_DIR.mkdir(parents=True, exist_ok=True)
SILVER_DIR.mkdir(parents=True, exist_ok=True)
```

The 2015 NYC street tree census — about 680k rows, 41 columns, real messy public data with addresses, species, dates, conditions.

---

## Exercise 3.1 — Sanity check the API

```python
with httpx.Client(timeout=30) as client:
    resp = client.get(BASE_URL, params={"$limit": 5})
    resp.raise_for_status()
    print(json.dumps(resp.json()[:2], indent=2))
```

You should see two tree records — JSON objects with fields like `tree_id`, `spc_common`, `health`, `borough`.

Count the total rows:

```python
with httpx.Client(timeout=30) as client:
    resp = client.get(BASE_URL, params={"$select": "count(*)"})
    print(resp.json())
# ~683k rows
```

---

## Exercise 3.2 — Build the resilient fetcher

```python
class TransientError(Exception):
    pass

@retry(
    retry=retry_if_exception_type((TransientError, httpx.HTTPError)),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    stop=stop_after_attempt(8),
    reraise=True,
)
def fetch_page(client: httpx.Client, offset: int, limit: int) -> list[dict]:
    resp = client.get(
        BASE_URL,
        params={"$limit": limit, "$offset": offset, "$order": "tree_id"},
    )
    if resp.status_code == 429:
        retry_after = int(resp.headers.get("Retry-After", "10"))
        time.sleep(retry_after)
        raise TransientError("rate limited")
    if 500 <= resp.status_code < 600:
        raise TransientError(f"server {resp.status_code}")
    resp.raise_for_status()
    return resp.json()
```

Test it once:

```python
with httpx.Client(timeout=30, headers={"User-Agent": "data-mentorship-lab/1.0"}) as client:
    page = fetch_page(client, offset=0, limit=100)
    print(f"got {len(page)} rows; first tree_id = {page[0]['tree_id']}")
```

The `User-Agent` is the polite-scraping signature from the theory (Part 4). Always set one.

---

## Exercise 3.3 — Write the bronze layer

Bronze = raw, untouched, append-only. JSONL is the right format here — easy to append, easy to inspect.

```python
def write_bronze_page(rows: list[dict], offset: int):
    """Write one page of API rows as gzipped JSONL. Filename includes offset."""
    out_path = BRONZE_DIR / f"page_{offset:07d}.jsonl.gz"
    with gzip.open(out_path, "wb") as f:
        for row in rows:
            f.write(orjson.dumps(row) + b"\n")
    return out_path

# Test it
sample = page[:5]
out = write_bronze_page(sample, offset=0)
print(f"wrote {out}, size {out.stat().st_size} bytes")

# Read back
with gzip.open(out, "rb") as f:
    for line in f:
        print(orjson.loads(line)["tree_id"])
```

Why `orjson` instead of stdlib `json`? It's 5-10× faster on encoding/decoding. For ingestion-hot paths it matters.

---

## Exercise 3.4 — Resumable pull with checkpointing

The full pull loop with state file:

```python
def load_state():
    if STATE_PATH.exists():
        return orjson.loads(STATE_PATH.read_bytes())
    return {"offset": 0, "rows_written": 0, "started_at": time.time()}

def save_state(state):
    tmp = STATE_PATH.with_suffix(".tmp")
    tmp.write_bytes(orjson.dumps(state))
    os.replace(tmp, STATE_PATH)   # atomic on POSIX

def ingest(max_rows: int = 5000, page_size: int = 1000):
    """Pull up to max_rows starting from saved checkpoint. Resumable."""
    state = load_state()
    print(f"starting at offset {state['offset']} (already wrote {state['rows_written']})")

    with httpx.Client(timeout=30, headers={"User-Agent": "data-mentorship-lab/1.0"}) as client:
        while state["rows_written"] < max_rows:
            page = fetch_page(client, offset=state["offset"], limit=page_size)
            if not page:
                print("API returned empty page; assuming end-of-data")
                break

            write_bronze_page(page, offset=state["offset"])
            state["offset"] += len(page)
            state["rows_written"] += len(page)
            save_state(state)

            print(f"  page @ offset {state['offset']}: {len(page)} rows  (total {state['rows_written']})")
            time.sleep(0.1)   # be polite

    state["finished_at"] = time.time()
    save_state(state)
    print(f"\nDone. {state['rows_written']} rows in {state['finished_at'] - state['started_at']:.1f}s")

# Pull 5000 rows
ingest(max_rows=5000)
```

**Now interrupt it** (Ctrl-C halfway through), then re-run. It picks up from where it stopped. That's the property of a senior ingestion pipeline.

---

## Exercise 3.5 — Bronze size sanity

```python
import subprocess

# How big is bronze?
total = sum(f.stat().st_size for f in BRONZE_DIR.glob("*.jsonl.gz"))
n_files = len(list(BRONZE_DIR.glob("*.jsonl.gz")))
print(f"bronze: {n_files} files, {total / 1024 / 1024:.2f} MB total")

# How many rows total?
rows = 0
for f in BRONZE_DIR.glob("*.jsonl.gz"):
    with gzip.open(f) as h:
        rows += sum(1 for _ in h)
print(f"rows: {rows}")
```

A 5000-row sample is typically ~500 KB compressed JSONL. Same data uncompressed ~3 MB.

---

## Exercise 3.6 — Bronze → Silver conversion

Silver is typed, validated, single-file Parquet. The conversion job:

```python
def bronze_to_silver():
    """Read all bronze JSONL.gz files into one typed Parquet."""
    all_rows = []
    for f in sorted(BRONZE_DIR.glob("*.jsonl.gz")):
        with gzip.open(f, "rb") as h:
            for line in h:
                all_rows.append(orjson.loads(line))

    # Load into Polars — get auto-inferred types
    df = pl.DataFrame(all_rows)
    print(f"loaded {len(df)} rows, {len(df.columns)} columns")
    print(df.schema)

    # Coerce specific known columns
    df = df.with_columns(
        tree_id=pl.col("tree_id").cast(pl.Int64),
        tree_dbh=pl.col("tree_dbh").cast(pl.Int32, strict=False),   # diameter at breast height
        created_at=pl.col("created_at").str.to_datetime(strict=False),
        latitude=pl.col("latitude").cast(pl.Float64, strict=False),
        longitude=pl.col("longitude").cast(pl.Float64, strict=False),
    )

    out_path = SILVER_DIR / "nyc_trees.parquet"
    df.write_parquet(out_path, compression="snappy")
    print(f"wrote {out_path}, size {out_path.stat().st_size / 1024:.1f} KB")
    return df

silver = bronze_to_silver()
silver.head(3)
```

---

## Exercise 3.7 — Measure the format wins

Compare bronze (gzipped JSONL), uncompressed JSONL, CSV, and Parquet on disk:

```python
# Write the silver data to all 4 formats
df = pl.read_parquet(SILVER_DIR / "nyc_trees.parquet")
pdf = df.to_pandas()

# Parquet (already exists)
parquet_size = (SILVER_DIR / "nyc_trees.parquet").stat().st_size

# CSV
csv_path = SILVER_DIR / "nyc_trees.csv"
pdf.to_csv(csv_path, index=False)
csv_size = csv_path.stat().st_size

# CSV gz
gz_path = SILVER_DIR / "nyc_trees.csv.gz"
pdf.to_csv(gz_path, index=False, compression="gzip")
gz_size = gz_path.stat().st_size

# JSONL
json_path = SILVER_DIR / "nyc_trees.jsonl"
df.write_ndjson(json_path)
json_size = json_path.stat().st_size

# JSONL gzipped
json_gz_path = SILVER_DIR / "nyc_trees.jsonl.gz"
with gzip.open(json_gz_path, "wb") as f:
    for row in df.iter_rows(named=True):
        f.write(orjson.dumps(row) + b"\n")
json_gz_size = json_gz_path.stat().st_size

print(f"{'format':<20s}  {'size (KB)':>10s}  {'ratio':>6s}")
print(f"{'parquet snappy':<20s}  {parquet_size/1024:>10.1f}  {parquet_size/parquet_size:>6.1f}x")
print(f"{'csv':<20s}  {csv_size/1024:>10.1f}  {csv_size/parquet_size:>6.1f}x")
print(f"{'csv gzip':<20s}  {gz_size/1024:>10.1f}  {gz_size/parquet_size:>6.1f}x")
print(f"{'jsonl':<20s}  {json_size/1024:>10.1f}  {json_size/parquet_size:>6.1f}x")
print(f"{'jsonl gzip':<20s}  {json_gz_size/1024:>10.1f}  {json_gz_size/parquet_size:>6.1f}x")
```

Typical ratios:

```
parquet snappy        ~500 KB    1.0x   ← baseline
csv                   ~2400 KB   4.8x
csv gzip              ~600 KB    1.2x
jsonl                 ~3500 KB   7.0x
jsonl gzip            ~800 KB    1.6x
```

**Parquet wins on size and (we'll show next) read speed simultaneously.** The CSV ecosystem is held together by inertia, not technical merit.

---

## Exercise 3.8 — Read-speed comparison + predicate pushdown

```python
def bench(fn, n_iter=5):
    times = []
    for _ in range(n_iter):
        t0 = time.perf_counter()
        result = fn()
        times.append(time.perf_counter() - t0)
    return min(times)

def read_csv():
    return pl.read_csv(csv_path)

def read_csv_gz():
    return pl.read_csv(gz_path)

def read_parquet_full():
    return pl.read_parquet(SILVER_DIR / "nyc_trees.parquet")

def read_parquet_subset():
    """Predicate-pushdown: only Manhattan trees, only 3 columns."""
    return (
        pl.scan_parquet(SILVER_DIR / "nyc_trees.parquet")
        .filter(pl.col("boroname") == "Manhattan")
        .select("tree_id", "spc_common", "health")
        .collect()
    )

print(f"csv:                  {bench(read_csv)*1000:>7.1f} ms")
print(f"csv gz:               {bench(read_csv_gz)*1000:>7.1f} ms")
print(f"parquet (full):       {bench(read_parquet_full)*1000:>7.1f} ms")
print(f"parquet (predicate):  {bench(read_parquet_subset)*1000:>7.1f} ms")
```

Typical results:

```
csv:                    52.0 ms
csv gz:                 67.0 ms       (gzip decompression cost)
parquet (full):          8.0 ms       ~7x faster than CSV
parquet (predicate):     3.0 ms       ~17x faster (read only the relevant slice)
```

**Predicate pushdown is the killer feature for large datasets.** On the 50 GB versions of this query, you'd see hours vs minutes — same query.

---

## Exercise 3.9 — Inspect Parquet metadata

DuckDB / pyarrow can show what's inside a Parquet file:

```python
import pyarrow.parquet as pq
import duckdb

# pyarrow: schema + row group stats
pq_file = pq.ParquetFile(SILVER_DIR / "nyc_trees.parquet")
print(f"num rows: {pq_file.metadata.num_rows}")
print(f"num row groups: {pq_file.metadata.num_row_groups}")
print(f"schema:\n{pq_file.schema}")

# Per-column statistics for row group 0
rg0 = pq_file.metadata.row_group(0)
for i in range(rg0.num_columns):
    col = rg0.column(i)
    print(f"  {col.path_in_schema:20s}  min={col.statistics.min}  max={col.statistics.max}  nulls={col.statistics.null_count}")
```

You'll see min/max per column per row group — the metadata that lets engines skip row groups during a filter scan. That's how Parquet does its magic.

---

## Exercise 3.10 (stretch) — Postgres logical replication into DuckDB

If you have Docker, set up Postgres and stream changes into DuckDB. This is the modern CDC pattern.

```bash
# Postgres with logical replication enabled
docker run -d --name pg \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  postgres:16 \
  postgres -c wal_level=logical -c max_wal_senders=10 -c max_replication_slots=10
```

```sql
-- In postgres
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT,
    amount NUMERIC(10,2),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE PUBLICATION orders_pub FOR TABLE orders;
SELECT pg_create_logical_replication_slot('orders_slot', 'pgoutput');
```

```python
# Python — incremental pull via updated_at watermark (simpler than wal2json)
import psycopg2

LAST_SEEN_PATH = Path("orders_last_seen.txt")

def load_watermark():
    if LAST_SEEN_PATH.exists():
        return LAST_SEEN_PATH.read_text().strip()
    return "1900-01-01T00:00:00Z"

def save_watermark(ts):
    LAST_SEEN_PATH.write_text(ts)

def incremental_sync():
    last_seen = load_watermark()
    conn = psycopg2.connect("postgresql://postgres:password@localhost:5432/postgres")
    cur = conn.cursor()
    cur.execute("SELECT * FROM orders WHERE updated_at > %s ORDER BY updated_at", (last_seen,))
    rows = cur.fetchall()
    if rows:
        new_watermark = rows[-1][3].isoformat()
        save_watermark(new_watermark)
        # Append to a Parquet file (or upsert)
        print(f"got {len(rows)} new/updated rows; new watermark = {new_watermark}")
    conn.close()
```

For real CDC you'd use [Debezium](https://debezium.io/) or [PeerDB](https://www.peerdb.io/) — they emit logical-decoding events to Kafka/HTTP. The watermark approach above is the simpler "good enough" pattern for the medium-data tier.

---

## Submission checklist

- [ ] `httpx` + `tenacity` retry pattern implemented
- [ ] Pulled real data from NYC Open Data (≥1000 rows)
- [ ] Bronze layer written as gzipped JSONL
- [ ] Resumability tested: interrupt the loop, restart, no data loss
- [ ] Bronze → silver conversion produces typed Parquet
- [ ] Format-size comparison shows Parquet wins on disk by ~5-10× over raw CSV
- [ ] Read-speed comparison shows Parquet wins by ~7× over CSV, ~17× with predicate pushdown
- [ ] Parquet row-group metadata inspected
- [ ] State file uses atomic write (`os.replace` from a tmp path)
- [ ] (Stretch) Postgres watermark-based incremental sync working

---

## What you just did

You built a real ingestion pipeline. Not toy. The pattern you wrote — paginated pull + retries + bronze storage + checkpoint + silver Parquet — is what dlt, Airbyte, Fivetran, and Stitch all implement underneath. **They're full-time engineers building a more polished version of what you just shipped in 200 lines.**

For the rest of this curriculum, you'll consume Parquet files via Polars and DuckDB rather than rebuilding the ingestion every week. Week 04 starts validating the data you just pulled — building the quality gates that decide whether silver gets blessed as gold.

---

**Next**: [Week 04: Data Quality Fundamentals →](../week-04-data-quality-fundamentals/readme.md)
