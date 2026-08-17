# SQL & PostgreSQL Complete Notes — Part 12: HAVING

## 📑 Table of Contents
1. `HAVING` — Filtering Groups
2. `WHERE` vs `HAVING` — The Core Comparison
3. `HAVING` with `COUNT()`
4. `HAVING` with `SUM()`
5. `HAVING` with `AVG()`
6. `HAVING` with `MAX()` / `MIN()`
7. Using `WHERE` and `HAVING` Together
8. Can `HAVING` Be Used Without `GROUP BY`?
9. Can Normal Columns Be Used in `HAVING`?
10. The Full SQL Logical Execution Order
11. Common Mistakes
12. How to Decide: `WHERE` or `HAVING`?
13. Interview Q&A
14. Quick Revision Sheet
15. Cheat Sheet
16. Preview of Part 13

**📋 Series Coverage (Part 12):** `HAVING`, `WHERE` vs `HAVING`, `HAVING` with `COUNT`/`SUM`/`AVG`/`MAX`/`MIN`, combining `WHERE` + `GROUP BY` + `HAVING`, `HAVING` without `GROUP BY`, full logical query execution order

> Same `employees` table as Part 11, plus `Neha | Sales | 55000`.

---

## 1. `HAVING` — Filtering Groups

**Definition** — Filters **groups** *after* `GROUP BY` has run — the group-level equivalent of `WHERE`.

**Why It Exists** — `WHERE` runs *before* grouping, so at the moment `WHERE` executes, aggregate results like `COUNT(*)` or `AVG(salary)` don't exist yet per group. `HAVING` runs *after* grouping specifically so it can filter on those computed aggregate values.

**Syntax**
```sql
SELECT group_column, AGG_FUNC(col)
FROM table_name
GROUP BY group_column
HAVING AGG_FUNC(col) condition;
```

**Example**
```sql
SELECT department, COUNT(*) AS total_employees
FROM employees
GROUP BY department
HAVING COUNT(*) > 2;
```
**Output**
```
department | total_employees
IT         | 3
```
*(HR and Sales both have exactly 2 — filtered out.)*

❌ **Common Mistakes**
```sql
-- ❌ COUNT(*) doesn't exist yet when WHERE runs — groups haven't formed
SELECT department, COUNT(*)
FROM employees
WHERE COUNT(*) > 2
GROUP BY department;
-- ERROR: aggregate functions are not allowed in WHERE
```
```sql
-- ✅ Use HAVING instead — it runs after grouping
SELECT department, COUNT(*)
FROM employees
GROUP BY department
HAVING COUNT(*) > 2;
```

---

## 2. `WHERE` vs `HAVING` — The Core Comparison

| | `WHERE` | `HAVING` |
|---|---|---|
| Filters | Individual **rows** | **Groups** |
| Runs | Before `GROUP BY` | After `GROUP BY` |
| Aggregate functions? | ❌ Not allowed | ✅ Commonly used |
| Typical condition | `salary > 50000` | `AVG(salary) > 50000` |
| Mental question | "Should *this row* participate?" | "Should *this group* survive?" |

**Side-by-side — same table, very different results**

```sql
-- Query A: filter EMPLOYEES first, then average what's left
SELECT department, AVG(salary)
FROM employees
WHERE salary > 50000
GROUP BY department;

-- Query B: average everything first, then filter DEPARTMENTS by their average
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000;
```
These answer **different questions** — Query A recalculates averages using only high earners; Query B keeps everyone in the average but only shows departments whose *overall* average clears the bar.

💡 **Memory trick:** `WHERE` = row filter, before grouping · `HAVING` = group filter, after grouping.

---

## 3. `HAVING` with `COUNT()`

```sql
SELECT department, COUNT(*) AS total_employees
FROM employees
GROUP BY department
HAVING COUNT(*) > 2;
```

---

## 4. `HAVING` with `SUM()`

