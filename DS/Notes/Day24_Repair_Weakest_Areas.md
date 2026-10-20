# Day 24: Targeted Gap Remediation & Weak Area Repair
## Deep Diagnostic Framework, Advanced Problem-Solving Drills & High-Difficulty Reinforcement

---

# Part 1: The Diagnostic Triad: Identifying Your Weakest Areas

In technical interview preparation, rereading familiar notes gives a false sense of mastery. Real improvement requires **active problem-solving on your exact points of failure**:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   THE THREE TYPICAL FAILURE ARCHETYPES                 │
├────────────────────────────────────────────────────────────────────────┤
│ ARCHETYPE A: SQL WINDOW FRAMING & COMPLEX STREAKS                      │
│   - Symptoms: Struggles when gaps are multi-day, or when streaks       │
│     reset based on numerical thresholds rather than consecutive dates. │
│   - Remedy: Master the running conditional sum & difference method.    │
├────────────────────────────────────────────────────────────────────────┤
│ ARCHETYPE B: DISTRIBUTED SYSTEMS, CONCURRENCY & LOCKING                │
│   - Symptoms: Confusion between Read Committed vs. Repeatable Read     │
│     anomalies; inability to trace deadlocks or query plan spills.      │
│   - Remedy: Draw physical transaction timelines & Wait-For graphs.     │
├────────────────────────────────────────────────────────────────────────┤
│ ARCHETYPE C: MEMORY SCALING & PRODUCTION CRASHES                       │
│   - Symptoms: Writing code that loads entire datasets into RAM;        │
│     unfamiliarity with chunking, generators, and INT8 quantization.    │
│   - Remedy: Enforce strict O(1) memory streaming architectures.        │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 2: Deep Repair Area 1: Advanced SQL Streaks & Custom Gaps

---

### Drill 1: Streaks Resetting on Thresholds (Non-Date Gaps)
**Problem Statement:**  
Given a sensor telemetry table `server_metrics (server_id INT, log_time TIMESTAMP, cpu_utilization FLOAT)`.  
Find all incidents where a server's CPU utilization exceeded **$90.0\%$ for at least 3 consecutive readings**, returning the `server_id`, the start time of the incident, the end time, and the count of readings.

```sql
WITH AlertFlags AS (
    SELECT 
        server_id,
        log_time,
        cpu_utilization,
        CASE WHEN cpu_utilization > 90.0 THEN 1 ELSE 0 END AS is_critical
    FROM server_metrics
),
IslandGroups AS (
    SELECT 
        server_id,
        log_time,
        cpu_utilization,
        is_critical,
        -- Subtracting row number of critical items from overall row number forms island groups!
        ROW_NUMBER() OVER (PARTITION BY server_id ORDER BY log_time) -
        ROW_NUMBER() OVER (PARTITION BY server_id, is_critical ORDER BY log_time) AS island_id
    FROM AlertFlags
)
SELECT 
    server_id,
    MIN(log_time) AS incident_start,
    MAX(log_time) AS incident_end,
    COUNT(*) AS consecutive_critical_readings
FROM IslandGroups
WHERE is_critical = 1
GROUP BY server_id, island_id
HAVING COUNT(*) >= 3
ORDER BY server_id, incident_start;
```

---

### Drill 2: Non-Overlapping User Sessionization in Pure SQL
**Problem Statement:**  
Given web clickstream events `web_events (event_id, user_id, event_time)`. A user session expires after **15 minutes of inactivity**. Assign a unique `session_number` to each user's events using pure ANSI SQL without procedural loops.

```sql
WITH LaggedEvents AS (
    SELECT 
        user_id,
        event_time,
        LAG(event_time) OVER (
            PARTITION BY user_id 
            ORDER BY event_time
        ) AS prev_event_time
    FROM web_events
),
SessionStarters AS (
    SELECT 
        user_id,
        event_time,
        CASE 
            WHEN prev_event_time IS NULL THEN 1
            WHEN event_time > prev_event_time + INTERVAL '15 minute' THEN 1
            ELSE 0 
        END AS is_new_session
    FROM LaggedEvents
)
SELECT 
    user_id,
    event_time,
    -- Running sum of session starters produces the session number!
    SUM(is_new_session) OVER (
        PARTITION BY user_id 
        ORDER BY event_time 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS session_number
FROM SessionStarters
ORDER BY user_id, event_time;
```

---

# Part 3: Deep Repair Area 2: Concurrency, Memory & Production Failures

---

### Drill 3: Diagnosing & Eliminating a Financial Double-Spend Deadlock
**Problem Statement:**  
Two transactions attempt to execute atomic transfers between Accounts A and B simultaneously:
- Txn 1: Transfers \$100 from Account 1 to Account 2.
- Txn 2: Transfers \$50 from Account 2 to Account 1.
In production under high load, this causes recurring `deadlock detected` crashes.  
Explain why the deadlock occurs and write the production SQL stored procedure that permanently eliminates the deadlock.

