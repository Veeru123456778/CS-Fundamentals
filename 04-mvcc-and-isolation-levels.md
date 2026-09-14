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


