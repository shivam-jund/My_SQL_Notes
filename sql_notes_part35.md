# SQL & PostgreSQL Complete Notes — Part 35: Query Optimization I — Principles & Execution Plans

## 📑 Table of Contents
1. What Is Query Optimization?
2. Why Queries Become Slow
3. The Optimization Workflow
4. The Execution Plan Is a Tree
5. Cost — Startup vs Total
6. Rows: Estimated vs Actual
7. `width`
8. `actual time` & `loops` — Reading Multi-Row Operations
9. `Filter` & `Rows Removed by Filter`
10. Bitmap Scan
11. `Sort` and `Aggregate` Nodes
12. The Golden Rule — Judge Cost, Not Scan Type
13. A Full Optimization Investigation, Start to Finish
14. Interview Q&A
15. Quick Revision Sheet
16. Cheat Sheet
17. Preview of Part 36

**📋 Series Coverage (Part 35):** what query optimization means, why queries slow down, the measure-change-measure workflow, execution plans as a tree structure, cost (startup vs total), estimated vs actual rows, `width`, `actual time`/`loops`, `Filter`/`Rows Removed by Filter`, Bitmap Scan, `Sort`/`Aggregate` plan nodes, a complete optimization investigation

> This part builds directly on `EXPLAIN`/`EXPLAIN ANALYZE` from Part 30 — if those basics feel unfamiliar, a quick look back there will help.

---

## 1. What Is Query Optimization?

**Definition** — The process of improving a SQL query so it produces the **same result** using **less time and fewer resources** (CPU, memory, disk I/O).

⚠️ **Notes & Caveats** — Optimization never changes the *result* — only how efficiently PostgreSQL gets there.

```
Same Result + Less Time + Less CPU + Less Memory + Less Disk I/O = Optimization
```

💡 **Analogy** — Finding "Tushar" in a 1,000-page book: reading every page works but is slow; using the book's index to jump straight to page 742 gets the same answer far faster.

---

## 2. Why Queries Become Slow

```
Large table
    ↓
No useful index
    ↓
Too many rows scanned
    ↓
Expensive joins
    ↓
Unnecessary sorting
    ↓
Unnecessary grouping
    ↓
Large intermediate results
    ↓
Slow query
```

**The main things worth examining when optimizing:**

| Area | Question |
|---|---|
| Indexes | Can PostgreSQL find the needed rows faster? |
| Joins | Are tables being joined efficiently? |
| Filtering | Can unnecessary rows be eliminated earlier? |
| Sorting | Is `ORDER BY` actually needed? |
| Aggregation | Are we grouping far more rows than necessary? |
| Query structure | Could the query itself be rewritten more efficiently? |
| Execution plan | What is PostgreSQL *actually* doing? |

---

## 3. The Optimization Workflow

```
SQL Query
   ↓
EXPLAIN ANALYZE          ← see what PostgreSQL actually did
   ↓
Find the expensive part
   ↓
Improve it
   ↓
EXPLAIN ANALYZE again
   ↓
Compare
```

💡 **The single most important rule of optimization:** *Never optimize by guessing. Measure first, change one thing, and measure again.*

❌ **Common Mistakes**
- Assuming "this query should be fast" without checking `EXPLAIN ANALYZE` — a query that looks simple can still scan millions of rows internally.
- Changing multiple things at once, making it impossible to tell which change actually helped.
- Assuming `SELECT col1, col2` is always meaningfully faster than `SELECT *` — if PostgreSQL still has to scan millions of rows to find the matches, trimming columns alone won't fix the real bottleneck.

---

## 4. The Execution Plan Is a Tree

⚠️ **Notes & Caveats** — An execution plan isn't a flat list — it's a **tree** of operations, where each node's output feeds into the node above it.

```
Hash Join
├── Seq Scan on employees
└── Hash
    └── Seq Scan on departments
```
```
              Hash Join
              /       \
             /         \
      employees      departments
```

💡 **Why this matters** — You can inspect each branch separately. If `employees` shows 10 million rows scanned but `departments` shows only 20, you immediately know where the bulk of the work — and any potential optimization — is concentrated.

---

## 5. Cost — Startup vs Total

```
Seq Scan on employees (cost=0.00..183.00 rows=5000 width=40)
```

⚠️ **Notes & Caveats** — `cost` values are **internal comparison units**, not milliseconds.

