# SQL & PostgreSQL Complete Notes — Part 7: SELECT — Reading Data

## 📑 Table of Contents
1. `SELECT *`
2. `SELECT column, column` (Specific Columns)
3. `SELECT DISTINCT`
4. `DISTINCT` with Multiple Columns
5. `ORDER BY`
6. `ASC` (Ascending)
7. `DESC` (Descending)
8. `ORDER BY` with Multiple Columns
9. `LIMIT`
10. `OFFSET`
11. `LIMIT` + `OFFSET` Together (Pagination)
12. The Logical Query Pipeline (Mental Model)
13. Interview Q&A
14. Quick Revision Sheet
15. Cheat Sheet
16. Preview of Part 8

**📋 Series Coverage (Part 7):** `SELECT *`, `SELECT` specific columns, `SELECT DISTINCT`, `DISTINCT` on multiple columns, `ORDER BY`, `ASC`, `DESC`, multi-column sorting, `LIMIT`, `OFFSET`, pagination, logical processing order

> All examples use this `students` table:
> ```
> student_id | name   | age | city       | marks
> 1          | Aman   | 20  | Mohali     | 80
> 2          | Ravi   | 21  | Chandigarh | 95
> 3          | Priya  | 19  | Delhi      | 75
> 4          | Karan  | 22  | Mohali     | 90
> 5          | Simran | 20  | Chandigarh | 85
> 6          | Arjun  | 21  | Mohali     | 70
> ```

---

## 1. `SELECT *`

**Definition** — Retrieves **all columns** from a table (not necessarily all rows — that depends on filtering, covered in Part 8).

**Syntax**
```sql
SELECT * FROM table_name;
```

**Example**
```sql
SELECT * FROM students;
```
**Output** — all 6 rows, all 5 columns.

⚠️ **Notes & Caveats** — `*` means *all columns*, not *all rows*. All rows appear here simply because no `WHERE` condition was applied.

💡 **How to Choose** — `SELECT *` is fine for quick, ad-hoc exploration. In application code, prefer naming exact columns — clearer intent, less wasted bandwidth, and safer against future schema changes.

---

## 2. `SELECT column, column` (Specific Columns)

**Definition** — Retrieves only the named columns, in the order they're listed.

**Syntax**
```sql
SELECT col1, col2 FROM table_name;
```

**Example**
```sql
SELECT name, city FROM students;
```
**Output**
```
name    | city
Aman    | Mohali
Ravi    | Chandigarh
Priya   | Delhi
...
```

⚠️ **Notes & Caveats** — The **output column order follows the SELECT list**, not the table's original column order.
```sql
SELECT marks, name FROM students;   -- marks appears first, regardless of table definition
```

---

## 3. `SELECT DISTINCT`

**Definition** — Removes duplicate rows from the result.

**Syntax**
```sql
SELECT DISTINCT column FROM table_name;
```

**Example**
```sql
SELECT DISTINCT city FROM students;
```
**Output**
```
city
Mohali
Chandigarh
Delhi
```
*(Without `DISTINCT`, `Mohali` and `Chandigarh` would each repeat multiple times.)*

💡 **How to Choose** — Use `DISTINCT` when you need the set of unique values/combinations, not a full listing. Overusing it on large result sets can be costly (it typically requires a sort/hash step) — only apply it where duplicates genuinely need removing.

---

## 4. `DISTINCT` with Multiple Columns

**Definition** — `DISTINCT` on multiple columns removes duplicate **combinations**, not each column independently.

**Example**
```sql
SELECT DISTINCT age, city FROM students;
```

Given:
```
name   | age | city
Aman   | 20  | Mohali
Ravi   | 20  | Mohali
Priya  | 21  | Delhi
```
**Output**
```
age | city
20  | Mohali
21  | Delhi
```
*(Aman and Ravi collapse into one row because their `(age, city)` combination is identical.)*

❌ **Common Mistakes** — Thinking `SELECT DISTINCT age, city` gives *"unique ages"* and *"unique cities"* separately. It gives unique `(age + city)` **pairs**.

⚠️ **Notes & Caveats** — `SELECT DISTINCT *` only collapses rows that are identical across **every selected column**. If a hidden primary key column differs between two otherwise-identical rows, they will **not** be merged.

---

## 5. `ORDER BY`

**Definition** — Sorts the query result by one or more columns.

**Why It Exists** — Without `ORDER BY`, PostgreSQL does **not guarantee** any particular row order.

**Syntax**
```sql
SELECT columns FROM table_name ORDER BY column;
```

