# Day 20: SQL & DBMS 35-Minute Mock Interview Simulation
## Timed Interview Protocol, Live Query Whiteboarding, Deep-Dive Systems Grilling & Post-Mortem Audit

---

# Part 1: The 35-Minute Timed Mock Interview Protocol

```
┌────────────────────────────────────────────────────────────────────────┐
│                   35-MINUTE SQL + DBMS INTERVIEW PACING                │
├────────────────────────────────────────────────────────────────────────┤
│ [00:00 - 08:00] PHASE 1: RAPID-FIRE THEORY SCREEN                      │
│   - Tests agility across ACID, Isolation Levels, Joins, and NULLs.     │
│   - Target: Crisp 30-to-45-second answers. No rambling.                │
├────────────────────────────────────────────────────────────────────────┤
│ [08:00 - 18:00] PHASE 2: ARCHITECTURE & SYSTEMS GRILLING               │
│   - Tests deep engine internals: B-Tree vs. Hash indexes, Deadlocks,   │
│     MVCC mechanics, and 3NF vs. BCNF normalization.                    │
├────────────────────────────────────────────────────────────────────────┤
│ [18:00 - 32:00] PHASE 3: LIVE CODING CHALLENGES (2 PROBLEMS)           │
│   - Problem 1 (7 mins): Top-N per Department with Strict Tie Handling. │
│   - Problem 2 (7 mins): Consecutive Transaction Streaks (Gaps/Islands).│
├────────────────────────────────────────────────────────────────────────┤
│ [32:00 - 35:00] PHASE 4: POST-MORTEM & SCORING AUDIT                   │
│   - Instant error diagnosis, execution plan review, and rubric scoring.│
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 2: Phase 1 — Rapid-Fire Theory Screen (8 Minutes)

### Q1 (01:00): What is the practical difference between `WHERE` and `HAVING`?
**Candidate Model Answer:**
> *"The `WHERE` clause filters individual base records before grouping occurs and can leverage standard B-tree indexes; it cannot contain aggregate functions. The `HAVING` clause filters aggregated group buckets after grouping has executed, and typically evaluates aggregate conditions like `COUNT(*) > 2`."*

### Q2 (02:30): What happens when an outer join predicate is placed in the `ON` clause versus the `WHERE` clause of a `LEFT JOIN`?
**Candidate Model Answer:**
> *"In a `LEFT JOIN`, conditions in the `ON` clause determine which right-table rows attach to the left-table rows; all left-table rows are preserved regardless of whether the right-side condition matches. Placing a filter on the right table in the `WHERE` clause evaluates post-join; unmatched left rows have `NULL` right attributes, causing the `WHERE` filter to evaluate to `UNKNOWN` and discard them, silently converting the `LEFT JOIN` into an `INNER JOIN`."*

### Q3 (04:00): Why does `COUNT(*)` differ from `COUNT(column_name)`?
**Candidate Model Answer:**
> *"`COUNT(*)` counts every row in the result set, including rows where all non-key columns are `NULL`. `COUNT(column_name)` counts strictly those rows where the specified column value is `NOT NULL`."*

### Q4 (05:30): Explain why `NULL = NULL` does not return `TRUE` in SQL.
**Candidate Model Answer:**
> *"In ANSI SQL's Three-Valued Logic, `NULL` represents an unknown or missing state. Two unknown values cannot be confirmed to be identical, so the equality comparison `NULL = NULL` yields `UNKNOWN`. To match or test for nulls, one must use `IS NULL` or `IS NOT DISTINCT FROM`."*

### Q5 (07:00): Under `READ COMMITTED` isolation, are "Non-Repeatable Reads" possible?
**Candidate Model Answer:**
> *"Yes. `READ COMMITTED` guarantees that a query only reads committed data, eliminating Dirty Reads. However, if Transaction A reads a row, and Transaction B concurrently updates that row and commits, Transaction A re-reading the same row will see the new value within the same transaction scope, experiencing a Non-Repeatable Read. Preventing this requires `REPEATABLE READ` or `SERIALIZABLE`."*

---

# Part 3: Phase 2 — Systems Architecture & Internals Deep Dive (10 Minutes)

### Q6 (09:30): How does a B+ Tree index differ physically from a Hash Index, and when does a Hash Index fail?
**Candidate Model Answer:**
> *"A B+ Tree stores keys in a self-balancing tree structure where all data resides in sequentially linked leaf disk pages. It excels at point lookups ($O(\log N)$), range queries (`BETWEEN`, `>`, `<`), prefix scans (`LIKE 'abc%'`), and pre-sorted output (`ORDER BY`).  
> A Hash Index uses an in-memory hash table offering $O(1)$ point lookups. However, a Hash Index completely fails for range scans, sorting, or prefix searches because the hash function destroys mathematical ordering."*

### Q7 (12:00): Explain the mechanics of a Deadlock. How does an engine detect it, and how do you resolve it in application architecture?
**Candidate Model Answer:**
> *"A deadlock occurs when two transactions hold exclusive locks on resources that the other requires to proceed, creating a circular wait dependency ($T_1 \to T_2 \to T_1$). The engine detects this using a background thread that traverses a directed **Wait-For Graph (WFG)** to detect cycles; upon detecting a cycle, it aborts the lowest-cost transaction and throws error `40P01`.  
> In application design, we prevent deadlocks by enforcing **Deterministic Lock Ordering**: all transactions must access tables and rows in the exact same sorted sequence (e.g., sorting account IDs before acquiring row locks).*`*

