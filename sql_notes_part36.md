# SQL & PostgreSQL Complete Notes — Part 36: Query Optimization II — Join Strategies

## 📑 Table of Contents
1. What Is Join Optimization?
2. Nested Loop Join
3. Nested Loop With an Index
4. When Nested Loop Is Good / Bad
5. Hash Join
6. When Hash Join Is Good / Not Suitable
7. Merge Join
8. When Merge Join Is Good
9. The Three Join Strategies — Comparison
10. Join Order & Why It Matters
11. Indexing Join Columns
12. Filtering Before Joining
13. Reading a Join in `EXPLAIN ANALYZE`
14. Common Join Optimization Mistakes
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Preview of Part 37

**📋 Series Coverage (Part 36):** Nested Loop Join, Hash Join, Merge Join, when PostgreSQL picks each strategy, join order and intermediate result size, indexing foreign-key/join columns, filtering before joining, reading join nodes in `EXPLAIN ANALYZE`, common join optimization misconceptions

---

## 1. What Is Join Optimization?

**Definition** — Finding the most efficient way for PostgreSQL to match rows between two (or more) tables during a `JOIN`.

**The core question** — Given
```sql
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.id;
```
PostgreSQL has **multiple possible strategies** to actually perform this match. The planner picks whichever it estimates will be cheapest. The three main strategies are: **Nested Loop**, **Hash Join**, and **Merge Join**.

---

## 2. Nested Loop Join

**Definition** — For each row in one table (the "outer" table), search the other table (the "inner" table) for matches.

```
Take Aman   → check departments
Take Ravi   → check departments
Take Neha   → check departments
```
With 3 employees and 2 departments, that's conceptually `3 × 2 = 6` comparisons — trivial for tiny tables.

---

## 3. Nested Loop With an Index

**This is where Nested Loop becomes genuinely powerful at scale.**

Suppose `employees` has 10 million rows, `departments` has 100 rows, and `employees.department_id` is indexed. PostgreSQL can:
```
For each department (100 of them)
       ↓
Use the index on employees.department_id
       ↓
Find matching employees directly — no full scan needed
```
So Nested Loop does **not** necessarily mean `10,000,000 × 100` comparisons — with the right index, it can be extremely efficient.

---

## 4. When Nested Loop Is Good / Bad

✅ **Good when:** the outer table is relatively small, **and** the inner table can be searched efficiently — especially via an index.

❌ **Bad when:** both tables are huge and there's no useful index — PostgreSQL could end up doing an enormous number of comparisons.

---

## 5. Hash Join

**Definition** — PostgreSQL builds an in-memory **hash table** from one input (mapping join-key → row), then scans the other input and probes the hash table for matches.

```
departments
     ↓ build hash table
10 → IT
20 → HR
30 → Finance

employees
     ↓
department_id = 20
     ↓ hash lookup
     20 → HR  → MATCH!
```

⚠️ **Notes & Caveats** — A Hash Join does **not** require an index on the join columns — it builds its own temporary hash structure instead.

---

## 6. When Hash Join Is Good / Not Suitable

✅ **Good for:** large tables joined on **equality** (`=`) conditions.

❌ **Not suitable for:** non-equality join conditions (`>`, `<`, range-based joins like `ON e.salary > s.minimum_salary`) — hash joins are fundamentally built around equality matching.

---

## 7. Merge Join

**Definition** — Works when **both** inputs are already sorted (or can be cheaply sorted) by the join key — PostgreSQL then walks through both sorted lists together in one pass, similar to the merge step in merge sort.

```
Table A sorted: 10, 20, 30, 40
Table B sorted: 10, 20, 30, 40
        ↓
Walk through both together, matching values as they align
```

---

## 8. When Merge Join Is Good

✅ **Good when:** both inputs are already sorted by the join key, or PostgreSQL can obtain sorted input relatively cheaply (e.g., via an existing index) — particularly effective for large datasets.

---

## 9. The Three Join Strategies — Comparison

| Join | Basic Idea | Often Good For |
|---|---|---|
| **Nested Loop** | For each outer row, find matching inner rows | Small outer table + a useful index on the inner table |
| **Hash Join** | Build a hash table, then probe it | Large **equality** joins |
| **Merge Join** | Walk two sorted inputs together | Large, already-sorted inputs |

⭐ **A very common interview trap:** *"Which join is fastest?"* — There's **no universal answer**. The correct response: *"It depends on data size, indexes, join condition, sort order, and statistics — PostgreSQL's planner picks whichever strategy it estimates to be cheapest."*

---

## 10. Join Order & Why It Matters

**The principle** — Reduce the amount of data flowing into an expensive operation *before* it happens, whenever possible.

```
10,000,000 employees
        ↓ filter first
     10,000 matching
        ↓ THEN join
        100 final rows
```
...is generally far better than...
```
10,000,000 employees
        ↓ join first
   Huge intermediate result
        ↓ THEN filter
        100 final rows
```

