# SQL & PostgreSQL Complete Notes — Part 34: Triggers

## 📑 Table of Contents
1. What Is a Trigger?
2. Trigger vs Function
3. The Trigger Function
4. `CREATE TRIGGER` Syntax
5. `BEFORE` vs `AFTER`
6. `FOR EACH ROW` vs `FOR EACH STATEMENT`
7. `NEW` and `OLD`
8. Which Events Have `NEW`/`OLD`?
9. Building an `INSERT` Audit Trigger
10. Building an `UPDATE` Audit Trigger
11. Building a `DELETE` Audit Trigger
12. `BEFORE` Triggers for Validation (`RAISE EXCEPTION`)
13. `BEFORE` Triggers for Modifying Data
14. `RETURN NEW` vs `RETURN OLD` vs `RETURN NULL`
15. Trigger Best Practices
16. Interview Q&A
17. Quick Revision Sheet
18. Cheat Sheet
19. Preview of Part 35

**📋 Series Coverage (Part 34):** what a trigger is, trigger vs function, trigger functions (`RETURNS TRIGGER`), `CREATE TRIGGER`, `BEFORE`/`AFTER`, `FOR EACH ROW`/`FOR EACH STATEMENT`, `NEW`/`OLD`, audit-log triggers for `INSERT`/`UPDATE`/`DELETE`, validation and data-modification via `BEFORE` triggers, `RAISE NOTICE`/`RAISE EXCEPTION`, `RETURN NEW`/`RETURN OLD`/`RETURN NULL`, trigger best practices

---

## 1. What Is a Trigger?

**Definition** — A database object that **automatically** executes a function when a specified event (`INSERT`, `UPDATE`, or `DELETE`) occurs on a table.

💡 **Analogy** — A door alarm: nobody manually switches it on — opening the door automatically triggers it.

```
INSERT/UPDATE/DELETE → Trigger fires → Trigger Function runs → Done
```

---

## 2. Trigger vs Function

| | Regular Function (Part 32) | Trigger |
|---|---|---|
| Invoked by | **You** call it (`SELECT my_func()`) | **PostgreSQL** calls it automatically |
| When it runs | Whenever you choose | When its registered event occurs |

⚠️ **Notes & Caveats** — A trigger itself contains no logic — it's just a rule: *"when this event happens, run that function."* The actual logic always lives in a separate **trigger function**.

---

## 3. The Trigger Function

**Definition** — A special function that a trigger executes. Unlike normal functions (which return `INT`, `TEXT`, `TABLE`, etc.), a trigger function always declares `RETURNS TRIGGER`.

**Example**
```sql
CREATE FUNCTION employee_log()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
BEGIN
    RAISE NOTICE 'Employee Inserted';
    RETURN NEW;
END;
$$;
```

| Piece | Meaning |
|---|---|
| `RETURNS TRIGGER` | Required for any function used by a trigger |
| `RAISE NOTICE 'text'` | Prints a message — useful for debugging |
| `RETURN NEW;` | Tells PostgreSQL to proceed with the incoming row (fully explained in Section 7) |

---

## 4. `CREATE TRIGGER` Syntax

**Syntax**
```sql
CREATE TRIGGER trigger_name
BEFORE|AFTER INSERT|UPDATE|DELETE
ON table_name
FOR EACH ROW
EXECUTE FUNCTION function_name();
```

**Example**
```sql
CREATE TRIGGER employee_insert_trigger
AFTER INSERT
ON employees
FOR EACH ROW
EXECUTE FUNCTION employee_log();
```
Read it like English: *"Create a trigger that, after every insert on `employees`, for each affected row, runs `employee_log()`."*

---

## 5. `BEFORE` vs `AFTER`

| | `BEFORE` | `AFTER` |
|---|---|---|
| Runs | Before the database performs the operation | After the operation has already completed |
| Good for | Validation, rejecting bad data, modifying incoming values | Logging, notifications, audit tables, updating related tables |

**Example — validation belongs in `BEFORE`**
```
Attempting to insert salary = -1000
        ↓
BEFORE Trigger checks: negative? → YES → reject
```

