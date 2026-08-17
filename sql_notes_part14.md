# SQL & PostgreSQL Complete Notes — Part 14: Multiple-Table Queries

## 📑 Table of Contents
1. The Multi-Table Query Idea
2. The Sample Schema for This Part
3. Query Involving 3 Tables
4. Query Involving 4 Tables
5. Query Involving 5 Tables
6. Bridge Tables — Joining Through a Table You Don't Even `SELECT`
7. Building a Query From a Question — Step-by-Step Method
8. Multiple Tables with `WHERE` / `GROUP BY` / `HAVING` / `LEFT JOIN`
9. Row Multiplication at Scale
10. Interview Q&A
11. Quick Revision Sheet
12. Cheat Sheet
13. Preview of Part 15

**📋 Series Coverage (Part 14):** joining 3/4/5 related tables, relationship-path thinking, bridge/intermediate tables, building multi-table queries from a plain-English question, `WHERE`/`GROUP BY`/`HAVING`/`LEFT JOIN` combined with multi-table joins, row multiplication risk with aggregates across joins

---

## 1. The Multi-Table Query Idea

**Definition** — There's no special syntax for joining 3, 4, or 5 tables — you simply keep chaining `JOIN` clauses, connecting each new table to the *existing joined result*, not necessarily directly back to the first table.

**Why It Exists** — Real schemas are normalized (Part 1) — the data you need for one report is often scattered across several related tables.

---

## 2. The Sample Schema for This Part

```
locations
   │ location_id
   ▼
departments
   │ department_id
   ▼
employees ──────┬──────────────┐
   │             │              │
   ▼             ▼              ▼
salaries      projects      (emp_id FK in both)
```

```sql
CREATE TABLE locations (
    location_id INT PRIMARY KEY,
    city        VARCHAR(100),
    country     VARCHAR(100)
);
CREATE TABLE departments (
    department_id   INT PRIMARY KEY,
    department_name VARCHAR(100),
    location_id     INT REFERENCES locations(location_id)
);
CREATE TABLE employees (
    emp_id        INT PRIMARY KEY,
    name          VARCHAR(100),
    department_id INT REFERENCES departments(department_id)
);
CREATE TABLE salaries (
    salary_id INT PRIMARY KEY,
    emp_id    INT REFERENCES employees(emp_id),
    salary    NUMERIC(10,2)
);
CREATE TABLE projects (
    project_id   INT PRIMARY KEY,
    project_name VARCHAR(100),
    emp_id       INT REFERENCES employees(emp_id)
);
```

💡 **Before writing any multi-table query, draw the relationship path first** — the `JOIN ... ON ...` clauses almost write themselves once you can see the path.

---

## 3. Query Involving 3 Tables

**Example — employee, department, salary**
```sql
SELECT e.name, d.department_name, s.salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN salaries s    ON e.emp_id = s.emp_id;
```

**Mental execution — one JOIN at a time, not "3 tables at once"**
```
employees
   ↓ JOIN departments
employees + departments               (temporary combined result)
   ↓ JOIN salaries
employees + departments + salaries    (final result)
```

---

## 4. Query Involving 4 Tables

```sql
SELECT e.name, d.department_name, s.salary, p.project_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN salaries s    ON e.emp_id = s.emp_id
JOIN projects p    ON e.emp_id = p.emp_id;
```

⚠️ **Notes & Caveats** — If an employee has 2 projects, they'll appear **twice** in this result (once per project) — each carrying the *same* salary value both times. This is expected row multiplication (Part 13, Section 12) — not a bug.

---

## 5. Query Involving 5 Tables

```sql
SELECT e.name, d.department_name, s.salary, p.project_name, l.city
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN salaries s    ON e.emp_id = s.emp_id
JOIN projects p    ON e.emp_id = p.emp_id
JOIN locations l   ON d.location_id = l.location_id;
```

**Step-by-step build**
```
1. FROM employees                          → base rows
2. JOIN departments  ON e.department_id    → + department info
3. JOIN salaries     ON e.emp_id           → + salary info
4. JOIN projects      ON e.emp_id           → + project info (may multiply rows)
5. JOIN locations      ON d.location_id     → + city info (via departments)
```

