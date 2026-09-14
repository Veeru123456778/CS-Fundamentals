# Database Locking and Concurrency Control

## Goal

Understand how a database safely executes multiple transactions at the same time, how row-level locks work, when locks are acquired and released, and how the database prevents concurrent updates from corrupting data.

> **Technology Agnostic Note:** Locking is implemented by the database engine (MySQL InnoDB, PostgreSQL, SQL Server, etc.). Backend frameworks only start transactions; the database manages locks internally.

---

# 1. Why Do We Need Locks?

Imagine two users buying the last product simultaneously.

Initial stock:

| product_id | stock |
|------------|-------|
| 101 | 1 |

Both transactions read `stock = 1` and both try to reduce it.

Without locking:

- T1 updates stock to 0.
- T2 also updates stock to 0.

Two orders are placed even though only one item existed.

**Locks prevent concurrent transactions from modifying data incorrectly.**

---

# 2. Who Maintains Locks?

The **database engine** maintains all locks internally.

The lock manager keeps track of:

- Which transaction owns which lock.
- Which rows are locked.
- Which transactions are waiting.
- Whether a requested lock can be granted.

Applications do not manage row locks manually.

---

# 3. Lock Granularity

Locks can exist at different levels.

| Lock Type | What Gets Locked |
|-----------|------------------|
| Row Lock | Individual row. |
| Page Lock | Database page containing multiple rows. |
| Table Lock | Entire table. |

In InnoDB, **row-level locking** is used for most transactional queries.

---

# 4. Shared Lock (S Lock)

A shared lock is used for **protected reading**.

Purpose:

- Read the row.
- Prevent other transactions from modifying it.

SQL example:

```sql
SELECT *
FROM inventory
WHERE product_id = 101
FOR SHARE;
```

Behavior:

- Multiple transactions can hold shared locks on the same row.
- Exclusive locks are blocked until all shared locks are released.


## Does a Normal SELECT Acquire a Shared Lock?

No.

A normal `SELECT` is a **non-locking read** in InnoDB and uses MVCC to read a committed snapshot.

A shared lock is acquired only when explicitly requested using `FOR SHARE` (or similar locking reads).


## When Should FOR SHARE Be Used?

Use `FOR SHARE` when a transaction needs to read data and ensure that no other transaction modifies those rows until the current transaction finishes.

Example:

- Read inventory.
- Perform business validation.
- Continue using the same inventory value inside the transaction.


## Inventory Example: Why Use FOR UPDATE Instead of FOR SHARE?

If two transactions both acquire a shared lock and later try to update the same row, both will need to upgrade to an exclusive lock, creating lock contention or a deadlock.

For read-modify-write operations like inventory deduction, `FOR UPDATE` is the correct locking strategy because it acquires the exclusive lock at the beginning.

---

# 5. Exclusive Lock (X Lock)

An exclusive lock is used for **writing**.

Purpose:

- Update, delete, or lock a row for future updates.

SQL example:

```sql
SELECT *
FROM inventory
WHERE product_id = 101
FOR UPDATE;
```

or

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 101;
```

Behavior:

- Only one transaction can hold an exclusive lock.
- No shared or exclusive locks can be granted on that row until it is released.

---

# 6. Shared Lock vs Exclusive Lock

| Situation | Allowed? |
|-----------|----------|
| Shared + Shared | ✅ Yes |
| Shared + Exclusive | ❌ No |
| Exclusive + Shared | ❌ No |
| Exclusive + Exclusive | ❌ No |

---

# 7. Does Every UPDATE Acquire a Lock?

**Yes.**

Even a single `UPDATE` statement acquires an exclusive row lock.

Example:

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 101;
```

The lock is automatically acquired by the database.

You do **not** write locking logic yourself.

---

# 8. Does a Transaction Acquire Locks or Queries?

Important distinction:

**Queries acquire locks on behalf of the transaction.**

Example:

```sql
BEGIN;

SELECT ... FOR SHARE;

UPDATE ...;

COMMIT;
```

- `SELECT ... FOR SHARE` acquires an S lock.
- `UPDATE` acquires (or upgrades to) an X lock.
- Locks belong to the transaction until release.

---

# 9. Lock Upgrade

Suppose one transaction reads first and updates later.

```sql
BEGIN;

SELECT * FROM inventory
WHERE product_id = 101
FOR SHARE;

UPDATE inventory
SET stock = stock - 1
WHERE product_id = 101;
```

