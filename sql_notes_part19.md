# SQL & PostgreSQL Complete Notes — Part 19: Window Functions — Interview Patterns & Mistakes

## 📑 Table of Contents
1. The Window Function Decision Tree
2. Pattern: Highest-Paid Employee Per Group (Top-1-per-Group)
3. Pattern: Second Highest (Distinct) Value
4. Pattern: Top N Rows vs. Top N Value Levels
5. Pattern: Difference From Previous Row & Trend Detection
6. Pattern: Running Total & Running Average
7. Pattern: Moving Average
8. Pattern: First/Last Value Per Group
9. Pattern: Quartiles / Top 25% (`NTILE`)
10. Why Window Functions Can't Be Used Directly in `WHERE`
11. The Full SQL Logical Execution Order (Window Functions Added)
12. Window Functions vs `GROUP BY` — Final Recap
13. Common Mistakes Roundup
14. Master Interview Cheat Sheet
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Preview of Part 20

**📋 Series Coverage (Part 19):** window function decision tree, top-N-per-group pattern, 2nd/Nth highest salary pattern, running totals, running/moving averages, `WHERE` + window function ordering problem, full logical execution order including window functions, consolidated common-mistakes review, master pattern cheat sheet

---

## 1. The Window Function Decision Tree

Memorize this — it turns almost any window-function interview question into an instant function choice:

```
Need a unique row number?             → ROW_NUMBER()
Need competition-style ranking?        → RANK()
Need ranking without gaps?             → DENSE_RANK()
Need equal-sized groups?               → NTILE(n)
Need the previous row's value?         → LAG()
Need the next row's value?             → LEAD()
Need the first/last/nth value in a group? → FIRST_VALUE() / LAST_VALUE() / NTH_VALUE()
Need a running or moving total/average? → SUM()/AVG() OVER (... ROWS BETWEEN ...)
```

---

## 2. Pattern: Highest-Paid Employee Per Group (Top-1-per-Group)

**The question:** *"Return the highest-paid employee from each department."*

```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;
```

⚠️ **Notes & Caveats** — If two employees tie for highest in a department and the interviewer wants **both** returned, use `RANK()` (or `DENSE_RANK()`) instead of `ROW_NUMBER()` — `ROW_NUMBER()` will arbitrarily pick just one.

---

## 3. Pattern: Second Highest (Distinct) Value

**The question:** *"Find the second highest salary."*

```sql
WITH ranked AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM employees
)
SELECT DISTINCT salary FROM ranked WHERE dr = 2;
```

💡 **Why `DENSE_RANK()`, not `RANK()` or `ROW_NUMBER()`?** — "Second highest salary" means the second **distinct** salary value. If salaries are `80000, 70000, 70000, 60000`, `DENSE_RANK()` correctly identifies `70000` as rank 2 (returning both employees who earn it); `ROW_NUMBER()` would arbitrarily split the tied 70000s into ranks 2 and 3, missing one of them if you filtered `= 2`.

---

## 4. Pattern: Top N Rows vs. Top N Value Levels

These sound similar but answer **different questions**.

| Question | Function |
|---|---|
| "Return exactly N rows" | `ROW_NUMBER() <= N` |
| "Return the top N distinct salary *levels*, including all ties" | `DENSE_RANK() <= N` |

**Example — salaries `100, 90, 90, 80`, asking for "top 2"**
```sql
-- "Exactly 2 rows" — the second 90-earner is excluded
... ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn ... WHERE rn <= 2;
→ 100, 90  (only the first 90)

-- "Top 2 salary levels" — both 90-earners included
... DENSE_RANK() OVER (ORDER BY salary DESC) AS dr ... WHERE dr <= 2;
→ 100, 90, 90
```

💡 Always clarify (mentally, or out loud in an interview) which version is being asked for — this is one of the most common sources of a "technically correct but not what they wanted" answer.

---

## 5. Pattern: Difference From Previous Row & Trend Detection

**Salary increase from the previous employee (by some order):**
```sql
SELECT name, salary,
       salary - LAG(salary) OVER (ORDER BY salary) AS increase
FROM employees;
```

**Comparing today's sales to yesterday's:**
```sql
SELECT sale_date, sales,
       sales - LAG(sales) OVER (ORDER BY sale_date) AS day_over_day_change
FROM sales;
```

**Detecting an increasing/decreasing trend:**
```sql
SELECT sale_date, sales,
       CASE
           WHEN sales > LAG(sales) OVER (ORDER BY sale_date) THEN 'Increase'
           ELSE 'Decrease'
       END AS trend
FROM sales;
```

---

## 6. Pattern: Running Total & Running Average

```sql
SELECT sale_date, sales,
       SUM(sales) OVER (ORDER BY sale_date) AS running_total,
       AVG(sales) OVER (ORDER BY sale_date) AS running_average
FROM sales;
```

**Mental picture — the window keeps growing**
```
Day 1: [100]                → total 100,  avg 100
Day 2: [100, 150]            → total 250,  avg 125
Day 3: [100, 150, 80]         → total 330,  avg 110
Day 4: [100, 150, 80, 70]      → total 400,  avg 100
```

