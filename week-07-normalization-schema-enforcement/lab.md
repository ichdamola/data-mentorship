# Week 07: Lab — Build the Schema Firewall

You'll start from a deliberately messy CSV (broken dates, three currencies, three phone formats, mixed booleans) and end with a typed, validated Parquet that downstream consumers can rely on.

## Setup

```bash
uv add pandera pyarrow pydantic phonenumbers babel polars
```

```python
import csv
import io
import polars as pl
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from pydantic import BaseModel, EmailStr, Field, ValidationError
from datetime import datetime
import phonenumbers
from babel.numbers import format_currency
from pathlib import Path
```

---

## Exercise 7.1 — Generate the messy CSV

```python
messy_csv = """order_id,customer_email,amount,currency,phone,country,created_at,active
ORD-001,alice@example.com,19.99,USD,(212) 555-1234,US,2024-01-15 09:30:00,true
ORD-002,bob@EXAMPLE.com,2500,JPY,03-1234-5678,JP,2024-01-15T09:35:00Z,True
ord-003, carol+work@example.com, 45.50 ,eur,+44 20 7946 0958,GB,15/01/2024 09:40,1
ORD-004,dan@example.com,1799,USD,2125559876,US,2024-01-15 09:45:00,yes
ord-005,eve.com,89.99,USD,not a number,US,2024-01-15 09:50:00,t
ORD-006,frank@example.com,-100,USD,+12125550001,US,2024-01-15 09:55:00,FALSE
ORD-007,grace@example.com,99.99,USD,(212) 555-2000,US,01/15/2024,false
ORD-008,heather@example.com,150,EUR,+33 1 23 45 67 89,FR,2024-01-15 10:05:00,0
"""

Path("data/bronze").mkdir(parents=True, exist_ok=True)
with open("data/bronze/orders_messy.csv", "w") as f:
    f.write(messy_csv)

# Read it as everything-is-string to start
df_raw = pl.read_csv("data/bronze/orders_messy.csv", infer_schema_length=0)
print(df_raw)
```

8 rows, every column needing work:
- `order_id`: mixed case, hyphens vs lowercase prefix
- `customer_email`: mixed case, trailing spaces, plus-tags, one broken email
- `amount`: floats AND ints AND with units to convert; one negative
- `currency`: mixed case
- `phone`: 5 different formats including "not a number"
- `country`: ISO codes (good)
- `created_at`: 4 different date formats including ambiguous DD/MM vs MM/DD
- `active`: 6 different boolean encodings

This is what real ingest data looks like.

---

## Exercise 7.2 — Pydantic for row-level parsing

```python
from typing import Literal

class OrderRow(BaseModel):
    order_id: str = Field(pattern=r"^ord-\d+$")
    customer_email: EmailStr
    amount_cents: int = Field(ge=0)
    currency: Literal["USD", "EUR", "GBP", "JPY"]
    phone_e164: str = Field(pattern=r"^\+\d+$")
    country: str = Field(pattern=r"^[A-Z]{2}$")
    created_at: datetime
    active: bool

    @classmethod
    def from_raw(cls, raw: dict) -> "OrderRow":
        """Coerce a raw row dict into our canonical form. Raises ValidationError on failure."""
        # Strip + lowercase order_id
        order_id = raw["order_id"].strip().lower()

        # Strip + lowercase email; drop plus-tags
        email = raw["customer_email"].strip().lower()
        if "+" in email and "@" in email:
            local, domain = email.split("@", 1)
            local = local.split("+", 1)[0]
            email = f"{local}@{domain}"

        # Amount: handle either dollars or integer cents
        amt_raw = float(str(raw["amount"]).strip())
        if "." in str(raw["amount"]):
            amount_cents = int(round(amt_raw * 100))
        else:
            amount_cents = int(amt_raw)
        # If JPY (no fractional), divide by 100 only if amount already in cents
        # (We're punting the truth-source problem; in production this is the contract.)

        # Currency
        currency = raw["currency"].strip().upper()

        # Phone via phonenumbers
        try:
            parsed_phone = phonenumbers.parse(raw["phone"], raw["country"])
            phone_e164 = phonenumbers.format_number(parsed_phone, phonenumbers.PhoneNumberFormat.E164)
        except phonenumbers.NumberParseException:
            raise ValueError(f"unparseable phone: {raw['phone']!r}")

        # Country
        country = raw["country"].strip().upper()

        # Created_at — explicit list of formats; refuse to guess
        ts_str = raw["created_at"].strip()
        ts = None
        for fmt in [
            "%Y-%m-%d %H:%M:%S",
            "%Y-%m-%dT%H:%M:%S",
            "%Y-%m-%dT%H:%M:%SZ",
            "%d/%m/%Y %H:%M",
            "%m/%d/%Y",
            "%d/%m/%Y",
        ]:
            try:
                ts = datetime.strptime(ts_str, fmt)
                break
            except ValueError:
                continue
        if ts is None:
            raise ValueError(f"unparseable date: {ts_str!r}")

        # Boolean — exhaustive list
        active_raw = str(raw["active"]).strip().lower()
        true_values = {"true", "t", "yes", "y", "1"}
        false_values = {"false", "f", "no", "n", "0"}
        if active_raw in true_values:
            active = True
        elif active_raw in false_values:
            active = False
        else:
            raise ValueError(f"unparseable bool: {raw['active']!r}")

        return cls(
            order_id=order_id, customer_email=email, amount_cents=amount_cents,
            currency=currency, phone_e164=phone_e164, country=country,
            created_at=ts, active=active,
        )
```

