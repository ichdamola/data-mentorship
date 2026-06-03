# Week 08: Lab — Clean Text Like You Mean It

You'll diagnose Unicode footguns, build readable regexes, do fuzzy address matching with rapidfuzz, run spaCy NER on real free-text data, and end with a complete "string-to-canonical" pipeline you'd use on any messy text column.

## Setup

```bash
uv add rapidfuzz unidecode spacy regex ftfy polars
uv run python -m spacy download en_core_web_sm
```

```python
import unicodedata
import re
import regex
import polars as pl
from rapidfuzz import fuzz, process
from unidecode import unidecode
import spacy
import ftfy

nlp = spacy.load("en_core_web_sm")
```

---

## Exercise 8.1 — Unicode footguns

```python
# Two strings that look identical
s1 = "café"                                  # NFC: 4 code points
s2 = "cafe" + "́"                       # NFD: e + combining acute
print(f"s1 = {s1!r}, len={len(s1)}")
print(f"s2 = {s2!r}, len={len(s2)}")
print(f"s1 == s2: {s1 == s2}")               # False — they're not equal
print(f"NFC normalize: {unicodedata.normalize('NFC', s1) == unicodedata.normalize('NFC', s2)}")
# True

# Mojibake — what UTF-8 looks like when decoded as Latin-1
broken = "café".encode("utf-8").decode("latin-1")
print(f"\nbroken: {broken!r}")
fixed = broken.encode("latin-1").decode("utf-8")
print(f"fixed:  {fixed!r}")

# ftfy handles common cases automatically
print(f"ftfy:   {ftfy.fix_text(broken)!r}")

# Confusables — Cyrillic а vs Latin a
cyr_a = "а"
lat_a = "a"
print(f"\nCyrillic а: {cyr_a!r} = {ord(cyr_a):#x}")
print(f"Latin a:    {lat_a!r} = {ord(lat_a):#x}")
print(f"Visually identical, comparing equal: {cyr_a == lat_a}")
# False
```

You should now feel why "lowercase and strip" isn't enough.

---

## Exercise 8.2 — The canonical text-normalizer

```python
def normalize_text(s: str | None) -> str:
    """Best-effort normalization for display / matching."""
    if s is None: return ""
    s = ftfy.fix_text(s)                         # repair mojibake
    s = unicodedata.normalize("NFKC", s)         # composed + compatibility
    s = re.sub(r"\s+", " ", s).strip()           # collapse whitespace
    return s

def normalize_for_match(s: str | None) -> str:
    """Aggressive normalization for fuzzy matching / dedup."""
    s = normalize_text(s).lower()
    s = unicodedata.normalize("NFKD", s)
    s = "".join(c for c in s if not unicodedata.combining(c))  # strip accents
    s = re.sub(r"[^a-z0-9 ]", "", s)                            # ASCII alphanum only
    return s

# Test on a real challenging string
tricky = "  CafÃ©  São Paulo! ​"
print(f"raw:                 {tricky!r}")
print(f"normalize_text:      {normalize_text(tricky)!r}")
print(f"normalize_for_match: {normalize_for_match(tricky)!r}")
```

You should see `"  CafÃ© ..."` → `"café são paulo"` for display, `"cafe sao paulo"` for matching.

---

## Exercise 8.3 — Regex done right

Build a readable regex for parsing a "price with optional currency" pattern:

```python
price_pat = regex.compile(r"""
    ^\s*
    (?P<currency>\$|€|£|¥|USD|EUR|GBP|JPY)?     # optional currency
    \s*
    (?P<amount>\d{1,3}(?:[,.]\d{3})*(?:\.\d{1,4})?)  # number with thousands & decimal
    \s*
    (?P<unit>[A-Z]{3})?                          # optional ISO currency code suffix
    \s*$
""", regex.VERBOSE)

samples = ["$19.99", "€45.50", "1,200 USD", "JPY 2500", "$1,234.56", "not a price"]
for s in samples:
    m = price_pat.match(s)
    if m:
        print(f"  {s:<20s} → {m.groupdict()}")
    else:
        print(f"  {s:<20s} → no match")
```

You should see the structured groupdict for each valid input. The verbose form means you can re-read this regex in 6 months.

---

## Exercise 8.4 — Fuzzy matching with rapidfuzz

Build a small "canonical address" example.

```python
canonical_addresses = [
    "123 Main St, New York, NY 10001",
    "456 Broadway, New York, NY 10013",
    "789 Park Ave, New York, NY 10021",
    "101 Wall St, New York, NY 10005",
    "202 5th Ave, New York, NY 10010",
]

queries = [
    "123 Main Street, NYC",
    "456 Broadway, New York",
    "789 PARK AVENUE NEW YORK NY",
    "wall st 101",
    "5th ave 202",
    "1600 Pennsylvania Ave",   # not in our list
]

# Normalize candidates
canonical_norm = [normalize_for_match(a) for a in canonical_addresses]

for q in queries:
    q_norm = normalize_for_match(q)
    match, score, idx = process.extractOne(
        q_norm, canonical_norm,
        scorer=fuzz.WRatio,
    )
    print(f"{q!r:<45s}  →  {canonical_addresses[idx]!r:<45s}  (score {score})")
```