```
DEADLOCK MECHANISM:
Txn 1 locks Account 1 (Exclusive Lock).
Txn 2 locks Account 2 (Exclusive Lock).
Txn 1 requests lock on Account 2 (BLOCKED by Txn 2).
Txn 2 requests lock on Account 1 (BLOCKED by Txn 1).
===> CIRCULAR WAIT: DEADLOCK!
```

**Production Resolution: Deterministic Lock Ordering**
```sql
CREATE OR REPLACE PROCEDURE transfer_balance(
    sender_id INT,
    receiver_id INT,
    transfer_amount DECIMAL
)
LANGUAGE plpgsql
AS $$
DECLARE
    first_id INT;
    second_id INT;
BEGIN
    -- ALWAYS lock accounts in the exact same numerical sequence!
    IF sender_id < receiver_id THEN
        first_id := sender_id;
        second_id := receiver_id;
    ELSE
        first_id := receiver_id;
        second_id := sender_id;
    END IF;

    -- Acquire locks in strict ascending order
    PERFORM 1 FROM accounts WHERE account_id = first_id FOR UPDATE;
    PERFORM 1 FROM accounts WHERE account_id = second_id FOR UPDATE;

    -- Execute transfer safely without risk of circular wait
    UPDATE accounts SET balance = balance - transfer_amount WHERE account_id = sender_id;
    UPDATE accounts SET balance = balance + transfer_amount WHERE account_id = receiver_id;
    
    COMMIT;
END;
$$;
```

---

### Drill 4: Streaming Multi-Gigabyte CSV Parsing in Python with Zero Memory Overhead
**Problem Statement:**  
A batch data pipeline receives a 25 GB CSV log file containing 100 million lines. The worker has strictly 2 GB of RAM. Write an idiomatic Python generator that parses, validates, and batches these records without crashing the process.

```python
import csv
from typing import Generator

def stream_large_log_file(file_path: str, batch_size: int = 10_000) -> Generator[list[dict], None, None]:
    """
    Streams and batches a multi-gigabyte file in O(1) memory.
    Reads line-by-line via iterator protocol, yielding memory-isolated chunks.
    """
    batch = []
    with open(file_path, mode="r", encoding="utf-8") as file:
        reader = csv.DictReader(file)
        for row in reader:
            # Inline data cleansing & validation
            if row.get("status_code") and row["status_code"].isdigit():
                clean_record = {
                    "user_id": int(row["user_id"]),
                    "endpoint": row["endpoint"].strip(),
                    "status_code": int(row["status_code"]),
                    "latency_ms": float(row["latency_ms"])
                }
                batch.append(clean_record)

            if len(batch) >= batch_size:
                yield batch
                batch = []  # Clear memory reference for garbage collection

        if batch:
            yield batch
```

---

# Part 4: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   WEAK AREA REPAIR CHEAT SHEET                         │
├────────────────────────────────────────────────────────────────────────┤
│ 1. CONDITIONAL ISLANDS:                                                │
│    - ROW_NUMBER() [overall] - ROW_NUMBER() [conditional] = Island Key. │
│ 2. PURE SQL SESSIONIZATION:                                            │
│    - LAG() -> Diff > 15m -> Flag (1/0) -> SUM() OVER (Running Total)   │
│      = Sequential Session Number!                                      │
├────────────────────────────────────────────────────────────────────────┤
│ 3. DEADLOCK PREVENTION:                                                │
│    - Always acquire locks in STRICT SORTED ORDER (first_id < second_id)│
│      to permanently break circular wait cycles.                        │
├────────────────────────────────────────────────────────────────────────┤
│ 4. STREAMING GENERATORS:                                               │
│    - csv.DictReader + yield batches prevents OutOfMemoryError.         │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 5: 100-Mark Self-Assessment Examination

### Section A: Concurrency & Diagnostic Analysis (25 Marks — 5 marks each)

**Q1:** What condition is mathematically required for a deadlock to occur between two transactions?  
(a) Both transactions must be read-only.  
(b) A circular wait dependency in the Wait-For Graph ($T_1 \to T_2 \to T_1$).  
(c) The database must be unindexed.  
(d) Isolation level must be Read Uncommitted.  

**Q2:** In pure SQL sessionization, how is the session number generated after identifying session start events?  
(a) Using `DENSE_RANK()`.  
(b) Computing a cumulative running sum of the session-start boolean flags: `SUM(is_new_session) OVER (ORDER BY event_time)`.  
(c) Multiplying by 15.  
(d) Using `AVG() OVER()`.  

