# SQL & PostgreSQL Complete Notes — Part 10: Constraints & Foreign Key Placement

## 📑 Table of Contents
1. What Are Constraints?
2. `PRIMARY KEY`
3. `FOREIGN KEY`
4. `UNIQUE`
5. `NOT NULL`
6. `CHECK`
7. `DEFAULT`
8. Constraint Comparison Table
9. Column-Level vs Table-Level Constraints & Naming
10. Parent & Child Tables
11. `ON DELETE CASCADE`
12. `ON DELETE SET NULL`
13. `ON DELETE CASCADE` vs `SET NULL` (Comparison)
14. `ON UPDATE CASCADE`
15. ⭐ Foreign Key Placement — Full Worked Examples (1:1, 1:N, M:N)
16. Complete Realistic Table (Every Constraint Together)
17. Interview Q&A
18. Quick Revision Sheet
19. Cheat Sheet
20. Preview of Part 11

**📋 Series Coverage (Part 10):** `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL`, `CHECK`, `DEFAULT`, column-level vs table-level syntax, named constraints, `ON DELETE CASCADE`/`SET NULL`, `ON UPDATE CASCADE`, foreign key placement for 1:1/1:N/M:N with full `CREATE TABLE` + `INSERT` + violation demonstrations

---

## 1. What Are Constraints?

**Definition** — Rules attached to columns or tables that control what data is allowed to be stored.

```
New data arrives
       ↓
Database checks constraints
       ↓
   Valid? ── NO ──▶ Reject (error) ❌
       │
      YES
       ↓
   Store data ✅
```

---

## 2. `PRIMARY KEY`

**Definition** — Uniquely identifies each row. Automatically implies `UNIQUE` **and** `NOT NULL`. Exactly one per table (can span multiple columns as a composite key).

**Syntax**
```sql
column_name datatype PRIMARY KEY                         -- column-level
CONSTRAINT pk_name PRIMARY KEY (column_name)              -- table-level, named
PRIMARY KEY (col1, col2)                                  -- composite
```

**Example**
```sql
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name       VARCHAR(100)
);
```

⚠️ **Notes & Caveats**
- `INSERT INTO students VALUES (1, 'Aman'); INSERT INTO students VALUES (1, 'Ravi');` → ❌ `duplicate key value violates unique constraint`.
- `INSERT INTO students VALUES (NULL, 'Aman');` → ❌ null value in column violates not-null constraint.

💡 **How to Choose** — Always define one, even on "temporary" tables — the debugging pain of a PK-less table (duplicate rows, no reliable way to update/delete a specific row) isn't worth the shortcut.

---

## 3. `FOREIGN KEY`

**Definition** — A column (or columns) in one table that must match a value already present in another table's `PRIMARY KEY` (or `UNIQUE`) column — establishing a relationship and preventing "orphan" rows.

**Syntax**
```sql
FOREIGN KEY (column_name) REFERENCES other_table(other_column)
```

**Example**
```sql
CREATE TABLE marks (
    mark_id    SERIAL PRIMARY KEY,
    student_id INT,
    subject    VARCHAR(50),
    marks      INT,
    FOREIGN KEY (student_id) REFERENCES students(student_id)
);
```

⚠️ **Notes & Caveats**
- The **referenced** column (`students.student_id`) must be unique (PK or `UNIQUE`) — otherwise "which row does this FK point to?" would be ambiguous.
- The **foreign key column itself does NOT need to be unique** — `marks.student_id` can repeat many times; that's exactly how a 1:N relationship works (one student, many marks rows).
- Inserting a `student_id` into `marks` that doesn't exist in `students` is rejected — this prevents "orphan records."

💡 **How to Choose** — Full placement rules (which side gets the FK, for each relationship type) are covered in depth in **Section 15** of this part.

---

## 4. `UNIQUE`

