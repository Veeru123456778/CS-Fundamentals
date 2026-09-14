# MVCC and Isolation Levels

## Goal

Understand how a database allows multiple transactions to read and write data concurrently without corrupting data, why transactions see different versions of the same row, and how isolation levels control this behavior.

> **Technology Agnostic Note:** MVCC (Multi-Version Concurrency Control) and isolation levels are database concepts implemented by the database engine (MySQL InnoDB, PostgreSQL, SQL Server, etc.). Backend frameworks only start transactions; the database controls visibility of data.

---

# 1. Why Do We Need Isolation?

Transactions often execute at the same time.

Example:

| Transaction T1 | Transaction T2 |
|----------------|----------------|
| Update stock from **10 → 9** | Read stock |
| Not committed yet | Wants current stock |

Question:

**Should T2 read `9` or `10`?**

If T2 reads `9` and T1 later rolls back, T2 has read invalid data.

Isolation exists to prevent these kinds of problems.

**Purpose of Isolation**

- Prevent transactions from seeing invalid intermediate states.
- Give each transaction a predictable view of the database.
- Allow concurrency while maintaining correctness.

---

# 2. What is MVCC (Multi-Version Concurrency Control)?

MVCC is the technique used by modern databases to allow **reads and writes to happen concurrently** without blocking each other in most cases.

## Core Idea

Instead of keeping only one copy of a row, the database temporarily maintains **multiple versions** of that row.

### Example

Initially:

| product_id | stock |
|------------|-------|
| 101 | **10** |

Transaction T1:

```sql
BEGIN;

UPDATE inventory
SET stock = 9
WHERE product_id = 101;
```

Now the database internally has:

| Version | Visible To |
|---------|------------|
| **10 (Committed Version)** | Other transactions |
| **9 (Uncommitted Version)** | Transaction T1 only |

Both versions coexist temporarily.

---

# 3. Committed vs Uncommitted Version

## Committed Version

The latest value that has successfully completed `COMMIT`.

Visible to all new transactions.

## Uncommitted Version

A temporary version created inside an active transaction.

Visible **only** to the transaction that created it.

### Important Rule

Other transactions never read uncommitted versions under normal isolation levels.

---

# 4. Read Your Own Writes

A transaction always sees its own modifications.

Example:

```sql
BEGIN;

UPDATE inventory
SET stock = 9
WHERE product_id = 101;

SELECT stock
FROM inventory
WHERE product_id = 101;
```

Result inside T1:

```text
9
```

Even though the transaction has not committed yet.

### Why?

A transaction always reads its own latest version first.

---

# 5. What Do Other Transactions See?

Suppose:

### Transaction T1

```sql
BEGIN;

UPDATE inventory
SET stock = 9
WHERE product_id = 101;
```

### Transaction T2

```sql
SELECT stock
FROM inventory
WHERE product_id = 101;
```

Result:

```text
10
```

T2 reads the last committed version.

It cannot see T1's uncommitted update.

---

# 6. Dirty Read

## Definition

A dirty read happens when one transaction reads **uncommitted data** written by another transaction.

### Example

| Transaction T1 | Transaction T2 |
|----------------|----------------|
| Update stock = 9 | Read stock = 9 |
| Rollback | Already used wrong value |

T2 observed data that never became permanent.

This is called a **Dirty Read**.

---

# 7. Why Dirty Reads Are Dangerous

Suppose inventory is 1.

T1 updates it to 0.

T2 sees 0 and rejects another customer's order.

Later T1 rolls back.

Inventory becomes 1 again.

Now one customer was incorrectly rejected.

---

# 8. How MVCC Prevents Dirty Reads

Instead of exposing the new value immediately:

| Version | Visible To |
|---------|------------|
| Stock = 10 | Everyone else |
| Stock = 9 | T1 only |

T2 receives:

```text
10
```

until T1 commits.

Dirty reads are prevented.

---

# 9. Read Uncommitted Isolation Level

This is the weakest isolation level.

Behavior:

- Transactions may read uncommitted versions.
- Dirty reads are allowed.

Example:

| T1 | T2 |
|----|----|
| Update stock = 9 | Reads 9 before commit |

Rarely used in production.

---

# 10. Read Committed Isolation Level

