# SQL & PostgreSQL Complete Notes — Part 18: Window Functions — Distribution, Navigation, Value & Frames

## 📑 Table of Contents
1. `NTILE(n)` — Splitting Rows Into Buckets
2. How `NTILE` Handles Uneven Splits
3. `NTILE` vs `RANK` / `DENSE_RANK`
4. `LAG()` — Looking at the Previous Row
5. `LEAD()` — Looking at the Next Row
6. Offset & Default Value Parameters
7. `PARTITION BY` with `LAG()` / `LEAD()`
8. The Window Frame — What Exactly Is "The Window"?
9. `ROWS BETWEEN` Syntax
10. `FIRST_VALUE()`
11. `LAST_VALUE()` — The Classic Interview Trap
12. `NTH_VALUE()`
13. Moving Averages (Window Frame in Practice)
14. Interview Q&A
15. Quick Revision Sheet
16. Cheat Sheet
17. Preview of Part 19

**📋 Series Coverage (Part 18):** `NTILE(n)` (bucketing/quartiles), `LAG()`, `LEAD()` (offset + default parameters), window frames, `ROWS BETWEEN UNBOUNDED PRECEDING/FOLLOWING/CURRENT ROW/N PRECEDING/N FOLLOWING`, `FIRST_VALUE()`, `LAST_VALUE()`, `NTH_VALUE()`, moving averages

---

## 1. `NTILE(n)` — Splitting Rows Into Buckets

**Definition** — Divides the ordered rows into `n` roughly equal groups ("buckets") and returns each row's bucket number. Unlike the ranking functions, `NTILE` isn't concerned with *position* — it's concerned with *which group*.

**Why It Exists** — Common analytics need: "split customers/employees/scores into quartiles (4 groups), deciles (10 groups), or any N even segments."

**Syntax**
```sql
NTILE(n) OVER (PARTITION BY col ORDER BY col)
```

**Example — 8 employees into 4 buckets**
```sql
SELECT name, salary, NTILE(4) OVER (ORDER BY salary DESC) AS bucket
FROM employees;
```
**Output** *(8 rows ÷ 4 buckets = 2 rows per bucket)*
```
name   | salary | bucket
Aman   | 90000  | 1
Priya  | 85000  | 1
Arjun  | 80000  | 2
Ravi   | 75000  | 2
Simran | 70000  | 3
Karan  | 65000  | 3
Rohit  | 60000  | 4
Neha   | 55000  | 4
```

---

## 2. How `NTILE` Handles Uneven Splits

⚠️ **Notes & Caveats** — When rows don't divide evenly, PostgreSQL gives the **extra rows to the earlier buckets**.

**Example — 10 rows into 4 buckets** (`10 ÷ 4 = 2` remainder `2`)
```
Bucket 1: 3 rows   ← got an extra row
Bucket 2: 3 rows   ← got an extra row
Bucket 3: 2 rows
Bucket 4: 2 rows
```

**Example — 11 rows into 4 buckets** (`11 ÷ 4 = 2` remainder `3`)
```
Bucket 1: 3 rows
Bucket 2: 3 rows
Bucket 3: 3 rows
Bucket 4: 2 rows
```

💡 **Memory trick** — Extra rows flow into Bucket 1, then Bucket 2, and so on, until they run out.

---

## 3. `NTILE` vs `RANK` / `DENSE_RANK`

| | `RANK()` / `DENSE_RANK()` | `NTILE(n)` |
|---|---|---|
| Answers | "What **position**?" | "Which **group**?" |
| Ties handled | Kept together (same rank) | ⚠️ Can split ties across adjacent buckets — `NTILE` cares about equal-sized groups, not keeping identical values together |
| Typical use | Leaderboards, competition ranking | Quartiles, deciles, percentile-style segmentation |

❌ **Common Mistakes**
- Thinking `NTILE` returns ranks — it returns **bucket/group numbers**, which is a different concept.
- Assuming every bucket is exactly the same size — only true when rows divide evenly (Section 2).
- Assuming duplicate values always land in the same bucket — `NTILE` does **not** guarantee this, unlike `RANK`/`DENSE_RANK`.

💡 **How to Choose** — "Top 25% of salaries" → `NTILE(4)`, bucket 1. "Top 10%" → `NTILE(10)`, bucket 1. "Which decile is this customer in?" → `NTILE(10)`.

---

## 4. `LAG()` — Looking at the Previous Row

**Definition** — Returns a value from a **previous** row in the ordered partition, without needing a self join.

