# SQL & PostgreSQL Complete Notes — Part 29: Indexes II — Unique, Composite, Partial & Expression

## 📑 Table of Contents
1. `UNIQUE INDEX`
2. `UNIQUE` Constraint vs `UNIQUE INDEX`
3. Composite (Multi-Column) Index
4. The Leftmost Prefix Rule ⭐⭐⭐⭐⭐
5. Column Order Golden Rule: Equality → Sort → Range
6. Partial Index
7. Expression Index
8. Covering Indexes (`INCLUDE`)
9. Interview Q&A
10. Quick Revision Sheet
11. Cheat Sheet
12. Preview of Part 30

**📋 Series Coverage (Part 29):** `UNIQUE INDEX`, unique constraint vs unique index, composite/multi-column indexes, the leftmost prefix rule, index column ordering (equality/sort/range), partial indexes, expression indexes, covering indexes with `INCLUDE`, index-only scans

---

## 1. `UNIQUE INDEX`

**Definition** — An index that also enforces that no two rows share the same value in the indexed column(s).

**Syntax**
```sql
CREATE UNIQUE INDEX index_name ON table_name(column_name);
```

**Example**
```sql
CREATE UNIQUE INDEX idx_email ON employees(email);
```
```sql
INSERT INTO employees (email) VALUES ('aman@gmail.com');   -- ✅ first time
INSERT INTO employees (email) VALUES ('aman@gmail.com');   -- ❌ duplicate key value violates unique constraint
```

---

## 2. `UNIQUE` Constraint vs `UNIQUE INDEX`

| | `UNIQUE` Constraint | `CREATE UNIQUE INDEX` |
|---|---|---|
| Written as | `email TEXT UNIQUE` inside `CREATE TABLE` | Separate `CREATE UNIQUE INDEX` statement |
| Conceptual purpose | Data-integrity **rule** | The **structure** enforcing it |
| Under the hood | PostgreSQL implements it using a unique index | — |

💡 **How to Choose** — Use a `UNIQUE` constraint when your intent is a straightforward business rule ("emails must be unique"). Reach for `CREATE UNIQUE INDEX` directly when you need something a plain constraint can't express — e.g., a unique **partial** index or a unique index on an **expression** (both covered below).

---

## 3. Composite (Multi-Column) Index

**Definition** — A single index built across **multiple columns together**, rather than one index per column.

**Why It Exists** — If your queries commonly filter on two columns together (e.g., `department` AND `salary`), one combined index is often far more efficient than PostgreSQL trying to intersect two separate single-column indexes.

**Syntax**
```sql
CREATE INDEX index_name ON table_name(column1, column2);
```

**Example**
```sql
CREATE INDEX idx_department_salary ON employees(department, salary);
```

💡 **Analogy** — A phone book sorted by **last name, then first name**. Within each last name, the first names are sorted — but first names are *not* globally sorted across the entire book.

```
Row | department (col 1) | salary (col 2)
1   | Engineering         | 80,000
2   | Engineering         | 95,000
3   | Engineering         | 120,000
4   | Sales               | 60,000
5   | Sales               | 105,000
```
*(Notice `salary` is only sorted **within** each department — not globally top-to-bottom.)*

---

## 4. The Leftmost Prefix Rule ⭐⭐⭐⭐⭐

**Definition** — A composite index on `(col1, col2)` can only be used efficiently when the query's condition includes `col1` — either alone, or `col1` together with `col2`. A condition on `col2` **alone** generally cannot use the index efficiently.

**Given:**
```sql
CREATE INDEX idx_dept_salary ON employees(department, salary);
```

| Query | Uses Index? | Why |
|---|---|---|
| `WHERE department = 'IT'` | ✅ Yes | Starts with the leftmost column |
| `WHERE department = 'IT' AND salary > 50000` | ✅ Yes | Uses both columns, in order |
| `WHERE salary > 50000` (alone) | ❌ Usually not | Skips the leftmost column — PostgreSQL doesn't know which department "section" to search inside |

💡 **Analogy** — In a phone book sorted by (Last Name, First Name), searching for "everyone named John" is useless without a last name — John could be scattered across every letter of the book. You'd have to scan the whole thing.

