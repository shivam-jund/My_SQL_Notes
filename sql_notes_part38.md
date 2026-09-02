# SQL & PostgreSQL Complete Notes — Part 38: Security I — Users & Roles

## 📑 Table of Contents
1. Why Database Security Matters
2. What Is a User?
3. `CREATE USER`
4. What Is a Role?
5. `USER` vs `ROLE` — The Core Relationship
6. `CREATE ROLE` — With and Without `LOGIN`
7. Role Membership — Roles as Permission Groups
8. Role Attributes
9. Changing Passwords & Dropping Roles
10. A Complete Example — Building an Analyst Role
11. Checking Roles (`\du`)
12. Interview Q&A
13. Quick Revision Sheet
14. Cheat Sheet
15. Preview of Part 39

**📋 Series Coverage (Part 38):** why database security exists, `CREATE USER`, `CREATE ROLE`, `LOGIN`/`NOLOGIN`, user vs role relationship, role membership (`GRANT role TO user`), role attributes (`SUPERUSER`, `CREATEDB`, `CREATEROLE`), `ALTER USER`/`DROP ROLE`, `\du`

---

## 1. Why Database Security Matters

**The problem** — A company database holds `employees`, `salaries`, `customers`, `orders`. Should every connected user be able to delete employees, view salaries, or drop tables? Obviously not — different people need different levels of access:

```
Admin              → nearly everything
HR                 → employee information
Regular employee   → only their own information
Reporting user     → read-only access to reports
```

That's what PostgreSQL's roles and privileges system manages.

---

## 2. What Is a User?

**Definition** — An identity that can **connect/log in** to PostgreSQL.

💡 **Analogy** — PostgreSQL as a building with a security guard at the door: the guard's first question is *"who are you?"* — that's the user.

---

## 3. `CREATE USER`

**Syntax**
```sql
CREATE USER username WITH PASSWORD 'password';
```

**Example**
```sql
CREATE USER tushar WITH PASSWORD 'mypassword';
```

⚠️ **Notes & Caveats — what actually happens under the hood:** `CREATE USER` creates a **role with the `LOGIN` attribute enabled**. PostgreSQL's entire security model is fundamentally built on roles — "user" is really just a convenient, familiar name for a login-capable role.
```
CREATE USER  ≈  CREATE ROLE ... LOGIN ...
```

---

## 4. What Is a Role?

**Definition** — An identity that can own database objects and receive privileges. Roles can also function as **permission groups**.

```
ROLE → has permissions → can access database objects
```

**Example roles:** `analyst`, `developer`, `hr`, `admin`.

---

## 5. `USER` vs `ROLE` — The Core Relationship

⭐ **This is one of the most confused PostgreSQL concepts, and a very common interview question.**

```
USER ≈ ROLE + LOGIN
```

```sql
CREATE USER tushar WITH PASSWORD 'mypassword';
```
is essentially equivalent to:
```sql
CREATE ROLE tushar LOGIN PASSWORD 'mypassword';
```

| | Role | User |
|---|---|---|
| Can have permissions | ✅ | ✅ |
| Can own objects | ✅ | ✅ |
| Can log in | Only if `LOGIN` is set | ✅ (by definition) |
| Commonly used as a group | ✅ Very common | Less common |

---

## 6. `CREATE ROLE` — With and Without `LOGIN`

**Without `LOGIN` (a "group" role):**
```sql
CREATE ROLE analyst;
```
By default, this role **cannot** connect to PostgreSQL directly — it acts purely as a permission container.

**With `LOGIN` (a "user" role):**
```sql
CREATE ROLE tushar LOGIN PASSWORD 'password';
```
This behaves exactly like a user.

**Explicitly stating `NOLOGIN`** (for clarity):
```sql
CREATE ROLE analyst NOLOGIN;
```

💡 **Memory trick:** `LOGIN` answers *"can this role enter PostgreSQL?"*

---

## 7. Role Membership — Roles as Permission Groups

**The problem** — Suppose 100 employees all need the exact same `SELECT` permission on `employees` and `departments`. Granting that permission to 100 individual users one at a time is unmanageable.

**The solution — group roles:**
```sql
CREATE ROLE analyst;
GRANT SELECT ON employees, departments TO analyst;
```
Then simply add members:
```sql
GRANT analyst TO tushar;
GRANT analyst TO rahul;
GRANT analyst TO aman;
```

💡 **Analogy** — Think of a role as an access badge. Give the badge new permissions once, and everyone already holding that badge automatically gets them.

**Removing membership:**
```sql
REVOKE analyst FROM tushar;
```

⚠️ **Notes & Caveats — two distinct uses of `GRANT`/`REVOKE`:**

| Form | Meaning |
|---|---|
| `GRANT SELECT ON table TO role;` | Grants a **privilege** (fully covered in Part 39) |
| `GRANT role TO user;` | Grants **role membership** |

**Role hierarchy** — roles can also be members of other roles:
```sql
CREATE ROLE readonly;
GRANT SELECT ON employees TO readonly;

CREATE ROLE analyst;
GRANT readonly TO analyst;   -- analyst inherits readonly's privileges
```
```
readonly → SELECT
analyst  → readonly + additional permissions
senior_analyst → analyst + additional permissions
```

---

## 8. Role Attributes

