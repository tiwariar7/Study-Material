# Day 26: Final Timed Comprehensive Technical Assessment
## 90-Minute Closed-Book Examination Across SQL, Python, DBMS, Statistics, ML & System Design

---

## Examination Instructions & Protocol

**Duration:** 90 Minutes  
**Format:** Closed-Book Proctored Technical Benchmark (No notes, no web searches, no AI assistance, no pausing the timer)  
**Total Marks:** 100  
**Passing Threshold:** 75 Marks (Interview Ready) | 90+ Marks (Tier-1 Tech Ready)  
**Structure:**
- **Section 1: Advanced Relational SQL (30 Marks — 3 Problems)**
- **Section 2: Algorithmic Python & Data Wrangling (25 Marks — 2 Problems)**
- **Section 3: Database Internals, Concurrency & Indexing (15 Marks — 3 Problems)**
- **Section 4: Applied Statistics, Probability & Machine Learning (15 Marks — 3 Problems)**
- **Section 5: End-to-End System Design & Architecture Defense (15 Marks — 1 Problem)**

---

# The Examination Paper

---

## Section 1: Advanced Relational SQL (30 Marks)

### Problem 1.1: Relative Ranking with Multi-Attribute Tiebreak (10 Marks)
Given tables `employees (emp_id INT, emp_name VARCHAR, dept_id INT, salary DECIMAL)` and `departments (dept_id INT, dept_name VARCHAR)`.  
Write an ANSI SQL query to find the employees who earn the **top 3 distinct salaries in each department**.
- If multiple employees share the exact same salary, include all of them.
- Output columns: `dept_name`, `emp_name`, `salary`, and `salary_rank`.
- Order the final output by `dept_name` ascending, `salary_rank` ascending, and `emp_name` ascending.

---

### Problem 1.2: Advanced Gaps & Islands Activity Streaks (10 Marks)
Given a website user activity log `user_activity (user_id INT, activity_timestamp TIMESTAMP)`.  
Find all `user_id`s who were active on **at least 3 consecutive calendar days**.
- Note: A user may log multiple events in a single calendar day.
- Output columns: `user_id`, `streak_start_date`, `streak_end_date`, and `total_consecutive_days`.
- Order by `user_id` ascending, `streak_start_date` ascending.

---

### Problem 1.3: Month-over-Month (MoM) Customer Retention Rate (10 Marks)
Given an e-commerce orders table `orders (order_id INT, customer_id INT, order_date DATE, amount DECIMAL)`.  
Write an ANSI SQL query to calculate the **Month-over-Month Customer Retention Rate** for every calendar month:
- For each month $M$, calculate:
  1. `active_customers`: Total distinct customers who placed an order in month $M$.
  2. `retained_customers`: Distinct customers who placed an order in month $M$ **AND** also placed an order in the immediately preceding month ($M-1$).
  3. `retention_rate_pct`: Percentage of month $M-1$'s active customers who returned in month $M$, rounded to 2 decimal places.

---

## Section 2: Algorithmic Python & Data Wrangling (25 Marks)

### Problem 2.1: Longest Substring with At Most K Distinct Characters (12 Marks)
Write a Python function `length_of_longest_substring_k_distinct(s: str, k: int) -> int` that returns the length of the longest contiguous substring that contains at most $k$ distinct characters.
- Constraints: $0 \le k \le 50$, $s$ contains standard ASCII characters.
- Your solution must run in strictly **$O(N)$ time complexity** using the Sliding Window pattern with a hash map.

---

### Problem 2.2: Advanced Clickstream Sessionization in Pandas (13 Marks)
You are given a Pandas DataFrame `clicks` with columns `['user_id', 'timestamp', 'page_url', 'action']`.  
An active user session is defined as a sequence of clicks where the time elapsed between consecutive clicks is **less than 20 minutes**.  
Write a Python function `sessionize_clicks(clicks: pd.DataFrame) -> pd.DataFrame` that:
1. Assigns a unique `session_id` string (e.g., `'user101_s1'`) to every row.
2. Returns an aggregated summary DataFrame grouped by `user_id` containing:
   - `total_sessions`: Count of unique sessions.
   - `avg_clicks_per_session`: Average number of clicks per session, rounded to 1 decimal place.
   - `bounced_sessions`: Count of sessions containing exactly 1 click.