You should see correct matches for the first 5 (scores >75) and the Pennsylvania Ave entry with a low score that you'd filter out with a threshold.

---

## Exercise 8.5 — Build the fuzzy-matcher with a threshold

```python
def match_address(query: str, candidates: list[str], threshold: int = 70):
    """Returns (match, score) or (None, None) if below threshold."""
    q_norm = normalize_for_match(query)
    cand_norm = [normalize_for_match(c) for c in candidates]
    result = process.extractOne(q_norm, cand_norm, scorer=fuzz.WRatio)
    if result is None:
        return None, None
    match, score, idx = result
    if score < threshold:
        return None, score
    return candidates[idx], score

for q in queries:
    match, score = match_address(q, canonical_addresses, threshold=70)
    if match:
        print(f"✓ {q!r:<45s}  →  {match!r}  ({score})")
    else:
        print(f"✗ {q!r:<45s}  →  (no match; best score {score})")
```

`1600 Pennsylvania Ave` now correctly returns no match (score below threshold).

---

## Exercise 8.6 — Process a batch

For a real catalog (say, 100k product descriptions to dedup against), batch processing matters. rapidfuzz's `cdist` returns the full pairwise score matrix:

```python
from rapidfuzz import process

# Suppose 1000 query strings against 200 canonical
import random
random.seed(42)
queries_big = [
    f"product number {random.randint(1, 100)} variant {random.choice('ABCD')}"
    for _ in range(1000)
]
canonical_big = [f"product number {i} variant A" for i in range(1, 101)]

# Time the batch
import time
t0 = time.perf_counter()
scores = process.cdist(
    [normalize_for_match(q) for q in queries_big],
    [normalize_for_match(c) for c in canonical_big],
    scorer=fuzz.WRatio,
)
print(f"cdist of {len(queries_big)} × {len(canonical_big)} in {time.perf_counter()-t0:.2f}s")
print(f"shape: {scores.shape}")

# Best match per query
import numpy as np
best_indices = np.argmax(scores, axis=1)
best_scores = np.max(scores, axis=1)
print(f"sample matches:")
for q, idx, s in list(zip(queries_big, best_indices, best_scores))[:5]:
    print(f"  {q!r:<35s} → {canonical_big[idx]!r}  ({s})")
```

You should see 1000 × 100 = 100k comparisons in well under a second.

---

## Exercise 8.7 — spaCy NER on free text

Apply NER to extract entities from realistic-shaped text.

```python
tickets = [
    "Customer reported water leakage at 123 Main St in Brooklyn on January 15, 2024. Refund amount: $250.",
    "Acme Corp is interested in our $500K enterprise plan, contact lisa@acme.com by next Friday.",
    "Tree in front of 456 Park Ave needs trimming. Reported by neighbor Mr. Smith.",
    "Pothole at 5th & Broadway, NYC. Reported 2024-01-15 14:30 by 5551234567.",
]

for ticket in tickets:
    doc = nlp(ticket)
    print(f"\n{ticket!r}")
    for ent in doc.ents:
        print(f"  {ent.label_:<10s}  {ent.text!r}")
```

You should see ORG (Acme Corp), MONEY ($250, $500K), DATE (January 15, 2024), GPE (Brooklyn, NYC), PERSON (Mr. Smith), CARDINAL (456, 5551234567).

The accuracy varies per category. For your own data, you'd want to validate on a labeled sample before relying on NER output downstream.

---

## Exercise 8.8 — Extract structured fields from tickets

Combine regex + NER to extract specific structured fields.

```python
def extract_ticket_fields(text: str) -> dict:
    """Extract address, dollar_amount, date, and reporter from a free-text ticket."""
    doc = nlp(text)
    out = {
        "raw": text,
        "address": None,
        "dollar_amount_cents": None,
        "date_iso": None,
        "reporter": None,
    }
    # NER fields
    for ent in doc.ents:
        if ent.label_ == "MONEY" and out["dollar_amount_cents"] is None:
            # Parse "$250" or "$500K"
            m = re.match(r"\$?(\d+(?:\.\d+)?)(K|M|k|m)?", ent.text)
            if m:
                amt = float(m.group(1))
                if m.group(2) in ("K", "k"): amt *= 1000
                elif m.group(2) in ("M", "m"): amt *= 1_000_000
                out["dollar_amount_cents"] = int(amt * 100)
        if ent.label_ in ("DATE",) and out["date_iso"] is None:
            out["date_iso"] = ent.text
        if ent.label_ == "PERSON" and out["reporter"] is None:
            out["reporter"] = ent.text

    # Regex for addresses (street number + street name)
    addr_m = re.search(r"\d+\s+[A-Z][a-z]+\s+(St|Ave|Avenue|Street|Blvd|Boulevard|Rd|Road)", text)
    if addr_m:
        out["address"] = addr_m.group(0)

    return out

for t in tickets:
    print(extract_ticket_fields(t))
    print()
```

