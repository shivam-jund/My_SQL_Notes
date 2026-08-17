# SQL & PostgreSQL Complete Notes — Part 15: Subqueries

## 📑 Table of Contents
1. What Is a Subquery?
2. Single-Row Subquery
3. Multiple-Row Subquery
4. Scalar Subquery
5. Correlated Subquery
6. Nested Subquery
7. `ANY`
8. `ALL`
9. `EXISTS`
10. `NOT EXISTS`
11. `IN` vs `EXISTS` (and the `NOT IN` + `NULL` Trap)
12. Where Subqueries Can Live: `WHERE`, `SELECT`, `FROM`, `HAVING`
13. Interview Q&A
14. Quick Revision Sheet
15. Cheat Sheet
16. Preview of Part 16

**📋 Series Coverage (Part 15):** single-row, multiple-row, scalar, correlated, and nested subqueries; `ANY`, `ALL`, `EXISTS`, `NOT EXISTS`; `IN` vs `EXISTS`; the `NOT IN`/`NULL` trap; subqueries in `WHERE`, `SELECT`, `FROM` (derived tables), and `HAVING`

> Examples use:
> ```
> employees:   emp_id | name | department_id | salary
>              1 | Aman  | 101 | 50000     4 | Karan | 103 | 60000
>              2 | Ravi  | 102 | 40000     5 | Simran| 102 | 45000
>              3 | Priya | 101 | 70000     6 | Arjun | 101 | 80000
> departments: department_id | department_name | 101 IT, 102 HR, 103 Sales, 104 Finance
> projects:    project_id | project_name | emp_id
> ```

---

## 1. What Is a Subquery?

**Definition** — A query written inside another query. The inner query (**subquery**) runs first (conceptually) and its result feeds the outer query.

**Why It Exists** — Hardcoding a computed value (like "the average salary is 57500") breaks the moment underlying data changes. A subquery keeps the comparison dynamic and always current.

**Syntax**
```sql
SELECT columns
FROM table
WHERE column operator (subquery);
```

**Example**
```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```
**Output**
```
name  | salary
Priya | 70000
Karan | 60000
Arjun | 80000
```

---

## 2. Single-Row Subquery

**Definition** — A subquery that returns exactly **one row** (with one or more columns). Can be compared with `=`, `>`, `<`, `>=`, `<=`, `!=`.

**Example — highest-paid employee**
```sql
SELECT name, salary
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);
```

⚠️ **Notes & Caveats** — "Single-row" describes the **subquery's** result, not the outer query's. If two employees tie for the max salary, the outer query can still return two rows.

---

## 3. Multiple-Row Subquery

**Definition** — A subquery returning more than one row. Must be used with `IN`, `ANY`, or `ALL` — never bare `=`.

**Example**
```sql
SELECT name
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE department_name IN ('IT', 'HR')
);
```

❌ **Common Mistakes**
```sql
-- ❌ Subquery returns multiple rows; = expects exactly one
WHERE department_id = (
    SELECT department_id FROM departments WHERE department_name IN ('IT', 'HR')
);
-- ERROR: more than one row returned by a subquery used as an expression
```
```sql
-- ✅ Use IN for a multi-row result
WHERE department_id IN (SELECT department_id FROM departments WHERE department_name IN ('IT', 'HR'));
```

---

## 4. Scalar Subquery

**Definition** — A subquery returning **exactly one row and one column** — a single value. Because it's a single value, it can be used directly inside a `SELECT` list, not just in `WHERE`.

**Example**
```sql
SELECT name, salary,
       (SELECT AVG(salary) FROM employees) AS company_average
FROM employees;
```
**Output**
```
name  | salary | company_average
Aman  | 50000  | 57500
Ravi  | 40000  | 57500
...
```

⚠️ **Notes & Caveats** — Every row gets the **same** `company_average` value, because the subquery here doesn't reference anything from the outer row (it's not correlated — see next section).

---

## 5. Correlated Subquery

**Definition** — A subquery that references a column from the **outer query**, so it conceptually re-evaluates once per outer row.

**Why It Exists** — Some comparisons genuinely depend on "which row am I currently looking at" — e.g., comparing an employee's salary to *their own department's* average, not the whole company's.

**Example**
```sql
SELECT e1.name, e1.salary
FROM employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e1.department_id   -- ⭐ references outer alias e1
);
```
**Output**
```
name   | salary
Priya  | 70000
Simran | 45000
Arjun  | 80000
```

**How to identify one** — Look inside the subquery: if it references an alias defined by the **outer** query (here, `e1`), it's correlated.

| | Normal Subquery | Correlated Subquery |
|---|---|---|
| Depends on outer row? | ❌ No — computed once | ✅ Yes — conceptually once per outer row |
| Example question | "What's the company average?" | "What's *this employee's* department average?" |

