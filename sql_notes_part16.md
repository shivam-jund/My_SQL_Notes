# SQL & PostgreSQL Complete Notes — Part 16: Common Table Expressions (CTEs)

## 📑 Table of Contents
1. What Is a CTE?
2. Basic `WITH` Syntax
3. A CTE Is Temporary — Scope Rules
4. CTE vs Subquery-in-`FROM` (Comparison)
5. CTE vs Temporary Table (Comparison)
6. CTE with `JOIN`
7. Quick Primer: `RANK()` / `OVER()` / `PARTITION BY`
8. CTE + Window Function — Solving "Can't Filter a Window Alias in `WHERE`"
9. Multiple CTEs
10. One CTE Referencing Another CTE
11. Recursive CTE — The Three-Question Framework
12. Recursive CTE Example: Number Series
13. Recursive CTE Example: Employee Hierarchy
14. Recursive CTE vs Self Join
15. Common Mistakes
16. Interview Q&A
17. Quick Revision Sheet
18. Cheat Sheet
19. Preview of Part 17

**📋 Series Coverage (Part 16):** `WITH`, CTE scope/lifetime, CTE vs subquery-in-FROM, CTE vs temp table, CTE + `JOIN`, CTE + window functions, multiple CTEs (comma-separated), CTEs referencing earlier CTEs, `WITH RECURSIVE`, anchor query, recursive query, `UNION ALL`, stop conditions, employee-hierarchy pattern, recursive CTE vs self join

---

## 1. What Is a CTE?

**Definition** — A CTE (Common Table Expression) is a **temporary, named query result**, defined with `WITH`, that the statement immediately following it can query like a table.

**Why It Exists** — Deeply nested subqueries (a query inside a query inside a query) get hard to read. A CTE lets you name each logical step, turning a nested mess into a readable, top-to-bottom pipeline.

```
😵 Subquery-in-subquery style:
SELECT ... FROM (SELECT ... FROM (SELECT ...) x) y JOIN (SELECT ...) z ON ...

🙂 CTE style:
WITH step1 AS (...), step2 AS (...)
SELECT ... FROM step1 JOIN step2 ...
```

---

## 2. Basic `WITH` Syntax

**Syntax**
```sql
WITH cte_name AS (
    SELECT ...
)
SELECT ...
FROM cte_name;
```

**Example**
```sql
WITH high_salary_employees AS (
    SELECT name, salary
    FROM employees
    WHERE salary > 50000
)
SELECT * FROM high_salary_employees;
```
**Output**
```
name  | salary
Priya | 70000
Karan | 60000
Arjun | 80000
```

**Example — replacing a subquery-in-FROM**
```sql
WITH department_averages AS (
    SELECT department_id, AVG(salary) AS average_salary
    FROM employees
    GROUP BY department_id
)
SELECT department_id, average_salary
FROM department_averages
WHERE average_salary > 50000;
```
*(Reads almost like English: "create department averages, then find department averages greater than 50000.")*

---

## 3. A CTE Is Temporary — Scope Rules

⚠️ **Notes & Caveats** — A CTE exists **only for the single SQL statement immediately following the `WITH`**. It is **not** written to the database as a real table.

❌ **Common Mistakes**
```sql
WITH high_salary AS (SELECT * FROM employees WHERE salary > 50000)
SELECT * FROM high_salary;

-- Later, as a SEPARATE statement:
SELECT * FROM high_salary;
-- ERROR: relation "high_salary" does not exist
```

---

## 4. CTE vs Subquery-in-`FROM` (Comparison)

| | Subquery-in-`FROM` | CTE |
|---|---|---|
| Style | Nested — query hidden inside brackets | Step-by-step — named first, used after |
| Readability | Gets messy with multiple levels | Reads top-to-bottom like a pipeline |
| Performance | — | Not automatically faster or slower; the PostgreSQL optimizer can treat equivalent forms similarly |

💡 **Best Practices** — A CTE isn't inherently faster than an equivalent subquery — its real advantage is **readability and organization**, especially once a query needs 2+ logical steps.

