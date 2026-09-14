# Database Transactions and ACID (Foundation of Database Consistency)

## Goal

Understand how a backend application communicates with a database, what a transaction is, why transactions exist, how `COMMIT` and `ROLLBACK` work, and how ACID guarantees consistency.

> **Technology Agnostic Note:** The concepts in this document apply to any backend technology (Java/Spring Boot, Go, Node.js, Python, .NET, Rust, etc.). Only the APIs used to start and manage transactions differ.

---

# 1. How Does a Backend Application Communicate with the Database?

A backend application does **not** read or write database files directly. It communicates with the database server over a TCP connection.

## Request Flow

1. A client sends a request to the backend.
2. The backend executes business logic.
3. The backend sends SQL queries to the database through a database connection.
4. The database executes the queries and returns results.
5. The backend returns the response to the client.

The backend and the database are separate processes, even when they run on the same server.

### Spring Boot Example

Uses JDBC (usually through JPA/Hibernate) to communicate with MySQL.

### Go Example

Uses the `database/sql` package with a MySQL/PostgreSQL driver.

---

# 2. What is a Database Connection?

A **database connection** is a long-lived TCP connection between the backend process and the database process.

Instead of creating a new connection for every request, applications reuse existing connections.

### Why Reuse Connections?

Creating a TCP connection is expensive because it involves:

- TCP handshake.
- Authentication.
- Database session creation.

Reusing connections improves performance significantly.

---

# 3. What is a Connection Pool?

A **connection pool** is a collection of reusable database connections maintained by the backend application.

Instead of opening a new connection for every request:

- Request gets an available connection.
- Executes queries.
- Returns the connection back to the pool.

### Example

Pool size = **20**

- First 20 requests get database connections immediately.
- 21st request waits until a connection becomes free.

### Spring Boot Example

Uses **HikariCP** by default.

### Go Example

`database/sql` maintains a connection pool automatically.

> **Interview takeaway:** Thread pool controls concurrent request execution, while connection pool controls concurrent database access.

---

# 4. What is a SQL Query?

A SQL query is a single database operation.

Examples:

```sql
SELECT * FROM products WHERE id = 101;

UPDATE products
SET stock = stock - 1
WHERE id = 101;

DELETE FROM cart
WHERE user_id = 5;
```

A query is executed independently unless it is part of a transaction.

---

# 5. What is a Transaction?

A **transaction** is a group of one or more SQL queries executed as a single logical unit.

The database guarantees that either:

- All queries succeed.
- Or none of them take effect.

### Example

Buying a product requires multiple operations.

```sql
BEGIN;

UPDATE inventory
SET stock = stock - 1
WHERE id = 101;

INSERT INTO orders (...);

UPDATE wallet
SET balance = balance - 100
WHERE user_id = 5;

COMMIT;
```

All three queries belong to one transaction.

---

# 6. Why Do We Need Transactions?

Without transactions:

- Inventory may decrease.
- Order insertion may fail.
- Wallet may remain unchanged.

The database becomes inconsistent.

Transactions ensure the database always remains valid.

---

# 7. Transaction Lifecycle

A transaction follows this lifecycle:

```text
BEGIN
   │
   ├── Query 1
   ├── Query 2
   ├── Query 3
   │
COMMIT  (Success)
   │
ROLLBACK (Failure)
```

### Steps

1. Transaction starts with `BEGIN`.
2. Queries execute sequentially.
3. If everything succeeds → `COMMIT`.
4. If something fails → `ROLLBACK`.

---

# 8. Implicit vs Explicit Transactions

## Implicit Transaction (Autocommit)

Every SQL query becomes its own transaction.

```sql
UPDATE products
SET stock = 9
WHERE id = 101;
```

Conceptually, the database performs:

```text
BEGIN
UPDATE ...
COMMIT
```

## Explicit Transaction

Developer controls the transaction boundaries.

```sql
BEGIN;

UPDATE ...
INSERT ...
DELETE ...

COMMIT;
```

Multiple queries commit together.

---

# 9. What Does COMMIT Mean?

`COMMIT` means:

- Transaction completed successfully.
- Changes become visible to other transactions.
- Database guarantees durability.

After `COMMIT`, rollback is no longer possible.

---

# 10. What Does ROLLBACK Mean?

`ROLLBACK` means:

- Cancel the transaction.
- Restore previous values.
- No partial updates remain.

Example:

```sql
BEGIN;

UPDATE inventory ...

-- Something failed

ROLLBACK;
```

Inventory returns to its previous value.

---

# 11. What Happens if the Application Crashes Before COMMIT?

Suppose:

```sql
BEGIN;

UPDATE inventory ...

INSERT order ...

-- Backend crashes here.
```

Since `COMMIT` never happened:

- Database detects connection loss.
- Entire transaction is rolled back automatically.
- No partial changes remain.

This is handled by the database, not by the application.

---

# 12. What is ACID?

ACID defines the guarantees provided by transactions.

| Property | Meaning |
|----------|---------|
| **Atomicity** | All operations succeed or all fail. |
| **Consistency** | Database always moves from one valid state to another. |
| **Isolation** | Concurrent transactions do not interfere incorrectly. |
| **Durability** | Committed data survives crashes. |

### Example

Order placement:

- Inventory update.
- Order creation.
- Wallet deduction.

All happen together because of **Atomicity**.

---

# 13. Atomicity

Either everything happens, or nothing happens.

Example:

- Inventory updated.
- Order insertion fails.

Database rolls back inventory automatically.

No partial state remains.

---

# 14. Consistency

Transactions preserve database rules.

Example:

- Stock should never become negative.
- Wallet balance should never violate constraints.

After the transaction finishes, all constraints remain valid.

---

# 15. Isolation (Introduction)

Multiple transactions may execute concurrently.

Isolation determines:

- What one transaction can see from another transaction.
- How concurrent transactions interact.

Detailed locking and isolation levels are covered in the next sections.

---

# 16. Durability (Introduction)

Once `COMMIT` succeeds:

- Data must survive crashes.
- Database uses **Redo Logs** internally to guarantee this.

Redo logs and crash recovery are discussed in a dedicated section.

---

# 17. How Transactions are Written in Backend Code

The concept is identical across technologies.

### Spring Boot Example

```java
@Transactional
public void placeOrder() {
    updateInventory();
    createOrder();
    deductWallet();
}
```

The framework starts and commits/rolls back the transaction automatically.

### Go Example

```go
tx, _ := db.Begin()

// queries using tx

tx.Commit()
// or
tx.Rollback()
```

The application defines transaction boundaries, while the database executes `COMMIT` and `ROLLBACK`.

---

# 18. Responsibilities: Application vs Database

| Application Responsibility | Database Responsibility |
|---------------------------|--------------------------|
| Start transaction boundary. | Execute queries. |
| Decide when work is successful. | Maintain transaction state. |
| Commit or signal failure. | Perform COMMIT or ROLLBACK. |
| Handle retry on transient failures. | Guarantee ACID properties. |

---

# Interview Takeaways

- A transaction groups multiple SQL operations into one atomic unit.
- `COMMIT` makes all changes permanent and visible.
- `ROLLBACK` restores the database to its previous committed state.
- Implicit transactions commit per query; explicit transactions commit once for the entire group.
- If the application crashes before `COMMIT`, the database automatically rolls back the uncommitted transaction.
- ACID provides Atomicity, Consistency, Isolation, and Durability guarantees.