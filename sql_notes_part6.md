# SQL & PostgreSQL Complete Notes — Part 6: INSERT — Adding Data

## 📑 Table of Contents
1. Basic `INSERT` (Positional Values)
2. `INSERT` with Column Names
3. Inserting Only Some Columns
4. Using `DEFAULT` Explicitly
5. Multi-Row `INSERT`
6. `INSERT INTO ... SELECT`
7. Column Matching in `INSERT INTO SELECT` — By Position, Not Name
8. `INSERT INTO SELECT` with Calculations, `GROUP BY`, Window Functions, CTEs
9. Single-Row vs Multi-Row vs `INSERT INTO SELECT` (Comparison)
10. Interview Q&A
11. Quick Revision Sheet
12. Cheat Sheet
13. Preview of Part 7

**📋 Series Coverage (Part 6):** `INSERT INTO ... VALUES` (positional & named columns), partial inserts + `NULL`/`DEFAULT` fallback, explicit `DEFAULT` keyword, multi-row `INSERT`, `INSERT INTO ... SELECT` (including with `WHERE`, calculations, `GROUP BY`, window functions, CTEs)

---

## 1. Basic `INSERT` (Positional Values)

**Definition** — Adds a new row to a table by supplying values that match the table's column order.

**Why It Exists** — `CREATE TABLE` builds an empty structure; `INSERT` is how rows actually get into it.

**Syntax**
```sql
INSERT INTO table_name
VALUES (value1, value2, value3, ...);
```

**Example**
```sql
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100),
    age        INT,
    city       VARCHAR(50)
);

INSERT INTO students
VALUES (1, 'Aman', 20, 'Mohali');
```
**Output**
```
student_id | name | age | city
1          | Aman | 20  | Mohali
```

⚠️ **Notes & Caveats**
- Values are matched **strictly by position** against the table's column order — get the order wrong and you'll either get a type error or silently wrong data.

❌ **Common Mistakes**
```sql
-- ❌ Wrong order: 'Aman' lands in the INTEGER student_id column
INSERT INTO students VALUES ('Aman', 1, 20, 'Mohali');
```
```sql
-- ✅ Match the declared column order exactly
INSERT INTO students VALUES (1, 'Aman', 20, 'Mohali');
```

💡 **Best Practices / How to Choose** — Avoid positional-only inserts in real code; use the named-column form below instead (safer against future schema changes).

---

## 2. `INSERT` with Column Names

**Definition** — Explicitly lists which columns you're supplying values for, in whatever order you choose.

**Syntax**
```sql
INSERT INTO table_name (col1, col2, col3)
VALUES (value1, value2, value3);
```

**Example**
```sql
INSERT INTO students (name, age, city)
VALUES ('Aman', 20, 'Mohali');
-- student_id is omitted — PostgreSQL fills it automatically (SERIAL)
```
**Output**
```
student_id | name | age | city
1          | Aman | 20  | Mohali
```

**Column order can be freely rearranged as long as VALUES matches:**
```sql
INSERT INTO students (city, name, age)
VALUES ('Mohali', 'Aman', 20);   -- ✅ still correct
```

💡 **Best Practices / How to Choose** — **Always prefer this form** in real applications. If someone later runs `ALTER TABLE students ADD COLUMN email ...`, positional inserts silently break or misalign, while named-column inserts keep working correctly.

---

## 3. Inserting Only Some Columns

**Definition** — Any column you don't mention gets either its `DEFAULT` value, or `NULL` if it allows NULLs, or an error if it's `NOT NULL` with no default.

**Example**
```sql
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    age        INT,
    city       VARCHAR(50) DEFAULT 'Mohali'
);

INSERT INTO students (name, age) VALUES ('Aman', 20);
```
**Output**
```
student_id | name | age | city
1          | Aman | 20  | Mohali     ← used DEFAULT
```

**Decision logic**

```
Column not provided in INSERT
        ↓
   Has a DEFAULT?
   ├── YES → use the default value
   └── NO  → is it NOT NULL?
             ├── YES → ERROR
             └── NO  → use NULL
```

❌ **Common Mistakes**
```sql
-- ❌ 'name' is NOT NULL but wasn't provided
CREATE TABLE students (name VARCHAR(100) NOT NULL, age INT);
INSERT INTO students (age) VALUES (20);
-- ERROR: null value in column "name" violates not-null constraint
```

---

## 4. Using `DEFAULT` Explicitly

**Definition** — The keyword `DEFAULT` can be written directly inside `VALUES` to force PostgreSQL to use that column's default, even while listing the column.

**Syntax**
```sql
INSERT INTO table_name (col1, col2)
VALUES (value1, DEFAULT);
```

