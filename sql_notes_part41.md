# SQL & PostgreSQL Complete Notes — Part 41: Backup & Restore II — pg_restore & pg_dumpall

## 📑 Table of Contents
1. What Is `pg_restore`?
2. Which Backup Formats Need `pg_restore` vs `psql`
3. Basic Syntax (`-d`)
4. Creating the Target Database (`-C`)
5. Listing Archive Contents (`-l`)
6. Selective Restore
7. `--clean` and `--if-exists`
8. Parallel Restore (`-j`)
9. Ownership & Privileges
10. What Is `pg_dumpall`?
11. `pg_dump` vs `pg_dumpall`
12. `--globals-only`, `--roles-only`, `--tablespaces-only`
13. Restoring `pg_dumpall` Output
14. A Practical Migration Strategy
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Preview of Part 42

**📋 Series Coverage (Part 41):** `pg_restore`, matching restore tool to dump format, `-d`/`-C`/`-l`/`-t`/`--schema-only`/`--data-only`/`--clean`/`--if-exists`/`-j`/`--no-owner`/`--no-privileges`, `pg_dumpall`, cluster-wide backup, `--globals-only`, combining `pg_dumpall --globals-only` with per-database `pg_dump`, full migration strategy

---

## 1. What Is `pg_restore`?

**Definition** — A PostgreSQL utility that restores a database from a **non-plain-text** archive created by `pg_dump` (custom, directory, or tar format).

```
company.dump → pg_restore → company database
```

---

## 2. Which Backup Formats Need `pg_restore` vs `psql`

⭐ **One of the most important distinctions in this phase.**

| Dump Format | Restore Tool |
|---|---|
| Plain SQL (`pg_dump db > db.sql`) | `psql db < db.sql` |
| Custom (`-Fc`) | `pg_restore -d db file.dump` |
| Directory / Tar | `pg_restore` |

```
Plain SQL              → psql
Custom / Directory / Tar → pg_restore
```

---

## 3. Basic Syntax (`-d`)

```bash
pg_restore -d database_name backup_file
```

**Example**
```bash
pg_restore -d company company.dump
```

⚠️ **Notes & Caveats** — The **target database must already exist** in the normal case (create it first with `createdb company_new` if needed) — `-d` tells `pg_restore` which existing database to restore into.

---

## 4. Creating the Target Database (`-C`)

**Definition** — `-C` (`--create`) tells `pg_restore` to **create** the database (using metadata stored in the dump) before restoring into it.

```bash
pg_restore -C -d postgres company.dump
```

⚠️ **Notes & Caveats — a genuinely confusing detail:** with `-C`, the database named after `-d` is only the **initial connection database** used to issue the `CREATE DATABASE` command — it is **not** the destination database's name. The actual destination comes from metadata inside the dump itself.

---

## 5. Listing Archive Contents (`-l`)

**Definition** — Shows what's inside a custom-format archive **without restoring anything**.

```bash
pg_restore -l company.dump
```
```
TABLE employees
TABLE departments
INDEX employees_pkey
...
```

💡 This is one of custom format's biggest advantages over plain SQL — you can **inspect** the archive before deciding exactly what (and how) to restore.

---

## 6. Selective Restore

**One specific table**
```bash
pg_restore -d company_new -t employees company.dump
```

**Schema only**
```bash
pg_restore --schema-only -d company_new company.dump
pg_restore -s -d company_new company.dump
```

**Data only**
```bash
pg_restore --data-only -d company_new company.dump
pg_restore -a -d company_new company.dump
```

**A specific schema**
```bash
pg_restore -n hr -d company_new company.dump
```

**Using a customized restore list** (built from `-l` output)
```bash
pg_restore -l company.dump > contents.txt
# edit contents.txt to remove unwanted items
pg_restore -L contents.txt -d company_new company.dump
```

---

## 7. `--clean` and `--if-exists`

**`--clean` (`-c`)** — drops existing objects before recreating them (avoids "relation already exists" errors when the target already has matching objects).
```bash
pg_restore --clean -d company_new company.dump
```

