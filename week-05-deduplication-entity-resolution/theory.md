# Week 05: Theory — Deduplication and Entity Resolution

The same person registers as "John Smith" once and "J. Smith" the next time. The same product appears in your catalog 14 times with subtly different names. The same company has 6 records because Sales and Marketing each created their own. **Entity resolution** — figuring out which records refer to the same real-world thing — is one of the dirtiest, highest-leverage parts of data work.

This week covers the spectrum: **exact dedup** (easy), **fuzzy dedup** (the actual work), **probabilistic record linkage** (the modern technique splink uses), and **survivorship** (which record wins).

---

## Part 1: The dedup spectrum

Three classes of dedup problem, in increasing difficulty:

| Class | Example | Tools |
|---|---|---|
| **Exact dedup** | Same `order_id` appears twice | SQL `DISTINCT` / `QUALIFY ROW_NUMBER = 1` |
| **Near-exact dedup** | Same person, identical fields except for trailing whitespace or case | Normalize first, then exact dedup |
| **Fuzzy dedup** | Different formatting, typos, abbreviations | rapidfuzz / splink / dedupe |
| **Cross-source record linkage** | "Match customers from CRM table A to leads from CRM table B" | splink + careful blocking |

The 60% that's free: normalize, then exact dedup. The 30% that's actual engineering: fuzzy. The 10% that's research-grade: cross-source linkage with no overlapping keys.

---

## Part 2: Exact dedup — the easy 60%

```sql
-- "Keep one row per (email, phone), the latest one"
SELECT * FROM customers
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY LOWER(TRIM(email)), regexp_replace(phone, '[^0-9]', '')
    ORDER BY updated_at DESC
) = 1;
```

The key move: **normalize inside the PARTITION BY**, so trivial differences (case, whitespace, punctuation) collapse. Then exact dedup does the rest.

Universal normalization recipes:

```sql
-- Email: lowercase + trim
LOWER(TRIM(email))

-- Phone: digits only
regexp_replace(phone, '[^0-9]', '')

-- Names: lowercase + collapse whitespace + remove punctuation
regexp_replace(LOWER(TRIM(name)), '[^a-z0-9 ]', '')

-- Addresses (basic): lowercase + collapse whitespace + common abbreviations
LOWER(regexp_replace(regexp_replace(address, '\s+', ' '), '\bstreet\b', 'st'))
```

For the trees data from week 03, exact dedup on `tree_id` should produce zero duplicates (the upstream uses it as a primary key). For real CRMs, even after normalization you'll have ~5-15% near-dupes — and that's the floor before fuzzy work.

---

## Part 3: The fuzzy dedup problem

You can't compare every pair. For N records, naive pairwise comparison is `N²/2` — for 1M records that's 500B comparisons. Even at 100k comparisons/second that's 8 weeks of compute.

The trick: **blocking**. Only compare pairs of records that *might* be matches.

```
For 1M records:
  All pairs:           500,000,000,000
  After blocking:                10,000,000   (50,000× fewer)
```

Blocking trades **completeness** for **tractability** — you might miss a few matches whose blocking keys diverge, but you make the problem solvable.

---

## Part 4: Blocking — the secret to making fuzzy work

### What a blocking key is

A simple deterministic function from a record to a string. Records with the **same blocking key** are compared against each other; records with **different** blocking keys are not.

```python
# A naive blocking key for people
def block_key(record):
    last_name = record["last_name"].lower()
    first_letter = record["first_name"][0].lower() if record["first_name"] else ""
    return f"{last_name[:4]}{first_letter}"

# "John Smith" → "smitj"
# "Jane Smith" → "smitj"  ← compared
# "John Jones" → "jonej"  ← not compared with Smiths
```

### The blocking key dilemma

- **Loose** blocking → big blocks, more comparisons, slower, but higher recall (catches more true matches)
- **Tight** blocking → small blocks, fewer comparisons, faster, but lower recall (misses matches where the key differs)

You usually do **multiple passes** with different blocking keys, then union the candidate pairs:

```python
def block_keys(record):
    return [
        f"name:{record['last_name'][:5].lower()}",
        f"email:{record['email'].split('@')[0][:4].lower()}",
        f"phone:{record['phone'][-7:]}",
    ]
```

A pair is a candidate if **any** blocking key matches. Recall goes up; comparison count stays bounded.

### Phonetic blocks: Soundex, Metaphone

For names where spelling varies but pronunciation doesn't ("Smith" vs "Smyth"), phonetic keys work better:

```python
import jellyfish
jellyfish.soundex("Smith")    # 'S530'
jellyfish.soundex("Smyth")    # 'S530'  ← same
jellyfish.metaphone("Smith")  # 'SM0'
jellyfish.metaphone("Smyth")  # 'SM0'   ← same
```

Metaphone is usually better than Soundex — it handles more languages and spelling variants. Use as a blocking key when name spelling is the dominant variability.

---

## Part 5: Similarity scoring

Once a pair is in a block, you need a similarity score. Common choices:

