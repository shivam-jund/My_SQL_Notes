# SQL & PostgreSQL Complete Notes — Part 4: Data Types (Numbers, Text, Boolean, Date & Time)

## 📑 Table of Contents
1. Why Data Types Matter
2. Whole Numbers: `SMALLINT`, `INTEGER`, `BIGINT`
3. Exact Decimals: `NUMERIC` / `DECIMAL`
4. Approximate Decimals: `REAL`, `FLOAT`, `DOUBLE PRECISION`
5. `FLOAT` vs `NUMERIC` (Comparison)
6. Text: `CHAR`, `VARCHAR`, `TEXT`
7. `CHAR` vs `VARCHAR` vs `TEXT` (Comparison)
8. `BOOLEAN`
9. Date & Time: `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`
10. `DATE` vs `TIME` vs `TIMESTAMP` vs `TIMESTAMPTZ` (Comparison)
11. Choosing the Right Data Type — Decision Map
12. Interview Q&A
13. Quick Revision Sheet
14. Cheat Sheet
15. Preview of Part 5

**📋 Series Coverage (Part 4):** `SMALLINT`, `INTEGER`/`INT`, `BIGINT`, `NUMERIC`/`DECIMAL` (precision & scale), `REAL`, `FLOAT`, `DOUBLE PRECISION`, `CHAR(n)`, `VARCHAR(n)`, `TEXT`, `BOOLEAN`, `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`

---

## 1. Why Data Types Matter

**Definition** — A data type tells PostgreSQL exactly what kind of value a column is allowed to hold.

**Why It Exists** — Without a declared type, the database wouldn't know whether `age` should accept `"twenty"` or `20`, or whether `price` should round to whole numbers or keep cents. Types enable validation, efficient storage, and correct sorting/math.

```sql
CREATE TABLE students (
    student_id     INTEGER,
    name           VARCHAR(100),
    age            INTEGER,
    fees           NUMERIC(10,2),
    is_active      BOOLEAN,
    admission_date DATE
);
```

---

## 2. Whole Numbers: `SMALLINT`, `INTEGER`, `BIGINT`

**Definition** — Store whole numbers (no decimal part), differing only by storage size and range.

**Why It Exists** — Choosing the right size balances storage efficiency against the range of values you actually need.

**Syntax**
```sql
column_name SMALLINT
column_name INTEGER   -- INT is a shorthand alias, identical
column_name BIGINT
```

**Parameters / Ranges**

| Type | Size | Range | Typical Use |
|---|---|---|---|
| `SMALLINT` | 2 bytes | -32,768 to 32,767 | Small counts (age, small quantities) |
| `INTEGER` / `INT` | 4 bytes | ≈ -2.1 billion to +2.1 billion | IDs, quantities, marks (default choice) |
| `BIGINT` | 8 bytes | ≈ -9 quintillion to +9 quintillion | Very large IDs, high-volume transaction counters |

**Example**
```sql
CREATE TABLE products (
    product_id     INTEGER,
    stock_quantity INTEGER
);
INSERT INTO products VALUES (1, 500);
```
**Output**
```
product_id | stock_quantity
1          | 500
```

⚠️ **Notes & Caveats**
- `INT` and `INTEGER` are exactly the same type — just different spellings.
- Using `BIGINT` "just in case" wastes storage on tables that will never approach billions of rows.

💡 **How to Choose**
- Default to `INTEGER` for IDs/counts. Only go to `BIGINT` if you genuinely expect > 2 billion rows/values (e.g., a high-throughput events table). Use `SMALLINT` only when you're certain the range is small and storage really matters (e.g., millions of rows).

---

## 3. Exact Decimals: `NUMERIC` / `DECIMAL`

**Definition** — Stores decimal numbers with **exact** precision — no rounding surprises. `DECIMAL` is a synonym for `NUMERIC` in PostgreSQL.

**Why It Exists** — Money and other exact values must never suffer floating-point rounding errors.

**Syntax**
```sql
column_name NUMERIC(precision, scale)
```

**Parameters**

| Name | Purpose | Default | Example |
|---|---|---|---|
| `precision` | Total number of digits (before + after decimal) | — | `10` |
| `scale` | Number of digits after the decimal point | 0 | `2` |

