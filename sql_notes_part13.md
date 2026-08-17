# SQL & PostgreSQL Complete Notes — Part 13: JOINs

## 📑 Table of Contents
1. What is a JOIN?
2. `INNER JOIN`
3. `LEFT JOIN`
4. `RIGHT JOIN`
5. `FULL OUTER JOIN`
6. `INNER` vs `LEFT` vs `RIGHT` vs `FULL` (Master Comparison)
7. `CROSS JOIN`
8. `SELF JOIN`
9. `ON` vs `USING` vs `NATURAL JOIN`
10. Non-Equi JOIN
11. JOIN + `GROUP BY` + `HAVING`
12. Row Multiplication in JOINs
13. The "Find Unmatched Rows" Pattern
14. How to Decide Which JOIN to Use
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Preview of Part 14

**📋 Series Coverage (Part 13):** `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `FULL OUTER JOIN`, `CROSS JOIN`, `SELF JOIN`, `ON`, `USING`, `NATURAL JOIN`, non-equi joins, join row multiplication, `LEFT JOIN ... WHERE right.id IS NULL` anti-join pattern

> Examples use:
> ```
> employees:   emp_id | name | department_id | salary
>              1 | Aman   | 101  | 50000
>              2 | Ravi   | 102  | 40000
>              3 | Priya  | 101  | 70000
>              4 | Karan  | 103  | 60000
>              5 | Simran | NULL | 45000
>
> departments: department_id | department_name
>              101 | IT
>              102 | HR
>              103 | Sales
>              104 | Finance   ← no employees
> ```

---

## 1. What is a JOIN?

**Definition** — Combines related rows from two or more tables based on a matching condition, usually a foreign-key-to-primary-key relationship.

**Why It Exists** — Normalized data (Part 1) is deliberately split across tables — a `JOIN` is how you bring related pieces back together for a query.

**General Syntax**
```sql
SELECT columns
FROM table1
JOIN table2 ON table1.column = table2.column;
```

---

## 2. `INNER JOIN`

**Definition** — Returns only rows that have a match in **both** tables. `JOIN` alone (no keyword before it) defaults to `INNER JOIN`.

```
employees                    departments
┌────────┬─────┐             ┌─────┬──────────┐
│ Aman   │ 101 │────match───▶│ 101 │ IT       │
│ Ravi   │ 102 │────match───▶│ 102 │ HR       │
│ Priya  │ 101 │────match───▶│ 101 │ IT       │
│ Karan  │ 103 │────match───▶│ 103 │ Sales    │
│ Simran │ NULL│    ✗ no match     │ 104 │ Finance │ ✗ no match
└────────┴─────┘             └─────┴──────────┘
```

**Syntax**
```sql
SELECT e.name, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;
```
**Output**
```
name  | department_name
Aman  | IT
Ravi  | HR
Priya | IT
Karan | Sales
```
*(Simran is dropped — no department_id to match. Finance is dropped — no employee.)*

⚠️ **Notes & Caveats** — `JOIN` and `INNER JOIN` are exactly the same in PostgreSQL.

💡 **How to Choose** — Use when you only want rows that have a valid relationship on both sides (e.g., "show employees **with** a valid department").

---

## 3. `LEFT JOIN`

**Definition** — Returns **all** rows from the left (first-listed) table, plus matching rows from the right table. Unmatched right-side columns become `NULL`.

```
LEFT TABLE (protected)         RIGHT TABLE
employees                       departments
Aman   →101 ────────────────▶  101 IT
Ravi   →102 ────────────────▶  102 HR
Priya  →101 ────────────────▶  101 IT
Karan  →103 ────────────────▶  103 Sales
Simran →NULL ───────────────▶  (no match) → NULL
                                104 Finance  ✗ dropped (not in left table)