**Example**
```sql
SELECT name, marks FROM students ORDER BY marks;
```
**Output**
```
name   | marks
Arjun  | 70
Priya  | 75
Aman   | 80
Simran | 85
Karan  | 90
Ravi   | 95
```

⚠️ **Notes & Caveats** — `ORDER BY marks` is shorthand for `ORDER BY marks ASC` — ascending is the default direction.

---

## 6. `ASC` (Ascending)

**Definition** — Sorts smallest → largest (numbers), A → Z (text), oldest → newest (dates). This is the default.

```sql
SELECT name, marks FROM students ORDER BY marks ASC;
```

---

## 7. `DESC` (Descending)

**Definition** — Sorts largest → smallest, Z → A, newest → oldest.

```sql
SELECT name, marks FROM students ORDER BY marks DESC;
```
**Output**
```
name   | marks
Ravi   | 95
Karan  | 90
Simran | 85
Aman   | 80
Priya  | 75
Arjun  | 70
```

**Comparison**

| | `ASC` | `DESC` |
|---|---|---|
| Numbers | Small → Large | Large → Small |
| Text | A → Z | Z → A |
| Dates | Old → New | New → Old |
| Default? | ✅ Yes | Must be written explicitly |

💡 A very common pattern — **highest value**:
```sql
SELECT name, marks FROM students ORDER BY marks DESC LIMIT 1;
```

---

## 8. `ORDER BY` with Multiple Columns

**Definition** — Sorts by the first column; when values tie, the second column breaks the tie; and so on.

**Syntax**
```sql
SELECT columns FROM table_name ORDER BY col1 ASC, col2 DESC;
```

**Example**
```sql
SELECT name, city, marks
FROM students
ORDER BY city ASC, marks DESC;
```
**Output**
```
name   | city       | marks
Ravi   | Chandigarh | 95
Simran | Chandigarh | 85
Priya  | Delhi      | 75
Karan  | Mohali     | 90
Aman   | Mohali     | 80
Arjun  | Mohali     | 70
```
*(Sorted by city first; within each city, by marks descending.)*

⚠️ **Notes & Caveats** — Column **order in `ORDER BY` matters**: `ORDER BY city, marks` and `ORDER BY marks, city` produce different results — the first listed column is always the primary sort key.

💡 **How to Choose** — Think of it like a school register: sort by class first, then by marks *within* each class.

---

## 9. `LIMIT`

**Definition** — Caps the maximum number of rows returned.

**Syntax**
```sql
SELECT columns FROM table_name LIMIT n;
```

**Example**
```sql
SELECT * FROM students LIMIT 3;
```
**Output** — the first 3 rows in whatever order the query returns them.

⚠️ **Notes & Caveats** — `LIMIT` alone does **not** mean "top N by some ranking" — it just cuts off however many rows come back. For "top 3 highest marks," you **must** combine it with `ORDER BY`:
```sql
SELECT name, marks FROM students ORDER BY marks DESC LIMIT 3;
```

---

## 10. `OFFSET`

**Definition** — Skips a specified number of rows before returning results.

**Syntax**
```sql
SELECT columns FROM table_name OFFSET n;
```

**Example**
```sql
SELECT * FROM students ORDER BY student_id OFFSET 2;
```
Skips the first 2 rows, returns rows 3 onward.

---

## 11. `LIMIT` + `OFFSET` Together (Pagination)

**Definition** — Skip `OFFSET` rows, then take `LIMIT` rows — the standard pattern for paginated results.

**Syntax**
```sql
SELECT columns FROM table_name
ORDER BY column
LIMIT n OFFSET m;
```

**Example — 10 rows per page**
```sql
-- Page 1
SELECT * FROM students ORDER BY student_id LIMIT 10 OFFSET 0;
-- Page 2
SELECT * FROM students ORDER BY student_id LIMIT 10 OFFSET 10;
-- Page 3
SELECT * FROM students ORDER BY student_id LIMIT 10 OFFSET 20;
```

**Formula**
```
OFFSET = (page_number - 1) × rows_per_page
```

⚠️ **Notes & Caveats** — Always pair `OFFSET`/`LIMIT` with `ORDER BY`. Without a defined order, PostgreSQL doesn't guarantee which rows count as "first" or "next," so pagination becomes unreliable.

💡 **Best Practices / How to Choose** — For very large tables, plain `OFFSET` pagination gets slower on later pages (it still has to scan past all skipped rows). For high-performance pagination, "keyset pagination" (`WHERE id > last_seen_id ORDER BY id LIMIT n`) is often preferred — worth knowing exists, even if `OFFSET` is fine for smaller datasets.

