# Day 5: Transactions, Database Internals & Query Optimization
## ACID Properties, Isolation Levels, MVCC, Locks, Query Plans & Systems Architecture

---

# Part 1: Transactions & ACID Properties in Depth

### 1.1 Definition & Transaction Boundaries
A **Transaction** is a logical unit of database processing that includes one or more database access operations (read, insert, update, or delete).
Transactions are delineated by explicit control statements:
- `BEGIN TRANSACTION` / `START TRANSACTION`: Initiates a private transaction workspace.
- `COMMIT`: Persists all modifications permanently to disk and releases locks.
- `ROLLBACK`: Aborts the transaction, undoing all uncommitted modifications back to the initial state.
- `SAVEPOINT <name>` / `ROLLBACK TO <name>`: Establishes a checkpoint within a transaction for partial rollbacks.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;
SAVEPOINT transfer_deducted;

-- Attempt transfer to recipient
UPDATE accounts SET balance = balance + 500 WHERE account_id = 999;
-- If recipient account is invalid or locked:
ROLLBACK TO transfer_deducted;
-- Fallback action: Deposit into escrow instead
UPDATE escrow_vault SET balance = balance + 500 WHERE vault_id = 1;
COMMIT;
```

---

### 1.2 ACID Properties Under the Hood

#### 1. Atomicity ("All or Nothing")
- **Concept:** Either all operations in the transaction succeed and become permanent, or none do.
- **Engine Implementation:** Maintained via the **Undo Log** and **Write-Ahead Logging (WAL)**. If a transaction crashes or aborts mid-flight, the recovery manager scans the Undo Log backward to revert modified pages in the buffer cache to their original pre-image values.

#### 2. Consistency ("Preserving Invariants")
- **Concept:** A transaction transforms the database from one valid state to another valid state, satisfying all explicit schema constraints (PK, FK, CHECK, UNIQUE) and implicit domain invariants (e.g., total account balances across the bank must remain conserved).
- **Engine Implementation:** Enforced by constraint validation engines before commit and application-level business logic.

#### 3. Isolation ("Execution Independence")
- **Concept:** The concurrent execution of multiple transactions must yield the same database state as if they had executed serially one after the other.
- **Engine Implementation:** Enforced through **Concurrency Control Mechanisms**:
  - Pessimistic: Two-Phase Locking (2PL), Intent Locks.
  - Optimistic / Snapshot: Multi-Version Concurrency Control (MVCC) and validation checks.

#### 4. Durability ("Permanent Once Committed")
- **Concept:** Once a transaction commits, its effects survive any subsequent system crash, power failure, or operating system abort.
- **Engine Implementation:**
  - **Write-Ahead Logging (WAL):** Before any dirty data page is written from memory buffer cache to table disk storage, the corresponding log entry describing the change must be synchronously flushed to non-volatile disk storage via an `fsync()` syscall.
  - **Checkpoints:** Periodically flush all dirty buffer pages to disk and write a checkpoint marker to WAL, allowing old WAL segments to be truncated.

---

# Part 2: Concurrency Phenomena & Read Anomalies

When multiple transactions execute concurrently without strict isolation, four classic read phenomena can occur:

### 2.1 Dirty Read (Reading Uncommitted Data)
- **Mechanism:** Transaction $T_1$ updates a row without committing. Transaction $T_2$ reads that updated row. $T_1$ then rolls back. $T_2$ has acted on "phantom" data that never formally existed in the database!

### 2.2 Non-Repeatable Read (Fuzzy Read)
- **Mechanism:** Transaction $T_1$ reads a row. Transaction $T_2$ updates or deletes that exact row and **commits**. Transaction $T_1$ re-reads the row and finds that its values have changed or it has disappeared.

### 2.3 Phantom Read
- **Mechanism:** Transaction $T_1$ executes a range query (e.g., `WHERE salary > 100000`) and retrieves 5 rows. Transaction $T_2$ **inserts** a new employee with a $120,000 salary and **commits**. Transaction $T_1$ re-runs the identical range query and discovers 6 rows. (The new row is the "phantom").

### 2.4 Serialization Anomaly / Write Skew
- **Mechanism:** Two concurrent transactions read overlapping data sets, make decisions based on the observed state, and execute non-conflicting writes that collectively violate an application invariant.
- *Classic Example (Doctor on Call):* Hospital policy requires at least 1 doctor on call. Doctors Alice and Bob are currently on call. Both simultaneously request leave. Alice sees Bob is on call and checks out. Bob sees Alice is on call and checks out. Both commit. Now 0 doctors are on call!

---

# Part 3: ANSI SQL Isolation Levels

The ANSI/ISO SQL standard defines four transaction isolation levels based on which phenomena they permit:

| Isolation Level | Dirty Read Permitted? | Non-Repeatable Read Permitted? | Phantom Read Permitted? | Write Skew Possible? |
| :--- | :--- | :--- | :--- | :--- |
| **Read Uncommitted** | **YES** | **YES** | **YES** | **YES** |
| **Read Committed** | **NO** | **YES** | **YES** | **YES** |
| **Repeatable Read** | **NO** | **NO** | **YES** (Standard) / **NO** (PostgreSQL MVCC) | **YES** |
| **Serializable** | **NO** | **NO** | **NO** | **NO** |

> [!NOTE]
> **Engine Defaults:**
> - PostgreSQL, Oracle, SQL Server default to: **Read Committed**.
> - MySQL InnoDB defaults to: **Repeatable Read** (and uses Next-Key Locks to eliminate Phantoms even under Repeatable Read!).

---

# Part 4: Modern MVCC (Multi-Version Concurrency Control)

Modern databases (PostgreSQL, MySQL InnoDB, Oracle, Snowflake) do not rely on heavy read locks to achieve isolation. Instead, they use **MVCC**: *"Readers never block Writers, and Writers never block Readers."*

### 4.1 How MVCC Operates
Under MVCC, an `UPDATE` does not overwrite the row in place. Instead, it:
1. Writes a brand new version of the row tuple.
2. Marks the old version as expired/deleted.

### 4.2 PostgreSQL Implementation: `xmin` and `xmax`
Every physical row (tuple) in PostgreSQL contains hidden system attributes:
- `xmin`: The transaction ID ($XID$) that inserted this tuple.
- `xmax`: The transaction ID ($XID$) that deleted or superseded this tuple (or 0 if still current).

When Transaction $T$ runs a query with snapshot ID $S$:
- It can only see tuples where `xmin < S` and committed, AND where `xmax > S` or uncommitted.
- **Vacuuming (`VACUUM`):** An asynchronous background process that reclaims dead tuple space once all active transactions that could possibly view those older row versions have completed.

---

# Part 5: Locking Mechanisms & Deadlocks

### 5.1 The Lock Hierarchy
When transactions do lock data, they use a hierarchical locking protocol:

```
Database Level
   └── Table Level (Intent Locks: IS, IX)
         └── Page / Extent Level
               └── Row Level (Shared: S, Exclusive: X)