### String distance

| Metric | What it computes | When |
|---|---|---|
| **Levenshtein** | Min edits (insert/delete/substitute) to transform A into B | General typos |
| **Jaro-Winkler** | Like Levenshtein but biases toward shared prefixes | Names, where first letters matter |
| **Damerau-Levenshtein** | Levenshtein + transpositions (swap adjacent chars) | Common typos |
| **Jaccard / token-set** | Overlap of token sets | Long strings; word order doesn't matter |
| **Cosine on token vectors** | Vector similarity | Long text, varying length |

Library: **rapidfuzz** is the modern fast implementation; up to 100× faster than the standard `python-Levenshtein`.

```python
from rapidfuzz import fuzz
fuzz.ratio("Apple Inc.", "Apple Incorporated")            # 78
fuzz.partial_ratio("Apple", "Apple Inc.")                  # 100
fuzz.token_sort_ratio("Apple Inc.", "Inc. Apple")          # 100
fuzz.token_set_ratio("Apple Inc., USA", "Apple, Inc. USA") # 100
```

### Numeric / date proximity

| Field | Strategy |
|---|---|
| Date of birth | Same year? Same year ±1? |
| Order amount | Within $1? Within 5%? |
| Geo coordinates | Haversine distance < 100m? |

### Composite scores

The senior pattern: **a record pair gets a score on each field, then aggregated**.

```python
def pair_score(a, b):
    return (
        0.4 * (fuzz.token_set_ratio(a["name"], b["name"]) / 100)
        + 0.3 * (1.0 if normalize_phone(a["phone"]) == normalize_phone(b["phone"]) else 0.0)
        + 0.2 * (1.0 if a["zip"][:5] == b["zip"][:5] else 0.0)
        + 0.1 * (jellyfish.jaro_winkler(a["company"], b["company"]))
    )

# Match if score > 0.7
```

You tune the threshold empirically against a labeled sample (Part 8).

---

## Part 6: Probabilistic record linkage — Fellegi-Sunter

