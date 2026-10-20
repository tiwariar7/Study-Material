# Day 16: Advanced SQL Interview Problem-Solving Masterclass
## Ranking, Gaps & Islands, Running Totals, Retention Cohorts & Verbal Interview Communication

---

# Part 1: The Senior Interview Communication Framework

When solving SQL live in technical interview rounds, **writing code in silence is a major failure mode**. Interviewers evaluate your ability to clarify ambiguity, anticipate edge cases, and reason through engine performance.

```
┌────────────────────────────────────────────────────────────────────────┐
│               THE 5-STEP VERBAL INTERVIEW COMMUNICATION FLOW           │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 1: CLARIFY REQUIREMENTS & AMBIGUITY                               │
│  - "Are there ties? How should identical salaries be ranked?"          │
│  - "Can columns contain NULLs? How should missing dates be handled?"   │
│  - "Should the output return NULL or an empty set if no match exists?" │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 2: STATE THE PROPOSED STRATEGY (BEFORE TYPING)                    │
│  - "I will structure this using a CTE with DENSE_RANK() partitioned   │
│    by department, then filter in the outer query where rank <= 3."     │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 3: CODE MODULARLY WITH CLEAN ALIASES                              │
│  - Use uppercase keywords, explicit table aliases (e, d, o), and CTEs. │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 4: WALK THROUGH A CONCRETE EXAMPLE MENTALLY                      │
│  - Trace inputs with ties or nulls through your execution pipeline.    │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 5: PROACTIVELY DISCUSS COMPLEXITY & INDEXING                      │
│  - "This window function requires a sort on (dept_id, salary DESC). A  │
│    composite B-tree index on those columns would eliminate the sort."  │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 2: The 15 High-Frequency SQL Interview Problems

---

### Problem 1: Second-Highest Salary
**Problem Statement:** Write a SQL query to find the second-highest distinct salary from an `employees` table. If there is no second-highest salary (e.g., table has 0 or 1 row, or all employees earn the exact same salary), the query must return `NULL`.

**Schema:** `employees (emp_id INT, emp_name VARCHAR, salary DECIMAL)`

**Verbal Explanation Script:**
> *"To ensure that ties don't distort our ranking, I'll use `DENSE_RANK()` ordered by salary descending inside a CTE. `DENSE_RANK()` guarantees that if two employees tie for the top salary of \$150k, both receive rank 1, and the next distinct salary receives rank 2. Furthermore, the requirement specifies returning a scalar `NULL` rather than an empty result set if fewer than two distinct salaries exist. I'll achieve this by wrapping the subquery in a scalar `SELECT COALESCE(...)`."*

**Production Solution:**
```sql
WITH RankedSalaries AS (
    SELECT 
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT COALESCE(
    (SELECT DISTINCT salary FROM RankedSalaries WHERE rnk = 2),
    NULL
) AS SecondHighestSalary;
```

---

### Problem 2: Top 3 Highest Earners per Department
**Problem Statement:** Find the employees who earn the top 3 highest distinct salaries in each department. If there are ties for 3rd place, return all tied employees.

**Schema:** `employees (emp_id, emp_name, salary, dept_id)`, `departments (dept_id, dept_name)`

**Verbal Explanation Script:**
> *"This is a classic Top-N per group problem. We must partition by department and order by salary descending. Because tied salaries should both be included without skipping ranks, `DENSE_RANK()` is mathematically required. I'll compute this in a CTE called `RankedDeptEmployees`, join with `departments` for readability, and then filter in the outer `WHERE` clause where `salary_rank <= 3`."*

**Production Solution:**
```sql
WITH RankedDeptEmployees AS (
    SELECT 
        e.emp_id,
        e.emp_name,
        e.salary,
        e.dept_id,
        DENSE_RANK() OVER (
            PARTITION BY e.dept_id 
            ORDER BY e.salary DESC
        ) AS salary_rank
    FROM employees e
)
SELECT 
    d.dept_name,
    r.emp_name,
    r.salary,
    r.salary_rank
FROM RankedDeptEmployees r
INNER JOIN departments d ON r.dept_id = d.dept_id
WHERE r.salary_rank <= 3
ORDER BY d.dept_name, r.salary_rank, r.emp_name;
```

---

### Problem 3: Three or More Consecutive Logins (Gaps & Islands)
**Problem Statement:** Given a table of user login activities, identify all users who logged in for at least 3 consecutive calendar days.

**Schema:** `user_logins (user_id INT, login_date DATE)`

**Verbal Explanation Script:**
> *"First, users might log in multiple times in a single day, so Step 1 is to deduplicate using `SELECT DISTINCT user_id, CAST(login_date AS DATE)`.  
> Next, I apply the classic 'Difference of Sequences' Gaps and Islands technique: if dates are consecutive ($D, D+1, D+2$) and we assign a sequential `ROW_NUMBER()` ($1, 2, 3$), subtracting the row number from the login date yields a constant anchor date for that consecutive island. When a gap occurs, the anchor date changes.  
> Finally, I group by `user_id` and `island_date`, filtering with `HAVING COUNT(*) >= 3`."*

**Production Solution:**
```sql
WITH CleanLogins AS (
    SELECT DISTINCT user_id, CAST(login_date AS DATE) AS log_date
    FROM user_logins
),
SequencedLogins AS (
    SELECT 
        user_id,
        log_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY log_date) AS rn,
        -- Subtracting rn days produces a constant date group for consecutive streaks
        log_date - CAST(ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY log_date) AS INT) * INTERVAL '1 day' AS island_group
    FROM CleanLogins
)
SELECT 
    user_id,
    MIN(log_date) AS streak_start,
    MAX(log_date) AS streak_end,
    COUNT(*) AS consecutive_days