💡 **Memory trick:** For `(department, salary)`, always think **Department → Salary** — you must start your search from the leftmost column.

---

## 5. Column Order Golden Rule: Equality → Sort → Range

When designing a composite index to satisfy both a `WHERE` clause and an `ORDER BY`, order the columns in this sequence:

```
1. EQUALITY columns first   (columns compared with = or IN)
2. SORT columns second       (columns used in ORDER BY)
3. RANGE columns last         (columns compared with >, <, BETWEEN)
```

**Why this matters — an index can eliminate a `Sort` step**
```sql
SELECT * FROM employees
WHERE status = 'Active' AND hire_date > '2020-01-01'
ORDER BY salary DESC;
```

| Index | Result |
|---|---|
| `(status, hire_date, salary)` ❌ | `hire_date` is a range, breaking the chain — PostgreSQL must still manually sort salaries in memory |
| `(status, salary, hire_date)` ✅ | Anchors on `status`, reads salaries pre-sorted, filters `hire_date` as it goes — **no memory sort needed** |

⚠️ **Notes & Caveats — direction matters too:**
```sql
-- ✅ Works — reads the index forward
ORDER BY department ASC, salary ASC

-- ✅ Works — reads the index backward
ORDER BY department DESC, salary DESC

-- ❌ Requires an extra Sort step — mixed directions don't align with a plain ascending index
ORDER BY department ASC, salary DESC
```
For genuinely mixed-direction needs, declare it explicitly when creating the index:
```sql
CREATE INDEX idx_dept_salary ON employees(department ASC, salary DESC);
```

---

## 6. Partial Index

**Definition** — An index built only on the rows that satisfy a specified condition — not the whole table.

**Why It Exists** — If 99% of your queries only ever ask for `status = 'Active'` rows, there's little value indexing the inactive 99% (or whatever proportion is rarely queried).

**Syntax**
```sql
CREATE INDEX index_name ON table_name(column) WHERE condition;
```

**Example**
```sql
CREATE INDEX idx_active_employees ON employees(name) WHERE status = 'Active';
```
```sql
SELECT * FROM employees WHERE status = 'Active' AND name = 'Aman';   -- ✅ can use the partial index
SELECT * FROM employees WHERE status = 'Inactive';                    -- ❌ can't — those rows were never indexed
```

**Advantages:** smaller index size → less storage, faster search, faster maintenance.

💡 **Great candidates:** active users, pending orders, unpaid invoices, open tickets, published posts — anything where most queries target a specific, stable subset.

⚠️ **Notes & Caveats** — Only queries whose `WHERE` clause is *compatible with* the partial index's condition can benefit from it.

---

## 7. Expression Index

**Definition** — An index built on the **result of an expression or function**, rather than a raw column value.

**The problem it solves**
```sql
SELECT * FROM employees WHERE LOWER(email) = 'abc@gmail.com';
```
A plain index on `email` doesn't directly help here, because the query is searching on `LOWER(email)`, a *computed* value.

**Syntax**
```sql
CREATE INDEX index_name ON table_name(expression);
```

**Example**
```sql
CREATE INDEX idx_lower_email ON employees(LOWER(email));
```
Now the query above can use this index, since the index itself stores the lowercased values.

---

## 8. Covering Indexes (`INCLUDE`)

**Definition** — An index that stores **extra, non-indexed columns** directly alongside the indexed value, so a query can be satisfied entirely from the index — without a trip back to the table.

**Why It Exists** — Every normal index lookup in PostgreSQL is a two-step process: (1) search the index, (2) follow a pointer back to the table to fetch the rest of the row. A covering index can skip step 2 entirely for queries that only need the included columns.

**Syntax**
```sql
CREATE INDEX index_name ON table_name(indexed_column) INCLUDE (extra_column1, extra_column2);
```

**Example**
```sql
CREATE INDEX idx_users_email_info
ON users (email)
INCLUDE (first_name, last_name);
```
```sql
SELECT first_name, last_name FROM users WHERE email = 'test@example.com';
```
Since `first_name` and `last_name` are already stored right inside the index, PostgreSQL can perform an **Index-Only Scan** — finding the email *and* returning the requested columns without ever touching the underlying table.