---

## Section 3: Database Internals, Concurrency & Indexing (15 Marks)

### Problem 3.1: The Composite Index Left-Prefix Rule (5 Marks)
A database table `transactions` has 50 million rows and a composite B-tree index:
`CREATE INDEX idx_txns_composite ON transactions (account_id, txn_date, txn_type);`  
Evaluate whether the following queries can utilize an **Index Seek** using this index, and justify your answer for each:
1. `SELECT * FROM transactions WHERE account_id = 1045 AND txn_type = 'WIRE';`
2. `SELECT * FROM transactions WHERE txn_date >= '2024-01-01' AND txn_type = 'WIRE';`

---

### Problem 3.2: Isolation Level Read Phenomena Matrix (5 Marks)
Fill in the following concurrency matrix with **YES** (Permitted) or **NO** (Eliminated) according to ANSI SQL standards:

| Isolation Level | Dirty Read Permitted? | Non-Repeatable Read Permitted? | Phantom Read Permitted? |
| :--- | :--- | :--- | :--- |
| **Read Committed** | | | |
| **Repeatable Read** | | | |
| **Serializable** | | | |

---

### Problem 3.3: Deadlock Detection & Deterministic Lock Ordering (5 Marks)
Explain how the database engine's **Wait-For Graph (WFG)** detects deadlocks. Provide the single application-level engineering practice that permanently eliminates circular wait deadlocks.

---

## Section 4: Applied Statistics & Machine Learning (15 Marks)

### Problem 4.1: Bayesian Diagnostic Paradox & Base Rate Fallacy (5 Marks)
A rare disease affects $0.05\%$ ($0.0005$) of a population. A diagnostic blood test has:
- **Sensitivity (True Positive Rate):** $98\%$ ($0.98$)
- **False Positive Rate:** $2\%$ ($0.02$)  
A randomly tested individual receives a positive result. Calculate the **exact probability** that they actually have the disease. Show all formula steps.

---

### Problem 4.2: Asymmetric Cost Matrix & Decision Threshold Optimization (5 Marks)
In an enterprise credit card fraud model:
- Cost of a **False Negative ($FN$)** (uncaught fraud): $\$250$
- Cost of a **False Positive ($FP$)** (declining a legitimate customer): $\$10$  
Explain why deploying this model with a default probability threshold of $0.50$ is financially suboptimal, and explain the mathematical direction in which the threshold must be adjusted to minimize total expected dollar loss.

---

### Problem 4.3: The Bias-Variance Trade-Off & Regularization (5 Marks)
Compare **Ridge (L2)** and **Lasso (L1)** regularization. Explain why Lasso is capable of performing automated feature selection by setting weights strictly to zero, whereas Ridge only shrinks weights asymptotically toward zero.

---

## Section 5: End-to-End System Design & Architecture Defense (15 Marks)

### Problem 5.1: The Modern Data Stack & Pipeline Resilience (15 Marks)
You are hired as a Lead Data Science Engineer to design a real-time analytics platform for **vConstruct**.  
The system must ingest sensor telemetry and construction progress updates from 50 global job sites:
1. **Draw the Architectural Diagram / Component Flow** from raw ingestion to executive dashboard (IoT/API $\to$ Ingestion Buffer $\to$ Raw Lake $\to$ Cloud Warehouse $\to$ dbt Modeling $\to$ Power BI). (5 marks)
2. **Defend Storage & Compute Decoupling:** Why is Snowflake's virtual warehouse architecture superior to a monolithic PostgreSQL server for this workload? (5 marks)
3. **Disaster Recovery & Failure Handling:** How does your architecture handle upstream schema changes and corrupted records without halting the entire daily reporting pipeline? (5 marks)