**Definition** — Prevents duplicate values in a column (or column combination), but — unlike `PRIMARY KEY` — allows `NULL` (PostgreSQL allows multiple `NULL`s in a `UNIQUE` column, since `NULL` is never considered equal to another `NULL`).

**Syntax**
```sql
column_name datatype UNIQUE
```

**Example**
```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email   VARCHAR(100) UNIQUE,
    phone   VARCHAR(15) UNIQUE
);
```

**`PRIMARY KEY` vs `UNIQUE`**

| | `PRIMARY KEY` | `UNIQUE` |
|---|---|---|
| Duplicates | ❌ Not allowed | ❌ Not allowed |
| `NULL` | ❌ Not allowed | ✅ Allowed (multiple, in Postgres) |
| Count per table | Exactly 1 | Many |

💡 **How to Choose** — Use `PRIMARY KEY` for the row's core identity; use `UNIQUE` for any *other* column that also must never repeat (email, phone, username).

---

## 5. `NOT NULL`

**Definition** — Makes a value compulsory — `NULL` is rejected.

**Syntax**
```sql
column_name datatype NOT NULL
```

**Example**
```sql
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL
);
```

⚠️ **Notes & Caveats** — `NOT NULL` does **not** imply uniqueness — two rows can both have `name = 'Aman'`.

---

## 6. `CHECK`

**Definition** — Ensures a value satisfies a boolean condition before it's accepted.

**Syntax**
```sql
column_name datatype CHECK (condition)             -- column-level
CHECK (condition)                                    -- table-level (can span columns)
```

**Example**
```sql
CREATE TABLE students (
    age   INT CHECK (age >= 18),
    marks INT CHECK (marks BETWEEN 0 AND 100)
);
```

**Multi-column table-level `CHECK`**
```sql
CREATE TABLE jobs (
    minimum_salary NUMERIC,
    maximum_salary NUMERIC,
    CHECK (minimum_salary <= maximum_salary)
);
```

⚠️ **Notes & Caveats** — A `CHECK` constraint only **rejects rows where the condition is explicitly `FALSE`**. If the column is `NULL`, the condition evaluates to `UNKNOWN` (not `FALSE`), so the row is **allowed** through — unless you also add `NOT NULL`.
```sql
age INT NOT NULL CHECK (age >= 18)   -- blocks both NULL and values < 18
```

---

## 7. `DEFAULT`

**Definition** — Supplies an automatic value when none is explicitly given in `INSERT`.

**Syntax**
```sql
column_name datatype DEFAULT value
```

**Example**
```sql
CREATE TABLE users (
    user_id    SERIAL PRIMARY KEY,
    is_active  BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

⚠️ **Notes & Caveats** — `DEFAULT` only applies when a value is **omitted** — explicitly inserting `FALSE` overrides the default without error.

---

## 8. Constraint Comparison Table

| Constraint | Main Purpose | Allows Duplicates? | Allows NULL? |
|---|---|---|---|
| `PRIMARY KEY` | Uniquely identify each row | ❌ | ❌ |
| `FOREIGN KEY` | Connect to another table | ✅ (on the FK side) | ✅ (unless also `NOT NULL`) |
| `UNIQUE` | Prevent duplicate values | ❌ | ✅ (multiple, in Postgres) |
| `NOT NULL` | Require a value | ✅ | ❌ |
| `CHECK` | Validate a condition | ✅ | ✅ (unless paired with `NOT NULL`) |
| `DEFAULT` | Auto-fill missing values | N/A | N/A |

---

## 9. Column-Level vs Table-Level Constraints & Naming

**Column-level** (constraint written right next to the column):
```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email   VARCHAR(100) UNIQUE,
    age     INT CHECK (age >= 18)
);
```

**Table-level** (constraints listed separately — required for composite keys and multi-column checks):
```sql
CREATE TABLE users (
    user_id INT,
    email   VARCHAR(100),
    age     INT,
    PRIMARY KEY (user_id),
    UNIQUE (email),
    CHECK (age >= 18)
);
```

**Named constraints** (recommended):
```sql
CREATE TABLE students (
    student_id INT,
    age        INT,
    CONSTRAINT pk_students    PRIMARY KEY (student_id),
    CONSTRAINT chk_student_age CHECK (age >= 18)
);
```

💡 **Best Practices** — Always name your constraints. `DROP CONSTRAINT` requires the name, and a clear name (`chk_student_age`) makes error messages self-explanatory (`violates check constraint "chk_student_age"`) instead of a cryptic auto-generated one.

---

## 10. Parent & Child Tables

| Term | Meaning | Example |
|---|---|---|
| **Parent table** | Holds the referenced key (usually the PK) | `students` |
| **Child table** | Holds the foreign key pointing back to the parent | `marks` |

```
PARENT                     CHILD
students.student_id  ◀────  marks.student_id
   (referenced)              (foreign key)
