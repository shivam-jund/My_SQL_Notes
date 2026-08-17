# SQL & PostgreSQL Complete Notes — Part 21: String Functions — Combining, Splitting & Formatting

## 📑 Table of Contents
1. `CONCAT()`
2. The `||` Concatenation Operator
3. `CONCAT()` vs `||` — NULL Handling
4. `SPLIT_PART()`
5. When the Requested Part Doesn't Exist
6. `INITCAP()`
7. Combining Functions — Real-World Formatting Patterns
8. Phase 14 Function Summary
9. Interview Q&A
10. Quick Revision Sheet
11. Cheat Sheet
12. Preview of Part 22

**📋 Series Coverage (Part 21):** `CONCAT()`, `||` operator, `CONCAT()` vs `||` NULL behavior, `SPLIT_PART()`, `INITCAP()`, nested formatting patterns (clean username, formatted full name), full Phase 14 (String Functions) recap

---

## 1. `CONCAT()`

**Definition** — Joins two or more strings into one.

**Syntax**
```sql
CONCAT(string1, string2, string3, ...)
```

**Example**
```sql
SELECT CONCAT('Hello', 'World');          -- 'HelloWorld'  (no automatic space!)
SELECT CONCAT('Hello', ' ', 'World');      -- 'Hello World'
```

**Real-world use — combine two columns**
```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;
```
**Output**
```
full_name
Aman Kumar
Priya Sharma
```

❌ **Common Mistakes** — Assuming `CONCAT()` automatically inserts a space between arguments — it doesn't; you must include `' '` explicitly.

---

## 2. The `||` Concatenation Operator

**Definition** — PostgreSQL's native string concatenation operator — an alternative to `CONCAT()`.

**Syntax**
```sql
string1 || string2 || string3
```

**Example**
```sql
SELECT first_name || ' ' || last_name AS full_name FROM employees;
```
*(Produces the exact same result as the `CONCAT()` example above.)*

---

## 3. `CONCAT()` vs `||` — NULL Handling

⚠️ **Notes & Caveats — this is the single most important difference:**

| | `CONCAT()` | `\|\|` |
|---|---|---|
| If one argument is `NULL` | Treats it as an **empty string** — the rest of the result is preserved | The **entire result becomes `NULL`** |

**Example**
```sql
SELECT CONCAT('Aman', NULL, 'Kumar');    -- 'AmanKumar'   (NULL silently skipped)
SELECT 'Aman' || NULL || 'Kumar';         -- NULL          (whole expression collapses!)
```

💡 **How to Choose** — Use `CONCAT()` when any of the values being joined might be `NULL` and you don't want the whole result to disappear. Use `||` when you're confident the values are always populated, or when you explicitly want `NULL` to propagate (e.g., to flag incomplete data).

---

## 4. `SPLIT_PART()`

**Definition** — Splits a string on a delimiter and returns one specific piece (by position).

**Syntax**
```sql
SPLIT_PART(string, delimiter, part_number)
```

**Parameters**

| Name | Purpose | Example |
|---|---|---|
| `string` | The text to split | `'aman@gmail.com'` |
| `delimiter` | The character(s) to split on | `'@'` |
| `part_number` | Which piece to return (1-indexed) | `1` |

**Example**
```sql
SELECT SPLIT_PART('aman@gmail.com', '@', 1);   -- 'aman'
SELECT SPLIT_PART('aman@gmail.com', '@', 2);   -- 'gmail.com'
SELECT SPLIT_PART('2026-07-17', '-', 2);         -- '07'
```
```
'aman@gmail.com'
       ↓ split on '@'
   ['aman', 'gmail.com']
        ↑        ↑
      part 1   part 2
```

💡 **How to Choose** — `SPLIT_PART()` is usually simpler than the `SUBSTRING` + `POSITION` combo from Part 20 when the delimiter is a single, known character — reach for it first before reaching for manual position math.

---

## 5. When the Requested Part Doesn't Exist

```sql
SELECT SPLIT_PART('aman@gmail.com', '@', 3);   -- '' (empty string)
```

⚠️ **Notes & Caveats** — Asking for a part number beyond what the string actually splits into returns an **empty string `''`**, not `NULL` and not an error. This is different from `POSITION()`'s "not found → `0`" behavior (Part 20) — worth remembering as a contrast.

---

## 6. `INITCAP()`

**Definition** — Capitalizes the first letter of **every** word and lowercases the rest.

**Syntax**
```sql
INITCAP(string)
```

**Example**
```sql
SELECT INITCAP('aMaN kuMaR');   -- 'Aman Kumar'
```
**Output**
```
name          | formatted_name
AMAN          | Aman
priya         | Priya
rAvI KuMaR    | Ravi Kumar
```

❌ **Common Mistakes** — Assuming `INITCAP()` only fixes the *first* word — it capitalizes the leading letter of **every** word in the string:
```sql
SELECT INITCAP('aman kumar singh');   -- 'Aman Kumar Singh'  (all three words fixed)
```

---

## 7. Combining Functions — Real-World Formatting Patterns

**Pattern 1 — clean, properly-cased username from a messy email**
```sql
SELECT INITCAP(LOWER(SPLIT_PART(email, '@', 1))) AS clean_username
FROM employees;
```
**Trace through `'AMAN@GMAIL.COM'`**
```
Step 1 — SPLIT_PART(email, '@', 1)   → 'AMAN'
Step 2 — LOWER('AMAN')                → 'aman'
Step 3 — INITCAP('aman')               → 'Aman'
```

**Pattern 2 — properly formatted full name from two messy columns**
```sql
SELECT INITCAP(CONCAT(first_name, ' ', last_name)) AS full_name
FROM employees;
```
**Trace through `first_name = 'AMAN'`, `last_name = 'kuMAR'`**
```
Step 1 — CONCAT('AMAN', ' ', 'kuMAR')  → 'AMAN kuMAR'
Step 2 — INITCAP('AMAN kuMAR')          → 'Aman Kumar'
```

