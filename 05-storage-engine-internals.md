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



---

# 16. What is the Undo Log?

The **Undo Log** stores enough information to restore the previous committed state of a row.

Its two responsibilities are:

1. **Rollback** uncommitted transactions.
2. **Provide older row versions for MVCC** (consistent snapshots).

> **Important:** Undo Log is **not** a list of SQL queries. It stores the previous values (or enough metadata to reconstruct them).

✅ The current uncommitted version lives in the row itself (in the Buffer Pool/table page), along with transaction metadata (trx_id and a pointer to the undo record).

There is at most one current uncommitted version per row because writes to the same row are serialized by the exclusive lock.

Before every UPDATE/DELETE transaction modifies a row, MySQL writes the previous version into the Undo Log. This creates a version chain of the row.

✅ If a transaction's snapshot needs an older committed version, MySQL walks this undo chain until it finds the version visible to that snapshot.


### Example: Undo Version Chain

Suppose the row is updated multiple times.

| Version Chain | Value |
|----------------|-------|
| **Current Row (latest version)** | **7** |
| Undo Version 1 (previous committed version) | **8** |
| Undo Version 2 (older committed version) | **9** |
| Undo Version 3 (oldest committed version) | **10** |

### How MVCC Uses It

If a transaction started when the committed value was **9**, MySQL follows the undo chain:

```text
Current Row (7)
      │
      ▼
Undo Version 1 (8)
      │
      ▼
Undo Version 2 (9)   ← Snapshot-visible version
      │
      ▼
Undo Version 3 (10)
```

The transaction receives **9**, even though the current row has already been updated to **7**.

---

# 17. What Exactly is Stored in the Undo Log?

Suppose the committed row is:

| order_id | amount |
|----------|--------|
| 105 | **500** |

Transaction updates it:

```sql
UPDATE orders
SET amount = 700
WHERE order_id = 105;
```

Before changing the row, the database writes an Undo Log record.

### Undo Record

| Row | Previous Value |
|-----|-----------------|
| 105 | amount = 500 |

Then the Buffer Pool page is updated to **700**.

### Mental Model

- Current row → `700`
- Undo Log → "Previous committed value was `500`."

---

# 18. How Rollback Uses the Undo Log

Suppose:

```sql
BEGIN;

UPDATE amount = 700;

ROLLBACK;
```

### Flow

1. Database looks at Undo Log.
2. Reads previous value (`500`).
3. Restores the row.
4. Transaction ends.

After rollback:

| order_id | amount |
|----------|--------|
| 105 | **500** |

No partial update remains.

---

# 19. Why Does MVCC Need the Undo Log?

MVCC allows transactions to see **older committed versions**.

Example:

Initial committed value:

```text
Stock = 10
```

Transaction T1 updates stock to `9`.

Buffer Pool now contains `9`.

Transaction T2 started earlier and still needs the snapshot value `10`.

### Where Does `10` Come From?

The database reconstructs it using the Undo Log.

### Internal View

| Component | Value |
|----------|-------|
| Current Row | 9 |
| Undo Log | Previous value = 10 |

This is how Repeatable Read provides a consistent snapshot.

---

# 20. Undo Log Lifecycle

Undo records do **not** live forever.

### After COMMIT

- Current row becomes committed.
- Older versions remain temporarily if active transactions still need them.

### After All Transactions Finish

Database removes obsolete Undo Log records.

This cleanup is performed by a background purge thread.

---

# 21. Undo Log vs Redo Log

| Undo Log | Redo Log |
|----------|----------|
| Restores previous state. | Replays committed changes. |
| Used for rollback. | Used for crash recovery. |
| Supports MVCC snapshots. | Supports durability. |
| Contains old versions. | Contains committed modifications. |

A good interview sentence:

> Undo moves **backward**. Redo moves **forward**.

---

# 22. What Happens During COMMIT?

Suppose:

```sql
BEGIN;

UPDATE inventory
SET stock = 9;

COMMIT;
```

### Internal Commit Flow