💡 **How to Choose** — If the question contains phrasing like "compared to **their own** X" or "within **the same** group as this row," you almost certainly need a correlated subquery.

---

## 6. Nested Subquery

**Definition** — A subquery inside another subquery (multiple levels of nesting).

**Example**
```sql
SELECT name
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE location_id = (
        SELECT location_id FROM departments WHERE department_name = 'IT'
    )
);
```

⚠️ **Notes & Caveats** — Always evaluate nested subqueries **from the innermost outward**: find the IT department's `location_id` first, then departments at that location, then employees in those departments.

---

## 7. `ANY`

**Definition** — A condition is `TRUE` if it holds for **at least one** value the subquery returns.

**Example**
```sql
SELECT name, salary
FROM employees
WHERE salary > ANY (
    SELECT salary FROM employees WHERE department_id = 102  -- HR
);
```

💡 For ordinary, non-empty, non-NULL value sets: `> ANY` behaves like `> MIN(list)`; `< ANY` behaves like `< MAX(list)`.

---

## 8. `ALL`

**Definition** — A condition is `TRUE` only if it holds for **every** value the subquery returns.

**Example**
```sql
SELECT name, salary
FROM employees
WHERE salary > ALL (
    SELECT salary FROM employees WHERE department_id = 102  -- HR
);
```

💡 For ordinary, non-empty, non-NULL value sets: `> ALL` behaves like `> MAX(list)`; `< ALL` behaves like `< MIN(list)`.

**`ANY` vs `ALL`**

| | `ANY` | `ALL` |
|---|---|---|
| Passes when | At least one comparison is `TRUE` | Every comparison is `TRUE` |
| Roughly equivalent to | comparison against the extreme value that's *easiest* to satisfy | comparison against the extreme value that's *hardest* to satisfy |

---

## 9. `EXISTS`

**Definition** — `TRUE` if the subquery returns **at least one row** — the actual selected values don't matter (hence the common convention `SELECT 1`).

**Example — employees who have at least one project**
```sql
SELECT e.name
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM projects p WHERE p.emp_id = e.emp_id
);
```

⚠️ **Notes & Caveats** — This is also a **correlated** subquery (`p.emp_id = e.emp_id` references the outer row). `EXISTS` only asks "did any row come back?" — not "what value did it return?"

---

## 10. `NOT EXISTS`

**Definition** — `TRUE` if the subquery returns **zero rows**.

**Example — employees with no project**
```sql
SELECT e.name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM projects p WHERE p.emp_id = e.emp_id
);
```

---

## 11. `IN` vs `EXISTS` (and the `NOT IN` + `NULL` Trap)

| | `IN` | `EXISTS` |
|---|---|---|
| Asks | "Is this value inside the returned set?" | "Does at least one matching row exist?" |
| Typical use | Compare a single column against a list | Correlated existence check |

⚠️ **Notes & Caveats — the classic `NOT IN` trap:**
```sql
-- If the subquery's result contains even ONE NULL:
SELECT e.name
FROM employees e
WHERE e.emp_id NOT IN (SELECT emp_id FROM projects);   -- ⚠️ dangerous if emp_id can be NULL
```
Because SQL uses three-valued logic, comparing anything to a `NULL` inside the list produces `UNKNOWN`, not `TRUE`/`FALSE` — which can silently make `NOT IN` return **zero rows for every employee**, even ones that obviously have no project.

💡 **Best Practices** — For "find rows with **no** related record" queries, prefer `NOT EXISTS` over `NOT IN` — it isn't vulnerable to this `NULL` trap, since it's checking row existence, not comparing against a value list.
```sql
-- ✅ Safe regardless of NULLs in projects.emp_id
SELECT e.name
FROM employees e
WHERE NOT EXISTS (SELECT 1 FROM projects p WHERE p.emp_id = e.emp_id);
```

---

## 12. Where Subqueries Can Live: `WHERE`, `SELECT`, `FROM`, `HAVING`

**In `WHERE`** (most common)
```sql
SELECT name FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
```

**In `SELECT`** (must be scalar — one row, one column)
```sql
SELECT name, salary,
       (SELECT AVG(salary) FROM employees) AS company_average
FROM employees;
```

**In `FROM`** (a "derived table" — an unnamed temporary result queried like a table)
```sql
SELECT department_id, average_salary
FROM (
    SELECT department_id, AVG(salary) AS average_salary
    FROM employees
    GROUP BY department_id
) AS department_averages
WHERE average_salary > 50000;
```
💡 This exact pattern gets **much more readable** using a CTE — covered in Part 16, right after this one.

**In `HAVING`**
```sql
SELECT department_id, AVG(salary) AS department_average
FROM employees
GROUP BY department_id
HAVING AVG(salary) > (SELECT AVG(salary) FROM employees);
```
*(department average vs. overall company average)*