```

1. **Shared Lock (S-Lock):** Acquired for read operations (`SELECT ... FOR SHARE`). Multiple transactions can hold S-locks on the same resource concurrently.
2. **Exclusive Lock (X-Lock):** Acquired for write operations (`INSERT`, `UPDATE`, `DELETE`, or `SELECT ... FOR UPDATE`). Only one transaction can hold an X-lock; blocks all other S and X locks.
3. **Intent Locks (IS, IX):** Placed at the table level to signal that a transaction intends to lock individual rows lower down the hierarchy. Prevents another transaction from acquiring an exclusive table-level lock while row-level modifications are underway.

---

### 5.2 Deadlocks: Mechanics, Detection & Prevention

#### Definition:
A **Deadlock** occurs when two or more transactions are permanently blocked because each holds a lock on a resource that the other requires to proceed.

```
Transaction 1: Holds Lock on Account A, Requests Lock on Account B
Transaction 2: Holds Lock on Account B, Requests Lock on Account A
              ===> DEADLOCK (Mutual circular dependency)
```

#### Deadlock Detection: The Wait-For Graph (WFG)
The database engine maintains a directed graph where nodes represent transactions and edges represent lock dependencies ($T_1 \rightarrow T_2$ means $T_1$ is waiting for $T_2$).
- If a cycle is detected ($T_1 \rightarrow T_2 \rightarrow T_1$), a deadlock exists!
- **Victim Selection:** The engine automatically terminates (rolls back) the transaction with the lowest cost (e.g., fewest modifications made) and throws a deadlock exception to the client:
  `ERROR: 40P01: deadlock detected`.

#### Deadlock Prevention Best Practices:
1. **Deterministic Lock Ordering:** Always access tables and rows in the exact same sequential order across all application code paths (e.g., sort account IDs before acquiring locks: `WHERE account_id IN (10, 20) ORDER BY account_id`).
2. **Keep Transactions Short:** Never perform external network calls, file I/O, or user interaction while holding database transactions open.
3. **Use Lock Timeouts:** Set `SET lock_timeout = '2s'` to fail fast rather than stalling connection pools.

---

# Part 6: Query Execution Plans & Optimization

### 6.1 `EXPLAIN` vs. `EXPLAIN ANALYZE`
- `EXPLAIN query`: Asks the Cost-Based Optimizer (CBO) to display the **estimated execution plan** based on table statistics without running the query.
- `EXPLAIN ANALYZE query`: **Actually executes the query**, displaying both the estimated cost and the **realized execution metrics** (actual startup time, execution time, row counts, and loops).

---

### 6.2 Key Query Plan Operators

#### 1. Data Access Operators:
- **Sequential Scan (Seq Scan / Table Scan):** Scans every physical disk block of the table sequentially. Optimal when reading $> 20-30\%$ of a table's data or for very small tables.
- **Index Scan:** Traverses the B-tree to find matching row pointers, then fetches each corresponding page from the table storage heap.
- **Index Only Scan:** Satisfies the entire query directly from the leaf nodes of an index (Covering Index) without touching the base table heap.
- **Bitmap Index Scan + Bitmap Heap Scan:** Scans the index to build a memory bitmap of matching disk pages, sorts the page numbers physically, and visits each table disk page exactly once. Ideal for queries combining multiple indexes (`AND`/`OR`).

#### 2. Join Operators:
- **Nested Loop:** Iterates through outer table rows and searches the inner table. Fastest when outer table is small and inner table has a selective B-tree index.
- **Hash Join:** Hashes the smaller table into memory buffer, then streams the larger table to probe matches. Excels for large, unindexed equijoins.
- **Merge Join:** Both inputs are sorted by the join key and zipped together. Highly efficient if inputs are already sorted by clustered indexes.

---

# Part 7: Systems Architecture: SQL vs. NoSQL & OLTP vs. OLAP

### 7.1 CAP Theorem & Relational Consistency
The **CAP Theorem** dictates that a distributed data store can guarantee at most two of three properties simultaneously:
- **Consistency (C):** Every read receives the most recent write or an error.
- **Availability (A):** Every non-failing node returns a non-error response for every request.
- **Partition Tolerance (P):** The system continues to operate despite arbitrary network message loss or network splits.

*Traditional RDBMS (PostgreSQL, MySQL, Oracle):* Prioritize **CA** on single nodes and **CP** in distributed clustering.  
*NoSQL Systems (Cassandra, DynamoDB, MongoDB):* Often choose **AP** (Eventual Consistency) or configurable consistency.

---

### 7.2 OLTP vs. OLAP Internal Storage Engine Architecture

```
Row-Oriented Storage (OLTP - PostgreSQL, MySQL)
Disk Page 1: [ID: 1, Name: Alice, Age: 28, Balance: 5000] [ID: 2, Name: Bob, Age: 34, Balance: 7200]
Disk Page 2: [ID: 3, Name: Charlie, Age: 45, Balance: 3100] ...

