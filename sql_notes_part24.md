# SQL & PostgreSQL Complete Notes — Part 24: Mathematical Functions

## 📑 Table of Contents
1. Why Different Rounding Functions Exist
2. `ROUND()`
3. `FLOOR()`
4. `CEIL()` / `CEILING()`
5. `ROUND` vs `FLOOR` vs `CEIL` — Including Negative Numbers
6. `POWER()`
7. `SQRT()`
8. `MOD()`
9. `RANDOM()`
10. Combining Functions — Random Integer in a Range, Random Row
11. Interview Q&A
12. Quick Revision Sheet
13. Cheat Sheet
14. Preview of Part 25

**📋 Series Coverage (Part 24):** `ROUND()` (with/without decimal places), `FLOOR()`, `CEIL()`/`CEILING()`, rounding behavior with negative numbers, `POWER()`, `SQRT()`, `MOD()`, `RANDOM()`, random-integer-in-a-range pattern, random-row-selection pattern

---

## 1. Why Different Rounding Functions Exist

Different real-world situations need different rounding behavior:
- "Round to the nearest value" (general reporting) → `ROUND()`
- "Always round down, no matter what" (max whole items that fit in a box) → `FLOOR()`
- "Always round up, no matter what" (shipping must cover partial weight) → `CEIL()`

---

## 2. `ROUND()`

**Definition** — Rounds a number to the nearest integer, or to a specified number of decimal places.

**Syntax**
```sql
ROUND(number)
ROUND(number, decimal_places)
```

**Parameters**

| Name | Purpose | Default |
|---|---|---|
| `number` | The value to round | — |
| `decimal_places` | How many digits to keep after the decimal | `0` (whole integer) |

**Example**
```sql
SELECT ROUND(12.3);   -- 12
SELECT ROUND(12.8);   -- 13
```

**Mental rule:** digits 0–4 round down, 5–9 round up.

**With decimal places**
```sql
SELECT ROUND(12.34567, 2);   -- 12.35
SELECT ROUND(18.12345, 3);   -- 18.123
```

**Real-world use — currency formatting**
```sql
SELECT product, ROUND(price, 2) FROM products;
```
**Output**
```
product | price
Phone   | 999.99
Mouse   | 550.55
```

---

## 3. `FLOOR()`

**Definition** — Always rounds **down** to the nearest integer, regardless of the decimal value.

**Syntax**
```sql
FLOOR(number)
```

**Example**
```sql
SELECT FLOOR(19.99);   -- 19
SELECT FLOOR(19.01);   -- 19
```

⚠️ **Notes & Caveats — negative numbers trip people up:**
```sql
SELECT FLOOR(-12.3);   -- -13, NOT -12
```
"Down" means toward **negative infinity**, not toward zero:
```
-13 ---- -12.3 ---- -12 ---- -11
           ↑
    next LOWER integer is -13
```

💡 **Memory trick** — `FLOOR` = go down, no matter what.

---

## 4. `CEIL()` / `CEILING()`

**Definition** — Always rounds **up** to the nearest integer. PostgreSQL accepts both spellings, identical result.

**Syntax**
```sql
CEIL(number)
CEILING(number)
```

**Example**
```sql
SELECT CEIL(12.1);   -- 13
SELECT CEIL(12.9);   -- 13
```

⚠️ **Notes & Caveats — negative numbers:**
```sql
SELECT CEIL(-12.7);   -- -12, NOT -13
```
```
-13 ---- -12.7 ---- -12 ---- -11
                      ↑
             next HIGHER integer is -12
```

---

## 5. `ROUND` vs `FLOOR` vs `CEIL` — Including Negative Numbers

| Input | `ROUND()` | `FLOOR()` | `CEIL()` |
|---|---|---|---|
| `12.6` | 13 | 12 | 13 |
| `12.2` | 12 | 12 | 13 |
| `-12.2` | -12 | -13 | -12 |

**Real-world use — shipping charges (always round up)**
```sql
SELECT CEIL(weight) FROM packages;   -- 3.2 kg → charged for 4 kg
```

**Real-world use — max whole items that fit (always round down)**
```sql
SELECT FLOOR(capacity) FROM boxes;   -- 9.8 → only 9 complete items
```

**Real-world use — combining with arithmetic (GST calculation)**
```sql
SELECT ROUND(price * 0.18, 2) FROM products;
```

