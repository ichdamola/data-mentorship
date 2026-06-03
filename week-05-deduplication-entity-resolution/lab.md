# Week 05: Lab — Dedup a Customer Table

You'll build a synthetic customers dataset with deliberate near-duplicates, dedup it three ways (exact normalize, rapidfuzz, splink), and measure precision/recall against a labeled sample. By the end you'll have a complete entity-resolution pipeline you'd run on real CRM data.

## Setup

```bash
uv add splink rapidfuzz jellyfish polars duckdb
```

```python
import polars as pl
import duckdb
from rapidfuzz import fuzz, process
import jellyfish
import random
from pathlib import Path
```

---

## Exercise 5.1 — Generate a realistic dirty customers table

```python
random.seed(42)

FIRST_NAMES = ["John", "Jane", "Mary", "Robert", "Patricia", "Michael", "Linda", "David", "Barbara", "Richard"]
LAST_NAMES = ["Smith", "Johnson", "Williams", "Brown", "Jones", "Garcia", "Miller", "Davis", "Rodriguez", "Martinez"]
DOMAINS = ["gmail.com", "yahoo.com", "outlook.com", "company.com"]

def make_clean_customer(i):
    first = random.choice(FIRST_NAMES)
    last = random.choice(LAST_NAMES)
    return {
        "customer_id": i,
        "first_name": first,
        "last_name": last,
        "email": f"{first.lower()}.{last.lower()}@{random.choice(DOMAINS)}",
        "phone": f"+1{random.randint(2000000000, 9999999999)}",
        "address": f"{random.randint(1, 9999)} Main St",
        "city": random.choice(["New York", "Boston", "Chicago"]),
    }

def dirty_copy(rec, copy_id):
    """Make a near-duplicate with realistic variations."""
    d = dict(rec)
    d["customer_id"] = copy_id

    # First name variations (random subset)
    r = random.random()
    if r < 0.2: d["first_name"] = d["first_name"][0] + "."     # "John" → "J."
    elif r < 0.3: d["first_name"] = d["first_name"].upper()
    elif r < 0.4: d["first_name"] = d["first_name"] + " "       # trailing space

    # Last name variations
    if random.random() < 0.15:
        d["last_name"] = d["last_name"].replace("Smith", "Smyth").replace("Jones", "Jonez")

    # Email variations
    r = random.random()
    if r < 0.2: d["email"] = d["email"].upper()
    elif r < 0.3: d["email"] = " " + d["email"]                  # leading whitespace
    elif r < 0.4: d["email"] = d["email"].replace("@", "+work@") # plus-tag

    # Phone variations
    r = random.random()
    if r < 0.3:
        digits = d["phone"][2:]   # strip +1
        d["phone"] = f"({digits[:3]}) {digits[3:6]}-{digits[6:]}"
    elif r < 0.5:
        d["phone"] = d["phone"][2:]   # strip country code

    # Address variations
    if random.random() < 0.3:
        d["address"] = d["address"].replace("St", "Street")

    return d


# Generate 500 clean customers, then make ~25% have a near-duplicate
clean = [make_clean_customer(i) for i in range(500)]
next_id = 500
all_records = list(clean)

duplicate_truth = []   # (id_a, id_b) pairs that should be matched
for rec in clean:
    if random.random() < 0.25:
        dup = dirty_copy(rec, next_id)
        all_records.append(dup)
        duplicate_truth.append((rec["customer_id"], next_id))
        next_id += 1

df = pl.DataFrame(all_records)
print(f"total records: {len(df)}, known duplicate pairs: {len(duplicate_truth)}")
df.head(5)
```

You should see ~625 records with ~125 known duplicate pairs. We'll measure each dedup approach against this ground truth.

---

## Exercise 5.2 — Exact dedup (after normalization)

