# SQL & PostgreSQL Complete Notes — Part 39: Security II — Privileges, GRANT & REVOKE

## 📑 Table of Contents
1. What Is a Privilege?
2. The Core Table Privileges
3. Privileges Are Object-Specific
4. `GRANT` — Giving Privileges
5. Multiple Privileges & Multiple Objects
6. `ALL PRIVILEGES` (and Why to Avoid It)
7. Database & Schema-Level Privileges
8. Column-Level Privileges
9. `REVOKE` — Removing Privileges
10. The Multi-Role Trap — Revoking Doesn't Always Remove All Access
11. `REVOKE` vs `DROP ROLE`
12. `PUBLIC` — Handle With Care
13. The Principle of Least Privilege
14. Authentication vs Authorization
15. Interview Q&A
16. Quick Revision Sheet
17. Cheat Sheet
18. Preview of Part 40

**📋 Series Coverage (Part 39):** privileges (`SELECT`/`INSERT`/`UPDATE`/`DELETE`/etc.), object-specific permissions, `GRANT` (privileges & role membership), `ALL PRIVILEGES`, database/schema/sequence/function privileges, column-level `GRANT`, `REVOKE`, the multi-role access trap, `REVOKE` vs `DROP ROLE`, `PUBLIC`, least privilege, authentication vs authorization

---

## 1. What Is a Privilege?

**Definition** — Permission to perform a specific operation on a specific database object.

```
Role: analyst  +  Privilege: SELECT  +  Object: employees
        ↓
analyst can SELECT from employees
```

💡 **Analogy** — Phone app permissions: Camera, Location, Microphone are each separate, specific grants — having one doesn't imply the others.

---

## 2. The Core Table Privileges

| Privilege | Meaning |
|---|---|
| `SELECT` | Read data |
| `INSERT` | Add new rows |
| `UPDATE` | Modify existing rows |
| `DELETE` | Remove rows |
| `TRUNCATE` | Empty the table |
| `REFERENCES` | Create a foreign key referencing this table |
| `TRIGGER` | Create triggers on this table |

💡 **Memory trick:** SELECT = See · INSERT = Add · UPDATE = Change · DELETE = Remove.

⚠️ **Notes & Caveats** — These are entirely **separate** grants. Having `SELECT` does **not** imply `INSERT`, `UPDATE`, or `DELETE` — each must be granted individually (or together, Section 5).

---

## 3. Privileges Are Object-Specific

A privilege isn't just "`SELECT`" in the abstract — it's always **`SELECT` on a specific object**.
```sql
GRANT SELECT ON employees TO analyst;
```
```
analyst
├── SELECT → employees   ✅
└── SELECT → salaries    ❌ (never granted)
```
`analyst` can query `employees` but gets a permission error querying `salaries`, unless that's separately granted too.

---

## 4. `GRANT` — Giving Privileges

**Syntax**
```sql
GRANT privilege ON object TO role;
```

**Example**
```sql
GRANT SELECT ON employees TO analyst;
```
Read it like English: *"Give the `analyst` role `SELECT` permission on `employees`."*

---

## 5. Multiple Privileges & Multiple Objects

**Multiple privileges in one statement**
```sql
GRANT SELECT, INSERT, UPDATE ON employees TO hr;
```
⚠️ `DELETE` is **not** included here — only the three explicitly listed.

**Multiple tables in one statement**
```sql
GRANT SELECT ON employees, departments, projects TO analyst;
```

---

## 6. `ALL PRIVILEGES` (and Why to Avoid It)

**Syntax**
```sql
GRANT ALL PRIVILEGES ON employees TO admin;
```
This grants every applicable privilege at once (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `REFERENCES`, `TRIGGER`).

⚠️ **Notes & Caveats** — Convenient, but a security anti-pattern by default. Reserve `ALL PRIVILEGES` for roles that genuinely need full control — not as a shortcut to avoid thinking about what's actually required (Section 13).

---

## 7. Database & Schema-Level Privileges

**Database-level — `CONNECT`**
```sql
GRANT CONNECT ON DATABASE company TO analyst;
```
Without `CONNECT`, a role can't even open a connection to that database.

**Schema-level — `USAGE` and `CREATE`**
```sql
GRANT USAGE ON SCHEMA public TO analyst;    -- can access objects within the schema
GRANT CREATE ON SCHEMA development TO developer;   -- can create new objects there
```