This is the default isolation level in PostgreSQL and Oracle.

Behavior:

- Transactions read only committed versions.
- Dirty reads are prevented.
- Every query sees the latest committed value available at the time that query starts.

### Example

Initial stock:

```text
10
```

Transaction T1:

```sql
BEGIN;

UPDATE inventory
SET stock = 9;
```

Transaction T2:

```sql
SELECT stock;
```

Result:

```text
10
```

After T1 commits:

```sql
COMMIT;
```

If T2 runs another query:

```sql
SELECT stock;
```

Now T2 sees:

```text
9
```

Each query gets a fresh committed snapshot.

> **Important:** The behavior depends on the isolation level.
>
> - **READ COMMITTED:** Every query gets a fresh committed snapshot, so the second `SELECT` returns `9`.
> - **REPEATABLE READ (MySQL default):** The transaction keeps one consistent snapshot from its first read, so the second `SELECT` still returns `10`.

---

# 11. Summary So Far

| Concept | Behavior |
|---------|----------|
| MVCC | Multiple versions of a row exist temporarily. |
| Read Your Own Writes | Transaction sees its own uncommitted updates. |
| Committed Version | Visible to other transactions. |
| Uncommitted Version | Visible only to the owning transaction. |
| Dirty Read | Reading another transaction's uncommitted data. |
| Read Uncommitted | Allows dirty reads. |
| Read Committed | Prevents dirty reads by exposing only committed versions. |



---

# 12. Non-Repeatable Read

## Definition

A **Non-Repeatable Read** happens when a transaction reads the same row twice and gets **different committed values**.

### Example

Initial stock:

| product_id | stock |
|------------|-------|
| 101 | **10** |

### READ COMMITTED Behavior

| Transaction T1 | Transaction T2 |
|----------------|----------------|
| `BEGIN` | `BEGIN` |
| Read stock → **10** | |
| | Update stock → **9** |
| | `COMMIT` |
| Read stock again → **9** | |

The same transaction (`T1`) read **10** first and **9** later.

This is a **Non-Repeatable Read**.

---

# 13. Why Is Non-Repeatable Read a Problem?

Suppose an order placement transaction:

1. Reads inventory = 10.
2. Performs validation.
3. Reads inventory again before placing the order.

If another transaction changes the inventory in between, the transaction observes different values while executing the same business logic.

Sometimes business logic requires a **consistent view** throughout the transaction.

---

# 14. Repeatable Read (MySQL Default)

**Repeatable Read** prevents non-repeatable reads.

Instead of giving every query a fresh snapshot, the database gives the transaction **one consistent snapshot**.

### Example

Initial stock:

| product_id | stock |
|------------|-------|
| 101 | **10** |

### Repeatable Read Behavior

| Transaction T1 | Transaction T2 |
|----------------|----------------|
| `BEGIN` | |
| Read stock → **10** | |
| | `BEGIN` |
| | Update stock → **9** |
| | `COMMIT` |
| Read stock again → **10** | |

Even after T2 commits, T1 continues reading **10**.

---

# 15. What is a Snapshot?

A **snapshot** is the committed view of the database visible to a transaction.

### Important Rule

The snapshot is created when the transaction performs its **first consistent read**.

Everything committed before that snapshot is visible.

Everything committed after that snapshot is invisible for normal reads.

### Snapshot Timeline

Initial committed value:

```text
Stock = 10
```

T1 starts transaction and performs first read.

Snapshot contains:

```text
Stock = 10
```

Later T2 commits:

```text
Stock = 9
```

Snapshot inside T1 **does not change**.

---

# 16. Read Your Own Writes Still Works

Snapshots do not hide a transaction's own updates.

Example:

```sql
BEGIN;

SELECT stock;      -- 10

UPDATE inventory
SET stock = 9
WHERE product_id = 101;

SELECT stock;      -- 9
```

Behavior:

- Snapshot says committed value is 10.
- Transaction's own update overrides the snapshot.
- Transaction reads 9.

### Rule

A transaction always sees:

1. Its own latest writes.
2. Otherwise, the snapshot version.

---

# 17. Read Committed vs Repeatable Read