**`--if-exists`** — commonly paired with `--clean`, so the generated `DROP` commands don't error if an object happens to be missing.
```bash
pg_restore --clean --if-exists -d company_new company.dump
```

⚠️ **Notes & Caveats** — `--clean` does **not** mean `DROP DATABASE` — it drops the individual **objects being restored** (tables, functions, etc.), not the database itself. Still, never run it blindly against a production database without understanding exactly what will be dropped.

⭐ **A classic short-option trap:** `-C` = `--create`, `-c` = `--clean` — easy to confuse, with very different (and potentially destructive) consequences. When learning, prefer the full `--long-option` spelling for clarity.

---

## 8. Parallel Restore (`-j`)

**Definition** — Runs the restore using multiple parallel jobs — a major advantage of archive-based formats.

```bash
pg_restore -j 4 -d company_new company.dump
```

⚠️ **Notes & Caveats** — Only available for custom/directory-format archives — a plain SQL dump restored via `psql` has no equivalent parallel mechanism.

---

## 9. Ownership & Privileges

```bash
pg_restore --no-owner -d company_new company.dump       -- skip restoring original object ownership
pg_restore --no-privileges -d company_new company.dump  -- skip restoring GRANT/REVOKE commands
```

💡 **How to Choose** — Very useful when migrating between environments where the original roles don't exist, or where permissions will be configured separately in the new environment.

---

## 10. What Is `pg_dumpall`?

**Definition** — A utility that creates a logical backup of **every database** in a PostgreSQL cluster, plus cluster-wide (global) objects such as roles.

```
PostgreSQL Cluster
│
├── company, college, ecommerce, testing   (all databases)
├── roles, tablespaces                      (global objects)
        ↓
   pg_dumpall
        ↓
   SQL script
```

**Basic command**
```bash
pg_dumpall > all_databases.sql
```

⚠️ **Notes & Caveats** — `pg_dumpall`'s output is always a **plain SQL script** — there's no custom/directory format option — so restoration is done with `psql`, not `pg_restore`.

---

## 11. `pg_dump` vs `pg_dumpall`

| | `pg_dump` | `pg_dumpall` |
|---|---|---|
| Scope | One database | All databases in the cluster |
| Formats | Plain, custom, directory, tar | Plain SQL only |
| Global objects (roles, tablespaces) | ❌ Not comprehensively | ✅ Yes |
| Selective table backup | ✅ | ❌ Not its purpose |
| Parallel dump | ✅ (with suitable format) | ❌ |
| Restore tool | `psql` or `pg_restore`, depending on format | `psql` only |

💡 **Memory trick:** `pg_dump` = **ONE** · `pg_dumpall` = **ALL**.

---

## 12. `--globals-only`, `--roles-only`, `--tablespaces-only`

Since you don't always want *every* database dumped, `pg_dumpall` offers narrower options:

```bash
pg_dumpall --globals-only > globals.sql          -- roles + other global objects
pg_dumpall --roles-only > roles.sql               -- just roles
pg_dumpall --tablespaces-only > tablespaces.sql   -- just tablespaces
```

💡 **A very common, practical strategy** — combine `pg_dumpall --globals-only` (for roles) with individual `pg_dump -Fc` backups per database, rather than one enormous all-in-one SQL file:
```bash
pg_dumpall --globals-only > globals.sql
pg_dump -Fc company > company.dump
pg_dump -Fc college > college.dump
```
This gives you the flexibility of `pg_restore`'s selective restoration for each database, while still capturing cluster-wide roles separately.

---

## 13. Restoring `pg_dumpall` Output

```bash
psql -f all_databases.sql postgres
# or equivalently:
psql postgres < all_databases.sql
```

⚠️ **Notes & Caveats** — You connect to an **initial** database (commonly `postgres`) to run the script, since the script itself contains the commands to create/populate the other databases.

**Restoring the "globals + per-database dumps" strategy from Section 12:**
```bash
psql -f globals.sql postgres            -- restore roles first
pg_restore -d company company.dump       -- then each database
pg_restore -d college college.dump
```

---

## 14. A Practical Migration Strategy

```
OLD SERVER
    │ pg_dump / pg_dumpall
    ↓
Backup file(s)
    │ transfer
    ↓
NEW SERVER
    │ psql / pg_restore
    ↓
Restored databases
```

