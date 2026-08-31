# SQL & PostgreSQL Complete Notes — Part 8: WHERE & Filtering

## 📑 Table of Contents
1. `WHERE`
2. Comparison Operators
3. `AND`
4. `OR`
5. `NOT`
6. Combining `AND` + `OR` — Precedence & Parentheses
7. `IN` / `NOT IN`
8. `BETWEEN` / `NOT BETWEEN`
9. `LIKE` / `ILIKE` — Pattern Matching
10. Wildcards: `%` and `_`
11. `NULL`, `IS NULL`, `IS NOT NULL`
12. `COALESCE()`
13. `NULLIF()`
14. `COALESCE()` vs `NULLIF()` (Comparison)
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Preview of Part 9

**📋 Series Coverage (Part 8):** `WHERE`, `=`/`!=`/`<>`/`>`/`<`/`>=`/`<=`, `AND`, `OR`, `NOT`, operator precedence, `IN`, `NOT IN`, `BETWEEN`, `NOT BETWEEN`, `LIKE`, `ILIKE`, `NOT LIKE`, `%` and `_` wildcards, `NULL`, `IS NULL`, `IS NOT NULL`, `COALESCE()`, `NULLIF()`

> Examples use this `employees` table:
> ```
> emp_id | name   | department | salary | city       | bonus
> 1      | Aman   | IT         | 50000  | Mohali     | 5000
> 2      | Ravi   | HR         | 40000  | Chandigarh | NULL
> 3      | Priya  | IT         | 70000  | Delhi      | 7000
> 4      | Karan  | Sales      | 60000  | Mohali     | NULL
> 5      | Simran | HR         | 45000  | Delhi      | 3000
> 6      | Arjun  | IT         | 80000  | Chandigarh | 8000
> 7      | Neha   | Sales      | 55000  | Mohali     | NULL
> ```

---

## 1. `WHERE`

**Definition** — Filters rows based on a condition; only rows where the condition evaluates `TRUE` appear in the result.

**Why It Exists** — Real tables have thousands/millions of rows; you almost never want every single one back.

**Syntax**
```sql
SELECT columns FROM table_name WHERE condition;
```

**Example**
```sql
SELECT * FROM employees WHERE department = 'IT';
```
**Output**
```
emp_id | name  | department | salary | city       | bonus
1      | Aman  | IT         | 50000  | Mohali     | 5000
3      | Priya | IT         | 70000  | Delhi      | 7000
6      | Arjun | IT         | 80000  | Chandigarh | 8000
```

⚠️ **Notes & Caveats** — `WHERE` never modifies the underlying table — it only controls which rows appear in **this query's result**.

---

## 2. Comparison Operators

| Operator | Meaning | Example |
|---|---|---|
| `=` | Equal to | `salary = 50000` |
| `!=` | Not equal to | `department != 'IT'` |
| `<>` | Not equal to (standard SQL) | `department <> 'IT'` |
| `>` | Greater than | `salary > 60000` |
| `<` | Less than | `salary < 50000` |
| `>=` | Greater than or equal | `salary >= 60000` |
| `<=` | Less than or equal | `salary <= 50000` |

⚠️ **Notes & Caveats** — `!=` and `<>` are functionally identical in PostgreSQL; `<>` is the standard-SQL spelling.

---

## 3. `AND`

**Definition** — Combines conditions; **all** must be `TRUE` for the row to be kept.

**Syntax**
```sql
WHERE condition1 AND condition2
```

**Example**
```sql
SELECT name, department, salary
FROM employees
WHERE department = 'IT' AND salary > 60000;
```
**Output**
```
name  | department | salary
Priya | IT         | 70000
Arjun | IT         | 80000
```

**Truth table**

| Cond 1 | Cond 2 | `AND` Result |
|---|---|---|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE |
| FALSE | TRUE | FALSE |
| FALSE | FALSE | FALSE |

---

## 4. `OR`

**Definition** — Combines conditions; **at least one** must be `TRUE`.

**Example**
```sql
SELECT name, department
FROM employees
WHERE department = 'IT' OR department = 'HR';
```

**Truth table**