**Example**
```sql
CREATE TABLE products (
    price NUMERIC(10,2)
);
INSERT INTO products VALUES (65000.50);
```
**Output**
```
price
65000.50
```

**How precision/scale limit values**

```
NUMERIC(5,2)
digits before decimal = 5 - 2 = 3
max value = 999.99   ✅
1000.00 → 6 total digits → ❌ too large
```

⚠️ **Notes & Caveats**
- `NUMERIC` and `DECIMAL` behave identically in PostgreSQL — pick one style and stay consistent (PostgreSQL docs favor `NUMERIC`).

❌ **Common Mistakes**
```sql
-- ❌ Overflow: 1000.00 needs 6 total digits, column allows only 5
price NUMERIC(5,2); INSERT ... VALUES (1000.00);
```
```sql
-- ✅ Give enough precision for realistic values
price NUMERIC(10,2);
```

💡 **How to Choose** — Always use `NUMERIC` for money, prices, or any value where exactness matters legally/financially. Never use `FLOAT`/`REAL` for currency.

---

## 4. Approximate Decimals: `REAL`, `FLOAT`, `DOUBLE PRECISION`

**Definition** — Store decimal numbers using floating-point representation, which is fast but only **approximately** precise.

**Why It Exists** — Scientific/sensor data rarely needs exact decimal precision, and floating-point math is faster and more compact than `NUMERIC`.

**Syntax**
```sql
column_name REAL               -- ~6 decimal digits precision
column_name DOUBLE PRECISION   -- ~15 decimal digits precision
column_name FLOAT              -- maps to DOUBLE PRECISION by default
```

**Example**
```sql
CREATE TABLE measurements (
    temperature FLOAT
);
INSERT INTO measurements VALUES (36.7);
-- Some calculations (e.g. 0.1 + 0.2) may show tiny rounding artifacts
-- like 0.30000000000000004 due to binary floating-point representation.
```

⚠️ **Notes & Caveats**
- Never use these for money — rounding drift accumulates across many operations.

💡 **How to Choose** — Use `FLOAT`/`REAL`/`DOUBLE PRECISION` for scientific measurements, sensor values, or coordinates where tiny imprecision is acceptable. Use `NUMERIC` whenever exactness matters (money, quantities that must reconcile exactly).

---

## 5. `FLOAT` vs `NUMERIC` (Comparison)

| | `FLOAT` / `REAL` / `DOUBLE PRECISION` | `NUMERIC` / `DECIMAL` |
|---|---|---|
| Precision | Approximate | Exact |
| Best for | Scientific measurements, sensors | Money, prices, exact quantities |
| Rounding risk | Yes | No |
| Example | Temperature = 36.712345 | Price = ₹199.99 |

💡 **Memory trick:** FLOAT ≈ approximately 3.14 · NUMERIC = exactly 3.14

---

## 6. Text: `CHAR`, `VARCHAR`, `TEXT`

**Definition**
- **`CHAR(n)`** — fixed-length text; shorter values are space-padded to length `n`.
- **`VARCHAR(n)`** — variable-length text, up to a maximum of `n` characters.
- **`TEXT`** — variable-length text with **no** declared length limit.

**Syntax**
```sql
column_name CHAR(n)
column_name VARCHAR(n)
column_name TEXT
```

**Example**
```sql
CREATE TABLE employees (
    country_code CHAR(2),          -- always exactly 2 chars: 'IN', 'US'
    name         VARCHAR(100),     -- up to 100 chars
    bio          TEXT              -- unlimited length
);
INSERT INTO employees VALUES ('IN', 'Aman', 'Loves SQL and backend systems...');
```

⚠️ **Notes & Caveats**
- `VARCHAR(100)` means *maximum* 100 characters, not *exactly* 100 — `'Aman'` (4 chars) is perfectly valid.
- `CHAR(10)` storing `'Aman'` is conceptually padded to `'Aman      '` (trailing spaces).
- In PostgreSQL, `TEXT` and `VARCHAR` perform almost identically — unlike some other databases, there's no meaningful speed penalty for `TEXT`.

💡 **How to Choose**
- `CHAR(n)` → only for genuinely fixed-width codes (`'IN'`, `'US'`, grade letters).
- `VARCHAR(n)` → when you want to *enforce* a maximum length rule (e.g., `username VARCHAR(30)`).
- `TEXT` → general-purpose text with no length rule (descriptions, comments, articles).
- When in doubt in PostgreSQL specifically, `TEXT` is a perfectly good default — you're not sacrificing performance.

