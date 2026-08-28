# SQL & PostgreSQL Complete Notes — Part 31: Transactions & ACID Properties

## 📑 Table of Contents
1. What Is a Transaction?
2. `BEGIN`
3. `COMMIT`
4. `ROLLBACK`
5. A Complete Transaction Example — Bank Transfer
6. `SAVEPOINT`
7. `ROLLBACK TO SAVEPOINT`
8. `RELEASE SAVEPOINT`
9. Nested Savepoints
10. ACID Properties
11. Interview Q&A
12. Quick Revision Sheet
13. Cheat Sheet
14. Preview of Part 32

**📋 Series Coverage (Part 31):** `BEGIN`, `COMMIT`, `ROLLBACK`, transactional bank-transfer example, `SAVEPOINT`, `ROLLBACK TO SAVEPOINT`, `RELEASE SAVEPOINT`, nested savepoints, ACID (Atomicity, Consistency, Isolation, Durability)

---

## 1. What Is a Transaction?

**Definition** — A transaction groups one or more SQL statements into a single **all-or-nothing** unit of work.

**Why It Exists** — Some real-world operations genuinely require multiple steps that must **all** succeed together, or **none** should apply. The classic example: a bank transfer is really two steps (debit one account, credit another) — if only the debit happened, money would simply vanish.

```
BEGIN
  ↓
Statement 1
  ↓
Statement 2
  ↓
Statement 3
  ↓
Everything OK?
  ├── YES → COMMIT   (make it all permanent)
  └── NO  → ROLLBACK (undo it all)
```

---

## 2. `BEGIN`

**Definition** — Starts a new transaction block. Statements that follow are **not yet permanent** — they're only finalized once `COMMIT` runs.

**Syntax**
```sql
BEGIN;
```
*(`START TRANSACTION;` is an equivalent, more standard-SQL spelling.)*

---

## 3. `COMMIT`

**Definition** — Ends the current transaction and makes every change made since `BEGIN` **permanent**.

**Syntax**
```sql
COMMIT;
```

⚠️ **Notes & Caveats** — Once committed, a transaction's changes are permanent — there's no "undo" after `COMMIT` succeeds (barring restoring from a backup).

---

## 4. `ROLLBACK`

**Definition** — Ends the current transaction and **undoes** every change made since `BEGIN`.

**Syntax**
```sql
ROLLBACK;
```

---

## 5. A Complete Transaction Example — Bank Transfer

```sql
-- Aman has ₹10,000. Transfer ₹3,000 to Priya.
BEGIN;

UPDATE accounts SET balance = balance - 3000 WHERE name = 'Aman';
UPDATE accounts SET balance = balance + 3000 WHERE name = 'Priya';

COMMIT;
```