**Example**
```sql
INSERT INTO students (name, city)
VALUES ('Aman', DEFAULT);   -- city becomes 'Mohali'
```

---

## 5. Multi-Row `INSERT`

**Definition** — Adds several rows in a single statement instead of repeating `INSERT INTO ...` multiple times.

**Syntax**
```sql
INSERT INTO table_name (col1, col2, col3)
VALUES
    (row1_val1, row1_val2, row1_val3),
    (row2_val1, row2_val2, row2_val3),
    (row3_val1, row3_val2, row3_val3);
```

**Example**
```sql
INSERT INTO students (name, age, city)
VALUES
    ('Aman',  20, 'Mohali'),
    ('Ravi',  21, 'Chandigarh'),
    ('Priya', 19, 'Delhi'),
    ('Karan', 22, 'Amritsar');
```
**Output**
```
student_id | name  | age | city
1          | Aman  | 20  | Mohali
2          | Ravi  | 21  | Chandigarh
3          | Priya | 19  | Delhi
4          | Karan | 22  | Amritsar
```

⚠️ **Notes & Caveats**
- Each parenthesized group `(...)` is exactly one row. Rows are separated by commas; no comma after the final row.
- **Every row must supply the same number of values as the column list** — you can't skip a value mid-row (use `NULL` or `DEFAULT` explicitly instead).

❌ **Common Mistakes**
```sql
-- ❌ Row 2 is missing a value for 'city'
INSERT INTO students (name, age, city)
VALUES
    ('Aman', 20, 'Mohali'),
    ('Ravi', 21),                 -- only 2 values, needs 3
    ('Priya', 19, 'Delhi');
```
```sql
-- ✅ Be explicit about missing values
INSERT INTO students (name, age, city)
VALUES
    ('Aman', 20, 'Mohali'),
    ('Ravi', 21, NULL),
    ('Priya', 19, 'Delhi');
```

💡 **Best Practices / How to Choose** — Use multi-row `INSERT` whenever you're loading a known, fixed batch of rows — it's a single round-trip to the database and much faster than looping single inserts.

---

## 6. `INSERT INTO ... SELECT`

**Definition** — Takes the result of a `SELECT` query and inserts those rows directly into another table — no `VALUES` clause needed, because the `SELECT` **is** the data source.

**Why It Exists** — Avoids manually re-typing data that the database already has (e.g., copying/filtering/archiving rows, building summary tables).

**Syntax**
```sql
INSERT INTO destination_table (col1, col2, col3)
SELECT col_a, col_b, col_c
FROM source_table
WHERE condition;
```

**Example**
```sql
CREATE TABLE mohali_students (
    name VARCHAR(100),
    age  INT,
    city VARCHAR(50)
);

INSERT INTO mohali_students (name, age, city)
SELECT name, age, city
FROM students
WHERE city = 'Mohali';
```
**Output**
```
name    | age | city
Aman    | 20  | Mohali
Karan   | 22  | Mohali
Simran  | 20  | Mohali
```

⚠️ **Notes & Caveats**
- No `VALUES` keyword — the `SELECT` result itself supplies the rows.
- The number of columns in the destination list **must match** the number of columns returned by `SELECT`.
- Data types must be compatible position-by-position.

---

## 7. Column Matching in `INSERT INTO SELECT` — By Position, Not Name

This is the single most important rule for this command.

```sql
INSERT INTO mohali_students (name, age, city)
SELECT city, age, name          -- ⚠️ columns swapped!
FROM students;
```

SQL does **not** match `name → name`. It matches by **position**:
```
destination.name  ← SELECT's 1st column (city)   ❌ wrong data, but no error
destination.age   ← SELECT's 2nd column (age)     ✅ correct by coincidence
destination.city  ← SELECT's 3rd column (name)    ❌ wrong data
```

❌ **Common Mistakes**
```sql
-- ❌ Column counts don't match: 3 destination columns, only 2 selected
INSERT INTO mohali_students (name, age, city)
SELECT name, age
FROM students;
-- ERROR: INSERT has more target columns than expressions
```

💡 **Best Practices** — Always list destination columns **and** select columns in the exact same conceptual order — never rely on matching names.

---

## 8. `INSERT INTO SELECT` with Calculations, `GROUP BY`, Window Functions, CTEs

**With a calculation**
```sql
CREATE TABLE employee_bonus (name VARCHAR(100), bonus NUMERIC(10,2));

INSERT INTO employee_bonus (name, bonus)
SELECT name, salary * 0.10
FROM employees;
```

**With `GROUP BY`**
```sql
CREATE TABLE department_summary (department VARCHAR(50), avg_salary NUMERIC(10,2));

INSERT INTO department_summary (department, avg_salary)
SELECT department, AVG(salary)
FROM employees
GROUP BY department;
```