```sql
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department
HAVING SUM(salary) > 100000;
```

---

## 5. `HAVING` with `AVG()`

```sql
SELECT department, AVG(salary) AS average_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000;
```

---

## 6. `HAVING` with `MAX()` / `MIN()`

```sql
SELECT department, MAX(salary) AS highest_salary
FROM employees
GROUP BY department
HAVING MAX(salary) > 70000;

SELECT department, MIN(salary) AS lowest_salary
FROM employees
GROUP BY department
HAVING MIN(salary) >= 50000;
```

---

## 7. Using `WHERE` and `HAVING` Together

**Example** — "Consider only employees earning more than 40000, then find departments with at least 2 such employees."

```sql
SELECT department, COUNT(*) AS total_employees
FROM employees
WHERE salary > 40000
GROUP BY department
HAVING COUNT(*) >= 2;
```

**Mental execution**
```
FROM employees
   ↓
WHERE salary > 40000        ← filters ROWS (Ravi, exactly 40000, is removed)
   ↓
GROUP BY department          ← creates groups from what's left
   ↓
COUNT(*)                      ← count per group
   ↓
HAVING COUNT(*) >= 2           ← filters GROUPS
```

---

## 8. Can `HAVING` Be Used Without `GROUP BY`?

**Yes.** Without `GROUP BY`, all rows form one implicit group.

```sql
SELECT AVG(salary)
FROM employees
HAVING AVG(salary) > 50000;
```
If the overall average clears 50000, one row is returned; otherwise, zero rows.

---

## 9. Can Normal Columns Be Used in `HAVING`?

**Technically yes** in PostgreSQL if the column is part of `GROUP BY`:
```sql
SELECT department, COUNT(*)
FROM employees
GROUP BY department
HAVING department = 'IT';   -- works, but...
```

💡 **Best Practices** — This is almost always better expressed with `WHERE`, since it's a plain row-level condition:
```sql
SELECT department, COUNT(*)
FROM employees
WHERE department = 'IT'
GROUP BY department;
```
Filtering early with `WHERE` avoids the wasted work of grouping rows you're going to throw away anyway.

---

## 10. The Full SQL Logical Execution Order

Even though you *type* clauses in this order:
```sql
SELECT department, AVG(salary)
FROM employees
WHERE salary > 40000
GROUP BY department
HAVING AVG(salary) > 50000
ORDER BY AVG(salary) DESC;
```

PostgreSQL logically **processes** them in this order:
```
1. FROM        → get the base table(s)
2. WHERE       → filter individual rows
3. GROUP BY    → create groups
4. HAVING      → filter groups
5. SELECT      → choose/compute output columns
6. ORDER BY    → sort the final result
7. LIMIT       → cap the row count
```

💡 This explains every "why doesn't X work here" question in SQL — a clause can only use things that already exist at its position in this pipeline (e.g., `WHERE` can't use a column alias defined in `SELECT`, because `SELECT` hasn't run yet).

---

## 11. Common Mistakes

❌ **Mistake 1 — using `WHERE` with an aggregate**
```sql
SELECT department, COUNT(*)
FROM employees
WHERE COUNT(*) > 2       -- ❌ groups don't exist yet
GROUP BY department;
```
✅ Fix: move the condition to `HAVING`.

❌ **Mistake 2 — using `HAVING` for a plain row condition**
```sql
SELECT * FROM employees HAVING salary > 50000;   -- ❌ no grouping happening here
```
✅ Fix:
```sql
SELECT * FROM employees WHERE salary > 50000;
```

❌ **Mistake 3 — confusing "filter rows, then average" with "average, then filter departments"**
```sql
-- This is NOT the same as "departments with average salary > 50000"
SELECT department, AVG(salary)
FROM employees
WHERE salary > 50000
GROUP BY department;
```
✅ Fix, if the goal really is "departments whose average is above 50000":
```sql
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000;
```

---

## 12. How to Decide: `WHERE` or `HAVING`?