```

**Syntax**
```sql
SELECT e.name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```
**Output**
```
name   | department_name
Aman   | IT
Ravi   | HR
Priya  | IT
Karan  | Sales
Simran | NULL
```

⚠️ **Notes & Caveats** — **Table order matters.** `FROM employees LEFT JOIN departments` keeps all employees; `FROM departments LEFT JOIN employees` keeps all departments instead.

💡 **How to Choose** — Use when you want every row from one table regardless of whether a related row exists — e.g., "show all customers, **including** those who never placed an order."

---

## 4. `RIGHT JOIN`

**Definition** — Returns **all** rows from the right table, plus matching rows from the left. The mirror image of `LEFT JOIN`.

```sql
SELECT e.name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```
**Output**
```
name  | department_name
Aman  | IT
Priya | IT
Ravi  | HR
Karan | Sales
NULL  | Finance      ← kept, because departments is the "right"/protected side
```

💡 **Best Practices / How to Choose** — Any `RIGHT JOIN` can be rewritten as a `LEFT JOIN` by swapping table order — most developers prefer always using `LEFT JOIN` and simply putting the table they want to protect first, since it reads more naturally.

---

## 5. `FULL OUTER JOIN`

**Definition** — Returns **all** rows from both tables — matched rows combined, unmatched rows from either side padded with `NULL`.

```sql
SELECT e.name, d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.department_id;
```
**Output**
```
name   | department_name
Aman   | IT
Priya  | IT
Ravi   | HR
Karan  | Sales
Simran | NULL          ← unmatched employee, kept
NULL   | Finance        ← unmatched department, kept
```

---

## 6. `INNER` vs `LEFT` vs `RIGHT` vs `FULL` (Master Comparison)

```
Table A = {1, 2, 3}        Table B = {2, 3, 4}

INNER JOIN → {2, 3}                 (only matches)
LEFT  JOIN → {1, 2, 3}              (all of A + matches)
RIGHT JOIN → {2, 3, 4}              (all of B + matches)
FULL  JOIN → {1, 2, 3, 4}           (everything)
```

| Join | Matched Rows | Unmatched Left | Unmatched Right |
|---|---|---|---|
| `INNER JOIN` | ✅ | ❌ | ❌ |
| `LEFT JOIN` | ✅ | ✅ | ❌ |
| `RIGHT JOIN` | ✅ | ❌ | ✅ |
| `FULL OUTER JOIN` | ✅ | ✅ | ✅ |

*(This is a simplified set-style mental model — real joins combine whole rows, not bare values, but it's the fastest way to remember which side survives.)*

---

## 7. `CROSS JOIN`

**Definition** — Combines **every** row of the first table with **every** row of the second — a Cartesian product. Typically written without an `ON` clause.

**Syntax**
```sql
SELECT e.name, p.project_name
FROM employees e
CROSS JOIN projects p;
```

**Row count formula**
```
Result rows = (rows in table A) × (rows in table B)
2 employees × 3 projects = 6 result rows
```

💡 **How to Choose** — Rare in everyday querying; useful for generating every possible combination (e.g., every size × every color for a product catalog).

---

## 8. `SELF JOIN`

**Definition** — Joins a table to **itself**, using two aliases to distinguish the two "roles" a row can play (classic case: employee ↔ manager, both stored in the same `employees` table).

⚠️ **Notes & Caveats** — There is no dedicated `SELF JOIN` keyword — it's just a normal `JOIN`/`LEFT JOIN` where both sides happen to be the same table, given two different aliases.

**Example**
```sql
CREATE TABLE employees (
    emp_id     SERIAL PRIMARY KEY,
    name       VARCHAR(100),
    manager_id INT
);
-- emp_id 2 (Ravi) has manager_id = 1 (Aman)

SELECT employee.name AS employee_name,
       manager.name  AS manager_name
FROM employees AS employee
LEFT JOIN employees AS manager
       ON employee.manager_id = manager.emp_id;