Column-Oriented Storage (OLAP - Snowflake, ClickHouse, Redshift)
Disk Column Page 1 (IDs):      [1, 2, 3, 4, 5, 6, 7, 8, ...]
Disk Column Page 2 (Names):    [Alice, Bob, Charlie, ...]
Disk Column Page 3 (Balances): [5000, 7200, 3100, 8900, ...]
```

#### Why Columnar Storage Dominates Analytics:
1. **I/O Elimination:** A query computing `SELECT AVG(Balance) FROM users` only reads Disk Column Page 3, skipping names and addresses entirely (reducing disk I/O by $90\%$).
2. **Extreme Compression:** Because a single column stores identical data types, compression algorithms (Run-Length Encoding, Dictionary, Delta encoding) achieve $10\times$ higher compression ratios than row-based storage.

---

# Part 8: The 7-Step Production Troubleshooting Framework: "Why is a Query Slow?"

When an interviewer asks: *"A critical production query has slowed down dramatically. How do you investigate and fix it?"*, follow this structured methodology:

### Step 1: Capture the Ground Truth & Execution Metrics
- Inspect query logs (`pg_stat_activity`, slow query log).
- Run `EXPLAIN (ANALYZE, BUFFERS)` to observe actual time spent, row counts vs. estimated rows, and buffer cache hits vs. disk reads.

### Step 2: Compare Estimated vs. Actual Cardinality
- Look for severe discrepancies (e.g., `estimated rows = 1`, `actual rows = 500,000`).
- **Cause:** Stale table statistics. The cost-based optimizer is making bad algorithm choices (e.g., choosing a Nested Loop instead of a Hash Join).
- **Fix:** Run `ANALYZE table_name;` to update planner statistics.

### Step 3: Identify the Access Path Bottlenecks
- Look for unexpected `Seq Scan` on multi-million row tables.
- Check if existing indexes are being ignored due to functions wrapping indexed columns (e.g., `WHERE LOWER(email) = '...'` or `WHERE EXTRACT(YEAR FROM created_at) = 2024`).
- **Fix:** Create functional indexes (`CREATE INDEX ON users (LOWER(email))`) or rewrite predicates to be **sargable** (`WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'`).