---

## 5. CTE vs Temporary Table (Comparison)

| | CTE | `CREATE TEMP TABLE` |
|---|---|---|
| Lifetime | One statement only | Survives across multiple statements in the session |
| Physically stored? | Not necessarily (optimizer-dependent) | Yes — an actual table object |
| Syntax | `WITH name AS (...)` | `CREATE TEMP TABLE name AS SELECT ...` |

💡 **How to Choose** — Use a CTE to organize a single complex query. Use a temp table when you genuinely need to reuse a result across several separate statements in the same session.

---

## 6. CTE with `JOIN`

**Example — solving "employees earning more than their department average" step by step**
```sql
WITH department_averages AS (
    SELECT department_id, AVG(salary) AS average_salary
    FROM employees
    GROUP BY department_id
)
SELECT e.name, e.salary, da.average_salary
FROM employees e
JOIN department_averages da ON e.department_id = da.department_id
WHERE e.salary > da.average_salary;
```
**Output**
```
name   | salary | average_salary
Priya  | 70000  | 66666.67
Simran | 45000  | 42500
Arjun  | 80000  | 66666.67
```

💡 This is the exact same result as the correlated subquery from Part 15 — but broken into two readable steps: *(1) compute averages, (2) join and compare.*

---

## 7. Quick Primer: `RANK()` / `OVER()` / `PARTITION BY`