---

## 7. Pattern: Moving Average

**The question:** *"Compute the average of the last 3 days for each day."*
```sql
SELECT sale_date, sales,
       AVG(sales) OVER (
           ORDER BY sale_date
           ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
       ) AS moving_avg_3day
FROM sales;
```
*(Unlike the running average in Section 6, this window has a **fixed size** — it slides forward, always covering exactly 3 rows, rather than growing indefinitely.)*

---

## 8. Pattern: First/Last Value Per Group

**Lowest salary per department:**
```sql
SELECT name, department, salary,
       FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary) AS lowest_in_dept
FROM employees;
```

**Highest salary per department — two options:**
```sql
-- Option 1 (simpler, preferred): sort DESC, take FIRST_VALUE
SELECT name, department, salary,
       FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS highest_in_dept
FROM employees;

-- Option 2: LAST_VALUE with the correct (full-partition) frame
SELECT name, department, salary,
       LAST_VALUE(salary) OVER (
           PARTITION BY department ORDER BY salary
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS highest_in_dept
FROM employees;
```
💡 **Best Practices** — Option 1 is simpler and avoids the `LAST_VALUE()` frame trap from Part 18 entirely — prefer it in interviews unless `LAST_VALUE()` is specifically what's being tested.

---

## 9. Pattern: Quartiles / Top 25% (`NTILE`)

```sql
SELECT name, salary, NTILE(4) OVER (ORDER BY salary DESC) AS quartile
FROM employees;
-- bucket 1 = top 25%, bucket 4 = bottom 25%
```

| Buckets Requested | Result |
|---|---|
| `NTILE(2)` | Top half / bottom half |
| `NTILE(4)` | Quartiles (top 25%, ..., bottom 25%) |
| `NTILE(10)` | Deciles (top 10%, ..., bottom 10%) |

---

## 10. Why Window Functions Can't Be Used Directly in `WHERE`

```sql
-- ❌ Invalid
SELECT *, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM employees
WHERE rn = 1;
-- ERROR: column "rn" does not exist
```

**Why:** Window functions are computed logically **after** `WHERE` (and even after `GROUP BY`/`HAVING`) in SQL's processing order — so at the moment `WHERE` runs, the alias `rn` doesn't exist yet.

✅ **The fix — compute in a CTE (or subquery), then filter the outer query**
```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;
```

💡 **Interview answer to remember:** *"Window functions are calculated after `WHERE`, so you can't filter on a window function's alias in the same `SELECT`. Compute it first in a CTE or subquery, then filter that result in the outer query."*

---

## 11. The Full SQL Logical Execution Order (Window Functions Added)

Extending Part 12's pipeline with window functions:

```
1. FROM             → get the base table(s) / joins
2. WHERE            → filter individual rows
3. GROUP BY         → create groups
4. HAVING           → filter groups
5. Window Functions  → compute OVER(...) calculations   ⭐ (this is why WHERE/HAVING can't see them)
6. SELECT            → choose/compute final output columns
7. ORDER BY           → sort the final result
8. LIMIT              → cap the row count
```

💡 This single ordering fact explains: why `WHERE` can't reference a `SELECT` alias, why `HAVING` can use aggregates but `WHERE` can't, and why window function results need a CTE/subquery wrapper before they can be filtered.

---

## 12. Window Functions vs `GROUP BY` — Final Recap

| | `GROUP BY` + Aggregate | Window Function |
|---|---|---|
| Row count | Reduced to one per group | Unchanged |
| Can show individual + group data together? | ❌ No | ✅ Yes |
| Can be filtered directly in the same query's `WHERE`? | ✅ Yes (row-level `WHERE`), aggregates need `HAVING` | ❌ No — needs a CTE/subquery wrapper |
| Typical question | "What's the average salary **per department**?" (one row per dept) | "Show each employee **alongside** their department's average" (all rows kept) |

---

## 13. Common Mistakes Roundup

❌ Using `ROW_NUMBER()` for "second highest salary" — ties can make you skip a valid answer; use `DENSE_RANK()`.

❌ Filtering a window function directly in `WHERE` — wrap it in a CTE or subquery first.

❌ Using `LAST_VALUE()` without widening its frame — remember, the default frame ends at the current row (Part 18, Section 11).

❌ Forgetting `ORDER BY` inside `OVER()` when "previous"/"next"/"running" is part of the question — without it, there's no defined sequence for `LAG()`/`LEAD()`/running calculations to follow.

❌ Assuming `NTILE` keeps tied values in the same bucket — it doesn't; it prioritizes equal group sizes.

---

## 14. Master Interview Cheat Sheet