---

# Official Answer Key & Scoring Guide

---

## Section 1: SQL Solutions (30 Marks)

### Solution 1.1 (10 Marks): Top-N per Department with Ties
```sql
WITH RankedEmployees AS (
    SELECT 
        d.dept_name,
        e.emp_name,
        e.salary,
        DENSE_RANK() OVER (
            PARTITION BY e.dept_id 
            ORDER BY e.salary DESC
        ) AS salary_rank
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
)
SELECT 
    dept_name,
    emp_name,
    salary,
    salary_rank
FROM RankedEmployees
WHERE salary_rank <= 3
ORDER BY dept_name ASC, salary_rank ASC, emp_name ASC;
```
*Grading Rubric:*
- 4 marks: Correct use of `DENSE_RANK()` with `PARTITION BY dept_id ORDER BY salary DESC`.
- 2 marks: Correct join between `employees` and `departments`.
- 2 marks: Proper outer `WHERE salary_rank <= 3` filter.
- 2 marks: Correct ordering clause.

---

### Solution 1.2 (10 Marks): Gaps & Islands Activity Streaks
```sql
WITH DistinctDays AS (
    SELECT DISTINCT 
        user_id,
        CAST(activity_timestamp AS DATE) AS act_date
    FROM user_activity
),
SequencedDays AS (
    SELECT 
        user_id,
        act_date,
        act_date - CAST(ROW_NUMBER() OVER (
            PARTITION BY user_id 
            ORDER BY act_date
        ) AS INT) * INTERVAL '1 day' AS island_key
    FROM DistinctDays
)
SELECT 
    user_id,
    MIN(act_date) AS streak_start_date,
    MAX(act_date) AS streak_end_date,
    COUNT(*) AS total_consecutive_days
FROM SequencedDays
GROUP BY user_id, island_key
HAVING COUNT(*) >= 3
ORDER BY user_id ASC, streak_start_date ASC;
```
*Grading Rubric:*
- 3 marks: Deduplicating timestamps to calendar dates via `DISTINCT`.
- 4 marks: Applying the Difference of Sequences technique (`act_date - ROW_NUMBER()`).
- 3 marks: Grouping by `island_key` and filtering with `HAVING COUNT(*) >= 3`.

---

### Solution 1.3 (10 Marks): MoM Customer Retention Rate
```sql
WITH MonthlyActiveUsers AS (
    SELECT DISTINCT 
        customer_id,
        DATE_TRUNC('month', order_date) AS order_month
    FROM orders
),
MonthlyCohorts AS (
    SELECT 
        m1.order_month,
        COUNT(DISTINCT m1.customer_id) AS active_customers,
        COUNT(DISTINCT m2.customer_id) AS retained_customers
    FROM MonthlyActiveUsers m1
    LEFT JOIN MonthlyActiveUsers m2
        ON m1.customer_id = m2.customer_id
       AND m1.order_month = m2.order_month + INTERVAL '1 month'
    GROUP BY m1.order_month
)
SELECT 
    order_month,
    active_customers,
    retained_customers,
    ROUND(
        100.0 * retained_customers / NULLIF(LAG(active_customers) OVER (ORDER BY order_month), 0),
        2
    ) AS retention_rate_pct
FROM MonthlyCohorts
ORDER BY order_month;
```
*Grading Rubric:*
- 3 marks: Isolating unique monthly active users per month.
- 4 marks: Self-join on `m1.order_month = m2.order_month + INTERVAL '1 month'`.
- 3 marks: Using `LAG()` to compute percentage relative to previous month with `NULLIF` divide-by-zero protection.

---

## Section 2: Python & Data Wrangling Solutions (25 Marks)

