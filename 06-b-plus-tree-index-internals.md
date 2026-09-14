# B+ Tree Index Internals (Clustered, Secondary & Composite Indexes)

## Goal

Understand how database indexes are physically stored, how B+ Trees make lookups `O(log N)`, how clustered and secondary indexes differ, and how inserts, page splits, and composite indexes work internally.

> **Technology Agnostic Note:** The concepts are explained using MySQL InnoDB because it stores indexes as B+ Trees. PostgreSQL, SQL Server, Oracle, and many other databases also use B+ Trees (with implementation differences).

---

# 1. What is an Index?

An **index** is a data structure maintained by the database to locate rows quickly without scanning the entire table.

Without an index:

```sql
SELECT * FROM orders WHERE order_id = 105;
```

The database scans the table row by row (**O(N)**).

With an index:

- Database traverses a **B+ Tree**.
- Reaches the required row in **O(log N)**.

> **Interview takeaway:** An index trades extra storage and write cost for much faster reads.

---

# 2. Is a B+ Tree Real or Just a Concept?

It is a **real B+ Tree** stored on disk.

When an index is created:

```sql
CREATE INDEX idx_email ON orders(email);
```

The database creates another B+ Tree for `email`.

Each index is maintained independently.

---

# 3. Where is the B+ Tree Stored?

Indexes are stored **inside the database files on SSD**.

Frequently accessed index pages are cached in RAM.

| Location | Stores |
|----------|--------|
| SSD | Entire B+ Tree pages. |
| Buffer Pool (RAM) | Frequently used index pages. |

Just like table pages, index pages are loaded into RAM on demand.

---

# 4. What is a B+ Tree Node?

A **node** is actually one **database page**.

For InnoDB:

- Default page size = **16 KB**.

Each page stores:

- Keys.
- Metadata.
- Child pointers (internal/root pages).
- Row data or primary keys (leaf pages).

A page can contain **hundreds or thousands of keys**, depending on key size.

---

# 5. Root, Internal and Leaf Nodes

## Root Node

Stores:

- Separator keys.
- Child page pointers.

Example:

| Pointer | Range |
|---------|-------|
| P0 | order_id less than 120 |
| P1 | 120 ≤ order_id less than 160 |
| P2 | 160 ≤ order_id less than 200 |
| P3 | order_id greater than or equal to 200 |

Root never stores row data.

---

## Internal Node

Stores:

- Separator keys.
- Child page pointers.

Internal nodes only help route searches.

They do not contain table rows.

---

## Leaf Node

Leaf nodes store sorted index entries.

The contents depend on the type of index:

- Clustered Index → Complete rows.
- Secondary Index → Secondary key + Primary key.

Leaf nodes are linked together using a **next leaf pointer**.

---

# 6. How Search Works

Suppose we search:

```sql
SELECT * FROM orders WHERE order_id = 125;
```

Steps:

1. Read Root page.
2. Choose correct child pointer using key ranges.
3. Traverse internal nodes.
4. Reach the leaf page.
5. Binary search inside the leaf page.
6. Return the matching entry.

### Complexity

Tree traversal:

**O(log N)**

Binary search inside one page:

**O(log page_size)**

Since page size is fixed, overall lookup is effectively **O(log N)**.

---

# 7. Why is B+ Tree Height So Small?

Each page stores many keys.

Approximate fanout:

| Tree Level | Rows Covered |
|------------|-------------|
| Root | ~500 child pages |
| Level 2 | ~250,000 pages |
| Level 3 | ~125 million pages |
| Level 4 | Tens of billions of rows |

Even billions of rows usually require only **3–4 page traversals**.

---

# 8. Clustered Index

Every InnoDB table has **one clustered index**.

If a Primary Key exists, it becomes the clustered index automatically.

### Important

The clustered index **is the table storage**.

There is **no separate heap table** storing duplicate rows.

Leaf pages contain the complete row.

Example:

| Primary Key | Complete Row |
|-------------|--------------|
| 101 | Alice, ₹250, Delivered |
| 105 | Bob, ₹500, Pending |
| 110 | Carol, ₹120, Packed |

Searching by primary key requires **one B+ Tree traversal**.

---

# 9. Why Does the Clustered Index Store Full Rows?

The table is physically ordered by the primary key.

Therefore the leaf pages are the actual table pages.

### Important Clarification

There is only **one persistent copy** of the row.

- SSD table file = Clustered B+ Tree pages.
- Buffer Pool = Cached copies in RAM.

No duplicate table exists elsewhere.

---

# 10. Secondary Index

A secondary index is another B+ Tree built on a non-primary column.

Example:

```sql
CREATE INDEX idx_email ON orders(email);
```

### Leaf Node Stores

| Secondary Key | Primary Key |
|---------------|-------------|
| alice@gmail.com | 101 |
| bob@gmail.com | 105 |
| carol@gmail.com | 110 |

**It stores the Primary Key value, not the full row.**

---

# 11. Why Doesn't Secondary Index Store the Full Row?

Suppose five indexes exist.

If every secondary index stored complete rows:

- Data would be duplicated many times.
- Every update would modify every index.

Instead, only the clustered index stores rows.

Secondary indexes only store `(secondary_key, primary_key)`.

This minimizes storage and update cost.

---

# 12. Why Doesn't Secondary Index Store a Disk Pointer?

