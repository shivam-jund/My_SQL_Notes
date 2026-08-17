# SQL & PostgreSQL Complete Notes — Part 1: Database & SQL Fundamentals

## 📑 Table of Contents
1. What is Data?
2. What is a Database?
3. What is a DBMS?
4. What is an RDBMS?
5. DBMS vs RDBMS
6. Why Are Databases Needed?
7. What is SQL?
8. SEQUEL vs SQL — A Quick History
9. Categories of SQL Commands (DDL/DML/DQL/DCL/TCL)
10. Tables, Rows, Columns, Records
11. Schema vs Instance
12. Database Keys (Primary, Foreign, Candidate, Composite, Super, Unique)
13. Constraints — A First Look
14. NULL — A First Look
15. Relationships (1:1, 1:N, M:N) & the Foreign Key Placement Rule
16. Normalization Overview (1NF–BCNF)
17. Interview Q&A
18. Quick Revision Sheet
19. Cheat Sheet
20. Preview of Part 2

**📋 Series Coverage (Part 1):** data vs information, database, DBMS, RDBMS, why databases exist, SQL, SEQUEL history, DDL/DML/DQL/DCL/TCL, tables/rows/columns/records, schema vs instance, primary key, foreign key, candidate key, composite key, super key, unique key, constraints overview, NULL overview, 1:1/1:N/M:N relationships, foreign key placement rule, normalization (1NF–BCNF)

> 💡 This part is conceptual — almost no SQL is written yet. Think of it as building the mental model that every later command will sit on top of. Part 2 onward starts writing real SQL.

---

## 1. What is Data?

**Definition** — Data is raw, unprocessed facts: numbers, words, dates, measurements. On their own they don't tell a story.

**Why It Exists** — Before we can talk about databases or SQL, we need to agree on what we're actually storing. Everything downstream (tables, rows, queries) exists to organize *data* into something useful.

**Example**
```
"Tushar"   "20"   "Mohali"
```
Alone, these are just three disconnected facts. Once structured as `name = Tushar, age = 20, city = Mohali`, they become **information** about one student.

⚠️ **Notes & Caveats**
- Data → Information happens when context/structure is added.
- Information → Knowledge happens when it's interpreted for decision-making (outside SQL's job).

---

## 2. What is a Database?

**Definition** — A database is an organized, structured collection of related data, stored electronically so it can be efficiently created, read, updated, and deleted.

**Why It Exists** — Storing data in flat files (spreadsheets, text files) breaks down fast: no easy way to enforce rules, prevent duplicate/contradictory data, search efficiently, or let multiple people/programs use the data safely at the same time. A database solves all of this.

**Example**
```
PostgreSQL Server
│
├── college_db
│   ├── students
│   ├── teachers
│   └── courses
│
└── company_db
    ├── employees
    ├── departments
    └── projects
```

⚠️ **Notes & Caveats**
- A database is a **container**. It holds tables (and other objects like views, sequences, indexes) — it isn't itself a table.
- A newly created database is always empty; tables are created inside it separately.

💡 **Best Practices**
- One database per logical application/project, not one giant shared database for unrelated systems.

---

## 3. What is a DBMS?

**Definition** — DBMS (Database Management System) is software that lets users and applications create, read, update, delete, and administer databases — without manually handling file storage, concurrency, or security themselves.

**Why It Exists** — Without a DBMS, every application would have to reinvent low-level file handling, locking, and data-integrity logic. A DBMS abstracts all of that behind a standard interface.

**Example** — PostgreSQL, MySQL, Oracle, SQL Server, MongoDB, SQLite.

⚠️ **Notes & Caveats**
- "Database" and "DBMS" are often used loosely as synonyms in conversation, but technically: DBMS = the *software*; database = the *data* it manages.

---

## 4. What is an RDBMS?

**Definition** — RDBMS (Relational DBMS) is a DBMS that stores data in **tables** (rows and columns) and enforces **relationships** between those tables using keys.

**Why It Exists** — Real-world data is relational — employees belong to departments, orders belong to customers. An RDBMS lets you model these relationships directly and enforce their integrity (e.g., you can't have an order for a customer that doesn't exist).