```

By default, PostgreSQL **blocks deleting a parent row** if child rows still reference it — the next two sections show how to change that behaviour.

---

## 11. `ON DELETE CASCADE`

**Definition** — When a parent row is deleted, automatically delete every child row that referenced it.

**Syntax**
```sql
FOREIGN KEY (col) REFERENCES parent(col) ON DELETE CASCADE
```

**Example**
```sql
CREATE TABLE marks (
    mark_id    SERIAL PRIMARY KEY,
    student_id INT,
    subject    VARCHAR(50),
    FOREIGN KEY (student_id) REFERENCES students(student_id) ON DELETE CASCADE
);

DELETE FROM students WHERE student_id = 1;
-- Aman's row in students is deleted, AND every marks row with student_id = 1
-- is automatically deleted too.
```

💡 **How to Choose** — Use when child rows **make no sense without** the parent (e.g., `order_items` without their `order`).

---

## 12. `ON DELETE SET NULL`

**Definition** — When a parent row is deleted, keep the child rows but set their foreign key column to `NULL`.

**Syntax**
```sql
FOREIGN KEY (col) REFERENCES parent(col) ON DELETE SET NULL
```

**Example**
```sql
CREATE TABLE projects (
    project_id SERIAL PRIMARY KEY,
    manager_id INT,
    FOREIGN KEY (manager_id) REFERENCES employees(emp_id) ON DELETE SET NULL
);

DELETE FROM employees WHERE emp_id = 5;
-- Manager 5 is deleted; their projects REMAIN, but manager_id becomes NULL
-- (meaning "currently unassigned")
```

⚠️ **Notes & Caveats** — The FK column **must allow `NULL`** for this to work — `manager_id INT NOT NULL` combined with `ON DELETE SET NULL` is a contradiction PostgreSQL can't satisfy.

---

## 13. `ON DELETE CASCADE` vs `SET NULL` (Comparison)

| | `ON DELETE CASCADE` | `ON DELETE SET NULL` |
|---|---|---|
| Child row | Deleted | Kept |
| Child's FK column | N/A (row is gone) | Set to `NULL` |
| Use when... | Child data is meaningless without the parent | Child should survive, just loses the association |
| Example | `orders` → `order_items` | `employees` → `projects.manager_id` |

**Default behaviour (no `ON DELETE` clause):** deleting the parent is **blocked** ("`RESTRICT`") if child rows still reference it.

---

## 14. `ON UPDATE CASCADE`

**Definition** — When the parent's **referenced key value itself changes**, automatically update matching child foreign key values too.

**Syntax**
```sql
FOREIGN KEY (col) REFERENCES parent(col) ON UPDATE CASCADE
```

**Example**
```sql
CREATE TABLE marks (
    mark_id    SERIAL PRIMARY KEY,
    student_id INT,
    FOREIGN KEY (student_id) REFERENCES students(student_id) ON UPDATE CASCADE
);