*(Full window function coverage is coming in Part 17 — here's just enough to follow the next section.)*

| Piece | Meaning |
|---|---|
| `RANK() OVER (ORDER BY salary DESC)` | Assigns rank 1, 2, 3... to rows based on sort order — **without** collapsing rows like `GROUP BY` would |
| `PARTITION BY department_id` | Restarts the ranking separately **within each** department, instead of one ranking across the whole table |

```sql
SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees;
```
```
name  | salary | salary_rank
Arjun | 80000  | 1
Priya | 70000  | 2
Karan | 60000  | 3
...
```

---

## 8. CTE + Window Function — Solving "Can't Filter a Window Alias in `WHERE`"

⚠️ **Notes & Caveats** — You **cannot** filter directly on a window function's alias in the same `SELECT`'s `WHERE` clause:
```sql
-- ❌ Invalid — WHERE runs before the window function's alias exists
SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS salary_rank
FROM employees
WHERE salary_rank <= 3;
-- ERROR: column "salary_rank" does not exist
```
This is the same logical-ordering issue as `WHERE` vs. aggregate functions (Part 12) — `WHERE` runs before the window function has computed anything.

✅ **Fix — compute the rank in a CTE, then filter the CTE's result**
```sql
WITH ranked_employees AS (
    SELECT name, salary,
           RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT * FROM ranked_employees WHERE salary_rank <= 3;
```

**Highest-paid employee *per department*** — a hugely common real-world pattern:
```sql
WITH ranked_employees AS (
    SELECT name, department_id, salary,
           RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT name, department_id, salary
FROM ranked_employees
WHERE salary_rank = 1;
```
**Output**
```
name   | department_id | salary
Arjun  | 101           | 80000
Simran | 102           | 45000
Karan  | 103           | 60000
```

💡 **Memorize this pattern** — `WITH ranked AS (SELECT ..., RANK() OVER (...) AS rnk FROM ...) SELECT * FROM ranked WHERE rnk = 1;` — it comes up constantly in interviews ("top N per group").

---

## 9. Multiple CTEs

**Definition** — More than one named temporary result, defined in the same `WITH` clause, separated by commas — `WITH` is written **once**.

**Syntax**
```sql
WITH cte1 AS (SELECT ...),
     cte2 AS (SELECT ...)
SELECT ... FROM cte1 JOIN cte2 ON ...;
```

❌ **Common Mistakes**
```sql
-- ❌ WITH written twice
WITH cte1 AS (...)
WITH cte2 AS (...)
SELECT ...;
```
```sql
-- ✅ One WITH, comma-separated definitions
WITH cte1 AS (...),
     cte2 AS (...)
SELECT ...;
```

**Example — "employees earning more than their department average, but only from departments with more than 2 employees"**
```sql
WITH department_averages AS (
    SELECT department_id, AVG(salary) AS average_salary
    FROM employees GROUP BY department_id
),
department_counts AS (
    SELECT department_id, COUNT(*) AS employee_count
    FROM employees GROUP BY department_id
)
SELECT e.name, e.salary, da.average_salary, dc.employee_count
FROM employees e
JOIN department_averages da ON e.department_id = da.department_id
JOIN department_counts dc   ON e.department_id = dc.department_id
WHERE e.salary > da.average_salary
  AND dc.employee_count > 2;
```
**Output**
```
name  | salary | average_salary | employee_count
Priya | 70000  | 66666.67       | 3
Arjun | 80000  | 66666.67       | 3
```

---

## 10. One CTE Referencing Another CTE

**Definition** — A later CTE can select `FROM` an earlier CTE, chaining them into a pipeline.

**Syntax**
```sql
WITH cte1 AS (SELECT ...),
     cte2 AS (SELECT * FROM cte1 WHERE ...)
SELECT * FROM cte2;
```

**Example**
```sql
WITH department_averages AS (
    SELECT department_id, AVG(salary) AS average_salary
    FROM employees GROUP BY department_id
),
high_average_departments AS (
    SELECT department_id, average_salary
    FROM department_averages
    WHERE average_salary > 50000
)
SELECT * FROM high_average_departments;
```

⚠️ **Notes & Caveats** — A CTE must be defined **before** any later CTE (or the main query) references it — you can't reference a CTE that hasn't been declared yet in the same `WITH` list.

---

## 11. Recursive CTE — The Three-Question Framework

**Definition** — A CTE that refers to **itself**, repeatedly generating more rows until a stop condition is met — used for sequences and hierarchical/tree-shaped data of unknown depth.

**Syntax**
```sql
WITH RECURSIVE cte_name AS (
    -- 1. ANCHOR: where do I start?
    SELECT ...

    UNION ALL

    -- 2. RECURSIVE PART: how do I find the next row?
    SELECT ...
    FROM cte_name        -- ⭐ refers to itself
    WHERE ...             -- 3. STOP CONDITION: when do I stop?
)
SELECT * FROM cte_name;
```

💡 **Always answer exactly three questions before writing one:**
```
1. ANCHOR            → Where does recursion start?
2. RECURSIVE PART     → How do I compute the next row from the previous one?
3. STOP CONDITION      → What makes recursion eventually produce zero new rows?
```

⚠️ **Notes & Caveats** — PostgreSQL requires the keyword `RECURSIVE` (`WITH RECURSIVE ...`) — a plain `WITH` won't allow the CTE to reference itself.

---

## 12. Recursive CTE Example: Number Series

**Example — generate 1 through 5**
```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS number                -- ANCHOR: start at 1

    UNION ALL

    SELECT number + 1                  -- RECURSIVE: next = previous + 1
    FROM numbers
    WHERE number < 5                    -- STOP: quit once number reaches 5
)
SELECT * FROM numbers;
```
**Output**
```
number
1
2
3
4
5
```

**Execution trace**
```
Anchor:        1
Recursive: 1<5 → 2
Recursive: 2<5 → 3
Recursive: 3<5 → 4
Recursive: 4<5 → 5
Recursive: 5<5 → FALSE → STOP (0 new rows)
```

❌ **Common Mistakes**
```sql
-- ❌ Missing WHERE — no stop condition, recursion never logically ends
WITH RECURSIVE numbers AS (
    SELECT 1 AS number
    UNION ALL
    SELECT number + 1 FROM numbers
)
SELECT * FROM numbers;
```
PostgreSQL may eventually error out or hit resource limits, but don't rely on that — always design an explicit stop condition.

---

## 13. Recursive CTE Example: Employee Hierarchy

**Table**
```
employees: emp_id | name   | manager_id
           1       | Aman   | NULL
           2       | Ravi   | 1
           3       | Priya  | 1
           4       | Karan  | 2
           5       | Simran | 2
           6       | Arjun  | 3
```
```
Aman
├── Ravi
│   ├── Karan
│   └── Simran
└── Priya
    └── Arjun
```

**Query**
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- ANCHOR: top-level manager(s) — no manager_id
    SELECT emp_id, name, manager_id, 1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- RECURSIVE: find employees reporting to anyone already in the hierarchy
    SELECT e.emp_id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.emp_id
    -- STOP: implicit — recursion ends once a level finds zero matching rows
)
SELECT * FROM employee_hierarchy;
```
**Output**
```
emp_id | name   | manager_id | level
1      | Aman   | NULL       | 1
2      | Ravi   | 1          | 2
3      | Priya  | 1          | 2
4      | Karan  | 2          | 3
5      | Simran | 2          | 3
6      | Arjun  | 3          | 3
```

⚠️ **Notes & Caveats** — Unlike the number-series example, there's **no explicit `WHERE`** stop condition here — recursion naturally stops once the recursive `JOIN` finds zero new matching rows (e.g., nobody reports to Karan, Simran, or Arjun).

**Finding everyone under one specific manager, excluding themselves**
```sql
WITH RECURSIVE employee_hierarchy AS (
    SELECT emp_id, name, manager_id FROM employees WHERE emp_id = 2   -- start at Ravi
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.emp_id
)
SELECT * FROM employee_hierarchy WHERE emp_id <> 2;    -- drop Ravi himself
```

💡 **How to Choose** — Recursive CTEs apply to any parent → child structure of unknown depth: org charts, category/subcategory trees, folder structures, comment threads, bill-of-materials.

---

## 14. Recursive CTE vs Self Join

| | Self Join (Part 13) | Recursive CTE |
|---|---|---|
| Levels handled | One fixed level (e.g., immediate manager only) | Unlimited/unknown depth |
| Example | "Employee's direct manager" | "Employee's entire management chain" |

💡 If you know exactly how many levels deep you need, a `SELF JOIN` (or a few chained ones) is simpler. If the depth is unknown or variable, use a recursive CTE.

---

## 15. Common Mistakes

❌ **Reusing a CTE across separate statements**
```sql
WITH high_salary AS (SELECT * FROM employees WHERE salary > 50000)
SELECT * FROM high_salary;
-- (new statement) SELECT * FROM high_salary;   -- ❌ fails, CTE is gone
```

❌ **Forgetting the comma between multiple CTEs**
```sql
WITH cte1 AS (...)
cte2 AS (...)          -- ❌ missing comma before cte2
SELECT ...;
```

❌ **Writing `WITH` more than once**
```sql
WITH cte1 AS (...)
WITH cte2 AS (...)      -- ❌ invalid
SELECT ...;
```

❌ **Referencing a later CTE from an earlier one**
```sql
WITH cte2 AS (SELECT * FROM cte1),   -- ❌ cte1 isn't defined yet at this point
     cte1 AS (SELECT ...)