| Number | Meaning |
|---|---|
| First (`0.00`) | **Startup cost** — estimated cost before the first row can be produced |
| Second (`183.00`) | **Total cost** — estimated cost to produce every row |

💡 Some operations (like sorting) have a high startup cost because they must do significant work before returning even the first row.

---

## 6. Rows: Estimated vs Actual

**In `rows=5000`** — this is the planner's **estimate**, made *before* running the query, based on table statistics.

**In `EXPLAIN ANALYZE`'s `actual rows=...`** — this is what the operation **actually** produced.

⚠️ **Notes & Caveats — a red flag to watch for:**
```
Estimated: rows=10
Actual:    rows=2,000,000
```
A massive mismatch like this suggests PostgreSQL's statistics are stale or the planner mis-estimated — which can lead it to choose a worse execution strategy than it otherwise would.

---

## 7. `width`

`width=40` means PostgreSQL estimates the **average output row size** to be about 40 bytes — used internally to estimate memory and data-movement costs. For beginners: `width` ≈ estimated row size.

---

## 8. `actual time` & `loops` — Reading Multi-Row Operations

```
Index Scan using employees_pkey on employees
(cost=0.29..8.30 rows=1 width=40)
(actual time=0.020..0.022 rows=1 loops=1)
```

| Field | Meaning |
|---|---|
| `actual time=0.020..0.022` | Real milliseconds: time to first row .. time to all rows |
| `actual rows=1` | Rows actually produced **per loop** |
| `loops=1` | How many times this operation executed |

⚠️ **Notes & Caveats — `loops` is critical inside joins:**
```
actual time=0.01..0.05 rows=10 loops=100
```
This operation ran **100 times**, producing about 10 rows **each time** — so the *total* rows processed across all loops (≈1,000) can be far larger than the `rows=10` figure alone suggests. Always multiply mentally when `loops > 1`.

---

## 9. `Filter` & `Rows Removed by Filter`

```
Seq Scan on employees
  Filter: (salary > 50000)
  Rows Removed by Filter: 999900
```

| Line | Meaning |
|---|---|
| `Filter: (...)` | The condition applied while scanning |
| `Rows Removed by Filter` | How many rows were read but then rejected |

💡 **Why this matters** — If PostgreSQL scanned 1,000,000 rows and rejected 999,900 of them, that's a strong signal an index on the filtered column could help avoid reading all those unnecessary rows in the first place.

---

## 10. Bitmap Scan

**Definition** — A strategy PostgreSQL uses when an index finds **many** matching rows — instead of fetching them one at a time (like a plain Index Scan), it builds a "map" of matching table locations first, then reads those table pages efficiently.

```
Bitmap Index Scan
        ↓
Find all matching locations
        ↓
Bitmap Heap Scan
        ↓
Read the required table pages efficiently
```

💡 **When it appears** — Typically when a condition matches a moderate-to-large chunk of the table (e.g., 100,000 out of 1,000,000 rows) — too many for a one-row-at-a-time index scan to be ideal, but still selective enough that a full sequential scan isn't the best choice either.

---

## 11. `Sort` and `Aggregate` Nodes

**`Sort`** — appears for queries with `ORDER BY`. Sorting millions of rows can be expensive; a matching B-Tree index can sometimes let PostgreSQL skip this node entirely (Part 29, Section 5).

**`Aggregate`** (`HashAggregate` or `GroupAggregate`) — appears for `GROUP BY`/aggregate queries like:
```sql
SELECT department, AVG(salary) FROM employees GROUP BY department;
```

---

## 12. The Golden Rule — Judge Cost, Not Scan Type

❌ **Common Mistakes** — Saying "this plan looks bad because it has a `Seq Scan`."

✅ **Instead ask:** *"Is this operation actually expensive for what this query needs?"*

- A `Seq Scan` on 10 rows → almost certainly fine, don't touch it.
- A `Seq Scan` on 100 million rows, returning only a handful of matches → worth investigating.

Recall from Part 30: PostgreSQL's planner chooses whichever strategy it estimates to be **cheapest** — `Seq Scan` is not inherently "bad," and `Index Scan` is not inherently "good."

---

## 13. A Full Optimization Investigation, Start to Finish

