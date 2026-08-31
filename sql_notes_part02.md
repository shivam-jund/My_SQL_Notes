# SQL & PostgreSQL Complete Notes — Part 2: PostgreSQL Setup & Database-Level Commands

## 📑 Table of Contents
1. Installing PostgreSQL
2. pgAdmin vs psql
3. Essential psql Meta-Commands
4. The Sample "Company" Database Used Throughout This Series
5. `CREATE DATABASE`
6. `DROP DATABASE`
7. `ALTER DATABASE`
8. `CREATE` vs `DROP` vs `ALTER` (Comparison)
9. Interview Q&A
10. Quick Revision Sheet
11. Cheat Sheet
12. Preview of Part 3

**📋 Series Coverage (Part 2):** installing PostgreSQL, pgAdmin, psql, meta-commands, sample company database design, `CREATE DATABASE`, `DROP DATABASE` (+ `IF EXISTS`), `ALTER DATABASE` (rename, owner, settings)

---

## 1. Installing PostgreSQL

**Definition** — Setting up the PostgreSQL server (and client tools) on your machine so you have somewhere to actually run SQL.

**Why It Exists** — Every command in this series needs a running PostgreSQL server to execute against.

**How to install (general guidance — check postgresql.org for current installers)**

| Platform | Typical Approach |
|---|---|
| Windows | Download the installer from postgresql.org — bundles PostgreSQL + pgAdmin + Stack Builder |
| macOS | `brew install postgresql` (Homebrew) or the Postgres.app graphical installer |
| Linux (Debian/Ubuntu) | `sudo apt install postgresql postgresql-contrib` |
| Linux (Fedora/RHEL) | `sudo dnf install postgresql-server` |

**Example** — verify install:
```bash
psql --version
```
**Output**
```
psql (PostgreSQL) 16.x
```

⚠️ **Notes & Caveats**
- Default superuser is commonly named `postgres`; default port is `5432`.
- On Linux, the service may need to be started/enabled: `sudo systemctl start postgresql`.
- You'll be asked to set a password for the `postgres` role during installation — don't lose it.

💡 **Best Practices**
- For team projects, consider Docker (`docker run postgres`) so everyone runs an identical version.

---

## 2. pgAdmin vs psql

**Definition**
- **pgAdmin** — a graphical (GUI) tool for browsing, querying, and administering PostgreSQL databases.
- **psql** — PostgreSQL's official command-line interactive terminal.

| | pgAdmin | psql |
|---|---|---|
| Interface | Graphical (point-and-click) | Text/command-line |
| Best for | Visual exploration, beginners, ER diagrams | Speed, scripting, automation, remote servers |
| Learning curve | Lower initially | Slightly higher, but far more powerful for scripting |
| Runs queries | Yes, via query tool | Yes, primary purpose |

💡 **How to Choose** — Use pgAdmin while you're still building intuition for the schema visually; switch to `psql` once you're comfortable, since almost every real-world workflow (scripts, CI/CD, servers) uses the command line.

---

## 3. Essential psql Meta-Commands

