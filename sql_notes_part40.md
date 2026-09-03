# SQL & PostgreSQL Complete Notes — Part 40: Backup & Restore I — pg_dump

## 📑 Table of Contents
1. What Is a Backup?
2. What Is `pg_dump`?
3. Why It's Called a "Logical" Backup
4. Basic Syntax
5. Where Do You Run `pg_dump`?
6. What `pg_dump` Backs Up
7. `pg_dump` Backs Up ONE Database
8. Plain vs Custom Format
9. `-f` (Output File) and `-F` (Format)
10. Selective Backups
11. Connection Options
12. `pg_dump` Doesn't Lock the Database
13. `pg_dump` Is Not a Complete Disaster-Recovery Strategy
14. Interview Q&A
15. Quick Revision Sheet
16. Cheat Sheet
17. Preview of Part 41

**📋 Series Coverage (Part 40):** what a backup is, `pg_dump` as a logical backup tool, basic syntax, running it from the terminal (not `psql`), what gets backed up, plain vs custom format, `-f`/`-F`, `-t` (table)/`-n` (schema)/`--schema-only`/`--data-only`/`--exclude-table`, connection options, snapshot consistency, `pg_dump`'s role in a broader backup strategy

---

## 1. What Is a Backup?

**Definition** — A saved copy of your database that lets you recover if something goes wrong: accidental deletion, corruption, a crashed server, or human error.

```
Database → BACKUP → File(s)
```

---

## 2. What Is `pg_dump`?

**Definition** — A PostgreSQL command-line utility that creates a **logical backup** of a single database — capturing its structure and data in a form that can be replayed to recreate it.

```
PostgreSQL Database → pg_dump → Backup File
```

---

## 3. Why It's Called a "Logical" Backup

⚠️ **Notes & Caveats** — `pg_dump` does **not** copy PostgreSQL's raw physical data files. Instead, it understands database *objects* — tables, columns, constraints, indexes, sequences, views, functions, data — and produces a representation of **how to recreate them** (conceptually, `CREATE TABLE` statements followed by data-loading commands).

---

## 4. Basic Syntax

```bash
pg_dump database_name > backup.sql
```

**Example**
```bash
pg_dump company > company_backup.sql
```

⚠️ **Notes & Caveats** — The `>` is **shell redirection**, not part of PostgreSQL — it sends `pg_dump`'s output into a file.

---

## 5. Where Do You Run `pg_dump`?

⭐ **A very common beginner mistake.**

```sql
-- ❌ WRONG — pg_dump is a terminal command, not SQL
company=# pg_dump company;
```
```bash
# ✅ Correct — run this in your terminal, outside of psql
pg_dump company > company_backup.sql
```

```
psql     → runs SQL commands
Terminal → runs pg_dump
```

**Verify it's installed:**
```bash
pg_dump --version
```

---

## 6. What `pg_dump` Backs Up

A typical dump can include: tables, data, indexes, sequences, constraints, views, functions, triggers, and types — the exact contents depend on the objects present and the options used.

---

## 7. `pg_dump` Backs Up ONE Database

⚠️ **Notes & Caveats — extremely important:** `pg_dump company` backs up **only** `company` — not the entire PostgreSQL server/cluster.
```
PostgreSQL Server
│
├── company   ← backed up
├── college   ← NOT backed up
└── ecommerce ← NOT backed up
```
For backing up **every** database at once (plus cluster-wide objects like roles), you need `pg_dumpall` — covered in Part 41.

---

## 8. Plain vs Custom Format

| Format | Flag | Characteristics |
|---|---|---|
| **Plain** (default) | `-Fp` | Human-readable SQL script; restore with `psql` |
| **Custom** | `-Fc` | PostgreSQL's own binary-ish archive format; restore with `pg_restore`; supports selective/partial restoration |

**Plain format**
```bash
pg_dump company > company.sql
```
**Custom format**
```bash
pg_dump -Fc company > company.dump
```

💡 **How to Choose** — Use plain SQL for small, simple, human-inspectable backups. Use custom format when you want the flexibility of `pg_restore` (Part 41) — like restoring only one table, or doing a schema-only restore later.

---

## 9. `-f` (Output File) and `-F` (Format)

**Two equivalent ways to write a plain SQL dump:**
```bash
pg_dump company > company.sql              -- shell redirection
pg_dump -f company.sql company              -- explicit output file flag
```

**Combining format + output file**
```bash
pg_dump -Fc -f company.dump company
```
```
-Fc              → custom format
-f company.dump  → output file
company          → database name
```

---

## 10. Selective Backups

**One specific table**
```bash
pg_dump -t employees company > employees.sql
```

**Multiple tables**
```bash
pg_dump -t employees -t departments company > selected_tables.sql
```

**Schema only (structure, no data)**
```bash
pg_dump --schema-only company > schema.sql
pg_dump -s company > schema.sql
```

**Data only (rows, no `CREATE TABLE` definitions)**
```bash
pg_dump --data-only company > data.sql
pg_dump -a company > data.sql
```

**Excluding a table**
```bash
pg_dump --exclude-table=logs company > company.sql
```

