# SQL & PostgreSQL Complete Notes — Part 33: Functions & Procedures II — PL/pgSQL

## 📑 Table of Contents
1. Why Plain SQL Isn't Enough — Introducing PL/pgSQL
2. The Structure of a PL/pgSQL Function
3. `DECLARE`, Variables & `:=`
4. `RETURN expression;` — The Scalar Return
5. `RETURN;` — The Void / End-of-Function Return
6. Functions Returning Tables (`RETURNS TABLE`)
7. `RETURN QUERY`
8. `RETURN NEXT` — The Row-by-Row Builder
9. `RETURN QUERY EXECUTE` — Dynamic SQL
10. `IN`, `OUT`, `INOUT` Parameters
11. `IF` / `IF-ELSE`
12. `LOOP`, `FOR`, `WHILE`
13. Function vs Procedure — Complete Comparison
14. Interview Q&A
15. Quick Revision Sheet
16. Cheat Sheet
17. Preview of Part 34

**📋 Series Coverage (Part 33):** PL/pgSQL, `DECLARE`/`BEGIN`/`END`, variable assignment (`:=`), every `RETURN` variant (scalar, void, `RETURN NEXT`, `RETURN QUERY`, `RETURN QUERY EXECUTE`), `RETURNS TABLE`, `IN`/`OUT`/`INOUT` parameters, `IF`/`IF-ELSE`, `LOOP`/`FOR`/`WHILE`, complete function vs procedure comparison

---

## 1. Why Plain SQL Isn't Enough — Introducing PL/pgSQL

Part 32's functions only ever contained a single `SELECT` statement. But what if you need **variables**, **conditions**, **loops**, or **multiple statements**? Plain SQL alone can't express that — which is why PostgreSQL provides **PL/pgSQL**.

**Definition** — PL/pgSQL is PostgreSQL's procedural language, letting you write variables, conditionals, loops, and multi-step logic inside functions and procedures.

```
SQL          → retrieve/manipulate data
PL/pgSQL     → SQL + programming constructs
```

---

## 2. The Structure of a PL/pgSQL Function

**Syntax**
```sql
CREATE FUNCTION function_name(...)
RETURNS return_type
LANGUAGE plpgsql
AS
$$
DECLARE
    -- variables
BEGIN
    -- logic
END;
$$;
```

💡 **Analogy** — Think of a house: `DECLARE` is the foundation (set up your variables), `BEGIN`...`END` is everything built on top (the actual logic).

| Section | Purpose |
|---|---|
| `DECLARE` | Create variables (optional — only needed if you use any) |
| `BEGIN` | Marks the start of executable logic |
| `END;` | Marks the end — **mandatory** for every PL/pgSQL block |

---

## 3. `DECLARE`, Variables & `:=`

**Definition** — `DECLARE` creates named, typed variables for temporary storage inside the function.

**Example — full first PL/pgSQL function**
```sql
CREATE FUNCTION square(num INT)
RETURNS INT
LANGUAGE plpgsql
AS
$$
DECLARE
    result INT;
BEGIN
    result := num * num;
    RETURN result;
END;
$$;
```

| Line | Meaning |
|---|---|
| `result INT;` | Create an integer variable named `result` |
| `result := num * num;` | Assign a value — `:=` means "assign," like `=` in C++/Python |
| `RETURN result;` | Send `result`'s value back to the caller |

---

## 4. `RETURN expression;` — The Scalar Return

**Definition** — Evaluates an expression, hands that single value back, and **immediately exits** the function — nothing after it runs.

**Example**
```sql
CREATE OR REPLACE FUNCTION calculate_bonus(salary NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    bonus NUMERIC;
BEGIN
    bonus := salary * 0.10;
    RETURN bonus;   -- exits instantly with the calculated value
END;
$$;
```

💡 **When to use** — Whenever the function returns exactly one value (`INT`, `NUMERIC`, `BOOLEAN`, `TEXT`, etc.).