| Behavior | READ COMMITTED | REPEATABLE READ |
|----------|----------------|-----------------|
| Dirty Reads | ❌ Prevented | ❌ Prevented |
| Snapshot Lifetime | Per Query | Per Transaction |
| Latest Committed Value | Every Query | Only First Snapshot |
| Non-Repeatable Reads | Possible | Prevented |

This is the biggest difference between the two isolation levels.

---

# 18. Phantom Read

## Definition

A phantom read happens when **new rows appear or disappear** within the same transaction.

Unlike non-repeatable reads, this is **not an update to an existing row**.

It is a change in the **set of rows returned**.

### Example

Orders table:

| amount |
|--------|
| 100 |
| 200 |

Transaction T1:

```sql
BEGIN;

SELECT *
FROM orders
WHERE amount > 50;
```

Returns **2 rows**.

Transaction T2:

```sql
INSERT INTO orders(amount)
VALUES (150);

COMMIT;
```

Now T1 executes the same query again.

```sql
SELECT *
FROM orders
WHERE amount > 50;
```

Returns **3 rows**.

A new row appeared.

This is a **Phantom Read**.

---

# 19. Why Phantom Reads Matter

Imagine calculating today's sales.

Transaction reads all today's orders.

Another transaction inserts a new order during the calculation.

The report changes midway through execution.

Sometimes this is undesirable.

---

# 20. Gap Locks

Gap Locks are MySQL's mechanism for preventing phantom reads.

Instead of locking only existing rows, the database locks the **gap between index values**.

### Example

Existing IDs:

| order_id |
|----------|
| 100 |
| 110 |
| 130 |

Transaction T1:

```sql
SELECT *
FROM orders
WHERE order_id BETWEEN 110 AND 130
FOR UPDATE;
```

Database locks:

- Row 110.
- Row 130.
- Gap between them.

Now T2 cannot insert:

```sql
INSERT INTO orders(order_id=120);
```

The insert waits.

---

# 21. Next-Key Lock

A **Next-Key Lock** combines:

- Row Lock.
- Gap Lock.

It locks both:

- Existing row.
- Gap before the next index value.

This is the default locking behavior for locking reads in MySQL's `REPEATABLE READ`.

---

# 22. When Are Gap Locks Used?

Gap locks are **not** used for ordinary `SELECT`.

They are used for **locking reads**, such as:

```sql
SELECT ...
FOR UPDATE;

SELECT ...
FOR SHARE;
```

under `REPEATABLE READ`.

Normal MVCC reads do not acquire gap locks.

---

# 23. Serializable Isolation Level

This is the strongest isolation level.

Behavior:

- Transactions execute as if they ran one after another.
- Readers may block writers.
- Writers may block readers.

Provides maximum correctness but lowest concurrency.

---

# 24. Isolation Levels Summary

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------------|-----------|---------------------|--------------|
| Read Uncommitted | ✅ Possible | ✅ Possible | ✅ Possible |
| Read Committed | ❌ Prevented | ✅ Possible | ✅ Possible |
| Repeatable Read (MySQL) | ❌ Prevented | ❌ Prevented | ❌ Prevented (using MVCC + Gap Locks) |
| Serializable | ❌ Prevented | ❌ Prevented | ❌ Prevented |

---

# 25. MVCC vs Locks

MVCC handles **ordinary reads**.

Locks handle **protected reads and writes**.

### Use MVCC

```sql
SELECT * FROM products;
```

- No row lock.
- Reads snapshot.

### Use Locks

```sql
SELECT * FROM products
FOR UPDATE;
```

- Exclusive lock acquired.
- Prevents concurrent modifications.

Use locking reads only when business logic requires protecting rows.

---

# Interview Takeaways

- MVCC keeps multiple versions of rows temporarily.
- Every transaction sees a consistent snapshot under `REPEATABLE READ`.
- A transaction always sees its own uncommitted writes.
- `READ COMMITTED` creates a new snapshot for every query.
- `REPEATABLE READ` keeps one snapshot for the entire transaction.
- Phantom reads involve changes in the result set, not updates to existing rows.
- Gap Locks prevent inserts into an index range.
- Next-Key Locks combine row locks and gap locks to prevent phantom reads.
- MVCC improves concurrency because normal reads usually do not block writes.