### Step 4: Detect Expensive Sorts & TempDB Spills
- Check execution plan for `Sort Method: external merge Disk`.
- **Cause:** Sorting requires more memory than `work_mem` permits, spilling intermediate rows to slow disk.
- **Fix:** Tune `work_mem` or create an index that matches the `ORDER BY` clause to provide pre-sorted data.

### Step 5: Check for Lock Contention & Blocking Transactions
- Query active locks (`pg_locks`, `sys.dm_tran_locks`).
- Determine if the slow query is waiting on row or table locks held by an uncommitted transaction or long-running backup.

### Step 6: Review Join Explosion & Implicit Type Conversions
- Verify that join keys share exact identical data types. An implicit cast (`VARCHAR` to `INT`) disables B-tree index seeks.
- Check if non-unique joins are causing row cardinality explosion.

### Step 7: Architecture & Scaling Remediations
- If indexing is exhausted: Partition large tables by date/range (`TABLE PARTITIONING`), evaluate Materialized Views, read replicas, or offload heavy aggregations to an OLAP engine.

---

# Part 9: Structured Interview Question Bank

### Q1: What is Write-Ahead Logging (WAL), and what happens if the database crashes before dirty buffer pages are written to the table data files?
**Answer:**
Write-Ahead Logging dictates that any data modification must first be appended to an immutable, sequential disk log before the actual data pages in memory are written to disk. If the database crashes, dirty in-memory pages are lost. Upon reboot, the recovery manager reads the WAL from the last checkpoint:
1. **Redo Phase:** Replays all committed transactions in the WAL to update data files.
2. **Undo Phase:** Rolls back all uncommitted transactions present in the WAL when the crash occurred, restoring the database to a consistent state.