FROM SequencedLogins
GROUP BY user_id, island_group
HAVING COUNT(*) >= 3
ORDER BY user_id, streak_start;
```

---

### Problem 4: Missing Records & Unmatched Transactions (Anti-Join)
**Problem Statement:** Find all customers who registered in 2024 but have never completed a single transaction.

**Schema:** `customers (customer_id, customer_name, registration_date)`, `transactions (txn_id, customer_id, amount, status)`

**Verbal Explanation Script:**
> *"This requires an Anti-Join pattern. While a `NOT IN` subquery is common, it is hazardous if the subquery contains any `NULL`s, which causes `NOT IN` to return zero rows. I will use a `LEFT JOIN` on `customer_id` and filter where the transaction primary key `IS NULL`, combined with an explicit check on the transaction status."*

**Production Solution:**
```sql
SELECT 
    c.customer_id,
    c.customer_name,
    c.registration_date
FROM customers c
LEFT JOIN transactions t 
    ON c.customer_id = t.customer_id 
    AND t.status = 'COMPLETED'
WHERE EXTRACT(YEAR FROM c.registration_date) = 2024
  AND t.txn_id IS NULL;
```

---

### Problem 5: Department-wise Salary Running Total
**Problem Statement:** For each department, list employees ordered by their hire date, along with the running cumulative sum of salaries paid within that department.

**Schema:** `employees (emp_id, emp_name, dept_id, salary, hire_date)`

**Verbal Explanation Script:**
> *"We use an aggregate window function `SUM(salary) OVER (...)`. We partition by `dept_id` and order by `hire_date`. To ensure that employees hired on the exact same date don't have their salaries lumped together prematurely by the default `RANGE` frame, I will explicitly specify the frame as `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`."*

**Production Solution:**
```sql
SELECT 
    dept_id,
    emp_name,
    hire_date,
    salary,
    SUM(salary) OVER (
        PARTITION BY dept_id 
        ORDER BY hire_date, emp_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_department_payroll
FROM employees
ORDER BY dept_id, hire_date;
```

---

### Problem 6: Latest Transaction per Customer
**Problem Statement:** For each customer, display their customer ID, their most recent transaction date, and the transaction amount of that most recent transaction.

**Schema:** `transactions (txn_id, customer_id, txn_date, amount)`

**Verbal Explanation Script:**
> *"A simple `GROUP BY customer_id` with `MAX(txn_date)` gives the date, but it cannot fetch the corresponding `amount` without an extra join. The most performant and clean pattern is using `ROW_NUMBER()` partitioned by `customer_id` and ordered by `txn_date DESC, txn_id DESC` inside a CTE, then filtering for `row_num = 1`."*

**Production Solution:**
```sql
WITH RankedTransactions AS (
    SELECT 
        customer_id,
        txn_date,
        amount,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY txn_date DESC, txn_id DESC
        ) AS rn
    FROM transactions
)
SELECT 
    customer_id,
    txn_date AS latest_txn_date,
    amount AS latest_txn_amount
