# SQL & PostgreSQL Complete Notes — Part 3: Table Commands (CREATE, DROP, TRUNCATE, ALTER)

## 📑 Table of Contents
1. `CREATE TABLE`
2. `DROP TABLE` (+ `CASCADE` / `RESTRICT`)
3. `TRUNCATE TABLE` (+ Identity & `CASCADE`)
4. `DROP` vs `TRUNCATE` (Comparison)
5. `ALTER TABLE` — The Complete Toolkit
6. `CREATE` vs `ALTER` vs `TRUNCATE` vs `DROP` (Master Comparison)
7. Interview Q&A
8. Quick Revision Sheet
9. Cheat Sheet
10. Preview of Part 4

**📋 Series Coverage (Part 3):** `CREATE TABLE` (+ `IF NOT EXISTS`, inline constraints), `DROP TABLE` (+ `CASCADE`/`RESTRICT`), `TRUNCATE TABLE` (+ `RESTART IDENTITY`/`CONTINUE IDENTITY`, `CASCADE`), `ALTER TABLE` — add/drop/rename column, change data type (`USING`), `SET`/`DROP NOT NULL`, `SET`/`DROP DEFAULT`, add/drop/rename constraint, rename table

---

## 1. `CREATE TABLE`

**Definition** — Creates a new table with a defined set of columns and, optionally, constraints.

**Why It Exists** — A database needs actual tables to store rows; `CREATE TABLE` is how you define their shape.

**Syntax**
```sql
CREATE TABLE table_name (
    column1 datatype constraints,
    column2 datatype constraints,
    ...
);
```

**Parameters**

| Name | Purpose | Default | Example |
|---|---|---|---|
| `table_name` | Name of the new table | — | `students` |
| `column datatype` | Column name + its data type | — | `age INT` |
| `constraints` | Rules on the column (optional) | none | `NOT NULL`, `UNIQUE` |

**Example**
```sql
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(100) UNIQUE,
    age        INT CHECK (age >= 18),
    city       VARCHAR(50) DEFAULT 'Mohali'
);
```
**Output**
```
CREATE TABLE
```
*(Table now exists with 0 rows.)*

⚠️ **Notes & Caveats**
- Running `CREATE TABLE` on a name that already exists throws `relation "students" already exists`.
- Use `CREATE TABLE IF NOT EXISTS students (...)` to avoid that error when re-running setup scripts.

❌ **Common Mistakes**
```sql
-- ❌ Re-running this on an existing table errors:
CREATE TABLE students (student_id INT);
```
```sql
-- ✅ Safe to re-run:
CREATE TABLE IF NOT EXISTS students (student_id INT);
```

💡 **Best Practices / How to Choose**
- Always add a Primary Key — even for "throwaway" tables. It saves debugging pain later.
- Name constraints explicitly (`CONSTRAINT pk_students PRIMARY KEY (...)`) so errors are easy to trace back (details in the Constraints part).

---

## 2. `DROP TABLE` (+ `CASCADE` / `RESTRICT`)

**Definition** — Permanently deletes a table, including its structure, constraints, and all data.

**Why It Exists** — Removes tables that are no longer needed.

**Syntax**
```sql
DROP TABLE table_name;
DROP TABLE IF EXISTS table_name;
DROP TABLE table_name CASCADE;
DROP TABLE table_name RESTRICT;    -- default behaviour
```

**Example**
```sql
DROP TABLE IF EXISTS students;
```
**Output**
```
DROP TABLE
```

**`CASCADE` vs `RESTRICT`**

Suppose `marks` has a foreign key referencing `students`:
```sql
DROP TABLE students;
-- ERROR: cannot drop table students because other objects depend on it
```

| Option | Behaviour |
|---|---|
| `RESTRICT` (default) | Refuses to drop if anything depends on the table |
| `CASCADE` | Drops the table **and** removes dependent objects (e.g., the foreign key constraint on `marks`) |

```sql
DROP TABLE students CASCADE;
```

⚠️ **Notes & Caveats**
- `CASCADE` does **not** necessarily delete the *dependent table itself* (e.g., `marks` may survive) — it removes the dependency/constraint tied to the dropped table. Always check what actually depends on a table before using `CASCADE` in production.

