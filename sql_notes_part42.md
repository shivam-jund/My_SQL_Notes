# SQL & PostgreSQL Complete Notes — Part 42: Backup & Restore III — Strategy, WAL & PITR

## 📑 Table of Contents
1. Two Major Types of Backup
2. Logical Backup — Recap & Trade-offs
3. Physical Backup
4. What Is WAL (Write-Ahead Logging)?
5. Base Backup
6. WAL Archiving
7. Point-in-Time Recovery (PITR)
8. `pg_dump` vs PITR — A Concrete Comparison
9. RPO — Recovery Point Objective
10. RTO — Recovery Time Objective
11. The 3-2-1 Backup Principle
12. Backup ≠ Recovery — Test Your Restores
13. Logical vs Physical — Full Comparison
14. A Realistic Production Strategy
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Series Status & What's Next

**📋 Series Coverage (Part 42):** logical vs physical backup, WAL (Write-Ahead Logging), base backups, WAL archiving, Point-in-Time Recovery (PITR), RPO, RTO, the 3-2-1 backup principle, restore testing, a realistic production backup architecture

---

## 1. Two Major Types of Backup

```
BACKUPS
│
├── Logical    → pg_dump, pg_dumpall
└── Physical   → base backup + WAL
```

---

## 2. Logical Backup — Recap & Trade-offs

Recall Parts 40–41: `pg_dump`/`pg_dumpall` represent database *objects and data* in a recreatable form.

**Strengths:** migration between servers, selective restore (one table, one schema), portability across environments, version upgrades.

**Weaknesses:** for very large databases (multi-terabyte), reading, serializing, and later restoring all that object/data information can take considerable time — logical backup alone may not meet aggressive recovery-speed requirements for huge production systems.

---

## 3. Physical Backup

**Definition** — Instead of representing tables/rows/SQL, a physical backup captures the database's **underlying storage state directly** — the actual data files PostgreSQL uses on disk.

```
Logical:  Tables → Rows → SQL representation
Physical: PostgreSQL cluster → raw data files
```

💡 **Why it can be faster for huge databases** — a physical backup works at the storage level rather than reconstructing individual objects — though real-world performance depends heavily on storage, network, and backup architecture.

**Commonly used for:** disaster recovery, very large production databases, Point-in-Time Recovery, replication architectures.

---

## 4. What Is WAL (Write-Ahead Logging)?

⭐ **One of PostgreSQL's most important internal concepts.**

**Definition** — PostgreSQL records every change to a **log** (the Write-Ahead Log) *before* the corresponding data change is considered safely persisted.

```
Application: UPDATE employee SET salary = 60000
        ↓
   WAL record written
        ↓
   Data pages updated
```

**Why it exists** — If the server crashes mid-operation, PostgreSQL needs a reliable way to recover changes that were committed but whose data pages weren't fully written to disk yet:
```
Crash → Read WAL → Replay necessary changes → Recover a consistent state
```

⚠️ **Notes & Caveats** — **WAL is not a complete backup by itself.** It only records *changes* — you still need a **base backup** (Section 5) to establish the starting physical state that WAL's changes get replayed on top of.
```
Base Backup + WAL = Point-in-Time Recovery capability
```

---

## 5. Base Backup

**Definition** — A physical copy of the entire database cluster's state at one specific point in time — the "starting point" that WAL records build forward from.

```
Monday → BASE BACKUP → database state
Monday → Tuesday → WAL WAL WAL WAL (continuous changes recorded)
```

---

## 6. WAL Archiving

**Definition** — Continuously copying completed WAL segments to a separate storage location, so they survive even if the primary server is lost.

```
PostgreSQL → WAL → Archive Storage (separate from the primary server)
```

⚠️ **Notes & Caveats** — If the primary server is destroyed, the archived WAL — combined with a base backup — can be used to reconstruct the database up to (nearly) the moment of failure.

---

## 7. Point-in-Time Recovery (PITR)

⭐ **A major production-database concept.**

**Definition** — Restoring a database to **any specific moment** in the past, not just the exact moment of your last full backup — by combining a base backup with the WAL records generated afterward.

**Example scenario**
```
10:00 → Base backup
10:10 → INSERT
10:30 → UPDATE
10:45 → DELETE
11:00 → Someone accidentally drops a table  😱
```
You don't want to recover to `11:05` (after the mistake) — you want `10:59` (just before it):
```
Base Backup (10:00)
       +
Replay WAL up to 10:59
       ↓
Recovered state, mistake avoided
```

💡 **Why this matters far more than a daily `pg_dump`** — a daily logical backup might only let you recover to *last night's midnight*, potentially losing an entire day of legitimate changes. With continuous WAL archiving, you can often recover to within seconds or minutes of the failure.

---

## 8. `pg_dump` vs PITR — A Concrete Comparison

```
Daily pg_dump at 12 AM
        ↓
Disaster strikes at 3 PM the next day
        ↓
Best-case recovery: last night's midnight
        ↓
Potential data loss: ~15 hours
```
```
Continuous WAL archiving + base backup
        ↓
Disaster strikes at 3 PM
        ↓
Recovery to: ~2:59 PM
        ↓
Potential data loss: minutes, not hours
```

---

## 9. RPO — Recovery Point Objective

**Definition** — The maximum amount of **data loss** (measured in time) an organization can tolerate.

```
RPO = 1 hour   → willing to lose up to ~1 hour of the most recent changes
RPO = 5 minutes → recovery must land within ~5 minutes of the failure point
```

💡 **Memory trick:** RPO ↔ **data loss**.

---

## 10. RTO — Recovery Time Objective

**Definition** — The maximum acceptable **downtime** — how quickly the system must be restored to service.

