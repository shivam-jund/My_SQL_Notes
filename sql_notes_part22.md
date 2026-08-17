# SQL & PostgreSQL Complete Notes — Part 22: Date & Time Functions I — Current Values, AGE, EXTRACT

## 📑 Table of Contents
1. Three Kinds of "Now": Date, Time, Timestamp
2. `CURRENT_DATE`
3. `CURRENT_TIME`
4. `NOW()` and `CURRENT_TIMESTAMP`
5. Basic Date Arithmetic (`+` / `-` with Integers)
6. `AGE()`
7. `DATE_PART()`
8. `EXTRACT()`
9. `DATE_PART()` vs `EXTRACT()`
10. Combining `AGE()` + `EXTRACT()`
11. Interview Q&A
12. Quick Revision Sheet
13. Cheat Sheet
14. Preview of Part 23

**📋 Series Coverage (Part 22):** `CURRENT_DATE`, `CURRENT_TIME`, `NOW()`, `CURRENT_TIMESTAMP`, basic date arithmetic with integers, `AGE()`, `DATE_PART()`, `EXTRACT()`, combining `AGE()` with `EXTRACT()`

---

## 1. Three Kinds of "Now": Date, Time, Timestamp

Just as Part 4 distinguished the *data types* `DATE`/`TIME`/`TIMESTAMP`, PostgreSQL provides matching functions to get the **current** value of each:

| Need | Function |
|---|---|
| Today's date only | `CURRENT_DATE` |
| Current time only | `CURRENT_TIME` |
| Both, together | `NOW()` / `CURRENT_TIMESTAMP` |

---

## 2. `CURRENT_DATE`

**Definition** — Returns today's date (no time component).

**Syntax**
```sql
SELECT CURRENT_DATE;
```

