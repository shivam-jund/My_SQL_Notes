# SQL & PostgreSQL Complete Notes — Part 23: Date & Time Functions II — INTERVAL & DATE_TRUNC

## 📑 Table of Contents
1. `INTERVAL`
2. `INTERVAL` vs Plain Integer Arithmetic
3. `DATE_TRUNC()`
4. Why `DATE_TRUNC()` Is Useful (Grouping by Day/Month/Year)
5. Interview Pattern: Orders in the Last 30 Days
6. Interview Pattern: Records From "This Month"
7. Interview Pattern: Sales by Month
8. Interview Pattern: First Day of the Month / Tomorrow's Date
9. Combining Functions: Years of Service
10. Phase 15 Function Summary
11. Interview Q&A
12. Quick Revision Sheet
13. Cheat Sheet
14. Preview of Part 24

**📋 Series Coverage (Part 23):** `INTERVAL` literals and arithmetic, `INTERVAL` vs plain integer date math, `DATE_TRUNC()`, grouping timestamps by day/month/year, "last 30 days," "this month," monthly sales report patterns, first-day-of-month, full Phase 15 recap

---

## 1. `INTERVAL`

**Definition** — Represents a **duration** of time (not a fixed point in time) — e.g., `5 days`, `2 months`, `1 year`, `3 hours`.

**Syntax**
```sql
INTERVAL 'value unit'
```

**Example**
```sql
SELECT CURRENT_DATE + INTERVAL '5 days';     -- 5 days from today
SELECT CURRENT_DATE + INTERVAL '2 months';    -- 2 months from today
SELECT CURRENT_DATE + INTERVAL '1 year';       -- 1 year from today
```

**Multiple units together**
```sql
SELECT CURRENT_DATE + INTERVAL '1 year 2 months 5 days';
```

❌ **Common Mistakes**
```sql
-- ❌ Missing the unit
CURRENT_DATE + INTERVAL '5';
```
```sql
-- ✅ Always include a unit
CURRENT_DATE + INTERVAL '5 days';
```

---

## 2. `INTERVAL` vs Plain Integer Arithmetic

| Operation | Best Choice |
|---|---|
| Add/subtract **days** | `CURRENT_DATE + 5` or `INTERVAL '5 days'` — either works |
| Add/subtract months, years, hours, minutes | **Must** use `INTERVAL` |

⚠️ **Notes & Caveats** — `CURRENT_DATE + 2` always means **2 days**, never 2 months. If you want 2 months, you must write `INTERVAL '2 months'` explicitly.

---

## 3. `DATE_TRUNC()`