```
Can the condition be checked on ONE individual row?
   YES → WHERE
   NO  → does it depend on COUNT/SUM/AVG/MAX/MIN?
           YES → HAVING
```

---

## 13. Interview Q&A

**Q: What's the difference between `WHERE` and `HAVING`?**
A: `WHERE` filters individual rows before grouping and can't reference aggregate functions. `HAVING` filters groups after `GROUP BY` has run and is commonly used with aggregate conditions like `COUNT()`, `SUM()`, or `AVG()`.

**Q: Why can't you use an aggregate function inside `WHERE`?**
A: Because `WHERE` executes before `GROUP BY` — at that point in the logical pipeline, no groups (and therefore no per-group aggregate values) exist yet.

**Q: Can `HAVING` be used without `GROUP BY`?**
A: Yes — without `GROUP BY`, all matching rows form one implicit group, and `HAVING` filters that single group's aggregate result.

**Q: Give an example where `WHERE salary > X` and `HAVING AVG(salary) > X` produce genuinely different results.**
A: `WHERE salary > 50000` removes individual low earners *before* averaging, so the department average is computed only from the survivors. `HAVING AVG(salary) > 50000` computes the average across *everyone* in the department first, then only keeps departments whose overall average clears 50000 — a department full of moderate earners could pass the `HAVING` version but fail the `WHERE` version (if too few employees remain to even form a group).

**Q: What's the full logical execution order of a `SELECT` query with `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, and `LIMIT`?**
A: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY` → `LIMIT`.

**Q: Is it valid to put a non-aggregated, non-grouped column condition inside `HAVING`?**
A: In PostgreSQL it's technically allowed if the column is part of `GROUP BY`, but it's considered bad practice — such row-level conditions belong in `WHERE`, which filters earlier and more efficiently.

---

## 14. Quick Revision Sheet

| Scenario | Clause |
|---|---|
| "employees earning > 50000" | `WHERE` |
| "departments with more than 2 employees" | `HAVING COUNT(*) > 2` |
| "departments whose average salary > 50000" | `HAVING AVG(salary) > 50000` |
| "IT department only" | `WHERE department = 'IT'` |
| Row condition | `WHERE` |
| Aggregate/group condition | `HAVING` |

---

## 15. Cheat Sheet

```sql
-- ── HAVING BASICS ─────────────────────────
SELECT department, COUNT(*) AS n
FROM employees
GROUP BY department
HAVING COUNT(*) > 2;

SELECT department, SUM(salary) AS total
FROM employees
GROUP BY department
HAVING SUM(salary) > 100000;

SELECT department, AVG(salary) AS avg_sal
FROM employees
GROUP BY department
HAVING AVG(salary) > 50000;

-- ── WHERE + GROUP BY + HAVING TOGETHER ────
SELECT department, COUNT(*) AS n
FROM employees
WHERE salary > 40000          -- 1. filter rows
GROUP BY department            -- 2. create groups
HAVING COUNT(*) >= 2;          -- 3. filter groups

-- ── HAVING WITHOUT GROUP BY ───────────────
SELECT AVG(salary)
FROM employees
HAVING AVG(salary) > 50000;    -- whole table = one implicit group

-- ── FULL PIPELINE ─────────────────────────
SELECT department, AVG(salary)
FROM employees
WHERE salary > 40000
GROUP BY department
HAVING AVG(salary) > 50000
ORDER BY AVG(salary) DESC;
```

---

## 16. Preview of Part 13

| Topic | What You'll Learn |
|---|---|
| `INNER JOIN` | Only matching rows from both tables |
| `LEFT` / `RIGHT` / `FULL OUTER JOIN` | Keeping unmatched rows |
| `CROSS JOIN` | Every combination |
| `SELF JOIN` | Joining a table to itself (employee ↔ manager) |
| `ON` vs `USING` vs `NATURAL JOIN` | Join syntax variants |
