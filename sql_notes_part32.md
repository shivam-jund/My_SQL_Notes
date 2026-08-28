# SQL & PostgreSQL Complete Notes — Part 32: Functions & Procedures I — Fundamentals

## 📑 Table of Contents
1. Why Functions & Procedures Exist
2. What Is a Function?
3. `CREATE FUNCTION` — Basic Syntax
4. Calling a Function
5. Functions with Parameters
6. Functions That Query the Database
7. What Is a Stored Procedure?
8. `CREATE PROCEDURE`
9. Calling a Procedure — `CALL`
10. Function vs Procedure — Core Comparison
11. Interview Q&A
12. Quick Revision Sheet
13. Cheat Sheet
14. Preview of Part 33

**📋 Series Coverage (Part 32):** why reusable SQL logic matters, `CREATE FUNCTION`, calling functions with `SELECT`, function parameters, functions that query tables, `CREATE PROCEDURE`, `CALL`, function vs procedure fundamentals

---

## 1. Why Functions & Procedures Exist

**The problem** — Suppose you run this exact query every single day:
```sql
SELECT department, AVG(salary) FROM employees GROUP BY department;
```
Instead of retyping it every time, what if you could save it, give it a name, and call it whenever needed?

💡 **Analogy** — Making tea: instead of re-explaining "boil water, add tea, add milk, add sugar" every morning, you just say "make tea" — the steps are already saved. That's exactly what a function or procedure is.

---

## 2. What Is a Function?

**Definition** — A reusable block of SQL (or PL/pgSQL) code that performs a task and **always returns a value**.

```
Input → Function → Process → Return Output
```

💡 **Analogy** — A calculator: give it `2 + 3`, it returns `5`.

---

## 3. `CREATE FUNCTION` — Basic Syntax

**Syntax**
```sql
CREATE FUNCTION function_name(parameter_name TYPE, ...)
RETURNS return_type
LANGUAGE SQL
AS
$$
    SELECT ...;
$$;
```

**Example — square a number**
```sql
CREATE FUNCTION square_number(num INT)
RETURNS INT
LANGUAGE SQL
AS
$$
    SELECT num * num;
$$;
```

**Reading it piece by piece**

| Piece | Meaning |
|---|---|
| `CREATE FUNCTION square_number` | Create a function named `square_number` |
| `(num INT)` | Takes one integer input, called `num` |
| `RETURNS INT` | Returns an integer |
| `SELECT num * num;` | The logic — square the input |

---

## 4. Calling a Function

**Syntax**
```sql
SELECT function_name(arguments);
```

**Example**
```sql
SELECT square_number(5);
```
**Output**
```
25
```

---

## 5. Functions with Parameters

**Example — two parameters**
```sql
CREATE FUNCTION add_numbers(a INT, b INT)
RETURNS INT
LANGUAGE SQL
AS
$$
    SELECT a + b;
$$;
```
```sql
SELECT add_numbers(10, 20);   -- 30
```

💡 **Real-world use** — instead of repeating `salary * 12` throughout every query, wrap it once: `annual_salary(salary)` — cleaner, reusable, easier to maintain if the calculation ever changes.

---

## 6. Functions That Query the Database

**Example — return a count**
```sql
CREATE FUNCTION employee_count()
RETURNS INT
LANGUAGE SQL
AS
$$
    SELECT COUNT(*) FROM employees;
$$;
```
```sql
SELECT employee_count();
```

**Example — a parameter used inside a query**
```sql
CREATE FUNCTION get_salary(empid INT)
RETURNS INT
LANGUAGE SQL
AS
$$
    SELECT salary FROM employees WHERE id = empid;
$$;
```
```sql
SELECT get_salary(2);   -- 45000
```

---

## 7. What Is a Stored Procedure?

**Definition** — A reusable block of SQL code that performs one or more operations. Unlike a function, it **does not have to return a value** — it's designed to *do* something, not necessarily *compute and hand back* something.

```
Input → Perform Task → Finished   (no required return value)
```

💡 **Analogy** — A washing machine: you put in dirty clothes, it washes and dries them — no value is "returned," it just performs the task.

| | Function | Procedure |
|---|---|---|
| Analogy | Calculator | Washing machine |
| Typical job | Compute → return a result | Perform an action (update data, run a workflow) |

---

## 8. `CREATE PROCEDURE`

**Syntax**
```sql
CREATE PROCEDURE procedure_name(parameters)
LANGUAGE SQL
AS
$$
    ...
$$;
```

**Example — give everyone a raise**
```sql
CREATE PROCEDURE increase_salary()
LANGUAGE SQL
AS
$$
    UPDATE employees SET salary = salary + 5000;
$$;
```