---

## 7. `CHAR` vs `VARCHAR` vs `TEXT` (Comparison)

| | `CHAR(n)` | `VARCHAR(n)` | `TEXT` |
|---|---|---|---|
| Length | Fixed (padded) | Variable, capped at `n` | Variable, uncapped |
| Storage of `'Aman'` in a `(10)` column | `'Aman      '` | `'Aman'` | `'Aman'` |
| Good for | Fixed codes (`'IN'`, grade `'A'`) | Names, emails, cities with a length rule | Descriptions, articles, long content |

---

## 8. `BOOLEAN`

**Definition** — Stores one of three possible values: `TRUE`, `FALSE`, or `NULL` (unless `NOT NULL` is added).

**Syntax**
```sql
column_name BOOLEAN
column_name BOOLEAN DEFAULT TRUE
```

**Example**
```sql
CREATE TABLE products (
    product_name VARCHAR(100),
    is_available BOOLEAN DEFAULT TRUE
);
INSERT INTO products (product_name) VALUES ('Laptop');
```
**Output**
```
product_name | is_available
Laptop       | true
```

⚠️ **Notes & Caveats** — `is_active BOOLEAN` (without `NOT NULL`) can still be `NULL`, meaning "unknown," which is different from `FALSE`.

💡 **How to Choose** — Use for any true/false flag: `is_active`, `is_available`, `is_verified`, `has_completed`.

---

## 9. Date & Time: `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`

**Definition**
- **`DATE`** — a calendar date only (`YYYY-MM-DD`).
- **`TIME`** — a time of day only (`HH:MM:SS`), no date.
- **`TIMESTAMP`** — date **and** time together, with no timezone awareness.
- **`TIMESTAMPTZ`** (`TIMESTAMP WITH TIME ZONE`) — an exact moment in time, displayed according to the viewer's timezone.

**Syntax**
```sql
column_name DATE
column_name TIME
column_name TIMESTAMP
column_name TIMESTAMPTZ
column_name TIMESTAMP DEFAULT CURRENT_TIMESTAMP
```

**Example**
```sql
CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    name       VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO users (name) VALUES ('Tushar');
```
**Output**
```
user_id | name   | created_at
1       | Tushar | 2026-07-08 10:30:00+05:30
```

⚠️ **Notes & Caveats**
- `DATE` alone can't tell you a time of day; `TIME` alone can't tell you which day.
- The same instant can *display* differently across timezones only with `TIMESTAMPTZ` — plain `TIMESTAMP` has no timezone concept at all.

💡 **How to Choose**
- `DATE` → birthdays, admission dates, expiry dates (no time component needed).
- `TIME` → opening/closing hours, class start times.
- `TIMESTAMP` → simple internal event logs where the app runs in a single timezone.
- `TIMESTAMPTZ` → almost always preferred for application events (`created_at`, `updated_at`, `login_time`) in any system that might span timezones.

---

## 10. `DATE` vs `TIME` vs `TIMESTAMP` vs `TIMESTAMPTZ` (Comparison)

| Type | Stores | Example |
|---|---|---|
| `DATE` | Date only | `2026-07-05` |
| `TIME` | Time only | `10:30:00` |
| `TIMESTAMP` | Date + time, no timezone | `2026-07-05 10:30:00` |
| `TIMESTAMPTZ` | Date + time, timezone-aware | `2026-07-05 10:30:00+05:30` |

---

## 11. Choosing the Right Data Type — Decision Map

```
WHOLE NUMBER?              → INTEGER (BIGINT if huge, SMALLINT if tiny + high volume)
EXACT DECIMAL / MONEY?     → NUMERIC(precision, scale)
APPROXIMATE DECIMAL?       → FLOAT / DOUBLE PRECISION
SHORT FIXED-WIDTH TEXT?    → CHAR(n)
TEXT WITH A LENGTH LIMIT?  → VARCHAR(n)
GENERAL / LONG TEXT?       → TEXT
TRUE OR FALSE?             → BOOLEAN
DATE ONLY?                 → DATE
TIME ONLY?                 → TIME
DATE + TIME, SAME ZONE?    → TIMESTAMP
APP EVENT, ANY TIMEZONE?   → TIMESTAMPTZ
```