FROM RankedTransactions
WHERE rn = 1;
```

---

### Problem 7: Monthly Customer Retention & Churn Cohorts
**Problem Statement:** Calculate the Month-over-Month (MoM) Customer Retention Rate: For each month, what percentage of customers who placed an order in the previous month returned to place an order in the current month?

**Schema:** `orders (order_id, customer_id, order_date)`

**Verbal Explanation Script:**
> *"First, I map each order to its truncated month (`DATE_TRUNC('month', order_date)`). Then I isolate unique `(customer_id, order_month)` pairs. Using a self-join with an offset of exactly 1 month (`current_month = prior_month + 1 month`), I count retained returning customers. Finally, I divide active returning users by the total active users from the prior month."*

**Production Solution:**
```sql
WITH MonthlyActiveUsers AS (
    SELECT DISTINCT 
        customer_id,
        DATE_TRUNC('month', order_date) AS active_month
    FROM orders
),
RetentionMatches AS (
    SELECT 
        m1.active_month AS current_month,
        COUNT(DISTINCT m1.customer_id) AS current_active_users,
        COUNT(DISTINCT m2.customer_id) AS retained_from_prior_month
    FROM MonthlyActiveUsers m1
    LEFT JOIN MonthlyActiveUsers m2 
        ON m1.customer_id = m2.customer_id 
        AND m1.active_month = m2.active_month + INTERVAL '1 month'
    GROUP BY m1.active_month
)
SELECT 
    current_month,
    current_active_users,
    retained_from_prior_month,
    ROUND(
        100.0 * retained_from_prior_month / NULLIF(LAG(current_active_users) OVER (ORDER BY current_month), 0),
        2
    ) AS mom_retention_rate_pct
FROM RetentionMatches
ORDER BY current_month;
```

---

### Problem 8: Finding Inactive Users (Churned in 2024)
**Problem Statement:** Identify all customers who placed at least one order in 2023, but zero orders in 2024.

**Schema:** `orders (order_id, customer_id, order_date)`

**Verbal Explanation Script:**
> *"This can be solved cleanly using `EXCEPT` (or `MINUS` in Oracle), or using conditional aggregation with `HAVING`. The conditional aggregation approach is exceptionally fast: group by `customer_id` and require `COUNT(CASE WHEN YEAR=2023 THEN 1 END) > 0` and `COUNT(CASE WHEN YEAR=2024 THEN 1 END) = 0`."*

**Production Solution:**
```sql
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING COUNT(CASE WHEN EXTRACT(YEAR FROM order_date) = 2023 THEN 1 END) > 0
   AND COUNT(CASE WHEN EXTRACT(YEAR FROM order_date) = 2024 THEN 1 END) = 0;
```

---

### Problem 9: Duplicate Detection Across Multiple Attributes
**Problem Statement:** Detect duplicate customer registrations where `first_name`, `last_name`, and `date_of_birth` are identical. Return all columns of all duplicate records so they can be audited.

**Schema:** `customers (customer_id, first_name, last_name, date_of_birth, email, created_at)`

**Verbal Explanation Script:**
> *"Instead of just returning the duplicated keys with a `HAVING COUNT(*) > 1`, we need to return the full rows. I'll use the window function `COUNT(*) OVER (PARTITION BY first_name, last_name, date_of_birth) AS dup_count` in a CTE, then filter where `dup_count > 1`."*

**Production Solution:**
```sql
WITH DuplicateFlags AS (
    SELECT 
        customer_id,
        first_name,
        last_name,
        date_of_birth,
        email,
        created_at,
        COUNT(*) OVER (
            PARTITION BY first_name, last_name, date_of_birth
        ) AS occurrences
    FROM customers
)
SELECT *
FROM DuplicateFlags
WHERE occurrences > 1
ORDER BY last_name, first_name, created_at;
```

---

### Problem 10: Month-over-Month (MoM) Category Revenue Growth
**Problem Statement:** Compute the total monthly revenue per product category, along with the percentage growth compared to the immediately preceding month.

**Schema:** `order_items (item_id, order_id, product_id, quantity, unit_price)`, `orders (order_id, order_date)`, `products (product_id, category)`

**Verbal Explanation Script:**
> *"We join `orders`, `order_items`, and `products` to aggregate monthly revenue by category. Then, using `LAG(revenue, 1) OVER (PARTITION BY category ORDER BY sales_month)`, we retrieve the prior month's revenue and compute `100.0 * (revenue - prev_revenue) / prev_revenue`."*

**Production Solution:**
```sql
WITH MonthlyCategorySales AS (
    SELECT 
        p.category,
        DATE_TRUNC('month', o.order_date) AS sales_month,
        SUM(oi.quantity * oi.unit_price) AS monthly_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY p.category, DATE_TRUNC('month', o.order_date)
),
GrowthCalculations AS (
    SELECT 
        category,
        sales_month,
        monthly_revenue,
        LAG(monthly_revenue, 1) OVER (
            PARTITION BY category 
            ORDER BY sales_month
        ) AS prior_month_revenue
    FROM MonthlyCategorySales
)
SELECT 
    category,
    sales_month,
    monthly_revenue,
    prior_month_revenue,
    ROUND(
        100.0 * (monthly_revenue - prior_month_revenue) / NULLIF(prior_month_revenue, 0),
        2
    ) AS mom_growth_pct