**Q3:** Why does deterministic lock ordering (e.g., sorting account IDs before `SELECT ... FOR UPDATE`) permanently prevent deadlocks?  
(a) It increases transaction speed.  
(b) It prevents circular wait: no transaction can request a lower-numbered resource while holding a higher-numbered resource.  
(c) It forces all queries to execute sequentially on a single thread.  
(d) It disables row-level locking.  

**Q4:** What happens if an unpruned Decision Tree is trained on a dataset with 1,000 features and 1,000 rows?  
(a) The model achieves high bias and underfits.  
(b) The tree memorizes every row, achieving 100% training accuracy and massive test variance (overfitting).  
(c) The tree cannot compile.  
(d) Feature weights become zero.  

**Q5:** In Python, what is the space complexity of an unconsumed generator expression `(x**2 for x in range(10_000_000))`?  
(a) $O(N)$ (requires $\approx 80$ MB of RAM)  
(b) $O(1)$ (requires $\approx 200$ bytes of RAM)  
(c) $O(\log N)$  
(d) $O(N^2)$  

---

### Section B: Advanced Problem Solving (50 Marks)

**Q6 (25 Marks - Stock Price Volatility Streak):**  
Given a financial table `stock_prices (stock_id, price_date, closing_price)`. Write a query to find all stocks that experienced **at least 4 consecutive trading days of strictly increasing closing prices**. Output `stock_id`, streak start date, streak end date, and total days in the streak.

**Q7 (25 Marks - Python LRU Cache):**  
Implement a Least Recently Used (LRU) Cache class `LRUCache(capacity: int)` with methods `get(key: int) -> int` and `put(key: int, value: int)` where both operations execute in strictly $O(1)$ average time complexity using `collections.OrderedDict`.

---

### Section C: Enterprise Architecture Failure Recovery (25 Marks)

**Q8 (25 Marks):** A nightly ETL pipeline loading 20 million rows into an analytics database fails with:  
`ERROR: out of memory for query result (disk spill quota exceeded in TempDB)`.  
1. Diagnose the query operator causing the spill (Hash Join, Sort, Window Function). (10 marks)
2. Detail three architectural strategies to permanently fix the memory spill. (15 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** Circular wait in the Wait-For Graph.
- **Q2:** **(b)** Cumulative running sum of session start flags.
- **Q3:** **(b)** Breaks circular wait condition by enforcing global resource hierarchy.
- **Q4:** **(b)** 100% training accuracy with severe overfitting.
- **Q5:** **(b)** $O(1)$ constant memory ($\approx 200$ bytes).

### Section B Solutions
**Q6 Solution:**
```sql
WITH PriceDeltas AS (
    SELECT 
        stock_id,
        price_date,
        closing_price,
        CASE 
            WHEN closing_price > LAG(closing_price) OVER (PARTITION BY stock_id ORDER BY price_date) 
            THEN 1 ELSE 0 
        END AS is_increase
    FROM stock_prices
),
IslandGroups AS (
    SELECT 
        stock_id,
        price_date,
        closing_price,
        is_increase,
        ROW_NUMBER() OVER (PARTITION BY stock_id ORDER BY price_date) -
        ROW_NUMBER() OVER (PARTITION BY stock_id, is_increase ORDER BY price_date) AS island_id
    FROM PriceDeltas
)
SELECT 
    stock_id,
    MIN(price_date) AS streak_start,
    MAX(price_date) AS streak_end,
    COUNT(*) AS consecutive_increasing_days
FROM IslandGroups
WHERE is_increase = 1
GROUP BY stock_id, island_id
HAVING COUNT(*) >= 4
ORDER BY stock_id, streak_start;
```

**Q7 Solution:**
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        # Move accessed key to end to represent most recent usage
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            # Pop first item (least recently used)
            self.cache.popitem(last=False)
```

### Section C Solutions
**Q8 Solution:**
1. **Diagnosis:** An unindexed `ORDER BY` or an analytical window function (`ROW_NUMBER() OVER (...)`) forced the execution engine to perform an external disk sort (`external merge disk`). Because the sort volume exceeded allocated `work_mem`, intermediate pages spilled to TempDB, exhausting storage quotas.
2. **Remediations:**
   - **Index Pre-sorting:** Create a composite B-tree index matching the exact `PARTITION BY` and `ORDER BY` keys, streaming pre-sorted records directly into the accumulator and eliminating the Sort operator entirely.
   - **Tuning Work Memory:** Increase `work_mem` or `temp_buffers` for the pipeline user connection (`SET work_mem = '512MB'`).
   - **Partitioned Chunking:** Break the 20-million row load into daily date partition slices using incremental dbt models or ADF tumbling windows.
