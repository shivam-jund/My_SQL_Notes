# SQL & PostgreSQL Complete Notes — Part 27: Views II — Materialized Views

## 📑 Table of Contents
1. The Problem with Normal Views (Recomputation Cost)
2. What Is a Materialized View?
3. `CREATE MATERIALIZED VIEW`
4. Querying a Materialized View
5. Materialized Views Go Stale — `REFRESH MATERIALIZED VIEW`
6. View vs Materialized View — Full Comparison
7. Storage & Performance Comparison
8. Real-World Use Cases
9. Interview Q&A
10. Quick Revision Sheet
11. Cheat Sheet
12. Preview of Part 28

**📋 Series Coverage (Part 27):** why normal views can be slow for expensive queries, `CREATE MATERIALIZED VIEW`, querying a materialized view, staleness, `REFRESH MATERIALIZED VIEW`, view vs materialized view comparison, storage/performance tradeoffs, real-world use cases (dashboards, monthly balances, analytics)

> ⭐ Almost every PostgreSQL interview asks: **"What's the difference between a View and a Materialized View?"** This part answers that thoroughly.

---

## 1. The Problem with Normal Views (Recomputation Cost)

Recall from Part 26: a normal view stores **only the query**, and PostgreSQL reruns it on every access.

```sql
CREATE VIEW employee_details AS
SELECT e.name, d.department_name, e.salary
FROM employees e
JOIN departments d ON e.department_id = d.id;
```

⚠️ **Notes & Caveats** — If `employees` has 50 million rows and `departments` has 20 million, **every single query** against this view re-runs the full join from scratch. For expensive, frequently-read queries, this becomes a real performance problem.

---

## 2. What Is a Materialized View?

**Definition** — A materialized view stores the query's **result physically on disk**, computed once, rather than recalculating it on every access.

**Mental model**
```
Normal view:          query → run EVERY TIME → result
Materialized view:    query → run ONCE → store result → just READ the stored result
```

💡 **Analogy** — A normal view is like every student solving `987654 × 456789` from scratch each time they're asked. A materialized view is one student solving it once, writing the answer on the board, and everyone just reading it afterward.

---

## 3. `CREATE MATERIALIZED VIEW`

**Definition** — Creates a materialized view — runs the query once immediately and stores its result set.

**Syntax**
```sql
CREATE MATERIALIZED VIEW view_name AS
SELECT ...
FROM table_name;
```

**Example**
```sql
CREATE MATERIALIZED VIEW employee_summary AS
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
```
**What happens:** the `GROUP BY`/`AVG` calculation runs **once**, right now, and the resulting rows are saved on disk:
```
department | avg_salary
IT         | 75000
HR         | 45000
```

---

## 4. Querying a Materialized View

```sql
SELECT * FROM employee_summary;
```

⚠️ **Notes & Caveats** — Unlike a normal view, this does **not** re-run the `GROUP BY`/`AVG` calculation — it simply reads the already-stored result rows, exactly like reading a regular table.

---

## 5. Materialized Views Go Stale — `REFRESH MATERIALIZED VIEW`

**The problem**
```sql
-- Suppose a new IT employee is inserted with salary 90000, changing the true IT average to 80000
INSERT INTO employees (department, salary) VALUES ('IT', 90000);

SELECT * FROM employee_summary;
-- Still shows IT → 75000  ❌  (the OLD stored average — the base table changed, the materialized view didn't)
```

**The fix**
```sql
REFRESH MATERIALIZED VIEW employee_summary;

SELECT * FROM employee_summary;
-- Now shows IT → 80000  ✅
```

**Mental model**
```
Base table changes
       ↓
Materialized view: still shows OLD data
       ↓
REFRESH MATERIALIZED VIEW
       ↓
Materialized view: now shows CURRENT data
```

⚠️ **Notes & Caveats** — A materialized view is **never** automatically refreshed just because the base table changed — refreshing is always a deliberate, explicit action (or a scheduled job in a real system).

---

