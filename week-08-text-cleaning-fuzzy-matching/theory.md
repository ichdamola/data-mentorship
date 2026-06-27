# Week 08: Theory - Text Cleaning and Fuzzy Matching

Half of every real-world dataset is text in a column that pretends to be structured. **Product names**, **addresses**, **free-form comments**, **support ticket descriptions**, **user-generated tags**. None of it is clean. Cleaning it is the difference between a dashboard that's accurate and one that pretends to be.

This week is the toolkit: **Unicode normalization** (the thing most engineers wing badly), **regex patterns** that survive code review, **fuzzy string distance** (rapidfuzz), and **lightweight named-entity extraction** (spaCy / GLiNER) when you need entities pulled out of free text.

---

## Part 1: Unicode is not optional

The "I'll just lower-case it and strip whitespace" instinct fails the moment Unicode shows up. Five things to internalize:

### Encoding ≠ representation

A string `"café"` in memory could be:

- 4 code points: `c`, `a`, `f`, `é` (one code point for é) - **NFC**
- 5 code points: `c`, `a`, `f`, `e`, ` ́ ` (e + combining acute) - **NFD**

Both render identically. Both compare *unequal* with `==`. **You must normalize.**

```python
import unicodedata
s1 = "café"
s2 = "café"
print(s1 == s2)                                          # False
print(unicodedata.normalize("NFC", s1) == unicodedata.normalize("NFC", s2))   # True
```

The forms:

| Form | Description |
|---|---|
| **NFC** (Composed) | Combine where possible - "é" as one code point |
| **NFD** (Decomposed) | Split - "é" as e + combining mark |
| **NFKC** (Compatibility composed) | Like NFC but also normalize compatibility characters (full-width digits → ASCII) |
| **NFKD** (Compatibility decomposed) | Like NFD with compatibility folding |

**For deduplication and matching, normalize to NFC first.** Always.

### Mojibake - the wrong-encoding curse

Mojibake is what you see when text was decoded with the wrong encoding. The classic: a string was UTF-8 but got interpreted as Latin-1, then re-encoded.

```python
broken = "café".encode("utf-8").decode("latin-1")
print(broken)    # 'cafÃ©'
```

Recovery is sometimes possible:

```python
fixed = broken.encode("latin-1").decode("utf-8")
print(fixed)     # 'café'
```

The **ftfy** library ("fixes text for you") automates this for many common cases. Use it when bronze data has known mojibake.

### BOMs (Byte-Order Marks)

A BOM is `﻿` at the start of a file. CSV writers from Excel sometimes prepend one. It silently breaks column name matching:

```python
df.columns
# Index(['﻿id', 'name', ...])   ← the first column secretly has BOM
```

Handle on read:

```python
pd.read_csv("file.csv", encoding="utf-8-sig")    # strips BOM
```

### Confusables - homoglyphs

Cyrillic `а` (U+0430) and Latin `a` (U+0061) look identical. Spammers exploit this. For high-stakes matching (KYC, fraud), use the `confusables` library or strip non-Latin scripts explicitly.

### Always declare the encoding

```python
# Good
with open("file.csv", encoding="utf-8") as f: ...

# Risky - depends on OS locale
with open("file.csv") as f: ...
```

`utf-8` should be your default for everything you write. Read with `utf-8-sig` for safety on files that might have BOMs.

---

## Part 2: Practical Unicode recipe

Most cleaning pipelines start with this 5-line normalize:

```python
import unicodedata
import re

def normalize_text(s: str) -> str:
    if s is None: return ""
    s = unicodedata.normalize("NFKC", s)        # composed + compatibility folding
    s = s.lower()
    s = re.sub(r"\s+", " ", s).strip()           # collapse whitespace
    return s
```

For dedup / matching, often add:

```python
def normalize_for_match(s: str) -> str:
    s = normalize_text(s)
    s = unicodedata.normalize("NFKD", s)
    s = "".join(c for c in s if not unicodedata.combining(c))   # strip accents
    s = re.sub(r"[^a-z0-9 ]", "", s)                              # ASCII alphanum + space only
    return s

normalize_for_match("Café SÃO Paulo!")    # 'cafe sao paulo'
```

This is the senior version of the "lowercase + strip" reflex. You'll see it in every entity-resolution pipeline.

---

## Part 3: Regex - write the read-later version

Regex is the universal tool for "find this pattern in messy text." It's also notorious for write-once unreadable code.

Three habits that scale:

### Use named groups + verbose mode