**Definition** — "Truncates" (cuts off, doesn't round) a date/timestamp down to a specified precision, zeroing out everything smaller.

**Syntax**
```sql
DATE_TRUNC('part', timestamp)
```

**Example**
```sql
SELECT DATE_TRUNC('hour',  TIMESTAMP '2026-07-19 15:45:30');   -- 2026-07-19 15:00:00
SELECT DATE_TRUNC('day',   TIMESTAMP '2026-07-19 15:45:30');   -- 2026-07-19 00:00:00
SELECT DATE_TRUNC('month', TIMESTAMP '2026-07-19 15:45:30');   -- 2026-07-01 00:00:00
SELECT DATE_TRUNC('year',  TIMESTAMP '2026-07-19 15:45:30');   -- 2026-01-01 00:00:00
```

```
2026-07-19 15:45:30
   ↓ 'year'         → 2026-01-01 00:00:00
   ↓ 'month'         → 2026-07-01 00:00:00
   ↓ 'day'            → 2026-07-19 00:00:00
   ↓ 'hour'            → 2026-07-19 15:00:00
```

⚠️ **Notes & Caveats — it truncates, it does NOT round:**
```sql
SELECT DATE_TRUNC('hour', TIMESTAMP '2026-07-19 15:59:59');   -- 2026-07-19 15:00:00 (NOT 16:00:00!)
```

---

## 4. Why `DATE_TRUNC()` Is Useful (Grouping by Day/Month/Year)

**The problem** — Grouping by a full, precise timestamp (`order_time`) means every row with even a slightly different second becomes its own group. You almost always want to group by the **day** (or month/year) instead.

**Example**
```sql
SELECT DATE_TRUNC('day', order_time) AS order_day, COUNT(*)
FROM orders
GROUP BY DATE_TRUNC('day', order_time);
```
**Output**
```
order_day             | count
2026-07-19 00:00:00   | 2
2026-07-20 00:00:00   | 1
```

💡 **How to Choose** — `DATE_TRUNC()` is almost always cleaner than manually grouping by separate `EXTRACT(YEAR ...)` + `EXTRACT(MONTH ...)` columns — one grouping expression instead of two, and it stays sortable as a real timestamp.

---

## 5. Interview Pattern: Orders in the Last 30 Days

```sql
SELECT * FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days';
```

---

## 6. Interview Pattern: Records From "This Month"

```sql
SELECT * FROM employees
WHERE DATE_TRUNC('month', joining_date) = DATE_TRUNC('month', CURRENT_DATE);
```

💡 **Why this works** — Both sides get truncated down to the 1st of their respective month (e.g., both become `2026-07-01`), so any date **within** the current month matches, regardless of the specific day.

---

## 7. Interview Pattern: Sales by Month

```sql
SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS total_sales
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

---

## 8. Interview Pattern: First Day of the Month / Tomorrow's Date

**First day of the current month**
```sql
SELECT DATE_TRUNC('month', CURRENT_DATE);   -- e.g. 2026-07-01 00:00:00
```

**Tomorrow's date — two equally valid ways**
```sql
SELECT CURRENT_DATE + 1;
SELECT CURRENT_DATE + INTERVAL '1 day';
```

---

## 9. Combining Functions: Years of Service

```sql
SELECT name, EXTRACT(YEAR FROM AGE(joining_date)) AS service_years
FROM employees;
```
**Trace through `joining_date = 2021-06-10`**
```
AGE(joining_date)              → 5 years 1 mon ...
EXTRACT(YEAR FROM ...)          → 5
```

---

## 10. Phase 15 Function Summary

| Function | Purpose |
|---|---|
| `CURRENT_DATE` | Today's date |
| `CURRENT_TIME` | Current time |
| `NOW()` / `CURRENT_TIMESTAMP` | Current date + time |
| `AGE()` | Human-readable difference between dates |
| `DATE_PART()` | Extract a date/time component (Postgres style) |
| `EXTRACT()` | Extract a date/time component (SQL-standard style) |
| `INTERVAL` | Represent a duration for arithmetic |
| `DATE_TRUNC()` | Round a timestamp **down** to a given precision |

---

## 11. Interview Q&A

**Q: Why would `CURRENT_DATE + 2` not work if you wanted "2 months from today"?**
A: Plain integer arithmetic on a date is always interpreted as days — you need `CURRENT_DATE + INTERVAL '2 months'` for anything other than day-level increments.

**Q: Does `DATE_TRUNC('hour', ...)` round to the nearest hour or always round down?**
A: Always down (truncates) — `15:59:59` truncated to the hour becomes `15:00:00`, not `16:00:00`.

**Q: How would you group and sum orders by month?**
A: `GROUP BY DATE_TRUNC('month', order_date)` — cleaner than separately extracting year and month.

**Q: How do you find all employees who joined "this month," regardless of the exact day?**
A: Compare `DATE_TRUNC('month', joining_date) = DATE_TRUNC('month', CURRENT_DATE)` — both sides collapse to the 1st of their month, so any date in that month matches.

**Q: How would you get the first day of the current month?**
A: `DATE_TRUNC('month', CURRENT_DATE)`.

**Q: Give two ways to get "tomorrow's date."**
A: `CURRENT_DATE + 1` or `CURRENT_DATE + INTERVAL '1 day'` — both are valid for day-level arithmetic.

---

## 12. Quick Revision Sheet

| Need | Syntax |
|---|---|
| Duration literal | `INTERVAL 'n unit'` (e.g., `'5 days'`, `'2 months'`) |
| Add/subtract a duration | `date + INTERVAL '...'` |
| Round a timestamp down | `DATE_TRUNC('part', timestamp)` |
| Group by day/month/year | `GROUP BY DATE_TRUNC('part', column)` |
| Last N days | `WHERE date_col >= CURRENT_DATE - INTERVAL 'N days'` |
| "This month" match | `DATE_TRUNC('month', col) = DATE_TRUNC('month', CURRENT_DATE)` |
| First of current month | `DATE_TRUNC('month', CURRENT_DATE)` |

---

## 13. Cheat Sheet

```sql
-- ── INTERVAL ──────────────────────────────
SELECT CURRENT_DATE + INTERVAL '5 days';
SELECT CURRENT_DATE + INTERVAL '2 months';
SELECT CURRENT_DATE + INTERVAL '1 year 2 months 5 days';

-- ── DATE_TRUNC ────────────────────────────
SELECT DATE_TRUNC('hour',  TIMESTAMP '2026-07-19 15:45:30');
SELECT DATE_TRUNC('day',   TIMESTAMP '2026-07-19 15:45:30');
SELECT DATE_TRUNC('month', TIMESTAMP '2026-07-19 15:45:30');
SELECT DATE_TRUNC('year',  TIMESTAMP '2026-07-19 15:45:30');

-- ── GROUP BY DAY ──────────────────────────
SELECT DATE_TRUNC('day', order_time) AS order_day, COUNT(*)
FROM orders GROUP BY DATE_TRUNC('day', order_time);

-- ── LAST 30 DAYS ──────────────────────────
SELECT * FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL '30 days';

-- ── THIS MONTH ────────────────────────────
SELECT * FROM employees
WHERE DATE_TRUNC('month', joining_date) = DATE_TRUNC('month', CURRENT_DATE);

-- ── SALES BY MONTH ────────────────────────
SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS total_sales
FROM orders GROUP BY DATE_TRUNC('month', order_date) ORDER BY month;

-- ── FIRST DAY OF MONTH / TOMORROW ─────────
SELECT DATE_TRUNC('month', CURRENT_DATE);
SELECT CURRENT_DATE + 1;

-- ── YEARS OF SERVICE ──────────────────────
SELECT name, EXTRACT(YEAR FROM AGE(joining_date)) AS service_years FROM employees;
```

---

## 14. Preview of Part 24

| Topic | What You'll Learn |
|---|---|
| `ROUND()` | Rounding to the nearest integer or N decimal places |
| `FLOOR()` / `CEIL()` | Always rounding down / up |
| `POWER()` / `SQRT()` | Exponents and square roots |
| `MOD()` | Remainders |
| `RANDOM()` | Generating random values and rows |