| Attribute | Meaning |
|---|---|
| `LOGIN` | Can connect/authenticate |
| `SUPERUSER` | Extremely powerful — bypasses most permission checks |
| `CREATEDB` | Can create new databases |
| `CREATEROLE` | Can create/manage other roles |
| `PASSWORD 'x'` | Sets the authentication password |

**Example**
```sql
CREATE ROLE developer LOGIN PASSWORD 'password' CREATEDB;
```

⚠️ **Notes & Caveats** — `SUPERUSER` and `CREATEROLE` are powerful administrative capabilities. Following the **least privilege** principle (fully explored in Part 39), don't grant them unless genuinely necessary — `postgres` is commonly the initial superuser role, and ordinary application users should almost never be superusers.

---

## 9. Changing Passwords & Dropping Roles

**Change a password**
```sql
ALTER USER tushar WITH PASSWORD 'newpassword';
-- equivalently:
ALTER ROLE tushar WITH PASSWORD 'newpassword';
```

**Drop a role/user**
```sql
DROP USER tushar;
-- or
DROP ROLE tushar;
```

⚠️ **Notes & Caveats** — If a role owns database objects or has other dependencies, PostgreSQL may block the drop until those dependencies are addressed first.

---

## 10. A Complete Example — Building an Analyst Role

```sql
-- Step 1: create the permission role
CREATE ROLE analyst;

-- Step 2: give it a privilege
GRANT SELECT ON employees TO analyst;

-- Step 3: create an actual login user
CREATE USER tushar WITH PASSWORD 'password';

-- Step 4: add the user to the role
GRANT analyst TO tushar;
```
**Result**
```
tushar → analyst → SELECT → employees
```
Tushar can now `SELECT * FROM employees;` — but cannot `INSERT`, `UPDATE`, or `DELETE`, since only `SELECT` was granted.

---

## 11. Checking Roles (`\du`)

**In `psql`:**
```
\du
```
```
Role name | Attributes
----------+----------------
postgres  | Superuser
analyst   |
tushar    | Login
```

---

## 12. Interview Q&A

**Q: What is a PostgreSQL user, technically?**
A: A login-capable role. PostgreSQL's security model is fundamentally role-based, and a "user" is simply a role with the `LOGIN` attribute enabled.

**Q: What's the difference between `CREATE USER` and `CREATE ROLE`?**
A: `CREATE USER` creates a role with `LOGIN` enabled by default. `CREATE ROLE` creates a role without `LOGIN` unless you explicitly add it — making it more suited to acting as a pure permission group.

**Q: Why use roles as permission groups instead of granting privileges to each user individually?**
A: It dramatically simplifies management — grant a privilege to the role once, then make users members of that role. Updating the role's privileges automatically affects every member, instead of requiring repeated individual grants.

**Q: What does `LOGIN` control?**
A: Whether a role can authenticate and connect to PostgreSQL directly. A role without `LOGIN` can still hold privileges and be granted to other roles/users — it just can't connect on its own.

**Q: What is role membership, and how is it granted?**
A: Making one role/user a member of another role, inheriting that role's privileges — done with `GRANT role_name TO user_or_role;`.

**Q: Why should `SUPERUSER` be granted carefully?**
A: A superuser bypasses most normal permission checks — granting it broadly defeats the purpose of having a fine-grained privilege system in the first place, and violates the least-privilege principle.

---

## 13. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Create a login user | `CREATE USER name WITH PASSWORD 'pw';` |
| Create a group role (no login) | `CREATE ROLE name;` |
| Create a login role explicitly | `CREATE ROLE name LOGIN PASSWORD 'pw';` |
| Add a user to a role | `GRANT role_name TO user_name;` |
| Remove a user from a role | `REVOKE role_name FROM user_name;` |
| Change a password | `ALTER USER name WITH PASSWORD 'new';` |
| Delete a role/user | `DROP ROLE name;` |
| List roles | `\du` |

---

## 14. Cheat Sheet

```sql
-- ── CREATE USERS / ROLES ──────────────────
CREATE USER tushar WITH PASSWORD 'mypassword';
CREATE ROLE analyst;                              -- no login by default
CREATE ROLE analyst NOLOGIN;                       -- explicit
CREATE ROLE developer LOGIN PASSWORD 'pw' CREATEDB;

-- ── ROLE MEMBERSHIP ───────────────────────
GRANT analyst TO tushar;      -- add to role
REVOKE analyst FROM tushar;   -- remove from role

-- ── ROLE HIERARCHY ────────────────────────
CREATE ROLE readonly;
GRANT SELECT ON employees TO readonly;
CREATE ROLE analyst;
GRANT readonly TO analyst;    -- analyst inherits readonly's privileges

-- ── PASSWORDS & CLEANUP ───────────────────
ALTER USER tushar WITH PASSWORD 'newpassword';
DROP ROLE IF EXISTS tushar;

-- ── INSPECT ───────────────────────────────
\du
```

---

## 15. Preview of Part 39

| Topic | What You'll Learn |
|---|---|
| Privileges | `SELECT`, `INSERT`, `UPDATE`, `DELETE`, and more |
| `GRANT` | Giving privileges and role membership |
| `REVOKE` | Removing them — and why removal isn't always total |
| Least Privilege | The guiding principle behind good access design |