**With a CTE**
```sql
WITH high_salary AS (
    SELECT name, salary FROM employees WHERE salary > 60000
)
INSERT INTO top_employees (name, salary)
SELECT name, salary FROM high_salary;
```

💡 **Notes & Caveats** — The `SELECT` half of `INSERT INTO SELECT` can use anything a normal `SELECT` can: `WHERE`, `GROUP BY`, joins, window functions, CTEs (covered later in this series). Whatever rows that `SELECT` would return are exactly the rows that get inserted.

---

## 9. Single-Row vs Multi-Row vs `INSERT INTO SELECT` (Comparison)

| Type | Data Source | Example |
|---|---|---|
| Single-row `INSERT` | You manually provide one row | `VALUES ('Aman', 20)` |
| Multi-row `INSERT` | You manually provide many rows | `VALUES ('Aman',20), ('Ravi',21)` |
| `INSERT INTO SELECT` | A query provides the rows | `SELECT name, age FROM students WHERE age > 20` |

---

## 10. Interview Q&A

**Q: What happens if you omit a column in `INSERT` that has no `DEFAULT` and is nullable?**
A: PostgreSQL stores `NULL` for that column.

**Q: What happens if you omit a `NOT NULL` column with no default?**
A: The insert fails with a not-null constraint violation error.

**Q: Why is naming columns explicitly in `INSERT` considered best practice?**
A: It protects the statement from breaking or silently misaligning if the table's column order changes later (e.g., after `ALTER TABLE ... ADD COLUMN`), and it makes the intent of the statement self-documenting.

**Q: In `INSERT INTO SELECT`, are columns matched by name or by position?**
A: Strictly by **position** — the first selected column fills the first destination column, and so on, regardless of column names.

**Q: Can `INSERT INTO SELECT` include a `WHERE` clause?**
A: Yes — the `SELECT` portion behaves like any normal query, so `WHERE`, `GROUP BY`, joins, and window functions can all be used to shape exactly which/how rows get inserted.

**Q: What's the difference between leaving a value out of a multi-row `INSERT` and explicitly writing `NULL`?**
A: You *can't* just leave a positional value out — every row must supply a value for every listed column. Writing `NULL` (or `DEFAULT`) explicitly is how you represent "no value" for that row/column.

**Q: Does `INSERT INTO SELECT` need a `VALUES` clause?**
A: No — the `SELECT` query itself supplies the data; combining `VALUES` and `SELECT` in the same statement isn't valid.

---

## 11. Quick Revision Sheet

| Goal | Pattern |
|---|---|
| Insert one row (all columns) | `INSERT INTO t VALUES (...);` |
| Insert one row (named columns) | `INSERT INTO t (a,b) VALUES (v1,v2);` |
| Insert many rows | `INSERT INTO t (a,b) VALUES (..),(..),(..);` |
| Force a column's default | `VALUES (v1, DEFAULT);` |
| Copy/derive rows from a query | `INSERT INTO t (a,b) SELECT x,y FROM s WHERE ...;` |

---

## 12. Cheat Sheet

```sql
-- ── SINGLE ROW ────────────────────────────
INSERT INTO students VALUES (1, 'Aman', 20, 'Mohali');           -- positional
INSERT INTO students (name, age, city) VALUES ('Aman', 20, 'Mohali');  -- named (preferred)
INSERT INTO students (name, city) VALUES ('Aman', DEFAULT);       -- force default

-- ── MULTIPLE ROWS ─────────────────────────
INSERT INTO students (name, age, city)
VALUES
    ('Aman',  20, 'Mohali'),
    ('Ravi',  21, 'Chandigarh'),
    ('Priya', 19, 'Delhi');

-- ── INSERT INTO SELECT ────────────────────
INSERT INTO mohali_students (name, age, city)
SELECT name, age, city
FROM students
WHERE city = 'Mohali';

INSERT INTO employee_bonus (name, bonus)
SELECT name, salary * 0.10 FROM employees;

INSERT INTO department_summary (department, avg_salary)
SELECT department, AVG(salary) FROM employees GROUP BY department;

WITH high_salary AS (
    SELECT name, salary FROM employees WHERE salary > 60000
)
INSERT INTO top_employees (name, salary)
SELECT name, salary FROM high_salary;
```

---

## 13. Preview of Part 7

| Topic | What You'll Learn |
|---|---|
| `SELECT *` vs specific columns | Reading data from a table |
| `DISTINCT` | Removing duplicate rows/combinations |
| `ORDER BY`, `ASC`/`DESC` | Sorting results |
| `LIMIT` / `OFFSET` | Capping rows returned, pagination |
