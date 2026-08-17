# SQL & PostgreSQL Complete Notes — Part 17: Window Functions — Foundations & Ranking

## 📑 Table of Contents
1. What Is a Window Function?
2. Why Not Just `GROUP BY`?
3. `OVER()` — The Core Idea
4. `PARTITION BY`
5. `ORDER BY` Inside `OVER()`
6. `ROW_NUMBER()`
7. `RANK()`
8. `DENSE_RANK()`
9. `ROW_NUMBER()` vs `RANK()` vs `DENSE_RANK()` — Side by Side
10. `PARTITION BY` with Ranking Functions
11. Choosing the Right Ranking Function
12. Interview Q&A
13. Quick Revision Sheet
14. Cheat Sheet
15. Preview of Part 18

**📋 Series Coverage (Part 17):** window function fundamentals, `OVER()`, `PARTITION BY`, `ORDER BY` inside `OVER()` (window order vs display order), `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, ranking with duplicate values, ranking + `PARTITION BY`

> ⭐ This phase is flagged **"very important"** in the roadmap — it's one of the most heavily interviewed SQL topics. Don't memorize the function names; understand how the "window" of visible rows moves, and every function here becomes obvious.

---

## 1. What Is a Window Function?

**Definition** — A function that performs a calculation across a set of related rows **while keeping every original row** in the output — nothing collapses.

**Why It Exists** — Sometimes you need a row's own detail **and** a group-level summary in the *same* row (e.g., "show me each employee's salary next to their company's average salary").

**Example**
```sql
SELECT name, salary, AVG(salary) OVER () AS company_avg
FROM employees;
```
**Output**
```
name  | salary | company_avg
Aman  | 50000  | 57500
Ravi  | 40000  | 57500
Priya | 70000  | 57500
...
```
*(Still 6 rows in, 6 rows out — only one extra column was added.)*

⚠️ **Notes & Caveats** — The name "window" comes from the idea of a camera/window that can only "see" a certain set of rows for each calculation — and that visible set can move as you move to the next row.

---

## 2. Why Not Just `GROUP BY`?

**The core difference — rows survive.**

```sql
-- GROUP BY: rows COLLAPSE
SELECT department, AVG(salary) FROM employees GROUP BY department;
-- 6 employee rows → 3 department rows