⚠️ **Notes & Caveats** — Notice there's **no `RETURNS`** clause — procedures aren't required to hand back a value.

---

## 9. Calling a Procedure — `CALL`

**Syntax**
```sql
CALL procedure_name(arguments);
```

**Example**
```sql
CALL increase_salary();
```

**Why `CALL` and not `SELECT`?**

| | Syntax | Why |
|---|---|---|
| Function | `SELECT add_numbers(2, 3);` | You want a **value** back |
| Procedure | `CALL increase_salary();` | You want an **action** performed |

❌ **Common Mistakes**
```sql
-- ❌ Wrong: calling a procedure with SELECT
SELECT increase_salary();
```
```sql
-- ✅ Correct
CALL increase_salary();
```
```sql
-- ❌ Wrong: calling a function with CALL
CALL add_numbers(2, 3);
```
```sql
-- ✅ Correct
SELECT add_numbers(2, 3);
```

⚠️ **Notes & Caveats** — Procedures **can** pass values back out using `OUT`/`INOUT` parameters (covered fully in Part 33), but they're still conceptually designed for performing operations — not for being embedded inside an expression the way a function's return value can be.

---

## 10. Function vs Procedure — Core Comparison

| Feature | Function | Procedure |
|---|---|---|
| Returns a value | ✅ Required | Not required |
| Called using | `SELECT function_name()` | `CALL procedure_name()` |
| Main purpose | Compute & return | Perform operations |
| Can be used inside another query (e.g., `SELECT col, my_func(col) FROM t`) | ✅ Yes | ❌ No |

❌ **Common Mistakes**
- Thinking functions must always modify tables — many functions are purely calculators/retrievers with no side effects.

---

## 11. Interview Q&A

**Q: What's the fundamental difference between a function and a procedure?**
A: A function always computes and returns a value, and can be embedded inside a `SELECT` expression. A procedure is designed to perform one or more operations and isn't required to return a value; it's invoked with `CALL`, not `SELECT`.

**Q: How do you call a function vs. a procedure?**
A: A function is called with `SELECT function_name(args)`. A procedure is called with `CALL procedure_name(args)`.

**Q: Why would you create a function instead of repeating the same calculation in many queries?**
A: Reusability and maintainability — if the calculation's logic ever needs to change, you update it in one place (the function) instead of hunting down every query that repeats it.

**Q: Can a procedure return a value?**
A: Not via a `RETURNS` clause like a function — but it can pass values back through `OUT`/`INOUT` parameters. Conceptually, though, procedures are built for performing actions, not for being used as expressions.

**Q: What happens if you call `SELECT my_procedure();`?**
A: It errors — procedures must be invoked with `CALL`, not `SELECT`.

---

## 12. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Create a function | `CREATE FUNCTION name(params) RETURNS type LANGUAGE SQL AS $$ ... $$;` |
| Call a function | `SELECT name(args);` |
| Create a procedure | `CREATE PROCEDURE name(params) LANGUAGE SQL AS $$ ... $$;` |
| Call a procedure | `CALL name(args);` |
| Function returns a value? | Always |
| Procedure returns a value? | Not required |

---

## 13. Cheat Sheet

```sql
-- ── FUNCTION: scalar ──────────────────────
CREATE FUNCTION square_number(num INT)
RETURNS INT
LANGUAGE SQL
AS $$ SELECT num * num; $$;

SELECT square_number(5);   -- 25

-- ── FUNCTION: two parameters ──────────────
CREATE FUNCTION add_numbers(a INT, b INT)
RETURNS INT
LANGUAGE SQL
AS $$ SELECT a + b; $$;

SELECT add_numbers(10, 20);   -- 30

-- ── FUNCTION: queries a table ─────────────
CREATE FUNCTION employee_count()
RETURNS INT
LANGUAGE SQL
AS $$ SELECT COUNT(*) FROM employees; $$;

CREATE FUNCTION get_salary(empid INT)
RETURNS INT
LANGUAGE SQL
AS $$ SELECT salary FROM employees WHERE id = empid; $$;

-- ── PROCEDURE ─────────────────────────────
CREATE PROCEDURE increase_salary()
LANGUAGE SQL
AS $$ UPDATE employees SET salary = salary + 5000; $$;

CALL increase_salary();
```

---

## 14. Preview of Part 33

| Topic | What You'll Learn |
|---|---|
| PL/pgSQL | `DECLARE`, `BEGIN`, `END`, variables |
| `RETURN` variants | Scalar, void, `RETURN NEXT`, `RETURN QUERY` |
| `RETURNS TABLE` | Functions that return multiple rows/columns |
| `IN`, `OUT`, `INOUT` | Parameter directions |
| `IF`, `LOOP`, `FOR`, `WHILE` | Control flow inside functions |