```python
def normalize_email(s: str) -> str:
    if not s: return ""
    s = s.strip().lower()
    # Strip plus-tags: "foo+bar@x.com" → "foo@x.com"
    if "+" in s and "@" in s:
        local, domain = s.split("@", 1)
        local = local.split("+", 1)[0]
        s = f"{local}@{domain}"
    return s

def normalize_phone(s: str) -> str:
    if not s: return ""
    return "".join(c for c in s if c.isdigit())[-10:]   # last 10 digits

def normalize_name(s: str) -> str:
    if not s: return ""
    return s.strip().lower().rstrip(".")

df_normalized = df.with_columns(
    n_email=pl.col("email").map_elements(normalize_email, return_dtype=pl.Utf8),
    n_phone=pl.col("phone").map_elements(normalize_phone, return_dtype=pl.Utf8),
    n_first=pl.col("first_name").map_elements(normalize_name, return_dtype=pl.Utf8),
    n_last=pl.col("last_name").map_elements(normalize_name, return_dtype=pl.Utf8),
)

# Group by normalized email
exact_matches = (
    df_normalized
    .group_by("n_email")
    .agg(
        ids=pl.col("customer_id"),
        count=pl.len(),
    )
    .filter(pl.col("count") > 1)
)
print(f"exact-match dupe groups (by normalized email): {len(exact_matches)}")

# Pairs found
exact_pairs = set()
for row in exact_matches.iter_rows(named=True):
    ids = sorted(row["ids"])
    for i in range(len(ids)):
        for j in range(i+1, len(ids)):
            exact_pairs.add((ids[i], ids[j]))

# Compare to truth
truth_set = {tuple(sorted(pair)) for pair in duplicate_truth}
tp = len(exact_pairs & truth_set)
fp = len(exact_pairs - truth_set)
fn = len(truth_set - exact_pairs)
precision = tp / (tp + fp) if (tp + fp) else 0
recall = tp / (tp + fn) if (tp + fn) else 0
print(f"exact-dedup: P={precision:.3f}  R={recall:.3f}  (caught {tp}/{len(truth_set)})")
```