❌ **Common Mistakes**
```sql
-- ❌ Blocked by a dependent foreign key
DROP TABLE students;
```
```sql
-- ✅ Explicitly acknowledge the dependency
DROP TABLE students CASCADE;
```

💡 **Best Practices / How to Choose** — Use plain `DROP TABLE` (implicit `RESTRICT`) by default so PostgreSQL warns you about dependencies; only add `CASCADE` once you've confirmed what will be affected.

---

## 3. `TRUNCATE TABLE` (+ Identity & `CASCADE`)

**Definition** — Removes **all rows** from a table but keeps the table structure (columns, constraints) intact.

**Why It Exists** — A fast way to empty a large table without dropping and recreating it.

**Syntax**
```sql
TRUNCATE TABLE table_name;
TRUNCATE TABLE table_name RESTART IDENTITY;
TRUNCATE TABLE table_name CONTINUE IDENTITY;  -- default
TRUNCATE TABLE table_name CASCADE;
TRUNCATE TABLE t1, t2, t3;                    -- multiple tables at once
```

**Example**
```sql
TRUNCATE TABLE students;
```
**Output**
```
TRUNCATE TABLE
```
*(0 rows remain; columns/constraints still exist.)*

**Parameters**

| Option | Purpose |
|---|---|
| `RESTART IDENTITY` | Also resets any `SERIAL`/`IDENTITY` counter back to its starting value |
| `CONTINUE IDENTITY` | (Default) Keeps the current counter — next insert continues from where it left off |
| `CASCADE` | Also truncates tables that have FK dependencies requiring it |

**Example — Identity reset matters**
```sql
-- students has SERIAL student_id, currently at 3 after 3 inserts
TRUNCATE TABLE students;                 -- rows gone, but counter still at 3
INSERT INTO students (name) VALUES ('Karan');
-- student_id = 4, NOT 1!

TRUNCATE TABLE students RESTART IDENTITY;
INSERT INTO students (name) VALUES ('Karan');
-- student_id = 1 ✅
```