### Q2: What is the difference between Pessimistic Locking and Optimistic Concurrency Control (OCC)?
**Answer:**
- **Pessimistic Locking:** Assumes conflicts will occur. Locks resources upfront before modification (`SELECT ... FOR UPDATE`). Other transactions must wait. Best for high-contention, low-latency transaction systems.
- **Optimistic Concurrency Control:** Assumes conflicts are rare. Reads data without locking, records a version number or timestamp, and checks during the commit phase if the version changed. If modified, the transaction rolls back and retries. Best for read-heavy workloads with low write collisions.

### Q3: Explain "Write Skew" and why Repeatable Read isolation fails to prevent it.
**Answer:**
Write Skew occurs when two concurrent transactions read disjoint or overlapping data, check a shared invariant, and modify distinct records that together violate the invariant. Because Repeatable Read ensures that repeated reads of the *same* row return identical values, but does not prevent concurrent transactions from modifying *different* rows, both transactions pass their constraint checks and commit. Preventing Write Skew requires either explicit locking (`SELECT FOR UPDATE`) or **Serializable** isolation.

### Q4: How does a B-tree index differ from a Hash index in a relational database?
**Answer:**
- **B-Tree Index:** Self-balancing search tree supporting point lookups ($O(\log N)$), range scans (`BETWEEN`, `<`, `>`), prefix searches (`LIKE 'abc%'`), and pre-sorted ordering (`ORDER BY`).
- **Hash Index:** Uses an in-memory hash table supporting strictly exact equality lookups ($O(1)$). Does **not** support range queries, partial prefix searches, or sorted retrieval.

---

# Part 10: End-to-End Enterprise Case Study: High-Concurrency Wallet Double-Spend

### Problem Statement
A mobile payments app allows users to transfer balance to friends. During flash sales, users triggered concurrent transfer requests from multiple devices simultaneously, resulting in a **double-spend exploit**: users with a $100 balance successfully transferred $100 twice, driving their account to a -$100 negative balance.

### Root Cause Analysis
The original application logic under default `READ COMMITTED` isolation executed:
```sql
-- Step 1: Read current balance (Device A & Device B read simultaneously)
SELECT balance FROM wallets WHERE user_id = 45; -- Both see $100

-- Application checks: If balance >= 100, proceed...

-- Step 2: Deduct balance (Both execute update)
UPDATE wallets SET balance = balance - 100 WHERE user_id = 45;
```
Because both transactions read $100 before either committed, both updates succeeded, resulting in $100 - 100 - 100 = -100$.

### Production Remediation: 3 Layer Defense
1. **Pessimistic Row-Level Locking:**
   ```sql
   BEGIN;
   -- Locks the wallet row exclusively; Device B blocks until Device A commits
   SELECT balance FROM wallets WHERE user_id = 45 FOR UPDATE;
   -- Verify balance >= 100 in app
   UPDATE wallets SET balance = balance - 100 WHERE user_id = 45;
   COMMIT;
   ```
2. **Atomic Single-Statement Guard (Conditional Update):**
   ```sql
   UPDATE wallets 
   SET balance = balance - 100 
   WHERE user_id = 45 AND balance >= 100;
   -- Check rows_affected in application: If 0, reject transfer!
   ```
3. **Database Schema Invariant:**
   ```sql
   ALTER TABLE wallets ADD CONSTRAINT chk_balance_non_negative CHECK (balance >= 0.00);
   ```

---