**Example — logging belongs in `AFTER`**
```
Employee successfully inserted
        ↓
AFTER Trigger: save an audit log entry
```

💡 **How to Choose** — Use `BEFORE` when you need to reject or transform data before it's written. Use `AFTER` when the main change should already be final and you're reacting to it (logging, cascading updates elsewhere).

---

## 6. `FOR EACH ROW` vs `FOR EACH STATEMENT`

| | `FOR EACH ROW` | `FOR EACH STATEMENT` |
|---|---|---|
| Fires | Once per **affected row** | Once per **SQL statement**, regardless of row count |
| Common use | The vast majority of triggers | Rare, statement-level bookkeeping |

**Example** — inserting 3 employees with a `FOR EACH ROW` trigger fires the trigger **3 times**, once per row.

---

## 7. `NEW` and `OLD`

**Definition** — Special row-like variables PostgreSQL provides inside a trigger function.

| Variable | Represents |
|---|---|
| `NEW` | The **new** version of the row (after the change) |
| `OLD` | The **previous** version of the row (before the change) |

💡 **Memory trick:** `OLD` = before, `NEW` = after. ⭐⭐⭐⭐⭐

**Example — an `UPDATE` changing salary from 70,000 to 80,000**
```sql
OLD.salary   -- 70000
NEW.salary   -- 80000
```

---

## 8. Which Events Have `NEW`/`OLD`?

| Event | `OLD` | `NEW` | Why |
|---|---|---|---|
| `INSERT` | ❌ | ✅ | There was no previous row |
| `UPDATE` | ✅ | ✅ | Both a previous and a new version exist |
| `DELETE` | ✅ | ❌ | The row is being removed — there's no "new" version |

❌ **Common Mistakes**
- Referencing `NEW.column` inside a `DELETE` trigger — `NEW` doesn't exist there.
- Referencing `OLD.column` inside an `INSERT` trigger — `OLD` doesn't exist there.

---

## 9. Building an `INSERT` Audit Trigger

**Step 1 — audit table**
```sql
CREATE TABLE employee_audit (
    id SERIAL PRIMARY KEY,
    employee_id INT,
    action TEXT,
    action_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Step 2 — trigger function**
```sql
CREATE FUNCTION log_employee_insert()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
BEGIN
    INSERT INTO employee_audit (employee_id, action)
    VALUES (NEW.id, 'INSERT');
    RETURN NEW;