-- Window function: rows STAY
SELECT name, department, salary, AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
-- 6 employee rows → still 6 rows, each annotated with its department's average
```

| | `GROUP BY` | Window Function |
|---|---|---|
| Row count | Reduced (one row per group) | Unchanged (original row count preserved) |
| Purpose | Summarize | Annotate each row with a summary value |
| Can mix detail + summary in one row? | ❌ No | ✅ Yes |

💡 **Memory trick:** `GROUP BY` = many rows → **one** row · Window function = many rows → **still many** rows, plus an extra calculated column.

---

## 3. `OVER()` — The Core Idea

**Definition** — `OVER(...)` is what turns an ordinary aggregate-style function into a **window function**. It tells SQL: *which rows should participate in this calculation for the current row?*

**Syntax**
```sql
FUNCTION(column) OVER (
    [PARTITION BY column, ...]
    [ORDER BY column, ...]
    [frame_clause]
)
```

**Example — simplest possible window function**
```sql
SELECT name, salary, AVG(salary) OVER () AS avg_salary
FROM employees;
```

⚠️ **Notes & Caveats** — With `OVER()` and **nothing inside the parentheses**, the "window" is the **entire table** — every row gets the exact same result, because every row is being compared against the whole thing, unchanged.

💡 Without `OVER(...)`, `AVG(salary)` is a normal aggregate (one summary value, needs `GROUP BY` to work per group). **With** `OVER(...)`, it becomes a window function.

---

## 4. `PARTITION BY`

**Definition** — Splits the table into independent sub-groups **before** the window function runs; each partition gets its own self-contained window/calculation.

**Why It Exists** — You rarely want one calculation across the *entire* table — usually you want it *per department*, *per customer*, *per category*, etc.

**Syntax**
```sql
FUNCTION(column) OVER (PARTITION BY grouping_column)
```

**Example**
```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```
**Output**
```
name  | department | salary | dept_avg
Aman  | IT         | 50000  | 66666.67
Priya | IT         | 70000  | 66666.67
Arjun | IT         | 80000  | 66666.67
Ravi  | HR         | 40000  | 42500
Simran| HR         | 45000  | 42500
```

💡 **Analogy** — Think classrooms, not the whole school: "average marks *within Class A*," then separately "average marks *within Class B*" — each partition's calculation never leaks into another partition.

---

## 5. `ORDER BY` Inside `OVER()`

**Definition** — Determines the **order in which rows are processed** for the window's calculation — this is fundamentally different from a normal `ORDER BY`, which only controls display order of the final result.

**Syntax**
```sql
FUNCTION(column) OVER (ORDER BY column)
FUNCTION(column) OVER (PARTITION BY column ORDER BY column)
```

**Example — running total**
```sql
SELECT emp_id, salary, SUM(salary) OVER (ORDER BY emp_id) AS running_total
FROM employees;
```
**Output**
```
emp_id | salary | running_total
1      | 50000  | 50000
2      | 40000  | 90000
3      | 70000  | 160000
4      | 60000  | 220000
```

**Mental picture — the window grows as it moves**
```
Row 1: [50000]                        → 50000
Row 2: [50000, 40000]                  → 90000
Row 3: [50000, 40000, 70000]            → 160000
Row 4: [50000, 40000, 70000, 60000]      → 220000
```

⚠️ **Notes & Caveats — a classic interview point:**

| | Normal `ORDER BY` (end of query) | `ORDER BY` inside `OVER()` |
|---|---|---|
| Controls | Final display order of the whole result | The sequence in which rows are consumed for the running calculation |
| Changes row order in output? | ✅ Yes | ❌ Not by itself — you still need a normal `ORDER BY` at the end if you want the *displayed* rows sorted |

**`PARTITION BY` + `ORDER BY` together (running total per group)**
```sql
SELECT department, emp_id, salary,
       SUM(salary) OVER (PARTITION BY department ORDER BY emp_id) AS running_total