Flow:

1. Shared lock acquired.
2. Same transaction requests an exclusive lock.
3. Database upgrades **S → X** if no other transaction holds a shared lock.

---

# 10. Can Multiple Transactions Hold Shared Locks?

**Yes.**

Example:

| Transaction | Lock |
|-------------|------|
| T1 | Shared |
| T2 | Shared |
| T3 | Shared |

All three can read simultaneously.

---

# 11. What Happens During Lock Upgrade?

Suppose:

| Transaction | Lock |
|-------------|------|
| T1 | Shared |
| T2 | Shared |

Now T1 wants an exclusive lock.

Database behavior:

- Upgrade request waits.
- T2 must release its shared lock first.
- After all other shared locks disappear, T1 gets the exclusive lock.

---

# 12. FOR SHARE vs FOR UPDATE

## FOR SHARE

```sql
SELECT ...
FOR SHARE;
```

- Protected read.
- Other readers allowed.
- Writers blocked.

## FOR UPDATE

```sql
SELECT ...
FOR UPDATE;
```

- Read while immediately acquiring an exclusive lock.
- Readers using locking reads wait.
- Writers wait.

Used when you know the row will be updated.

---

# 13. When Should FOR UPDATE Be Used?

Typical inventory example.

```sql
BEGIN;

SELECT stock
FROM inventory
WHERE product_id = 101
FOR UPDATE;

UPDATE inventory
SET stock = stock - 1
WHERE product_id = 101;

COMMIT;
```

Reason:

No other transaction can modify the stock between the read and the update.

---

# 14. What Happens if Two Transactions Request the Same Exclusive Lock?

Example:

T1:

```sql
SELECT ...
FOR UPDATE;
```

T2:

```sql
SELECT ...
FOR UPDATE;
```

Behavior:

- Database grants the lock to one transaction.
- The other transaction waits.
- This is **lock waiting**, not a deadlock.

---

# 15. What Happens if One Transaction Requests S Lock and Another Requests X Lock Together?

Both requests arrive almost simultaneously.

Behavior:

- Database decides based on arrival order.
- One transaction acquires the lock first.
- The other transaction waits.
- Requests are **not rejected**.

---

# 16. When Are Locks Released?

| Lock Type | Released When |
|-----------|---------------|
| Autocommit query | Query completes. |
| Explicit transaction | COMMIT or ROLLBACK. |

Example:

```sql
BEGIN;

SELECT ... FOR UPDATE;

UPDATE ...;

COMMIT;
```

The exclusive lock remains until `COMMIT`.

---

# 17. Read Your Own Writes

Inside the same transaction:

```sql
BEGIN;

UPDATE stock = 9;

SELECT stock;

COMMIT;
```

The transaction reads **9**, even before commit.

Reason:

A transaction always sees its own uncommitted changes.

---

# 18. What Do Other Transactions See?

Example:

T1:

```sql
BEGIN;

UPDATE stock = 9;
```

T2:

```sql
SELECT stock;
```

Behavior:

- T2 does **not** see `9`.
- T2 sees the last committed value.
- Uncommitted changes are invisible to other transactions.

---

# 19. Lock Waiting vs Deadlock

## Lock Waiting

T1 owns a lock.

T2 waits.

Eventually T1 commits.

T2 continues.

## Deadlock

T1 waits for T2.

T2 waits for T1.

Neither can continue.

The database detects this cycle and aborts one transaction automatically.

Detailed deadlock detection is covered in the next section.

---

# 20. Responsibilities of the Database

The database automatically:

- Acquires row locks.
- Tracks lock ownership.
- Blocks incompatible requests.
- Upgrades locks when possible.
- Releases locks after commit/rollback.
- Detects deadlocks.

The application only defines transaction boundaries.

---

# Interview Takeaways

- Locks are managed entirely by the database engine.
- Queries acquire locks on behalf of a transaction.
- Shared locks allow concurrent reads but block writers.
- Exclusive locks allow one writer and block everyone else.
- `FOR UPDATE` acquires an exclusive lock before updating.
- Locks remain until `COMMIT` or `ROLLBACK` inside an explicit transaction.
- A transaction can read its own uncommitted writes, but other transactions cannot.
- Lock waiting is normal; deadlock is a cyclic waiting situation detected by the database.