**Sequence-level** (relevant for `SERIAL`/identity columns)
```sql
GRANT USAGE, SELECT ON SEQUENCE employees_id_seq TO hr;
```

**Function-level**
```sql
GRANT EXECUTE ON FUNCTION calculate_salary(integer) TO analyst;
```

💡 **The layering to remember:**
```
Database → Schema → Table/Sequence/Function
```
A role may need the appropriate permission at **each** relevant layer, not just the final table.

---

## 8. Column-Level Privileges

**Definition** — PostgreSQL can restrict `SELECT` (and some other privileges) to specific **columns**, not the whole row.

**Example — hide salary from analysts**
```sql
GRANT SELECT (id, name, department) ON employees TO analyst;
```
```
analyst
├── id          ✅
├── name        ✅
├── department  ✅
└── salary      ❌
```

---

## 9. `REVOKE` — Removing Privileges

**Syntax**
```sql
REVOKE privilege ON object FROM role;
```

**Example**
```sql
REVOKE SELECT ON employees FROM analyst;
```

**Two uses, mirroring `GRANT`:**

| Form | Meaning |
|---|---|
| `REVOKE SELECT ON employees FROM analyst;` | Remove a **privilege** |
| `REVOKE analyst FROM tushar;` | Remove **role membership** |

⚠️ **Notes & Caveats** — `REVOKE` never deletes the role itself — only the specified permission/membership.

---

## 10. The Multi-Role Trap — Revoking Doesn't Always Remove All Access

⭐ **A genuinely important, commonly-missed real-world scenario.**

Suppose:
```
tushar
├── analyst → SELECT employees
└── manager → SELECT employees
```
```sql
REVOKE SELECT ON employees FROM analyst;
```
**What happens?** `analyst` loses `SELECT` — **but Tushar still has it**, because the `manager` role independently grants the same privilege.

```
tushar
├── analyst → ❌ SELECT
└── manager → SELECT ✅  (still active!)
```

💡 **The lesson** — Before revoking a privilege from a role, always ask: *"Could this user still have the same access through another role, or a direct grant?"* Revoking from one source doesn't guarantee the access disappears everywhere.

---

## 11. `REVOKE` vs `DROP ROLE`

| | `REVOKE` | `DROP ROLE` |
|---|---|---|
| Removes | A specific privilege or role membership | The **entire role/user itself** |
| Example | `REVOKE SELECT ON employees FROM analyst;` | `DROP ROLE analyst;` |

⚠️ **Notes & Caveats** — Don't confuse the two: revoking access is reversible (just `GRANT` again); dropping a role is a structural deletion.

---

## 12. `PUBLIC` — Handle With Care

**Definition** — A special PostgreSQL pseudo-role representing **all roles**.

```sql
GRANT SELECT ON employees TO PUBLIC;
```
means **everyone** who can connect gets `SELECT` on `employees`.

⚠️ **Notes & Caveats** — Accidentally granting broad privileges `TO PUBLIC` can expose sensitive data to every role on the system. Reserve it for genuinely public, non-sensitive data.

---

## 13. The Principle of Least Privilege

⭐ **The single most important security principle in this whole phase.**

**Definition** — Give a role only the permissions it actually needs to do its job — nothing more.

```sql
-- ✅ An analyst who only reads reports
GRANT SELECT ON reports TO analyst;

-- ❌ Overkill — unnecessary risk
GRANT ALL PRIVILEGES ON reports TO analyst;
```

**Why it matters** — If an analyst accidentally (or maliciously) runs `DELETE FROM reports;`, the database should **reject** it outright because the `DELETE` privilege was never granted in the first place — least privilege turns a potential disaster into a simple permission error.

---

## 14. Authentication vs Authorization

⭐ **A classic, very common interview question.**

| | Authentication | Authorization |
|---|---|---|
| Question | *Who are you?* | *What are you allowed to do?* |
| Example | Username + password | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |

**Complete chain:**
```
USER (who?)
   ↓
ROLE (which group?)
   ↓
PRIVILEGE (what can they do?)
   ↓
OBJECT (where?)
```
Example: *Tushar is a member of `analyst`, and `analyst` has `SELECT` on `employees` — so Tushar can read `employees`.*

---

## 15. Interview Q&A

**Q: What is a privilege in PostgreSQL?**
A: Permission granted to a role to perform a specific operation — like selecting, inserting, updating, or executing — on a specific database object.

