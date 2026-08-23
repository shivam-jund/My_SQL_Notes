# SQL & PostgreSQL Complete Notes — Part 28: Indexes I — Fundamentals & B-Tree

## 📑 Table of Contents
1. What Is an Index?
2. Sequential Scan vs Index Scan
3. `CREATE INDEX`
4. `DROP INDEX`
5. The Cost of Indexes — Why Not Index Everything
6. B-Tree — PostgreSQL's Default Index
7. What Queries Benefit from a B-Tree Index
8. Primary Keys, Foreign Keys, and Indexes
9. Interview Q&A
10. Quick Revision Sheet
11. Cheat Sheet
12. Preview of Part 29

**📋 Series Coverage (Part 28):** what an index is, sequential scan vs index scan, `CREATE INDEX`, `DROP INDEX` (+ `IF EXISTS`), index storage/write costs, B-Tree structure and time complexity, which operators B-Tree supports, automatic PK indexing, why FK columns often need manual indexes, why duplicating the PK's index is wasteful

> ⭐ Indexes are flagged **"very important"** in the roadmap — one of the most consistently asked SQL performance topics in interviews.

---

## 1. What Is an Index?

**Definition** — A separate data structure that helps PostgreSQL locate rows **without scanning the entire table**.

**Why It Exists** — Without an index, finding a single row among millions means checking every row one by one.

💡 **Analogy** — A book's back-of-book index: instead of reading every page to find "PostgreSQL," you look it up and jump straight to page 742.

⚠️ **Notes & Caveats** — An index does **not** copy or duplicate your table's data. It only stores a **search structure** (indexed column values + pointers back to the actual rows). The table itself still holds the real data.

```
              DATABASE
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
      TABLE               INDEX
        │                   │
    Actual data        Search structure
```

---

## 2. Sequential Scan vs Index Scan

**Without an index — Sequential Scan ("Seq Scan")**
```
Row 1 → Row 2 → Row 3 → ... → Row 10,000,000
```
Time complexity: **O(n)** — potentially checks every row.

**With an index — Index Scan**
```
Index → jump directly to matching location(s) → fetch row
```
For a B-Tree, roughly **O(log n)** — dramatically fewer comparisons for large tables.

---

## 3. `CREATE INDEX`

**Definition** — Builds an index on one or more columns of a table.

**Syntax**
```sql
CREATE INDEX index_name
ON table_name (column_name);
```

**Example**
```sql
CREATE INDEX idx_employee_name ON employees(name);
```
**Output**
```
CREATE INDEX
```
Now `SELECT * FROM employees WHERE name = 'Aman';` can potentially use this index instead of scanning every row.

---

## 4. `DROP INDEX`

**Syntax**
```sql
DROP INDEX index_name;
DROP INDEX IF EXISTS index_name;
```

⚠️ **Notes & Caveats** — Dropping an index never affects the table's data — only the search structure is removed.

---

## 5. The Cost of Indexes — Why Not Index Everything

❌ **Common Mistakes** — Assuming "more indexes = always faster." Indexes are **not free**:

| Cost | Why It Happens |
|---|---|
| Extra storage | Every index is its own physical structure on disk |
| Slower `INSERT` | PostgreSQL must add a new entry to every index on the table |
| Slower `UPDATE` | Modifying an indexed column requires updating that index too |
| Slower `DELETE` | Removing a row means removing its entries from every index |

**Rule of thumb — good candidates for indexing:**
- Columns frequently used in `WHERE`
- Columns frequently used in `JOIN`
- Columns frequently used in `ORDER BY` / `GROUP BY`

**Avoid indexing:**
- Columns that change very frequently
- Columns with very few distinct values (low selectivity — covered in Part 29)
- Columns rarely referenced in queries

💡 **Best Practices** — The right index for the right query is good; indexing "just in case" is usually not.

---

## 6. B-Tree — PostgreSQL's Default Index

**Definition** — A self-balancing tree structure that keeps values sorted, allowing PostgreSQL to navigate to a target value in relatively few steps instead of checking every row.

⚠️ **Notes & Caveats** — When you write plain `CREATE INDEX ... ON table(column);` with no method specified, PostgreSQL creates a **B-Tree** by default.

```
              50
            /    \
          30      70
         /  \    /  \
       10   40  60   80
```
Instead of checking every value in order, PostgreSQL follows branches — dramatically fewer comparisons.

**Time complexity**

| | Complexity |
|---|---|
| Sequential Scan | O(n) |
| B-Tree Index Scan | ~O(log n) |

---

## 7. What Queries Benefit from a B-Tree Index

B-Tree supports:
```sql
=          -- equality
<  <=      -- less than / less-or-equal
>  >=      -- greater than / greater-or-equal
BETWEEN    -- range
ORDER BY   -- because the index is already sorted
```