END;
$$;
```

**Step 3 — attach the trigger**
```sql
CREATE TRIGGER employee_insert_trigger
AFTER INSERT
ON employees
FOR EACH ROW
EXECUTE FUNCTION log_employee_insert();
```

**Step 4 — test it**
```sql
INSERT INTO employees (id, name, salary) VALUES (5, 'Rahul', 60000);
```
**Result — `employee_audit`**
```
id | employee_id | action
1  | 5           | INSERT
```

---

## 10. Building an `UPDATE` Audit Trigger

**Audit table**
```sql
CREATE TABLE salary_audit (
    id SERIAL PRIMARY KEY,
    employee_id INT,
    old_salary NUMERIC,
    new_salary NUMERIC,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Trigger function — using both `OLD` and `NEW`**
```sql
CREATE FUNCTION log_salary_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
BEGIN
    INSERT INTO salary_audit (employee_id, old_salary, new_salary)
    VALUES (OLD.id, OLD.salary, NEW.salary);
    RETURN NEW;
END;
$$;
```

**Attach — narrowed to only fire when `salary` changes**
```sql
CREATE TRIGGER salary_update_trigger
AFTER UPDATE OF salary
ON employees
FOR EACH ROW
EXECUTE FUNCTION log_salary_change();
```
*(`UPDATE OF salary` means this trigger only fires when the `salary` column specifically is part of the update.)*

```sql
UPDATE employees SET salary = 80000 WHERE id = 1;
```
**Result — `salary_audit`**
```
employee_id | old_salary | new_salary
1           | 70000      | 80000
```

---

## 11. Building a `DELETE` Audit Trigger

```sql
CREATE FUNCTION log_employee_delete()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
BEGIN
    INSERT INTO employee_audit (employee_id, action)
    VALUES (OLD.id, 'DELETE');
    RETURN OLD;    -- ⭐ DELETE has no NEW — must return OLD
END;
$$;

CREATE TRIGGER employee_delete_trigger
AFTER DELETE
ON employees
FOR EACH ROW
EXECUTE FUNCTION log_employee_delete();
```
```sql
DELETE FROM employees WHERE id = 5;
```
**Result — `employee_audit`**
```
employee_id | action
5           | DELETE
```

💡 **Why audit logs matter** — banking, healthcare, e-commerce, and security systems all rely on this pattern to answer "who changed what, and when?"

---

## 12. `BEFORE` Triggers for Validation (`RAISE EXCEPTION`)

**Example — reject a negative salary before it's ever written**
```sql
CREATE FUNCTION validate_salary()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
BEGIN
    IF NEW.salary < 0 THEN
        RAISE EXCEPTION 'Salary cannot be negative';
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER validate_salary_trigger
BEFORE INSERT OR UPDATE
ON employees
FOR EACH ROW
EXECUTE FUNCTION validate_salary();
```
```sql
INSERT INTO employees (id, name, salary) VALUES (10, 'Test', -5000);
-- ERROR: Salary cannot be negative
-- The row is never inserted.
```

| | `RAISE NOTICE` | `RAISE EXCEPTION` |
|---|---|---|
| Effect | Prints a message, execution continues | **Stops** the operation entirely and returns an error |

---

## 13. `BEFORE` Triggers for Modifying Data

**Example — force names to uppercase before saving**
```sql
CREATE FUNCTION uppercase_employee_name()
RETURNS TRIGGER
LANGUAGE plpgsql
AS
$$
BEGIN
    NEW.name := UPPER(NEW.name);
    RETURN NEW;
END;
$$;

CREATE TRIGGER uppercase_name_trigger
BEFORE INSERT OR UPDATE
ON employees
FOR EACH ROW
EXECUTE FUNCTION uppercase_employee_name();
```
```sql
INSERT INTO employees (id, name, salary) VALUES (10, 'rahul', 60000);
-- NEW.name is rewritten to 'RAHUL' BEFORE the row is saved
```

💡 This is exactly why `BEFORE` triggers are powerful — they can **validate** data (Section 12) or **modify** it before it ever reaches the table.

---

## 14. `RETURN NEW` vs `RETURN OLD` vs `RETURN NULL`

| Trigger Event | Return | Why |
|---|---|---|
| `INSERT` | `RETURN NEW;` | There's a new row to proceed with |
| `UPDATE` | `RETURN NEW;` (usually) | Continue with the (possibly modified) new version |
| `DELETE` | `RETURN OLD;` | The row being removed is the "old" one |
| Any `BEFORE` trigger | `RETURN NULL;` | Silently **cancel** the operation entirely |

⚠️ **Notes & Caveats** — For row-level `BEFORE` triggers, the row you return controls what actually gets written (or whether anything is written at all). Forgetting to return the appropriate value is a common source of subtle bugs.

---

## 15. Trigger Best Practices

- **Keep trigger logic simple** — avoid stuffing large business workflows inside a trigger.
- **Use triggers for database-level rules** — audit logs, validation, automatic timestamps, keeping derived data in sync.
- **Watch performance on bulk operations** — inserting 1,000,000 rows with a row-level trigger means that trigger function runs 1,000,000 times.
- **Avoid hidden side effects** — a simple `UPDATE employees SET salary = salary + 1000;` might silently fire several triggers a developer isn't aware of. Document trigger behavior clearly.

---

## 16. Interview Q&A

**Q: What is a trigger, and how does it differ from a regular function?**
A: A trigger is a database object that automatically executes a trigger function when a specified event (`INSERT`/`UPDATE`/`DELETE`) occurs — it's invoked by PostgreSQL itself, not called explicitly the way a regular function is with `SELECT`.

**Q: What does `NEW` represent, and for which events is it available?**
A: `NEW` represents the new version of a row; it's available in `INSERT` and `UPDATE` triggers (there's no "new" row in a `DELETE`).