---

## 15. Interview Q&A

**Q: Can `pg_restore` restore a plain `.sql` file?**
A: Normally, no — plain SQL dumps are restored with `psql`. `pg_restore` is built for archive formats (custom, directory, tar).

**Q: What does `-C` do in `pg_restore`, and why is it commonly misunderstood?**
A: It tells `pg_restore` to create the target database (from metadata in the dump) before restoring. It's confusing because the database named after `-d` in this mode is only the *initial connection* database — not the actual destination, which comes from the dump's own metadata.

**Q: What's the difference between `-C` and `-c` in `pg_restore`?**
A: `-C` means `--create` (create the database first); `-c` means `--clean` (drop existing objects before recreating them). They're easy to confuse and have very different effects — prefer the long-option spelling when clarity matters.

**Q: What's the main difference between `pg_dump` and `pg_dumpall`?**
A: `pg_dump` creates a logical backup of a single database and supports multiple formats (plain, custom, directory, tar). `pg_dumpall` creates a plain-SQL logical dump covering *every* database in the cluster, and can include cluster-wide objects like roles and tablespaces.

**Q: How would you back up just the roles in a PostgreSQL cluster?**
A: `pg_dumpall --globals-only > globals.sql` (or `--roles-only` for roles specifically).

**Q: Why might you combine `pg_dumpall --globals-only` with individual `pg_dump` backups instead of one big `pg_dumpall` dump?**
A: It gives you the selective-restoration flexibility of `pg_restore` (custom format) for each individual database, while still capturing cluster-wide roles once — versus one large, less flexible plain-SQL script.

---

## 16. Quick Revision Sheet

| Goal | Command |
|---|---|
| Restore a custom-format dump | `pg_restore -d db file.dump` |
| Restore a plain SQL dump | `psql db < file.sql` |
| Create DB + restore | `pg_restore -C -d postgres file.dump` |
| List archive contents | `pg_restore -l file.dump` |
| Restore one table | `pg_restore -d db -t table file.dump` |
| Drop existing objects first | `pg_restore --clean --if-exists -d db file.dump` |
| Parallel restore | `pg_restore -j 4 -d db file.dump` |
| Back up all databases | `pg_dumpall > all.sql` |
| Back up just roles/globals | `pg_dumpall --globals-only > globals.sql` |
| Restore `pg_dumpall` output | `psql -f all.sql postgres` |

---

## 17. Cheat Sheet

```bash
# ── PG_RESTORE ─────────────────────────────
createdb company_new
pg_restore -d company_new company.dump

pg_restore -C -d postgres company.dump                    # create + restore
pg_restore -l company.dump                                  # list contents
pg_restore -d company_new -t employees company.dump          # one table
pg_restore -s -d company_new company.dump                     # schema only
pg_restore -a -d company_new company.dump                      # data only
pg_restore --clean --if-exists -d company_new company.dump      # drop-then-recreate
pg_restore -j 4 -d company_new company.dump                       # parallel
pg_restore --no-owner --no-privileges -d company_new company.dump  # migration-friendly

# ── PG_DUMPALL ─────────────────────────────
pg_dumpall > all_databases.sql
pg_dumpall --globals-only > globals.sql
pg_dumpall --roles-only > roles.sql

# ── RESTORE PG_DUMPALL OUTPUT ─────────────
psql -f all_databases.sql postgres

# ── COMBINED STRATEGY (recommended) ───────
pg_dumpall --globals-only > globals.sql
pg_dump -Fc company > company.dump
pg_dump -Fc college > college.dump
#   restore:
psql -f globals.sql postgres
pg_restore -d company company.dump
pg_restore -d college college.dump
```

---

## 18. Preview of Part 42

| Topic | What You'll Learn |
|---|---|
| Logical vs Physical Backup | Two fundamentally different backup approaches |
| WAL (Write-Ahead Logging) | PostgreSQL's crash-recovery foundation |
| PITR (Point-in-Time Recovery) | Recovering to any moment, not just your last backup |
| RPO & RTO | How organizations define acceptable data loss and downtime |