The classical statistical framework, from the [1969 paper](https://courses.cs.washington.edu/courses/cse590q/04au/papers/Felligi69.pdf). The core idea: for each field, learn two probabilities:

- **m-probability**: P(field A = field B | the records are a match)
- **u-probability**: P(field A = field B | the records are NOT a match)

The **log-likelihood ratio** for a pair:

```
log(m / u)   when fields agree
log((1-m) / (1-u))   when fields disagree
```

Sum across fields. High total = likely match.

Intuition: a field with `m=0.99, u=0.01` (almost always agrees on matches, almost never on non-matches) carries enormous evidential weight. A field with `m=0.6, u=0.4` (slight signal) contributes little.

**Splink** ([documentation](https://moj-analytical-services.github.io/splink/)) is the modern open-source implementation. It learns m and u from your data via EM, supports DuckDB and Spark, and is the default tool for serious record-linkage projects.

The basic splink workflow:

1. Define **comparison columns** and their levels (e.g., name: "exact" / "fuzzy match >85" / "fuzzy 60-85" / "no match")
2. Define **blocking rules** (multiple passes)
3. Run **EM training** — splink estimates m and u from candidate pairs
4. Run **predictions** — splink produces a match probability per pair
5. **Threshold** to produce clusters

```python
from splink.duckdb.linker import DuckDBLinker
import splink.duckdb.comparison_library as cl

settings = {
    "link_type": "dedupe_only",
    "blocking_rules_to_generate_predictions": [
        "l.first_name = r.first_name",
        "l.surname = r.surname",
    ],
    "comparisons": [
        cl.exact_match("first_name"),
        cl.exact_match("surname"),
        cl.levenshtein_at_thresholds("dob", [1, 2]),
        cl.exact_match("email"),
    ],
}

linker = DuckDBLinker(df, settings)
linker.estimate_u_using_random_sampling(target_rows=1e6)
linker.estimate_parameters_using_expectation_maximisation("l.first_name = r.first_name")
results = linker.predict(threshold_match_probability=0.95)
```

For this curriculum, week 05's lab walks splink hands-on. After that you can dismiss most rolled-your-own fuzzy-match code; splink does it better.

---

## Part 7: Clustering — pairs to entities

Splink (and any pairwise scorer) produces **pairs with scores**. You then need to convert pairs into entity clusters: "records A, B, C are the same person."

The simplest approach: **transitive closure**. If A matches B and B matches C, treat A, B, C as one cluster.

```python
# graph-based clustering
import networkx as nx
G = nx.Graph()
for a, b, score in matching_pairs:
    if score > 0.9:
        G.add_edge(a, b)
clusters = list(nx.connected_components(G))
```

Problems with naive transitive closure:

- **Chaining**: A↔B at 0.91, B↔C at 0.91, but A↔C might be 0.4. Should they really be one cluster?
- **Hub explosion**: one frequently-matching record links everything into one giant cluster.

Splink handles this with **connected-components plus per-cluster sanity checks**. For most use cases the defaults work; if you find a "blob" cluster of 10,000 records, that's your signal that thresholds need tuning or a record is over-matchy.

---

## Part 8: Measuring precision and recall

You can't tune a dedup pipeline without ground truth. The standard pattern:

1. Sample 200-500 pairs from your candidate set
2. Manually label each as **match / no-match** (or have multiple labelers do it independently and resolve disagreements)
3. Compute precision and recall:

```
Precision = true_matches / (true_matches + false_matches)
Recall    = true_matches / (true_matches + missed_matches)
F1        = 2 × precision × recall / (precision + recall)
```

For dedup:
- **Precision** is "of the pairs you said match, what fraction actually do?"
- **Recall** is "of the pairs that actually match, what fraction did you find?"

**You can't compute recall from your output alone** — you don't know what you missed. You estimate it via a **stratified labeling** of pairs across the score distribution (some at score 0.99, some at 0.5, some at 0.1) and infer the missed-match rate from those samples.

A working number: precision > 0.95 and recall > 0.85 is good for most production dedup. Hitting precision > 0.99 usually requires sacrificing some recall (more conservative thresholds).

---

## Part 9: Survivorship — picking the winner

You've identified that records 17, 882, and 9943 are the same person. **Which one survives?** Or do you merge fields?

### Strategies

| Strategy | When |
|---|---|
| **Latest wins** | One row, the most recent (highest `updated_at`) |
| **Source priority** | One source is more trusted (e.g., "Stripe overrides CRM") |
| **Field-level merge** | For each field, pick the best value (longest, most recent, most-common, non-null) |
| **All-fields union** | Keep every value as an array (for audit / downstream tooling) |

The **field-level merge** is the most common. A worked example:

```python
def merge_records(records: list[dict]) -> dict:
    """For each field, pick:
       - email: latest non-null
       - name: longest non-null
       - phone: most-recent normalized
       - any other field: latest non-null
       
       Plus add `merged_from` list of input IDs for traceability.
    """
    sorted_recs = sorted(records, key=lambda r: r["updated_at"], reverse=True)
    merged = {
        "id": min(r["id"] for r in records),     # canonical: smallest ID
        "merged_from": [r["id"] for r in records],
    }
    for field in ["email", "phone", "name"]:
        values = [r[field] for r in sorted_recs if r.get(field)]
        if field == "name":
            merged[field] = max(values, key=len, default=None)
        else:
            merged[field] = values[0] if values else None
    return merged
```

### The golden-record pattern

A separate, downstream **golden_records** table that contains one row per entity with merged values. Upstream tables retain the original rows; downstream consumers see only the golden table.

```
customers_raw         ← all rows ever ingested
customer_clusters      ← record_id → cluster_id (from dedup pipeline)
customer_golden        ← cluster_id → merged record (one row per real customer)
```

This separation matters for audit, debugging, and re-running the dedup pipeline without losing source records.

---

## Part 10: Operational considerations

### Running on a schedule

Dedup needs to handle **incremental updates**. The naive approach is "re-cluster everything weekly" — expensive but simple. The senior approach:

- **Incremental linkage**: new rows are blocked against existing clusters; matches are added to the cluster
- **Periodic full re-cluster**: every N weeks, do a full re-run to catch missed matches and split over-merged clusters
- **Cluster IDs are stable**: when re-cluster reassigns, you keep a history (cluster_id_v1, cluster_id_v2, ...) so downstream joins survive

### Privacy

Dedup output is often **more sensitive** than the input — you've now linked "Jane Smith at one address" and "Jane Smith at another address" into one identity. This may require:

- Access controls on the clustered table (not just the raw)
- Right-to-be-forgotten handling: a deletion request must propagate through clusters
- A documented retention policy on cluster IDs

If your dedup involves names + DOB + address, you're in regulated-data territory regardless of jurisdiction.

---

## Part 11: When dedup is the wrong question

Sometimes "duplicates" are actually meaningful:

| Pattern | Why duplicates aren't bugs |
|---|---|
| "Customer placed 3 orders" | Each is a separate order row, even with the same customer |
| "User has 2 active subscriptions" | Each is a valid contract |
| "Same name, different people" | "John Smith" is genuinely common |
| "Snapshot at multiple points in time" | Each row is a snapshot, not a duplicate of the entity |

The question to ask first: **"At what grain is this table?"** If the grain is "one row per order," repeated customer IDs are expected. If the grain is "one row per customer," they're a bug.

---

## What's next

In [lab.md](lab.md) you'll:

- Generate a synthetic-but-realistic customers dataset with deliberately introduced duplicates
- Do exact dedup; measure how many it catches
- Build blocking rules; count candidate pairs
- Score pairs with rapidfuzz; tune a threshold against a labeled sample
- Run splink end-to-end on the same data; compare quality and dev time
- Build a survivorship function and a golden_records table
- Compute precision / recall against your labels

By end of week 05 you can take any real customer / product / company dataset and produce a golden-record table you'd trust.