❌ **Common Mistakes**
- Assuming `ROUND()` always rounds up — it rounds to the *nearest* value.
- Assuming `FLOOR(-12.3)` is `-12` — it's `-13`, because "down" means toward negative infinity.
- Assuming `CEIL()` returns the nearest integer — it always moves upward, regardless of how close the value is.

---

## 6. `POWER()`

**Definition** — Raises a number to a specified exponent.

**Syntax**
```sql
POWER(base, exponent)
```

**Parameters**

| Name | Purpose | Example |
|---|---|---|
| `base` | The number being raised | `2` |
| `exponent` | The power to raise it to | `3` |

**Example**
```sql
SELECT POWER(2, 3);    -- 8    (2³)
SELECT POWER(5, 2);    -- 25   (5²)
SELECT POWER(10, 4);   -- 10000
```

**Negative exponent**
```sql
SELECT POWER(2, -2);   -- 0.25   (1 / 2²)
```

**Decimal exponent (square root via exponent 0.5)**
```sql
SELECT POWER(9, 0.5);  -- 3    (√9)
```

**Real-world use — area of a circle**
```sql
SELECT radius, 3.14159 * POWER(radius, 2) AS area FROM circles;
```

❌ **Common Mistakes** — Thinking `POWER(2, 3)` means `2 × 3` — it means `2³ = 8`, not `6`.

---

## 7. `SQRT()`

**Definition** — Returns the square root of a number.

**Syntax**
```sql
SQRT(number)
```

**Example**
```sql
SELECT SQRT(25);   -- 5
SELECT SQRT(81);   -- 9
SELECT SQRT(2);     -- 1.414213... (irrational, returns a decimal)
```

💡 **Relationship to `POWER()`** — `POWER(5, 2) = 25` and `SQRT(25) = 5` are inverse operations of each other.

**Real-world use — distance formula**
```sql
SELECT SQRT(x*x + y*y) FROM points;
```

---

## 8. `MOD()`

**Definition** — Returns the remainder after division.

**Syntax**
```sql
MOD(number, divisor)
```

**Example**
```sql
SELECT MOD(10, 3);   -- 1  (10 ÷ 3 = 3 remainder 1)
SELECT MOD(20, 5);   -- 0
SELECT MOD(17, 4);   -- 1
```

💡 **Analogy** — 17 chocolates packed into boxes of 4: `4 × 4 = 16` used, `1` chocolate left over — that leftover is exactly `MOD(17, 4)`.

**Real-world use — even/odd check**
```sql
SELECT * FROM numbers WHERE MOD(value, 2) = 0;   -- even numbers
SELECT * FROM numbers WHERE MOD(value, 2) = 1;   -- odd numbers
```

❌ **Common Mistakes** — Thinking `MOD()` returns the quotient — it returns only the **remainder**. `MOD(17, 4)` is `1`, not `4`.

---

## 9. `RANDOM()`

**Definition** — Returns a random decimal number between `0` (inclusive) and `1` (exclusive). Produces a new value on every call.

**Syntax**
```sql
SELECT RANDOM();
```

**Example**
```sql
SELECT RANDOM();   -- e.g. 0.582731
SELECT RANDOM();   -- e.g. 0.103442 (different every time)
```

**Random integer between 1 and 10**
```sql
SELECT FLOOR(RANDOM() * 10) + 1;
```
**Trace through**
```
RANDOM()          → 0.63
0.63 × 10           → 6.3
FLOOR(6.3)           → 6
6 + 1                 → 7
```

**Random row from a table**
```sql
SELECT * FROM employees ORDER BY RANDOM() LIMIT 1;
```
*(Every row is assigned a random value, the result is sorted by that value, and the first row is taken — a different employee each run.)*

⚠️ **Notes & Caveats** — `RANDOM()` returns a **decimal**, not a whole number — you always need `FLOOR()` (and typically a `+ offset`) to produce a usable random integer.

---

## 10. Combining Functions — Random Integer in a Range, Random Row

**Random integer between 50 and 100 (inclusive)**
```sql
SELECT FLOOR(RANDOM() * 51) + 50;
```
*(51 possible integers from 50 through 100 inclusive: `100 - 50 + 1 = 51`.)*

**Trace through**
```
RANDOM()             → 0.40
0.40 × 51               → 20.4
FLOOR(20.4)               → 20
20 + 50                     → 70
```