⚠️ **Notes & Caveats** — **No parentheses.** `CURRENT_DATE()` is invalid — this is one of the most common beginner slips (it looks like a function call, but it's a reserved keyword-style expression).

**Example**
```sql
SELECT CURRENT_DATE;   -- 2026-07-19
```

**Real-world use**
```sql
SELECT * FROM employees WHERE join_date = CURRENT_DATE;   -- employees who joined today
```

---

## 3. `CURRENT_TIME`

**Definition** — Returns the current time of day (with time zone), no date.

**Syntax**
```sql
SELECT CURRENT_TIME;
```

**Example**
```sql
SELECT CURRENT_TIME;   -- 15:30:20+05:30
```

⚠️ **Notes & Caveats** — Same rule as `CURRENT_DATE`: no parentheses. The time zone offset shown depends on your session settings.

---

## 4. `NOW()` and `CURRENT_TIMESTAMP`

**Definition** — Both return the current **date and time together** (as a timestamp with time zone).

**Syntax**
```sql
SELECT NOW();
SELECT CURRENT_TIMESTAMP;
```

⚠️ **Notes & Caveats** — Unlike `CURRENT_DATE`/`CURRENT_TIME`, `NOW()` **does** use parentheses (it's a genuine function call).

**Example**
```sql
SELECT NOW();   -- 2026-07-19 15:30:20+05:30
```

**Return types**

| Function | Returns |
|---|---|
| `CURRENT_DATE` | `DATE` |
| `CURRENT_TIME` | `TIME WITH TIME ZONE` |
| `NOW()` / `CURRENT_TIMESTAMP` | `TIMESTAMP WITH TIME ZONE` |

💡 **How to Choose** — `NOW()` and `CURRENT_TIMESTAMP` behave almost identically in everyday PostgreSQL use — `NOW()` is simply shorter and more commonly written.

**Real-world use — auto-stamping a row**
```sql
INSERT INTO users (created_at) VALUES (NOW());
```
*(In practice, this is almost always done via `DEFAULT CURRENT_TIMESTAMP` on the column itself, from Part 4, rather than passed explicitly on every insert.)*

---

## 5. Basic Date Arithmetic (`+` / `-` with Integers)

**Definition** — Adding or subtracting a plain integer to/from a `DATE` shifts it by that many **days**.

**Example**
```sql
SELECT CURRENT_DATE + 5;   -- 5 days from today
SELECT CURRENT_DATE - 7;   -- 7 days ago
```

**Real-world use — "last 7 days"**
```sql
SELECT * FROM orders WHERE order_date >= CURRENT_DATE - 7;
```

⚠️ **Notes & Caveats** — Plain integer arithmetic only makes sense for **days**. For months, years, hours, or minutes, you need `INTERVAL` (fully covered in Part 23) — `CURRENT_DATE + 2` always means 2 days, never 2 months.

---

## 6. `AGE()`

**Definition** — Calculates the difference between two dates/timestamps and returns it as a **human-readable interval** (years, months, days) instead of a raw day count.

**Syntax**
```sql
AGE(date)              -- difference between date and TODAY
AGE(date1, date2)       -- difference between two specific dates
```

**Example — age from today**
```sql
SELECT name, AGE(dob) AS age FROM employees;
```
**Output**
```
name  | age
Aman  | 20 years 11 mons 6 days
```

**Example — difference between two specific dates**
```sql
SELECT AGE('2026-07-19', '2025-01-10');   -- 1 year 6 mons 9 days
```

⚠️ **Notes & Caveats — vs. simple subtraction:**
```sql
SELECT CURRENT_DATE - dob FROM employees;   -- 7664  (raw number of days — not human-friendly)
SELECT AGE(dob) FROM employees;              -- 20 years 11 mons 6 days  (readable)
```

💡 PostgreSQL abbreviates months as `mons` in `AGE()` output.

---

## 7. `DATE_PART()`

**Definition** — Extracts one specific component (year, month, day, hour, etc.) from a date/timestamp.

**Syntax**
```sql
DATE_PART('part', date)
```

**Example**
```sql
SELECT DATE_PART('year',  DATE '2026-07-19');   -- 2026
SELECT DATE_PART('month', DATE '2026-07-19');   -- 7
SELECT DATE_PART('day',   DATE '2026-07-19');   -- 19
```

**Common parts**

| Part | Example Result |
|---|---|
| `'year'` | `2026` |
| `'month'` | `7` |
| `'day'` | `19` |
| `'hour'` | `15` |
| `'minute'` | `30` |
| `'second'` | `45` |

**Real-world use**
```sql
SELECT * FROM employees WHERE DATE_PART('year', joining_date) = 2025;
```

❌ **Common Mistakes**
```sql
-- ❌ Missing quotes around the part name
DATE_PART(year, joining_date);
```
```sql
-- ✅ The part name must be a quoted string
DATE_PART('year', joining_date);
```

---

## 8. `EXTRACT()`

**Definition** — Also extracts one component from a date/timestamp — functionally almost identical to `DATE_PART()`, but with different (SQL-standard) syntax.

**Syntax**
```sql
EXTRACT(part FROM date)
```

**Example**
```sql
SELECT EXTRACT(YEAR  FROM DATE '2026-07-19');   -- 2026
SELECT EXTRACT(MONTH FROM DATE '2026-07-19');   -- 7
SELECT EXTRACT(DAY   FROM DATE '2026-07-19');   -- 19
SELECT EXTRACT(HOUR  FROM NOW());                -- e.g. 15
```

⚠️ **Notes & Caveats** — `EXTRACT()` uses `FROM`, no comma, and **no quotes** around the part keyword.

❌ **Common Mistakes**
```sql
-- ❌ Don't quote the part name in EXTRACT
EXTRACT('YEAR' FROM date);
```
```sql
-- ✅ Correct
EXTRACT(YEAR FROM date);
```

**Real-world use**
```sql
SELECT * FROM employees WHERE EXTRACT(MONTH FROM dob) = 8;   -- born in August
SELECT COUNT(*) FROM employees
WHERE EXTRACT(YEAR FROM joining_date) = EXTRACT(YEAR FROM CURRENT_DATE);  -- joined this year
```

---

## 9. `DATE_PART()` vs `EXTRACT()`

| | `DATE_PART('year', date)` | `EXTRACT(YEAR FROM date)` |
|---|---|---|
| Syntax style | Comma-separated, quoted part | `FROM`-based, unquoted part |
| Standard | PostgreSQL-specific | SQL standard |
| Result | Identical | Identical |

💡 **How to Choose** — Both are fully acceptable in PostgreSQL. `EXTRACT()` is preferred when writing SQL that might need to be portable to other databases; `DATE_PART()` is equally common in existing PostgreSQL codebases.

---

## 10. Combining `AGE()` + `EXTRACT()`

**The question:** *"Show each employee's age in completed years only, no months/days."*
```sql
SELECT name, EXTRACT(YEAR FROM AGE(dob)) AS age_years FROM employees;
```
**Trace through**
```
Step 1 — AGE(dob)                    → 20 years 11 mons 6 days
Step 2 — EXTRACT(YEAR FROM ...)       → 20
```

**Same pattern — years of service**
```sql
SELECT name, EXTRACT(YEAR FROM AGE(joining_date)) AS service_years FROM employees;
```

---

## 11. Interview Q&A

**Q: Why is `CURRENT_DATE()` invalid in PostgreSQL?**
A: `CURRENT_DATE` (and `CURRENT_TIME`) are not function calls — they're evaluated without parentheses. Writing `CURRENT_DATE()` will error.

**Q: What's the difference between `AGE(dob)` and `CURRENT_DATE - dob`?**
A: `AGE(dob)` returns a human-readable interval like `20 years 11 mons 6 days`. `CURRENT_DATE - dob` returns a raw integer number of days, which is accurate but not immediately readable.

**Q: How would you find employees born in August, regardless of year?**
A: `WHERE EXTRACT(MONTH FROM dob) = 8`.

**Q: `DATE_PART()` vs `EXTRACT()` — functionally, what's the difference?**
A: None — they return identical results for the same date part; they only differ in syntax (`DATE_PART('year', col)` vs `EXTRACT(YEAR FROM col)`).

**Q: How do you get someone's age as a single whole number of years?**
A: `EXTRACT(YEAR FROM AGE(dob))` — compute the human-readable age interval first, then pull out just the year component.

**Q: Does `CURRENT_DATE + 2` add 2 days or 2 months?**
A: 2 days — plain integer arithmetic on a date is always in day units; use `INTERVAL '2 months'` for anything other than days.

---

## 12. Quick Revision Sheet

| Need | Syntax |
|---|---|
| Today's date | `CURRENT_DATE` (no parens) |
| Current time | `CURRENT_TIME` (no parens) |
| Current date + time | `NOW()` or `CURRENT_TIMESTAMP` |
| Add/subtract days | `CURRENT_DATE + n` / `- n` |
| Human-readable age/difference | `AGE(date)` or `AGE(date1, date2)` |
| Extract a part (Postgres style) | `DATE_PART('part', date)` |
| Extract a part (SQL-standard style) | `EXTRACT(PART FROM date)` |
| Age in whole years | `EXTRACT(YEAR FROM AGE(date))` |

---

## 13. Cheat Sheet

```sql
-- ── CURRENT VALUES ────────────────────────
SELECT CURRENT_DATE;         -- today's date, no parens
SELECT CURRENT_TIME;          -- current time, no parens
SELECT NOW();                  -- current date + time
SELECT CURRENT_TIMESTAMP;       -- same as NOW()

-- ── BASIC ARITHMETIC ──────────────────────
SELECT CURRENT_DATE + 5;
SELECT CURRENT_DATE - 7;
SELECT * FROM orders WHERE order_date >= CURRENT_DATE - 7;   -- last 7 days

-- ── AGE ────────────────────────────────────
SELECT AGE(dob) FROM employees;                 -- age from today
SELECT AGE('2026-07-19', '2025-01-10');           -- difference between two dates

-- ── DATE_PART / EXTRACT ───────────────────
SELECT DATE_PART('year', joining_date) FROM employees;
SELECT EXTRACT(YEAR FROM joining_date) FROM employees;
SELECT * FROM employees WHERE EXTRACT(MONTH FROM dob) = 8;
SELECT EXTRACT(YEAR FROM AGE(dob)) AS age_years FROM employees;
```

---

## 14. Preview of Part 23

| Topic | What You'll Learn |
|---|---|
| `INTERVAL` | Representing and adding/subtracting durations (months, years, hours) |
| `DATE_TRUNC()` | Rounding a timestamp down to a unit (day, month, year) |
| Date arithmetic patterns | "Last 30 days," "this month," monthly sales reports |