| Cond 1 | Cond 2 | `OR` Result |
|---|---|---|
| TRUE | TRUE | TRUE |
| TRUE | FALSE | TRUE |
| FALSE | TRUE | TRUE |
| FALSE | FALSE | FALSE |

💡 **Memory trick:** `AND` = everything must pass · `OR` = at least one must pass.

---

## 5. `NOT`

**Definition** — Reverses a condition's result (`TRUE` ↔ `FALSE`).

**Example**
```sql
SELECT * FROM employees WHERE NOT department = 'IT';
```
Equivalent to `department != 'IT'` / `department <> 'IT'`.

---

## 6. Combining `AND` + `OR` — Precedence & Parentheses

⚠️ **Notes & Caveats** — `AND` is evaluated **before** `OR` unless parentheses say otherwise. This trips up almost everyone at least once.

```sql
-- ⚠️ Ambiguous-looking, but AND binds tighter than OR:
WHERE department = 'IT'
   OR department = 'HR'
  AND salary > 50000;
-- PostgreSQL reads this as:
--   department = 'IT'  OR  (department = 'HR' AND salary > 50000)
```

❌ **Common Mistakes**
```sql
-- ❌ Intends "(IT or HR) AND salary > 50000" but doesn't get it
WHERE department = 'IT' OR department = 'HR' AND salary > 50000;
```
```sql
-- ✅ Use parentheses to make intent explicit
WHERE (department = 'IT' OR department = 'HR')
  AND salary > 50000;
```

💡 **Best Practices** — Whenever `AND` and `OR` appear together, add parentheses — don't rely on remembering precedence rules.

---

## 7. `IN` / `NOT IN`

**Definition** — `IN` checks whether a value matches **any** value in a given list — shorthand for a chain of `OR`s.

**Syntax**
```sql
WHERE column IN (value1, value2, value3)
WHERE column NOT IN (value1, value2, value3)
```

**Example**
```sql
SELECT * FROM employees WHERE department IN ('IT', 'HR', 'Sales');
-- equivalent to:
-- WHERE department = 'IT' OR department = 'HR' OR department = 'Sales'
```

⚠️ **Notes & Caveats** — `NOT IN` behaves unexpectedly if the list contains a `NULL` — the whole condition can evaluate to `UNKNOWN` for every row (see Part 16, Subqueries, for the full explanation). Prefer `NOT EXISTS` over `NOT IN` when the list might contain `NULL`s.

---

## 8. `BETWEEN` / `NOT BETWEEN`

**Definition** — Checks whether a value lies within an **inclusive** range.

**Syntax**
```sql
WHERE column BETWEEN low AND high
WHERE column NOT BETWEEN low AND high
```

**Example**
```sql
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 70000;
```
`50000` and `70000` are **both included** — equivalent to `salary >= 50000 AND salary <= 70000`.

⚠️ **Notes & Caveats** — `BETWEEN` is inclusive on both ends — this is one of the most common interview trick points.

---

## 9. `LIKE` / `ILIKE` — Pattern Matching

**Definition** — `LIKE` matches text against a pattern (case-sensitive in PostgreSQL); `ILIKE` does the same but case-**in**sensitively (PostgreSQL-specific, not standard SQL).

**Syntax**
```sql
WHERE column LIKE 'pattern'
WHERE column ILIKE 'pattern'
WHERE column NOT LIKE 'pattern'
```

**Example**
```sql
SELECT * FROM employees WHERE name LIKE 'A%';    -- starts with 'A', case-sensitive
SELECT * FROM employees WHERE name ILIKE 'a%';   -- starts with 'a' or 'A'
```

---

## 10. Wildcards: `%` and `_`

| Wildcard | Meaning |
|---|---|
| `%` | Zero or more characters (any length) |
| `_` | Exactly one character |

**Examples**

| Pattern | Meaning | Matches |
|---|---|---|
| `'A%'` | Starts with A | Aman, Arjun, A |
| `'%a'` | Ends with a | Priya, Simrana |
| `'%ar%'` | Contains "ar" | Karan |
| `'A___'` | Starts with A, exactly 4 chars total | Aman, Arun |
| `'_a%'` | 2nd character is 'a' | Karan, Ravi (if 2nd char = a) |