**Example** — PostgreSQL, MySQL, Oracle Database, SQL Server are RDBMS. MongoDB (document-based) and Redis (key-value) are **not** RDBMS — they're NoSQL.

---

## 5. DBMS vs RDBMS

| Aspect | DBMS | RDBMS |
|---|---|---|
| Data storage | Files, hierarchical, navigational, or tables | Strictly tables (rows & columns) |
| Relationships | Not necessarily enforced | Enforced via Primary/Foreign Keys |
| Normalization | Not required | Encouraged/supported |
| Examples | File systems, some early DBMS, XML DBs | PostgreSQL, MySQL, Oracle, SQL Server |
| Query language | Varies | SQL (standardized) |

💡 Every RDBMS is a DBMS, but not every DBMS is an RDBMS.

---

## 6. Why Are Databases Needed?

**Problems databases solve:**
- **Redundancy** — avoid storing the same fact in ten different spreadsheets that drift out of sync.
- **Data integrity** — constraints stop invalid data (e.g., negative age) from ever being saved.
- **Concurrent access** — many users/programs can safely read and write at the same time.
- **Efficient search** — indexes let you find rows among millions almost instantly.
- **Security** — fine-grained control over who can see or change what.
- **Backup & recovery** — structured, reliable ways to restore data after a failure.

---

## 7. What is SQL?

**Definition** — SQL (Structured Query Language) is the standard language used to create, read, update, and delete data and structures inside a relational database.

**Why It Exists** — Different vendors (PostgreSQL, MySQL, Oracle...) needed one common way for people to *talk* to any relational database, instead of a different bespoke language per product.

**Example**
```sql
SELECT * FROM students;
```

⚠️ **Notes & Caveats**
- SQL is **declarative** — you describe *what* data you want, not *how* to fetch it step by step (the database engine decides the "how").
- SQL syntax has minor differences across vendors (PostgreSQL vs MySQL vs SQL Server) — this series uses **PostgreSQL** as the default, and calls out MySQL differences where they matter.

---

## 8. SEQUEL vs SQL — A Quick History

**Definition** — SEQUEL (Structured English Query Language) was IBM's original 1970s name for this language. It was renamed **SQL** after a trademark conflict with an existing UK company's "SEQUEL" trademark.

⚠️ **Notes & Caveats**
- This is *why* many people still pronounce "SQL" as "sequel" out loud, even though the letters officially stand for Structured Query Language.
- Both names refer to the same lineage of language — there's no functional difference today.

---

## 9. Categories of SQL Commands

| Category | Stands For | Purpose | Example Commands |
|---|---|---|---|
| **DDL** | Data Definition Language | Define/change structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DML** | Data Manipulation Language | Change data | `INSERT`, `UPDATE`, `DELETE` |
| **DQL** | Data Query Language | Read data | `SELECT` |
| **DCL** | Data Control Language | Permissions | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | Manage transactions | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

💡 Some textbooks fold DQL into DML — don't worry about the exact taxonomy in interviews, just know what each command *does*.

---

## 10. Tables, Rows, Columns, Records

| Term | Definition |
|---|---|
| **Table** | A structured collection of related data, organized as rows and columns |
| **Column (Field)** | A single named attribute of a table, with one data type (e.g., `age INT`) |
| **Row (Record / Tuple)** | One entry in a table, representing one real-world instance |

**Example**
```
students table
┌────────────┬────────┬─────┐
│ student_id │ name   │ age │  ← columns
├────────────┼────────┼─────┤
│ 1          │ Aman   │ 20  │  ← row / record
│ 2          │ Ravi   │ 21  │  ← row / record
└────────────┴────────┴─────┘
```

---

## 11. Schema vs Instance

**Definition**
- **Schema** — the structural design/blueprint of the database: table names, columns, data types, constraints. Changes rarely.
- **Instance** — the actual data present at a given moment. Changes constantly.

**Analogy** — Schema is a building's blueprint; Instance is a photo of the building right now, with its current occupants.

