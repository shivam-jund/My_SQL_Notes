# SQL & PostgreSQL Complete Notes — Part 37: Query Optimization III — CTE vs Subquery & Partitioning

## 📑 Table of Contents
1. CTE vs Subquery — Is One Faster?
2. CTE Inlining (PostgreSQL 12+)
3. `MATERIALIZED` and `NOT MATERIALIZED`
4. When to Use a CTE vs a Subquery
5. What Is Partitioning?
6. Partition Pruning
7. `RANGE` Partitioning
8. `LIST` Partitioning
9. `HASH` Partitioning
10. Partitioning vs Indexing
11. Partitioning + Indexing Together
12. Dropping Old Partitions (Data Lifecycle)
13. Choosing a Partition Key
14. Interview Q&A
15. Quick Revision Sheet
16. Cheat Sheet
17. Preview of Part 38

**📋 Series Coverage (Part 37):** CTE vs subquery performance myths, CTE inlining, `MATERIALIZED`/`NOT MATERIALIZED`, partitioning fundamentals, partition pruning, `RANGE`/`LIST`/`HASH` partitioning, partitioning vs indexing, dropping old partitions, choosing a partition key

---

## 1. CTE vs Subquery — Is One Faster?

Recall the CTE syntax from Part 16. Compare:
```sql
-- Subquery
SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- CTE
WITH avg_salary AS (
    SELECT AVG(salary) AS avg FROM employees
)
SELECT * FROM employees WHERE salary > (SELECT avg FROM avg_salary);
```

⭐ **The short answer:** *Neither is automatically faster.* PostgreSQL's optimizer can often transform both forms into similar execution strategies.

❌ **Common Mistakes** — Answering an interview question with a flat "CTEs are faster" or "subqueries are faster." The **correct** approach is always: *check `EXPLAIN ANALYZE`.*

💡 So why use a CTE at all, if it's not automatically faster? **Readability** — breaking a deeply nested query into named, sequential steps (Part 16) is the real advantage, not raw performance.

---

## 2. CTE Inlining (PostgreSQL 12+)

⚠️ **Notes & Caveats — an important version-dependent fact:** Before PostgreSQL 12, CTEs were generally treated as an "optimization fence" (always computed as a separate, materialized step). **From PostgreSQL 12 onward**, the planner can often **inline** many non-recursive CTEs — meaning the CTE's logic gets folded directly into the surrounding query and optimized as a whole, rather than being forced to execute as an isolated step.

```sql
WITH active_employees AS (
    SELECT * FROM employees WHERE active = true
)
SELECT * FROM active_employees WHERE salary > 50000;
```
Modern PostgreSQL may effectively optimize this **as if** it had been written:
```sql
SELECT * FROM employees WHERE active = true AND salary > 50000;
```

💡 If you've seen older tutorials claiming "CTEs always materialize" — that's outdated for modern PostgreSQL.

---

## 3. `MATERIALIZED` and `NOT MATERIALIZED`

**Definition** — Explicit keywords to override PostgreSQL's default inlining behavior.

```sql
WITH active_employees AS MATERIALIZED (
    SELECT * FROM employees WHERE active = true
)
SELECT * FROM active_employees WHERE salary > 50000;
```
```sql
WITH active_employees AS NOT MATERIALIZED (
    SELECT * FROM employees WHERE active = true
)
SELECT * FROM active_employees WHERE salary > 50000;
```

| Keyword | Effect |
|---|---|
| `MATERIALIZED` | Force PostgreSQL to compute this CTE **once**, store the result, and reuse it — even if it's referenced multiple times |
| `NOT MATERIALIZED` | Allow PostgreSQL to fold/inline this CTE into the surrounding query when possible |

💡 **When `MATERIALIZED` helps** — If a genuinely expensive CTE result is referenced **multiple times** in the outer query, materializing it once can avoid recomputing that expensive logic repeatedly.

⚠️ **When `MATERIALIZED` can hurt** — If the CTE would produce 10 million rows but the outer query only needs 10 of them, forcing materialization means PostgreSQL computes and stores all 10 million *before* filtering — work that inlining could have avoided by pushing the filter down first.

---

## 4. When to Use a CTE vs a Subquery