SELECT * FROM cte2;
```

❌ **No stop condition in a recursive CTE** — always define an explicit `WHERE` (or rely on the recursive `JOIN` naturally finding zero rows, as in the hierarchy example).

---

## 16. Interview Q&A

**Q: What is a CTE?**
A: A temporary, named result set defined with `WITH`, available only to the single SQL statement immediately following it.

**Q: Is a CTE the same as a temporary table?**
A: No — a CTE's lifetime is a single statement and it isn't necessarily materialized as a physical table; a temp table is an actual database object that persists across multiple statements within the session.

**Q: Is a CTE always faster than an equivalent subquery?**
A: Not necessarily — its main benefit is readability and the ability to break a complex query into named logical steps, not guaranteed performance gains.

**Q: Why would you compute a `RANK()` window function inside a CTE instead of directly filtering it?**
A: Because `WHERE` runs before the `SELECT` list's window function alias exists — you can't filter on it in the same query level. Wrapping the ranking in a CTE and filtering the CTE's result in the outer query solves this.

**Q: How do you define multiple CTEs in one query?**
A: Write `WITH` once, then separate each `cte_name AS (...)` definition with a comma; a later CTE may reference an earlier one, but not vice versa.

**Q: What are the three components you must define in a recursive CTE?**
A: An anchor query (starting rows), a recursive query (how to generate the next rows from the previous result, referencing the CTE itself), and a stop condition (what eventually makes the recursive part return zero new rows).

**Q: Why must you write `WITH RECURSIVE` instead of just `WITH` for a self-referencing CTE?**
A: PostgreSQL requires the `RECURSIVE` keyword to allow a CTE definition to reference itself; a plain `WITH` doesn't permit that self-reference.

**Q: When would you choose a recursive CTE over a self join?**
A: When the hierarchy depth is unknown or variable (e.g., an org chart of arbitrary depth) — a self join only handles one fixed level per join, while a recursive CTE walks arbitrarily many levels until no more matches are found.

---

## 17. Quick Revision Sheet

| Goal | Pattern |
|---|---|
| Name a query result | `WITH name AS (SELECT ...) SELECT ... FROM name;` |
| Multiple named steps | `WITH a AS (...), b AS (...) SELECT ...;` |
| Later CTE uses earlier one | `WITH a AS (...), b AS (SELECT * FROM a ...) ...` |
| Filter a window function result | `WITH ranked AS (..., RANK() OVER (...) AS r FROM ...) SELECT * FROM ranked WHERE r <= n;` |
| Top-1-per-group | `PARTITION BY group_col ORDER BY metric DESC` + `WHERE rank = 1` |
| Recursive structure | `WITH RECURSIVE name AS (anchor UNION ALL recursive-referencing-name) SELECT * FROM name;` |

---

## 18. Cheat Sheet

```sql
-- ── BASIC CTE ─────────────────────────────
WITH high_salary_employees AS (
    SELECT name, salary FROM employees WHERE salary > 50000
)
SELECT * FROM high_salary_employees;