| | Schema | Instance |
|---|---|---|
| What it describes | Structure | Actual data |
| Changes | Rarely (via `ALTER`) | Constantly (via `INSERT`/`UPDATE`/`DELETE`) |
| Example | "students has student_id, name, age" | "row 1 = Aman, 20" |

---

## 12. Database Keys

**Definition** — Keys are columns (or sets of columns) used to uniquely identify rows and to connect related tables.

### Primary Key (PK)
Uniquely identifies each row in a table. Automatically **UNIQUE** + **NOT NULL**. Exactly **one** per table (though it can span multiple columns — see Composite Key).

### Foreign Key (FK)
A column (or set of columns) in one table that references a Primary Key (or Unique key) in another table, creating a relationship between them.

### Candidate Key
Any column, or minimal set of columns, that *could* qualify to be the Primary Key (i.e., it's unique and not null). A table can have several candidate keys; only one is chosen as the actual PK — the rest remain available (often enforced with `UNIQUE`).

### Composite Key
A Primary Key (or candidate key) made up of **two or more columns combined**, used when no single column is unique by itself. Example: in an `order_items` table, `(order_id, product_id)` together form the key.

### Super Key
**Any** set of columns that can uniquely identify a row — including candidate keys *plus* extra, unnecessary columns. Every candidate key is a super key, but not every super key is minimal enough to be a candidate key.

### Unique Key
Like a Primary Key in that it prevents duplicate values, but it **allows NULL** (PostgreSQL allows multiple NULLs in a UNIQUE column), and a table can have **many** UNIQUE constraints.

**Comparison Table**

| Key Type | Uniqueness | Allows NULL? | Count per Table | Example |
|---|---|---|---|---|
| Primary Key | ✅ | ❌ | Exactly 1 | `student_id` |
| Unique Key | ✅ | ✅ (Postgres: multiple NULLs OK) | Many | `email` |
| Candidate Key | ✅ | ❌ (must qualify as PK) | Many (one becomes PK) | `email`, `student_id` |
| Composite Key | ✅ (as a combination) | Depends on columns | Part of PK/candidate keys | `(order_id, product_id)` |
| Super Key | ✅ (may include redundant columns) | Depends | Many | `(student_id, name)` |
| Foreign Key | ❌ (can repeat) | ✅ (unless `NOT NULL` added) | Many | `department_id` in `employees` |

⚠️ **Notes & Caveats**
- A **Foreign Key column itself does NOT need to be unique** — it commonly repeats (many employees can share the same `department_id`). This is exactly how a 1:N relationship works.
- Full `CREATE TABLE ... PRIMARY KEY / FOREIGN KEY` syntax is covered in **Part 3** (table commands) and **Part 11** (constraints), where we'll write real, runnable examples.

---

## 13. Constraints — A First Look

**Definition** — Constraints are rules attached to columns (or the whole table) that control what data is allowed to be stored.

**Preview of the main constraints** (full syntax + examples arrive in the dedicated Constraints part later in this series):

| Constraint | Purpose |
|---|---|
| `PRIMARY KEY` | Uniquely identify each row |
| `FOREIGN KEY` | Connect to another table |
| `UNIQUE` | Prevent duplicate values |
| `NOT NULL` | Value is compulsory |
| `CHECK` | Value must satisfy a condition |
| `DEFAULT` | Auto-fill a value when none is given |

---

## 14. NULL — A First Look

**Definition** — `NULL` represents a **missing or unknown** value. It is not the same as `0`, not the same as an empty string `''`, and not the same as `FALSE`.

⚠️ **Notes & Caveats**
- `NULL = NULL` does **not** evaluate to `TRUE` in SQL — it evaluates to `UNKNOWN` (three-valued logic: `TRUE` / `FALSE` / `UNKNOWN`).
- Full operational handling (`IS NULL`, `COALESCE`, `NULLIF`) is covered in the WHERE/Filtering part later in this series.

---

## 15. Relationships & the Foreign Key Placement Rule

**Definition** — A relationship describes how rows in one table relate to rows in another. There are three kinds:

```
ONE-TO-ONE (1:1)         ONE-TO-MANY (1:N)         MANY-TO-MANY (M:N)

users        user_profiles   departments   employees   students      courses
┌────┐       ┌────┐          ┌────┐        ┌────┐      ┌────┐        ┌────┐
│ id │──────▶│ id │          │ id │◀──┬────│dept│      │ id │◀──┐ ┌─▶│ id │
└────┘       └────┘          └────┘   │    │_id │      └────┘  │ │  └────┘
one user →                   one dept │    └────┘      needs a  │ │
one profile                   ┌───────┴──▶ many         JUNCTION │ │
                               employees   employees    TABLE    └─┘
```

| Relationship | Real Example | Where the Foreign Key Goes |
|---|---|---|
| **One-to-One (1:1)** | `users` ↔ `user_profiles` | FK in **either** table — usually the "weaker"/dependent one (e.g., `user_id` FK lives in `user_profiles`) |
| **One-to-Many (1:N)** | one `department` → many `employees` | FK **always** on the **"many"** side (`employees.department_id`) |
| **Many-to-Many (M:N)** | `students` ↔ `courses` | Cannot use a plain FK in either table — requires a **junction/bridge table** (e.g., `student_courses`) holding FKs to **both** parent tables as a composite PK |

💡 **The core rule to memorize:**
> In 1:N, the foreign key lives on the side that has "many" of something. In M:N, you don't put a foreign key in either original table at all — you build a new table in between.

⚠️ We'll write the actual `CREATE TABLE` statements, matching `INSERT`s, and see what happens when a foreign key constraint is violated, for **all three** relationship types, once `FOREIGN KEY` syntax is formally introduced in the dedicated **Constraints** part later in this series.

---

## 16. Normalization Overview (1NF–BCNF)

**Definition** — Normalization is the process of organizing columns and tables to minimize data redundancy and avoid update/insert/delete anomalies.

| Normal Form | Rule | Fixes |
|---|---|---|
| **1NF** | Every column holds atomic (indivisible) values; no repeating groups/arrays crammed into one column | "Multiple phone numbers in one column" problem |
| **2NF** | 1NF + no **partial dependency** — every non-key column depends on the **whole** composite primary key, not just part of it | Redundancy when using composite keys |
| **3NF** | 2NF + no **transitive dependency** — non-key columns shouldn't depend on other non-key columns | e.g., storing `city` and `zip_code` where `zip_code` alone determines `city` |
| **BCNF** | Stricter 3NF — **every determinant must be a candidate key** | Edge cases 3NF misses when multiple overlapping candidate keys exist |

**Example (fixing a 1NF violation)**

❌ Not atomic:
```
student_id | name  | phones
1          | Aman  | "9876543210, 9123456789"
```

✅ Atomic (1NF-compliant), using a separate table:
```
students                    student_phones
student_id | name           student_id | phone
1          | Aman           1          | 9876543210
                             1          | 9123456789
```

💡 **Best Practices**
- Aim for 3NF for most transactional (OLTP) systems — it's the sweet spot of integrity vs. simplicity.
- Reporting/analytics systems sometimes *deliberately* denormalize for read speed — normalization is a design tool, not a religion.

---

## 17. Interview Q&A

**Q: What's the difference between DBMS and RDBMS?**
A: DBMS manages data generally; RDBMS specifically stores data in tables and enforces relationships via keys, using SQL. Every RDBMS is a DBMS, not the reverse.

**Q: Primary Key vs Candidate Key?**
A: Every candidate key is eligible to become the Primary Key (unique + not null). Only one candidate key is chosen as PK per table; the others remain candidate keys, often still enforced as UNIQUE.

**Q: Primary Key vs Unique Key?**
A: Both enforce uniqueness. PK disallows NULL and allows only one per table. UNIQUE allows NULL (Postgres permits multiple NULLs) and a table can have several UNIQUE constraints.

**Q: What is a Composite Key?**
A: A primary/candidate key made of two or more columns combined, used when no single column is unique alone — e.g., `(student_id, course_id)` in an enrollment table.

**Q: Super Key vs Candidate Key?**
A: A super key is any column combination that uniquely identifies a row (may include redundant columns). A candidate key is a *minimal* super key — no column can be removed without losing uniqueness.

**Q: Give one real example each of 1:1, 1:N, and M:N.**
A: 1:1 — `users` & `user_profiles`. 1:N — one `department`, many `employees`. M:N — `students` & `courses` (needs a junction table).

**Q: Where does the foreign key go in a One-to-Many relationship?**
A: Always on the "many" side — e.g., `employees.department_id` references `departments.department_id`.

**Q: What problem does normalization solve?**
A: It reduces data redundancy and prevents update/insert/delete anomalies by ensuring columns depend only on the appropriate key.

**Q: Difference between 3NF and BCNF?**
A: 3NF removes transitive dependencies. BCNF is stricter — every determinant (left side of a functional dependency) must be a candidate key, which resolves edge cases 3NF misses when overlapping candidate keys exist.

**Q: Schema vs Instance?**
A: Schema is the fixed structural design (tables/columns/types/constraints); instance is the actual data present right now — it changes constantly while the schema stays stable.

---

## 18. Quick Revision Sheet

| Term | One-line Meaning |
|---|---|
| Data | Raw, unprocessed facts |
| Database | Organized, structured collection of related data |
| DBMS | Software that manages databases |
| RDBMS | DBMS that uses tables + enforced relationships |
| SQL | Standard language to talk to an RDBMS |
| Table | Rows + columns of related data |
| Row/Record | One entry/instance in a table |
| Column/Field | One named, typed attribute |
| Schema | Structure/blueprint of a database |
| Instance | Actual data at a point in time |
| Primary Key | Uniquely identifies a row; no NULL, no duplicates |
| Foreign Key | Column referencing another table's PK/unique key |
| Candidate Key | Any column(s) eligible to be PK |
| Composite Key | Key made of 2+ columns |
| Super Key | Any column set that uniquely identifies a row |
| Unique Key | Enforces uniqueness, allows NULL |
| 1NF | Atomic values, no repeating groups |
| 2NF | 1NF + no partial dependency |
| 3NF | 2NF + no transitive dependency |
| BCNF | Every determinant is a candidate key |

---

## 19. Cheat Sheet

```
# ── CORE TERMS ──────────────────────────
Data            → raw facts
Database        → organized data container
DBMS            → software managing databases
RDBMS           → DBMS + tables + relationships
SQL             → language to talk to an RDBMS

# ── SQL COMMAND CATEGORIES ──────────────
DDL  → CREATE, ALTER, DROP, TRUNCATE   (structure)
DML  → INSERT, UPDATE, DELETE          (data)
DQL  → SELECT                          (read)
DCL  → GRANT, REVOKE                   (permissions)
TCL  → COMMIT, ROLLBACK, SAVEPOINT     (transactions)

# ── KEYS ─────────────────────────────────
Primary Key     → unique + not null, identifies row
Foreign Key     → references another table's PK
Candidate Key   → eligible-to-be-PK column(s)
Composite Key   → PK/candidate key of 2+ columns
Super Key       → any uniquely-identifying column set
Unique Key      → unique, NULL allowed

# ── RELATIONSHIPS ────────────────────────
1:1  → FK in either table (usually the dependent one)
1:N  → FK on the "many" side
M:N  → junction table, FK to both parents (composite PK)

# ── NORMALIZATION ────────────────────────
1NF  → atomic columns, no repeating groups
2NF  → 1NF + no partial dependency
3NF  → 2NF + no transitive dependency
BCNF → every determinant is a candidate key
```

---

## 20. Preview of Part 2

| Topic | What You'll Learn |
|---|---|
| PostgreSQL Setup | Installing PostgreSQL, pgAdmin vs psql, connecting |
| `CREATE DATABASE` | Making a new database |
| `DROP DATABASE` | Deleting a database safely (`IF EXISTS`) |
| `ALTER DATABASE` | Renaming, changing owner, database-level settings |