---

## 12. Interview Q&A

**Q: What's the practical difference between `INTEGER` and `BIGINT`?**
A: Both store whole numbers; `BIGINT` supports a far larger range (up to ~9 quintillion vs ~2.1 billion) at the cost of double the storage (8 bytes vs 4). Use `BIGINT` only when you genuinely expect values beyond `INTEGER`'s range.

**Q: `VARCHAR` vs `TEXT` — when would you pick one over the other?**
A: `VARCHAR(n)` enforces a maximum length; `TEXT` has no limit. In PostgreSQL they perform almost identically, so the choice is really about whether you want a length *rule*, not about performance.

**Q: Why should money never be stored as `FLOAT`?**
A: `FLOAT` uses approximate binary floating-point representation, which can introduce tiny rounding errors that accumulate over many calculations — unacceptable for financial values. `NUMERIC` stores exact decimal values instead.

**Q: What do the two numbers in `NUMERIC(10,2)` mean?**
A: `10` is the precision (total digits allowed), `2` is the scale (digits after the decimal point) — so up to 8 digits before the decimal and 2 after.

**Q: `CHAR(n)` vs `VARCHAR(n)`?**
A: `CHAR(n)` is fixed-length and pads shorter values with trailing spaces; `VARCHAR(n)` is variable-length up to a maximum of `n` characters and doesn't pad.

**Q: Why prefer `TIMESTAMPTZ` over `TIMESTAMP` for application timestamps?**
A: `TIMESTAMPTZ` represents an exact, unambiguous moment in time and displays it correctly across different viewer timezones; plain `TIMESTAMP` has no timezone concept, which causes bugs in systems used across regions.

**Q: Is `INT` different from `INTEGER`?**
A: No — `INT` is simply a shorter alias for `INTEGER`; they're the exact same type.

---

## 13. Quick Revision Sheet

| Need | Type |
|---|---|
| Small whole number | `SMALLINT` |
| Normal whole number | `INTEGER` / `INT` |
| Huge whole number | `BIGINT` |
| Exact decimal (money) | `NUMERIC(p,s)` |
| Approximate decimal | `FLOAT` / `REAL` / `DOUBLE PRECISION` |
| Fixed-width text | `CHAR(n)` |
| Limited-length text | `VARCHAR(n)` |
| Unlimited text | `TEXT` |
| True/false | `BOOLEAN` |
| Date only | `DATE` |
| Time only | `TIME` |
| Date+time, no zone | `TIMESTAMP` |
| Date+time, zone-aware | `TIMESTAMPTZ` |

---

## 14. Cheat Sheet

```sql
-- ── WHOLE NUMBERS ────────────────────────
age            SMALLINT                      -- 2 bytes, -32,768..32,767
student_id     INTEGER                       -- 4 bytes, ~-2.1B..2.1B  (INT = same)
transaction_id BIGINT                        -- 8 bytes, ~-9 quintillion..9 quintillion

-- ── DECIMALS ─────────────────────────────
price          NUMERIC(10,2)                 -- exact:  precision, scale
rating         FLOAT                         -- approximate
measurement    DOUBLE PRECISION              -- approximate, more precision than REAL

-- ── TEXT ─────────────────────────────────
grade          CHAR(1)                       -- fixed length, space-padded
name           VARCHAR(100)                  -- variable, max 100 chars
description    TEXT                          -- variable, unlimited

-- ── BOOLEAN ──────────────────────────────
is_active      BOOLEAN DEFAULT TRUE

-- ── DATE & TIME ──────────────────────────
admission_date DATE                          -- 2026-07-05
class_time     TIME                          -- 10:30:00
created_at     TIMESTAMP DEFAULT CURRENT_TIMESTAMP        -- no timezone
created_at     TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP       -- timezone-aware
```

---

## 15. Preview of Part 5

| Topic | What You'll Learn |
|---|---|
| Auto-generated IDs | `SERIAL`, `SMALLSERIAL`, `BIGSERIAL`, `GENERATED AS IDENTITY` |
| `UUID` | Globally unique identifiers, `gen_random_uuid()` |
| `JSON` / `JSONB` | Storing flexible structured data, `->` and `->>` |
| `ARRAY` | Storing multiple values in one column |
| `BYTEA` | Binary data |