Now process the rows:

```python
clean_rows = []
quarantine_rows = []

for row in df_raw.iter_rows(named=True):
    try:
        order = OrderRow.from_raw(row)
        clean_rows.append(order.model_dump())
    except (ValueError, ValidationError) as e:
        quarantine_rows.append({**row, "_error": str(e)[:200]})

print(f"clean:       {len(clean_rows)}")
print(f"quarantine:  {len(quarantine_rows)}")
print()
print("clean sample:")
for r in clean_rows[:3]: print("  ", r)
print()
print("quarantine reasons:")
for r in quarantine_rows: print("  ", r["order_id"], "→", r["_error"][:100])
```

You should see ~6 clean, ~2 quarantined (eve.com isn't a valid email; -100 is negative; "not a number" can't parse).

---

## Exercise 7.3 — Write canonical Parquet with an explicit schema

```python
schema = pa.schema([
    pa.field("order_id", pa.string(), nullable=False),
    pa.field("customer_email", pa.string(), nullable=False),
    pa.field("amount_cents", pa.int64(), nullable=False),
    pa.field("currency", pa.string(), nullable=False),
    pa.field("phone_e164", pa.string(), nullable=False),
    pa.field("country", pa.string(), nullable=False),
    pa.field("created_at", pa.timestamp("us"), nullable=False),
    pa.field("active", pa.bool_(), nullable=False),
])

# Build pyarrow table
df_clean = pl.DataFrame(clean_rows)
print(df_clean.schema)

Path("data/silver").mkdir(parents=True, exist_ok=True)
table = df_clean.to_arrow()

# Re-cast to the declared schema (forces correctness)
table_typed = table.cast(schema)
pq.write_table(table_typed, "data/silver/orders.parquet")

# Quarantine to its own Parquet
if quarantine_rows:
    Path("data/quarantine").mkdir(parents=True, exist_ok=True)
    pl.DataFrame(quarantine_rows).write_parquet("data/quarantine/orders_bad.parquet")

print(f"\nwrote {len(clean_rows)} rows to data/silver/orders.parquet")
```

Inspect the written Parquet's schema (the schema travels with the file):

```python
pf = pq.ParquetFile("data/silver/orders.parquet")
print(pf.schema_arrow)
```

You should see your exact pyarrow schema. **Anyone who reads this file gets the types automatically.** That's the storage-layer schema in action.

---

## Exercise 7.4 — Pandera schema with strict + coerce