**Q: Does granting `SELECT` on a table also allow `INSERT`?**
A: No — each privilege is separate and must be granted individually (or together in one `GRANT` statement listing multiple privileges).

**Q: What's the difference between the two uses of `GRANT`?**
A: `GRANT privilege ON object TO role;` gives a specific permission. `GRANT role TO user;` makes a user (or another role) a member of that role, inheriting its privileges.

**Q: If you revoke a privilege from a role, is a user who was a member of that role guaranteed to lose that access?**
A: Not necessarily — if the same user is also a member of a different role that independently grants the same privilege, or has a direct grant, they'll retain access through that other path.

**Q: What's the difference between `REVOKE` and `DROP ROLE`?**
A: `REVOKE` removes a specific privilege or role membership, leaving the role itself intact. `DROP ROLE` deletes the role/user entirely.

**Q: Why is `GRANT ... TO PUBLIC` risky?**
A: It grants the privilege to every role on the system, potentially exposing sensitive data far more broadly than intended — it should be reserved for genuinely public, non-sensitive access.

**Q: What is the principle of least privilege?**
A: Granting a role only the specific permissions it needs to perform its job — nothing broader — so that mistakes or misuse are contained by the permission system itself.

**Q: What's the difference between authentication and authorization?**
A: Authentication establishes *who* is connecting (e.g., username/password). Authorization determines *what* that identity is allowed to do once connected (e.g., which `SELECT`/`INSERT`/`UPDATE`/`DELETE` privileges it holds).

---

## 16. Quick Revision Sheet

| Goal | Syntax |
|---|---|
| Grant a privilege | `GRANT privilege ON object TO role;` |
| Grant several privileges | `GRANT SELECT, INSERT ON t TO role;` |
| Grant on several objects | `GRANT SELECT ON t1, t2 TO role;` |
| Grant everything | `GRANT ALL PRIVILEGES ON t TO role;` (use sparingly) |
| Database connect access | `GRANT CONNECT ON DATABASE db TO role;` |
| Schema access | `GRANT USAGE ON SCHEMA s TO role;` |
| Column-only access | `GRANT SELECT (col1, col2) ON t TO role;` |
| Remove a privilege | `REVOKE privilege ON object FROM role;` |
| Remove role membership | `REVOKE role FROM user;` |
| Delete a role entirely | `DROP ROLE role;` |
| Grant to everyone | `GRANT privilege ON object TO PUBLIC;` (use carefully) |

---

## 17. Cheat Sheet

```sql
-- ── GRANT: PRIVILEGES ─────────────────────
GRANT SELECT ON employees TO analyst;
GRANT SELECT, INSERT, UPDATE ON employees TO hr;
GRANT SELECT ON employees, departments, projects TO analyst;
GRANT ALL PRIVILEGES ON employees TO admin;         -- use sparingly

-- ── GRANT: DATABASE / SCHEMA / SEQUENCE / FUNCTION ──
GRANT CONNECT ON DATABASE company TO analyst;
GRANT USAGE ON SCHEMA public TO analyst;
GRANT CREATE ON SCHEMA development TO developer;
GRANT USAGE, SELECT ON SEQUENCE employees_id_seq TO hr;
GRANT EXECUTE ON FUNCTION calculate_salary(integer) TO analyst;

-- ── GRANT: COLUMN-LEVEL ───────────────────
GRANT SELECT (id, name, department) ON employees TO analyst;

-- ── GRANT: ROLE MEMBERSHIP ────────────────
GRANT analyst TO tushar;

-- ── REVOKE ────────────────────────────────
REVOKE SELECT ON employees FROM analyst;
REVOKE INSERT, UPDATE, DELETE ON employees FROM hr;
REVOKE ALL PRIVILEGES ON employees FROM analyst;
REVOKE analyst FROM tushar;

-- ── PUBLIC (careful!) ─────────────────────
GRANT SELECT ON employees TO PUBLIC;
REVOKE SELECT ON employees FROM PUBLIC;

-- ── DELETE THE ROLE ITSELF ────────────────
DROP ROLE analyst;
```

---

## 18. Preview of Part 40

| Topic | What You'll Learn |
|---|---|
| `pg_dump` | Creating a logical backup of one database |
| Plain vs Custom format | `-Fp` vs `-Fc`, and when to use each |
| Schema-only / data-only backups | `-s` and `-a` |
| Selective backups | Specific tables (`-t`) and schemas (`-n`) |