-- ── CTE + JOIN ────────────────────────────
WITH department_averages AS (
    SELECT department_id, AVG(salary) AS average_salary
    FROM employees GROUP BY department_id
)
SELECT e.name, e.salary, da.average_salary
FROM employees e
JOIN department_averages da ON e.department_id = da.department_id
WHERE e.salary > da.average_salary;

-- ── CTE + WINDOW FUNCTION (top N per group) ──
WITH ranked_employees AS (
    SELECT name, department_id, salary,
           RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT name, department_id, salary
FROM ranked_employees
WHERE salary_rank = 1;

-- ── MULTIPLE CTEs ─────────────────────────
WITH department_averages AS (
    SELECT department_id, AVG(salary) AS average_salary FROM employees GROUP BY department_id
),
department_counts AS (
    SELECT department_id, COUNT(*) AS employee_count FROM employees GROUP BY department_id
)
SELECT e.name, da.average_salary, dc.employee_count
FROM employees e
JOIN department_averages da ON e.department_id = da.department_id
JOIN department_counts dc   ON e.department_id = dc.department_id
WHERE e.salary > da.average_salary AND dc.employee_count > 2;

-- ── RECURSIVE CTE: number series ──────────
WITH RECURSIVE numbers AS (
    SELECT 1 AS number
    UNION ALL
    SELECT number + 1 FROM numbers WHERE number < 5
)
SELECT * FROM numbers;

-- ── RECURSIVE CTE: hierarchy ──────────────
WITH RECURSIVE employee_hierarchy AS (
    SELECT emp_id, name, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.emp_id
)
SELECT * FROM employee_hierarchy;
```

---

## 19. Preview of Part 17

| Topic | What You'll Learn |
|---|---|
| `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE()` | Ranking window functions |
| `LEAD()`, `LAG()` | Looking at neighboring rows |
| `FIRST_VALUE()`, `LAST_VALUE()`, `NTH_VALUE()` | Value window functions |
| `OVER()`, `PARTITION BY`, window `ORDER BY` | Full window function mechanics |