Typical result: precision ~1.0 (exact normalize is reliable), recall ~0.4-0.6 (it misses anything where normalization didn't suffice). **The 60% you get for free.**

---

## Exercise 5.3 — Block + fuzzy score

Now add fuzzy dedup with rapidfuzz. First, define blocking:

```python
def blocking_keys(rec):
    """Multiple blocking keys; a candidate pair shares ANY of them."""
    out = []
    if rec["n_last"]:
        out.append(f"last:{rec['n_last'][:4]}")          # first 4 chars of last name
        out.append(f"sdx:{jellyfish.soundex(rec['n_last'])}")  # phonetic
    if rec["n_phone"]:
        out.append(f"phone7:{rec['n_phone'][-7:]}")      # last 7 digits
    if rec["n_email"]:
        out.append(f"email4:{rec['n_email'][:4]}")
    return out

# Build block index
from collections import defaultdict
block_to_ids = defaultdict(list)
records_by_id = {r["customer_id"]: r for r in df_normalized.iter_rows(named=True)}

for rec in df_normalized.iter_rows(named=True):
    for key in blocking_keys(rec):
        block_to_ids[key].append(rec["customer_id"])

# Enumerate candidate pairs (records sharing at least one block)
candidate_pairs = set()
for key, ids in block_to_ids.items():
    if len(ids) < 2:
        continue
    for i in range(len(ids)):
        for j in range(i+1, len(ids)):
            pair = tuple(sorted([ids[i], ids[j]]))
            candidate_pairs.add(pair)

n_all = len(df) * (len(df) - 1) // 2
print(f"all possible pairs:    {n_all:,}")
print(f"candidate pairs after blocking: {len(candidate_pairs):,}  ({len(candidate_pairs)/n_all:.2%})")
```

You should see blocking reduce ~200k pairs to ~10k. That's the **~95% comparison reduction** that makes fuzzy dedup tractable.

---

## Exercise 5.4 — Score candidate pairs

```python
def pair_score(a, b):
    """Composite fuzzy score in [0, 1]."""
    name_score = fuzz.token_set_ratio(
        f"{a['n_first']} {a['n_last']}",
        f"{b['n_first']} {b['n_last']}",
    ) / 100
    email_match = 1.0 if a["n_email"] == b["n_email"] and a["n_email"] else 0.0
    phone_match = 1.0 if a["n_phone"] == b["n_phone"] and a["n_phone"] else 0.0

    return 0.4 * name_score + 0.3 * email_match + 0.3 * phone_match

scored = []
for a_id, b_id in candidate_pairs:
    a = records_by_id[a_id]
    b = records_by_id[b_id]
    scored.append((a_id, b_id, pair_score(a, b)))

# Inspect score distribution
import statistics
scores = [s for _, _, s in scored]
print(f"score distribution: min={min(scores):.2f}  median={statistics.median(scores):.2f}  max={max(scores):.2f}")
```

---

## Exercise 5.5 — Pick a threshold using ground truth

For a labeled sample, sweep the threshold and pick what maximizes F1:

```python
def evaluate(scored_pairs, threshold, truth):
    predicted = {tuple(sorted([a, b])) for a, b, s in scored_pairs if s >= threshold}
    tp = len(predicted & truth)
    fp = len(predicted - truth)
    fn = len(truth - predicted)
    p = tp / (tp + fp) if (tp + fp) else 0
    r = tp / (tp + fn) if (tp + fn) else 0
    f1 = 2*p*r / (p+r) if (p + r) else 0
    return p, r, f1

print(f"{'thresh':<8s}  {'P':<6s}  {'R':<6s}  {'F1':<6s}")
for t in [0.5, 0.6, 0.7, 0.75, 0.8, 0.85, 0.9, 0.95]:
    p, r, f1 = evaluate(scored, t, truth_set)
    print(f"{t:<8.2f}  {p:<6.3f}  {r:<6.3f}  {f1:<6.3f}")
```

You should see F1 peak around 0.7-0.8 threshold. **In real production work you can't compute F1 against the full truth set — you stratified-sample ~300 pairs across the score range and label each.** The synthetic ground truth makes this exercise easy; the technique generalizes.

---

## Exercise 5.6 — Build pair-based clusters

Now turn matching pairs into entity clusters via connected components.

```python
threshold = 0.7
matching_pairs = [(a, b) for a, b, s in scored if s >= threshold]

# Union-Find / disjoint-set
class UnionFind:
    def __init__(self):
        self.parent = {}
    def find(self, x):
        if self.parent.setdefault(x, x) != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    def union(self, x, y):
        rx, ry = self.find(x), self.find(y)
        if rx != ry:
            self.parent[rx] = ry

uf = UnionFind()
for a, b in matching_pairs:
    uf.union(a, b)

# Cluster ID = the root
clusters_by_root = defaultdict(list)
for id_ in records_by_id:
    clusters_by_root[uf.find(id_)].append(id_)

n_singletons = sum(1 for ids in clusters_by_root.values() if len(ids) == 1)
n_multi = sum(1 for ids in clusters_by_root.values() if len(ids) > 1)
print(f"total clusters: {len(clusters_by_root)}")
print(f"  singletons: {n_singletons}")
print(f"  multi-record clusters: {n_multi}")

# Sample biggest cluster
biggest = max(clusters_by_root.values(), key=len)
print(f"\nbiggest cluster ({len(biggest)} records):")
for i in biggest:
    r = records_by_id[i]
    print(f"  {i:>4}  {r['first_name']} {r['last_name']}  {r['email']}  {r['phone']}")
```

The biggest cluster should be a real customer with their dirty duplicates. **If you see a cluster of 30+ records, your threshold is too loose** — usually one over-matchy record is pulling everything in.

---

## Exercise 5.7 — Survivorship

Build the golden record per cluster.

```python
def merge_cluster(records):
    """Pick best value per field. Track inputs."""
    merged = {
        "cluster_id": min(r["customer_id"] for r in records),
        "n_records_merged": len(records),
        "merged_from": [r["customer_id"] for r in records],
    }
    # Email: most recent non-empty (use customer_id as proxy for recency)
    sorted_recs = sorted(records, key=lambda r: r["customer_id"], reverse=True)
    for field in ["first_name", "last_name", "email", "phone", "address", "city"]:
        values = [r[field] for r in sorted_recs if r.get(field)]
        if field in ("first_name", "last_name"):
            merged[field] = max(values, key=len, default=None)
        else:
            merged[field] = values[0] if values else None
    return merged

golden_records = []
for ids in clusters_by_root.values():
    records = [records_by_id[i] for i in ids]
    golden_records.append(merge_cluster(records))

golden_df = pl.DataFrame(golden_records)
print(f"golden records: {len(golden_df)}")
print(golden_df.head(10))
```

You should see ~500 golden records (one per real customer) from your 625 raw records — losing ~125 duplicates. Save it:

```python
Path("data/silver").mkdir(parents=True, exist_ok=True)
golden_df.write_parquet("data/silver/customers_golden.parquet")
```

---

## Exercise 5.8 — splink walkthrough (stretch)

Now do the same thing with splink, the production-grade tool.

```python
from splink import Linker, settings_dict
import splink.comparison_library as cl
from splink.duckdb.connection import DuckDBAPI

# Convert to pandas (splink supports DuckDB and Spark; pandas via DuckDB)
import pandas as pd
df_pd = df.to_pandas()

# Splink wants a unique 'unique_id' column
df_pd = df_pd.rename(columns={"customer_id": "unique_id"})

settings = {
    "link_type": "dedupe_only",
    "blocking_rules_to_generate_predictions": [
        "l.last_name = r.last_name",
        "substr(l.email, 1, 4) = substr(r.email, 1, 4)",
    ],
    "comparisons": [
        cl.NameComparison("first_name").configure(term_frequency_adjustments=True),
        cl.NameComparison("last_name").configure(term_frequency_adjustments=True),
        cl.EmailComparison("email"),
    ],
}

linker = Linker(df_pd, settings, db_api=DuckDBAPI())
linker.estimate_u_using_random_sampling(max_pairs=1e5)
linker.estimate_parameters_using_expectation_maximisation("l.last_name = r.last_name")

predictions = linker.predict(threshold_match_probability=0.8)
splink_pairs = set()
for row in predictions.as_pandas_dataframe().itertuples():
    splink_pairs.add(tuple(sorted([row.unique_id_l, row.unique_id_r])))

p, r, f1 = evaluate(
    [(a, b, 1.0) for a, b in splink_pairs],
    threshold=0.5, truth=truth_set,
)
print(f"splink @ 0.8 prob: P={p:.3f}  R={r:.3f}  F1={f1:.3f}")
```

Splink (with EM-learned m/u probabilities) typically beats hand-tuned rapidfuzz on real data — and it took fewer lines of code. **For real projects, splink is the answer.**

The rapidfuzz exercise was to teach you what splink is doing under the hood; the splink exercise is what you'd actually ship.

---

## Exercise 5.9 — Real-world generalization

For your own data: pick a column with known duplicates (customer name from a CRM dump, product name from a catalog, company name from an investor list). Apply the same pipeline:

1. Profile the column (Week 04)
2. Normalize aggressively
3. Block on multiple keys (one phonetic, one prefix-based, one exact-on-a-different-field)
4. Score candidate pairs
5. Label 200 pairs by hand across the score range
6. Pick a threshold for F1 > 0.9
7. Cluster + survivorship
8. Write the golden table

In a real project this takes 1-2 weeks. By the end you'll have removed 5-30% noise from the source data. Stakeholders will be very happy.

---

## Submission checklist

- [ ] Synthetic dirty customer dataset generated with ~125 known duplicates
- [ ] Exact normalize + dedup achieves precision > 0.95
- [ ] Blocking reduces candidate pairs by ≥ 90% vs all-pairs
- [ ] Score-distribution histogram inspected
- [ ] Threshold sweep produces an F1 maximum
- [ ] Union-Find clustering producing realistic cluster sizes (no 100-record blobs)
- [ ] Survivorship function picks defensible winners per field
- [ ] `customers_golden.parquet` written
- [ ] (Stretch) Splink achieves P, R > 0.9 with less code than the manual pipeline

---

## What you just did

You built a production-shape entity resolution pipeline: normalize, block, score, cluster, survivor. You measured precision and recall against ground truth instead of eyeballing. You saw splink make the whole thing one third the code.

For weeks 06-08 we keep cleaning: missing data (06), normalization for canonical formats (07), text + fuzzy work (08). Week 09 then joins this clean data to other clean data; week 16 puts it on a schedule.

---

**Next**: [Week 06: Missing Data + Outliers →](../week-06-missing-data-outliers/readme.md)