FROM employees;
```
The running total **restarts at 0** every time a new partition (department) begins.

---

## 6. `ROW_NUMBER()`

**Definition** — Assigns a **unique**, sequential integer to every row within its partition, in the given order — never repeats, even for tied values.

**Syntax**
```sql
ROW_NUMBER() OVER (PARTITION BY col ORDER BY col)
```

**Example** — salaries with duplicates: `80000, 70000, 70000, 60000, 50000, 50000`
```sql
SELECT name, salary, ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num
FROM employees;
```
**Output**
```
name   | salary | row_num
Aman   | 80000  | 1
Priya  | 70000  | 2
Arjun  | 70000  | 3    ← same salary as Priya, but a DIFFERENT number
Karan  | 60000  | 4
Simran | 50000  | 5
Ravi   | 50000  | 6
```

💡 **Analogy** — attendance roll numbers: even identical twins get different roll numbers.

⚠️ **Notes & Caveats** — `ROW_NUMBER()` completely **ignores** whether values are tied — it only counts rows.

---

## 7. `RANK()`

**Definition** — Gives **the same rank to tied values**, but **skips** the subsequent rank number(s) by however many rows tied.

**Syntax**
```sql
RANK() OVER (PARTITION BY col ORDER BY col)
```

**Example**
```sql
SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS rnk
FROM employees;
```
**Output**
```
name   | salary | rnk
Aman   | 80000  | 1
Priya  | 70000  | 2
Arjun  | 70000  | 2   ← tied
Karan  | 60000  | 4   ← rank 3 is SKIPPED (two rows already claimed rank 2)
Simran | 50000  | 5
Ravi   | 50000  | 5   ← tied
```

💡 **Analogy** — Olympic medals: two athletes tie for Silver (rank 2), so nobody gets Bronze (rank 3) — the next athlete is rank 4.

---

## 8. `DENSE_RANK()`

**Definition** — Gives the same rank to tied values, but **never skips** — the next distinct value always gets the immediately following integer.

**Syntax**
```sql
DENSE_RANK() OVER (PARTITION BY col ORDER BY col)
```

**Example**
```sql
SELECT name, salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS d_rnk
FROM employees;
```
**Output**
```
name   | salary | d_rnk
Aman   | 80000  | 1
Priya  | 70000  | 2
Arjun  | 70000  | 2
Karan  | 60000  | 3   ← no gap, immediately follows 2
Simran | 50000  | 4
Ravi   | 50000  | 4
```

💡 **Analogy** — distinct salary *levels*: there are only 4 unique salary values in this data, so ranks only ever go 1–4.

---

## 9. `ROW_NUMBER()` vs `RANK()` vs `DENSE_RANK()` — Side by Side

| Salary | `ROW_NUMBER()` | `RANK()` | `DENSE_RANK()` |
|---|---|---|---|
| 80000 | 1 | 1 | 1 |
| 70000 | 2 | 2 | 2 |
| 70000 | 3 | 2 | 2 |
| 60000 | 4 | 4 | 3 |
| 50000 | 5 | 5 | 4 |
| 50000 | 6 | 5 | 4 |

| Function | Duplicates get same value? | Skips a number after a tie? | Always unique? |
|---|---|---|---|
| `ROW_NUMBER()` | ❌ No | ❌ No | ✅ Yes |
| `RANK()` | ✅ Yes | ✅ Yes | ❌ No |
| `DENSE_RANK()` | ✅ Yes | ❌ No | ❌ No |

❌ **Common Mistakes**
- Thinking `ROW_NUMBER()` "checks" for duplicates — it doesn't; it just counts rows regardless.
- Thinking `RANK()` never skips — it does, by design, whenever there's a tie.
- Thinking `DENSE_RANK()` skips — it specifically never does; that's its entire purpose.

---

## 10. `PARTITION BY` with Ranking Functions

**Example — rank employees within their own department**
```sql
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
```
**Output**
```
name   | department | salary | dept_rank
Arjun  | IT         | 80000  | 1
Priya  | IT         | 70000  | 2
Aman   | IT         | 50000  | 3
Simran | HR         | 45000  | 1
Ravi   | HR         | 40000  | 2
```

⚠️ **Notes & Caveats** — Ranking **restarts at 1** for every new partition — this is the same "new partition = new window" rule from Section 4.

💡 **Top-1-per-group pattern (previewed here, used constantly in Part 19):**
```sql
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;    -- highest earner per department
```
*(Remember from Part 16: a window function's alias can't be filtered in the same query's `WHERE` — you must wrap it in a CTE or subquery first. Fully explained in Part 19.)*

---

## 11. Choosing the Right Ranking Function

```
Need a plain unique serial number?               → ROW_NUMBER()
Need EXACTLY N rows (e.g. "top 3 rows")?          → ROW_NUMBER()
Need top N DISTINCT value levels, ties included?  → DENSE_RANK()
Need true competition-style ranking (with gaps)?  → RANK()
```

**Example — the difference matters for "Top 2"**
Salaries: `80000, 70000, 70000, 60000`
```sql
-- ROW_NUMBER: exactly 2 rows — the second 70000 earner is UNFAIRLY excluded
... WHERE row_num <= 2;    → 80000, 70000 (first one only)