## 6. View vs Materialized View — Full Comparison

| | View | Materialized View |
|---|---|---|
| Stores | Only the query | The actual result data |
| Data freshness | Always current (reruns every time) | Can become stale until refreshed |
| Creation speed | Instant | Takes time (must compute the full result upfront) |
| Query speed | Slower for complex/expensive queries (recomputed each time) | Usually much faster (just reads stored rows) |
| Refresh needed? | No — never stale | Yes — `REFRESH MATERIALIZED VIEW` |
| Storage used | Minimal | Full result set stored on disk |

💡 **Memory trick:** normal View = always fresh, but potentially slow · Materialized View = fast, but can go stale until refreshed.

---

## 7. Storage & Performance Comparison

**Storage**
```
View               → stores only the SQL text (tiny)
Materialized View  → stores the entire result set (can be large)
```

**Performance — a 10-table join over 100 million rows**
```
Normal View:          JOIN → GROUP BY → SUM → result   (recomputed EVERY execution)
Materialized View:    (computed once, earlier)  → just READ stored rows   (fast)
```

---

## 8. Real-World Use Cases

| Scenario | Why a Materialized View Fits |
|---|---|
| **Sales dashboard** | Total/monthly/average sales don't need to be recalculated on every page load — refresh nightly |
| **Banking — monthly balances** | Millions of transactions; compute once at midnight, serve reads all day |
| **Analytics — top-selling products** | Expensive query; refresh hourly instead of running it live every minute |

---

## 9. Interview Q&A

**Q: Which is faster to query — a View or a Materialized View?**
A: For complex, frequently-read queries, a Materialized View is usually faster, because its result is precomputed and stored — a normal View recomputes the underlying query on every access.

**Q: Which always shows the absolute latest data?**
A: A normal View — it reruns its query fresh every time, so it can never be stale (unlike a Materialized View, which reflects data as of its last refresh).

**Q: Which consumes more storage?**
A: A Materialized View — it physically stores the full result set on disk, while a normal View stores only the query definition.

**Q: Do Materialized Views update automatically when the base table changes?**
A: No — they must be explicitly refreshed with `REFRESH MATERIALIZED VIEW view_name;`; otherwise they continue showing the data as of their last refresh.

**Q: Can a Materialized View become outdated?**
A: Yes — if the underlying tables change after the last refresh, the Materialized View keeps showing the old, stale stored data until it's refreshed again.

**Q: When would you choose a Materialized View over a normal View?**
A: When the underlying query is expensive (large joins, heavy aggregation) and is read far more often than the underlying data changes — e.g., dashboards, nightly reports, analytics that tolerate slightly-stale data in exchange for fast reads.

---

## 10. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Create a materialized view | `CREATE MATERIALIZED VIEW name AS SELECT ...;` |
| Query it | `SELECT * FROM name;` (reads stored data, no recompute) |
| Update stale data | `REFRESH MATERIALIZED VIEW name;` |
| Always fresh, but slower on expensive queries | Normal `VIEW` |
| Fast reads, but needs manual refresh | `MATERIALIZED VIEW` |

---

## 11. Cheat Sheet

```sql
-- ── CREATE ────────────────────────────────
CREATE MATERIALIZED VIEW employee_summary AS
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;

-- ── QUERY (reads stored data, no recompute) ──
SELECT * FROM employee_summary;

-- ── REFRESH (after base table changes) ────
REFRESH MATERIALIZED VIEW employee_summary;

-- ── DROP ──────────────────────────────────
DROP MATERIALIZED VIEW IF EXISTS employee_summary;
```

---

## 12. Preview of Part 28

| Topic | What You'll Learn |
|---|---|
| `CREATE INDEX` | Speeding up lookups |
| `UNIQUE INDEX`, Composite Index, Partial Index | Index variations |
| B-tree, Hash, GIN, GiST | Index types and when to use each |
| `EXPLAIN` / `EXPLAIN ANALYZE` | Reading a query's execution plan |
