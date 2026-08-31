# SQL & PostgreSQL Complete Notes — Part 9: UPDATE, DELETE, RETURNING & UPSERT

## 📑 Table of Contents
1. `UPDATE`
2. Updating Multiple Columns
3. Updating Multiple Rows
4. Updating Using the Existing Value
5. `UPDATE` Without `WHERE` — Dangerous
6. `UPDATE` with `AND`/`OR`/`IN`/`BETWEEN`
7. `DELETE`
8. Deleting Multiple Rows
9. `DELETE` Without `WHERE` — Extremely Dangerous
10. `DELETE` vs `DROP TABLE` vs `TRUNCATE` (Comparison)
11. `RETURNING`
12. `RETURNING` with `INSERT` / `UPDATE` / `DELETE`
13. `UPSERT` (`INSERT ... ON CONFLICT`)
14. The `EXCLUDED` Keyword
15. `ON CONFLICT DO NOTHING` vs `DO UPDATE`
16. `INSERT` vs `UPDATE` vs `UPSERT` (Comparison)
17. Interview Q&A
18. Quick Revision Sheet
19. Cheat Sheet
20. Preview of Part 10

**📋 Series Coverage (Part 9):** `UPDATE ... SET ... WHERE`, multi-column/multi-row updates, self-referencing updates (`salary = salary + x`), danger of missing `WHERE`, `DELETE FROM ... WHERE`, `DELETE` vs `DROP` vs `TRUNCATE`, `RETURNING`, `UPSERT` / `INSERT ... ON CONFLICT ... DO UPDATE / DO NOTHING`, `EXCLUDED`

> Examples use:
> ```
> employees: emp_id | name | department | salary | city
> products:  product_id | product_name | price | stock
> ```

---

## 1. `UPDATE`

**Definition** — Changes the value(s) of existing rows.

**Syntax**
```sql
UPDATE table_name
SET column = value
WHERE condition;
```

**Parameters**

| Clause | Purpose |
|---|---|
| `UPDATE table_name` | Which table |
| `SET column = value` | What changes, and to what |
| `WHERE condition` | Which rows change |

**Example**
```sql
UPDATE employees
SET salary = 55000
WHERE emp_id = 1;
```
**Output**
```
UPDATE 1
```
*(The number tells you how many rows were affected.)*

---

## 2. Updating Multiple Columns

**Syntax**
```sql
UPDATE table_name
SET col1 = val1, col2 = val2
WHERE condition;
```

**Example**
```sql
UPDATE employees
SET salary = 60000,
    city = 'Delhi',
    department = 'Sales'
WHERE emp_id = 1;
```

---

## 3. Updating Multiple Rows

**Definition** — `UPDATE` changes **every** row matching the `WHERE` condition, not just one.

**Example**
```sql
UPDATE employees
SET salary = 75000
WHERE department = 'IT';
```
**Output**
```
UPDATE 3    -- Aman, Priya, Arjun all updated
```

---

## 4. Updating Using the Existing Value

**Definition** — The new value can reference the column's **current** value before it changes.

**Example — flat raise**
```sql
UPDATE employees
SET salary = salary + 5000
WHERE department = 'IT';
```

**Example — percentage raise**
```sql
UPDATE employees
SET salary = salary * 1.10;    -- +10%
```

⚠️ **Notes & Caveats** — `salary * 1.10` = original salary + 10% (`100% + 10% = 110% = ×1.10`). For a 20% raise: `× 1.20`. For a 5% cut: `× 0.95`.

---

## 5. `UPDATE` Without `WHERE` — Dangerous

⚠️ **Notes & Caveats**
```sql
-- ⚠️ No WHERE = updates EVERY row in the table
UPDATE employees SET salary = 50000;
```

❌ **Common Mistakes**
```sql
-- ❌ Forgetting WHERE wipes every employee's salary to the same value
UPDATE employees SET salary = 50000;
```
```sql
-- ✅ Always double-check with SELECT first, then reuse the same condition
SELECT * FROM employees WHERE department = 'IT';
UPDATE employees SET salary = salary + 5000 WHERE department = 'IT';
```