-- RANK: both 70000 earners included — fairer for "top salaries"
... WHERE rnk <= 2;        → 80000, 70000, 70000
```

---

## 12. Interview Q&A

**Q: What's the fundamental difference between `GROUP BY` and a window function?**
A: `GROUP BY` collapses multiple rows into one summary row per group. A window function keeps every original row and simply attaches a calculated value alongside each one.

**Q: What does `PARTITION BY` do inside `OVER()`?**
A: It splits the table into independent groups before the window function runs, so the calculation restarts fresh for each group — analogous to `GROUP BY`, but without collapsing rows.

**Q: How is `ORDER BY` inside `OVER()` different from a normal `ORDER BY`?**
A: The one inside `OVER()` controls the sequence rows are processed in for the window calculation (e.g., producing a running total); it does not by itself change the final displayed row order — you'd still need a separate `ORDER BY` at the end of the query for that.

**Q: `ROW_NUMBER()` vs `RANK()` — how do they treat tied values differently?**
A: `ROW_NUMBER()` gives every row a different number regardless of ties. `RANK()` gives tied rows the same rank, then skips the next rank number(s) by the count of tied rows.

**Q: Why does `DENSE_RANK()` exist if `RANK()` already handles ties?**
A: `DENSE_RANK()` is for when you want ranks to represent distinct value *levels* without gaps — useful for questions like "find the second-highest **distinct** salary," where a gap in numbering would be misleading.

**Q: You need "the top 3 highest-paid employees" but ties should all be included — which function?**
A: `DENSE_RANK()`, filtering `<= 3` — this returns every employee at the top 3 distinct salary levels, even if more than 3 rows share those levels.

**Q: You need exactly 3 rows back, no more, no less — which function?**
A: `ROW_NUMBER()`, filtering `= 1, 2, 3` (or `<= 3`) — it guarantees a unique number per row, so you always get exactly the count you filter for.

**Q: Does `PARTITION BY` reset ranking for each group?**
A: Yes — ranking functions restart from 1 at the beginning of every new partition.

---

## 13. Quick Revision Sheet

| Need | Function |
|---|---|
| Whole table as one window | `OVER()` |
| Independent groups | `OVER (PARTITION BY col)` |
| Running/cumulative calc | `OVER (ORDER BY col)` |
| Per-group running calc | `OVER (PARTITION BY col ORDER BY col)` |
| Unique serial number | `ROW_NUMBER()` |
| Competition-style rank (gaps) | `RANK()` |
| Rank without gaps | `DENSE_RANK()` |
| Exactly N rows | `ROW_NUMBER() <= N` |
| Top N levels incl. ties | `DENSE_RANK() <= N` |

---

## 14. Cheat Sheet

```sql
-- ── BASIC WINDOW ──────────────────────────
SELECT name, salary, AVG(salary) OVER () AS company_avg FROM employees;

-- ── PARTITION BY ──────────────────────────
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;

-- ── ORDER BY (running total) ──────────────
SELECT emp_id, salary, SUM(salary) OVER (ORDER BY emp_id) AS running_total
FROM employees;

-- ── PARTITION BY + ORDER BY (per-group running total) ──
SELECT department, emp_id, salary,
       SUM(salary) OVER (PARTITION BY department ORDER BY emp_id) AS dept_running_total
FROM employees;

-- ── RANKING FUNCTIONS ─────────────────────
SELECT name, salary, ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num   FROM employees;
SELECT name, salary, RANK()       OVER (ORDER BY salary DESC) AS rnk       FROM employees;
SELECT name, salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS d_rnk     FROM employees;

-- ── RANKING PER PARTITION ─────────────────
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;

-- ── TOP-1-PER-GROUP PATTERN ───────────────
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;
```

---

## 15. Preview of Part 18

| Topic | What You'll Learn |
|---|---|
| `NTILE(n)` | Splitting rows into equal-sized buckets (quartiles, deciles) |
| `LAG()` / `LEAD()` | Reading the previous/next row without a self join |
| Window Frames (`ROWS BETWEEN`) | Exactly which rows a window function "sees" |
| `FIRST_VALUE()` / `LAST_VALUE()` / `NTH_VALUE()` | Pulling specific values from a window — including the famous `LAST_VALUE()` trap |