**Why It Exists** — Comparing "today vs. yesterday," "this employee vs. the previous one in sorted order" used to require an awkward self join — `LAG()` does it directly.

**Syntax**
```sql
LAG(column, offset, default) OVER (PARTITION BY col ORDER BY col)
```

**Parameters**

| Name | Purpose | Default |
|---|---|---|
| `column` | Which column's value to pull | — |
| `offset` | How many rows back | `1` |
| `default` | Value to use when there's no such previous row | `NULL` |

**Example**
```sql
SELECT name, salary, LAG(salary) OVER (ORDER BY emp_id) AS previous_salary
FROM employees;
```
**Output**
```
name  | salary | previous_salary
Aman  | 50000  | NULL     ← no row before the first one
Priya | 60000  | 50000
Arjun | 65000  | 60000
```

💡 **Analogy** — a car's rear-view mirror: `LAG()` looks **behind**.

---

## 5. `LEAD()` — Looking at the Next Row

**Definition** — The mirror image of `LAG()` — returns a value from a **following** row.

**Syntax**
```sql
LEAD(column, offset, default) OVER (PARTITION BY col ORDER BY col)
```

**Example**
```sql
SELECT name, salary, LEAD(salary) OVER (ORDER BY emp_id) AS next_salary
FROM employees;
```
**Output**
```
name  | salary | next_salary
Aman  | 50000  | 60000
Priya | 60000  | 65000
Arjun | 65000  | NULL   ← no row after the last one
```

💡 **Analogy** — the windshield: `LEAD()` looks **ahead**.

---

## 6. Offset & Default Value Parameters

**Example — looking 2 rows back**
```sql
SELECT salary, LAG(salary, 2) OVER (ORDER BY emp_id) AS lag_2
FROM employees;
```

**Example — custom default instead of `NULL`**
```sql
SELECT salary, LAG(salary, 1, 0) OVER (ORDER BY emp_id) AS previous_or_zero
FROM employees;
```
**Output**
```
salary | previous_or_zero
50000  | 0        ← no previous row, so the DEFAULT (0) is used instead of NULL
60000  | 50000
```

**Real use — salary difference from the previous employee**
```sql
SELECT name, salary,
       salary - LAG(salary) OVER (ORDER BY emp_id) AS difference
FROM employees;
```

**Real use — detecting a trend**
```sql
SELECT sale_date, sales,
       CASE
           WHEN sales > LAG(sales) OVER (ORDER BY sale_date) THEN 'Increase'
           ELSE 'Decrease'
       END AS trend
FROM sales;
```

---

## 7. `PARTITION BY` with `LAG()` / `LEAD()`

**Example**
```sql
SELECT department, salary,
       LAG(salary) OVER (PARTITION BY department ORDER BY salary) AS prev_in_dept
FROM employees;
```

⚠️ **Notes & Caveats** — When SQL crosses into a new partition, `LAG()`/`LEAD()` **resets** — the first row of each new department has no "previous" row *within that department*, even if there were rows immediately before it in the unpartitioned table.

❌ **Common Mistakes**
- Assuming `LAG()` means "previous by ID" — it actually means "previous **according to the `ORDER BY` inside `OVER()`**," which may have nothing to do with ID order.
- Forgetting `ORDER BY` entirely — without it, "previous" and "next" are undefined, since there's no established sequence.

---

## 8. The Window Frame — What Exactly Is "The Window"?

**Definition** — The **window frame** is the precise subset of rows, relative to the current row, that a window function's calculation actually uses.

**Why It Matters** — This single concept explains the entire `LAST_VALUE()` "trap" coming up in Section 11.

⚠️ **Notes & Caveats — the default frame** — When `ORDER BY` is present inside `OVER()` but no frame is written explicitly, PostgreSQL's default frame is (conceptually) **"from the start of the partition up to and including the current row"** — not the whole partition.
```
Row 1: window sees [Row 1]
Row 2: window sees [Row 1, Row 2]
Row 3: window sees [Row 1, Row 2, Row 3]
Row 4: window sees [Row 1, Row 2, Row 3, Row 4]
```
This is exactly why `SUM(...) OVER (ORDER BY ...)` naturally produces a **running total** — the window keeps growing to include one more row each time.

---

## 9. `ROWS BETWEEN` Syntax

**Syntax**
```sql
FUNCTION(column) OVER (
    ORDER BY column
    ROWS BETWEEN frame_start AND frame_end
)
```

**Frame boundary keywords**

