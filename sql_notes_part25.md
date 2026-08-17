# SQL & PostgreSQL Complete Notes — Part 25: Conditional Expressions

## 📑 Table of Contents
1. `CASE` — Decision Logic in SQL
2. `CASE` with Multiple Conditions — Order Matters
3. `CASE` Inside Aggregate Functions
4. `COALESCE()` — Quick Recap + New Patterns
5. `NULLIF()` — Quick Recap + New Patterns
6. `GREATEST()`
7. `LEAST()`
8. Combining `NULLIF()` + `COALESCE()` — Safe, Defaulted Division
9. Combining `GREATEST()` / `LEAST()` with `NULLIF()`
10. Interview Q&A
11. Quick Revision Sheet
12. Cheat Sheet
13. Preview of Part 26

**📋 Series Coverage (Part 25):** `CASE WHEN...THEN...ELSE...END`, condition ordering, `CASE` inside `SUM()`/aggregate functions, `COALESCE()` and `NULLIF()` revisited with combined patterns, `GREATEST()`, `LEAST()`, safe percentage-with-default pattern

> `COALESCE()` and `NULLIF()` were first introduced in Part 8 (WHERE & Filtering) with full syntax and basic examples. This part briefly recaps them and focuses on **new** patterns — combining them with `CASE`, `GREATEST`, and `LEAST`.

---

## 1. `CASE` — Decision Logic in SQL

**Definition** — `CASE` lets SQL make decisions, similar to `if`/`else if`/`else` in general-purpose programming languages.

**Why It Exists** — Real reporting/business logic constantly needs "if this, show that" — e.g., labeling a salary as "High" or "Low."

**Syntax**
```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END
```

**Example**
```sql
SELECT name, salary,
       CASE WHEN salary > 50000 THEN 'High' ELSE 'Low' END AS salary_level
FROM employees;
```
**Output**
```
name   | salary | salary_level
Aman   | 70000  | High
Priya  | 45000  | Low
```

⚠️ **Notes & Caveats** — `ELSE` is optional; if omitted and no `WHEN` matches, the result is `NULL`.

---

## 2. `CASE` with Multiple Conditions — Order Matters

**Example**
```sql
SELECT name, marks,
       CASE
           WHEN marks >= 90 THEN 'A'
           WHEN marks >= 75 THEN 'B'
           WHEN marks >= 50 THEN 'C'
           ELSE 'Fail'
       END AS grade
FROM students;
```

⚠️ **Notes & Caveats — `CASE` stops at the first matching condition.** For `marks = 95`, SQL checks `>= 90` first (`TRUE`) and returns `'A'` immediately — it never evaluates the remaining `WHEN` clauses.

❌ **Common Mistakes — writing conditions in the wrong order:**
```sql
-- ❌ Wrong order: least-specific condition checked first
CASE
    WHEN marks >= 50 THEN 'C'
    WHEN marks >= 75 THEN 'B'
    WHEN marks >= 90 THEN 'A'
END
-- marks = 95 → incorrectly returns 'C', because 95 >= 50 is TRUE and SQL stops there
```
```sql
-- ✅ Always write the MOST specific condition first
CASE
    WHEN marks >= 90 THEN 'A'
    WHEN marks >= 75 THEN 'B'
    WHEN marks >= 50 THEN 'C'
    ELSE 'Fail'
END
```

💡 **Best Practices** — For range-style grading logic, always order `WHEN` clauses from most restrictive to least restrictive.

---

## 3. `CASE` Inside Aggregate Functions

**The question:** *"Count employees earning more than 50000."*
```sql
SELECT SUM(CASE WHEN salary > 50000 THEN 1 ELSE 0 END) AS high_earners
FROM employees;
```
**Trace through**
```
salary  | CASE result
70000   | 1
45000   | 0
30000   | 0
90000   | 1
        ↓ SUM
        2
```

**Treating `NULL` bonuses as 0 (an alternative to `COALESCE`, Section 4)**
```sql
SELECT SUM(CASE WHEN bonus IS NULL THEN 0 ELSE bonus END) AS total_bonus
FROM employees;
```