# Part 11: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│               TRANSACTIONS & DATABASE INTERNALS CHEAT SHEET            │
├────────────────────────────────────────────────────────────────────────┤
│ 1. ACID: Atomicity (Undo Log), Consistency (Constraints),              │
│          Isolation (Locks/MVCC), Durability (WAL + fsync).             │
│ 2. WAL: Changes written sequentially to disk log BEFORE dirty pages.   │
├────────────────────────────────────────────────────────────────────────┤
│                       CONCURRENCY ANOMALIES                            │
│  - Dirty Read         : Reading uncommitted changes that roll back.    │
│  - Non-Repeatable Read: Same row value changes between reads.          │
│  - Phantom Read       : New rows appear in range query between reads.  │
│  - Write Skew         : Invariant violated across distinct rows.       │
├────────────────────────────────────────────────────────────────────────┤
│                        ISOLATION LEVELS (ANSI)                         │
│  - Read Uncommitted : Allows all anomalies.                            │
│  - Read Committed   : Blocks Dirty Reads. (Postgres default).          │
│  - Repeatable Read  : Blocks Dirty & Fuzzy Reads. (MySQL default).     │
│  - Serializable     : Completely isolates concurrent execution.        │
├────────────────────────────────────────────────────────────────────────┤
│                              LOCKING                                   │
│  - Shared (S)       : Read lock (Multiple readers permitted).          │
│  - Exclusive (X)    : Write lock (Strictly 1 holder; blocks all).     │
│  - Deadlock         : Circular dependency on Wait-For Graph.           │
│                       Fix via deterministic lock ordering & timeouts.  │
├────────────────────────────────────────────────────────────────────────┤
│                           QUERY OPTIMIZATION                           │
│  - EXPLAIN ANALYZE  : Actual execution times and realized rows.        │
│  - Seq Scan         : Full table read (Bad for selective point lookups)│
│  - Index Scan       : B-Tree seek + Heap tuple lookup.                 │
│  - Index Only Scan  : Satisfied 100% from index leaf (Covering).       │
│  - Sargable Queries : Avoid functions on indexed columns!              │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 12: 100-Mark Self-Assessment Examination

### Section A: Conceptual & Architectural (25 Marks — 5 marks each)

**Q1:** Which ACID property is directly guaranteed by Write-Ahead Logging (WAL)?  
(a) Atomicity and Durability  
(b) Consistency only  
(c) Isolation only  
(d) Normalization  

**Q2:** Under which ANSI SQL isolation level is a "Dirty Read" permitted?  
(a) Serializable  
(b) Repeatable Read  
(c) Read Committed  
(d) Read Uncommitted  

**Q3:** In PostgreSQL MVCC, what does the hidden column `xmin` represent?  
(a) The minimum allowed value of the primary key.  
(b) The Transaction ID ($XID$) that inserted the tuple.  
(c) The timestamp when the row was updated.  
(d) The size of the tuple in bytes.  

**Q4:** What join physical algorithm is typically chosen by the optimizer when joining two large tables on an equality predicate when neither table has an index on the join key?  
(a) Nested Loop Join  
(b) Hash Join  
(c) Cartesian Join  
(d) B-Tree Merge Join  

**Q5:** Why does `SELECT * FROM sales WHERE EXTRACT(YEAR FROM sale_date) = 2024;` typically fail to use an index on `sale_date`?  
(a) The query optimizer cannot evaluate date types.  
(b) Wrapping an indexed column in a function makes the predicate non-sargable, forcing a sequential scan.  
(c) Indexes cannot store year values.  
(d) Dates must always be formatted as strings.  

---

### Section B: Concurrency Analysis & Anomaly Identification (25 Marks)

**Q6 (10 Marks):** Consider two transactions executing concurrently:
- $T_1$: `BEGIN; SELECT balance FROM accounts WHERE id = 1;` (Returns $1000)
- $T_2$: `BEGIN; UPDATE accounts SET balance = 1500 WHERE id = 1; COMMIT;`
- $T_1$: `SELECT balance FROM accounts WHERE id = 1;` (Returns $1500)
- $T_1$: `COMMIT;`
1. Name the exact concurrency anomaly demonstrated above. (5 marks)
2. Which isolation level(s) would prevent this anomaly from occurring? (5 marks)

**Q7 (15 Marks):** Explain the deadlock condition that occurs between the two transactions below. Construct the Wait-For Graph and explain how the database resolves it:
- $T_1$: Updates row in `products` where `id = 10`.
- $T_2$: Updates row in `inventory` where `id = 50`.
- $T_1$: Attempts to update row in `inventory` where `id = 50`.
- $T_2$: Attempts to update row in `products` where `id = 10`.

---

### Section C: Query Plan Diagnostics & Performance Tuning (30 Marks)