FROM GrowthCalculations
ORDER BY category, sales_month;
```

---

### Problem 11: Median Salary per Department (Without Built-in MEDIAN)
**Problem Statement:** Calculate the median salary for each department in standard ANSI SQL without using dialect-specific `MEDIAN()` functions.

**Schema:** `employees (emp_id, dept_id, salary)`

**Verbal Explanation Script:**
> *"The median of a set of $N$ sorted items is the value whose index satisfies `ABS(rn_asc - rn_desc) <= 1`. By assigning two row numbers—one ordered ascending and one descending—the median values are exactly those where the ascending and descending row numbers differ by at most 1 (which handles both odd and even counts). Taking the `AVG(salary)` of these matching rows gives the exact median."*

**Production Solution:**
```sql
WITH RankedRows AS (
    SELECT 
        dept_id,
        salary,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary ASC, emp_id) AS rn_asc,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC, emp_id) AS rn_desc
    FROM employees
)
SELECT 
    dept_id,
    ROUND(AVG(salary), 2) AS median_salary
FROM RankedRows
WHERE ABS(CAST(rn_asc AS INT) - CAST(rn_desc AS INT)) <= 1
GROUP BY dept_id;
```

---

### Problem 12: Detecting Overlapping Hotel / Flight Bookings
**Problem Statement:** Find all pairs of conflicting hotel room bookings where two different reservations overlap for the exact same room.

**Schema:** `room_bookings (booking_id, room_id, guest_id, check_in_date, check_out_date)`

**Verbal Explanation Script:**
> *"Two date intervals $[S_1, E_1)$ and $[S_2, E_2)$ overlap if and only if $S_1 < E_2$ AND $S_2 < E_1$. We self-join `room_bookings` on `b1.room_id = b2.room_id` and `b1.booking_id < b2.booking_id` (to avoid comparing a booking to itself and prevent duplicate inverted pairs), adding the overlap condition."*

**Production Solution:**
```sql
SELECT 
    b1.room_id,
    b1.booking_id AS booking_1_id,
    b1.check_in_date AS b1_start,
    b1.check_out_date AS b1_end,
    b2.booking_id AS booking_2_id,
    b2.check_in_date AS b2_start,
    b2.check_out_date AS b2_end
FROM room_bookings b1
INNER JOIN room_bookings b2 
    ON b1.room_id = b2.room_id 
    AND b1.booking_id < b2.booking_id
    AND b1.check_in_date < b2.check_out_date
    AND b2.check_in_date < b1.check_out_date;
```

---

### Problem 13: Rolling 7-Day Moving Average of Daily Active Users
**Problem Statement:** Calculate the 7-day rolling average of Daily Active Users (DAU) for each calendar day.

**Schema:** `daily_active_users (metric_date DATE, dau_count INT)`

**Verbal Explanation Script:**
> *"We use `AVG(dau_count) OVER (...)` ordered by `metric_date`. To make it a true 7-day moving window that includes the current day and the preceding 6 days, we declare the frame `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`."*

**Production Solution:**
```sql
SELECT 
    metric_date,
    dau_count,
    ROUND(
        AVG(dau_count) OVER (
            ORDER BY metric_date 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ),
        1
    ) AS rolling_7day_avg_dau