💡 **Best Practices** — When nesting several functions, always trace them **inside-out** (innermost first) — this is the fastest way to debug an unexpected result.

---

## 8. Phase 14 Function Summary

| Function | Purpose | Example Input | Example Output |
|---|---|---|---|
| `UPPER()` | Uppercase | `'aman'` | `AMAN` |
| `LOWER()` | Lowercase | `'AMAN'` | `aman` |
| `LENGTH()` | Character count | `'Aman'` | `4` |
| `TRIM()` | Remove leading/trailing spaces | `' Aman '` | `Aman` |
| `LTRIM()` / `RTRIM()` | Remove left / right spaces only | `' Aman '` | `'Aman '` / `' Aman'` |
| `SUBSTRING()` | Extract part of a string | `'DATABASE' FROM 1 FOR 4` | `DATA` |
| `POSITION()` | Find a substring's position | `'B' IN 'DATABASE'` | `5` |
| `REPLACE()` | Replace text | `'Hello World' → 'SQL'` | `Hello SQL` |
| `CONCAT()` | Join strings (NULL-safe) | `'Aman', ' ', 'Kumar'` | `Aman Kumar` |
| `\|\|` | Join strings (NULL propagates) | `'Aman' \|\| ' ' \|\| 'Kumar'` | `Aman Kumar` |
| `SPLIT_PART()` | Split & return one piece | `'aman@gmail.com', '@', 2` | `gmail.com` |
| `INITCAP()` | Capitalize every word | `'aMaN kuMaR'` | `Aman Kumar` |

---

## 9. Interview Q&A

**Q: Does `CONCAT('Hello', 'World')` produce `'Hello World'`?**
A: No — it produces `'HelloWorld'` with no space; you must explicitly include a `' '` argument if you want one.

**Q: What's the key behavioral difference between `CONCAT()` and `||` regarding `NULL`?**
A: `CONCAT()` treats a `NULL` argument as an empty string and continues joining the rest normally. `||` propagates `NULL` — if any operand is `NULL`, the entire concatenated result becomes `NULL`.

**Q: How would you get the domain part of an email address like `'aman@gmail.com'`?**
A: `SPLIT_PART(email, '@', 2)` — splits on `@` and returns the second piece.

**Q: What does `SPLIT_PART()` return if you request a part number that doesn't exist in the string?**
A: An empty string `''` — not `NULL`, and not an error.

**Q: Does `INITCAP()` only capitalize the first word of a string?**
A: No — it capitalizes the first letter of **every** word and lowercases the rest of each word.

**Q: Write a query to produce a clean, title-cased username from a messy, all-caps email column.**
A: `SELECT INITCAP(LOWER(SPLIT_PART(email, '@', 1))) FROM employees;` — split off the part before `@`, lowercase it, then title-case it.

**Q: When nesting multiple string functions, in what order do they execute?**
A: Inside-out — the innermost function runs first, and its result is passed to the next function wrapping it, and so on outward.

---

## 10. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Join strings (NULL-safe) | `CONCAT(a, b, c)` |
| Join strings (NULL propagates) | `a \|\| b \|\| c` |
| Split & take one piece | `SPLIT_PART(str, delimiter, n)` |
| Missing split part | Returns `''` |
| Title-case every word | `INITCAP(str)` |
| Build a full name | `CONCAT(first, ' ', last)` or `INITCAP(CONCAT(first, ' ', last))` |
| Extract email username | `SPLIT_PART(email, '@', 1)` |
| Extract email domain | `SPLIT_PART(email, '@', 2)` |

---

## 11. Cheat Sheet

```sql
-- ── CONCAT / || ───────────────────────────
SELECT CONCAT('Hello', ' ', 'World');                 -- 'Hello World'
SELECT 'Hello' || ' ' || 'World';                       -- 'Hello World'
SELECT CONCAT(first_name, ' ', last_name) FROM employees;
SELECT CONCAT('Aman', NULL, 'Kumar');                  -- 'AmanKumar' (NULL-safe)
SELECT 'Aman' || NULL || 'Kumar';                        -- NULL (propagates!)

-- ── SPLIT_PART ────────────────────────────
SELECT SPLIT_PART('aman@gmail.com', '@', 1);   -- 'aman'
SELECT SPLIT_PART('aman@gmail.com', '@', 2);   -- 'gmail.com'
SELECT SPLIT_PART('aman@gmail.com', '@', 3);   -- '' (doesn't exist → empty string)

-- ── INITCAP ────────────────────────────────
SELECT INITCAP('aMaN kuMaR');   -- 'Aman Kumar'
SELECT INITCAP(name) FROM employees;

-- ── REAL-WORLD PATTERNS ───────────────────
-- Clean username from a messy email:
SELECT INITCAP(LOWER(SPLIT_PART(email, '@', 1))) AS clean_username FROM employees;

-- Formatted full name from two columns:
SELECT INITCAP(CONCAT(first_name, ' ', last_name)) AS full_name FROM employees;

-- Email domain:
SELECT SPLIT_PART(email, '@', 2) AS domain FROM employees;
```

---

## 12. Preview of Part 22

| Topic | What You'll Learn |
|---|---|
| `CURRENT_DATE`, `CURRENT_TIME`, `NOW()` | Getting the current date/time |
| `AGE()` | Calculating age/duration between dates |
| `EXTRACT()` / `DATE_PART()` | Pulling out year, month, day, etc. |
| `DATE_TRUNC()` | Rounding dates down to a unit (month, year...) |
| `INTERVAL` & date arithmetic | Adding/subtracting time periods |