**What if the second `UPDATE` fails partway through?**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 3000 WHERE name = 'Aman';
-- ❌ error occurs here
ROLLBACK;
```
Both balances remain **exactly as they were before `BEGIN`** — Aman's money is never deducted without Priya receiving it.

⚠️ **Notes & Caveats — why this matters** — Without wrapping both statements in a transaction, running them as two independent, auto-committed statements risks a real problem: if the first succeeds and the second fails, Aman's balance is permanently reduced while Priya never receives anything. Transactions prevent this "half-completed" state entirely.

---

## 6. `SAVEPOINT`

**Definition** — Creates a named checkpoint **inside** a transaction that you can roll back to, without undoing the entire transaction.

**Why It Exists** — Sometimes only the most recent step of a multi-step transaction needs undoing — not everything back to `BEGIN`.

💡 **Analogy** — A video game checkpoint: dying doesn't send you back to Level 1, just to your most recent save.

**Syntax**
```sql
SAVEPOINT savepoint_name;
```

**Example**
```sql
-- Aman starts at 10,000
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE name = 'Aman';   -- → 9000
SAVEPOINT s1;
UPDATE accounts SET balance = balance - 2000 WHERE name = 'Aman';   -- → 7000
ROLLBACK TO s1;                                                       -- back to 9000
COMMIT;
```
**Execution trace**
```
10000 → (-1000) → 9000 → SAVEPOINT s1 → (-2000) → 7000 → ROLLBACK TO s1 → 9000 → COMMIT → 9000 (final)
```
⚠️ Only the second update is undone — the first update (before the savepoint) survives.

---

## 7. `ROLLBACK TO SAVEPOINT`

**Definition** — Undoes only the changes made **after** the specified savepoint, leaving everything before it intact.

**Syntax**
```sql
ROLLBACK TO savepoint_name;
```

**`ROLLBACK` vs `ROLLBACK TO SAVEPOINT`**

| | `ROLLBACK` | `ROLLBACK TO savepoint` |
|---|---|---|
| Undoes | The entire transaction | Only changes made after that savepoint |
| Transaction status after | Ended | **Still open** — you can continue and later `COMMIT` |

---

## 8. `RELEASE SAVEPOINT`

**Definition** — Removes a savepoint you no longer need, **without** ending the transaction or undoing any work.

**Syntax**
```sql
RELEASE SAVEPOINT savepoint_name;
```

**Example**
```sql
BEGIN;
SAVEPOINT s1;
UPDATE ...;
RELEASE SAVEPOINT s1;   -- checkpoint removed, transaction continues
COMMIT;
```

---

## 9. Nested Savepoints

PostgreSQL allows multiple savepoints in one transaction — you can return to any earlier one.

```sql
BEGIN;
UPDATE ...;
SAVEPOINT A;
UPDATE ...;
SAVEPOINT B;
UPDATE ...;
ROLLBACK TO B;   -- undoes only the update made after SAVEPOINT B
COMMIT;
```
```
Update 1 → kept
SAVEPOINT A
Update 2 → kept
SAVEPOINT B
Update 3 → removed (rolled back to B)
```

❌ **Common Mistakes**
- Thinking `SAVEPOINT` ends the transaction — only `COMMIT` or `ROLLBACK` does.
- Thinking `ROLLBACK TO SAVEPOINT` undoes the *entire* transaction — it only undoes work after that specific savepoint.
- Thinking a `COMMIT` can be undone afterward — once committed, changes are permanent.

---

## 10. ACID Properties

⭐ **One of the most famous SQL interview topics — don't just memorize the acronym, understand what each letter guarantees.**

### A — Atomicity
**Definition** — A transaction is all-or-nothing: either every statement succeeds, or none of them take effect.

**Bank example** — Transfer ₹5,000: deduct from Aman, add to Priya. If the "add" step fails, atomicity guarantees the "deduct" step is undone too — nobody loses money.

💡 **Memory trick:** Atomic = one indivisible piece — can't be half-completed.

### C — Consistency
**Definition** — A transaction must move the database from one **valid** state to another valid state — it can never leave data in a broken/invalid condition.

**Example** — If a `CHECK` constraint says balance can never go negative, a transaction attempting to withdraw more than the available balance is rejected — the database stays consistent.

### I — Isolation
**Definition** — Concurrently running transactions should not improperly interfere with each other's intermediate (uncommitted) state.

**Example** — If Person A is mid-transfer and Person B checks the balance at the exact same moment, isolation controls whether B sees the old balance, the new balance, or is made to wait — preventing B from seeing a half-completed, inconsistent value. *(The precise behavior depends on the transaction's isolation level.)*

### D — Durability
**Definition** — Once a transaction is committed, its changes survive **even a crash immediately afterward**.

**Example** — `COMMIT;` succeeds, then the server loses power a millisecond later. When PostgreSQL restarts, the committed data is still there.

💡 **Memory trick:** COMMIT = permanent, forever.

**Complete ACID Table**

| Property | Guarantees |
|---|---|
| **A**tomicity | Everything or nothing |
| **C**onsistency | Database always remains valid |
| **I**solation | Transactions don't improperly interfere with each other |
| **D**urability | Committed data survives crashes |

---

## 11. Interview Q&A

**Q: What's the difference between `COMMIT` and `SAVEPOINT`?**
A: `COMMIT` ends the transaction and makes all its changes permanent. `SAVEPOINT` creates an intermediate checkpoint **within** an ongoing transaction — the transaction is still open afterward.

**Q: What's the difference between `ROLLBACK` and `ROLLBACK TO SAVEPOINT`?**
A: `ROLLBACK` undoes the entire transaction and ends it. `ROLLBACK TO SAVEPOINT` undoes only the changes made after that specific savepoint, and the transaction remains open, so you can continue working and eventually `COMMIT`.

**Q: Which ACID property guarantees "all or nothing"?**
A: Atomicity.

**Q: Which ACID property guarantees committed data survives a crash?**
A: Durability.

**Q: Why does a bank transfer need to be wrapped in a transaction?**
A: Because it's really two dependent steps (debit + credit). Without a transaction, a failure between the two steps could leave the database in an inconsistent state — money deducted from one account without appearing in the other. Wrapping both in a transaction guarantees atomicity: either both succeed, or neither does.

**Q: Can you `COMMIT` and later undo it?**
A: No — once a transaction is committed, its changes are permanent (short of restoring from a backup).

---

## 12. Quick Revision Sheet

| Goal | Command |
|---|---|
| Start a transaction | `BEGIN;` |
| Make changes permanent | `COMMIT;` |
| Undo the whole transaction | `ROLLBACK;` |
| Create a checkpoint | `SAVEPOINT name;` |
| Undo back to a checkpoint | `ROLLBACK TO name;` |
| Remove a checkpoint (keep going) | `RELEASE SAVEPOINT name;` |
| All-or-nothing | Atomicity |
| Always valid state | Consistency |
| No improper interference | Isolation |
| Survives crashes once committed | Durability |

---

## 13. Cheat Sheet

```sql
-- ── BASIC TRANSACTION ─────────────────────
BEGIN;
UPDATE accounts SET balance = balance - 3000 WHERE name = 'Aman';
UPDATE accounts SET balance = balance + 3000 WHERE name = 'Priya';
COMMIT;

-- ── ROLLING BACK ON ERROR ─────────────────
BEGIN;
UPDATE accounts SET balance = balance - 3000 WHERE name = 'Aman';
-- error happens
ROLLBACK;

-- ── SAVEPOINTS ────────────────────────────
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE name = 'Aman';
SAVEPOINT s1;
UPDATE accounts SET balance = balance - 2000 WHERE name = 'Aman';
ROLLBACK TO s1;              -- undo only the second update
RELEASE SAVEPOINT s1;         -- optional: drop the checkpoint, keep going
COMMIT;

-- ── NESTED SAVEPOINTS ─────────────────────
BEGIN;
SAVEPOINT a;
SAVEPOINT b;
SAVEPOINT c;
ROLLBACK TO b;   -- undoes work after b (including anything after c)
COMMIT;
```

---

## 14. Preview of Part 32

| Topic | What You'll Learn |
|---|---|
| `CREATE FUNCTION` | Reusable blocks of SQL that return a value |
| `CREATE PROCEDURE` | Reusable blocks that perform an action, `CALL` |
| Function vs Procedure | When to use which |
| Parameters | Passing values into functions and procedures |