⚠️ **Notes & Caveats** — Notice `locations` connects through `d.location_id`, **not** `e.location_id` — employees has no direct location column. This is normal; you follow whatever path the foreign keys actually define.

---

## 6. Bridge Tables — Joining Through a Table You Don't Even `SELECT`

**Definition** — Sometimes a table must be joined purely to *connect* two other tables, even if none of its own columns appear in the final `SELECT`.

**Example — employee name + city, no department info requested**
```sql
SELECT e.name, l.city
FROM employees e
JOIN departments d ON e.department_id = d.department_id   -- bridge — not selected!
JOIN locations l   ON d.location_id = l.location_id;
```

❌ **Common Mistakes**
```sql
-- ❌ There's no direct relationship between employees and locations
SELECT e.name, l.city
FROM employees e
JOIN locations l ON e.department_id = l.location_id;   -- comparing unrelated IDs!
```

💡 **Best Practices** — **Join tables based on actual foreign-key relationships, not based on which columns you want to display.** `departments` is required here purely as a bridge, even though `department_name` never appears in the output.

---

## 7. Building a Query From a Question — Step-by-Step Method

**Question:** *"Show the name and salary of employees working on projects in departments located in Mohali."*

```
Step 1 — Which columns are needed, and which table holds each?
   name    → employees
   salary  → salaries
   project → projects
   Mohali  → locations (needs departments as the bridge)

Step 2 — Draw the relationship path
   locations → departments → employees → salaries
                                      └──→ projects

Step 3 — Start from the "center" table and add JOINs one at a time
   FROM employees e
   JOIN departments d ON e.department_id = d.department_id
   JOIN salaries s    ON e.emp_id = s.emp_id
   JOIN projects p    ON e.emp_id = p.emp_id
   JOIN locations l   ON d.location_id = l.location_id

Step 4 — Add the filter
   WHERE l.city = 'Mohali'
```

**Final query**
```sql
SELECT e.name, s.salary, p.project_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN salaries s    ON e.emp_id = s.emp_id
JOIN projects p    ON e.emp_id = p.emp_id
JOIN locations l   ON d.location_id = l.location_id
WHERE l.city = 'Mohali';
```

---

## 8. Multiple Tables with `WHERE` / `GROUP BY` / `HAVING` / `LEFT JOIN`

**`WHERE` after multiple joins**
```sql
SELECT e.name, d.department_name, s.salary, p.project_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN salaries s    ON e.emp_id = s.emp_id
JOIN projects p    ON e.emp_id = p.emp_id
WHERE d.department_name = 'IT' AND s.salary > 60000;
```

**`GROUP BY` + `HAVING` across joined tables**
```sql
SELECT d.department_name, l.city, AVG(s.salary) AS average_salary
FROM departments d
JOIN employees e  ON d.department_id = e.department_id
JOIN salaries s   ON e.emp_id = s.emp_id
JOIN locations l  ON d.location_id = l.location_id
GROUP BY d.department_id, d.department_name, l.city
HAVING AVG(s.salary) > 50000;
```

**`LEFT JOIN` for an optional relationship (employees without a project)**
```sql
SELECT e.name, d.department_name, s.salary, l.city
FROM employees e
JOIN departments d      ON e.department_id = d.department_id
JOIN salaries s          ON e.emp_id = s.emp_id
LEFT JOIN projects p     ON e.emp_id = p.emp_id     -- optional relationship
JOIN locations l          ON d.location_id = l.location_id
WHERE p.project_id IS NULL;                          -- Part 13's anti-join pattern
```

💡 **How to Choose** — Use `JOIN` for relationships you expect to always exist (an employee should always have a valid department); use `LEFT JOIN` for genuinely optional relationships (not every employee has a project) so you don't accidentally drop otherwise-valid rows.

---

## 9. Row Multiplication at Scale

⚠️ **Notes & Caveats** — The more tables you join with 1:N relationships, the more the row count can multiply. If Aman has 2 salary rows *and* 3 project rows, joining both in the same query produces **2 × 3 = 6** combined rows for Aman alone — each row pairing one salary with one project. Running `SUM(salary)` directly on that joined result would wildly over-count.