You should see structured dicts for each ticket. This is the pattern for converting text-heavy data (support tickets, contracts, emails) into queryable tables.

---

## Exercise 8.9 — Vectorized string ops in Polars

Speed comparison: per-row Python vs Polars's built-in string ops.

```python
import time
df = pl.DataFrame({
    "text": [f"Café São Paulo, Brazil — record #{i}" for i in range(100000)],
})

# Per-row Python
t0 = time.perf_counter()
result_python = df.with_columns(
    norm=pl.col("text").map_elements(normalize_text, return_dtype=pl.Utf8)
)
t_python = time.perf_counter() - t0
print(f"per-row Python: {t_python:.3f}s")

# Polars built-in (vectorized C)
t0 = time.perf_counter()
result_polars = df.with_columns(
    norm=pl.col("text")
        .str.strip_chars()
        .str.to_lowercase()
        .str.replace_all(r"\s+", " "),
)
t_polars = time.perf_counter() - t0
print(f"polars vectorized: {t_polars:.3f}s")

print(f"speedup: {t_python / t_polars:.1f}x")
```

You should see ~10-50× speedup. **For ingest-time pipelines processing millions of rows, always use the vectorized string ops.**

---

## Exercise 8.10 — Long-tail categorical handling

Apply the "top-N + other" pattern to a free-text category column.

```python
# Simulate skewed category distribution
random.seed(42)
categories = ["sales", "support", "billing", "technical", "feedback"]
weights = [0.4, 0.3, 0.15, 0.10, 0.05]
typed_categories = random.choices(categories, weights=weights, k=5000)

# Add long-tail noise — typos and weird variants
noise_variants = [
    "Sales", "SALES", "sale", "sals",
    "Support", "Customer Support", "supprt",
    "Bllling", "Billing  ", " billing",
    "tech", "Tech Support",
    "FB", "Feedback Form",
] + [f"misc-{i}" for i in range(500)]
typed_categories += random.choices(noise_variants, k=500)

df = pl.DataFrame({"raw_category": typed_categories})

# Step 1: normalize
df = df.with_columns(
    normalized=pl.col("raw_category")
        .str.to_lowercase()
        .str.strip_chars()
        .str.replace_all(r"\s+", " "),
)

# Step 2: dictionary collapse
synonyms = {
    "sales": "sales", "sale": "sales", "sals": "sales",
    "support": "support", "customer support": "support", "supprt": "support",
    "billing": "billing", "bllling": "billing",
    "tech": "technical", "technical": "technical", "tech support": "technical",
    "feedback": "feedback", "fb": "feedback", "feedback form": "feedback",
}

df = df.with_columns(
    canonical=pl.col("normalized").map_elements(
        lambda s: synonyms.get(s, "other"),
        return_dtype=pl.Utf8,
    ),
)

print(df.group_by("canonical").len().sort("len", descending=True))
```

You should see the 5 canonical categories plus "other" (containing the 500 long-tail values you couldn't map).

**This pattern handles 95% of free-text categorical fields.** For the remaining 5% where synonyms can't be enumerated, week 12's embeddings approach takes over.

---

## Submission checklist

- [ ] Demonstrated NFC vs NFD non-equality; fixed mojibake with ftfy
- [ ] `normalize_text` and `normalize_for_match` functions written
- [ ] Verbose, anchored, named-group regex for prices
- [ ] Fuzzy address matcher hits ≥ 85% precision on labeled queries
- [ ] Batch `cdist` matches 1000×100 in under a second
- [ ] spaCy NER applied to 4+ ticket-shaped strings; entities extracted correctly
- [ ] `extract_ticket_fields()` parses structured info from text
- [ ] Polars vectorized string ops shown to be 10×+ faster than `.map_elements`
- [ ] Long-tail categorical collapsed to canonical + "other"

---

## What you just did

You can clean Unicode without losing data, write regexes that survive a code review six months later, build a fuzzy address matcher in 20 lines, extract structured fields from free-form support tickets, and process millions of rows with vectorized string ops.

This is the toolkit for any real-world cleaning project. By week 12 we add semantic similarity via embeddings — the modern technique that catches the ones rapidfuzz misses.

---

**Next**: [Week 09: Joins at Scale →](../week-09-joins-at-scale/readme.md)