**General formula — random integer between `min` and `max` inclusive**
```sql
FLOOR(RANDOM() * (max - min + 1)) + min
```

---

## 11. Interview Q&A

**Q: Does `ROUND()` always round up?**
A: No — it rounds to the *nearest* value (digits 0–4 down, 5–9 up); only `CEIL()` always rounds up regardless of the digit.

**Q: What does `FLOOR(-12.3)` return, and why does it surprise people?**
A: `-13`, not `-12` — "down" means toward negative infinity on the number line, not toward zero.

**Q: What's the difference between `POWER()` and `SQRT()`?**
A: `POWER(base, exponent)` raises a number to a power; `SQRT(x)` is the inverse operation, finding what number squared gives `x` (equivalent to `POWER(x, 0.5)`).

**Q: What does `MOD()` return — the quotient or the remainder?**
A: The remainder — e.g., `MOD(17, 4)` is `1`, not the quotient `4`.

**Q: What range of values does `RANDOM()` return?**
A: A decimal between 0 (inclusive) and 1 (exclusive) — never a whole number on its own.

**Q: How would you generate a random integer between 1 and 100?**
A: `FLOOR(RANDOM() * 100) + 1`.

**Q: How would you select one random row from a table?**
A: `SELECT * FROM table ORDER BY RANDOM() LIMIT 1`.

**Q: Compare `ROUND(12.5)`, `FLOOR(12.5)`, and `CEIL(12.5)`.**
A: `ROUND(12.5) = 13` (rounds to nearest, .5 rounds up), `FLOOR(12.5) = 12` (always down), `CEIL(12.5) = 13` (always up).

---

## 12. Quick Revision Sheet

| Need | Function |
|---|---|
| Nearest integer / N decimals | `ROUND(x)` / `ROUND(x, n)` |
| Always round down | `FLOOR(x)` |
| Always round up | `CEIL(x)` / `CEILING(x)` |
| Exponent | `POWER(base, exp)` |
| Square root | `SQRT(x)` |
| Remainder | `MOD(x, y)` |
| Random decimal [0, 1) | `RANDOM()` |
| Random integer 1–N | `FLOOR(RANDOM() * N) + 1` |
| Random integer min–max | `FLOOR(RANDOM() * (max - min + 1)) + min` |
| Random row | `ORDER BY RANDOM() LIMIT 1` |

---

## 13. Cheat Sheet

```sql
-- ── ROUNDING ──────────────────────────────
SELECT ROUND(12.6);        -- 13
SELECT ROUND(12.345, 2);   -- 12.35
SELECT FLOOR(19.99);        -- 19
SELECT FLOOR(-12.3);         -- -13 (toward negative infinity)
SELECT CEIL(12.1);            -- 13
SELECT CEIL(-12.7);            -- -12 (toward positive infinity)
SELECT CEILING(12.1);           -- 13 (same as CEIL)

-- ── POWER / SQRT ──────────────────────────
SELECT POWER(2, 3);    -- 8
SELECT POWER(2, -2);   -- 0.25
SELECT POWER(9, 0.5);  -- 3 (square root via exponent)
SELECT SQRT(25);         -- 5

-- ── MOD ───────────────────────────────────
SELECT MOD(10, 3);   -- 1
SELECT MOD(17, 4);   -- 1
SELECT * FROM numbers WHERE MOD(value, 2) = 0;   -- even
SELECT * FROM numbers WHERE MOD(value, 2) = 1;   -- odd

-- ── RANDOM ────────────────────────────────
SELECT RANDOM();                                    -- decimal [0, 1)
SELECT FLOOR(RANDOM() * 10) + 1;                      -- random int 1-10
SELECT FLOOR(RANDOM() * 51) + 50;                      -- random int 50-100
SELECT * FROM employees ORDER BY RANDOM() LIMIT 1;      -- random row

-- ── COMBINED ──────────────────────────────
SELECT ROUND(price * 0.18, 2) AS gst FROM products;
```

---

## 14. Preview of Part 25

| Topic | What You'll Learn |
|---|---|
| `CASE` | If-else style decision logic in SQL |
| `COALESCE()` / `NULLIF()` | Revisited with new patterns (division-by-zero, combined with `CASE`) |
| `GREATEST()` / `LEAST()` | Largest/smallest value across multiple columns |