---

## 13. Interview Q&A

**Q: What's the difference between a single-row and a multiple-row subquery?**
A: A single-row subquery returns exactly one row and can be compared with `=`, `>`, `<`, etc. A multiple-row subquery returns several rows and must be used with `IN`, `ANY`, or `ALL` instead.

**Q: What makes a subquery "scalar"?**
A: It returns exactly one row **and** one column — a single value — which is why it can be embedded directly inside a `SELECT` list as a computed column.

**Q: What makes a subquery "correlated"?**
A: It references a column from the outer query's alias, so it conceptually depends on and re-evaluates for each outer row — as opposed to a normal subquery, which is computed independently once.

**Q: Give an example question that requires a correlated subquery rather than a normal one.**
A: "Find employees earning more than their **own department's** average salary" — the average being compared against changes depending on which employee (and therefore which department) is currently being evaluated.

**Q: `ANY` vs `ALL`?**
A: `ANY` passes if the condition holds for at least one value in the subquery's result; `ALL` requires it to hold for every value.

**Q: What does `EXISTS` actually check?**
A: Whether the subquery returns at least one row — not what value that row contains. This is why `SELECT 1` is a common convention inside `EXISTS` subqueries.

**Q: Why is `NOT EXISTS` often preferred over `NOT IN` for "find unmatched rows" queries?**
A: If the `NOT IN` subquery's result contains a `NULL`, three-valued logic can cause the entire condition to evaluate to `UNKNOWN` for every row, silently returning zero results. `NOT EXISTS` checks row existence rather than comparing values, so it isn't affected by `NULL`s in the same way.

**Q: Can a subquery appear inside `FROM`?**
A: Yes — this is called a derived table; the subquery's result is treated like a temporary, unnamed table that the outer query can select from and filter.

---

## 14. Quick Revision Sheet

| Need | Use |
|---|---|
| Compare to one computed value | Single-row subquery (`=`, `>`, etc.) |
| Compare to a list of values | `IN` / `ANY` / `ALL` |
| Embed a value as a column | Scalar subquery in `SELECT` |
| Depends on the current outer row | Correlated subquery |
| Query inside a query inside a query | Nested subquery (innermost evaluates first) |
| At least one comparison true | `ANY` |
| Every comparison true | `ALL` |
| Row existence check | `EXISTS` |
| Row non-existence check (NULL-safe) | `NOT EXISTS` (prefer over `NOT IN`) |
| Query result as a temporary table | Subquery in `FROM` (derived table) |

---

## 15. Cheat Sheet

```sql
-- ── SINGLE-ROW / SCALAR ───────────────────
SELECT name, salary FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
SELECT name, salary FROM employees WHERE salary = (SELECT MAX(salary) FROM employees);
SELECT name, salary, (SELECT AVG(salary) FROM employees) AS co_avg FROM employees;

-- ── MULTIPLE-ROW ──────────────────────────
SELECT name FROM employees
WHERE department_id IN (SELECT department_id FROM departments WHERE department_name IN ('IT','HR'));

-- ── CORRELATED ────────────────────────────
SELECT e1.name, e1.salary
FROM employees e1
WHERE e1.salary > (SELECT AVG(e2.salary) FROM employees e2 WHERE e2.department_id = e1.department_id);

-- ── ANY / ALL ─────────────────────────────
WHERE salary > ANY (SELECT salary FROM employees WHERE department_id = 102);  -- > at least one
WHERE salary > ALL (SELECT salary FROM employees WHERE department_id = 102);  -- > every one

-- ── EXISTS / NOT EXISTS ───────────────────
SELECT e.name FROM employees e
WHERE EXISTS (SELECT 1 FROM projects p WHERE p.emp_id = e.emp_id);

SELECT e.name FROM employees e
WHERE NOT EXISTS (SELECT 1 FROM projects p WHERE p.emp_id = e.emp_id);  -- NULL-safe anti-join

-- ── SUBQUERY IN FROM (derived table) ──────
SELECT department_id, average_salary
FROM (
    SELECT department_id, AVG(salary) AS average_salary
    FROM employees GROUP BY department_id
) AS department_averages
WHERE average_salary > 50000;

-- ── SUBQUERY IN HAVING ────────────────────
SELECT department_id, AVG(salary) AS department_average
FROM employees
GROUP BY department_id
HAVING AVG(salary) > (SELECT AVG(salary) FROM employees);
```

---

## 16. Preview of Part 16

| Topic | What You'll Learn |
|---|---|
| `WITH` | Naming a query result before using it |
| Multiple CTEs | Chaining several named steps together |
| Recursive CTE | Solving hierarchical data (employee ↔ manager, category trees) |