```python
import re

# Awful
pat = r"(\d{3})-(\d{3})-(\d{4})"

# Better - readable, named groups
pat = re.compile(r"""
    (?P<area>\d{3})       # area code
    [-.\s]?               # optional separator
    (?P<exchange>\d{3})   # exchange code  
    [-.\s]?               # optional separator
    (?P<line>\d{4})       # line number
""", re.VERBOSE)

m = pat.match("212-555-1234")
print(m.group("area"))    # '212'
```

`re.VERBOSE` lets you whitespace and comment a regex without affecting matching.

### Anchor explicitly

```python
# Bad - matches "abc123def"
re.match(r"\d+", "abc123def")

# Good - fully anchored
re.match(r"^\d+$", "abc123def")    # None - correctly rejects
```

`^` and `$` (with `re.MULTILINE` if needed) prevent partial-match surprises. For full-string matching, prefer `re.fullmatch()`.

### Watch for catastrophic backtracking

Regexes like `(a+)+b` against `aaaaaaaaaaaaaaaX` can take *exponential time*. The pattern: nested quantifiers with overlap.

The **`regex`** library (third-party) has a `TIMEOUT` parameter; the standard `re` library doesn't. For untrusted input, use `regex` with a timeout, or write the simpler pattern.

```python
import regex
try:
    regex.match(r"(a+)+b", "aaaaaaaaaaaaaaaX", timeout=0.5)
except TimeoutError:
    print("regex too slow")
```

The Russ Cox paper "Regular Expression Matching Can Be Simple and Fast" explains why some regex engines are quadratic-exponential. The takeaway: keep patterns simple; profile when in doubt.

---

## Part 4: String distance - rapidfuzz

Modern dedup, matching, search-as-you-type, autocorrect - all rest on string distance.

### Distance vs similarity

- **Distance**: 0 for identical, larger for more different
- **Similarity**: 0 for entirely different, 1 (or 100) for identical

Most libraries (rapidfuzz, fuzzywuzzy) report similarity scaled 0-100.

### The metrics that matter

| Metric | What | When |
|---|---|---|
| **Levenshtein** | Min single-char edits (insert/delete/substitute) | General typos |
| **Damerau-Levenshtein** | Levenshtein + transpositions | Common typing errors |
| **Jaro-Winkler** | Biased toward shared prefixes | Names |
| **`fuzz.ratio`** | Levenshtein-based similarity 0-100 | Default |
| **`fuzz.partial_ratio`** | Best score of any substring | One string is contained in or extends another |
| **`fuzz.token_sort_ratio`** | Sort words, then compare | Word-order doesn't matter |
| **`fuzz.token_set_ratio`** | Set of words, ignore order + duplicates | Word-order doesn't matter AND extras allowed |
| **`fuzz.QRatio`** / **`WRatio`** | Heuristic combination of the above | Production default - picks the right one |

```python
from rapidfuzz import fuzz, process

fuzz.ratio("Apple Inc.", "Apple Incorporated")            # 78 - quite different
fuzz.partial_ratio("Apple", "Apple Inc.")                  # 100 - "Apple" is in "Apple Inc."
fuzz.token_sort_ratio("Apple Inc.", "Inc. Apple")          # 100 - same tokens, different order
fuzz.token_set_ratio("Apple Inc., USA", "Apple, Inc. USA") # 100 - same tokens
fuzz.WRatio("Apple Inc.", "apple incorporated")            # ~88 - combined heuristic
```

### Find best match in a list

```python
candidates = ["Apple Inc.", "Microsoft Corp.", "Alphabet Inc."]
match, score, idx = process.extractOne("aple inc", candidates, scorer=fuzz.WRatio)
print(match, score)    # 'Apple Inc.' 86
```

`extract` and `extractOne` are the right APIs for "find me the best match for this query in this list." Polished, fast, what you want.

### Speed

rapidfuzz is **C-backed**; ~50-100× faster than `fuzzywuzzy` (pure Python). For matching 1M queries against 100k candidates: rapidfuzz finishes in a coffee break, fuzzywuzzy in a workday.

For at-scale matching, also consider **datasketch** (MinHash) and **rapidfuzz's `cdist`** for batch all-pairs.

---

## Part 5: Lightweight NER (Named Entity Recognition)

Sometimes a text field hides structured data. "Customer reported issue with PRODUCT_X in NYC on Monday" can be parsed for product, location, and date.

Two open-source options for getting entities out of text:

### spaCy

The classic. Pre-trained on news / web text; recognizes PERSON, ORG, GPE (geo-political entity), DATE, MONEY, etc.

```python
import spacy
nlp = spacy.load("en_core_web_sm")
doc = nlp("Acme Corp announced $1.2M in funding on January 15, 2024 in Brooklyn.")
for ent in doc.ents:
    print(ent.text, ent.label_)
# 'Acme Corp' ORG
# '$1.2M' MONEY
# 'January 15, 2024' DATE
# 'Brooklyn' GPE
```