```python
import pandera.polars as ppa
from pandera.polars import DataFrameSchema, Column, Check

schema_pa = DataFrameSchema(
    columns={
        "order_id": Column(str, checks=Check.str_matches(r"^ord-\d+$"), nullable=False),
        "customer_email": Column(str, nullable=False),
        "amount_cents": Column(int, checks=Check.ge(0), nullable=False, coerce=True),
        "currency": Column(str, checks=Check.isin(["USD", "EUR", "GBP", "JPY"]), nullable=False),
        "phone_e164": Column(str, checks=Check.str_matches(r"^\+\d+$"), nullable=False),
        "country": Column(str, checks=Check.str_matches(r"^[A-Z]{2}$"), nullable=False),
        "active": Column(bool, nullable=False),
    },
    strict=True,  # reject unexpected columns
)

# Validate — should pass on the clean data we just produced
df_silver = pl.read_parquet("data/silver/orders.parquet")
df_silver_subset = df_silver.drop("created_at")   # pandera + polars: timestamp checks slightly different

try:
    schema_pa.validate(df_silver_subset, lazy=True)
    print("✓ all rows pass pandera validation")
except ppa.errors.SchemaErrors as e:
    print("✗ failures:")
    print(e.failure_cases.head())
```

Now break it on purpose — add a row with `currency='XYZ'` and watch pandera fail:

```python
bad = df_silver_subset.head(1).with_columns(currency=pl.lit("XYZ"))
df_test = pl.concat([df_silver_subset, bad])
try:
    schema_pa.validate(df_test, lazy=True)
except ppa.errors.SchemaErrors as e:
    print("Caught:")
    print(e.failure_cases.head())
```

You should see the XYZ row called out with the failed check.

---

## Exercise 7.5 — The schema firewall as a CI gate

Wrap validation in a function you'd run as part of every pipeline:

```python
def validate_silver(parquet_path: str, schema) -> tuple[int, int]:
    """Returns (rows_passed, rows_failed). Raises if any failure."""
    df = pl.read_parquet(parquet_path)
    df_subset = df.drop("created_at")

    try:
        schema.validate(df_subset, lazy=True)
        return len(df), 0
    except ppa.errors.SchemaErrors as e:
        n_bad = len(set(e.failure_cases["index"].drop_nulls().to_list()))
        raise RuntimeError(
            f"silver schema violations: {n_bad} rows. "
            f"Top errors:\n{e.failure_cases.head(5)}"
        )

# Run as a "CI check"
try:
    passed, failed = validate_silver("data/silver/orders.parquet", schema_pa)
    print(f"✓ schema check passed: {passed} rows clean")
except RuntimeError as e:
    print(f"✗ schema check FAILED:")
    print(e)
    raise
```

Put this in your CI / pipeline at the bronze → silver boundary. **The schema firewall pattern means broken data never reaches gold.**

---

## Exercise 7.6 — Schema evolution: v1 → v2 migration

Suppose your v1 schema had a single `amount` column (in dollars as a float), and v2 standardizes on `amount_cents` (integer). Build the migration.

```python
# Simulate v1 data
v1_data = pl.DataFrame({
    "order_id": ["ord-100", "ord-101", "ord-102"],
    "amount": [19.99, 45.50, 89.99],     # dollars as floats
    "currency": ["USD", "EUR", "USD"],
})
v1_data.write_parquet("data/bronze/orders_v1.parquet")

# Migration to v2
v2_data = (
    pl.read_parquet("data/bronze/orders_v1.parquet")
    .with_columns(
        amount_cents=(pl.col("amount") * 100).round().cast(pl.Int64),
    )
    .drop("amount")
    .select("order_id", "amount_cents", "currency")
)
print(v2_data)

# Now both v1 and v2 silver paths exist
v2_data.write_parquet("data/silver/orders_v2.parquet")
```

In a real org, you keep the migration as a versioned SQL/dbt model. Old consumers point at v1; new ones at v2; eventually you sunset v1.

---

## Exercise 7.7 — Multi-currency handling

A common real-world question: "store the dollar equivalent or the local amount?"

The answer: **store BOTH, with FX rate provenance**.

