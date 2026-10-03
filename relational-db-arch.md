# Postgres

PostgreSQL uses a process-based architecture where it spawns a completely separate operating system process (using fork) for each client connection, managed by a main postgres postmaster supervisor process.

# What PostgreSQL Does for Each Connection

• Spawns a Dedicated Process: When a client connects, the postmaster process forks a new, independent server process dedicated solely to handling that specific connection's lifecycle, queries, and transactions.

• Allocates Isolated Memory: Each backend process gets its own private memory space for parsing, planning, and executing queries (controlled by settings like work_mem), preventing one connection from stepping on another's memory.

• Communicates via IPC: Connections talk back and forth with shared memory components (such as the Shared Buffer Cache and Write-Ahead Log buffers) using Inter-Process Communication (IPC) mechanisms.

• Provides High Fault Isolation: Because each connection runs in its own OS process, a crash or memory leak inside a single rogue connection process typically does not crash the entire database engine or affect other client sessions.

• Postgres also supports something that is 


# MySQL and Postgres Architectural Comaprision:

| Feature / Behavior | PostgreSQL (Process-Based) | MySQL (Thread-Based) |
| :--- | :--- | :--- |
| **Concurrency Model** | Spawns a new OS **process** per connection. | Spawns a lightweight **thread** within a single shared OS process. |
| **Memory Footprint** | Higher memory usage per connection because each process carries its own overhead. | Lower memory footprint because threads share the same process memory space. |
| **Connection Scaling** | High connection counts can exhaust system RAM or require heavy CPU context switching; **connection pooling** (like PgBouncer) is practically mandatory in production. | Handles thousands of concurrent threads more cheaply out-of-the-box, making connection pooling optional for many setups. |
| **Fault Tolerance** | If a connection process crashes, the main engine stays up and isolates the failure. | If a thread encounters a severe internal memory/state corruption, it can destabilize the shared process. |
| **Data Types & Extensibility** | Object-relational; native support for complex data types (JSONB, arrays, custom types, GIN/GiST indexes). | Purely relational; simpler data type support and more basic index options (B-Tree). |
| **Write/Read Workloads** | Excels at heavy concurrent writes and complex analytical queries via MVCC. | Traditionally optimized for high-speed, simple reads and high-throughput point lookups. |


# Partial Indexing on Postgres:

PostgreSQL partial indexing builds an index on a subset of a table defined by a conditional WHERE expression.

## Primary Use Cases

• Excluding Irrelevant or Null Values: Index only active records or non-null fields while skipping large blocks of archived, deleted, or default data (e.g., indexing email only where deleted_at IS NULL).

• Accelerating Job or Queue Processing: Speed up access to pending tasks in a background queue by indexing only rows where status = 'pending'.

• Enforcing Partial Uniqueness: Guarantee a unique constraint on a specific subset of rows (such as ensuring a unique email address only among active accounts, allowing multiple deactivated accounts to share an email).

• Reducing Storage and RAM Footprint: Shrink index file sizes drastically—sometimes from tens of megabytes down to a few megabytes—allowing the entire index to reside smoothly inside memory (RAM) buffers.

• Improving Write (INSERT/UPDATE/DELETE) Performance: Lower the overhead of maintaining indexes because PostgreSQL only updates the index when modified data matches the partial predicate.


## Key Considerations

• Query Match Requirement: The query planner will only use a partial index if your query's WHERE clause mathematically implies or explicitly matches the index's condition.

• Predictable Workloads: They work best when application access patterns are consistent; if queries frequently change conditions or span unindexed subsets, performance can fall back to full table scans.


# Real World Use Case:

## The Problem Instagram Faced

When a user typed a character prefix into the Instagram search bar (e.g., typing "snow"), the application needed to instantly recommend the top relevant hashtags matching that prefix, ordered by popularity (the number of photos/videos tagged with it).

Initially, they ran a query similar to this:

sql
SELECT * FROM hashtags 
WHERE name LIKE 'snow%' 
ORDER BY media_count DESC 
LIMIT 10;


The Bottleneck: Hashtags follow a classic long-tail distribution. While a tiny percentage of hashtags are massively popular (#snow, #snowboarding), millions of hashtags are completely obscure, used only once or twice by a single user.

A standard compound index on (name, media_count) would index every single hashtag ever created. When a user searched for "snow%", PostgreSQL still had to look up and sort tens of thousands of obscure rows in memory just to extract the top 10 results—wasting massive amounts of CPU and disk I/O under high load.

How Instagram Solved It (The Implementation)

The engineering team realized they didn't need to index unpopular hashtags because a user looking for auto-suggestions is almost always looking for established, active tags.

They created a composite partial index targeting only popular hashtags:

sql
CREATE INDEX CONCURRENTLY idx_popular_hashtags 
ON hashtags (name, media_count DESC) 
WHERE media_count > 100;
Use code with caution.
To take advantage of this index, they adjusted their application query to explicitly include that threshold:
sql


SELECT * FROM hashtags 
WHERE name LIKE 'snow%' AND media_count > 100
ORDER BY media_count DESC 
LIMIT 10;


## The Core Benefits Instagram Achieved

By isolating the index to a specific subset (WHERE media_count > 100), they unlocked huge performance gains:

• Dramatic Latency Reduction: In testing, this change slashed their query response times from roughly 200 milliseconds down to just 3 milliseconds!

• Reduced Sort Overhead: Instead of pulling and sorting thousands of rows matching a text prefix, the query planner only had to look at a highly reduced subset. For example, in a benchmark query, the rows to sort dropped from 15,455 down to just 169 rows.

• Massive RAM and Storage Savings: Because millions of single-use or low-count hashtags were entirely excluded from the index tree, the size of the index shrank dramatically. This allowed the entire active index to easily fit into the database's RAM buffers, eliminating slow disk lookups.

• Faster Write Performance: Whenever a user posted a brand new, obscure hashtag, PostgreSQL didn't have to spend write cycles updating this specific search index, because the new row's media_count was less than 100.