⚠️ **Notes & Caveats**
- `TABLE` is technically optional (`TRUNCATE students;` works), but writing it out is clearer while learning.
- `TRUNCATE ... CASCADE` can empty **dependent tables too** (unlike `DROP TABLE CASCADE`, which only removes the *constraint*, not the dependent table's data) — be careful.

❌ **Common Mistakes**
```sql
-- ❌ WHERE is not valid with TRUNCATE
TRUNCATE TABLE students WHERE city = 'Mohali';
```
```sql
-- ✅ Use DELETE instead for row-level filtering (covered in Part 8)
DELETE FROM students WHERE city = 'Mohali';
```

💡 **Best Practices / How to Choose** — Use `TRUNCATE` when you want to wipe an *entire* table quickly (e.g., resetting test/staging data); use `DELETE ... WHERE` when you need to remove only some rows.

---

## 4. `DROP` vs `TRUNCATE` (Comparison)

| | `DROP TABLE` | `TRUNCATE TABLE` |
|---|---|---|
| Table itself | Deleted | Kept |
| Columns/constraints | Deleted | Kept |
| Rows/data | Deleted | Deleted |
| Can filter with `WHERE`? | N/A | ❌ No |
| Speed on huge tables | N/A (gone entirely) | ✅ Very fast |
| Resets identity counter? | N/A | Only with `RESTART IDENTITY` |

💡 **Memory trick:** DROP = destroy the box 📦💥 · TRUNCATE = empty the box 📦

---

## 5. `ALTER TABLE` — The Complete Toolkit

**Definition** — `ALTER TABLE` modifies the structure or properties of an *existing* table — add/remove columns, change types, add/remove constraints, rename things — all without losing existing data (unless you explicitly drop something).

**Why It Exists** — Schemas evolve. You rarely want to drop and rebuild a table just to add one column.

**General Syntax**
```sql
ALTER TABLE table_name
    action1,
    action2,
    ...;
```

**Syntax Variants (all under one `ALTER TABLE`)**

| Goal | Syntax |
|---|---|
| Add a column | `ADD COLUMN col_name datatype;` |
| Add multiple columns | `ADD COLUMN a ..., ADD COLUMN b ...;` (comma-separated) |
| Drop a column | `DROP COLUMN col_name;` / `DROP COLUMN col_name CASCADE;` |
| Rename a column | `RENAME COLUMN old_name TO new_name;` |
| Change data type | `ALTER COLUMN col_name TYPE new_type;` |
| Change type + convert existing values | `ALTER COLUMN col_name TYPE new_type USING col_name::new_type;` |
| Make a value compulsory | `ALTER COLUMN col_name SET NOT NULL;` |
| Make a value optional again | `ALTER COLUMN col_name DROP NOT NULL;` |
| Add a default | `ALTER COLUMN col_name SET DEFAULT value;` |
| Remove a default | `ALTER COLUMN col_name DROP DEFAULT;` |
| Add a named constraint | `ADD CONSTRAINT name UNIQUE/CHECK/PRIMARY KEY (...)` |
| Drop a constraint | `DROP CONSTRAINT constraint_name;` |
| Rename a constraint | `RENAME CONSTRAINT old_name TO new_name;` |
| Rename the table | `RENAME TO new_table_name;` (PostgreSQL — **not** MySQL's `RENAME TABLE ...`) |

**Full Walkthrough Example**

```sql
-- Start:
CREATE TABLE students (
    student_id INT,
    name       VARCHAR(100),
    age        INT
);

-- Add a column
ALTER TABLE students ADD COLUMN email VARCHAR(100);

-- Add two columns at once
ALTER TABLE students
    ADD COLUMN city VARCHAR(50),
    ADD COLUMN is_active BOOLEAN DEFAULT TRUE;

-- Drop a column
ALTER TABLE students DROP COLUMN city;

-- Rename a column
ALTER TABLE students RENAME COLUMN name TO student_name;

-- Change a data type
ALTER TABLE students ALTER COLUMN age TYPE SMALLINT;

-- Change a type that needs conversion (e.g., TEXT '20' → INT 20)
ALTER TABLE students ALTER COLUMN age TYPE INT USING age::INT;

-- Require a value
ALTER TABLE students ALTER COLUMN student_name SET NOT NULL;

-- Make it optional again
ALTER TABLE students ALTER COLUMN student_name DROP NOT NULL;

-- Add / remove a default
ALTER TABLE students ALTER COLUMN is_active SET DEFAULT TRUE;
ALTER TABLE students ALTER COLUMN is_active DROP DEFAULT;

-- Add a named UNIQUE constraint
ALTER TABLE students ADD CONSTRAINT uq_email UNIQUE (email);

-- Add a named CHECK constraint
ALTER TABLE students ADD CONSTRAINT chk_age CHECK (age >= 18);

-- Add a Primary Key
ALTER TABLE students ADD CONSTRAINT pk_students PRIMARY KEY (student_id);

-- Drop a constraint (by name — not by definition)
ALTER TABLE students DROP CONSTRAINT uq_email;

-- Rename a constraint
ALTER TABLE students RENAME CONSTRAINT chk_age TO check_student_age;

-- Rename the table itself
ALTER TABLE students RENAME TO college_students;
```
**Output** (for each statement)
```
ALTER TABLE
```

⚠️ **Notes & Caveats**
- To drop a constraint you need its **name** — this is exactly why naming constraints explicitly (Section on Constraints, later) pays off.
- `DROP COLUMN ... CASCADE` also removes anything depending on that column (e.g., a view referencing it).
- MySQL has a dedicated `RENAME TABLE old TO new;` statement; PostgreSQL uses `ALTER TABLE old RENAME TO new;` instead.
- Changing a column's type across incompatible data (e.g., `VARCHAR '20'` → `INT`) needs `USING column::new_type` to tell PostgreSQL *how* to convert existing values.

❌ **Common Mistakes**
```sql
-- ❌ Wrong: dropping a constraint by its definition
ALTER TABLE students DROP CONSTRAINT UNIQUE(email);
```
```sql
-- ✅ Correct: drop by the constraint's actual name
ALTER TABLE students DROP CONSTRAINT uq_email;
```
```sql
-- ❌ Fails on non-numeric-looking text without a cast
ALTER TABLE students ALTER COLUMN age TYPE INT;   -- age currently VARCHAR '20'
```
```sql
-- ✅ Works — tells Postgres how to convert
ALTER TABLE students ALTER COLUMN age TYPE INT USING age::INT;
```

💡 **Best Practices**
- Batch related column changes into a single `ALTER TABLE` statement (comma-separated) instead of many separate statements — faster and clearer.
- Always name constraints when creating them, so future `DROP CONSTRAINT` / error messages are unambiguous.
- Practice the operations in this order to build muscle memory: `ADD COLUMN` → `DROP COLUMN` → `RENAME COLUMN` → `TYPE` → `SET/DROP NOT NULL` → `SET/DROP DEFAULT` → `ADD/DROP/RENAME CONSTRAINT` → `RENAME TABLE`.

---

## 6. `CREATE` vs `ALTER` vs `TRUNCATE` vs `DROP` (Master Comparison)

| Command | Table | Structure | Data |
|---|---|---|---|
| `CREATE TABLE` | Created | Defined | 0 rows |
| `ALTER TABLE` | Kept | Modified 🔧 | Kept (unless a dropped column takes data with it) |
| `TRUNCATE TABLE` | Kept | Kept | Deleted 🗑️ |
| `DROP TABLE` | Deleted 💥 | Deleted | Deleted |

💡 **Memory trick:** CREATE = build the box 📦 · ALTER = modify the box 🔧 · TRUNCATE = empty the box 🗑️ · DROP = destroy the box 💥 · a table **rename** = change the label on the box 🏷️

---

## 7. Interview Q&A

**Q: `DROP` vs `TRUNCATE` vs `DELETE` — what's the core difference?**
A: `DROP` removes the table entirely (structure + data). `TRUNCATE` empties all rows but keeps structure, and can't use `WHERE`. `DELETE` (covered later) removes rows matching a `WHERE` condition, can be part of a transaction, and fires triggers — but is slower on huge tables than `TRUNCATE`.

**Q: When would you use `CASCADE` with `DROP TABLE`?**
A: When other tables have foreign keys referencing the table you're dropping, and you want PostgreSQL to also remove those dependent constraints instead of blocking the drop.

**Q: What does `TRUNCATE ... RESTART IDENTITY` do differently from plain `TRUNCATE`?**
A: Plain `TRUNCATE` (or `CONTINUE IDENTITY`) empties the table but keeps the `SERIAL`/`IDENTITY` sequence counter where it was. `RESTART IDENTITY` also resets that counter, so the next insert starts back at 1.

**Q: How do you change a column's data type when the existing data needs conversion?**
A: Use `ALTER COLUMN col TYPE new_type USING col::new_type;` — the `USING` clause tells PostgreSQL how to cast existing values.

**Q: Difference between `DROP COLUMN` and `DROP COLUMN ... CASCADE`?**
A: Plain `DROP COLUMN` fails if something else (like a view) depends on that column. `CASCADE` drops the column and any dependent objects along with it.

**Q: Why use `IF EXISTS` / `IF NOT EXISTS`?**
A: To make DDL scripts idempotent (safe to re-run) — `CREATE TABLE IF NOT EXISTS` avoids "already exists" errors, and `DROP TABLE IF EXISTS` avoids "does not exist" errors.

**Q: How do you rename a table in PostgreSQL vs MySQL?**
A: PostgreSQL: `ALTER TABLE old_name RENAME TO new_name;`. MySQL also supports a dedicated `RENAME TABLE old_name TO new_name;` statement, which PostgreSQL does not have.

**Q: What happens to constraints when you `TRUNCATE` a table?**
A: They remain — `TRUNCATE` only removes rows; the table's structure and constraints stay exactly as defined.

**Q: Can `ALTER TABLE` add multiple columns in a single statement?**
A: Yes — separate each `ADD COLUMN` action with a comma under one `ALTER TABLE table_name` clause.

**Q: Why should constraints be named explicitly?**
A: Unnamed constraints get auto-generated names that are hard to reference later; a clear name (e.g., `uq_email`) makes `DROP CONSTRAINT` and error messages unambiguous.

---

## 8. Quick Revision Sheet

| Task | Command |
|---|---|
| Create table | `CREATE TABLE t (...);` |
| Create if missing | `CREATE TABLE IF NOT EXISTS t (...);` |
| Drop table | `DROP TABLE IF EXISTS t;` |
| Drop with dependents | `DROP TABLE t CASCADE;` |
| Empty a table | `TRUNCATE TABLE t;` |
| Empty + reset IDs | `TRUNCATE TABLE t RESTART IDENTITY;` |
| Add column | `ALTER TABLE t ADD COLUMN c type;` |
| Drop column | `ALTER TABLE t DROP COLUMN c;` |
| Rename column | `ALTER TABLE t RENAME COLUMN a TO b;` |
| Change type | `ALTER TABLE t ALTER COLUMN c TYPE new USING c::new;` |
| Require value | `ALTER TABLE t ALTER COLUMN c SET NOT NULL;` |
| Set default | `ALTER TABLE t ALTER COLUMN c SET DEFAULT v;` |
| Add constraint | `ALTER TABLE t ADD CONSTRAINT name ...;` |
| Drop constraint | `ALTER TABLE t DROP CONSTRAINT name;` |
| Rename table | `ALTER TABLE t RENAME TO new_t;` |

---

## 9. Cheat Sheet

```sql
-- ── CREATE TABLE ─────────────────────────
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(100) UNIQUE,
    age        INT CHECK (age >= 18),
    city       VARCHAR(50) DEFAULT 'Mohali'
);
CREATE TABLE IF NOT EXISTS students (...);   -- safe re-run

-- ── DROP TABLE ───────────────────────────
DROP TABLE students;
DROP TABLE IF EXISTS students;
DROP TABLE students CASCADE;                  -- also drop dependent objects
DROP TABLE students RESTRICT;                 -- default: block if dependents exist

-- ── TRUNCATE TABLE ───────────────────────
TRUNCATE TABLE students;                      -- empty, keep identity counter
TRUNCATE TABLE students RESTART IDENTITY;     -- empty, reset counter to 1
TRUNCATE TABLE students CONTINUE IDENTITY;    -- default behaviour
TRUNCATE TABLE students CASCADE;              -- also empties dependent tables
TRUNCATE TABLE a, b, c;                       -- multiple tables at once

-- ── ALTER TABLE: COLUMNS ─────────────────
ALTER TABLE students ADD COLUMN email VARCHAR(100);
ALTER TABLE students ADD COLUMN a TEXT, ADD COLUMN b BOOLEAN DEFAULT TRUE;
ALTER TABLE students DROP COLUMN city;
ALTER TABLE students DROP COLUMN city CASCADE;
ALTER TABLE students RENAME COLUMN name TO student_name;
ALTER TABLE students ALTER COLUMN age TYPE SMALLINT;
ALTER TABLE students ALTER COLUMN age TYPE INT USING age::INT;
ALTER TABLE students ALTER COLUMN name SET NOT NULL;
ALTER TABLE students ALTER COLUMN name DROP NOT NULL;
ALTER TABLE students ALTER COLUMN city SET DEFAULT 'Mohali';
ALTER TABLE students ALTER COLUMN city DROP DEFAULT;

-- ── ALTER TABLE: CONSTRAINTS ─────────────
ALTER TABLE students ADD CONSTRAINT uq_email UNIQUE (email);
ALTER TABLE students ADD CONSTRAINT chk_age CHECK (age >= 18);
ALTER TABLE students ADD CONSTRAINT pk_students PRIMARY KEY (student_id);
ALTER TABLE students DROP CONSTRAINT uq_email;
ALTER TABLE students RENAME CONSTRAINT chk_age TO check_student_age;

-- ── ALTER TABLE: RENAME TABLE ────────────
ALTER TABLE students RENAME TO college_students;   -- PostgreSQL
-- RENAME TABLE students TO college_students;      -- MySQL only, not Postgres
```

---

## 10. Preview of Part 4

| Topic | What You'll Learn |
|---|---|
| Whole numbers | `SMALLINT`, `INTEGER`, `BIGINT` |
| Exact/approximate decimals | `NUMERIC`, `DECIMAL`, `REAL`, `FLOAT`, `DOUBLE PRECISION` |
| Text | `CHAR`, `VARCHAR`, `TEXT` |
| Boolean | `BOOLEAN` |
| Date & time | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ` |