```python
fx_rates_2024_01_15 = {"USD": 1.0, "EUR": 0.92, "GBP": 0.79, "JPY": 149.5}
fx_date = "2024-01-15"

df_with_usd = (
    df_silver
    .with_columns(
        usd_equivalent_cents=(
            pl.col("amount_cents") / pl.col("currency").map_elements(
                lambda c: fx_rates_2024_01_15.get(c, 1.0),
                return_dtype=pl.Float64,
            )
        ).round().cast(pl.Int64),
        fx_rate_used=pl.col("currency").map_elements(
            lambda c: fx_rates_2024_01_15.get(c, 1.0),
            return_dtype=pl.Float64,
        ),
        fx_rate_date=pl.lit(fx_date),
    )
)
print(df_with_usd.select("order_id", "amount_cents", "currency", "usd_equivalent_cents", "fx_rate_used", "fx_rate_date"))
```

**Never store `usd_equivalent` without `fx_rate_used` and `fx_rate_date`.** Rates change. A report that shows "Q1 revenue" needs to specify which rates were used; otherwise it's not reproducible.

---

## Exercise 7.8 — Format formatted output for humans

For dashboards and reports, format with locale awareness:

```python
for row in df_silver.head(5).iter_rows(named=True):
    formatted = format_currency(row["amount_cents"] / 100, row["currency"])
    print(f"{row['order_id']}: {formatted} (country {row['country']})")
```

The output:

```
ord-001: $19.99 (country US)
ord-002: ¥2,500 (country JP)
ord-003: €45.50 (country GB)
...
```

`babel` handles the symbols, decimal placement, and thousands separators for each locale. **For internationalized output, don't roll your own.**

---

## Exercise 7.9 — One-page summary of the firewall

Document your pipeline. Save this as `docs/schema.md`:

```markdown
# Orders schema

## v2.0.0 (current)

### Source
- Daily SFTP drop from `payments-team@example.com`
- Bronze landing: `data/bronze/orders_YYYY-MM-DD.csv`

### Silver schema
| Column          | Type         | Required | Constraints                   |
|-----------------|--------------|----------|-------------------------------|
| order_id        | string       | yes      | `^ord-\d+$`                   |
| customer_email  | string       | yes      | RFC 5322 email                |
| amount_cents    | int64        | yes      | ≥ 0                           |
| currency        | string       | yes      | ISO 4217; subset of USD/EUR/GBP/JPY |
| phone_e164      | string       | yes      | `^\+\d+$`                     |
| country         | string       | yes      | ISO 3166-1 alpha-2            |
| created_at      | timestamptz  | yes      | UTC                           |
| active          | bool         | yes      |                               |

### Quarantine rules
- Invalid email → quarantine
- Unparseable phone → quarantine
- Currency not in set → quarantine
- Amount < 0 → quarantine
- > 5% of rows quarantined → CI fails

### Migration from v1
- `amount` (float, dollars) → `amount_cents` (int): `cast(amount * 100 as bigint)`
- New required field: `phone_e164` — backfill from `phone` column via phonenumbers library

### Owner
- Team: data-mentorship-lab
- SLA: schema changes get 14 days notice
```

This is the artifact that goes in the repo, gets reviewed in PRs, and answers questions before they're asked.

---

## Submission checklist

- [ ] Messy CSV generated with 4+ kinds of inconsistency
- [ ] Pydantic `OrderRow` class parses + validates rows; bad rows raise
- [ ] Clean rows written as Parquet with explicit pyarrow schema
- [ ] pandera schema with `strict=True` + `coerce=True` validates the silver file
- [ ] `validate_silver()` function works as a CI gate
- [ ] v1 → v2 schema migration produces both formats
- [ ] Multi-currency table includes FX rate + FX date columns
- [ ] Babel-formatted output for human display
- [ ] One-page schema doc written

---

## What you just did

You built the schema firewall: bronze in (messy), silver out (typed, validated, documented). Downstream consumers get a Parquet with embedded schema, a pandera spec for validation, and a markdown doc that explains every column. That stack is the senior 2026 default for any production data pipeline.

Week 08 brings text cleaning + fuzzy matching — the same discipline applied to free-form text fields like product descriptions and support tickets.

---

**Next**: [Week 08: Text Cleaning + Fuzzy Matching →](../week-08-text-cleaning-fuzzy-matching/readme.md)