💡 **Best Practices** — Before running any `UPDATE`, run the equivalent `SELECT ... WHERE ...` first and eyeball the rows that would be affected.

---

## 6. `UPDATE` with `AND`/`OR`/`IN`/`BETWEEN`

Everything from Part 8's `WHERE` toolkit applies directly to `UPDATE`:
```sql
UPDATE employees SET salary = salary + 5000
WHERE department = 'IT' AND salary < 70000;

UPDATE employees SET salary = salary + 5000
WHERE department IN ('IT', 'HR');

UPDATE employees SET salary = salary + 2000
WHERE salary BETWEEN 40000 AND 60000;
```

---

## 7. `DELETE`

**Definition** — Removes entire rows matching a condition.

**Syntax**
```sql
DELETE FROM table_name WHERE condition;
```

**Example**
```sql
DELETE FROM employees WHERE emp_id = 2;
```
**Output**
```
DELETE 1
```

⚠️ **Notes & Caveats** — `DELETE` always removes **whole rows** — there's no way to delete "just one column's value" with `DELETE`. To clear a single value, use `UPDATE ... SET column = NULL` instead.

---

## 8. Deleting Multiple Rows

```sql
DELETE FROM employees WHERE department = 'HR';
```
Removes **every** row matching the condition — not just one.

---

## 9. `DELETE` Without `WHERE` — Extremely Dangerous

```sql
-- ⚠️ Deletes every row; table structure remains, but 0 rows left
DELETE FROM employees;
```

💡 **Best Practices** — Same rule as `UPDATE`: run a matching `SELECT` first to confirm exactly which rows will be hit.

---

## 10. `DELETE` vs `DROP TABLE` vs `TRUNCATE` (Comparison)

| | `DELETE FROM t;` | `TRUNCATE TABLE t;` | `DROP TABLE t;` |
|---|---|---|---|
| Table itself | Kept | Kept | Deleted |
| Structure/constraints | Kept | Kept | Deleted |
| Rows | Deleted (optionally filtered) | Deleted (all) | Deleted |
| `WHERE` support | ✅ Yes | ❌ No | N/A |
| Speed on huge tables | Slower (row-by-row, logged) | ✅ Fast | N/A |
| Fires triggers | ✅ Yes | ❌ No (usually) | N/A |

💡 **How to Choose** — Need to remove *some* rows → `DELETE`. Need to empty an *entire* table fast → `TRUNCATE`. Need the table gone completely → `DROP TABLE`.

---

## 11. `RETURNING`

**Definition** — Returns the rows affected by `INSERT`, `UPDATE`, `DELETE`, or `UPSERT` — without a separate follow-up `SELECT`.

**Why It Exists** — Applications frequently need to know exactly what changed (e.g., a newly generated ID) right after the write — `RETURNING` avoids a second round-trip to the database.

**Syntax**
```sql
UPDATE table_name SET col = val WHERE condition RETURNING *;
UPDATE table_name SET col = val WHERE condition RETURNING col1, col2;
```

---

## 12. `RETURNING` with `INSERT` / `UPDATE` / `DELETE`

**With `UPDATE`**
```sql
UPDATE employees
SET salary = salary + 5000
WHERE emp_id = 1
RETURNING name, salary;
```
**Output**
```
name | salary
Aman | 55000
```

**With `INSERT`** — extremely useful for grabbing an auto-generated ID:
```sql
INSERT INTO students (name)
VALUES ('Aman')
RETURNING student_id;
```
**Output**
```
student_id
1
```

**With `DELETE`** — see exactly what was removed:
```sql
DELETE FROM employees
WHERE emp_id = 2
RETURNING *;
```
**Output**
```
emp_id | name | department | salary | city
2      | Ravi | HR         | 40000  | Chandigarh
```

