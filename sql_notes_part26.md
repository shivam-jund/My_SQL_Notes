# SQL & PostgreSQL Complete Notes — Part 26: Views I — Creating, Querying & Updating

## 📑 Table of Contents
1. What Is a View?
2. Why Do We Need Views?
3. `CREATE VIEW`
4. Querying a View
5. View vs Table
6. Views Always Reflect Current Data
7. Filtering and Joins Inside a View
8. Updating Data Through a View
9. Updatable vs Non-Updatable Views
10. `CREATE OR REPLACE VIEW`
11. `DROP VIEW`
12. `WITH CHECK OPTION`
13. Interview Q&A
14. Quick Revision Sheet
15. Cheat Sheet
16. Preview of Part 27

**📋 Series Coverage (Part 26):** what a view is, why views exist (security, simplification, reuse), `CREATE VIEW`, querying a view, view vs table, updatable vs non-updatable views, `INSERT`/`UPDATE`/`DELETE` through a view, `CREATE OR REPLACE VIEW`, `DROP VIEW` (+ `IF EXISTS`), `WITH CHECK OPTION`

---

## 1. What Is a View?

**Definition** — A view is a **virtual table** — the saved definition of a `SELECT` query, not a copy of the data itself.

**Why It Exists** — Sometimes you want to present data differently (hide columns, pre-join tables, pre-filter rows) without duplicating storage or forcing every user to rewrite the same complex query.

```
Real table: employees
        ↓
View: employee_public  → remembers  SELECT name, department FROM employees;
```

⚠️ **Notes & Caveats** — A view does **not** copy or store the underlying data. Every time it's queried, PostgreSQL **re-runs** the saved query against the real table(s).

💡 **Analogy** — A camera doesn't create another world; it just shows a different perspective of the same one. A view is the same relationship to its underlying table(s).

---

## 2. Why Do We Need Views?

| Reason | Example |
|---|---|
| **Hide sensitive data** | HR shouldn't see `salary` — a view can expose only `name`, `department` |
| **Simplify complex queries** | A 3-table join used daily → wrap it in a view, then just `SELECT * FROM report_view` |
| **Reuse** | The same query used repeatedly → save it once as a view |
| **Security / abstraction** | Grant access to the view, not the raw table |

---

## 3. `CREATE VIEW`

**Definition** — Saves a `SELECT` query under a name, so it can be queried like a table.

**Syntax**
```sql
CREATE VIEW view_name AS
SELECT ...
FROM table_name;
```

**Example**
```sql
CREATE VIEW employee_public AS
SELECT name, department
FROM employees;
```
**Output**
```
CREATE VIEW
```
*(No data is copied — PostgreSQL just stores the query definition.)*

**Example — with a `WHERE` filter**
```sql
CREATE VIEW it_employees AS
SELECT id, name, salary
FROM employees
WHERE department = 'IT';
```

---

## 4. Querying a View

Once created, a view behaves exactly like a table for `SELECT` purposes.

```sql
SELECT * FROM employee_public;
```
**Output**
```
name  | department
Aman  | IT
Priya | HR
Ravi  | IT
```

⚠️ **Notes & Caveats** — Internally, PostgreSQL translates this into the view's stored query — here, `SELECT name, department FROM employees;` — and runs it fresh.

---

## 5. View vs Table

| | Table | View |
|---|---|---|
| Stores data? | Yes | No — stores only the query |
| Storage used | Full data | Minimal (just the query definition) |
| Can exist independently? | Yes | No — depends on its underlying table(s) |
| Where does the data come from? | Entered directly | Computed from the underlying table(s) each time |

---

## 6. Views Always Reflect Current Data

**Example**
```sql
-- Original table has: Aman, Priya
INSERT INTO employees (name) VALUES ('Ravi');

SELECT * FROM employee_public;
-- Now returns: Aman, Priya, Ravi
```

⚠️ **Notes & Caveats** — A normal view has **no memory of the past** — it always reruns its stored query, so it automatically reflects the current state of the underlying table(s). (This changes with **materialized** views — Part 27.)

