# SQL & PostgreSQL Complete Notes — Part 20: String Functions — Case, Cleanup, Extraction & Search

## 📑 Table of Contents
1. What Are String Functions?
2. `UPPER()`
3. `LOWER()`
4. `UPPER()` vs `LOWER()` & Case-Insensitive Comparison
5. `LENGTH()`
6. `TRIM()` / `LTRIM()` / `RTRIM()`
7. Combining Functions (Nesting)
8. `SUBSTRING()`
9. `POSITION()`
10. `REPLACE()`
11. Combining `SUBSTRING` + `POSITION` — Extracting a Username from an Email
12. Interview Q&A
13. Quick Revision Sheet
14. Cheat Sheet
15. Preview of Part 21

**📋 Series Coverage (Part 20):** `UPPER()`, `LOWER()`, case-insensitive comparison, `LENGTH()`, `TRIM()`, `LTRIM()`, `RTRIM()`, function nesting, `SUBSTRING()` (1-indexed), `POSITION()`, `REPLACE()`, the extract-username-from-email pattern

---

## 1. What Are String Functions?

**Definition** — Functions that clean, search, extract, or reformat text data.

**Why It Exists** — Real-world text is messy: inconsistent casing (`aman`, `AMAN`, `Aman`), stray whitespace, or data crammed into one column that needs splitting apart (like an email address).

```
Text  →  String Function  →  New Text
```

⚠️ **Notes & Caveats** — String functions **do not modify stored data** — they only affect the query's output, unless used inside an `UPDATE`:
```sql
UPDATE employees SET name = UPPER(name);   -- this one DOES change stored data
```

---

## 2. `UPPER()`

**Definition** — Converts every letter in a string to uppercase.

**Syntax**
```sql
UPPER(string)
```

**Example**
```sql
SELECT name, UPPER(name) AS upper_name FROM employees;
```
**Output**
```
name   | upper_name
aman   | AMAN
Priya  | PRIYA
ARjun  | ARJUN
```

💡 **Real-world use — case-insensitive matching**
```sql
SELECT * FROM employees WHERE UPPER(name) = 'AMAN';
-- matches 'Aman', 'aman', 'AMAN', 'aMaN' ... all the same once uppercased
```

---

## 3. `LOWER()`

**Definition** — Converts every letter to lowercase.

**Syntax**
```sql
LOWER(string)
```

**Example**
```sql
SELECT LOWER(name) AS lower_name FROM employees;
```
**Output**
```
lower_name
aman
priya
arjun
```

---

## 4. `UPPER()` vs `LOWER()` & Case-Insensitive Comparison

| Function | Output |
|---|---|
| `UPPER('abc')` | `ABC` |
| `LOWER('ABC')` | `abc` |

**Two ways to do case-insensitive comparison in PostgreSQL**
```sql
WHERE LOWER(name) = 'aman';    -- portable across most SQL databases
WHERE name ILIKE 'aman';        -- PostgreSQL-specific, from Part 8; often simpler for patterns
```

💡 **How to Choose** — `ILIKE` is more concise for pattern-based, case-insensitive text search; `LOWER()`/`UPPER()` comparisons are useful when you need exact-match logic that's portable to non-PostgreSQL databases too.

---

## 5. `LENGTH()`

**Definition** — Returns the number of characters in a string.

**Syntax**
```sql
LENGTH(string)
```

**Example**
```sql
SELECT name, LENGTH(name) AS name_length FROM employees;
```
**Output**
```
name       | name_length
Aman       | 4
Priya      | 5
Ravi Kumar | 10
```

⚠️ **Notes & Caveats** — `LENGTH()` counts **everything**, including spaces and punctuation. `'Ravi Kumar'` has 10 characters — the space between the words counts too.

💡 **Real-world use**
```sql
SELECT * FROM employees WHERE LENGTH(name) = 5;   -- names with exactly 5 characters
```

---

## 6. `TRIM()` / `LTRIM()` / `RTRIM()`

**Definition** — `TRIM()` removes whitespace from the **start and end** of a string. `LTRIM()` removes only leading (left) whitespace; `RTRIM()` removes only trailing (right) whitespace. **None of these touch whitespace in the middle.**

**Syntax**
```sql
TRIM(string)
LTRIM(string)
RTRIM(string)
```

