## Postgres

PostgreSQL uses a process-based architecture where it spawns a completely separate operating system process (using fork) for each client connection, managed by a main postgres postmaster supervisor process.

# What PostgreSQL Does for Each Connection

• Spawns a Dedicated Process: When a client connects, the postmaster process forks a new, independent server process dedicated solely to handling that specific connection's lifecycle, queries, and transactions.

• Allocates Isolated Memory: Each backend process gets its own private memory space for parsing, planning, and executing queries (controlled by settings like work_mem), preventing one connection from stepping on another's memory.

• Communicates via IPC: Connections talk back and forth with shared memory components (such as the Shared Buffer Cache and Write-Ahead Log buffers) using Inter-Process Communication (IPC) mechanisms.

• Provides High Fault Isolation: Because each connection runs in its own OS process, a crash or memory leak inside a single rogue connection process typically does not crash the entire database engine or affect other client sessions.


# MySQL and Postgres Architectural Comaprision:

| Feature / Behavior | PostgreSQL (Process-Based) | MySQL (Thread-Based) |
| :--- | :--- | :--- |
| **Concurrency Model** | Spawns a new OS **process** per connection. | Spawns a lightweight **thread** within a single shared OS process. |
| **Memory Footprint** | Higher memory usage per connection because each process carries its own overhead. | Lower memory footprint because threads share the same process memory space. |
| **Connection Scaling** | High connection counts can exhaust system RAM or require heavy CPU context switching; **connection pooling** (like PgBouncer) is practically mandatory in production. | Handles thousands of concurrent threads more cheaply out-of-the-box, making connection pooling optional for many setups. |
| **Fault Tolerance** | If a connection process crashes, the main engine stays up and isolates the failure. | If a thread encounters a severe internal memory/state corruption, it can destabilize the shared process. |
| **Data Types & Extensibility** | Object-relational; native support for complex data types (JSONB, arrays, custom types, GIN/GiST indexes). | Purely relational; simpler data type support and more basic index options (B-Tree). |
| **Write/Read Workloads** | Excels at heavy concurrent writes and complex analytical queries via MVCC. | Traditionally optimized for high-speed, simple reads and high-throughput point lookups. |