| Use a **Subquery** when... | Use a **CTE** when... |
|---|---|
| The logic is small and self-contained | The query has multiple logical steps |
| No benefit to naming the intermediate result | You want readable, named intermediate steps |
| — | You need `WITH RECURSIVE` (subqueries can't do this — Part 16) |
| — | You specifically need deliberate materialization |

💡 **Mental model:** Subquery = *"I need this small calculation right here."* CTE = *"I want to name this intermediate result and treat it as a logical step."*

---

## 5. What Is Partitioning?

**Definition** — Dividing one large **logical** table into smaller **physical** pieces (partitions), while still being able to query it as a single table.

```
ONE BIG TABLE (orders)
      ↓
 ┌────┼────┬────┐
 ↓    ↓    ↓    ↓
2024 2025 2026 2027   ← physical partitions
```
You still write `SELECT * FROM orders ...` — PostgreSQL handles routing to the right partition(s) internally.

⚠️ **Notes & Caveats** — Think of this as **one logical table**, not four separate unrelated tables.

---

## 6. Partition Pruning

**Definition** — PostgreSQL's ability to **skip entire partitions** that can't possibly contain rows matching the query's `WHERE` condition.

```sql
SELECT * FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01';
```
```
orders_2024 ❌ (skipped entirely)
orders_2025 ❌ (skipped entirely)
orders_2026 ✅ (only this one is actually scanned)
orders_2027 ❌ (skipped entirely)
```

💡 This is the entire *point* of partitioning — for a well-chosen partition key, queries only have to consider a fraction of the total data.

---

## 7. `RANGE` Partitioning

**Definition** — Divides data based on **ranges** of values — extremely common for dates.

**Syntax**
```sql
CREATE TABLE orders (
    id BIGINT,
    order_date DATE,
    amount NUMERIC
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2026 PARTITION OF orders
FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

CREATE TABLE orders_2027 PARTITION OF orders
FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');
```

⚠️ **Notes & Caveats** — You `INSERT` into the parent table (`orders`), never directly into a specific partition — PostgreSQL automatically routes each row to the correct partition based on its `order_date`.

💡 **Best for:** dates, age ranges, price ranges — any naturally ordered numeric/date value.

---

## 8. `LIST` Partitioning

**Definition** — Divides data based on **specific, discrete values** — a natural fit for categorical data.

**Syntax**
```sql
CREATE TABLE customers (
    id BIGINT, name TEXT, country TEXT
) PARTITION BY LIST (country);

CREATE TABLE customers_india PARTITION OF customers FOR VALUES IN ('India');
CREATE TABLE customers_usa   PARTITION OF customers FOR VALUES IN ('USA');
```

💡 **Best for:** country, region, department, status, business unit.

---

## 9. `HASH` Partitioning

**Definition** — Distributes rows relatively **evenly** across a fixed number of partitions using a hash function, when there's no meaningful range or category to partition by.

**Syntax**
```sql
CREATE TABLE customers (id BIGINT, name TEXT) PARTITION BY HASH (id);

CREATE TABLE customers_p0 PARTITION OF customers FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE customers_p1 PARTITION OF customers FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... and so on for REMAINDER 2, 3
```

💡 **Best for:** `customer_id`, `user_id`, `account_id` — columns you want to spread evenly, without a natural range or category.

**Memory trick for all three types:**
```
RANGE → "between"     (e.g. dates)
LIST  → "which one?"  (e.g. country)
HASH  → "spread evenly"
```

---

## 10. Partitioning vs Indexing

❌ **Common Mistakes** — Thinking "if I already have an index, why bother partitioning?" They solve **different** problems.

| | Index | Partitioning |
|---|---|---|
| Purpose | Find rows efficiently **within** a table | Reduce **how much data** needs to be considered at all |

💡 **Analogy** — A library with 1 million books:
- **Index** = a catalog card: "Book title → shelf number" — helps you find a specific book quickly.
- **Partitioning** = dividing the library into floors by year: "2026 books → Floor 3." If someone wants only 2026 books, you never even walk onto the other floors.

---

## 11. Partitioning + Indexing Together

Both can — and often should — be used together:
```
orders
   ↓ Partition by order_date
2025 partition   2026 partition   2027 partition
   ↓ Index inside each partition (e.g., on customer_id)
```
```sql
SELECT * FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01'
AND customer_id = 100;
```
```
Partition pruning → only the 2026 partition is scanned
Index on customer_id → quickly finds customer 100 within it
```

---

## 12. Dropping Old Partitions (Data Lifecycle)

**A major practical benefit** — suppose company policy says "delete all data older than 2024."
```sql
-- ❌ Deleting 100 million individual rows can be slow
DELETE FROM orders WHERE order_date < '2024-01-01';
```
```sql
-- ✅ Dropping (or detaching) a whole partition is dramatically cheaper
DROP TABLE orders_2023;
```
💡 This is one of the strongest real-world arguments for time-based partitioning — old data retention/cleanup becomes a near-instant structural operation instead of a massive row-by-row deletion.

---

## 13. Choosing a Partition Key

⚠️ **Notes & Caveats** — A good partition key is typically a column that appears **frequently in your `WHERE` filters**. If your table is partitioned by `order_date` but most queries filter by `customer_name`, partition pruning won't help those queries at all — the partition key must match how the data is actually queried.

❌ **Common Mistakes**
- Partitioning every table "just in case" — only genuinely large tables with a workload that benefits from pruning are good candidates.
- Assuming more partitions is always better — too many partitions adds planning and management overhead.
- Assuming partitioning automatically makes *every* query faster — it only helps queries whose filters align with the partition key.

---

## 14. Interview Q&A

**Q: Is a CTE faster than an equivalent subquery?**
A: Not necessarily — PostgreSQL (12+) can inline many non-recursive CTEs, so an equivalent CTE and subquery often produce similar execution plans. Performance should be verified with `EXPLAIN ANALYZE`, not assumed from the syntax alone.

**Q: What does `MATERIALIZED` do to a CTE, and when might you want it?**
A: It forces PostgreSQL to compute the CTE once and store its result, rather than potentially inlining it into the surrounding query. It's useful when an expensive CTE result is referenced multiple times — but can hurt performance if it prevents beneficial filter pushdown.

**Q: What is partitioning, in one sentence?**
A: Dividing one large logical table into smaller physical pieces (partitions) based on a partition key, while still querying it as a single table.

**Q: What is partition pruning?**
A: PostgreSQL's ability to eliminate partitions that cannot contain rows matching the query's condition, so it only scans the relevant subset of the data.

**Q: What are the three main partitioning strategies in PostgreSQL?**
A: `RANGE` (divides by value ranges, e.g., dates), `LIST` (divides by specific discrete values, e.g., country), and `HASH` (distributes rows evenly via a hash function when there's no natural range or category).

**Q: How is partitioning different from indexing?**
A: An index helps locate rows efficiently *within* the data being searched. Partitioning reduces *how much data* needs to be considered in the first place, via pruning. They solve different problems and are often used together.

**Q: Why is dropping a partition often preferable to a bulk `DELETE`?**
A: Deleting millions of individual rows via `DELETE ... WHERE` can be slow and resource-intensive. Dropping (or detaching) an entire partition removes that data as a near-instant structural operation instead.

---

## 15. Quick Revision Sheet

| Goal | Approach |
|---|---|
| Name an intermediate step for readability | CTE |
| Small, self-contained calculation | Subquery |
| Force a CTE to compute once, reuse the result | `MATERIALIZED` |
| Allow a CTE to fold into the outer query | `NOT MATERIALIZED` (often the default behavior) |
| Divide by date/numeric ranges | `PARTITION BY RANGE` |
| Divide by discrete categories | `PARTITION BY LIST` |
| Spread rows evenly, no natural category | `PARTITION BY HASH` |
| Fast bulk deletion of old data | Drop/detach a partition |

---

## 16. Cheat Sheet

```sql
-- ── CTE MATERIALIZATION CONTROL ───────────
WITH data AS MATERIALIZED (SELECT ... FROM huge_table)
SELECT ... FROM data a JOIN data b ON ...;

WITH data AS NOT MATERIALIZED (SELECT ... FROM t WHERE ...)
SELECT ... FROM data WHERE ...;

-- ── RANGE PARTITIONING ────────────────────
CREATE TABLE orders (id BIGINT, order_date DATE, amount NUMERIC)
PARTITION BY RANGE (order_date);

CREATE TABLE orders_2026 PARTITION OF orders
FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- ── LIST PARTITIONING ─────────────────────
CREATE TABLE customers (id BIGINT, name TEXT, country TEXT)
PARTITION BY LIST (country);

CREATE TABLE customers_india PARTITION OF customers FOR VALUES IN ('India');

-- ── HASH PARTITIONING ─────────────────────
CREATE TABLE customers (id BIGINT, name TEXT) PARTITION BY HASH (id);
CREATE TABLE customers_p0 PARTITION OF customers FOR VALUES WITH (MODULUS 4, REMAINDER 0);

-- ── VERIFY PRUNING ────────────────────────
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01';
-- look for only orders_2026 appearing in the plan

-- ── FAST OLD-DATA CLEANUP ─────────────────
DROP TABLE orders_2023;
```

---

## 17. Preview of Part 38

| Topic | What You'll Learn |
|---|---|
| Users & Roles | `CREATE USER`, `CREATE ROLE`, `LOGIN`, role membership |
| Role attributes | `SUPERUSER`, `CREATEDB`, `CREATEROLE` |
| Least privilege | The core security principle behind all PostgreSQL access control |