**Example**
```sql
SELECT TRIM('   Aman   ') AS trimmed;   -- 'Aman'
SELECT LTRIM('   Aman   ') AS left_trimmed;   -- 'Aman   '
SELECT RTRIM('   Aman   ') AS right_trimmed;  -- '   Aman'
```

⚠️ **Notes & Caveats** — `TRIM('Ravi Kumar')` stays `'Ravi Kumar'` — the space **between** the two words is not "leading" or "trailing," so it's left untouched.

❌ **Common Mistakes**
```sql
-- ❌ 'Aman' and ' Aman ' look "the same" to a human but are DIFFERENT strings
WHERE name = 'Aman';   -- silently misses ' Aman ' (with surrounding spaces)
```
```sql
-- ✅ Trim before comparing
WHERE TRIM(name) = 'Aman';
```

💡 **Best Practices** — Better yet, clean data **on the way in** (e.g., `TRIM()` inside your `INSERT`/application logic) so you're not compensating for messy data on every single query afterward.

---

## 7. Combining Functions (Nesting)

**Definition** — String functions can be nested — the innermost function runs first, and its result feeds the next function out.

**Example**
```sql
SELECT UPPER(TRIM(name)) AS cleaned FROM employees;
```
**Execution order**
```
' aman '  →  TRIM()  →  'aman'  →  UPPER()  →  'AMAN'
```

**Another example**
```sql
SELECT LENGTH(TRIM(name)) AS clean_length FROM employees;
-- trims first, THEN counts characters — so surrounding spaces don't inflate the count
```

---

## 8. `SUBSTRING()`

**Definition** — Extracts a portion of a string, starting at a given position, for a given length.

**Syntax**
```sql
SUBSTRING(string FROM start FOR length)
```

**Parameters**

| Name | Purpose | Example |
|---|---|---|
| `start` | Character position to begin extracting from | `1` |
| `length` | How many characters to take | `4` |

**Example**
```sql
SELECT SUBSTRING('DATABASE' FROM 1 FOR 4);   -- 'DATA'
SELECT SUBSTRING('DATABASE' FROM 5 FOR 4);   -- 'BASE'
```
```
D  A  T  A  B  A  S  E
1  2  3  4  5  6  7  8
└──1..4──┘  └──5..8──┘
  'DATA'      'BASE'
```

⚠️ **Notes & Caveats — the #1 beginner mistake:** **PostgreSQL string positions start at 1, not 0** (same rule as PostgreSQL arrays from Part 5). `SUBSTRING('DATABASE' FROM 1 FOR 4)` starts at the very first character, not the second.

---

## 9. `POSITION()`

**Definition** — Finds the (1-based) position where a substring first occurs inside another string. Returns `0` if not found.

**Syntax**
```sql
POSITION(substring IN string)
```

**Example**
```sql
SELECT POSITION('B' IN 'DATABASE');   -- 5
SELECT POSITION('@' IN 'aman@gmail.com');   -- 5
```

⚠️ **Notes & Caveats** — Not-found returns `0`, **not** `-1` (a common expectation carried over from other programming languages).
```sql
SELECT POSITION('Z' IN 'DATABASE');   -- 0
```

---

## 10. `REPLACE()`

**Definition** — Replaces every occurrence of a substring with another string.

**Syntax**
```sql
REPLACE(string, old_substring, new_substring)
```

**Example**
```sql
SELECT REPLACE('Hello World', 'World', 'SQL');   -- 'Hello SQL'
SELECT REPLACE('987-654-3210', '-', '');           -- '9876543210'  (empty string = delete)
SELECT REPLACE(email, 'gmail.com', 'googlemail.com') FROM employees;
```

⚠️ **Notes & Caveats** — Like every function in this part, `REPLACE()` only changes the **query output**, not stored data:
```sql
-- To make it permanent:
UPDATE employees SET email = REPLACE(email, 'gmail.com', 'googlemail.com');
```

---

## 11. Combining `SUBSTRING` + `POSITION` — Extracting a Username from an Email

**The classic interview pattern**
```sql
SELECT SUBSTRING(email FROM 1 FOR POSITION('@' IN email) - 1) AS username
FROM employees;
```

**Trace through `'aman@gmail.com'`**
```
Step 1 — POSITION('@' IN email)     → 5     (the @ is the 5th character)
Step 2 — length = 5 - 1              → 4     (take everything BEFORE the @)
Step 3 — SUBSTRING(email FROM 1 FOR 4) → 'aman'
```