Strengths: fast, well-supported, runs in 50ms/document on CPU.
Weaknesses: fixed label set; struggles on domain-specific data without retraining.

### GLiNER (2024+)

A newer transformer-based NER that lets you **specify the labels at inference time** - no retraining. "Find me products, prices, and locations" using natural language labels.

```python
from gliner import GLiNER
model = GLiNER.from_pretrained("urchade/gliner_medium-v2.1")
entities = model.predict_entities(
    "Acme Corp announced $1.2M in funding on January 15, 2024 in Brooklyn.",
    labels=["company", "money_amount", "date", "city"]
)
for e in entities:
    print(e["text"], e["label"])
```

Slower than spaCy but vastly more flexible. For one-off domain extraction without retraining a model, GLiNER is the modern go-to.

### When NER isn't worth it

For structured-but-messy fields (orders, products), regex + rule-based parsing is often better than NER:

```python
# "$45.50 USD" → 4550, "USD"
pat = re.compile(r"\$?(?P<amount>\d+(?:\.\d+)?)\s*(?P<currency>[A-Z]{3})?")
m = pat.match("$45.50 USD")
# Works for most formats; faster than spaCy
```

NER shines on truly free text (support tickets, contract clauses, news articles).

---

## Part 6: Tokenization - the gateway concept

Tokenization splits text into units. Three common granularities:

| Tokenization | Result on "Don't worry, be happy" |
|---|---|
| Whitespace | `["Don't", "worry,", "be", "happy"]` |
| Word (proper) | `["Do", "n't", "worry", ",", "be", "happy"]` |
| Subword (BPE) | `["Don", "'t", "worry", ",", "be", "happy"]` (varies by model) |

For dedup and matching, word-level tokenization (via spaCy or `nltk.word_tokenize`) is usually enough. For embedding (week 12), you'll use the tokenizer that ships with your embedding model.

**Polars `str.split` is the fast option** for simple whitespace tokenization at scale:

```python
df.select(pl.col("description").str.split(" ").alias("tokens"))
```

---

## Part 7: Text-as-categorical - beware the long tail

When you have a free-text column ("city", "department", "category"), the distinct-value count often has a Zipf distribution:

- Top 50 values cover 80% of rows
- Next 200 values cover 15%
- Long tail of 10,000+ singletons covers the last 5%

Strategies for handling:

| Strategy | When |
|---|---|
| **Keep top N as buckets, "other" for rest** | Most common for classification labels |
| **Standardize via dictionary** | Replace synonyms ("USA" / "United States" / "U.S.A.") with canonical form |
| **Embed (week 12)** | When semantic similarity matters and you'll cluster |
| **Drop** | When the column is too noisy to be useful |

The "top N + other" pattern is the right default. Plot the value-count distribution first; the elbow tells you N.

---

## Part 8: Anti-patterns

| Anti-pattern | Why bad |
|---|---|
| `.lower()` without NFC | Misses combining-character cases |
| Strip whitespace but not zero-width unicode chars (`​`, `﻿`) | Invisible characters break matching |
| Regex with no anchors | Substring matches surprise you |
| Levenshtein on long strings | O(n×m); slow for 100+ char strings |
| Treating accents as significant | "Jose" and "José" are usually the same person |
| Per-row Python loops on string operations | 10-100× slower than vectorized Polars/pandas string ops |

The senior pattern across all of these: **normalize aggressively at ingest; preserve original for audit**.

---

## Part 9: Connect to the rest of the curriculum

- **Week 04 (quality)**: Validation includes string-pattern checks. Pandera's `Check.str_matches(...)`.
- **Week 05 (dedup)**: Already used rapidfuzz blocking + scoring.
- **Week 07 (schema)**: The Pydantic regex patterns for order_id, phone, etc. are tiny text validators.
- **Week 12 (embeddings)**: When text dedup with rapidfuzz hits its limits, embeddings take over.

This week's tools become the foundation for the rest - every later week assumes you can clean text into canonical form before doing anything with it.

---

## What's next

In [lab.md](lab.md) you'll:

- Diagnose and fix mojibake; normalize Unicode end-to-end
- Build named-group regexes for parsing common patterns (prices, dates, IDs)
- Use rapidfuzz to build a fuzzy address matcher
- Run spaCy NER on real free-text data (NYC 311 service requests)
- Build a "string-to-canonical" pipeline for a column of your choice
- Measure speed: per-row vs vectorized vs C-backed

By end of week 08 you can take any text column from any source and produce a clean, normalized, matchable canonical version.