### Solution 2.1 (12 Marks): Longest Substring with At Most K Distinct Characters
```python
def length_of_longest_substring_k_distinct(s: str, k: int) -> int:
    if k == 0 or not s:
        return 0

    char_counts = {}
    max_len = 0
    left = 0

    for right, char in enumerate(s):
        char_counts[char] = char_counts.get(char, 0) + 1

        # Contract window if distinct character count exceeds k
        while len(char_counts) > k:
            left_char = s[left]
            char_counts[left_char] -= 1
            if char_counts[left_char] == 0:
                del char_counts[left_char]
            left += 1

        max_len = max(max_len, right - left + 1)

    return max_len
```
*Grading Rubric:*
- 4 marks: Correct sliding window structure with `left` and `right` pointers.
- 4 marks: Proper hash map frequency updates and key deletion when count reaches 0.
- 4 marks: Accurate window length calculation and handling edge cases ($k=0$, empty string).

---

### Solution 2.2 (13 Marks): Clickstream Sessionization
```python
import pandas as pd

def sessionize_clicks(clicks: pd.DataFrame) -> pd.DataFrame:
    df = clicks.copy()
    df["timestamp"] = pd.to_datetime(df["timestamp"])
    df = df.sort_values(["user_id", "timestamp"]).reset_index(drop=True)

    # Calculate gap to previous click
    time_diff = df.groupby("user_id")["timestamp"].diff()
    new_session = time_diff.isna() | (time_diff >= pd.Timedelta(minutes=20))

    # Sequential session numbering per user
    df["session_idx"] = df.groupby("user_id")[new_session.name].cumsum()
    df["session_id"] = df["user_id"].astype(str) + "_s" + df["session_idx"].astype(str)

    # Session level metrics
    session_stats = df.groupby(["user_id", "session_id"]).agg(
        click_count=("page_url", "count")
    ).reset_index()

    # User level report
    user_report = session_stats.groupby("user_id").agg(
        total_sessions=("session_id", "count"),
        avg_clicks_per_session=("click_count", lambda x: round(x.mean(), 1)),
        bounced_sessions=("click_count", lambda x: (x == 1).sum())
    ).reset_index()

    return user_report
```
*Grading Rubric:*
- 4 marks: Correctly identifying session boundaries via `.diff() >= 20 mins`.
- 4 marks: Generating cumulative session IDs via `.cumsum()`.
- 5 marks: Accurate two-stage aggregation producing `total_sessions`, `avg_clicks`, and `bounced_sessions`.

---

## Section 3: Database Internals Solutions (15 Marks)

### Solution 3.1 (5 Marks): Left-Prefix Rule
1. **Query 1 (`account_id = 1045 AND txn_type = 'WIRE'`):** **PARTIAL SEEK.**  
   - The engine uses `account_id` to seek directly to matching leaf pages ($O(\log N)$). However, because `txn_date` was skipped, it cannot seek on `txn_type`; it scans the remaining matching rows in memory to evaluate `txn_type`.
2. **Query 2 (`txn_date >= '2024-01-01' AND txn_type = 'WIRE'`):** **NO SEEK (FULL SCAN).**  
   - The query completely omits the leading column `account_id`. The B-tree is sorted primarily by `account_id`, so the engine cannot navigate the tree hierarchy and must resort to a full index or table scan.

---

### Solution 3.2 (5 Marks): Isolation Level Matrix
| Isolation Level | Dirty Read Permitted? | Non-Repeatable Read Permitted? | Phantom Read Permitted? |
| :--- | :--- | :--- | :--- |
| **Read Committed** | **NO** | **YES** | **YES** |
| **Repeatable Read** | **NO** | **NO** | **YES** (Standard ANSI) / **NO** (Postgres/InnoDB) |
| **Serializable** | **NO** | **NO** | **NO** |

---

### Solution 3.3 (5 Marks): Deadlock Detection & Resolution
- **Wait-For Graph:** The engine maintains a directed graph where transactions are nodes and locks are edges ($T_1 \to T_2$ means $T_1$ waits for $T_2$). A background thread traverses the graph; if a **directed cycle** is detected ($T_1 \to T_2 \to T_1$), a deadlock exists, and the engine aborts the lowest-cost victim transaction.
- **Prevention Practice:** **Deterministic Lock Ordering**: all application transactions must acquire locks on resources in the exact same sorted numerical sequence (e.g., sorting primary keys before executing `SELECT ... FOR UPDATE`), mathematically eliminating circular wait conditions.