---

## 5. `RETURN;` — The Void / End-of-Function Return

**Definition** — A blank `RETURN;` immediately terminates the function **without** passing any value back.

**When to use it:**
1. In a function declared `RETURNS VOID` (performs an action, returns nothing).
2. At the very end of a table-returning function, to signal "the result set is complete."

**Example**
```sql
CREATE OR REPLACE FUNCTION log_system_event(event_name VARCHAR)
RETURNS VOID
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO system_logs (event, log_time) VALUES (event_name, NOW());
    RETURN;   -- exits, no value returned
END;
$$;
```

---

## 6. Functions Returning Tables (`RETURNS TABLE`)

**Definition** — Instead of returning one value, a function can return an entire result set — multiple rows and columns.

**Syntax**
```sql
CREATE FUNCTION function_name()
RETURNS TABLE (col1 TYPE, col2 TYPE)
LANGUAGE SQL
AS
$$
    SELECT ...
$$;
```

**Example**
```sql
CREATE FUNCTION get_it_employees()
RETURNS TABLE (name TEXT, salary INT)
LANGUAGE SQL
AS
$$
    SELECT name, salary FROM employees WHERE department = 'IT';
$$;
```
```sql
SELECT * FROM get_it_employees();
```
**Output**
```
name  | salary
Aman  | 70000
Ravi  | 80000
```

---

## 7. `RETURN QUERY`

**Definition** — Inside a PL/pgSQL (not plain SQL) function, `RETURN QUERY` executes a `SELECT` and dumps **all** its resulting rows into the function's output — and, unlike `RETURN`, does **not** immediately exit; execution continues.

**Example**
```sql
CREATE FUNCTION get_it()
RETURNS TABLE (name TEXT, salary INT)
LANGUAGE plpgsql
AS
$$
BEGIN
    RETURN QUERY
    SELECT name, salary FROM employees WHERE department = 'IT';
END;
$$;
```

💡 **`RETURN` vs `RETURN QUERY`**

| | Returns | Exits Immediately? |
|---|---|---|
| `RETURN value;` | One value | ✅ Yes |
| `RETURN QUERY select;` | Many rows | ❌ No — keeps running |

---

## 8. `RETURN NEXT` — The Row-by-Row Builder

**Definition** — Used inside a loop, for a function declared `RETURNS SETOF ...` or `RETURNS TABLE(...)`. Each call adds **one row** to an invisible output buffer and keeps the function running — it does **not** exit.

**Example — generate even numbers**
```sql
CREATE OR REPLACE FUNCTION generate_even_numbers(max_val INT)
RETURNS SETOF INT
LANGUAGE plpgsql
AS $$
DECLARE
    current_num INT := 0;
BEGIN
    WHILE current_num <= max_val LOOP
        RETURN NEXT current_num;      -- add to output buffer, keep looping
        current_num := current_num + 2;
    END LOOP;
    RETURN;   -- signal done, dump the buffer
END;
$$;
```

💡 **When to use** — When rows need to be computed one at a time inside a loop, rather than produced all at once by a single `SELECT`.

---

## 9. `RETURN QUERY EXECUTE` — Dynamic SQL

**Definition** — Like `RETURN QUERY`, but the query is built as a **dynamic text string** — necessary because plain SQL doesn't allow variables to stand in for table or column names.

**Example**
```sql
CREATE OR REPLACE FUNCTION get_table_data(table_name VARCHAR)
RETURNS SETOF RECORD
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY EXECUTE 'SELECT * FROM ' || quote_ident(table_name);
    RETURN;
END;
$$;
```

⚠️ **Notes & Caveats** — `quote_ident()` safely escapes the identifier — always use it (or similar) when building dynamic SQL from a variable, to avoid SQL injection—style bugs even inside your own database logic.

💡 **When to use** — When the table name, column name, or overall query shape must vary based on input at call time.