| Keyword | Meaning |
|---|---|
| `UNBOUNDED PRECEDING` | From the very first row of the partition |
| `CURRENT ROW` | The current row |
| `UNBOUNDED FOLLOWING` | Through the very last row of the partition |
| `N PRECEDING` | N rows before the current row |
| `N FOLLOWING` | N rows after the current row |

**Common frame patterns**
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW    -- default-like: start → current (running calc)
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING     -- current → end
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- ENTIRE partition, every row
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING              -- just the neighbors: prev, current, next
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW               -- last 3 rows including current (moving avg)
```

**Visual — `1 PRECEDING AND 1 FOLLOWING` for row 3 of 5**
```
1   2  [3]  4   5
    ↑———↑———↑
  visible window: rows 2, 3, 4 only
```

---

## 10. `FIRST_VALUE()`

**Definition** — Returns the value from the **first row** of the current window frame.

**Syntax**
```sql
FIRST_VALUE(column) OVER (PARTITION BY col ORDER BY col)
```

**Example — lowest salary per department (sorted ascending, so "first" = lowest)**
```sql
SELECT name, department, salary,
       FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary) AS lowest_in_dept
FROM employees;
```
**Output**
```
name  | department | salary | lowest_in_dept
Aman  | IT         | 50000  | 50000
Priya | IT         | 70000  | 50000
Arjun | IT         | 80000  | 50000
```

⚠️ **Notes & Caveats** — With the default frame (growing window), `FIRST_VALUE()` typically stays constant across all rows in a partition, because the *first* row is always inside the window once it's been reached.

---

## 11. `LAST_VALUE()` — The Classic Interview Trap

**Definition** — Returns the value from the **last row of the current window frame** — **not** necessarily the last row of the whole partition.

**The trap**
```sql
SELECT emp_id, salary, LAST_VALUE(salary) OVER (ORDER BY emp_id) AS last_val
FROM employees;
```
**Output (what beginners *expect* — the overall max):**
```
emp_id | salary | last_val
1      | 50000  | 80000   ❌ WRONG EXPECTATION
2      | 60000  | 80000   ❌
```
**Output (what you actually get):**
```
emp_id | salary | last_val
1      | 50000  | 50000   ← because with the DEFAULT frame, "last row of the window" = the current row itself
2      | 60000  | 60000
3      | 70000  | 70000
4      | 80000  | 80000
```

⚠️ **Notes & Caveats — why this happens** — Recall Section 8's default frame: "start of partition → **current row**." The **last** row *inside that frame* is, by definition, almost always the current row itself. So `LAST_VALUE()` with the default frame just echoes back the current row's own value.

✅ **The fix — widen the frame to cover the whole partition**
```sql
SELECT emp_id, salary,
       LAST_VALUE(salary) OVER (
           ORDER BY emp_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS true_last_value
FROM employees;
```
**Output**
```
emp_id | salary | true_last_value
1      | 50000  | 80000   ✅ now correct
2      | 60000  | 80000
3      | 70000  | 80000
4      | 80000  | 80000
```

💡 **Interview answer to remember:** *"`LAST_VALUE()` returns the last row of the current window **frame**, and the default frame ends at the current row — so without explicitly widening the frame to `UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, `LAST_VALUE()` just returns the current row's own value."*

💡 **Practical tip** — For "the maximum value per group," `FIRST_VALUE()` with `ORDER BY col DESC` is usually simpler than fighting with `LAST_VALUE()`'s frame requirements.

---

## 12. `NTH_VALUE()`

**Definition** — Returns the value from the **Nth row** of the current window frame.

**Syntax**
```sql
NTH_VALUE(column, n) OVER (
    ORDER BY column
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

**Example — 2nd salary in the whole ordered set**
```sql
SELECT salary,
       NTH_VALUE(salary, 2) OVER (
           ORDER BY emp_id
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS second_salary
FROM employees;
```

⚠️ **Notes & Caveats** — Just like `LAST_VALUE()`, `NTH_VALUE()` needs the **full-partition frame** — otherwise, rows before the Nth position in a growing default frame will return `NULL` (since the window hasn't grown large enough yet to contain a 2nd row).

---

## 13. Moving Averages (Window Frame in Practice)

**Definition** — An average computed only over a **sliding window** of N most recent rows — a direct, practical use of `ROWS BETWEEN`.

**Example — 3-day moving average of sales**
```sql
SELECT sale_date, sales,
       AVG(sales) OVER (
           ORDER BY sale_date
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3day
FROM sales;
```
*(For any given row, the window is exactly: 2 rows before it + itself = 3 rows total.)*

---

## 14. Interview Q&A

**Q: What does `NTILE(4)` do?**
A: Divides the ordered rows into 4 roughly equal-sized groups (buckets) and returns each row's bucket number — commonly used for quartile-style analysis.

**Q: If rows don't divide evenly across `NTILE` buckets, which buckets get the extra rows?**
A: The **earlier** buckets — PostgreSQL distributes remainder rows starting from bucket 1 onward.

**Q: Does `NTILE` guarantee tied values stay in the same bucket?**
A: No — unlike `RANK()`/`DENSE_RANK()`, `NTILE`'s priority is equal-sized groups, so identical values can land in adjacent buckets.

**Q: What does `LAG()` return for the very first row in its partition?**
A: `NULL` by default (or a custom default value if a third argument is supplied) — there's no row before it to look back at.

**Q: How would you calculate the salary difference between each employee and the one before them (by ID)?**
A: `salary - LAG(salary) OVER (ORDER BY emp_id) AS difference`.

**Q: Why does `LAST_VALUE()` often return the current row's own value instead of the group's actual last value?**
A: Because the default window frame (when `ORDER BY` is present) spans from the start of the partition up to the *current row* — so the "last row" inside that frame is almost always the current row itself. Fixing it requires explicitly widening the frame with `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

**Q: What's a simpler alternative to `LAST_VALUE()` for finding a group's maximum value per row?**
A: `FIRST_VALUE()` with `ORDER BY column DESC` — it avoids needing to manually widen the window frame.

**Q: How do you compute a 7-day moving average in SQL?**
A: `AVG(value) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`.

---

## 15. Quick Revision Sheet

| Need | Syntax |
|---|---|
| Equal-sized groups | `NTILE(n) OVER (ORDER BY col)` |
| Previous row's value | `LAG(col) OVER (ORDER BY col)` |
| Next row's value | `LEAD(col) OVER (ORDER BY col)` |
| N rows back/ahead | `LAG(col, n)` / `LEAD(col, n)` |
| Custom default instead of NULL | `LAG(col, 1, default_value)` |
| First value in window | `FIRST_VALUE(col) OVER (...)` |
| Last value in **entire** partition | `LAST_VALUE(col) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` |
| Nth value in **entire** partition | `NTH_VALUE(col, n) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` |
| Moving average (last 3 rows) | `AVG(col) OVER (ORDER BY col ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` |

---

## 16. Cheat Sheet

```sql
-- ── NTILE ─────────────────────────────────
SELECT name, salary, NTILE(4) OVER (ORDER BY salary DESC) AS quartile FROM employees;

-- ── LAG / LEAD ────────────────────────────
SELECT salary, LAG(salary)  OVER (ORDER BY emp_id) AS prev_salary FROM employees;
SELECT salary, LEAD(salary) OVER (ORDER BY emp_id) AS next_salary FROM employees;
SELECT salary, LAG(salary, 2)    OVER (ORDER BY emp_id) AS two_back       FROM employees;
SELECT salary, LAG(salary, 1, 0) OVER (ORDER BY emp_id) AS prev_or_zero   FROM employees;

SELECT name, salary,
       salary - LAG(salary) OVER (ORDER BY emp_id) AS difference
FROM employees;

SELECT sale_date, sales,
       CASE WHEN sales > LAG(sales) OVER (ORDER BY sale_date)
            THEN 'Increase' ELSE 'Decrease' END AS trend
FROM sales;

-- ── WINDOW FRAMES ─────────────────────────
... OVER (ORDER BY col ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)     -- default-like
... OVER (ORDER BY col ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)  -- whole partition
... OVER (ORDER BY col ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)              -- neighbors only
... OVER (ORDER BY col ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)               -- moving avg window

-- ── FIRST_VALUE / LAST_VALUE / NTH_VALUE ──
SELECT salary, FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary) AS lowest
FROM employees;

SELECT salary,
       LAST_VALUE(salary) OVER (
           ORDER BY emp_id ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS true_last
FROM employees;

SELECT salary,
       NTH_VALUE(salary, 2) OVER (
           ORDER BY emp_id ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS second_value
FROM employees;

-- ── MOVING AVERAGE ────────────────────────
SELECT sale_date, sales,
       AVG(sales) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3day
FROM sales;
```

---

## 17. Preview of Part 19

| Topic | What You'll Learn |
|---|---|
| The window function decision tree | Instantly matching an interview question to the right function |
| Top-N-per-group, 2nd highest salary, running totals | Fully worked interview patterns |
| Why window functions can't be filtered in `WHERE` | The logical execution order, explained precisely |
| Master cheat sheet | Every window function pattern in one place |