**Q: What does `OLD` represent, and for which events is it available?**
A: `OLD` represents the previous version of a row; it's available in `UPDATE` and `DELETE` triggers (there's no "old" row in an `INSERT`).

**Q: What's the difference between a `BEFORE` and an `AFTER` trigger?**
A: `BEFORE` runs before the underlying operation and can validate or modify the incoming row (even canceling it via `RETURN NULL`). `AFTER` runs once the operation has already completed and is typically used for logging or related follow-up actions.

**Q: Why is validation usually implemented with a `BEFORE` trigger rather than `AFTER`?**
A: Because a `BEFORE` trigger runs before the row is actually written — it can reject or correct bad data before it's ever persisted. An `AFTER` trigger only reacts after the (potentially invalid) change has already been saved.

**Q: What does `RAISE EXCEPTION` do, compared to `RAISE NOTICE`?**
A: `RAISE NOTICE` simply prints a message and lets execution continue. `RAISE EXCEPTION` stops the entire operation and returns an error — the triggering `INSERT`/`UPDATE`/`DELETE` never completes.

**Q: Why must a `DELETE` trigger function `RETURN OLD;` instead of `RETURN NEW;`?**
A: Because a `DELETE` has no "new" row at all — the only row available is the one being removed, referenced via `OLD`.

---

## 17. Quick Revision Sheet

| Event | `OLD` | `NEW` | Typical `RETURN` |
|---|---|---|---|
| `INSERT` | ❌ | ✅ | `RETURN NEW;` |
| `UPDATE` | ✅ | ✅ | `RETURN NEW;` |
| `DELETE` | ✅ | ❌ | `RETURN OLD;` |

| Goal | Timing |
|---|---|
| Validate / reject bad data | `BEFORE` |
| Modify incoming values | `BEFORE` |
| Logging / audit trail | `AFTER` |
| Notifications / related updates | `AFTER` |

---

## 18. Cheat Sheet

```sql
-- ── TRIGGER FUNCTION SKELETON ─────────────
CREATE FUNCTION my_trigger_fn()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- logic using NEW / OLD
    RETURN NEW;   -- or OLD, or NULL to cancel
END;
$$;

-- ── ATTACH A TRIGGER ──────────────────────
CREATE TRIGGER my_trigger
AFTER INSERT   -- or BEFORE, or UPDATE/DELETE, or "INSERT OR UPDATE"
ON my_table
FOR EACH ROW
EXECUTE FUNCTION my_trigger_fn();

-- ── VALIDATION (BEFORE) ───────────────────
CREATE FUNCTION validate_salary()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF NEW.salary < 0 THEN
        RAISE EXCEPTION 'Salary cannot be negative';
    END IF;
    RETURN NEW;
END; $$;

CREATE TRIGGER validate_salary_trigger
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW EXECUTE FUNCTION validate_salary();

-- ── MODIFY INCOMING DATA (BEFORE) ─────────
CREATE FUNCTION uppercase_name()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.name := UPPER(NEW.name);
    RETURN NEW;
END; $$;

-- ── AUDIT LOG (AFTER) ─────────────────────
CREATE FUNCTION log_salary_change()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO salary_audit (employee_id, old_salary, new_salary)
    VALUES (OLD.id, OLD.salary, NEW.salary);
    RETURN NEW;
END; $$;

CREATE TRIGGER salary_update_trigger
AFTER UPDATE OF salary ON employees
FOR EACH ROW EXECUTE FUNCTION log_salary_change();
```

---

## 19. Preview of Part 35

| Topic | What You'll Learn |
|---|---|
| Query Optimization | What it means, why queries become slow |
| `EXPLAIN` / `EXPLAIN ANALYZE` deep dive | Cost, rows, width, actual time, loops |
| The optimization workflow | Measure → change one thing → measure again |