---

## 7. Filtering and Joins Inside a View

**Views can include `WHERE`:**
```sql
CREATE VIEW high_salary AS
SELECT * FROM employees WHERE salary > 60000;
```

**Views can include `JOIN`:**
```sql
CREATE VIEW employee_department AS
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.id;
```
*(Anyone using this view never has to rewrite the join.)*

---

## 8. Updating Data Through a View

**`UPDATE` through a view**
```sql
UPDATE employee_public SET department = 'Finance' WHERE id = 2;
```
This actually modifies the **underlying `employees` table** — the view has no separate copy to update.

**`INSERT` through a view**
```sql
INSERT INTO employee_public (id, name, department) VALUES (4, 'Simran', 'Sales');
```

**`DELETE` through a view**
```sql
DELETE FROM employee_public WHERE id = 3;
```
*(Deletes the row from the base `employees` table.)*

---

## 9. Updatable vs Non-Updatable Views

**A view is usually updatable when it is:**
- Based on a **single table**
- A simple `SELECT` — no `GROUP BY`, `DISTINCT`, aggregate functions, or set operations (`UNION`/`INTERSECT`/`EXCEPT`)

**Example — NOT updatable (uses `GROUP BY` + aggregate)**
```sql
CREATE VIEW department_salary AS
SELECT department, AVG(salary) FROM employees GROUP BY department;

UPDATE department_salary SET department = 'IT';
-- ERROR: cannot update a view that ... uses aggregate functions
```

⚠️ **Notes & Caveats — why this fails** — `AVG(salary)` doesn't correspond to any single row; one result row represents *many* underlying employees. PostgreSQL has no way to know which specific employee row(s) an update should apply to.

**Comparison table**

| Usually Updatable | Usually NOT Updatable |
|---|---|
| Single table | `JOIN`s across multiple tables* |
| Plain `SELECT` | `GROUP BY` |
| No `DISTINCT` | `DISTINCT` |
| No aggregates | `COUNT()`, `SUM()`, `AVG()`, etc. |
| | `UNION` / `INTERSECT` / `EXCEPT` |

*\*Simple single-table-per-side joins can sometimes be partially updatable in PostgreSQL, but treat multi-table views as read-only by default unless you've confirmed otherwise.*

---

## 10. `CREATE OR REPLACE VIEW`

**Definition** — Redefines an existing view's query without dropping and recreating it.

**Syntax**
```sql
CREATE OR REPLACE VIEW view_name AS
SELECT ...
```

**Example**
```sql
CREATE OR REPLACE VIEW employee_public AS
SELECT name, department, salary   -- salary column added
FROM employees;
```

⚠️ **Notes & Caveats** — The view's **name** stays the same; only the stored query is overwritten.

---

## 11. `DROP VIEW`

**Definition** — Removes a view definition. The underlying table and its data are **not** affected.

**Syntax**
```sql
DROP VIEW view_name;
DROP VIEW IF EXISTS view_name;
```

**Example**
```sql
DROP VIEW IF EXISTS employee_public;
```

---

## 12. `WITH CHECK OPTION`

**Definition** — Rejects any `INSERT`/`UPDATE` through the view that would produce a row **no longer matching** the view's own `WHERE` condition.

**The problem without it**
```sql
CREATE VIEW it_employees AS
SELECT * FROM employees WHERE department = 'IT';

UPDATE it_employees SET department = 'HR' WHERE name = 'Aman';
-- succeeds — but Aman silently DISAPPEARS from the it_employees view afterward,
-- because he no longer satisfies department = 'IT'
```

**The fix**
```sql
CREATE VIEW it_employees AS
SELECT * FROM employees
WHERE department = 'IT'
WITH CHECK OPTION;

UPDATE it_employees SET department = 'HR' WHERE name = 'Aman';
-- ❌ REJECTED — the update would make this row no longer satisfy the view's WHERE clause
```