### Q8 (14:30): What is the difference between 3NF and BCNF? Give a concrete scenario where 3NF is satisfied but BCNF is violated.
**Candidate Model Answer:**
> *"3NF allows a functional dependency $X \to Y$ if $X$ is a super key OR if $Y$ is a prime attribute. BCNF is stricter: for every functional dependency, the determinant $X$ must be a super key.  
> A classic violation occurs with overlapping candidate keys: In `StudentAdvisor(student, subject, advisor)`, each advisor teaches one subject (`advisor -> subject`), but each student has one advisor per subject (`(student, subject) -> advisor`). Because `subject` is a prime attribute (part of candidate key 1), it passes 3NF. But `advisor` is not a super key, so it violates BCNF, causing insertion anomalies when adding a new advisor before any students enroll."*

### Q9 (17:00): What is Write-Ahead Logging (WAL) and how does it guarantee Atomicity and Durability?
**Candidate Model Answer:**
> *"WAL dictates that any data modification must be written to an immutable, sequential disk log before dirty buffer cache pages are flushed to the database storage files. If a crash occurs, the recovery manager replays committed transactions from the WAL (Redo phase) ensuring **Durability**, and rolls back uncommitted transactions using pre-image Undo logs ensuring **Atomicity**."*

---

# Part 4: Phase 3 — Live Timed SQL Coding (14 Minutes)

---

### Challenge 1 (7 Minutes): Department Top 2 Earners with Strict Tie Preservation

**Problem Statement:**  
Given tables `emp (emp_id INT, emp_name VARCHAR, salary DECIMAL, dept_id INT)` and `dept (dept_id INT, dept_name VARCHAR)`.  
Find the top 2 highest distinct salaries in each department. If employees tie for 1st or 2nd place, return all tied employees. Include `dept_name`, `emp_name`, and `salary`, ordered by `dept_name` ascending and `salary` descending.

**Candidate Spoken Thought Process:**
> *"I'll structure this with a CTE using `DENSE_RANK()`. `DENSE_RANK()` ensures that if two employees tie for the top salary, both receive rank 1, and the next distinct salary receives rank 2. I'll join `emp` with `dept` inside the CTE, partition by `dept_id`, order by `salary DESC`, and then filter where `rnk <= 2`."*

**Production Solution:**
```sql
WITH RankedDepartmentSalaries AS (
    SELECT 
        d.dept_name,
        e.emp_name,
        e.salary,
        DENSE_RANK() OVER (
            PARTITION BY e.dept_id 
            ORDER BY e.salary DESC
        ) AS salary_rank
    FROM emp e
    INNER JOIN dept d ON e.dept_id = d.dept_id
)
SELECT 
    dept_name,
    emp_name,
    salary
FROM RankedDepartmentSalaries
WHERE salary_rank <= 2
ORDER BY dept_name ASC, salary DESC;
```

---

### Challenge 2 (7 Minutes): Consecutive Active Transaction Days (Gaps & Islands)

**Problem Statement:**  
Given a financial transactions table `transactions (txn_id INT, account_id INT, txn_date TIMESTAMP, amount DECIMAL)`.  
Find all `account_id`s that executed at least one transaction on **four or more consecutive calendar days**.

**Candidate Spoken Thought Process:**
> *"First, an account can execute multiple transactions in a single day, so I must deduplicate `(account_id, CAST(txn_date AS DATE))` in Step 1.  
> Next, I apply the Difference of Sequences technique: subtract a continuous `ROW_NUMBER()` from the calendar date. For consecutive dates, this difference produces an identical anchor date (`island_id`).  
> Finally, I group by `account_id` and `island_id`, filtering with `HAVING COUNT(*) >= 4`, and select distinct account IDs."*