💡 This "conditional counting/summing" pattern (`SUM(CASE WHEN condition THEN 1 ELSE 0 END)`) is one of the most common building blocks in reporting SQL — worth memorizing.

---

## 4. `COALESCE()` — Quick Recap + New Patterns

**Recap (full detail in Part 8):** `COALESCE(value1, value2, ...)` returns the first **non-NULL** value in the list.
```sql
SELECT COALESCE(bonus, 0) FROM employees;   -- NULL bonus displayed as 0
```

**New pattern — safe total with a `NULL` component**
```sql
SELECT salary + COALESCE(bonus, 0) AS total_salary FROM employees;
```
⚠️ Without `COALESCE`, `salary + NULL` evaluates to `NULL` entirely — any arithmetic touching `NULL` collapses to `NULL` (this is the same three-valued-logic rule from Part 8).

---

## 5. `NULLIF()` — Quick Recap + New Patterns

**Recap (full detail in Part 8):** `NULLIF(value1, value2)` returns `NULL` if the two values are equal; otherwise returns `value1`.
```sql
SELECT NULLIF(10, 10);   -- NULL
SELECT NULLIF(10, 5);    -- 10
```

**Recap — the classic safe-division use**
```sql
SELECT marks / NULLIF(total_marks, 0) FROM exams;
-- if total_marks = 0, NULLIF turns it into NULL, so the division safely returns NULL instead of erroring
```

---

## 6. `GREATEST()`

**Definition** — Returns the **largest** value from a list of values.

**Syntax**
```sql
GREATEST(value1, value2, value3, ...)
```

**Example**
```sql
SELECT GREATEST(10, 25, 18);   -- 25
```

**Real-world use — highest score across three subject columns**
```sql
SELECT name, GREATEST(math, science, english) AS highest_marks FROM students;
```

**Works on dates too**
```sql
SELECT GREATEST(DATE '2026-07-10', DATE '2026-07-20', DATE '2026-07-15');   -- 2026-07-20
```

❌ **Common Mistakes**
```sql
-- ❌ Incompatible data types — PostgreSQL can't compare a number to text
SELECT GREATEST(10, 'Hello');
```

💡 `GREATEST()` works with any comparable type — numbers, dates, times, strings (lexicographically) — as long as all arguments share a compatible type.

---

## 7. `LEAST()`

**Definition** — The mirror image of `GREATEST()` — returns the **smallest** value from a list.

**Syntax**
```sql
LEAST(value1, value2, value3, ...)
```

**Example**
```sql
SELECT LEAST(10, 25, 18);   -- 10
```

**Real-world use — lowest score, earliest date**
```sql
SELECT name, LEAST(math, science, english) AS lowest_marks FROM students;
SELECT LEAST(DATE '2026-07-10', DATE '2026-07-20', DATE '2026-07-15');   -- 2026-07-10
```

---

## 8. Combining `NULLIF()` + `COALESCE()` — Safe, Defaulted Division

**The question:** *"Calculate a percentage, safely avoiding division by zero, and show `0` instead of `NULL` when that happens."*
```sql
SELECT COALESCE(marks / NULLIF(total_marks, 0), 0) AS percentage
FROM exams;
```

**Trace through `total_marks = 0`**
```
Step 1 — NULLIF(0, 0)              → NULL          (turns the zero into NULL)
Step 2 — marks / NULL                → NULL          (division involving NULL is NULL)
Step 3 — COALESCE(NULL, 0)            → 0             (final fallback)
```

💡 This `NULLIF` + `COALESCE` combination is extremely common in production reporting SQL — memorize the pattern, not just the individual functions.

---

## 9. Combining `GREATEST()` / `LEAST()` with `NULLIF()`

**The question:** *"Show the highest of three subject marks, but if the highest is 0, show `NULL` instead."*
```sql
SELECT NULLIF(GREATEST(math, science, english), 0) FROM students;
```
**Trace through**
```
Step 1 — GREATEST(math, science, english)   → e.g. 0 (all subjects scored zero)
Step 2 — NULLIF(0, 0)                          → NULL
```

---

## 10. Interview Q&A