FROM daily_active_users
ORDER BY metric_date;
```

---

### Problem 14: Customer Lifetime Value (LTV) & First vs. Latest Order
**Problem Statement:** For each customer, output their name, their first order date, their most recent order date, their total lifetime orders, and their total lifetime revenue.

**Schema:** `customers (customer_id, customer_name)`, `orders (order_id, customer_id, order_date, total_amount)`

**Verbal Explanation Script:**
> *"This requires standard aggregation combined with a `LEFT JOIN` so that registered customers who have never placed an order are preserved with 0 orders and 0 revenue."*

**Production Solution:**
```sql
SELECT 
    c.customer_id,
    c.customer_name,
    MIN(o.order_date) AS first_order_date,
    MAX(o.order_date) AS latest_order_date,
    COUNT(o.order_id) AS total_lifetime_orders,
    COALESCE(SUM(o.total_amount), 0.00) AS lifetime_value_revenue
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY lifetime_value_revenue DESC;
```

---

### Problem 15: Detecting Missing Numbers in an Auto-Increment Sequence
**Problem Statement:** Find all missing IDs in an `orders` table that was supposed to have a continuous sequence from 1 to the maximum `order_id`.

**Schema:** `orders (order_id INT PRIMARY KEY)`

**Verbal Explanation Script:**
> *"We use a Recursive CTE to generate all integers from 1 to `MAX(order_id)`. Then we perform a Left Anti-Join against `orders` where `orders.order_id IS NULL` to reveal all missing IDs."*

**Production Solution:**
```sql
WITH RECURSIVE FullSequence AS (
    SELECT 1 AS expected_id
    UNION ALL
    SELECT expected_id + 1 
    FROM FullSequence 
    WHERE expected_id < (SELECT MAX(order_id) FROM orders)
)
SELECT s.expected_id AS missing_order_id
FROM FullSequence s
LEFT JOIN orders o ON s.expected_id = o.order_id
WHERE o.order_id IS NULL;
```

---

# Part 3: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   SQL INTERVIEW PATTERNS CHEAT SHEET                   │
├────────────────────────────────────────────────────────────────────────┤
│ 1. Nth HIGHEST:                                                        │
│    - DENSE_RANK() OVER (ORDER BY col DESC) -> handles ties.            │
│    - Return scalar NULL: SELECT COALESCE((SELECT ... WHERE rnk=N),NULL)│
├────────────────────────────────────────────────────────────────────────┤
│ 2. TOP-N PER GROUP:                                                    │
│    - DENSE_RANK() OVER (PARTITION BY grp ORDER BY col DESC) <= N.      │
├────────────────────────────────────────────────────────────────────────┤
│ 3. GAPS & ISLANDS (Consecutive streaks):                               │
│    - Date - ROW_NUMBER() * INTERVAL '1 day' = CONSTANT ISLAND KEY.     │
├────────────────────────────────────────────────────────────────────────┤
│ 4. INTERVAL OVERLAPS:                                                  │
│    - Start_A < End_B AND Start_B < End_A.                              │
├────────────────────────────────────────────────────────────────────────┤
│ 5. MEDIAN WITHOUT BUILT-IN:                                            │
│    - ABS(ROW_NUMBER_ASC - ROW_NUMBER_DESC) <= 1.                       │
├────────────────────────────────────────────────────────────────────────┤
│ 6. RUNNING TOTAL:                                                      │
│    - SUM(val) OVER (ORDER BY dt ROWS BETWEEN UNBOUNDED PRECEDING       │
│      AND CURRENT ROW).                                                 │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 4: 100-Mark Self-Assessment Examination

### Section A: Query Prediction & Tracing (25 Marks — 5 marks each)

**Q1:** What does `DENSE_RANK()` return for scores `[100, 100, 95, 90]`?  
(a) `1, 2, 3, 4`  
(b) `1, 1, 3, 4`  
(c) `1, 1, 2, 3`  
(d) `1, 1, 1, 2`  

**Q2:** In the Gaps & Islands formula `date - ROW_NUMBER() * INTERVAL '1 day'`, why does the resulting date value remain constant across consecutive dates?  
(a) Dates are converted to Unix timestamps.  
(b) Both the calendar dates and the row numbers increment by exactly 1 per step, so their mathematical difference remains invariant.  
(c) The database optimizer caches the first date.  
(d) ROW_NUMBER() resets on every date.  

**Q3:** What is the condition for two date ranges $[S_1, E_1)$ and $[S_2, E_2)$ to overlap?  
(a) $S_1 = S_2$ OR $E_1 = E_2$  
(b) $S_1 < E_2$ AND $S_2 < E_1$  
(c) $S_1 > S_2$ AND $E_1 < E_2$  
(d) $E_1 - S_1 = E_2 - S_2$  

**Q4:** Why does `ROW_NUMBER()` always resolve tiebreaks whereas `RANK()` does not?  
(a) `ROW_NUMBER()` sorts by primary key automatically.  
(b) `ROW_NUMBER()` assigns strictly sequential unique integers ($1, 2, 3, \dots$) arbitrarily for ties.  
(c) `ROW_NUMBER()` throws an error on ties.  
(d) `RANK()` cannot be used with `ORDER BY`.  

**Q5:** What happens when evaluating `SELECT AVG(val) FROM tbl` if `tbl` contains values `[10, 20, NULL]`?  
(a) Returns 10 (divides 30 by 3).  
(b) Returns 15 (divides 30 by 2; NULL is completely ignored).  
(c) Returns NULL.  
(d) Throws a NullPointerException.  

---

### Section B: Live Query Writing (50 Marks)

**Q6 (25 Marks - Department Top Earners):**  
Given a schema `employees (id, name, salary, department_id)` and `department (id, name)`. Write a query to find the employees who have the highest salary in each department. If there is a tie for the top salary, return all tied employees. Include `department_name`, `employee_name`, and `salary`.

**Q7 (25 Marks - Retention & Consecutive Purchase):**  
Given a schema `customer_transactions (txn_id, customer_id, txn_date)`. Write a query to find all `customer_id`s who made at least one transaction on three consecutive calendar days.

---

### Section C: Query Tuning & System Scenarios (25 Marks)

**Q8 (25 Marks):** An e-commerce analyst runs the following query on a 30-million row `orders` table:
```sql
SELECT customer_id, order_date, amount
FROM (
    SELECT customer_id, order_date, amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) as rn
    FROM orders
) t
WHERE rn = 1;
```
The query takes 45 seconds to run.
1. Explain why the database execution engine is spending 90% of its time on an expensive Sort operator. (10 marks)
2. Design the exact composite B-tree index that will reduce this query's runtime to under 2 seconds. (15 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(c)** `1, 1, 2, 3`.
- **Q2:** **(b)** Invariant difference $(D + k) - k = D$.
- **Q3:** **(b)** $S_1 < E_2$ AND $S_2 < E_1$.
- **Q4:** **(b)** Arbitrary sequential integer assignment.
- **Q5:** **(b)** 15. Aggregates ignore NULLs.

### Section B Solutions
**Q6 Solution:**
```sql
WITH RankedEmployees AS (
    SELECT 
        e.name AS employee_name,
        e.salary,
        d.name AS department_name,
        DENSE_RANK() OVER (PARTITION BY e.department_id ORDER BY e.salary DESC) AS rnk
    FROM employees e
    INNER JOIN department d ON e.department_id = d.id
)
SELECT department_name, employee_name, salary
FROM RankedEmployees
WHERE rnk = 1
ORDER BY department_name, employee_name;
```

**Q7 Solution:**
```sql
WITH DistinctDays AS (
    SELECT DISTINCT customer_id, CAST(txn_date AS DATE) AS t_date
    FROM customer_transactions
),
Sequenced AS (
    SELECT 
        customer_id,
        t_date,
        t_date - CAST(ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY t_date) AS INT) * INTERVAL '1 day' AS island_id
    FROM DistinctDays
)
SELECT DISTINCT customer_id
FROM Sequenced
GROUP BY customer_id, island_id
HAVING COUNT(*) >= 3;
```

### Section C Solutions
**Q8 Solution:**
1. **Bottleneck:** The window function `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC)` forces the engine to partition and sort all 30 million rows in memory or disk TempDB ($O(N \log N)$ Sort operator).
2. **Optimal Composite Index:**
   ```sql
   CREATE INDEX idx_orders_cust_date_covering 
   ON orders (customer_id, order_date DESC) 
   INCLUDE (amount);
   ```
   - **Mechanism:** The B-tree is already physically structured by `customer_id` and pre-sorted by `order_date DESC`. The engine streams directly from the index without sorting, and `amount` is covered via `INCLUDE`, eliminating all base table key lookups.