---

## 10. `IN`, `OUT`, `INOUT` Parameters

| Parameter Type | Meaning |
|---|---|
| `IN` (default) | Data flows **into** the function only |
| `OUT` | The parameter **is** the output — an alternative to `RETURNS` |
| `INOUT` | Acts as **both** input and output — the same variable, modified |

**Example — `OUT` parameter**
```sql
CREATE FUNCTION square(IN num INT, OUT result INT)
LANGUAGE plpgsql
AS $$
BEGIN
    result := num * num;
END;
$$;
```

**Example — `INOUT` parameter**
```sql
CREATE FUNCTION double_number(INOUT num INT)
LANGUAGE plpgsql
AS $$
BEGIN
    num := num * 2;
END;
$$;
```
```sql
-- input 10 → the SAME variable becomes 20 → returned as 20
```

---

## 11. `IF` / `IF-ELSE`

**Syntax**
```sql
IF condition THEN
    ...
ELSIF another_condition THEN
    ...
ELSE
    ...
END IF;
```

**Example**
```sql
IF salary > 50000 THEN
    RETURN 'High';
ELSE
    RETURN 'Low';
END IF;
```
*(Functionally identical to `if`/`else` in C++/Python/Java.)*

---

## 12. `LOOP`, `FOR`, `WHILE`

**`LOOP`** — repeats indefinitely until an explicit `EXIT`:
```sql
LOOP
    ...
    EXIT WHEN condition;
END LOOP;
```

**`FOR`** — a known number of iterations:
```sql
FOR i IN 1..5 LOOP
    ...
END LOOP;
```
*(Equivalent to `for (int i = 1; i <= 5; i++)`.)*

**`WHILE`** — repeats while a condition stays true:
```sql
WHILE x < 10 LOOP
    ...
END LOOP;
```

| Construct | Best For |
|---|---|
| `LOOP` | Repeats until an explicit `EXIT` |
| `FOR` | A known number of iterations |
| `WHILE` | Repeats until a condition becomes false |

---

## 13. Function vs Procedure — Complete Comparison

| Feature | Function | Procedure |
|---|---|---|
| Returns value | ✅ Yes (scalar or table) | Not required |
| Can return a table | ✅ Yes (`RETURNS TABLE`) | ❌ No (not via `RETURNS TABLE`) |
| Called using | `SELECT function()` | `CALL procedure()` |
| Usable inside another query | ✅ Yes | ❌ No |
| Main purpose | Compute and return data | Perform operations (insert/update/delete, workflows) |

❌ **Common Mistakes**
- Using `RETURN` (singular value) when you actually meant `RETURN QUERY` (many rows).
- Writing `BEGIN` without a matching `END;` — every PL/pgSQL block needs one.
- Assuming `DECLARE` is mandatory — it's only needed when the function actually uses variables.

---

## 14. Interview Q&A

**Q: What's the difference between `RETURN` and `RETURN QUERY`?**
A: `RETURN` sends back a single value and exits the function immediately. `RETURN QUERY` executes a `SELECT` and adds all its resulting rows to the output, without exiting — execution can continue afterward.

**Q: Why does PostgreSQL provide PL/pgSQL if plain SQL functions already exist?**
A: Plain SQL functions can only contain SQL statements — no variables, conditionals, loops, or multi-step logic. PL/pgSQL adds those procedural programming constructs on top of SQL.

**Q: What's the difference between `IN`, `OUT`, and `INOUT` parameters?**
A: `IN` (the default) passes data only into the function. `OUT` designates a parameter as the return value itself, as an alternative to `RETURNS`. `INOUT` does both — the same variable carries a value in and, potentially modified, back out.

**Q: When would you use `RETURN NEXT` instead of `RETURN QUERY`?**
A: When rows need to be built one at a time inside a loop (e.g., generated or computed incrementally), rather than produced all at once by a single `SELECT` statement.