**Q: Does `CASE` evaluate every `WHEN` clause, or stop at the first match?**
A: It stops at the **first** condition that evaluates `TRUE` and returns that result — later `WHEN` clauses are never checked for that row.

**Q: Why does `WHEN marks >= 50 THEN 'C'` written before `WHEN marks >= 90 THEN 'A'` produce wrong grades?**
A: Because `CASE` stops at the first match — a student with 95 marks satisfies `>= 50` immediately and gets `'C'`, never reaching the `>= 90` check. Conditions must be ordered from most to least specific.

**Q: How would you count rows matching a condition using `CASE` inside an aggregate?**
A: `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` — converts each matching row to `1` and non-matching rows to `0`, then sums them.

**Q: What's the difference in "direction" between `COALESCE()` and `NULLIF()`?**
A: `COALESCE()` turns a `NULL` into a real fallback value. `NULLIF()` turns a specific matching value into `NULL`. They work in opposite directions.

**Q: How do you calculate a safe percentage that shows `0` instead of erroring or returning `NULL` when the denominator is zero?**
A: `COALESCE(numerator / NULLIF(denominator, 0), 0)` — `NULLIF` prevents the division-by-zero error by converting a zero denominator to `NULL`, and the outer `COALESCE` converts the resulting `NULL` into `0`.

**Q: Can `GREATEST()`/`LEAST()` be used with dates?**
A: Yes — they work with any mutually comparable type, including numbers, dates, times, and text (compared lexicographically), as long as all arguments share a compatible type.

**Q: What happens if you call `GREATEST(10, 'Hello')`?**
A: An error — PostgreSQL cannot compare incompatible data types (a number and text) against each other.

---

## 11. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| If/else logic | `CASE WHEN cond THEN val ... ELSE val END` |
| Conditional count | `SUM(CASE WHEN cond THEN 1 ELSE 0 END)` |
| First non-NULL value | `COALESCE(a, b, ...)` |
| Value → NULL if equal | `NULLIF(a, b)` |
| Safe division | `a / NULLIF(b, 0)` |
| Safe division with default | `COALESCE(a / NULLIF(b, 0), 0)` |
| Largest of several values | `GREATEST(a, b, c, ...)` |
| Smallest of several values | `LEAST(a, b, c, ...)` |

---

## 12. Cheat Sheet

```sql
-- ── CASE ──────────────────────────────────
SELECT name, salary,
       CASE WHEN salary > 50000 THEN 'High' ELSE 'Low' END AS level
FROM employees;

SELECT name, marks,
       CASE
           WHEN marks >= 90 THEN 'A'      -- most specific FIRST
           WHEN marks >= 75 THEN 'B'
           WHEN marks >= 50 THEN 'C'
           ELSE 'Fail'
       END AS grade
FROM students;

SELECT SUM(CASE WHEN salary > 50000 THEN 1 ELSE 0 END) AS high_earners FROM employees;

-- ── COALESCE / NULLIF RECAP ───────────────
SELECT COALESCE(bonus, 0) FROM employees;
SELECT salary + COALESCE(bonus, 0) AS total FROM employees;
SELECT marks / NULLIF(total_marks, 0) FROM exams;

-- ── GREATEST / LEAST ──────────────────────
SELECT GREATEST(math, science, english) AS highest FROM students;
SELECT LEAST(math, science, english) AS lowest FROM students;
SELECT GREATEST(DATE '2026-07-10', DATE '2026-07-20', DATE '2026-07-15');

-- ── COMBINED PATTERNS ─────────────────────
SELECT COALESCE(marks / NULLIF(total_marks, 0), 0) AS percentage FROM exams;
SELECT NULLIF(GREATEST(math, science, english), 0) FROM students;
```

---

## 13. Preview of Part 26

| Topic | What You'll Learn |
|---|---|
| `CREATE VIEW` | Building a virtual table from a query |
| Updatable vs non-updatable views | Which views support `INSERT`/`UPDATE`/`DELETE` |
| `CREATE OR REPLACE VIEW`, `DROP VIEW` | Modifying and removing views |
| `WITH CHECK OPTION` | Preventing updates that break a view's own filter |