💡 **Memory trick:** `%` = any **length** · `_` = one **position**.

---

## 11. `NULL`, `IS NULL`, `IS NOT NULL`

**Definition** — `NULL` means a **missing or unknown** value — not `0`, not `''`, not `FALSE`.

**Why `= NULL` fails**
```sql
-- ❌ Never returns rows the way you'd expect
WHERE bonus = NULL;
```
SQL uses **three-valued logic** (`TRUE`/`FALSE`/`UNKNOWN`). `NULL = NULL` evaluates to `UNKNOWN`, not `TRUE` — because comparing two unknown values can't logically be declared equal.

**Correct syntax**
```sql
WHERE bonus IS NULL;
WHERE bonus IS NOT NULL;
```

**Example**
```sql
SELECT name, bonus FROM employees WHERE bonus IS NULL;
```
**Output**
```
name  | bonus
Ravi  | NULL
Karan | NULL
Neha  | NULL
```

❌ **Common Mistakes**
```sql
-- ❌ Never matches, even for rows that ARE NULL
WHERE bonus = NULL;
```
```sql
-- ✅ Correct
WHERE bonus IS NULL;
```

---

## 12. `COALESCE()`

**Definition** — Returns the **first non-NULL value** from a list of arguments, checked left to right.

**Syntax**
```sql
COALESCE(value1, value2, ..., fallback)
```

**Example**
```sql
SELECT name, COALESCE(bonus, 0) AS bonus FROM employees;
```
**Output**
```
name  | bonus
Aman  | 5000
Ravi  | 0        ← bonus was NULL, replaced with 0
Priya | 7000
Karan | 0
```

**Real use — safe arithmetic with possible NULLs**
```sql
SELECT name, salary + COALESCE(bonus, 0) AS total_income
FROM employees;
```
⚠️ Without `COALESCE`, `salary + NULL` evaluates to `NULL` — any arithmetic touching a `NULL` collapses to `NULL`.

**Multi-argument fallback chain**
```sql
COALESCE(phone, email, 'No Contact')
-- checks phone first, then email, then falls back to the literal string
```

---

## 13. `NULLIF()`

**Definition** — Returns `NULL` if two values are equal; otherwise returns the first value.

**Syntax**
```sql
NULLIF(value1, value2)
```

**Example**
```sql
SELECT NULLIF(10, 10);   -- NULL  (equal → NULL)
SELECT NULLIF(10, 20);   -- 10    (not equal → first value)
```

**Real use — avoiding division by zero**
```sql
SELECT total_sales / NULLIF(quantity, 0) FROM products;
```
If `quantity = 0`, `NULLIF(quantity, 0)` becomes `NULL`, and `total_sales / NULL` safely returns `NULL` instead of throwing a division-by-zero error.

---

## 14. `COALESCE()` vs `NULLIF()` (Comparison)

| | `COALESCE()` | `NULLIF()` |
|---|---|---|
| Direction | `NULL → real value` | `matching value → NULL` |
| Purpose | Provide a fallback for missing data | Intentionally create NULL for a "sentinel" value |
| Example | `COALESCE(bonus, 0)` | `NULLIF(discount, 0)` |
| Common use | Safe math, display defaults | Avoiding division by zero, hiding placeholder values |

---

## 15. Interview Q&A

**Q: Why does `WHERE bonus = NULL` never return rows?**
A: SQL uses three-valued logic — comparing anything to `NULL` (including `NULL` to `NULL`) evaluates to `UNKNOWN`, not `TRUE`. You must use `IS NULL` / `IS NOT NULL` instead.

**Q: Is `BETWEEN 10 AND 20` inclusive or exclusive?**
A: Inclusive on both ends — equivalent to `>= 10 AND <= 20`.

**Q: `LIKE` vs `ILIKE`?**
A: `LIKE` is case-sensitive; `ILIKE` (PostgreSQL-specific) is case-insensitive.

**Q: What do `%` and `_` mean in a `LIKE` pattern?**
A: `%` matches zero or more characters of any value; `_` matches exactly one character.