**Q: Why is `quote_ident()` important in `RETURN QUERY EXECUTE`?**
A: Because dynamic SQL built from a variable (like a table name) needs to be safely escaped — `quote_ident()` prevents malformed or unsafe identifiers from breaking or compromising the dynamically constructed query.

**Q: Can a procedure use `RETURNS TABLE`?**
A: No — that capability is specific to functions; procedures perform operations and are called with `CALL`, not embedded inside expressions.

---

## 15. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Create a variable | `DECLARE var_name TYPE;` |
| Assign a value | `var_name := expression;` |
| Return one value | `RETURN expression;` |
| Return nothing | `RETURN;` |
| Return a query's rows | `RETURN QUERY SELECT ...;` |
| Build rows in a loop | `RETURN NEXT value;` |
| Return dynamic SQL rows | `RETURN QUERY EXECUTE 'SELECT ...';` |
| Function returns a table | `RETURNS TABLE (col TYPE, ...)` |
| Output-only parameter | `OUT param TYPE` |
| Input + output parameter | `INOUT param TYPE` |
| Conditional | `IF ... THEN ... ELSE ... END IF;` |
| Fixed iteration count | `FOR i IN 1..n LOOP ... END LOOP;` |
| Condition-based repeat | `WHILE condition LOOP ... END LOOP;` |

---

## 16. Cheat Sheet

```sql
-- ── PL/pgSQL STRUCTURE ────────────────────
CREATE FUNCTION square(num INT)
RETURNS INT
LANGUAGE plpgsql
AS $$
DECLARE
    result INT;
BEGIN
    result := num * num;
    RETURN result;
END;
$$;

-- ── VOID RETURN ───────────────────────────
CREATE FUNCTION log_event(event_name VARCHAR)
RETURNS VOID
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO system_logs (event, log_time) VALUES (event_name, NOW());
    RETURN;
END;
$$;

-- ── RETURNS TABLE + RETURN QUERY ──────────
CREATE FUNCTION get_it_employees()
RETURNS TABLE (name TEXT, salary INT)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY SELECT name, salary FROM employees WHERE department = 'IT';
END;
$$;

-- ── RETURN NEXT (row-by-row) ──────────────
CREATE FUNCTION generate_even_numbers(max_val INT)
RETURNS SETOF INT
LANGUAGE plpgsql
AS $$
DECLARE current_num INT := 0;
BEGIN
    WHILE current_num <= max_val LOOP
        RETURN NEXT current_num;
        current_num := current_num + 2;
    END LOOP;
    RETURN;
END;
$$;

-- ── DYNAMIC SQL ───────────────────────────
CREATE FUNCTION get_table_data(table_name VARCHAR)
RETURNS SETOF RECORD
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY EXECUTE 'SELECT * FROM ' || quote_ident(table_name);
END;
$$;

-- ── IN / OUT / INOUT ──────────────────────
CREATE FUNCTION square_out(IN num INT, OUT result INT)
LANGUAGE plpgsql
AS $$ BEGIN result := num * num; END; $$;

CREATE FUNCTION double_number(INOUT num INT)
LANGUAGE plpgsql
AS $$ BEGIN num := num * 2; END; $$;

-- ── CONTROL FLOW ──────────────────────────
IF salary > 50000 THEN RETURN 'High'; ELSE RETURN 'Low'; END IF;

FOR i IN 1..5 LOOP ... END LOOP;

WHILE x < 10 LOOP ... END LOOP;
```

---

## 17. Preview of Part 34

| Topic | What You'll Learn |
|---|---|
| `CREATE TRIGGER` | Automatically running a function on `INSERT`/`UPDATE`/`DELETE` |
| `BEFORE` vs `AFTER` | When trigger logic runs relative to the change |
| `NEW` and `OLD` | Accessing the changed row's before/after values |
| Audit logs | The most common real-world trigger use case |