Rows move between pages because of:

- Page splits.
- Page merges.
- Reorganization.

A physical pointer would become invalid.

Primary Key values never change logically.

Therefore secondary indexes remain valid even if rows move.

---

# 13. Two-Step Lookup Using Secondary Index

Example:

```sql
SELECT amount
FROM orders
WHERE email='bob@gmail.com';
```

### Step 1

Traverse secondary index.

Result:

| email | primary_key |
|-------|-------------|
| bob@gmail.com | 105 |

### Step 2

Traverse clustered index using `105`.

Return:

| order_id | amount |
|----------|--------|
| 105 | ₹500 |

This is called a **Back-to-Table Lookup** (Bookmark Lookup).

---

# 14. Covering Index

Suppose query:

```sql
SELECT order_id
FROM orders
WHERE email='bob@gmail.com';
```

Secondary index already contains:

| email | primary_key |
|-------|-------------|
| bob@gmail.com | 105 |

The database returns the result immediately.

No clustered index traversal is required.

This is called a **Covering Index**.

---

# 15. Composite Index

A composite index contains multiple columns.

Example:

```sql
CREATE INDEX idx_name_city
ON users(name, city);
```

The B+ Tree is sorted by:

1. `name`
2. `city`

Example leaf entries:

| Composite Key | Primary Key |
|---------------|-------------|
| (Alice, Delhi) | 101 |
| (Alice, Mumbai) | 205 |
| (Bob, Delhi) | 150 |

The entire composite key becomes the indexed value.

---

# 16. Duplicate Values in Secondary Index

Secondary indexes do **not** need unique values.

Example:

| name | primary_key |
|------|-------------|
| Alice | 101 |
| Alice | 205 |
| Alice | 310 |
| Bob | 150 |

All matching primary keys are stored.

A query on `name='Alice'` returns every matching primary key.

---

# 17. Are Duplicate Keys Stored Together?

Yes.

B+ Trees keep keys sorted.

All `"Alice"` entries appear in one continuous range across adjacent leaf pages.

This makes equality and range scans efficient.

---

# 18. Composite Index Ordering Rule

For index `(name, city)`:

Efficient:

```sql
WHERE name='Alice'
```

Efficient:

```sql
WHERE name='Alice'
AND city='Delhi'
```

Not efficient:

```sql
WHERE city='Delhi'
```

Because sorting starts with `name`.

This is called the **Leftmost Prefix Rule**.

---

# 19. Page Split During INSERT

Suppose a leaf page is full.

Current page:

| Keys |
|------|
| 101 |
| 105 |
| 110 |
| 115 |

Insert:

```sql
INSERT order_id = 108;
```

Temporary keys:

101, 105, 108, 110, 115

Overflow occurs.

### Page Split

Database creates a new page.

Left page:

101, 105

Right page:

108, 110, 115

The smallest key of the new right page (`108`) becomes the separator key in the parent page.

---

# 20. What Happens if the Parent Page is Full?

Page splits propagate upward.

If the root becomes full:

- Root splits.
- New root created.
- Tree height increases by one.

Height increases very rarely.

---

# 21. Why Inserts Remain O(log N)

Insert operation performs:

1. Traverse tree to leaf.
2. Insert into leaf page.
3. Split page if necessary.
4. Update parent pages.

At most one root-to-leaf path changes.

Therefore inserts remain **O(log N)**.

---

# 22. Why Sequential Primary Keys Are Faster

Auto Increment IDs:

101, 102, 103...

Every insert goes to the **rightmost leaf page**.

Benefits:

- Fewer page splits.
- Better cache locality.
- Faster inserts.

Random UUIDs:

- Inserts happen in random pages.
- More page splits.
- More SSD reads.
- Lower cache efficiency.

---

# 23. Clustered vs Secondary Index Summary

| Clustered Index | Secondary Index |
|-----------------|-----------------|
| Exactly one per table. | Many allowed. |
| Usually Primary Key. | Any indexed column. |
| Leaf stores complete rows. | Leaf stores secondary key + primary key. |
| One tree traversal for PK lookup. | Usually two tree traversals. |
| Defines physical row order. | Separate lookup structure. |

---

# 24. B+ Tree Summary

| Concept | Meaning |
|---------|---------|
| Root Node | Separator keys + child pointers. |
| Internal Node | Separator keys + child pointers. |
| Leaf Node (Clustered) | Primary key + complete row. |
| Leaf Node (Secondary) | Secondary key + primary key. |
| Page | Smallest storage unit (16 KB in InnoDB). |
| Fanout | Hundreds of child pointers per page. |
| Page Split | Splits full pages during insert. |
| Covering Index | Query answered entirely from the secondary index. |
| Composite Index | Index sorted by multiple columns. |

---

# Interview Takeaways

- InnoDB stores indexes as real B+ Trees on SSD.
- Every node is a database page, not a single key.
- Internal nodes store separator keys and child page pointers.
- Clustered index leaf pages contain the actual table rows.
- Secondary index leaf pages contain `(secondary_key, primary_key)`.
- Secondary-index lookups usually require two B+ Tree traversals.
- Composite indexes follow the **leftmost prefix rule**.
- Duplicate secondary-key values are stored together in sorted leaf pages.
- Inserts remain `O(log N)` because only one root-to-leaf path is modified, and page splits propagate upward only when necessary.