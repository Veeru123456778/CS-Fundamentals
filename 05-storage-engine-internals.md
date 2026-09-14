# Storage Engine Internals (Buffer Pool, WAL, Undo/Redo Logs)

## Goal

Understand how a database stores data on SSD, why it first writes data to RAM, how committed and uncommitted changes are managed, and how the database survives crashes without losing committed data.

> **Technology Agnostic Note:** The concepts in this document are explained using MySQL InnoDB terminology (Buffer Pool, Undo Log, Redo Log, Checkpoint), but similar ideas exist in PostgreSQL, SQL Server, Oracle, and other transactional databases.

---

# 1. Why Doesn't the Database Read and Write SSD Directly?

SSD is much slower than RAM.

Every database operation would become expensive if every query directly modified files on disk.

### Solution

The database keeps frequently used pages in **RAM** and periodically synchronizes them with SSD.

### Memory Flow

```text
Application
      │
      ▼
Buffer Pool (RAM)
      │
      ▼
Table Files (SSD)
```

**RAM is used for speed. SSD is used for durability.**

---

# 2. What is the Buffer Pool?

The **Buffer Pool** is a large region of RAM managed by the database engine.

It caches:

- Table pages.
- Index pages.
- Frequently accessed data.
- Frequently accessed B+ Tree nodes.

### Responsibilities

- Read pages from SSD.
- Cache pages in RAM.
- Update pages in RAM.
- Flush modified pages back to SSD later.

### Important

The Buffer Pool belongs to the **database process**, not the backend application.

---

# 3. What is a Database Page?

The database does not read individual rows from SSD.

It reads **pages**.

A page is the smallest storage unit managed by the storage engine.

### In InnoDB

- Default page size = **16 KB**.

### One Page Can Contain

- Multiple rows.
- Index entries.
- Metadata.

Example:

```text
Page (16 KB)

----------------------------------
Row 101
Row 102
Row 103
Row 104
...
----------------------------------
```

Every read/write happens at the page level.

---

# 4. How Reading Works

Suppose:

```sql
SELECT *
FROM orders
WHERE order_id = 101;
```

### Flow

1. Database checks Buffer Pool.
2. If page exists → Return immediately.
3. Otherwise:
   - Read page from SSD.
   - Load page into Buffer Pool.
   - Return requested row.

### Cache Hit

Page already in RAM.

Very fast.

### Cache Miss

Page loaded from SSD first.

Slower.

---

# 5. How Writing Works

Suppose:

```sql
UPDATE orders
SET amount = 700
WHERE order_id = 105;
```

The database **does not immediately update SSD**.

### Flow

1. Load page into Buffer Pool.
2. Modify page in RAM.
3. Page becomes a **Dirty Page**.
4. SSD update happens later.

---

# 6. What is a Dirty Page?

A **Dirty Page** is a page whose RAM version differs from the SSD version.

### Example

SSD:

| order_id | amount |
|----------|--------|
| 105 | 500 |

Buffer Pool:

| order_id | amount |
|----------|--------|
| 105 | **700** |

RAM contains newer data.

SSD still contains older data.

This page is now **dirty**.

### Important

Dirty **does not mean uncommitted**.

It only means:

> RAM version has not yet been synchronized to SSD.

---

# 7. Dirty Page vs Uncommitted Data

These are different concepts.

| Concept | Meaning |
|---------|---------|
| Dirty Page | RAM page differs from SSD page. |
| Uncommitted Data | Changes belong to an active transaction. |

A dirty page may contain:

- Committed updates.
- Uncommitted updates.
- Both.

The storage engine knows how to handle both safely.

---

# 8. Why Can Dirty Pages Be Flushed Before Commit?

This seems surprising.

Suppose:

```sql
BEGIN;

UPDATE inventory
SET stock = 9;
```

Transaction has **not committed**.

Yet the database may flush this dirty page to SSD.

### Why is this Safe?

Because the database also stores **Undo Log** information.

If the transaction later rolls back:

- Undo Log restores the previous value.
- SSD returns to the committed state.

Therefore flushing a dirty page **does not mean the transaction committed**.

---

# 9. Why Flush Dirty Pages Before Commit?

RAM is limited.

The Buffer Pool eventually fills up.

The database needs free pages.

So background threads periodically flush dirty pages to SSD.

Reasons:

- Free Buffer Pool memory.
- Reduce recovery work after crashes.
- Keep SSD synchronized gradually.

---

# 10. What is Write-Ahead Logging (WAL)?

**Write-Ahead Logging** means:

> Before modifying the table files, write the change to the Redo Log.

### Rule

Redo Log is persisted **before** table pages.

This guarantees crash recovery.

---

# 11. What is the Redo Log?

The **Redo Log** records committed changes.

Purpose:

- Recover committed transactions after a crash.
- Guarantee durability.

It stores information describing page modifications.

### Important

Redo Log is **not** the table.

It is a recovery log.

---

# 12. Commit Flow (High-Level)

Suppose:

```sql
BEGIN;

UPDATE orders
SET amount = 700
WHERE order_id = 105;

COMMIT;
```

### Flow

1. Update Buffer Pool page.
2. Dirty page created.
3. Write corresponding Redo Log record to SSD.
4. `COMMIT` succeeds.
5. Dirty page is flushed to SSD later.

### Key Rule

A transaction is considered durable **after the Redo Log is safely written**, not after the table page is flushed.

---

# 13. Why WAL Improves Performance

Without WAL:

Every commit would require writing the full data page to SSD.

With WAL:

- Write a small sequential Redo Log entry.
- Return success quickly.
- Flush full pages later in batches.

Sequential SSD writes are much faster than random page writes.

---

# 14. What Happens If the Server Crashes Immediately After COMMIT?

Example:

1. Redo Log written.
2. Dirty page still only in RAM.
3. Server crashes.

After restart:

- Database reads Redo Log.
- Reapplies committed changes.
- Updates table pages.

Committed data is recovered.

This is durability.

---

# 15. Summary So Far

| Component | Purpose |
|-----------|---------|
| Buffer Pool | RAM cache for table/index pages. |
| Page | Smallest storage unit (16 KB). |
| Dirty Page | RAM page differs from SSD page. |
| WAL | Write Redo Log before table pages. |
| Redo Log | Recover committed changes after crashes. |