💡 **How to Choose** — Reach for `INCLUDE` when a specific, frequently-run query only ever needs a small, predictable set of extra columns alongside your search condition.

---

## 9. Interview Q&A

**Q: What's the difference between a `UNIQUE` constraint and a `UNIQUE INDEX`?**
A: They're conceptually different framings of the same underlying mechanism — a `UNIQUE` constraint expresses a data-integrity rule, while PostgreSQL implements that rule internally using a unique index. You'd reach for `CREATE UNIQUE INDEX` directly when you need capabilities a plain constraint doesn't support, like a partial or expression-based unique index.

**Q: Given an index on `(department, salary)`, will `WHERE salary > 50000` (alone) use it efficiently?**
A: Usually not — this violates the leftmost prefix rule. Since the index is sorted by `department` first, PostgreSQL can't efficiently jump to matching salaries without first knowing which department to look under.

**Q: What's the "golden rule" for ordering composite index columns when both filtering and sorting are involved?**
A: Equality columns first, sort columns second, range columns last — this maximizes the chance PostgreSQL can use the index both to filter and to avoid a separate in-memory sort step.

**Q: When would you use a partial index instead of a normal index?**
A: When queries consistently target a specific, stable subset of rows (e.g., only active records) — a partial index covering just that subset is smaller, faster to search, and cheaper to maintain than indexing the entire table.

**Q: What problem does an expression index solve?**
A: It allows an index to match queries that filter on a *computed* value (like `LOWER(email)`) rather than the raw column — without an expression index, such a query can't benefit from a plain index on the raw column.

**Q: What is an Index-Only Scan, and how does `INCLUDE` enable it?**
A: An Index-Only Scan answers a query using only the index, without a separate trip to fetch the row from the table. `INCLUDE` lets you store extra "payload" columns directly in the index so that queries needing only the indexed column plus those extras can be satisfied this way.

---

## 10. Quick Revision Sheet

| Need | Syntax |
|---|---|
| Prevent duplicates + fast lookup | `CREATE UNIQUE INDEX name ON t(col);` |
| Index multiple columns together | `CREATE INDEX name ON t(col1, col2);` |
| Query must start with... | The index's **leftmost** column |
| Index only some rows | `CREATE INDEX name ON t(col) WHERE condition;` |
| Index a computed value | `CREATE INDEX name ON t(LOWER(col));` |
| Avoid a heap trip (Index-Only Scan) | `CREATE INDEX name ON t(col) INCLUDE (extra1, extra2);` |

---

## 11. Cheat Sheet

```sql
-- ── UNIQUE INDEX ──────────────────────────
CREATE UNIQUE INDEX idx_email ON employees(email);

-- ── COMPOSITE INDEX ───────────────────────
CREATE INDEX idx_dept_salary ON employees(department, salary);
-- Leftmost prefix: WHERE department = 'IT' ✅ | WHERE salary > 50000 alone ❌

-- ── COLUMN ORDER FOR WHERE + ORDER BY ─────
-- Equality → Sort → Range
CREATE INDEX idx_status_salary_hire ON employees(status, salary, hire_date);
CREATE INDEX idx_dept_salary_desc ON employees(department ASC, salary DESC);   -- explicit mixed direction

-- ── PARTIAL INDEX ─────────────────────────
CREATE INDEX idx_active_employees ON employees(name) WHERE status = 'Active';

-- ── EXPRESSION INDEX ──────────────────────
CREATE INDEX idx_lower_email ON employees(LOWER(email));

-- ── COVERING INDEX (INCLUDE) ──────────────
CREATE INDEX idx_users_email_info ON users(email) INCLUDE (first_name, last_name);
```

---

## 12. Preview of Part 30

| Topic | What You'll Learn |
|---|---|
| Clustered vs Non-Clustered | What PostgreSQL's "heap" architecture actually means |
| Hash, GIN, GiST, BRIN | Specialized index types and when each shines |
| `EXPLAIN` / `EXPLAIN ANALYZE` | Reading what PostgreSQL actually did |
| `CREATE INDEX CONCURRENTLY` | Building indexes without locking production writes |