**Q8 (30 Marks):** An e-commerce platform has 25 million records in an `orders` table. A customer dashboard query runs:
```sql
SELECT order_id, order_date, total_amount, order_status
FROM orders
WHERE customer_id = 91823 AND order_status = 'Shipped'
ORDER BY order_date DESC;
```
Running `EXPLAIN ANALYZE` outputs:
```
Seq Scan on orders  (cost=0.00..845120.00 rows=12 width=32) (actual time=4210.12..4210.15 rows=8 loops=1)
  Filter: ((customer_id = 91823) AND ((order_status)::text = 'Shipped'::text))
  Rows Removed by Filter: 24999992
Planning Time: 0.12 ms
Execution Time: 4210.22 ms
```
1. Diagnose the exact performance bottleneck in the query plan. (10 marks)
2. Write the DDL statement for the single most optimal composite index to reduce execution time to under 2 milliseconds. (10 marks)
3. Explain why this index completely eliminates the Sequential Scan, the Key Lookup, and the Sort operator. (10 marks)

---

### Section D: Architectural Systems Design (20 Marks)

**Q9 (20 Marks):** Compare an OLTP database (e.g., PostgreSQL) with an OLAP data warehouse (e.g., Snowflake) for an enterprise analytics workload aggregating 5 billion rows. Explain why the columnar architecture outperforms row-based relational storage by orders of magnitude for aggregate queries.

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(a)** Atomicity (via Undo rollback logging) and Durability (via Redo WAL logging).
- **Q2:** **(d)** Read Uncommitted.
- **Q3:** **(b)** The Transaction ID ($XID$) that inserted the tuple.
- **Q4:** **(b)** Hash Join.
- **Q5:** **(b)** Wrapping the column in `EXTRACT()` makes the predicate non-sargable.

### Section B Solutions
**Q6 Solution:**
1. **Non-Repeatable Read (Fuzzy Read):** The exact same row returned two differing values within the same transaction scope after an external commit.
2. Prevented by: **Repeatable Read** and **Serializable**.

**Q7 Solution:**
- $T_1$ holds exclusive lock on `products(10)` and requests lock on `inventory(50)`.
- $T_2$ holds exclusive lock on `inventory(50)` and requests lock on `products(10)`.
- **Wait-For Graph:** $T_1 \rightarrow T_2$ and $T_2 \rightarrow T_1$ (Directed cycle detected).
- **Resolution:** The deadlock detection engine runs a cycle detection algorithm, designates one transaction as the victim (e.g., $T_2$), issues an internal `ROLLBACK`, releases its locks, and sends error `40P01` to $T_2$, allowing $T_1$ to complete.

### Section C Solutions
**Q8 Solution:**
1. **Bottleneck:** The engine performs a full `Seq Scan`, scanning all 25 million disk rows and discarding $24,999,992$ rows, taking over 4.2 seconds.
2. **Optimal Index DDL:**
   ```sql
   CREATE INDEX idx_orders_cust_status_date_covering 
   ON orders (customer_id, order_status, order_date DESC) 
   INCLUDE (order_id, total_amount);
   ```
3. **Execution Elimination:**
   - **Seq Scan eliminated:** Engine performs an $O(\log N)$ B-tree index seek matching `customer_id` and `order_status`.
   - **Sort eliminated:** Leaf nodes are already ordered physically by `order_date DESC`.
   - **Key Lookup eliminated:** `order_id` and `total_amount` are stored directly in the leaf pages via `INCLUDE` (Covering Index), resulting in an **Index Only Scan**.

### Section D Solutions
**Q9 Solution:**
In row-oriented OLTP storage, all attributes of a row are packed together on the same disk page. Computing `SUM(amount)` over 5 billion rows requires loading every single column (IDs, text strings, timestamps) into memory, causing massive I/O saturation.  
In columnar OLAP storage (Snowflake), each column is stored in isolated, compressed micro-partitions. The engine reads **only** the `amount` column files, skipping all other columns entirely. Furthermore, SIMD vectorized CPU execution scans compressed integer/decimal arrays at hardware line speeds, resulting in 100x to 1000x faster execution.