**Production Solution:**
```sql
WITH DistinctDailyTxns AS (
    SELECT DISTINCT 
        account_id,
        CAST(txn_date AS DATE) AS activity_date
    FROM transactions
),
SequencedDays AS (
    SELECT 
        account_id,
        activity_date,
        activity_date - CAST(ROW_NUMBER() OVER (
            PARTITION BY account_id 
            ORDER BY activity_date
        ) AS INT) * INTERVAL '1 day' AS island_key
    FROM DistinctDailyTxns
)
SELECT DISTINCT 
    account_id
FROM SequencedDays
GROUP BY account_id, island_key
HAVING COUNT(*) >= 4
ORDER BY account_id;
```

---

# Part 5: Phase 4 — Post-Mortem Diagnostic Guide & Common Pitfalls

Review your performance against the common mistakes observed in technical rounds:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   MOCK INTERVIEW POST-MORTEM AUDIT                     │
├────────────────────────────────────────────────────────────────────────┤
│ 1. DID YOU USE RANK() INSTEAD OF DENSE_RANK()?                         │
│    - If two employees tie for 1st place, RANK() assigns (1, 1, 3).     │
│      Filtering for <= 2 would MISS the 2nd distinct salary entirely!   │
├────────────────────────────────────────────────────────────────────────┤
│ 2. DID YOU FORGET TO DEDUPLICATE DATES IN GAPS & ISLANDS?              │
│    - If a user transacts 4 times on Monday and zero times on Tuesday,  │
│      a query without DISTINCT daily dates will flag Monday as a        │
│      4-day streak! Always deduplicate dates first.                     │
├────────────────────────────────────────────────────────────────────────┤
│ 3. DID YOU HANG SILENTLY WHILE CODING?                                 │
│    - Always state your plan before typing: "I will use a CTE with      │
│      DENSE_RANK() to handle ties, then filter in the outer WHERE."     │
├────────────────────────────────────────────────────────────────────────┤
│ 4. ARE YOUR JOIN COLUMNS COVERED BY B-TREE INDEXES?                    │
│    - Mentioning: "In production, an index on (dept_id, salary DESC)    │
│      eliminates the Sort operator" converts an L4 answer into an L5.   │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 6: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   MOCK INTERVIEW SUMMARY CHEAT SHEET                   │
├────────────────────────────────────────────────────────────────────────┤
│ 1. RAPID THEORY:                                                       │
│    - WHERE: Pre-aggregation row filter. Indexes used. No aggregates.   │
│    - HAVING: Post-aggregation group filter. Evaluates aggregates.      │
│    - ON vs WHERE in LEFT JOIN: ON preserves left rows; WHERE drops them│
│    - NULL = NULL: Evaluates to UNKNOWN. Use IS NULL.                   │
├────────────────────────────────────────────────────────────────────────┤
│ 2. SYSTEMS INTERNALS:                                                  │
│    - B+ Tree: O(log N). Handles ranges, sorting, prefix seeks.         │
│    - Hash Index: O(1) point lookups only. Fails on ranges/sorts.       │
│    - Deadlock: Cycle in Wait-For Graph. Fix via lock ordering.         │
│    - BCNF: Every determinant must be a Super Key.                      │
├────────────────────────────────────────────────────────────────────────┤
│ 3. LIVE CODING CORE:                                                   │
│    - Top-N: DENSE_RANK() OVER (PARTITION BY grp ORDER BY col DESC) <= N│
│    - Gaps/Islands: Date - ROW_NUMBER() * INTERVAL '1 day' = Island Key.│
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 7: 100-Mark Self-Assessment Examination

### Section A: Technical Conceptual Defense (25 Marks — 5 marks each)

**Q1:** What is the result of `SELECT COUNT(*)` on a table containing 10 rows where every column in 3 of those rows is `NULL`?  
(a) 0  
(b) 7  
(c) 10  
(d) Throws a NullPointerException  

**Q2:** Under which ANSI SQL isolation level are Phantom Reads completely eliminated in standard relational databases?  
(a) Read Uncommitted  
(b) Read Committed  
(c) Repeatable Read (Postgres MVCC / MySQL Next-Key) and Serializable  
(d) Snapshot Read Only  

**Q3:** Why does a composite index on `(dept_id, salary, hire_date)` fail to accelerate a query with `WHERE salary > 80000`?  
(a) `salary` is a numeric type.  
(b) It violates the Left-Prefix Rule: the leftmost column (`dept_id`) is omitted from the predicate.  
(c) Greater-than operators cannot be indexed.  
(d) Indexes only support primary keys.  