| Problem | Best Function |
|---|---|
| Unique numbering | `ROW_NUMBER()` |
| Competition ranking (with gaps) | `RANK()` |
| Ranking without gaps | `DENSE_RANK()` |
| Top N *salary levels* (ties included) | `DENSE_RANK() <= N` |
| Top N *rows* (exact count) | `ROW_NUMBER() <= N` |
| Previous row's value | `LAG()` |
| Next row's value | `LEAD()` |
| Running total | `SUM() OVER (ORDER BY ...)` |
| Running average | `AVG() OVER (ORDER BY ...)` |
| Moving average (fixed window) | `AVG() OVER (ORDER BY ... ROWS BETWEEN n PRECEDING AND CURRENT ROW)` |
| Equal-sized buckets/quartiles | `NTILE(n)` |
| First value in a group | `FIRST_VALUE()` |
| Last value in a group (careful with frame!) | `LAST_VALUE()` + full-partition frame, or `FIRST_VALUE()` with `DESC` instead |

---

## 15. Interview Q&A

**Q: An interviewer asks for "the top 3 salaries" — what's your first clarifying thought?**
A: Whether they mean exactly 3 rows (`ROW_NUMBER() <= 3`) or the top 3 distinct salary *levels* including ties (`DENSE_RANK() <= 3`) — these can return different row counts if there are ties near the cutoff.

**Q: Why can't you write `WHERE row_num = 1` in the same `SELECT` that defines `row_num` via `ROW_NUMBER() OVER (...)`?**
A: Because window functions are logically computed after `WHERE` runs in SQL's execution order — the alias doesn't exist yet at that point. You need to compute it in a CTE or subquery first, then filter in the outer query.

**Q: What's the difference between a "running average" and a "moving average"?**
A: A running average's window keeps growing to include every row from the start up to the current one. A moving average uses a **fixed-size** sliding window (e.g., always exactly the last 3 rows), achieved with `ROWS BETWEEN n PRECEDING AND CURRENT ROW`.

**Q: How would you find the highest-paid employee per department, and what should you watch out for?**
A: Use `ROW_NUMBER()` (or `RANK()`/`DENSE_RANK()` if ties should all be returned) partitioned by department, ordered by salary descending, inside a CTE, then filter `WHERE rn = 1` in the outer query. Watch out for whether ties should return one row or all tied rows.

**Q: Where do window functions sit in SQL's logical execution order, relative to `WHERE`, `GROUP BY`, and `HAVING`?**
A: After all three — `FROM` → `WHERE` → `GROUP BY` → `HAVING` → window functions → `SELECT` → `ORDER BY` → `LIMIT`.

**Q: Give a simpler alternative to `LAST_VALUE()` for "highest value per group."**
A: `FIRST_VALUE()` ordered `DESC` — it returns the same result without needing to manually widen the window frame.

---

## 16. Quick Revision Sheet

| Interview Phrase | Function |
|---|---|
| "unique serial number" | `ROW_NUMBER()` |
| "rank like a competition" | `RANK()` |
| "rank with no gaps" | `DENSE_RANK()` |
| "top N *rows*" | `ROW_NUMBER() <= N` |
| "top N *values*, ties included" | `DENSE_RANK() <= N` |
| "compare to previous row" | `LAG()` |
| "compare to next row" | `LEAD()` |
| "running total/average" | `SUM()`/`AVG() OVER (ORDER BY ...)` |
| "moving/rolling average" | `AVG() OVER (... ROWS BETWEEN n PRECEDING AND CURRENT ROW)` |
| "divide into N equal groups" | `NTILE(n)` |
| "filter a window function result" | wrap in CTE, filter the CTE |

---

## 17. Cheat Sheet

```sql
-- ── TOP-1 (OR TOP-N) PER GROUP ────────────
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;

-- ── NTH HIGHEST (DISTINCT) VALUE ──────────
WITH ranked AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr FROM employees
)
SELECT DISTINCT salary FROM ranked WHERE dr = 2;    -- 2nd highest

-- ── TREND / DIFFERENCE FROM PREVIOUS ──────
SELECT sale_date, sales,
       sales - LAG(sales) OVER (ORDER BY sale_date) AS change,
       CASE WHEN sales > LAG(sales) OVER (ORDER BY sale_date)
            THEN 'Increase' ELSE 'Decrease' END AS trend
FROM sales;

-- ── RUNNING TOTAL / AVERAGE ────────────────
SELECT sale_date, sales,
       SUM(sales) OVER (ORDER BY sale_date) AS running_total,
       AVG(sales) OVER (ORDER BY sale_date) AS running_avg
FROM sales;

-- ── MOVING AVERAGE (fixed window) ─────────
SELECT sale_date, sales,
       AVG(sales) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3day
FROM sales;

-- ── QUARTILES ──────────────────────────────
SELECT name, salary, NTILE(4) OVER (ORDER BY salary DESC) AS quartile FROM employees;

-- ── FILTERING A WINDOW FUNCTION (must use CTE) ──
WITH ranked AS (SELECT *, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn FROM employees)
SELECT * FROM ranked WHERE rn <= 3;
```

---

## 18. Preview of Part 20

| Topic | What You'll Learn |
|---|---|
| `UPPER`, `LOWER`, `LENGTH`, `TRIM` | Basic string cleanup |
| `SUBSTRING`, `POSITION`, `REPLACE` | Extracting & modifying text |
| `CONCAT`, `SPLIT_PART`, `INITCAP` | Combining & splitting strings |