**Definition** — Meta-commands are `psql`-specific shortcuts (not SQL itself) that start with a backslash `\`.

**Syntax & Parameters**

| Command | Purpose |
|---|---|
| `\l` | List all databases |
| `\c dbname` | Connect/switch to a database |
| `\dt` | List tables in the current database |
| `\d tablename` | Describe a table's structure (columns, types, constraints) |
| `\du` | List roles/users |
| `\q` | Quit psql |

**Example**
```
postgres=# \c college_db
postgres=# \dt
```
**Output**
```
You are now connected to database "college_db" as user "postgres".
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | students | table | postgres
```

⚠️ **Notes & Caveats** — Meta-commands are NOT terminated with a semicolon; regular SQL statements are.

---

## 4. The Sample "Company" Database Used Throughout This Series

To keep every part consistent, all examples in this series use a **company-style schema** built around these tables (created step-by-step starting in Part 3):

```
company_db
│
├── departments   (department_id, department_name, location_id)
├── locations      (location_id, city, country)
├── employees      (emp_id, name, department_id, manager_id)
├── salaries       (salary_id, emp_id, salary)
├── projects       (project_id, project_name, emp_id)
├── customers       (customer_id, customer_name)
└── orders          (order_id, customer_id, amount, order_date)
```

💡 We'll build these tables with real `CREATE TABLE` statements starting in **Part 3**, and reuse the exact same data across Joins, Subqueries, CTEs, and Window Functions later in the series — so relationships stay familiar throughout.

---

## 5. `CREATE DATABASE`

**Definition** — Creates a new, empty database.

**Why It Exists** — You need a dedicated container to hold one project's tables, separate from other projects on the same server.

**Syntax**
```sql
CREATE DATABASE database_name;
```

**Parameters**

| Name | Purpose | Default | Example |
|---|---|---|---|
| `database_name` | Name of the new database | — | `college_db` |

**Example**
```sql
CREATE DATABASE company_db;
```
**Output**
```
CREATE DATABASE
```
*(No rows — the database now exists but is empty.)*

⚠️ **Notes & Caveats**
- PostgreSQL does **not** support `IF NOT EXISTS` directly on `CREATE DATABASE` (unlike MySQL). Running it twice on the same name errors with `database "company_db" already exists`.
- A newly created database has 0 tables until you explicitly create them.

❌ **Common Mistakes**
```sql
-- ❌ Invalid in PostgreSQL (this is MySQL syntax):
CREATE DATABASE IF NOT EXISTS college_db;
```
```sql
-- ✅ In PostgreSQL, check first via psql, or catch the error in application code:
\l   -- list databases before creating
```

💡 **Best Practices / How to Choose**
- Use descriptive, `snake_case` names (`company_db`, not `db1`).
- One database per logical application — don't cram unrelated projects into a single database.

---

## 6. `DROP DATABASE`

**Definition** — Permanently deletes an entire database, including every table and row inside it.

**Why It Exists** — Cleans up databases that are no longer needed.

**Syntax**
```sql
DROP DATABASE database_name;
DROP DATABASE IF EXISTS database_name;
```

**Example**
```sql
DROP DATABASE IF EXISTS hospital_db;
```
**Output**
```
DROP DATABASE
```

⚠️ **Notes & Caveats**
- **You cannot drop the database you're currently connected to.** Switch to another database first (commonly `postgres`):
  ```
  \c postgres
  DROP DATABASE college_db;
  ```
- This is irreversible without a prior backup (`pg_dump`).
- Without `IF EXISTS`, dropping a non-existent database throws an error.

❌ **Common Mistakes**
```sql
-- ❌ Fails: currently connected to college_db
\c college_db
DROP DATABASE college_db;
```
```sql
-- ✅ Switch away first
\c postgres
DROP DATABASE college_db;
```

💡 **Best Practices**
- Always take a backup before dropping anything in a production environment.
- Use `IF EXISTS` defensively in setup/teardown scripts so re-running them doesn't error.

---

## 7. `ALTER DATABASE`

**Definition** — Modifies a property of an *existing* database — rename it, change its owner, or change a database-level setting — without touching its tables or data.

**Syntax**
```sql
ALTER DATABASE database_name RENAME TO new_name;
ALTER DATABASE database_name OWNER TO new_owner;
ALTER DATABASE database_name SET parameter TO value;
```

**Example — Rename**
```sql
ALTER DATABASE college_db RENAME TO university_db;
```
**Output**
```
ALTER DATABASE
```
*(Tables and data inside are untouched — only the database's name changes.)*

**Example — Change Owner**
```sql
ALTER DATABASE college_db OWNER TO tushar;
```
⚠️ `tushar` must already exist as a PostgreSQL role — this command does not create users.

**Example — Change a Setting**
```sql
ALTER DATABASE college_db SET timezone TO 'Asia/Kolkata';
```
This applies only to connections to `college_db`; other databases keep their own settings.

⚠️ **Notes & Caveats**
- `ALTER DATABASE` never deletes tables or rows — it only changes metadata/properties.

💡 **Best Practices / How to Choose**
- Use `SET` for connection-level defaults (like `timezone`) that should differ per-database.
- Rename databases only during planned maintenance windows — active connections may be disrupted.

---

## 8. `CREATE` vs `DROP` vs `ALTER` (Comparison)

| Command | Meaning | Effect |
|---|---|---|
| `CREATE DATABASE` | Make a new database | New, empty database created |
| `DROP DATABASE` | Delete a database | Database **and everything inside it** deleted |
| `ALTER DATABASE` | Modify an existing database | Rename / owner / settings changed — data untouched |

💡 **Memory trick:** CREATE = birth 👶 · ALTER = change 🔧 · DROP = death 💀

---

## 9. Interview Q&A

**Q: Can you use `IF NOT EXISTS` with `CREATE DATABASE` in PostgreSQL?**
A: No — unlike MySQL, PostgreSQL doesn't support it directly on `CREATE DATABASE`. You'd check existence beforehand (e.g., query `pg_database`) or handle the error in application code.

**Q: Why can't you drop the database you're currently connected to?**
A: PostgreSQL requires zero active connections to a database before dropping it, since dropping an in-use database would corrupt the session. Switch to a different database (e.g., `postgres`) first.

**Q: What happens to the tables inside a database when you `DROP` it?**
A: Everything is deleted permanently — all tables, rows, and constraints inside.

**Q: How do you rename a database in PostgreSQL?**
A: `ALTER DATABASE old_name RENAME TO new_name;`

**Q: What's the difference between changing a database's owner and its settings?**
A: `OWNER TO` changes which role administratively owns the database. `SET` changes a database-specific configuration parameter (like `timezone`) applied on every connection to that database.

**Q: Is `DROP DATABASE` reversible?**
A: No — not without a prior backup (e.g., via `pg_dump`). It's a permanent, destructive operation.

---

## 10. Quick Revision Sheet

| Task | Command |
|---|---|
| List databases | `\l` |
| Switch database | `\c dbname` |
| List tables | `\dt` |
| Describe table | `\d tablename` |
| Create database | `CREATE DATABASE name;` |
| Drop database (safe) | `DROP DATABASE IF EXISTS name;` |
| Rename database | `ALTER DATABASE old RENAME TO new;` |
| Change owner | `ALTER DATABASE name OWNER TO role;` |
| Change setting | `ALTER DATABASE name SET param TO value;` |

---

## 11. Cheat Sheet

```sql
-- ── PSQL META-COMMANDS ──────────────────
\l                          -- list databases
\c dbname                   -- connect/switch database
\dt                          -- list tables
\d tablename                 -- describe table structure
\du                          -- list roles/users
\q                            -- quit psql

-- ── DATABASE COMMANDS ───────────────────
CREATE DATABASE db_name;
DROP DATABASE db_name;
DROP DATABASE IF EXISTS db_name;

ALTER DATABASE db_name RENAME TO new_name;
ALTER DATABASE db_name OWNER TO role_name;
ALTER DATABASE db_name SET timezone TO 'Asia/Kolkata';
```

---

## 12. Preview of Part 3

| Topic | What You'll Learn |
|---|---|
| `CREATE TABLE` | Building tables with columns + constraints |
| `DROP TABLE` | Deleting tables (`CASCADE` / `RESTRICT`) |
| `TRUNCATE TABLE` | Emptying a table (`RESTART`/`CONTINUE IDENTITY`, `CASCADE`) |
| `ALTER TABLE` | Add/drop/rename columns, change types, constraints, rename table |