**Mental model**
```
Before saving the change:
   Would this row still belong to the view afterward?
       YES → save
       NO  → reject
```

❌ **Common Mistakes**
- Assuming dropping a view deletes the underlying table — it doesn't; only the view definition is removed.
- Assuming every view supports `INSERT`/`UPDATE`/`DELETE` — only updatable views do (Section 9).
- Forgetting `WITH CHECK OPTION` and being surprised when an update makes a row vanish from a filtered view.

---

## 13. Interview Q&A

**Q: What is a view?**
A: A virtual table — a saved `SELECT` query that behaves like a table when queried, but stores no data of its own; it re-executes its underlying query on every access.

**Q: Does a view consume significant storage?**
A: No — a normal view only stores the query definition, not the result data. (Materialized views, covered next, are the exception.)

**Q: If you insert a new row into the base table, does an existing view automatically reflect it?**
A: Yes — because the view reruns its stored query fresh every time it's queried, it always shows current data from the underlying table(s).

**Q: Why can't you `UPDATE` a view built with `GROUP BY` and `AVG()`?**
A: Because an aggregated result row represents multiple underlying rows — PostgreSQL has no way to determine which specific base row(s) an update should be applied to.

**Q: What's the difference between `DROP VIEW` and dropping the underlying table?**
A: `DROP VIEW` only removes the view definition; the base table and its data are completely unaffected.

**Q: What problem does `WITH CHECK OPTION` solve?**
A: Without it, an `UPDATE`/`INSERT` through a filtered view can silently produce a row that no longer satisfies the view's own `WHERE` condition, causing it to disappear from the view. `WITH CHECK OPTION` rejects such changes instead.

**Q: How do you modify an existing view's definition without dropping it first?**
A: `CREATE OR REPLACE VIEW view_name AS SELECT ...` — the view's name is preserved, only its stored query changes.

---

## 14. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Create a view | `CREATE VIEW name AS SELECT ...;` |
| Query a view | `SELECT * FROM view_name;` |
| Update a view's definition | `CREATE OR REPLACE VIEW name AS SELECT ...;` |
| Delete a view | `DROP VIEW IF EXISTS name;` |
| Prevent updates that break the filter | Add `WITH CHECK OPTION` |
| Usually updatable | Single table, no `GROUP BY`/`DISTINCT`/aggregates |
| Usually NOT updatable | Multi-table joins, `GROUP BY`, aggregates, `UNION` |

---

## 15. Cheat Sheet

```sql
-- ── CREATE / QUERY ────────────────────────
CREATE VIEW employee_public AS
SELECT name, department FROM employees;

SELECT * FROM employee_public;

-- ── FILTERED / JOINED VIEW ────────────────
CREATE VIEW it_employees AS
SELECT id, name, salary FROM employees WHERE department = 'IT';

CREATE VIEW employee_department AS
SELECT e.name, d.department_name
FROM employees e JOIN departments d ON e.department_id = d.id;

-- ── UPDATE / INSERT / DELETE THROUGH A VIEW ──
UPDATE employee_public SET department = 'Finance' WHERE id = 2;
INSERT INTO employee_public (id, name, department) VALUES (4, 'Simran', 'Sales');
DELETE FROM employee_public WHERE id = 3;

-- ── REPLACE / DROP ────────────────────────
CREATE OR REPLACE VIEW employee_public AS
SELECT name, department, salary FROM employees;

DROP VIEW IF EXISTS employee_public;

-- ── WITH CHECK OPTION ─────────────────────
CREATE VIEW it_employees AS
SELECT * FROM employees
WHERE department = 'IT'
WITH CHECK OPTION;
```

---

## 16. Preview of Part 27

| Topic | What You'll Learn |
|---|---|
| Materialized Views | Storing a view's result physically on disk |
| `CREATE MATERIALIZED VIEW` / `REFRESH MATERIALIZED VIEW` | Creating and updating precomputed results |
| View vs Materialized View | The most-asked interview question in this chapter |