**Example**
```sql
CREATE INDEX idx_salary ON employees(salary);
SELECT * FROM employees WHERE salary > 70000;
```

💡 **Notes & Caveats** — A B-Tree indexes the **whole value** as one unit — it's not designed to efficiently search *inside* a complex value like a JSON document or an array (that's what GIN/GiST, covered in Part 30, are for).

---

## 8. Primary Keys, Foreign Keys, and Indexes

**PostgreSQL automatically indexes a `PRIMARY KEY`** — because enforcing uniqueness on every `INSERT`/`UPDATE` requires a fast way to check "does this value already exist?"

⚠️ **Notes & Caveats — a very common beginner mistake:**
```sql
-- ❌ Redundant — student_id is ALREADY indexed automatically via PRIMARY KEY
CREATE TABLE students (student_id INT PRIMARY KEY, ...);
CREATE INDEX idx_student_id ON students(student_id);   -- wasteful duplicate
```
A duplicate index on the exact same column as the PK provides no benefit and only adds write overhead, storage, and query-planner confusion.

**But PostgreSQL does NOT automatically index a `FOREIGN KEY` column.**
```sql
CREATE TABLE employees (
    emp_id        SERIAL PRIMARY KEY,     -- ⭐ auto-indexed
    department_id INT REFERENCES departments(department_id)   -- ⚠️ NOT auto-indexed
);
```
💡 **Best Practices** — Since a typical `JOIN` matches a foreign key against a primary key, and the FK side isn't automatically indexed, it's often worth adding one manually if that join happens frequently:
```sql
CREATE INDEX idx_employees_department ON employees(department_id);
```

**The one exception worth knowing — composite primary keys:**
```sql
PRIMARY KEY (user_id, group_id)
```
- An index on just `(user_id)` is redundant — the composite PK index can already be used as a "leftmost prefix" for `user_id` alone (fully explained in Part 29).
- An index on just `(group_id)` **can** be genuinely useful — the composite index can't efficiently search by `group_id` alone.

---

## 9. Interview Q&A

**Q: Does an index change or duplicate the table's data?**
A: No — an index is a separate search structure containing indexed values and pointers back to the real rows; the table's actual data is untouched.

**Q: What's the time complexity difference between a sequential scan and a B-Tree index scan?**
A: A sequential scan is O(n) — it may check every row. A B-Tree index scan is roughly O(log n) — it navigates directly toward the target using the tree's sorted structure.

**Q: Why shouldn't you index every column?**
A: Every index adds storage overhead and slows down `INSERT`/`UPDATE`/`DELETE`, since each one must also update every index on the table. Indexes should be added deliberately for columns frequently used in `WHERE`, `JOIN`, `ORDER BY`, or `GROUP BY`.

**Q: Is it useful to manually create an index on a `PRIMARY KEY` column?**
A: No — PostgreSQL automatically creates a unique index to enforce the primary key constraint; a duplicate manual index on the same column is pure waste.

**Q: Does PostgreSQL automatically index foreign key columns?**
A: No — only the referenced primary key side is automatically indexed. The foreign key column itself is not, so it's often worth adding manually if it's used in frequent joins or lookups.

**Q: What operators does a B-Tree index support well?**
A: `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, and it can also help avoid sorting for a matching `ORDER BY`.

---

## 10. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Create a basic (B-Tree) index | `CREATE INDEX name ON table(column);` |
| Drop an index | `DROP INDEX IF EXISTS name;` |
| Good index candidates | `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY` columns |
| Already auto-indexed | `PRIMARY KEY` columns |
| Usually needs manual index | `FOREIGN KEY` columns used in joins |
| B-Tree supports | `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `ORDER BY` |

---

## 11. Cheat Sheet

```sql
-- ── BASIC INDEX ───────────────────────────
CREATE INDEX idx_employee_name ON employees(name);
CREATE INDEX idx_salary ON employees(salary);

-- ── DROP ──────────────────────────────────
DROP INDEX idx_salary;
DROP INDEX IF EXISTS idx_salary;

-- ── FOREIGN KEY INDEX (often worth adding manually) ──
CREATE INDEX idx_employees_department ON employees(department_id);

-- ── WHAT B-TREE SPEEDS UP ─────────────────
SELECT * FROM employees WHERE salary > 70000;         -- range
SELECT * FROM employees WHERE name = 'Aman';            -- equality
SELECT * FROM employees ORDER BY salary;                 -- sort
```

---

## 12. Preview of Part 29

| Topic | What You'll Learn |
|---|---|
| `UNIQUE INDEX` | Preventing duplicates + fast lookup |
| Composite Index | Indexing multiple columns together |
| Leftmost Prefix Rule ⭐ | Which queries a composite index can and can't help |
| Partial Index | Indexing only a subset of rows |
| Expression Index | Indexing a computed expression like `LOWER(email)` |