1. Page updated in Buffer Pool.
2. Undo Log already exists.
3. Redo Log record written to SSD.
4. COMMIT succeeds.
5. Transaction releases locks.
6. Dirty page remains in RAM until background flush.

### Important

COMMIT does **not** immediately write the table page to SSD.

---

# 23. Background Flush Thread

The database continuously runs background threads.

One responsibility:

**Flush dirty pages from Buffer Pool to SSD.**

### Why?

- Free RAM.
- Keep table files updated.
- Reduce crash recovery work.

Applications do not trigger flushing manually.

---

# 24. What is a Checkpoint?

A **checkpoint** is a marker maintained by the database saying:

> All dirty pages up to this Redo Log position have been written to the table files.

### Why Is This Needed?

Without checkpoints:

- Redo Log would grow forever.
- Crash recovery would replay every log ever written.

Checkpoint limits recovery work.

---

# 25. Checkpoint Flow

### Step 1

Redo Log contains committed updates.

### Step 2

Background thread flushes dirty pages to SSD.

### Step 3

Checkpoint advances.

Meaning:

Those Redo Log entries are now reflected in table files.

---

# 26. Does the Database Write Table Pages From the Redo Log?

**No.**

This is an important distinction.

### Normal Operation

```text
Buffer Pool
     │
     ▼
Table File (SSD)
```

Dirty pages are flushed **from RAM to SSD**.

### Role of Redo Log

Redo Log is only used when:

- Crash recovery is needed.
- Database must replay committed changes that were not flushed.

Redo Log is **not** the source of normal table writes.

---

# 27. Crash Recovery

Suppose:

1. Transaction commits.
2. Redo Log safely written.
3. Dirty page still only in RAM.
4. Server crashes.

### Startup Recovery

Database performs:

1. Read latest checkpoint.
2. Replay Redo Log records after the checkpoint.
3. Restore committed changes.
4. Undo unfinished transactions.

Database reaches a consistent state automatically.

---

# 28. What Happens to Uncommitted Transactions After Crash?

Suppose:

```sql
BEGIN;

UPDATE inventory
SET stock = 9;

-- Crash here
```

There was no COMMIT.

Recovery process:

- Redo committed transactions.
- Undo unfinished transaction using Undo Log.

Final value becomes the previous committed value.

---

# 29. Full Data Lifecycle (RAM to SSD)

### Read Operation

```text
SSD
 │
 ▼
Buffer Pool
 │
 ▼
Query Result
```

### Write Operation

```text
Application
      │
      ▼
Buffer Pool (Dirty Page)
      │
      ├── Undo Log (Old Value)
      ├── Redo Log (Recovery Record)
      ▼
COMMIT
      │
Background Flush
      ▼
Table Files (SSD)
```

### Crash Recovery

```text
Checkpoint
      │
      ▼
Redo Committed Changes
      │
      ▼
Undo Uncommitted Changes
```

---

# 30. Storage Engine Summary

| Component | Responsibility |
|-----------|----------------|
| Buffer Pool | Cache pages in RAM. |
| Page | Smallest storage unit (16 KB). |
| Dirty Page | RAM version differs from SSD version. |
| Undo Log | Rollback + MVCC old versions. |
| Redo Log | Recover committed changes after crash. |
| WAL | Write Redo Log before table pages. |
| Checkpoint | Marks Redo Log entries already persisted to table files. |
| Background Flush Thread | Writes dirty pages from RAM to SSD. |

---

# Interview Takeaways

- The Buffer Pool caches table and index pages in RAM for fast reads and writes.
- A dirty page means the RAM page is newer than the SSD page; it does **not** mean the transaction is committed.
- Undo Log stores previous row versions for rollback and MVCC snapshots.
- Redo Log stores committed modifications for crash recovery.
- COMMIT succeeds after the Redo Log is persisted, not after table pages are flushed.
- Dirty pages are flushed from the Buffer Pool to SSD by background threads.
- Checkpoints record how far committed dirty pages have been persisted, reducing crash recovery work.
- Crash recovery **replays Redo Logs** for committed transactions and **uses Undo Logs** to roll back unfinished transactions.