```
**Output**
```
employee_name | manager_name
Aman          | NULL
Ravi          | Aman
Priya         | Aman
Karan         | Ravi
```

⚠️ **Notes & Caveats** — Aliases (`employee`, `manager`) are **mandatory** here — without them, `employees.name` is ambiguous, since both "copies" being joined are the same table.

💡 **How to Choose** — Use `LEFT JOIN` (not plain `JOIN`) for self-joins involving hierarchies, so top-level rows (e.g., the CEO with no manager) aren't accidentally dropped.

---

## 9. `ON` vs `USING` vs `NATURAL JOIN`

| Syntax | When to Use |
|---|---|
| `ON a.col = b.col` | Always works; required when column names differ or the condition is complex |
| `USING (col)` | Shorthand when both tables use the **exact same column name** |
| `NATURAL JOIN` | Auto-matches **all** same-named columns — convenient but risky |

```sql
-- Equivalent forms when both sides use "department_id":
SELECT e.name, d.department_name
FROM employees e JOIN departments d ON e.department_id = d.department_id;

SELECT e.name, d.department_name
FROM employees e JOIN departments d USING (department_id);
```

⚠️ **Notes & Caveats** — `NATURAL JOIN` silently matches on **every** identically-named column (e.g., it might accidentally also match on a shared `created_at` column), which can change the query's meaning without any error. It's generally avoided in production SQL for exactly this reason.

💡 **Best Practices / How to Choose** — Default to `ON` — it's explicit and never surprising. Reach for `USING` only as a minor shorthand when the column names truly match; avoid `NATURAL JOIN` in real applications.

---

## 10. Non-Equi JOIN

**Definition** — A join whose condition uses something other than `=` (e.g., `BETWEEN`, `<`, `>`).

**Example** — matching salaries to salary grade bands:
```sql
SELECT e.name, e.salary, sg.grade
FROM employees e
JOIN salary_grades sg
  ON e.salary BETWEEN sg.min_salary AND sg.max_salary;
```

---

## 11. JOIN + `GROUP BY` + `HAVING`

```sql
SELECT c.customer_name, SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING SUM(o.amount) > 5000;
```
**Mental flow:** `JOIN` (connect tables) → `GROUP BY` (create per-customer groups) → `SUM()` (total per group) → `HAVING` (keep only big spenders).

---

## 12. Row Multiplication in JOINs

⚠️ **Notes & Caveats** — A `JOIN` produces **one output row per matching combination** — if a customer has 3 orders, joining `customers` to `orders` produces 3 rows for that customer, not 1.

```sql
SELECT c.name, o.order_id
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;
```
```
name | order_id
Aman | 101
Aman | 102
Aman | 103
```

**Danger with aggregates after multiple joins** — if `Aman` has 1 salary row and 2 project rows, joining both at once produces 2 combined rows (each with the *same* salary duplicated), so `SUM(salary)` after that join **double-counts** it. Always inspect the joined row set (row count, duplicated values) before applying `SUM`/`COUNT`/`AVG` on top of a multi-table join.

---

## 13. The "Find Unmatched Rows" Pattern

**Definition** — The standard way to answer "which rows in A have **no** related row in B."

**Syntax**
```sql
SELECT a.*
FROM a
LEFT JOIN b ON a.id = b.a_id
WHERE b.id IS NULL;
```

**Example — customers who never ordered**
```sql
SELECT c.customer_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

💡 This pattern — `LEFT JOIN` + `WHERE right_table.column IS NULL` — is one of the most common real-world and interview SQL patterns. Memorize it.

---

## 14. How to Decide Which JOIN to Use

```
Only rows matching on BOTH sides?          → INNER JOIN
Every row from the first table, matches optional? → LEFT JOIN
Every row from the second table, matches optional? → RIGHT JOIN (or flip to LEFT JOIN)
Every row from BOTH tables?                → FULL OUTER JOIN
Every possible combination?                → CROSS JOIN
Connecting rows within the SAME table?     → SELF JOIN
```

---

## 15. Interview Q&A