**Q: Why should you use parentheses when mixing `AND` and `OR`?**
A: Because `AND` has higher precedence than `OR` by default, so `a OR b AND c` is evaluated as `a OR (b AND c)`, which is often not what was intended. Parentheses make the logic explicit.

**Q: What's the difference between `COALESCE()` and `NULLIF()`?**
A: `COALESCE()` returns the first non-NULL value from a list (turns `NULL` into something real); `NULLIF(a, b)` returns `NULL` when `a` and `b` are equal (turns a specific value into `NULL`) — they work in opposite directions.

**Q: Why might `NOT IN` behave unexpectedly with a subquery?**
A: If the list/subquery result contains even one `NULL`, the entire `NOT IN` condition can evaluate to `UNKNOWN` for every row, silently returning zero rows. `NOT EXISTS` is the safer alternative in that situation.

**Q: How do you safely divide by a column that might be zero?**
A: `numerator / NULLIF(denominator, 0)` — converts a `0` denominator into `NULL`, so the division returns `NULL` instead of erroring.

---

## 16. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Filter rows | `WHERE condition` |
| Equal / not equal | `=` / `!=` or `<>` |
| Range comparisons | `>`, `<`, `>=`, `<=` |
| All conditions true | `AND` |
| At least one true | `OR` |
| Reverse a condition | `NOT` |
| Value in a list | `IN (a, b, c)` |
| Value not in a list | `NOT IN (a, b, c)` |
| Inclusive range | `BETWEEN low AND high` |
| Outside a range | `NOT BETWEEN low AND high` |
| Pattern match (case-sensitive) | `LIKE 'pattern'` |
| Pattern match (case-insensitive) | `ILIKE 'pattern'` |
| Any characters | `%` |
| Exactly one character | `_` |
| Missing value check | `IS NULL` / `IS NOT NULL` |
| First non-NULL value | `COALESCE(a, b, ...)` |
| Value → NULL if equal | `NULLIF(a, b)` |

---

## 17. Cheat Sheet

```sql
-- ── COMPARISON ───────────────────────────
WHERE salary = 50000;
WHERE salary != 50000;      -- or <>
WHERE salary > 50000;
WHERE salary >= 50000;

-- ── LOGICAL ──────────────────────────────
WHERE department = 'IT' AND salary > 60000;
WHERE department = 'IT' OR department = 'HR';
WHERE NOT department = 'IT';
WHERE (department = 'IT' OR department = 'HR') AND salary > 50000;  -- use parens!

-- ── LIST & RANGE ─────────────────────────
WHERE department IN ('IT', 'HR', 'Sales');
WHERE department NOT IN ('IT', 'HR');
WHERE salary BETWEEN 50000 AND 70000;       -- inclusive
WHERE salary NOT BETWEEN 50000 AND 70000;

-- ── PATTERN MATCHING ─────────────────────
WHERE name LIKE 'A%';        -- starts with A, case-sensitive
WHERE name ILIKE 'a%';       -- starts with A/a, case-insensitive
WHERE name LIKE '%a';        -- ends with a
WHERE name LIKE '%ar%';      -- contains ar
WHERE name LIKE 'A___';      -- starts with A, exactly 4 chars
WHERE name NOT LIKE 'A%';

-- ── NULL HANDLING ────────────────────────
WHERE bonus IS NULL;
WHERE bonus IS NOT NULL;
SELECT COALESCE(bonus, 0) FROM employees;                 -- NULL → fallback
SELECT salary + COALESCE(bonus, 0) AS total FROM employees;
SELECT NULLIF(discount, 0) FROM products;                  -- value → NULL
SELECT total_sales / NULLIF(quantity, 0) FROM products;    -- safe division
```

---

## 18. Preview of Part 9

| Topic | What You'll Learn |
|---|---|
| `UPDATE` | Changing existing rows, single & multiple columns |
| `DELETE` | Removing rows |
| `RETURNING` | Getting back the rows an INSERT/UPDATE/DELETE affected |
| `UPSERT` | `INSERT ... ON CONFLICT DO UPDATE / DO NOTHING` |