```
RTO = 30 minutes → service must be back up within ~30 minutes of a failure
```

💡 **Memory trick:** RTO ↔ **downtime**.

**RPO vs RTO — side by side**

| | Question |
|---|---|
| **RPO** | How much data can we afford to lose? |
| **RTO** | How much downtime can we tolerate? |

**Example** — An e-commerce company might target `RPO = 5 minutes`, `RTO = 15 minutes` — meaning the backup/recovery architecture must be specifically designed to meet both numbers; a nightly `pg_dump` alone likely can't hit either.

---

## 11. The 3-2-1 Backup Principle

A widely taught general-purpose backup strategy:
```
3 copies of your data
2 different storage/media types
1 copy stored off-site
```
**Example implementation:** primary database + local backup + cloud/off-site backup.

⚠️ **Notes & Caveats — why off-site matters:** if backups live on the **same** server as the primary database, destroying that server destroys both the data *and* its backup simultaneously. Backups should generally live somewhere physically/logically separate.

---

## 12. Backup ≠ Recovery — Test Your Restores

⚠️ **Notes & Caveats** — Taking backups is only half the job. A backup could silently be corrupted, incomplete, misconfigured, or missing required WAL — and you won't know until you actually try to restore it.

**A responsible practice:**
```
Production → Backup → Restore to a TEST environment → Verify → Measure recovery time
```
**What to verify after a test restore:** row counts, table/index/constraint/view/function presence, permissions, and actual application connectivity — not just "the restore command finished without error."

---

## 13. Logical vs Physical — Full Comparison

| Feature | Logical (`pg_dump`/`pg_dumpall`) | Physical (base backup + WAL) |
|---|---|---|
| Level | Database objects/data | Physical cluster state |
| Selective restore | ✅ Excellent | Less flexible |
| Migration between environments | ✅ Excellent | More environment-dependent |
| PITR support | ❌ Not by itself | ✅ Yes, with WAL |
| Large-database recovery speed | Can be slower | Often better suited |
| Human-readable (plain format) | ✅ Can be | ❌ No |
| Typical use case | Migration, individual DB backup, portability | Disaster recovery |

❌ **Common Mistakes** — Assuming "logical = worse, physical = better" (or vice versa). They solve **different** problems, and a mature strategy typically uses **both**.

---

## 14. A Realistic Production Strategy

For a large (e.g., multi-terabyte) production database, a reasonable combined approach:
```
Physical base backups
        +
Continuous WAL archiving
        +
Off-site storage
        +
Periodic logical dumps (for portability/migration)
        +
Regular restore testing
```
**Why combine them:** physical + WAL gives fast disaster recovery and PITR; logical dumps remain valuable for migration, portability, and selective recovery scenarios that physical backups don't handle as gracefully.

---

## 15. Interview Q&A

**Q: What is WAL, and why does PostgreSQL use it?**
A: Write-Ahead Logging — PostgreSQL records changes to a log before the corresponding data pages are considered safely persisted. It underpins crash recovery: after a crash, PostgreSQL can replay WAL records to reach a consistent state, and it's also foundational for replication and PITR.

**Q: Is WAL by itself a complete backup?**
A: No — WAL only records changes. You also need a base backup providing the starting physical state that those changes get replayed on top of.

**Q: What is Point-in-Time Recovery (PITR)?**
A: The ability to restore a database to any specific moment in the past — not just the time of your last full backup — by combining a base backup with the WAL records generated after it.

**Q: What's the difference between RPO and RTO?**
A: RPO (Recovery Point Objective) defines the maximum acceptable data loss, measured in time. RTO (Recovery Time Objective) defines the maximum acceptable downtime before service is restored.

**Q: Is `pg_dump` sufficient for production disaster recovery?**
A: Not necessarily on its own — it's excellent for portability, migration, and selective restoration, but production disaster recovery for large, critical systems often requires physical backups and WAL archiving to achieve tighter RPO/RTO targets and true point-in-time recovery.

**Q: What's the 3-2-1 backup principle?**
A: Keep 3 copies of your data, across 2 different storage/media types, with 1 copy stored off-site — reducing the risk that a single failure destroys both the primary data and its backups together.

**Q: Why is restore testing important, separate from just taking backups?**
A: A backup that exists but has never been restored successfully might be corrupted, incomplete, or missing dependencies (like required WAL) — you only find out for certain by periodically restoring it to a test environment and verifying the result.

---

## 16. Quick Revision Sheet

| Concept | One-Line Meaning |
|---|---|
| Logical backup | `pg_dump`/`pg_dumpall` — objects & data |
| Physical backup | Base backup + WAL — raw cluster state |
| WAL | Change log written before data pages are finalized |
| Base backup | Starting physical snapshot for WAL to build on |
| WAL archiving | Continuously copying WAL to separate storage |
| PITR | Recover to any specific past moment |
| RPO | Max acceptable **data loss** (time) |
| RTO | Max acceptable **downtime** |
| 3-2-1 | 3 copies, 2 media types, 1 off-site |

---

## 17. Cheat Sheet

```
pg_dump                        → Logical backup of ONE database
pg_dumpall                     → Logical backup of ALL databases
pg_dumpall --globals-only      → Roles + other global objects
pg_restore                     → Restore archive-format pg_dump backups
psql                           → Restore plain SQL scripts / pg_dumpall output

WAL                            → PostgreSQL's change/recovery log
Base Backup                    → Starting physical state
Base Backup + WAL              → PITR capability
PITR                           → Recover to a chosen point in time

RPO                            → Acceptable DATA LOSS
RTO                            → Acceptable DOWNTIME
3-2-1                          → 3 copies / 2 media types / 1 off-site
```

---