💡 **Notes & Caveats** — Modern PostgreSQL's optimizer can often perform this kind of "filter pushdown" automatically. As a SQL writer, you should still express your logic clearly — don't assume you must manually restructure every query, but be aware of this principle when investigating a genuinely slow join.

---

## 11. Indexing Join Columns

Recall from Part 28: the **referenced** side (usually a `PRIMARY KEY`) is automatically indexed, but the **referencing** foreign-key side is **not**.

```sql
SELECT * FROM employees e
JOIN departments d ON e.department_id = d.id;
```
```sql
CREATE INDEX idx_employee_department ON employees(department_id);
```
This gives PostgreSQL a fast way to look up matching employees for any given department — particularly valuable if the planner chooses a Nested Loop strategy.

---

## 12. Filtering Before Joining

```sql
SELECT * FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.active = true;
```
If only 10% of employees are `active`, reducing that input matters:
```
10,000,000 employees
        ↓ active = true
    1,000,000
        ↓ JOIN
```
rather than carrying all 10 million rows through the join unnecessarily. Again, PostgreSQL's optimizer often does this automatically — don't manually wrap every query in a filtering subquery "just in case," since modern PostgreSQL can usually optimize the straightforward version just as well.

---

## 13. Reading a Join in `EXPLAIN ANALYZE`

```sql
EXPLAIN ANALYZE
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.id;
```
```
Nested Loop (actual time=0.05..100.00 rows=5000 loops=1)
  -> Seq Scan on departments (actual rows=100)
  -> Index Scan on employees (actual rows=50 loops=100)
```

**Reading this:**
```
100 departments (outer)
        ×
50 matching employees per department (inner, via index)
        =
~5,000 output rows
```
This matches the earlier point about `loops` (Part 35, Section 8) — the inner `Index Scan` ran 100 times (once per outer department row), producing about 50 rows each time.

---

## 14. Common Join Optimization Mistakes

❌ Thinking Nested Loop is inherently bad — with a good index on the inner side, it can be excellent.

❌ Thinking Hash Join is always fastest — it isn't suited to non-equality conditions, and for smaller data a Nested Loop might well be cheaper.

❌ Creating indexes on every join column without checking actual workload — not automatically necessary.

❌ Assuming CTEs always improve join performance — not necessarily true (fully explored in Part 37).

❌ Ignoring table size — joining `10 rows + 10 rows` is a completely different problem than `100 million + 100 million`.

---

## 15. Interview Q&A

**Q: What are the main join algorithms PostgreSQL uses, and how does it choose between them?**
A: Nested Loop, Hash Join, and Merge Join. The planner selects whichever it estimates to be cheapest, based on table sizes, available indexes, the join condition, data ordering, and table statistics.

**Q: When is a Nested Loop join typically effective?**
A: When the outer relation is relatively small and the inner relation can be searched efficiently — especially with an index on the join column.

**Q: Does a Hash Join require an index on the join columns?**
A: No — it builds its own temporary in-memory hash table from one input and probes it with the other, so it doesn't rely on a pre-existing index for the join itself.

**Q: When would Merge Join be chosen over the other two?**
A: When both inputs are already sorted by the join key (or can be cheaply obtained in sorted order), which is particularly effective for large datasets.

**Q: Why does join order/filtering matter for performance?**
A: Reducing the number of rows flowing into an expensive join *before* it happens (rather than joining everything and filtering afterward) minimizes the size of intermediate results the database has to process.

**Q: Are join columns always in need of a manual index?**
A: Not automatically — PostgreSQL indexes the primary-key side of a relationship automatically, but the foreign-key side is not auto-indexed. Whether it's worth adding manually depends on how frequently that join or lookup actually occurs in your workload.

---

## 16. Quick Revision Sheet

| Strategy | Mental Model | Best For |
|---|---|---|
| Nested Loop | Small outside + quick lookup inside | Small outer table, indexed inner table |
| Hash Join | Build hash, then look up matches | Large equality joins |
| Merge Join | Sorted + sorted, walk together | Large, pre-sorted inputs |

---

## 17. Cheat Sheet

```sql
-- ── SEE WHICH JOIN STRATEGY WAS CHOSEN ────
EXPLAIN ANALYZE
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.id;

-- ── INDEX THE FOREIGN-KEY SIDE OF A JOIN ──
CREATE INDEX idx_employee_department ON employees(department_id);

-- ── FILTER BEFORE AN EXPENSIVE JOIN ───────
SELECT e.name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.active = true;   -- reduces rows entering the join
```

---

## 18. Preview of Part 37

| Topic | What You'll Learn |
|---|---|
| CTE vs Subquery performance | `MATERIALIZED` / `NOT MATERIALIZED`, CTE inlining |
| Partitioning | `RANGE`, `LIST`, `HASH` partitioning |
| Partition Pruning | How PostgreSQL skips irrelevant partitions entirely |