---

## 12. The Logical Query Pipeline (Mental Model)

Even though you *write* `SELECT` first, PostgreSQL logically processes clauses in this order:

```
FROM        → get the table
  ↓
WHERE       → filter rows           (Part 8)
  ↓
SELECT      → choose columns
  ↓
DISTINCT    → remove duplicate result rows
  ↓
ORDER BY    → sort
  ↓
OFFSET      → skip rows
  ↓
LIMIT       → take rows
```

**Combined example**
```sql
SELECT DISTINCT name, marks
FROM students
WHERE city = 'Mohali'
ORDER BY marks DESC
LIMIT 2
OFFSET 0;
```

---

## 13. Interview Q&A

**Q: Does `SELECT *` also mean "all rows"?**
A: No — `*` only means all *columns*. Rows are controlled separately by `WHERE`; without a `WHERE` clause, all rows simply happen to pass through.

**Q: Is row order guaranteed without `ORDER BY`?**
A: No. PostgreSQL may return rows in any order unless `ORDER BY` is explicitly specified.

**Q: How do you get the "top 3 highest salaries"?**
A: `ORDER BY salary DESC LIMIT 3` — sort descending first, then take the first 3.

**Q: What's the difference between `LIMIT` and `OFFSET`?**
A: `LIMIT` controls how many rows to return; `OFFSET` controls how many rows to skip before returning results.

**Q: Why should `OFFSET`/`LIMIT` pagination always include `ORDER BY`?**
A: Without a defined order, PostgreSQL doesn't guarantee a stable row sequence, so "page 2" could overlap or skip rows inconsistently between queries.

**Q: `SELECT DISTINCT age, city` — does this give unique ages and unique cities separately?**
A: No — it gives unique `(age, city)` **combinations**. Two rows are only collapsed if every selected column matches.

**Q: What's the formula for calculating `OFFSET` for page N with page size P?**
A: `OFFSET = (N - 1) × P`.

---

## 14. Quick Revision Sheet

| Goal | Clause |
|---|---|
| All columns | `SELECT *` |
| Specific columns | `SELECT col1, col2` |
| Remove duplicates | `SELECT DISTINCT ...` |
| Sort ascending (default) | `ORDER BY col` / `ORDER BY col ASC` |
| Sort descending | `ORDER BY col DESC` |
| Sort, tie-break | `ORDER BY col1, col2 DESC` |
| Cap row count | `LIMIT n` |
| Skip rows | `OFFSET n` |
| Page N, size P | `LIMIT P OFFSET (N-1)*P` |

---

## 15. Cheat Sheet

```sql
-- ── SELECT BASICS ────────────────────────
SELECT * FROM students;                        -- all columns
SELECT name, city FROM students;                -- specific columns
SELECT DISTINCT city FROM students;              -- unique values
SELECT DISTINCT age, city FROM students;         -- unique combinations

-- ── SORTING ──────────────────────────────
SELECT name, marks FROM students ORDER BY marks;          -- ASC (default)
SELECT name, marks FROM students ORDER BY marks DESC;      -- DESC
SELECT name, city, marks FROM students
    ORDER BY city ASC, marks DESC;                          -- multi-column

-- ── LIMIT / OFFSET ───────────────────────
SELECT * FROM students LIMIT 3;                  -- first 3 rows (unordered result)
SELECT * FROM students ORDER BY marks DESC LIMIT 3;   -- top 3 by marks
SELECT * FROM students ORDER BY student_id OFFSET 2;   -- skip first 2
SELECT * FROM students ORDER BY student_id LIMIT 10 OFFSET 10;  -- page 2, size 10

-- ── FULL PIPELINE EXAMPLE ────────────────
SELECT DISTINCT name, marks
FROM students
WHERE city = 'Mohali'
ORDER BY marks DESC
LIMIT 2 OFFSET 0;
```

---

## 16. Preview of Part 8

| Topic | What You'll Learn |
|---|---|
| `WHERE` | Filtering rows |
| Comparison operators | `=`, `!=`/`<>`, `>`, `<`, `>=`, `<=` |
| `AND`/`OR`/`NOT` | Combining conditions |
| `IN`, `BETWEEN` | List and range checks |
| `LIKE`/`ILIKE` | Pattern matching |
| `NULL` handling | `IS NULL`, `COALESCE()`, `NULLIF()` |