💡 The `- 1` is essential — without it, you'd include the `@` symbol itself in the extracted username.

---

## 12. Interview Q&A

**Q: Does `UPPER()`/`LOWER()` permanently change the data in the table?**
A: No — they only affect the query's output. To permanently change stored values, you'd need `UPDATE table SET column = UPPER(column);`.

**Q: Does `LENGTH()` count spaces?**
A: Yes — it counts every character, including spaces and punctuation, not just letters.

**Q: Does `TRIM()` remove spaces in the middle of a string, like `'Ravi Kumar'`?**
A: No — `TRIM()` only removes whitespace from the **beginning and end** of a string; internal spaces are untouched.

**Q: Why might `WHERE name = 'Aman'` fail to match a row that visually looks like `Aman`?**
A: If the stored value has leading/trailing whitespace (e.g., `' Aman '`), it's a genuinely different string from `'Aman'` — spaces are real characters. Wrapping the comparison in `TRIM()` fixes this.

**Q: Are PostgreSQL string positions 0-indexed or 1-indexed?**
A: 1-indexed — the first character of a string is at position 1, not 0 (consistent with PostgreSQL's array indexing).

**Q: What does `POSITION()` return when the substring isn't found?**
A: `0` — not `-1`, which is a common assumption carried over from other programming languages.

**Q: How would you extract everything before the `@` in an email column?**
A: `SUBSTRING(email FROM 1 FOR POSITION('@' IN email) - 1)` — find the `@`'s position first, then extract everything up to (but not including) it.

**Q: Does `REPLACE()` change the underlying table data?**
A: No — like other string functions, it only affects the query's result unless wrapped in an `UPDATE` statement.

---

## 13. Quick Revision Sheet

| Goal | Function |
|---|---|
| Uppercase | `UPPER(str)` |
| Lowercase | `LOWER(str)` |
| Character count | `LENGTH(str)` |
| Remove leading + trailing spaces | `TRIM(str)` |
| Remove only leading spaces | `LTRIM(str)` |
| Remove only trailing spaces | `RTRIM(str)` |
| Extract part of a string | `SUBSTRING(str FROM start FOR length)` |
| Find substring's position (0 = not found) | `POSITION(sub IN str)` |
| Replace text | `REPLACE(str, old, new)` |
| Delete a character/substring | `REPLACE(str, target, '')` |

---

## 14. Cheat Sheet

```sql
-- ── CASE ──────────────────────────────────
SELECT UPPER(name) FROM employees;
SELECT LOWER(name) FROM employees;
SELECT * FROM employees WHERE UPPER(name) = 'AMAN';    -- case-insensitive match
SELECT * FROM employees WHERE name ILIKE 'aman';         -- PostgreSQL alternative

-- ── LENGTH ────────────────────────────────
SELECT name, LENGTH(name) FROM employees;
SELECT * FROM employees WHERE LENGTH(name) = 5;

-- ── WHITESPACE ────────────────────────────
SELECT TRIM('  Aman  ');    -- 'Aman'
SELECT LTRIM('  Aman  ');   -- 'Aman  '
SELECT RTRIM('  Aman  ');   -- '  Aman'

-- ── NESTED FUNCTIONS ──────────────────────
SELECT UPPER(TRIM(name)) FROM employees;
SELECT LENGTH(TRIM(name)) FROM employees;

-- ── EXTRACT / SEARCH ──────────────────────
SELECT SUBSTRING('DATABASE' FROM 1 FOR 4);           -- 'DATA'
SELECT POSITION('B' IN 'DATABASE');                   -- 5
SELECT POSITION('@' IN email) FROM employees;

-- ── REPLACE ───────────────────────────────
SELECT REPLACE('Hello World', 'World', 'SQL');
SELECT REPLACE(phone, '-', '') FROM contacts;

-- ── EXTRACT USERNAME FROM EMAIL ───────────
SELECT SUBSTRING(email FROM 1 FOR POSITION('@' IN email) - 1) AS username
FROM employees;
```

---

## 15. Preview of Part 21

| Topic | What You'll Learn |
|---|---|
| `CONCAT()` / `\|\|` | Joining strings together |
| `SPLIT_PART()` | Splitting a string on a delimiter |
| `INITCAP()` | Title-casing text |
| Combined formatting patterns | Building clean full names and usernames end to end |