```sql
EXPLAIN ANALYZE
SELECT * FROM employees WHERE email = 'tushar@example.com';
```
**Before:**
```
Seq Scan on employees
(actual time=0.5..500.0 rows=1 loops=1)
Rows Removed by Filter: 999999
```
**Diagnosis:** 1 million rows scanned to find exactly 1 match → an index on `email` is likely to help.
```sql
CREATE INDEX idx_employees_email ON employees(email);
```
**After:**
```sql
EXPLAIN ANALYZE
SELECT * FROM employees WHERE email = 'tushar@example.com';
```
```
Index Scan using idx_employees_email
(actual time=0.02..0.03 rows=1 loops=1)
```
**Result:** ~500ms → ~0.03ms — a real, *measured* improvement, not a guess.

---

## 14. Interview Q&A

**Q: What is query optimization?**
A: The process of improving a SQL query so it produces the same result with less execution time and fewer resources — commonly done by examining the execution plan with `EXPLAIN`/`EXPLAIN ANALYZE`, identifying expensive operations, and then addressing them through indexing, restructuring, or filtering.

**Q: What's the difference between estimated rows and actual rows in an execution plan?**
A: Estimated rows come from the planner's statistics-based prediction before running the query; actual rows (visible with `EXPLAIN ANALYZE`) are what the operation genuinely produced. A large mismatch between them can indicate stale statistics leading to a suboptimal plan choice.

**Q: What does a high `loops` value mean, and why does it matter?**
A: It means that plan node executed multiple times (typical inside joins) — the total rows processed is roughly `rows × loops`, which can be far larger than the displayed row count alone suggests.

**Q: What is `Rows Removed by Filter`, and why would a large value be a signal?**
A: It's the count of rows read during a scan but rejected by the filter condition. A large value relative to the table size suggests PostgreSQL is doing a lot of unnecessary reading — often a sign that an index on the filtered column could help.

**Q: When does PostgreSQL use a Bitmap Scan instead of a plain Index Scan?**
A: When an index match returns a moderate-to-large number of rows — too many to efficiently fetch one at a time via a plain index scan, but still selective enough that a full sequential scan isn't ideal either.

**Q: Is a `Seq Scan` always something to fix?**
A: No — for small tables or queries needing a large fraction of the table anyway, a sequential scan can genuinely be the cheapest strategy. The concern is proportional: a `Seq Scan` on millions of rows returning very few matches is far more suspicious than the same operation on a small table.

---

## 15. Quick Revision Sheet

| Field | Meaning |
|---|---|
| `cost=a..b` | Startup cost `a`, total cost `b` (internal units, not ms) |
| `rows=n` | Estimated row count |
| `actual rows=n` | Real row count (needs `EXPLAIN ANALYZE`) |
| `width=n` | Estimated average row size in bytes |
| `actual time=a..b` | Real ms: first row .. all rows |
| `loops=n` | How many times this node executed |
| `Filter: (...)` | The condition being applied |
| `Rows Removed by Filter` | Rows read but rejected |

---

## 16. Cheat Sheet

```sql
-- ── THE OPTIMIZATION LOOP ─────────────────
EXPLAIN ANALYZE SELECT ...;      -- 1. measure
-- 2. identify the expensive node (scan type, rows removed, loops)
-- 3. make ONE change (index, rewrite, filter earlier)
EXPLAIN ANALYZE SELECT ...;      -- 4. measure again, compare

-- ── SAFE EXPLAIN ANALYZE FOR WRITES ───────
BEGIN;
EXPLAIN ANALYZE DELETE FROM employees WHERE id = 10;
ROLLBACK;

-- ── SPOT A MISSING INDEX ──────────────────
-- Look for: Seq Scan + large "Rows Removed by Filter" on a huge table
EXPLAIN ANALYZE SELECT * FROM employees WHERE email = 'x@example.com';
CREATE INDEX idx_employees_email ON employees(email);
EXPLAIN ANALYZE SELECT * FROM employees WHERE email = 'x@example.com';  -- compare
```

---

## 17. Preview of Part 36

| Topic | What You'll Learn |
|---|---|
| Nested Loop Join | For-each-row-check-the-other-table strategy |
| Hash Join | Building and probing a hash table for equality joins |
| Merge Join | Walking two pre-sorted inputs together |
| Join order & filtering | Why reducing data *before* an expensive join matters |