UPDATE students SET student_id = 100 WHERE student_id = 1;
-- students.student_id: 1 → 100
-- marks.student_id automatically updates: 1 → 100, keeping the relationship intact
```

⚠️ **Notes & Caveats** — IDs (`SERIAL`, `UUID`) rarely change once assigned, so this is used less often than `ON DELETE` actions — but it matters when a FK references a natural key that *could* change (e.g., a department code).

**Combining both:**
```sql
FOREIGN KEY (student_id) REFERENCES students(student_id)
    ON DELETE CASCADE
    ON UPDATE CASCADE
```

---

## 15. ⭐ Foreign Key Placement — Full Worked Examples

**The core rule (first introduced conceptually in Part 1):**

| Relationship | Where the FK Goes |
|---|---|
| **One-to-One (1:1)** | FK in either table — usually the dependent/"weaker" one — with a `UNIQUE` constraint on the FK column |
| **One-to-Many (1:N)** | FK **always** on the "many" side |
| **Many-to-Many (M:N)** | No FK in either original table — build a **junction table** holding FKs to both, usually with a composite Primary Key |

---

### 15a. One-to-One (1:1) — `users` ↔ `user_profiles`

```sql
CREATE TABLE users (
    user_id  SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE user_profiles (
    profile_id SERIAL PRIMARY KEY,
    user_id    INT UNIQUE NOT NULL,     -- ⭐ UNIQUE is what makes this 1:1, not 1:N
    bio        TEXT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

**✅ Inserting data that respects the relationship**
```sql
INSERT INTO users (username) VALUES ('aman123');                 -- user_id = 1
INSERT INTO user_profiles (user_id, bio) VALUES (1, 'Loves SQL'); -- ✅ OK, one profile
```

**❌ What happens if you violate it**
```sql
-- Trying to give the same user a SECOND profile:
INSERT INTO user_profiles (user_id, bio) VALUES (1, 'Second profile');
-- ERROR: duplicate key value violates unique constraint "user_profiles_user_id_key"
-- DETAIL: Key (user_id)=(1) already exists.

-- Trying to attach a profile to a user that doesn't exist:
INSERT INTO user_profiles (user_id, bio) VALUES (999, 'Ghost user');
-- ERROR: insert or update on table "user_profiles" violates foreign key constraint
-- DETAIL: Key (user_id)=(999) is not present in table "users".
```

💡 Without the `UNIQUE` on `user_profiles.user_id`, this would just be a normal 1:N relationship (one user could have many profiles) — `UNIQUE` is the piece that enforces "at most one."

---

### 15b. One-to-Many (1:N) — `departments` → `employees`

```sql
CREATE TABLE departments (
    department_id   SERIAL PRIMARY KEY,
    department_name VARCHAR(100) NOT NULL
);

CREATE TABLE employees (
    emp_id        SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    department_id INT,                    -- ⭐ FK on the "many" side — NOT unique
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

**✅ Inserting data that respects the relationship**
```sql
INSERT INTO departments (department_name) VALUES ('IT');          -- department_id = 1
INSERT INTO employees (name, department_id) VALUES ('Aman', 1);   -- ✅
INSERT INTO employees (name, department_id) VALUES ('Priya', 1);  -- ✅ same dept, OK!
```
*(Two employees sharing `department_id = 1` is expected and correct — that's the "many" in one-to-many.)*

**❌ What happens if you violate it**
```sql
INSERT INTO employees (name, department_id) VALUES ('Ghost', 999);
-- ERROR: insert or update on table "employees" violates foreign key constraint
-- DETAIL: Key (department_id)=(999) is not present in table "departments".
```

---

### 15c. Many-to-Many (M:N) — `students` ↔ `courses`

```sql
CREATE TABLE students (
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL
);

CREATE TABLE courses (
    course_id   SERIAL PRIMARY KEY,
    course_name VARCHAR(100) NOT NULL
);

-- ⭐ Junction / bridge table — holds FKs to BOTH parents
CREATE TABLE student_courses (
    student_id  INT,
    course_id   INT,
    enrolled_on DATE DEFAULT CURRENT_DATE,
    PRIMARY KEY (student_id, course_id),          -- composite PK: no duplicate enrollments
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id)  REFERENCES courses(course_id)
);
```

**✅ Inserting data that respects the relationship**
```sql
INSERT INTO students (name) VALUES ('Aman'), ('Ravi');           -- ids 1, 2
INSERT INTO courses (course_name) VALUES ('SQL'), ('Python');     -- ids 1, 2

INSERT INTO student_courses (student_id, course_id) VALUES (1, 1);  -- Aman → SQL
INSERT INTO student_courses (student_id, course_id) VALUES (1, 2);  -- Aman → Python too
INSERT INTO student_courses (student_id, course_id) VALUES (2, 1);  -- Ravi → SQL
```

**❌ What happens if you violate it**
```sql
-- Enrolling Aman in SQL a second time (violates the composite PK):
INSERT INTO student_courses (student_id, course_id) VALUES (1, 1);
-- ERROR: duplicate key value violates unique constraint "student_courses_pkey"
-- DETAIL: Key (student_id, course_id)=(1, 1) already exists.

-- Enrolling in a course that doesn't exist:
INSERT INTO student_courses (student_id, course_id) VALUES (1, 999);
-- ERROR: insert or update on table "student_courses" violates foreign key constraint
-- DETAIL: Key (course_id)=(999) is not present in table "courses".
```

💡 **Why a junction table is mandatory for M:N** — a single FK column can only point to *one* row. Since both a student can take many courses **and** a course can have many students, neither `students` nor `courses` can hold a single FK column that captures the relationship — you need a third table whose entire job is to record each valid pairing.

---

## 16. Complete Realistic Table (Every Constraint Together)

```sql
CREATE TABLE departments (
    department_id   SERIAL PRIMARY KEY,
    department_name VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE employees (
    emp_id        SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(100) UNIQUE NOT NULL,
    age           INT CHECK (age >= 18),
    salary        NUMERIC(10,2) CHECK (salary >= 0),
    department_id INT,
    is_active     BOOLEAN DEFAULT TRUE,
    created_at    TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (department_id) REFERENCES departments(department_id)
        ON DELETE SET NULL
        ON UPDATE CASCADE
);
```

| Column | Constraints Applied | Meaning |
|---|---|---|
| `emp_id` | `SERIAL PRIMARY KEY` | auto ID, unique, not null |
| `name` | `NOT NULL` | compulsory |
| `email` | `UNIQUE NOT NULL` | required, no duplicates |
| `age` | `CHECK (age >= 18)` | must be an adult (if provided) |
| `salary` | `CHECK (salary >= 0)` | never negative |
| `department_id` | `FOREIGN KEY ... ON DELETE SET NULL ON UPDATE CASCADE` | valid department; survives dept deletion; follows dept ID changes |
| `is_active` | `DEFAULT TRUE` | defaults to active |
| `created_at` | `DEFAULT CURRENT_TIMESTAMP` | auto-stamped |

---

## 17. Interview Q&A

**Q: Does a `PRIMARY KEY` allow `NULL`?**
A: No — a Primary Key is automatically both `UNIQUE` and `NOT NULL`.

**Q: Can a Foreign Key column contain duplicate values?**
A: Yes — that's normal and expected in a 1:N relationship (many child rows point to the same parent).

**Q: What must be true about the column a Foreign Key references?**
A: It must be unique in the referenced table — typically its `PRIMARY KEY`, or a column with a `UNIQUE` constraint.

**Q: How do you turn a normal 1:N foreign key relationship into a 1:1 relationship?**
A: Add a `UNIQUE` constraint to the foreign key column — this caps it at one matching child row per parent row.

**Q: Where does the foreign key live in a Many-to-Many relationship?**
A: Nowhere in the two original tables — you create a junction table holding a foreign key to each parent, typically with a composite primary key on both FK columns together.

**Q: `ON DELETE CASCADE` vs `ON DELETE SET NULL` — how do you decide which to use?**
A: Ask: "if the parent disappears, does the child row still make sense?" If no (e.g., order items without an order) → `CASCADE`. If yes, just orphaned (e.g., a project without a manager) → `SET NULL`.

**Q: Why does `ON DELETE SET NULL` require the FK column to be nullable?**
A: Because the action explicitly tries to set that column to `NULL` on parent deletion — if the column is `NOT NULL`, that action is impossible and the two constraints contradict each other.

**Q: What's the default behaviour if you don't specify `ON DELETE` at all?**
A: PostgreSQL blocks (`RESTRICT`s) deletion of a parent row as long as any child row still references it.

**Q: Why should you always name your constraints?**
A: Unnamed constraints get auto-generated names that are hard to reference; a clear name like `chk_student_age` makes `DROP CONSTRAINT` calls and error messages self-explanatory.

---

## 18. Quick Revision Sheet

| Constraint | Syntax Snippet |
|---|---|
| Primary Key | `PRIMARY KEY (col)` |
| Foreign Key | `FOREIGN KEY (col) REFERENCES parent(col)` |
| Unique | `UNIQUE (col)` |
| Not Null | `col type NOT NULL` |
| Check | `CHECK (condition)` |
| Default | `col type DEFAULT value` |
| Cascade delete | `... ON DELETE CASCADE` |
| Null on delete | `... ON DELETE SET NULL` |
| Cascade update | `... ON UPDATE CASCADE` |
| 1:1 FK | FK column + `UNIQUE` |
| 1:N FK | FK column on the "many" table |
| M:N FK | Junction table, composite PK, 2 FKs |

---

## 19. Cheat Sheet

```sql
-- ── BASIC CONSTRAINTS ─────────────────────
CREATE TABLE t (
    id     SERIAL PRIMARY KEY,
    email  VARCHAR(100) UNIQUE NOT NULL,
    age    INT CHECK (age >= 18),
    status BOOLEAN DEFAULT TRUE
);

-- ── NAMED / TABLE-LEVEL ───────────────────
CREATE TABLE t (
    id  INT,
    age INT,
    CONSTRAINT pk_t  PRIMARY KEY (id),
    CONSTRAINT chk_t CHECK (age >= 18)
);
ALTER TABLE t DROP CONSTRAINT chk_t;

-- ── FOREIGN KEY ACTIONS ───────────────────
FOREIGN KEY (col) REFERENCES parent(col);                    -- default: RESTRICT
FOREIGN KEY (col) REFERENCES parent(col) ON DELETE CASCADE;
FOREIGN KEY (col) REFERENCES parent(col) ON DELETE SET NULL;
FOREIGN KEY (col) REFERENCES parent(col) ON UPDATE CASCADE;

-- ── 1:1 ───────────────────────────────────
CREATE TABLE user_profiles (
    profile_id SERIAL PRIMARY KEY,
    user_id    INT UNIQUE NOT NULL REFERENCES users(user_id)
);

-- ── 1:N ───────────────────────────────────
CREATE TABLE employees (
    emp_id        SERIAL PRIMARY KEY,
    department_id INT REFERENCES departments(department_id)  -- FK on "many" side
);

-- ── M:N ───────────────────────────────────
CREATE TABLE student_courses (
    student_id INT REFERENCES students(student_id),
    course_id  INT REFERENCES courses(course_id),
    PRIMARY KEY (student_id, course_id)                        -- composite PK
);
```

---

## 20. Preview of Part 11

| Topic | What You'll Learn |
|---|---|
| `COUNT()`, `SUM()`, `AVG()`, `MAX()`, `MIN()` | Aggregate functions |
| `GROUP BY` | Splitting rows into groups before aggregating |
| Aggregates vs Window Functions | A first comparison (full window function coverage comes later) |
