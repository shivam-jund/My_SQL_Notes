# SQL & PostgreSQL Complete Notes — Part 11: Aggregate Functions & GROUP BY

## 📑 Table of Contents
1. What Are Aggregate Functions?
2. `COUNT()`
3. `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`
4. `SUM()`
5. `AVG()`
6. `MAX()`
7. `MIN()`
8. `GROUP BY`
9. Aggregate Functions Combined with `GROUP BY`
10. The `GROUP BY` Rule (What's Allowed in `SELECT`)
11. `GROUP BY` with Multiple Columns
12. `WHERE` + `GROUP BY` — Order Matters
13. Aggregates Without `GROUP BY`
14. Aggregate Functions vs Window Functions — A First Look
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Preview of Part 12

**📋 Series Coverage (Part 11):** `COUNT()`, `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT ...)`, `SUM()`, `AVG()`, `MAX()`, `MIN()`, NULL-handling in aggregates, `GROUP BY`, `GROUP BY` with multiple columns, `WHERE` vs aggregate calculation ordering, aggregates without `GROUP BY`, a first comparison against window functions

> Examples use:
> ```
> employees: emp_id | name | department | salary
> 1 | Aman   | IT    | 50000
> 2 | Ravi   | HR    | 40000
> 3 | Priya  | IT    | 70000
> 4 | Karan  | Sales | 60000
> 5 | Simran | HR    | 45000
> 6 | Arjun  | IT    | 80000
> ```

---

## 1. What Are Aggregate Functions?

**Definition** — Functions that take **many rows** and calculate **one** summary result from them.

```
50000, 40000, 70000, 60000, 45000, 80000
              ↓  SUM()
            345000
```

---

## 2. `COUNT()`

**Definition** — Counts rows, or non-NULL values in a specific column.

**Syntax**
```sql
COUNT(*)
COUNT(column)
COUNT(DISTINCT column)
```

**Example**
```sql
SELECT COUNT(*) FROM employees;
```
**Output**
```
count
6
```

---

## 3. `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`

| Form | Counts |
|---|---|
| `COUNT(*)` | All rows, regardless of NULLs |
| `COUNT(column)` | Only **non-NULL** values in that column |
| `COUNT(DISTINCT column)` | Only **unique, non-NULL** values |

**Example** — with `bonus`: `5000, NULL, 7000, NULL, 3000, 8000`
```sql
SELECT COUNT(*)         FROM employees;  -- 6 (all rows)
SELECT COUNT(bonus)     FROM employees;  -- 4 (skips the 2 NULLs)
SELECT COUNT(DISTINCT department) FROM employees;  -- 3 (IT, HR, Sales)
```

❌ **Common Mistakes** — Assuming `COUNT(column)` behaves the same as `COUNT(*)`. It doesn't — NULLs are silently excluded.

---

## 4. `SUM()`

**Definition** — Adds numeric values together, **ignoring `NULL`s**.

```sql
SELECT SUM(salary) FROM employees;             -- 345000
SELECT SUM(salary) FROM employees WHERE department = 'IT';  -- 200000
```

---

## 5. `AVG()`

**Definition** — Calculates the average of non-NULL numeric values: `SUM(non-NULL values) ÷ COUNT(non-NULL values)` — **not** `÷ COUNT(*)`.

```sql
SELECT AVG(salary) FROM employees;   -- 345000 / 6 = 57500
```

⚠️ **Notes & Caveats** — If `bonus` has values `5000, NULL, 7000`, `AVG(bonus)` computes `(5000+7000)/2 = 6000`, **not** `(5000+0+7000)/3`. `NULL`s are excluded from both the sum and the count used for averaging.

---

## 6. `MAX()`

**Definition** — Returns the largest value in a column.

```sql
SELECT MAX(salary) FROM employees;   -- 80000
```

⚠️ **Notes & Caveats** — `MAX(salary)` returns only the salary value — **not** the whole row/employee that earns it. To find *who* earns it, you need a subquery or `ORDER BY ... LIMIT 1` (covered in later parts).

---

## 7. `MIN()`

**Definition** — Returns the smallest value in a column.

```sql
SELECT MIN(salary) FROM employees;   -- 40000
```

**All five together**
```sql
SELECT
    COUNT(*)      AS total_employees,
    SUM(salary)   AS total_salary,
    AVG(salary)   AS average_salary,
    MAX(salary)   AS highest_salary,
    MIN(salary)   AS lowest_salary
FROM employees;
```
**Output**
```
total_employees | total_salary | average_salary | highest_salary | lowest_salary
6                | 345000       | 57500           | 80000           | 40000
```
*(6 input rows collapse into exactly 1 summary row — this is the defining trait of aggregate functions without `GROUP BY`.)*

---

## 8. `GROUP BY`

**Definition** — Splits rows into groups (based on matching column values); aggregate functions then compute a **separate result per group** instead of one result overall.

**Syntax**
```sql
SELECT group_column, AGG_FUNC(other_column)
FROM table_name
GROUP BY group_column;
```

**Example**
```sql
SELECT department, AVG(salary) AS average_salary
FROM employees
GROUP BY department;
```
**Output**
```
department | average_salary
IT         | 66666.67
HR         | 42500
Sales      | 60000
```

**Mental model**
```
All employees
     ↓ GROUP BY department
┌──────────┐  ┌──────────┐  ┌──────────┐
│ IT       │  │ HR       │  │ Sales    │
│ 50000    │  │ 40000    │  │ 60000    │
│ 70000    │  │ 45000    │  └──────────┘
│ 80000    │  └──────────┘
└──────────┘
     ↓ AVG() inside each box
  66666.67       42500          60000
```

---

## 9. Aggregate Functions Combined with `GROUP BY`

```sql
SELECT department, COUNT(*)      AS total_employees FROM employees GROUP BY department;
SELECT department, SUM(salary)   AS total_salary     FROM employees GROUP BY department;
SELECT department, MAX(salary)   AS highest_salary    FROM employees GROUP BY department;
SELECT department, MIN(salary)   AS lowest_salary      FROM employees GROUP BY department;

-- All together
SELECT department,
       COUNT(*)    AS total_employees,
       SUM(salary) AS total_salary,
       AVG(salary) AS average_salary,
       MAX(salary) AS highest_salary,
       MIN(salary) AS lowest_salary
FROM employees
GROUP BY department;
```

---

## 10. The `GROUP BY` Rule (What's Allowed in `SELECT`)

**The Rule** — Every column in `SELECT` must be either (a) listed in `GROUP BY`, or (b) wrapped inside an aggregate function.

❌ **Common Mistakes**
```sql
-- ❌ 'name' is neither grouped nor aggregated — which name would represent the IT group?
SELECT department, name, AVG(salary)
FROM employees
GROUP BY department;
-- ERROR: column "employees.name" must appear in the GROUP BY clause
--        or be used in an aggregate function
```
```sql
-- ✅ Every SELECT column is either grouped or aggregated
SELECT department, COUNT(*), AVG(salary)
FROM employees
GROUP BY department;
```

**Decision test**
```
For each SELECT column, ask:
  Is it inside COUNT/SUM/AVG/MAX/MIN?
    YES → fine
    NO  → it must appear in GROUP BY
```

---

## 11. `GROUP BY` with Multiple Columns

**Definition** — Groups by the **combination** of listed columns, not each independently.

```sql
SELECT department, city, COUNT(*) AS total_employees
FROM employees
GROUP BY department, city;
```
**Output**
```
department | city       | count
IT         | Mohali     | 2
IT         | Delhi      | 1
HR         | Chandigarh | 1
...
```
Each unique `(department, city)` pair forms its own group.

---

## 12. `WHERE` + `GROUP BY` — Order Matters

**Definition** — `WHERE` filters individual rows **before** grouping happens.

```sql
SELECT department, AVG(salary) AS average_salary
FROM employees
WHERE salary > 45000
GROUP BY department;
```

**Mental execution order**
```
FROM employees
   ↓
WHERE salary > 45000     ← remove rows first
   ↓
GROUP BY department      ← group what's left
   ↓
AVG(salary)               ← compute per group
```

⚠️ **Notes & Caveats** — If `WHERE` removes every row of a group entirely (e.g., every HR employee earns ≤ 45000), that group **disappears entirely** from the result — it's not shown with a zero/empty value.

---

## 13. Aggregates Without `GROUP BY`

**Definition** — Without `GROUP BY`, **all matching rows are treated as one single group**.

```sql
SELECT AVG(salary) FROM employees;   -- one row: the company-wide average
```

| | Without `GROUP BY` | With `GROUP BY` |
|---|---|---|
| Number of groups | 1 (everything) | One per distinct group value |
| Result rows | 1 | 1 per group |

---

## 14. Aggregate Functions vs Window Functions — A First Look

**Definition** — `GROUP BY` + aggregate **collapses** multiple rows into one summary row per group. Window functions (full coverage later in this series) compute an aggregate-like value **without** collapsing rows — every original row survives.

```sql
-- GROUP BY: 6 rows in → 3 rows out
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- Window function (preview): 6 rows in → 6 rows out, each annotated
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS department_avg
FROM employees;
```
**Output (window function version)**
```
name   | department | salary | department_avg
Aman   | IT         | 50000  | 66666.67
Priya  | IT         | 70000  | 66666.67
Arjun  | IT         | 80000  | 66666.67
Ravi   | HR         | 40000  | 42500
...
```

💡 We'll cover `OVER()`, `PARTITION BY`, and the full window function family in a dedicated part later in this series.

---

## 15. Interview Q&A

**Q: Difference between `COUNT(*)` and `COUNT(column)`?**
A: `COUNT(*)` counts every row regardless of NULLs; `COUNT(column)` only counts rows where that specific column is non-NULL.

**Q: Does `AVG()` divide by the total row count or the non-NULL count?**
A: The non-NULL count — `AVG()` ignores NULLs both in the sum and in the divisor, so it's really `SUM(non-null) / COUNT(non-null)`.

**Q: What is `GROUP BY` doing conceptually?**
A: Splitting rows into buckets based on matching column values, so aggregate functions compute a separate result per bucket instead of one result for the whole table.

**Q: Why does `SELECT department, name, AVG(salary) FROM employees GROUP BY department` fail?**
A: Because `name` isn't grouped and isn't wrapped in an aggregate function — since a department group can contain multiple different names, SQL has no single value to return for `name` per group.

**Q: What happens to a group if `WHERE` removes all its rows before `GROUP BY` runs?**
A: The group disappears from the output entirely — it isn't shown with a zero or NULL value, because there was nothing left to group by the time `GROUP BY` ran.

**Q: How many result rows does an aggregate function return without `GROUP BY`?**
A: Exactly one — all matching rows are treated as a single implicit group.

**Q: How is a window function different from `GROUP BY` + aggregate?**
A: `GROUP BY` collapses multiple rows into one summary row per group. A window function computes a similar aggregate value but keeps every original row intact, attaching the calculated value alongside each one.

---

## 16. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Count all rows | `COUNT(*)` |
| Count non-NULL values | `COUNT(column)` |
| Count unique values | `COUNT(DISTINCT column)` |
| Total | `SUM(column)` |
| Average (ignores NULL) | `AVG(column)` |
| Largest value | `MAX(column)` |
| Smallest value | `MIN(column)` |
| Per-group results | `GROUP BY column` |
| Multi-column grouping | `GROUP BY col1, col2` |
| Filter before grouping | `WHERE ... GROUP BY ...` |

---

## 17. Cheat Sheet

```sql
-- ── AGGREGATE FUNCTIONS ───────────────────
SELECT COUNT(*)               FROM employees;             -- all rows
SELECT COUNT(salary)          FROM employees;              -- non-NULL only
SELECT COUNT(DISTINCT department) FROM employees;           -- unique values
SELECT SUM(salary)            FROM employees;
SELECT AVG(salary)            FROM employees;               -- ignores NULL
SELECT MAX(salary)            FROM employees;
SELECT MIN(salary)            FROM employees;

-- ── GROUP BY ──────────────────────────────
SELECT department, COUNT(*) AS total_employees
FROM employees GROUP BY department;

SELECT department, city, COUNT(*)
FROM employees GROUP BY department, city;              -- combination groups

SELECT department, AVG(salary)
FROM employees
WHERE salary > 45000                                     -- filter FIRST
GROUP BY department;                                       -- then group

SELECT department,
       COUNT(*)    AS total_employees,
       SUM(salary) AS total_salary,
       AVG(salary) AS average_salary,
       MAX(salary) AS highest_salary,
       MIN(salary) AS lowest_salary
FROM employees
GROUP BY department;
```

---

## 18. Preview of Part 12

| Topic | What You'll Learn |
|---|---|
| `HAVING` | Filtering **groups** after `GROUP BY` |
| `WHERE` vs `HAVING` | The single most common SQL interview question |
| Combining both | `WHERE` (row filter) + `GROUP BY` + `HAVING` (group filter) in one query |