**Q: What's the default join type when you write just `JOIN`?**
A: `INNER JOIN` — `JOIN` alone always means `INNER JOIN` in PostgreSQL.

**Q: How would you find all customers who have never placed an order?**
A: `LEFT JOIN` customers to orders, then filter with `WHERE orders.order_id IS NULL` — this is the standard "unmatched rows" pattern.

**Q: Why does table order matter with `LEFT JOIN` but not with `INNER JOIN`?**
A: `LEFT JOIN` explicitly protects/preserves every row from whichever table is listed first (the "left" table); swapping the tables changes which side's unmatched rows survive. `INNER JOIN` only keeps matches, so which table is "first" doesn't affect the result set.

**Q: What is a `CROSS JOIN`, and how many rows does it produce?**
A: It pairs every row of one table with every row of another (a Cartesian product) — the result has `(rows in A) × (rows in B)` rows.

**Q: What makes a `SELF JOIN` different syntactically from a normal join?**
A: Nothing special syntax-wise — it's a normal `JOIN`/`LEFT JOIN` where both sides reference the same table, distinguished with two different aliases (since otherwise the column references would be ambiguous).

**Q: Why is `NATURAL JOIN` generally discouraged?**
A: It automatically joins on every identically-named column across both tables, which can silently change the query's meaning (and results) if a new same-named column is added later — `ON`/`USING` make the join condition explicit instead.

**Q: A customer has 3 orders and you join `customers` to `orders` — how many result rows appear for that customer?**
A: 3 — one result row is produced per matching row combination, so a single customer joined against 3 matching orders yields 3 rows in the output.

**Q: What's a non-equi join?**
A: A join whose `ON` condition uses something other than `=`, such as `BETWEEN`, `<`, or `>` — commonly used to match a value into a range, e.g., mapping salaries into salary grade bands.

---

## 16. Quick Revision Sheet

| Goal | Join |
|---|---|
| Only matches | `INNER JOIN` |
| All of left + matches | `LEFT JOIN` |
| All of right + matches | `RIGHT JOIN` |
| All of both | `FULL OUTER JOIN` |
| Every combination | `CROSS JOIN` |
| Table joined to itself | `SELF JOIN` (alias twice) |
| Find unmatched rows | `LEFT JOIN ... WHERE right.col IS NULL` |
| Shorthand for same-named join column | `USING (col)` |
| Explicit join condition | `ON a.col = b.col` |

---

## 17. Cheat Sheet

```sql
-- ── INNER JOIN ────────────────────────────
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- ── LEFT / RIGHT / FULL ───────────────────
SELECT e.name, d.department_name
FROM employees e LEFT JOIN departments d ON e.department_id = d.department_id;

SELECT e.name, d.department_name
FROM employees e RIGHT JOIN departments d ON e.department_id = d.department_id;

SELECT e.name, d.department_name
FROM employees e FULL OUTER JOIN departments d ON e.department_id = d.department_id;

-- ── CROSS JOIN ────────────────────────────
SELECT e.name, p.project_name FROM employees e CROSS JOIN projects p;

-- ── SELF JOIN ─────────────────────────────
SELECT emp.name AS employee, mgr.name AS manager
FROM employees emp
LEFT JOIN employees mgr ON emp.manager_id = mgr.emp_id;

-- ── ON vs USING ───────────────────────────
... JOIN departments d ON e.department_id = d.department_id;
... JOIN departments d USING (department_id);

-- ── FIND UNMATCHED ROWS ───────────────────
SELECT c.customer_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- ── JOIN + GROUP BY + HAVING ──────────────
SELECT c.customer_name, SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING SUM(o.amount) > 5000;
```

---

## 18. Preview of Part 14

| Topic | What You'll Learn |
|---|---|
| Joining 3, 4, 5 tables | Building multi-table queries step by step |
| Bridge tables | Joining through a table just to reach another one |
| Row multiplication at scale | Spotting and avoiding duplicate-aggregate bugs |
