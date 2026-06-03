# Week 07: Theory — Normalization and Schema Enforcement

The data is clean enough that you trust the numbers (weeks 04-06). Now it needs to be **shaped** before downstream consumers can use it: consistent types, normalized representations, and **schemas enforced at every boundary**.

Without enforcement, every consumer reimplements the cleaning logic; every consumer gets a slightly different version; meetings ensue. With enforcement, there's one definition of what each table looks like, and the pipeline fails loudly when reality diverges.

---

## Part 1: The two layers — types and semantics

A column has two kinds of correctness:

1. **Type correctness** — the value is a string, integer, boolean, datetime, decimal
2. **Semantic correctness** — within its type, it follows the rules (ISO country code; positive integer; in {Manhattan, Bronx, ...}; etc.)

Type correctness comes from the file format (Parquet's schema) or the deserializer (Pydantic, pandera). Semantic correctness comes from your explicit checks (week 04). Both matter; conflating them is one of the rookie mistakes.

---

## Part 2: Type coercion — the rules

Three responses to a value that doesn't match the declared type:

| Strategy | What | When |
|---|---|---|
| **Upcast (widen)** | Parse and convert if possible | "123" → 123 when the column is declared int |
| **Error (strict)** | Refuse the load | The column is declared `currency: USD/EUR/GBP/JPY`, and `XYZ` shows up |
| **Quarantine** | Set aside the bad row; keep the rest | Most production pipelines (week 04 pattern) |
| **Coerce to null** | Replace bad value with null; flag | When nulls are acceptable downstream |

The rule of thumb:

- **Bronze layer** (week 03): lenient. Take whatever the source gives you. Strings everywhere is fine.
- **Silver layer**: strict. Enforce types and semantics. Reject or quarantine on failure.
- **Gold layer**: pre-validated. Downstream consumers shouldn't have to check.

---

## Part 3: The canonical messy fields

Five fields where the input is *always* messy and downstream needs canonical form.

### Dates and times

```
2024-01-15
2024-01-15T08:30:00
2024-01-15T08:30:00Z
2024-01-15T08:30:00-05:00
1/15/2024
15/1/2024
January 15, 2024
1705310200            ← unix epoch
```

Canonical: **ISO 8601 with timezone, stored as `timestamp with time zone` (`TIMESTAMPTZ`) in UTC.**

Pitfalls:
- **Ambiguous formats**: `01/02/2024` could be Jan 2 (US) or Feb 1 (EU). Refuse to guess; require unambiguous input.
- **Naive datetimes**: `datetime` without timezone is a bug 99% of the time. Always store UTC offset.
- **Unix epochs in seconds vs milliseconds**: `1705310200` is 2024; `1705310200000` is also 2024 (but in ms). Detect by magnitude.

Use Polars's `str.to_datetime(...)` or pandas `pd.to_datetime(..., utc=True)`. Convert to UTC at the boundary; never store local time.

### Phone numbers

```
(212) 555-1234
+1-212-555-1234
12125551234
2125551234
0044 20 7946 0958
```

Canonical: **E.164** — `+[country code][number]`, no spaces or punctuation. Example: `+12125551234`.

Use the **phonenumbers** library:

```python
import phonenumbers
parsed = phonenumbers.parse("(212) 555-1234", "US")
e164 = phonenumbers.format_number(parsed, phonenumbers.PhoneNumberFormat.E164)
# '+12125551234'
```

Pitfalls:
- The library wants a default country if the input has no `+`. Either know your data's country or refuse to parse.
- "Phone-like" strings (extensions, SMS short codes, fictional 555-numbers) need explicit handling.

### Addresses

Canonical address normalization is essentially impossible without a geocoder. **Don't try to normalize addresses by hand.** Use:

- **`libpostal`** (open source) for parsing free-form addresses into structured components
- **Google / Mapbox / HERE geocoding API** (commercial) for full normalization + geocoding
- **US Census ZIP/ZCTA lookups** for US-only (week 10)

The minimal you should do without a geocoder:

```python
# Lowercase + collapse whitespace + standardize common abbreviations
import re
ABBR = {
    "street": "st", "avenue": "ave", "boulevard": "blvd",
    "road": "rd", "drive": "dr", "lane": "ln",
}
def normalize_address(s):
    s = s.lower().strip()
    s = re.sub(r"\s+", " ", s)
    for long, short in ABBR.items():
        s = re.sub(rf"\b{long}\b\.?", short, s)
    return s
```

For week 10's enrichment lab you'll add real geocoding via Nominatim.

### Currency / money

**Never use float for money.** Floats can't represent 0.1 exactly, and accumulated errors break accounting reports.

Canonical: **integer cents** (or smaller unit if you trade fractions of a cent). For Bitcoin/eth: integer satoshis/wei.

```python
# Bad
amount_usd = 19.99
total = amount_usd * 100   # might be 1999.0000000000002

# Good
amount_cents = 1999
total = amount_cents       # always exact
```

For display, format with a locale-aware library (Babel):

```python
from babel.numbers import format_currency
format_currency(1999 / 100, "USD")   # '$19.99'
```

Multi-currency tables need both amount AND currency:

```
| amount_cents | currency |
|-------------|----------|
| 1999        | USD      |
| 1800        | EUR      |
```

Never store "amount_usd_equivalent" without showing the FX rate and date used; conversions go stale.

### Units

If you have a column called "weight", document the unit. Mixed-unit columns are a perennial source of bugs:

```
"weight": 70   # kg? lb? oz? 
```

Either:
- Pick one canonical unit; convert on ingest
- Store TWO columns: `weight_value` and `weight_unit`
- Use a typed dimensional library (`pint`) if you need real unit arithmetic

The Mars Climate Orbiter (1999) destroyed itself because of an unannounced unit mix-up. Your dashboard isn't worth $327M but the principle is the same.

---

## Part 4: Schema frameworks — pandera, Pydantic, pyarrow

Three tools, three concerns:

| Tool | Operates on | Strength |
|---|---|---|
| **pyarrow** | Whole tables; Parquet on-disk schemas | Type-strict; embedded in files |
| **pandera** | pandas / Polars DataFrames in memory | Schema + semantic checks; runs inline |
| **Pydantic** | Single records / rows / dicts | Row-level parsing & validation; great for API input |

### pyarrow — the storage-layer schema

When you write Parquet with explicit types, the schema is **embedded in the file**:

```python
import pyarrow as pa

schema = pa.schema([
    ("order_id", pa.string()),
    ("amount_cents", pa.int64()),
    ("currency", pa.string()),
    ("created_at", pa.timestamp("us", tz="UTC")),
])

table = pa.Table.from_pydict({...}, schema=schema)
pa.parquet.write_table(table, "out.parquet")
```

Anyone reading the Parquet gets the schema. No external definition file needed.

### pandera — the dataframe-layer schema

We saw pandera in week 04. The headline pattern for week 07 — strict mode + coercion:

```python
import pandera.polars as ppa
from pandera.polars import DataFrameSchema, Column, Check

schema = DataFrameSchema(
    columns={
        "order_id": Column(str, nullable=False),
        "amount_cents": Column(int, checks=Check.ge(0), nullable=False, coerce=True),
        "currency": Column(str, checks=Check.isin(["USD", "EUR", "GBP", "JPY"]), nullable=False),
    },
    strict=True,      # reject if extra columns appear
)

# coerce=True means pandera will try to cast types before validating
validated = schema.validate(df, lazy=True)
```

`strict=True` rejects unexpected columns. Combined with `coerce=True` per-column, you get strong typing + early failure.

### Pydantic — the row-level schema

For API responses, JSON streams, single-record validation:

```python
from pydantic import BaseModel, EmailStr, Field
from datetime import datetime

class Order(BaseModel):
    order_id: str = Field(pattern=r"^ord_[a-z0-9]+$")
    amount_cents: int = Field(ge=0)
    currency: str = Field(pattern=r"^(USD|EUR|GBP|JPY)$")
    customer_email: EmailStr
    created_at: datetime

# Parse + validate
o = Order(**raw_dict)
```

Pydantic gives you:
- Type coercion (`"123"` → `123` for an int field)
- Validation errors with structured paths
- JSON schema generation
- Performance (v2 is ~10× faster than v1)

For ingestion of API responses (week 03), Pydantic at the boundary is the cleanest pattern. For DataFrames, pandera. They complement.

---

## Part 5: Schema evolution — the production reality

Pipelines run for years. Schemas change. Five kinds of change:

| Change | Severity | Strategy |
|---|---|---|
| Add nullable column | Safe | Absorb; existing code keeps working |
| Add required column with default | Mostly safe | Backfill old data with default |
| Rename column | Breaking | Alias old name for N versions, then drop |
| Change type (widen) | Mostly safe | int32 → int64; bf16 → float32. Watch for narrowing reversal. |
| Change type (narrow) | Breaking | varchar(50) → varchar(10) may truncate. Avoid. |
| Change semantics | Disaster | Same column name, different meaning — this is the contract-violation case. Version your schema. |

The medallion architecture gives you space:

- **Bronze**: append-only. Old format files stay; new ones go in with new schema.
- **Silver**: enforce current schema. Run the cleaning logic per version.
- **Gold**: stable; consumers see one schema.

For data formats:

- **Apache Iceberg** and **Delta Lake**: native schema evolution; rename / drop / type widening supported as metadata operations
- **Parquet**: silently allows additions; everything else breaks. The lakehouse formats exist to fix this gap.

### Versioning your schema

The contract pattern (week 04 Part 5) extended to versioned schemas:

```yaml
# orders_v2.yml
name: orders
version: 2.0.0
breaking_from_v1:
  - "amount" column renamed to "amount_cents"; semantic: now always cents (not dollars)
  - "country" required; previously nullable
schema:
  ...
migration:
  v1_to_v2: |
    SELECT
      order_id,
      amount * 100 AS amount_cents,   -- v1 stored dollars; v2 stores cents
      COALESCE(country, 'UNKNOWN') AS country,
      ...
    FROM orders_v1
```

When v3 comes you keep v1 → v2 → v3 migrations. Old data stays accessible.

---

## Part 6: The "schema firewall"

The senior pattern for a multi-team data platform:

```
External source ──┐
                  │
                  ▼
        ┌─────────────────────────┐
        │  Bronze (raw)            │   ← lenient
        └─────────┬───────────────┘
                  │
                  ▼  ← schema firewall (validate; error or quarantine)
        ┌─────────────────────────┐
        │  Silver (typed)          │   ← stable schema
        └─────────┬───────────────┘
                  │
                  ▼
        ┌─────────────────────────┐
        │  Gold (modeled)          │   ← business semantics
        └─────────────────────────┘
                                          ▲
                                          │
                              External consumers join here
```

The "firewall" is a step that **runs validation in CI on every change** and **runs it on every load**. Bronze can drift (the upstream source's problem); silver does not (your team's problem).

Code-organization-wise:

```
project/
  ingestion/         # source-specific; reads bronze; tolerates messiness
  schemas/           # pandera + pyarrow definitions
  transformations/   # SQL / dbt models; produce silver and gold
  validation/        # checks run between layers
  contracts/         # YAML files; reviewed with stakeholders
```

---

## Part 7: Things that look like data quality but aren't

| Concern | Where it actually lives |
|---|---|
| "I want to add a check that revenue > 0" | Validation (week 04) |
| "I want to make sure phone numbers are E.164" | Normalization (this week) |
| "I want one row per customer, not duplicates" | Dedup (week 05) |
| "I want to fill in missing zip codes" | Imputation (week 06) |
| "I want a documented business definition of 'active user'" | Data contracts + dbt models |

Drawing these lines makes pipelines maintainable. Putting business rules into ingestion code, normalization into validation, and validation into dbt makes everything tangled.

---

## Part 8: Anti-patterns

| Anti-pattern | Cost |
|---|---|
| Floats for money | Accounting errors that compound |
| Storing local time, no timezone | Daylight-saving bugs twice a year |
| String "true"/"false" instead of bool | Comparisons silently fail |
| `dtype=object` columns in pandas | Slow, memory-hungry, hides bugs |
| One column with mixed units | The Mars Orbiter pattern |
| Schema "documented" in the team's Notion | Drifts away from reality in months |
| Validation in the consumer code | N consumers, N inconsistent implementations |

---

## What's next

In [lab.md](lab.md) you'll:

- Normalize a deliberately messy CSV (mixed dtypes, half-baked dates, three currencies, mixed phone formats) into canonical Parquet
- Use Pydantic for row-level parsing with informative error messages
- Build a pandera schema with strict mode + coercion
- Set up a schema firewall: validate as CI check before silver writes
- Demonstrate schema evolution: v1 → v2 migration with backfill
- Quarantine pattern with bad-row export

By end of week 07 you can take any source data and produce a typed, documented, contracted Parquet that downstream consumers can rely on for years.