💡 **Best Practices** — Before aggregating on top of a multi-table join, mentally (or literally) inspect the joined row set first: *how many rows does this employee actually appear in, and why?* If numbers look inflated, consider aggregating each side (e.g., total salary per employee) in a separate step (subquery/CTE, covered in Part 15/16) *before* joining, rather than joining first and aggregating after.

---

## 10. Interview Q&A

**Q: Is there special syntax for joining more than 2 tables?**
A: No — you simply add more `JOIN ... ON ...` clauses, each one connecting to the combined result built so far.

**Q: Do all joined tables need to connect directly back to the first (`FROM`) table?**
A: No — a later `JOIN` can connect to *any* table already brought into the result. E.g., `locations` connects via `departments.location_id`, not directly to `employees`.

**Q: What is a "bridge table" in a multi-table query?**
A: A table that must be joined purely to establish a relationship path between two other tables, even if none of its own columns appear in the final `SELECT` list.

**Q: What's the risk of joining an employee to both their salaries and their projects in one query?**
A: If the employee has multiple rows on either side, the join produces every combination (rows multiply), which can silently inflate any `SUM`/`COUNT`/`AVG` computed afterward on the combined result.

**Q: How should you decide the order to build a complex multi-table query?**
A: Identify which columns you need and which tables hold them, draw the relationship path between those tables, then add `JOIN`s one at a time starting from a central table, checking the result after each addition before layering on `WHERE`/`GROUP BY`/`HAVING`.

**Q: When would you use `LEFT JOIN` instead of `JOIN` in a multi-table query?**
A: When a relationship is genuinely optional (e.g., not every employee has a project) and you don't want rows missing that optional link to disappear from the result entirely.

---

## 11. Quick Revision Sheet

| Step | Action |
|---|---|
| 1 | List needed columns → identify their source tables |
| 2 | Draw the relationship path between those tables |
| 3 | `FROM` the most central table |
| 4 | Add one `JOIN` at a time, following the path (bridge tables included) |
| 5 | Add `WHERE` / `GROUP BY` / `HAVING` last |
| 6 | Watch for row multiplication before aggregating |

---

## 12. Cheat Sheet

```sql
-- ── 3-TABLE JOIN ──────────────────────────
SELECT e.name, d.department_name, s.salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN salaries s    ON e.emp_id = s.emp_id;

-- ── 5-TABLE JOIN ──────────────────────────
SELECT e.name, d.department_name, s.salary, p.project_name, l.city
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN salaries s    ON e.emp_id = s.emp_id
JOIN projects p    ON e.emp_id = p.emp_id
JOIN locations l   ON d.location_id = l.location_id;

-- ── BRIDGE TABLE (not selected, still required) ──
SELECT e.name, l.city
FROM employees e
JOIN departments d ON e.department_id = d.department_id   -- bridge
JOIN locations l   ON d.location_id = l.location_id;

-- ── MULTI-TABLE + GROUP BY + HAVING ───────
SELECT d.department_name, l.city, AVG(s.salary) AS avg_salary
FROM departments d
JOIN employees e ON d.department_id = e.department_id
JOIN salaries s   ON e.emp_id = s.emp_id
JOIN locations l  ON d.location_id = l.location_id
GROUP BY d.department_id, d.department_name, l.city
HAVING AVG(s.salary) > 50000;

-- ── MULTI-TABLE + OPTIONAL RELATIONSHIP ───
SELECT e.name, d.department_name
FROM employees e
JOIN departments d       ON e.department_id = d.department_id
LEFT JOIN projects p     ON e.emp_id = p.emp_id
WHERE p.project_id IS NULL;    -- employees with no project
```

---

## 13. Preview of Part 15

| Topic | What You'll Learn |
|---|---|
| Single-row / multi-row / scalar subqueries | Nesting one query inside another |
| Correlated subqueries | Subqueries that depend on the outer row |
| `ANY`, `ALL` | Comparing against a set of values |
| `EXISTS`, `NOT EXISTS` | Existence checks — and why they're safer than `NOT IN` |