**Q4:** What is the primary difference between a clustered index and a non-clustered index?  
(a) Clustered indexes are stored in RAM; non-clustered indexes are on disk.  
(b) A clustered index dictates the physical sort order of actual data pages on disk (strictly 1 per table); a non-clustered index is a secondary B-tree pointing to row locators.  
(c) Clustered indexes do not support integers.  
(d) Non-clustered indexes cannot have duplicate keys.  

**Q5:** In a BCNF decomposition of $R(A, B, C)$ where $A \to B$ and $B \to C$, what are the resulting relations?  
(a) $R_1(A, B)$ and $R_2(B, C)$  
(b) $R_1(A, C)$ and $R_2(B, C)$  
(c) $R_1(A, B, C)$  
(d) $R_1(A)$ and $R_2(B)$  

---

### Section B: Live Query Problem Solving (50 Marks)

**Q6 (25 Marks - Department Budget Ranking):**  
Given `projects (project_id, dept_id, budget, start_date)`. Write a query to find the 3rd highest distinct budget in each department. If a department has fewer than 3 distinct budgets, return `NULL` for that department. Include `dept_id` and `third_highest_budget`.

**Q7 (25 Marks - Monthly User Churn):**  
Given a table `logins (user_id INT, login_date DATE)`. Write a query to calculate the number of users who logged in during January 2024 but did NOT log in during February 2024.

---

### Section C: Systems Optimization & Execution Plans (25 Marks)

**Q8 (25 Marks):** An interviewer shows you this `EXPLAIN ANALYZE` snippet on a 15-million row `transactions` table:
```
Hash Join  (cost=125000.00..845000.00 rows=450 width=48) (actual time=1820.12..3450.45 rows=12 loops=1)
  Hash Cond: (t.user_id = u.user_id)
  ->  Seq Scan on transactions t  (actual rows=15000000 loops=1)
  ->  Hash  (actual rows=50 loops=1)
        ->  Index Scan on users u  (actual rows=50 loops=1)
```
1. Explain what the query planner did and why execution took over 3.4 seconds. (10 marks)
2. What single index should you create on `transactions` to replace the 15-million row `Seq Scan` with an immediate `Index Seek`, dropping runtime to under 5ms? (15 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(c)** 10 (`COUNT(*)` counts all physical rows).
- **Q2:** **(c)** Serializable (and Repeatable Read in Postgres/MySQL InnoDB).
- **Q3:** **(b)** Violates the Left-Prefix Rule.
- **Q4:** **(b)** Clustered index orders physical data pages directly on disk (strictly 1).
- **Q5:** **(a)** $R_1(A, B)$ and $R_2(B, C)$.

### Section B Solutions
**Q6 Solution:**
```sql
WITH RankedBudgets AS (
    SELECT 
        dept_id,
        budget,
        DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY budget DESC) AS rnk
    FROM projects
),
ThirdBudgets AS (
    SELECT DISTINCT dept_id, budget AS third_highest_budget
    FROM RankedBudgets
    WHERE rnk = 3
)
SELECT 
    d.dept_id,
    t.third_highest_budget
FROM (SELECT DISTINCT dept_id FROM projects) d
LEFT JOIN ThirdBudgets t ON d.dept_id = t.dept_id;
```

**Q7 Solution:**
```sql
SELECT COUNT(DISTINCT user_id) AS churned_users_count
FROM logins
WHERE login_date >= '2024-01-01' AND login_date < '2024-02-01'
  AND user_id NOT IN (
      SELECT DISTINCT user_id 
      FROM logins 
      WHERE login_date >= '2024-02-01' AND login_date < '2024-03-01'
  );
```

### Section C Solutions
**Q8 Solution:**
1. **Bottleneck:** The planner built an in-memory hash table of 50 users from the `Index Scan` on `users`. However, because `transactions.user_id` lacked a usable index, the engine had to perform a full `Seq Scan` across all 15 million rows of `transactions` to probe the hash table, taking 3.45 seconds.
2. **Optimal Index:**
   ```sql
   CREATE INDEX idx_transactions_user_id ON transactions (user_id);
   ```
   - With an index on `transactions.user_id`, the planner replaces the Hash Join with an **Indexed Nested Loop Join**: for each of the 50 users, it performs a logarithmic $O(\log N)$ B-tree index seek into `transactions`, executing in $< 5$ milliseconds.