---

## Section 4: Statistics & Machine Learning Solutions (15 Marks)

### Solution 4.1 (5 Marks): Bayes' Theorem Proof
$$P(D) = 0.0005, \quad P(H) = 0.9995$$
$$P(+ \mid D) = 0.98, \quad P(+ \mid H) = 0.02$$

$$P(+) = (0.98 \times 0.0005) + (0.02 \times 0.9995) = 0.00049 + 0.01999 = 0.02048$$

$$P(D \mid +) = \frac{0.98 \times 0.0005}{0.02048} = \frac{0.00049}{0.02048} \approx 0.0239 \quad (\mathbf{2.39\%})$$
- *Conclusion:* Even with a 98% sensitive test, an individual who tests positive has only a **$2.39\%$ chance** of actually having the disease due to the extremely rare base rate ($0.05\%$).

---

### Solution 4.2 (5 Marks): Asymmetric Cost Optimization
- **Why 0.5 is Suboptimal:** A default 0.5 threshold assumes symmetric error costs ($C(FN) = C(FP)$). In this scenario, missing a fraudulent transaction ($FN$) costs $\$250$, which is **$25\times$ more expensive** than a false alarm ($FP = \$10$).
- **Direction of Adjustment:** The decision threshold must be **lowered significantly** (e.g., from $0.50$ down to $0.08–0.15$). This forces the model to flag transactions as fraud even with low probability, deliberately trading off more false positives ($10) to eliminate catastrophic false negatives ($250).

---

### Solution 4.3 (5 Marks): Ridge vs. Lasso Regularization
- **L1 (Lasso):** Uses an absolute value penalty ($\sum |w_j|$), which creates a **diamond-shaped constraint boundary** with sharp corners on the coordinate axes. Loss contours frequently hit these corners directly on the axes, setting non-informative feature weights **strictly to zero** (performing automatic feature selection).
- **L2 (Ridge):** Uses a squared penalty ($\sum w_j^2$), creating a smooth **circular/spherical constraint boundary**. Loss contours touch the circle along continuous curves, shrinking weights close to zero but never reaching exact zero.

---

## Section 5: System Design & Architecture Solutions (15 Marks)

### Solution 5.1 (15 Marks): Enterprise IoT Analytics Platform
1. **Architecture Blueprint (5 marks):**
   - *Ingestion:* IoT Gateways buffer telemetry $\to$ Azure Event Hubs / Kafka $\to$ Azure Data Factory tumbling window jobs.
   - *Storage:* Raw JSON landed immutably in ADLS Gen2 Bronze lake.
   - *Warehousing:* Snowflake loads Bronze via `COPY INTO`.
   - *Transformation:* dbt executes scheduled runs compiling Silver conformed tables and Gold Star Schema models (`fct_equipment_telemetry`, `dim_job_site`, `dim_equipment`).
   - *BI Layer:* Power BI connects to Gold tables in Import Mode; DAX measures calculate utilization and idle fuel costs.
2. **Storage & Compute Decoupling Defense (5 marks):**
   - Independent Virtual Warehouses eliminate resource contention: heavy batch transformations running on `ETL_WH` do not degrade interactive Power BI executive queries running on `BI_WH`.
   - Per-second billing with Auto-Suspend ensures compute clusters shut down when idle, saving up to 70% in cloud infrastructure costs compared to an always-on monolithic PostgreSQL instance.
3. **Disaster Recovery & Failure Handling (5 marks):**
   - Ingestion pipelines implement a **Quarantine Pattern / Dead Letter Queue (DLQ)**: schema-corrupted sensor payloads are routed to an error table while clean records process uninterrupted.
   - Raw data in Bronze ADLS Gen2 is append-only and immutable, allowing full replay from scratch if transformation logic requires updates.