⚠️ **Notes & Caveats** — `RETURNING *` returns only the rows **affected by that statement**, not the whole table.

---

## 13. `UPSERT` (`INSERT ... ON CONFLICT`)

**Definition** — Insert a row if it doesn't exist yet; otherwise, update the existing row. ("UPSERT" = UPDATE + INSERT.)

**Why It Exists** — Avoids the awkward "check if it exists, then decide whether to `INSERT` or `UPDATE`" dance in application code — PostgreSQL handles it atomically in one statement.

**Syntax**
```sql
INSERT INTO table_name (col1, col2, ...)
VALUES (val1, val2, ...)
ON CONFLICT (unique_or_pk_column)
DO UPDATE SET col1 = value, col2 = value;
```

**Example**
```sql
INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 65000, 15)
ON CONFLICT (product_id)
DO UPDATE
SET price = 65000,
    stock = 15;
```

**Mental flow**
```
Try INSERT
     ↓
Does product_id already exist?
     ├── NO  → normal INSERT
     └── YES → CONFLICT → run DO UPDATE instead
```

⚠️ **Notes & Caveats** — `ON CONFLICT` needs a genuine uniqueness rule (`PRIMARY KEY` or `UNIQUE` constraint) on the referenced column(s) — without one, PostgreSQL has nothing to detect a "conflict" against.

---

## 14. The `EXCLUDED` Keyword

**Definition** — Inside `DO UPDATE`, `EXCLUDED` refers to the **new row that was attempted** (and blocked by the conflict) — as opposed to `table_name.column`, which refers to the **existing row already in the table**.

**Example — avoid repeating literal values**
```sql
INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 65000, 15)
ON CONFLICT (product_id)
DO UPDATE
SET price = EXCLUDED.price,
    stock = EXCLUDED.stock;
```

**Example — combine old and new values (add incoming stock)**
```sql
INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 60000, 5)
ON CONFLICT (product_id)
DO UPDATE
SET stock = products.stock + EXCLUDED.stock;
-- existing stock (10) + newly delivered stock (5) = 15
```

| Reference | Meaning |
|---|---|
| `products.stock` | The value **already in the database** (old) |
| `EXCLUDED.stock` | The value from the **attempted new insert** |

---

## 15. `ON CONFLICT DO NOTHING` vs `DO UPDATE`

**Syntax**
```sql
INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 65000, 15)
ON CONFLICT (product_id) DO NOTHING;
```

| | `DO NOTHING` | `DO UPDATE` |
|---|---|---|
| On conflict | Silently ignore the new row | Update the existing row |
| Existing row | Unchanged | Modified per `SET` clause |
| Error thrown? | No | No |

**With `RETURNING`**
```sql
INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 65000, 5)
ON CONFLICT (product_id)
DO UPDATE SET price = EXCLUDED.price, stock = products.stock + EXCLUDED.stock
RETURNING *;
```

---

## 16. `INSERT` vs `UPDATE` vs `UPSERT` (Comparison)

| Command | Row Already Exists | Row Doesn't Exist |
|---|---|---|
| `INSERT` | ❌ Errors (PK/unique violation) | ✅ Inserts |
| `UPDATE` | ✅ Updates | ⚠️ Silently affects 0 rows |
| `UPSERT` | ✅ Updates | ✅ Inserts |

---

## 17. Interview Q&A

**Q: What happens if you run `UPDATE` without a `WHERE` clause?**
A: Every row in the table gets updated — this is one of the most common and dangerous beginner mistakes.

**Q: Difference between `DELETE` and `TRUNCATE`?**
A: `DELETE` removes rows matching an optional `WHERE` condition, can be part of a transaction, and fires triggers; `TRUNCATE` empties the entire table at once, can't filter with `WHERE`, and is typically much faster on large tables.