**A specific schema (namespace)**
```bash
pg_dump --schema=hr company > hr.sql
pg_dump -n hr company > hr.sql
```

**Comparison table**

| Option | Meaning |
|---|---|
| (no option) | Schema + data |
| `-s` / `--schema-only` | Structure only |
| `-a` / `--data-only` | Data only |
| `-t table` | Specific table(s) |
| `-n schema` | Specific schema |
| `--exclude-table=x` | Skip a specific table |

---

## 11. Connection Options

```bash
pg_dump -h localhost -p 5432 -U postgres company > company.sql
```

| Flag | Meaning |
|---|---|
| `-h` | Host |
| `-p` | Port |
| `-U` | Username |
| `-W` | Force a password prompt |
| `-v` | Verbose (progress output) |

⚠️ **Notes & Caveats** — Avoid embedding real passwords directly in shell commands (e.g., `--password=secret`) — they can leak via shell history or process inspection. Use PostgreSQL's `.pgpass` mechanism for safer, non-interactive authentication.

---

## 12. `pg_dump` Doesn't Lock the Database

❌ **Common Mistakes** — Assuming *"if `pg_dump` is running, nobody can use the database."* That's not how it works — PostgreSQL takes a **consistent snapshot** at the start of the dump, so normal database activity can continue concurrently. The dump represents the database as of that snapshot moment, not a live, continuously-updating copy.

---

## 13. `pg_dump` Is Not a Complete Disaster-Recovery Strategy

⚠️ **Notes & Caveats** — `pg_dump` is excellent for: database migration, cloning, development copies, selective restoration, and version upgrades. But for **serious production disaster recovery**, PostgreSQL also relies on **physical backups**, **WAL archiving**, and **Point-in-Time Recovery** — all covered in Part 42.

**Logical vs Physical (preview of Part 42):**

| | `pg_dump` (Logical) | Physical Backup |
|---|---|---|
| What it captures | Database objects & data | The underlying data files themselves |

---

## 14. Interview Q&A

**Q: What is `pg_dump`?**
A: A PostgreSQL command-line utility that creates a logical backup of a single database, including its schema and data.

**Q: Does `pg_dump` back up the entire PostgreSQL server?**
A: No — it backs up exactly one specified database. Cluster-wide objects (like roles) and other databases require a different tool, `pg_dumpall`.

**Q: Is `pg_dump` a logical or physical backup tool?**
A: Logical — it represents database objects and data in a recreatable form, rather than copying PostgreSQL's raw physical files.

**Q: What's the difference between plain and custom dump format?**
A: Plain format is a human-readable SQL script, restored with `psql`. Custom format is PostgreSQL's own archive format, restored with `pg_restore`, and supports selective/partial restoration that plain SQL doesn't.

**Q: Does running `pg_dump` block other users from using the database?**
A: No — PostgreSQL uses a consistent snapshot mechanism, so normal reads and writes can continue while the dump runs; the backup reflects the database as of that snapshot point.

**Q: Is a daily `pg_dump` sufficient for production disaster recovery?**
A: Not necessarily by itself — `pg_dump` is excellent for portability, migration, and selective restores, but production disaster recovery often also requires physical backups and WAL archiving to minimize data loss and enable point-in-time recovery.

---

## 15. Quick Revision Sheet

| Goal | Command |
|---|---|
| Check version | `pg_dump --version` |
| Basic SQL backup | `pg_dump db > db.sql` |
| Custom-format backup | `pg_dump -Fc -f db.dump db` |
| Schema only | `pg_dump -s db > schema.sql` |
| Data only | `pg_dump -a db > data.sql` |
| One table | `pg_dump -t table db > table.sql` |
| One schema | `pg_dump -n schema db > schema.sql` |
| Exclude a table | `pg_dump --exclude-table=x db > db.sql` |

---

## 16. Cheat Sheet

```bash
# ── BASIC ──────────────────────────────────
pg_dump --version
pg_dump company > company.sql

# ── FORMATS ────────────────────────────────
pg_dump -Fp company > company.sql          # plain (default)
pg_dump -Fc -f company.dump company         # custom

# ── SELECTIVE ──────────────────────────────
pg_dump -s company > schema.sql             # schema only
pg_dump -a company > data.sql               # data only
pg_dump -t employees company > employees.sql
pg_dump -t employees -t departments company > selected.sql
pg_dump -n hr company > hr.sql
pg_dump --exclude-table=logs company > company.sql
pg_dump --exclude-schema=logs company > company.sql

# ── CONNECTION ─────────────────────────────
pg_dump -h localhost -p 5432 -U postgres -Fc -f company.dump company

# ── PRODUCTION-STYLE FULL EXAMPLE ─────────
pg_dump \
  -h localhost -p 5432 -U postgres \
  -Fc -f company_backup.dump \
  company
```

---

## 17. Preview of Part 41

| Topic | What You'll Learn |
|---|---|
| `pg_restore` | Restoring custom/directory/tar-format archives |
| `psql` restore | Restoring plain SQL dumps |
| `pg_dumpall` | Backing up every database + roles at once |