**Q: What does `RETURNING` do?**
A: Returns the rows actually affected by an `INSERT`, `UPDATE`, `DELETE`, or `UPSERT` statement, avoiding a separate follow-up `SELECT`.

**Q: What is UPSERT, and how is it written in PostgreSQL?**
A: UPSERT means insert a row if it doesn't exist, otherwise update the existing one. In PostgreSQL: `INSERT ... ON CONFLICT (unique_col) DO UPDATE SET ...` (or `DO NOTHING` to just ignore duplicates).

**Q: What does `EXCLUDED` represent inside `ON CONFLICT DO UPDATE`?**
A: The row that was attempted to be inserted but was blocked by the conflict — i.e., the "new" incoming values, as opposed to `table.column`, which refers to the row already stored.

**Q: Why does `ON CONFLICT` require a `UNIQUE`/`PRIMARY KEY` constraint on the target column?**
A: Because "conflict" is defined as a violation of a uniqueness rule — without one, PostgreSQL has no basis for detecting a duplicate in the first place.

**Q: `DO NOTHING` vs `DO UPDATE`?**
A: `DO NOTHING` silently ignores the conflicting insert, leaving the existing row untouched. `DO UPDATE` modifies the existing row using the values from the attempted insert (via `EXCLUDED`).

---

## 18. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Update one column | `UPDATE t SET c = v WHERE ...;` |
| Update multiple columns | `UPDATE t SET c1=v1, c2=v2 WHERE ...;` |
| Update using old value | `SET salary = salary + 5000` |
| Delete rows | `DELETE FROM t WHERE ...;` |
| Get affected rows back | add `RETURNING *;` (or specific columns) |
| Insert-or-update | `INSERT ... ON CONFLICT (col) DO UPDATE SET ...;` |
| Insert-or-ignore | `INSERT ... ON CONFLICT (col) DO NOTHING;` |
| New attempted value | `EXCLUDED.column` |
| Existing stored value | `table_name.column` |

---

## 19. Cheat Sheet

```sql
-- ── UPDATE ────────────────────────────────
UPDATE employees SET salary = 55000 WHERE emp_id = 1;
UPDATE employees SET salary = 60000, city = 'Delhi' WHERE emp_id = 1;
UPDATE employees SET salary = salary + 5000 WHERE department = 'IT';
UPDATE employees SET salary = salary * 1.10;                 -- +10% raise, ALL rows

-- ── DELETE ────────────────────────────────
DELETE FROM employees WHERE emp_id = 2;
DELETE FROM employees WHERE department = 'HR';
DELETE FROM employees;                                        -- ⚠️ deletes ALL rows

-- ── RETURNING ─────────────────────────────
UPDATE employees SET salary = salary + 5000 WHERE emp_id = 1 RETURNING *;
INSERT INTO students (name) VALUES ('Aman') RETURNING student_id;
DELETE FROM employees WHERE emp_id = 2 RETURNING *;

-- ── UPSERT ────────────────────────────────
INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 65000, 15)
ON CONFLICT (product_id)
DO UPDATE SET price = EXCLUDED.price, stock = EXCLUDED.stock;

INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 60000, 5)
ON CONFLICT (product_id)
DO UPDATE SET stock = products.stock + EXCLUDED.stock;         -- add to existing

INSERT INTO products (product_id, product_name, price, stock)
VALUES (1, 'Laptop', 65000, 15)
ON CONFLICT (product_id) DO NOTHING;                            -- ignore duplicates
```

---

## 20. Preview of Part 10

| Topic | What You'll Learn |
|---|---|
| `PRIMARY KEY`, `FOREIGN KEY`, `UNIQUE`, `NOT NULL`, `CHECK`, `DEFAULT` | Full constraint syntax |
| `ON DELETE CASCADE` / `SET NULL`, `ON UPDATE CASCADE` | Foreign key behaviour |
| **Full worked FK-placement examples** | `CREATE TABLE` + `INSERT` + constraint-violation demo for 1:1, 1:N, and M:N relationships |
