# Day 3: Intermediate & Advanced SQL
## Subqueries, CTEs, Window Functions & Analytical Querying

---

# Part 1: Subqueries & Derived Tables

### 1.1 Types of Subqueries
A subquery (or nested query) is an inner query embedded within an outer SQL statement.

1. **Scalar Subquery:** Returns exactly one row and one column (a single scalar value). Can be placed anywhere an expression is valid (`SELECT`, `WHERE`, `HAVING`, `ORDER BY`).
   ```sql
   -- Find employees earning above the company-wide average
   SELECT emp_name, salary
   FROM employees
   WHERE salary > (SELECT AVG(salary) FROM employees);
   ```

2. **Multi-Row Subquery:** Returns a single column containing multiple rows. Evaluated using multi-value comparison operators (`IN`, `NOT IN`, `ANY` / `SOME`, `ALL`).
   ```sql
   -- Find employees working in US-based departments
   SELECT emp_name 
   FROM employees
   WHERE dept_id IN (SELECT dept_id FROM departments WHERE location = 'USA');
   ```

3. **Multi-Column Subquery:** Returns multiple columns and multiple rows (supported in PostgreSQL, MySQL, Oracle):
   ```sql
   SELECT emp_id, dept_id, salary
   FROM employees
   WHERE (dept_id, salary) IN (
       SELECT dept_id, MAX(salary)
       FROM employees
       GROUP BY dept_id
   );
   ```

4. **Derived Table (Table Subquery in FROM):** A subquery in the `FROM` clause that acts as a temporary inline relation. Must be given an explicit table alias:
   ```sql
   SELECT dept_id, avg_sal
   FROM (
       SELECT dept_id, AVG(salary) AS avg_sal
       FROM employees
       GROUP BY dept_id
   ) AS dept_summary
   WHERE avg_sal > 90000;
   ```

---

### 1.2 Correlated Subqueries & The `EXISTS` Operator

#### Definition & Mechanics
Unlike a standard subquery (which executes once, caching its result for the outer query), a **correlated subquery** references columns from the outer query's row. It must be evaluated **once for every candidate row** processed by the outer query.

```sql
-- Find employees who earn more than the average salary of THEIR OWN department
SELECT e.emp_name, e.dept_id, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(sub.salary)
    FROM employees sub
    WHERE sub.dept_id = e.dept_id -- Correlation predicate linking inner to outer
);
```

#### `EXISTS` vs `IN` vs `JOIN` Performance Internals
- `EXISTS` checks for the **existence** of at least one row satisfying the condition. As soon as the storage engine finds a single match, evaluation terminates immediately (short-circuit boolean optimization).
- `IN` must evaluate and collect the entire set of matching values before testing membership.
- **The NULL Trap with `NOT IN` vs `NOT EXISTS`:**
  If an inner subquery evaluated by `NOT IN` returns even a single `NULL` value, the entire `NOT IN` condition evaluates to `UNKNOWN` for all rows, returning **zero results**:
  ```sql
  -- DANGEROUS: If subquery returns (1, 2, NULL), this outputs NOTHING!
  SELECT emp_name FROM employees 
  WHERE emp_id NOT IN (SELECT manager_id FROM employees);

  -- BULLETPROOF: Handles NULLs safely and short-circuits efficiently
  SELECT emp_name FROM employees e
  WHERE NOT EXISTS (
      SELECT 1 FROM employees m WHERE m.manager_id = e.emp_id
  );
  ```

---

# Part 2: Common Table Expressions (CTEs) & Recursive CTEs

### 2.1 Standard CTEs (`WITH` Clause)
A CTE defines a temporary, named result set within the execution scope of a single `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statement.

#### Advantages Over Subqueries:
1. **Readability & Modularity:** Queries read linearly from top to bottom instead of deeply nested inside-out logic.
2. **Reusability:** A CTE can be referenced multiple times within the subsequent query without recalculating or repeating code.

```sql
WITH HighEarners AS (
    SELECT emp_id, emp_name, dept_id, salary
    FROM employees
    WHERE salary > 100000
),
DeptStats AS (
    SELECT dept_id, COUNT(*) AS count_high_earners
    FROM HighEarners
    GROUP BY dept_id
)
SELECT d.dept_name, ds.count_high_earners
FROM DeptStats ds
INNER JOIN departments d ON ds.dept_id = d.dept_id;
```

---

### 2.2 Recursive CTEs
A **Recursive CTE** references itself in its own definition. It is the relational solution for processing hierarchical structures (organizational charts, bill-of-materials, network graphs, file systems) and generating series.

#### Structure of a Recursive CTE:
A recursive CTE contains two parts joined by `UNION ALL`:
1. **Anchor Member:** The base query that selects initial rows (e.g., top-level root nodes).
2. **Recursive Member:** Joins the CTE back to the base table to fetch child records until the join produces zero rows (termination condition).

#### Example 1: Traversing an Organizational Hierarchy (CEO to Direct & Indirect Reports)
```sql
WITH RECURSIVE OrgHierarchy AS (
    -- Anchor Member: Select the CEO (manager_id IS NULL)
    SELECT 
        emp_id, 
        emp_name, 
        manager_id, 
        1 AS org_level,
        CAST(emp_name AS VARCHAR(1000)) AS reporting_path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive Member: Join employees to OrgHierarchy on manager_id
    SELECT 
        e.emp_id, 
        e.emp_name, 
        e.manager_id, 
        h.org_level + 1,
        CAST(h.reporting_path || ' -> ' || e.emp_name AS VARCHAR(1000))
    FROM employees e
    INNER JOIN OrgHierarchy h ON e.manager_id = h.emp_id
)
SELECT emp_id, emp_name, org_level, reporting_path
FROM OrgHierarchy
ORDER BY org_level, emp_id;
```

#### Example 2: Generating a Date Series (Calendar Dimension Generation)
```sql
-- Generate daily dates for January 2024 (PostgreSQL / SQLite / SQL Server)
WITH RECURSIVE DateGenerator AS (
    SELECT CAST('2024-01-01' AS DATE) AS calendar_date
    UNION ALL
    SELECT calendar_date + INTERVAL '1 day'
    FROM DateGenerator
    WHERE calendar_date < '2024-01-31'
)
SELECT calendar_date FROM DateGenerator;
```

---

# Part 3: Analytical Window Functions

### 3.1 What is a Window Function?
Unlike a standard `GROUP BY` aggregation (which collapses multiple rows into a single summary row), a **Window Function** computes values over a specified window (subset) of rows **while preserving the identity and granularity of every individual row**.

```
Input: 100 rows ---> GROUP BY dept_id ---> Output: 5 rows (collapsed)
Input: 100 rows ---> Window Function  ---> Output: 100 rows (original rows retained + calculated metric appended)
```

### 3.2 Anatomy of the `OVER()` Clause
```sql
FUNCTION_NAME(...) OVER (
    PARTITION BY partition_column_1, ...
    ORDER BY sort_column_1 ASC/DESC, ...
    ROWS | RANGE BETWEEN frame_start AND frame_end
)
```

- **`PARTITION BY`:** Divides the rows into distinct groups (windows) to which the function applies. If omitted, the entire result set is treated as a single partition.
- **`ORDER BY`:** Defines the logical sequence of rows within each partition. Critical for running totals, rankings, and lead/lag operations.
- **Window Framing (`ROWS` vs `RANGE`):** Defines the exact sliding physical or logical boundary of rows evaluated relative to the current row.

---

### 3.3 The Ranking Functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`

This is universally tested in technical interviews.

| Function | Behavior with Ties (Identical Values) | Gap in Numbering Sequence? | Example on Scores: [100, 90, 90, 80] |
| :--- | :--- | :--- | :--- |
| **`ROW_NUMBER()`** | Strictly arbitrary / sequential numbering. Ties receive consecutive integers. | No | 1, 2, 3, 4 |
| **`RANK()`** | Tied rows receive identical rank. Skips subsequent ranks proportional to tie count. | **Yes** | 1, 2, 2, **4** |
| **`DENSE_RANK()`** | Tied rows receive identical rank. Does **not** skip subsequent ranks. | **No** | 1, 2, 2, **3** |
| **`NTILE(N)`** | Divides partition into $N$ roughly equal buckets (quartiles, deciles, percentiles). | No | For $N=2$: 1, 1, 2, 2 |

#### Visual Comparison Query:
```sql
SELECT 
    emp_name, 
    dept_id, 
    salary,
    ROW_NUMBER() OVER(PARTITION BY dept_id ORDER BY salary DESC) AS row_num,
    RANK()       OVER(PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER(PARTITION BY dept_id ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

---

### 3.4 Value Navigation Functions: `LAG` and `LEAD`

#### Syntax & Parameters:
- `LAG(expression, [offset], [default_value]) OVER (...)`: Fetches value from a row $N$ steps prior to the current row.
- `LEAD(expression, [offset], [default_value]) OVER (...)`: Fetches value from a row $N$ steps after the current row.

```sql
SELECT 
    customer_id,
    order_date,
    total_amount,
    LAG(total_amount, 1, 0.00) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) AS previous_order_amount,
    (total_amount - LAG(total_amount, 1, total_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    )) AS spend_delta_vs_last_order
FROM orders;
```

#### First and Last Values:
- `FIRST_VALUE(col) OVER (...)`: Returns first value in the current window frame.
- `LAST_VALUE(col) OVER (...)`: Returns last value in the current window frame.
> [!WARNING]
> By default, adding `ORDER BY` to an `OVER` clause sets the default frame to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This causes `LAST_VALUE()` to return the current row's value rather than the partition's ultimate value! To fix this, explicitly specify `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

---

# Part 4: Window Framing Specifications (`ROWS` vs `RANGE`)

### 4.1 Physical vs Logical Framing
- **`ROWS` (Physical Framing):** Considers exact physical offsets of rows.
- **`RANGE` (Logical Framing):** Considers value offsets according to the `ORDER BY` value (groups duplicate values together).

### 4.2 Standard Frame Declarations
```sql
-- 1. True Cumulative Running Total (From start of partition to current row)
SUM(amount) OVER (
    PARTITION BY account_id 
    ORDER BY txn_date 
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)

-- 2. 7-Day / 3-Row Rolling Moving Average (Current row and preceding 2 rows)
AVG(amount) OVER (
    PARTITION BY account_id 
    ORDER BY txn_date 
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)

-- 3. Centered Moving Average (Preceding 1 row, current row, following 1 row)
AVG(amount) OVER (
    ORDER BY txn_date 
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
)

-- 4. Full Partition Total (Without collapsing rows)
SUM(amount) OVER (
    PARTITION BY account_id 
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

---

# Part 5: Classic Interview Problem Archetypes

### Archetype 1: Second-Highest Salary
Interviewers test this to see if you understand `NULL` edge cases and multiple salary ties.

#### Approach 1: Window Function CTE (`DENSE_RANK`) — Recommended
```sql
WITH RankedSalaries AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT COALESCE(
    (SELECT DISTINCT salary FROM RankedSalaries WHERE rnk = 2), 
    NULL
) AS SecondHighestSalary;
```
*Why DENSE_RANK?* If the two highest employees both make $150k, `RANK()` gives them (1, 1, 3). The second distinct salary ($120k) is rank 2 in `DENSE_RANK()`.

#### Approach 2: Correlated Subquery / Scalar Filter
```sql
SELECT MAX(salary) AS SecondHighestSalary
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

#### Approach 3: LIMIT and OFFSET
```sql
SELECT DISTINCT salary AS SecondHighestSalary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```
*Limitation of Approach 3:* If there are fewer than 2 distinct salaries, Approach 3 returns an empty set instead of `NULL` unless wrapped in an outer scalar query.

---

### Archetype 2: Top-N Records per Group (Top 3 Salaries per Department)
```sql
WITH DepartmentSalaries AS (
    SELECT 
        e.emp_id,
        e.emp_name,
        e.salary,
        d.dept_name,
        DENSE_RANK() OVER (
            PARTITION BY e.dept_id 
            ORDER BY e.salary DESC
        ) AS salary_rank
    FROM employees e
    INNER JOIN departments d ON e.dept_id = d.dept_id
)
SELECT dept_name, emp_name, salary, salary_rank
FROM DepartmentSalaries
WHERE salary_rank <= 3
ORDER BY dept_name, salary_rank, emp_name;
```

---

### Archetype 3: Latest Transaction per Customer
```sql
WITH RankedOrders AS (
    SELECT 
        order_id,
        customer_id,
        order_date,
        total_amount,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id 
            ORDER BY order_date DESC, order_id DESC
        ) AS seq
    FROM orders
)
SELECT order_id, customer_id, order_date, total_amount
FROM RankedOrders
WHERE seq = 1;
```

---

### Archetype 4: Gaps & Islands (Finding Consecutive Activity Days)
Given a table of user logins, identify streaks of consecutive active days:

```sql
WITH DistinctLogins AS (
    -- Step 1: Deduplicate multiple logins on the same day
    SELECT DISTINCT user_id, CAST(login_time AS DATE) AS login_date
    FROM user_activity
),
DatedSequences AS (
    -- Step 2: Subtract ROW_NUMBER() days from login_date to find constant anchor date
    SELECT 
        user_id,
        login_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn,
        login_date - CAST(ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS INT) * INTERVAL '1 day' AS island_group
    FROM DistinctLogins
)
-- Step 3: Group by user and island_group to measure streak length
SELECT 
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS consecutive_days
FROM DatedSequences
GROUP BY user_id, island_group
HAVING COUNT(*) >= 3
ORDER BY consecutive_days DESC;
```

*Mathematical Intuition:* If dates are consecutive ($D, D+1, D+2$) and row numbers are consecutive ($1, 2, 3$), the difference $(D + k) - k = D$ remains completely constant for all consecutive dates! When a gap occurs, the constant shifts.

---

# Part 6: Logical Execution Order with Window Functions

Where do Window Functions fit in the engine pipeline?

```
1. FROM / ON / JOIN
2. WHERE
3. GROUP BY
4. HAVING
5. WINDOW FUNCTIONS (Evaluated here, alongside projection)
6. SELECT (Expressions, aliases, projection)
7. DISTINCT
8. ORDER BY
9. LIMIT / OFFSET
```

> [!IMPORTANT]
> **Can you put a Window Function in the WHERE clause?**
> **NO.** Because `WHERE` (Step 2) executes before Window Functions (Step 5), writing `WHERE ROW_NUMBER() OVER(...) = 1` causes an immediate syntax error. You **must** compute the window function inside a CTE or derived table first, and then filter in the outer query's `WHERE` clause.

---

# Part 7: Comprehensive Comparison Guide

### Comparison 1: CTE vs Subquery vs Temporary Table vs View
| Structure | Scope / Lifespan | Stored on Disk? | Indexed? | Best Used For |
| :--- | :--- | :--- | :--- | :--- |
| **CTE** | Single statement | In-memory (transient) | No | Modularizing complex single-query pipelines, recursion. |
| **Subquery** | Single statement / clause | In-memory | No | Simple scalar comparisons or quick exists checks. |
| **Temp Table (`#table`)**| Session / Connection | TempDB / Disk | **Yes** (Supports indexes & statistics) | Staging millions of intermediate rows across multiple pipeline steps. |
| **View** | Permanent database object | Schema only (metadata) | Indexed views only | Standardized security, abstracting complex joins for analysts. |

---

# Part 8: Structured Interview Question Bank

### Q1: What is the difference between `EXISTS` and `IN` when evaluating subqueries containing NULL values?
**Answer:**
`IN` expands to a series of `OR` comparisons (`x = val1 OR x = val2 ...`). If the subquery contains a `NULL`, `x IN (1, NULL)` evaluates to `TRUE` if $x=1$, but `UNKNOWN` if $x \ne 1$. Consequently, `NOT IN` will fail completely if any returned value is `NULL`. In contrast, `EXISTS` checks solely for row presence (`count > 0`); `NULL` rows inside `EXISTS` evaluate as valid rows, so `EXISTS` and `NOT EXISTS` handle null-laden tables safely without returning unexpected empty sets.

### Q2: How does `COUNT(*)` differ from `COUNT(col)` when used as a window function?
**Answer:**
- `COUNT(*) OVER (PARTITION BY dept_id)` counts all rows in the partition, regardless of whether individual columns contain `NULL`s.
- `COUNT(col) OVER (PARTITION BY dept_id)` counts only rows where `col IS NOT NULL`.

### Q3: What is the default window frame when `ORDER BY` is specified inside `OVER()`, and why does it matter?
**Answer:**
When `ORDER BY` is present, ANSI SQL defaults the frame to:
`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.
This matters for two reasons:
1. `SUM(val) OVER (ORDER BY dt)` will compute a running cumulative sum rather than the total partition sum.
2. If duplicate values exist in the `ORDER BY` column, `RANGE` evaluates duplicates simultaneously, causing the running sum to jump rather than increment smoothly row-by-row (as `ROWS` would).

### Q4: Explain the difference between `NTILE(4)` and percentile functions.
**Answer:**
`NTILE(4)` divides the ordered rows into 4 equal cardinality bins ($1, 2, 3, 4$), assigning row counts as evenly as possible. Percentile functions like `PERCENT_RANK()` or `CUME_DIST()` compute a relative fractional rank between $0.0$ and $1.0$ based on value distribution rather than discrete row bucket indices.

---

# Part 9: Hands-On Practice Lab (20 Problems)

### Database Initialization Script
```sql
DROP TABLE IF EXISTS sales_performance;
DROP TABLE IF EXISTS project_assignments;
DROP TABLE IF EXISTS staff;
DROP TABLE IF EXISTS divisions;

CREATE TABLE divisions (
    div_id INT PRIMARY KEY,
    div_name VARCHAR(50),
    headquarters VARCHAR(50)
);

CREATE TABLE staff (
    staff_id INT PRIMARY KEY,
    name VARCHAR(50),
    div_id INT,
    role VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE,
    FOREIGN KEY (div_id) REFERENCES divisions(div_id)
);

CREATE TABLE sales_performance (
    sale_id INT PRIMARY KEY,
    staff_id INT,
    sale_date DATE,
    amount DECIMAL(10,2),
    region VARCHAR(30),
    FOREIGN KEY (staff_id) REFERENCES staff(staff_id)
);

-- Seed Data
INSERT INTO divisions VALUES 
(1, 'Cloud Services', 'Seattle'),
(2, 'Enterprise AI', 'San Francisco'),
(3, 'Cyber Security', 'Austin');

INSERT INTO staff VALUES
(101, 'Alex Rivera', 1, 'Senior Architect', 150000.00, '2020-03-15'),
(102, 'Brenda Vance', 1, 'Cloud Engineer', 110000.00, '2021-06-01'),
(103, 'Carlos Ortiz', 1, 'Cloud Engineer', 110000.00, '2022-01-15'),
(104, 'Diana Prince', 2, 'Research Scientist', 160000.00, '2019-11-01'),
(105, 'Evan Wright', 2, 'ML Engineer', 130000.00, '2021-02-20'),
(106, 'Fiona Gallagher', 2, 'ML Engineer', 130000.00, '2023-08-10'),
(107, 'George Clark', 3, 'Security Analyst', 95000.00, '2022-05-12'),
(108, 'Hannah Abbott', 3, 'Security Analyst', 95000.00, '2023-01-10');

INSERT INTO sales_performance VALUES
(1, 101, '2024-01-05', 12000.00, 'West'),
(2, 101, '2024-01-20', 18000.00, 'West'),
(3, 102, '2024-01-15', 7500.00, 'West'),
(4, 102, '2024-02-10', 9200.00, 'West'),
(5, 103, '2024-01-28', 11000.00, 'West'),
(6, 104, '2024-01-10', 25000.00, 'North'),
(7, 104, '2024-02-18', 30000.00, 'North'),
(8, 105, '2024-01-22', 14000.00, 'East'),
(9, 105, '2024-02-25', 16000.00, 'East'),
(10, 106, '2024-02-05', 8000.00, 'East'),
(11, 107, '2024-01-18', 6000.00, 'South'),
(12, 107, '2024-02-12', 7200.00, 'South');
```

---

### Practice Problem Statements (Try Before Looking at Answers!)

1. **Company Average Comparison:** Find staff names who earn strictly more than the company-wide average salary.
2. **Division Peer Comparison:** Find staff members who earn more than the average salary of their respective division.
3. **Division Salary Ranks:** Assign `ROW_NUMBER`, `RANK`, and `DENSE_RANK` to staff within each division based on salary descending.
4. **Second Highest Salary Overall:** Find the second highest distinct salary across all staff members.
5. **Top Earner per Division:** Identify the top earner in each division using a window CTE. (If tied, display both).
6. **Cumulative Division Payroll:** For each division, list staff ordered by `hire_date` along with a running total of salaries paid up to that hire.
7. **Salary Delta to Top Earner:** For each staff member, show their salary and the difference between their salary and the maximum salary in their division.
8. **Previous Sale Comparison:** For each sale made by Alex Rivera (staff_id = 101), show the sale date, amount, and the amount of the immediately preceding sale.
9. **Sales Growth Percentage:** Calculate the percentage growth in sale amount between consecutive sales for each staff member.
10. **Rolling 2-Sale Moving Average:** Compute a moving average of sale amounts considering the current sale and the immediately prior sale for each staff member.
11. **Quartile Salary Buckets:** Divide all staff into 4 compensation quartiles using `NTILE(4)`.
12. **First and Most Recent Sale:** For each staff member, display their earliest and latest sale date side-by-side on each sales row.
13. **Correlated Existence:** List all divisions that have at least one employee earning over $140,000 using `EXISTS`.
14. **Anti-Join with Window Context:** Find staff members who have never generated a sale using a subquery.
15. **Top 2 Sales per Region:** Find the top 2 highest sale amounts for each region.
16. **Sales Contribution Ratio:** For each sale, show the percentage contribution it made to the staff member's total lifetime sales.
17. **Dense Rank Without Gaps:** Find staff members holding the 2nd lowest distinct salary across the entire company.
18. **Subquery in SELECT:** Display staff name, salary, and the difference from the lowest salary in the entire company without using window functions.
19. **Lead Sale Date Difference:** Calculate the number of days between each sale and the subsequent sale for staff member 101.
20. **Recursive Number Generation:** Use a recursive CTE to output numbers from 1 to 10.

---

# Part 10: Solutions & In-Depth Walkthroughs

### Solution 1
```sql
SELECT name, salary
FROM staff
WHERE salary > (SELECT AVG(salary) FROM staff);
```

### Solution 2
```sql
SELECT s.name, s.div_id, s.salary
FROM staff s
WHERE s.salary > (
    SELECT AVG(sub.salary)
    FROM staff sub
    WHERE sub.div_id = s.div_id
);
```

### Solution 3
```sql
SELECT 
    div_id, 
    name, 
    salary,
    ROW_NUMBER() OVER(PARTITION BY div_id ORDER BY salary DESC) AS seq_num,
    RANK() OVER(PARTITION BY div_id ORDER BY salary DESC) AS rnk,
    DENSE_RANK() OVER(PARTITION BY div_id ORDER BY salary DESC) AS dense_rnk
FROM staff;
```

### Solution 4
```sql
WITH RankedSalaries AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM staff
)
SELECT MAX(salary) AS second_highest_salary
FROM RankedSalaries
WHERE rnk = 2;
```

### Solution 5
```sql
WITH RankedStaff AS (
    SELECT 
        s.name, 
        d.div_name, 
        s.salary,
        DENSE_RANK() OVER (PARTITION BY s.div_id ORDER BY s.salary DESC) AS rnk
    FROM staff s
    INNER JOIN divisions d ON s.div_id = d.div_id
)
SELECT div_name, name, salary
FROM RankedStaff
WHERE rnk = 1;
```

### Solution 6
```sql
SELECT 
    div_id, 
    name, 
    hire_date, 
    salary,
    SUM(salary) OVER (
        PARTITION BY div_id 
        ORDER BY hire_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_payroll
FROM staff;
```

### Solution 7
```sql
SELECT 
    name, 
    div_id, 
    salary,
    MAX(salary) OVER (PARTITION BY div_id) AS div_max_salary,
    (MAX(salary) OVER (PARTITION BY div_id) - salary) AS shortfall_from_top
FROM staff;
```

### Solution 8
```sql
SELECT 
    sale_date, 
    amount,
    LAG(amount, 1) OVER (ORDER BY sale_date) AS prev_sale_amount
FROM sales_performance
WHERE staff_id = 101;
```

### Solution 9
```sql
SELECT 
    staff_id, 
    sale_date, 
    amount,
    ROUND(
        100.0 * (amount - LAG(amount, 1) OVER(PARTITION BY staff_id ORDER BY sale_date)) 
        / LAG(amount, 1) OVER(PARTITION BY staff_id ORDER BY sale_date), 
        2
    ) AS pct_growth
FROM sales_performance;
```

### Solution 10
```sql
SELECT 
    staff_id, 
    sale_date, 
    amount,
    ROUND(
        AVG(amount) OVER (
            PARTITION BY staff_id 
            ORDER BY sale_date 
            ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
        ), 
        2
    ) AS rolling_2_sale_avg
FROM sales_performance;
```

### Solution 11
```sql
SELECT 
    name, 
    salary,
    NTILE(4) OVER (ORDER BY salary DESC) AS salary_quartile
FROM staff;
```

### Solution 12
```sql
SELECT 
    staff_id,
    sale_id,
    sale_date,
    FIRST_VALUE(sale_date) OVER(
        PARTITION BY staff_id 
        ORDER BY sale_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_sale_date,
    LAST_VALUE(sale_date) OVER(
        PARTITION BY staff_id 
        ORDER BY sale_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS latest_sale_date
FROM sales_performance;
```

### Solution 13
```sql
SELECT d.div_name
FROM divisions d
WHERE EXISTS (
    SELECT 1 FROM staff s 
    WHERE s.div_id = d.div_id AND s.salary > 140000
);
```

### Solution 14
```sql
SELECT name, role 
FROM staff
WHERE staff_id NOT IN (
    SELECT DISTINCT staff_id FROM sales_performance
);
```

### Solution 15
```sql
WITH RegionalRanks AS (
    SELECT 
        region, 
        sale_id, 
        amount,
        DENSE_RANK() OVER (PARTITION BY region ORDER BY amount DESC) AS rnk
    FROM sales_performance
)
SELECT region, sale_id, amount
FROM RegionalRanks
WHERE rnk <= 2;
```

### Solution 16
```sql
SELECT 
    sale_id, 
    staff_id, 
    amount,
    ROUND(
        100.0 * amount / SUM(amount) OVER (PARTITION BY staff_id), 
        2
    ) AS pct_of_personal_total
FROM sales_performance;
```

### Solution 17
```sql
WITH LowestSalaries AS (
    SELECT name, salary, DENSE_RANK() OVER (ORDER BY salary ASC) AS asc_rnk
    FROM staff
)
SELECT name, salary
FROM LowestSalaries
WHERE asc_rnk = 2;
```

### Solution 18
```sql
SELECT 
    name, 
    salary,
    (salary - (SELECT MIN(salary) FROM staff)) AS spread_from_minimum
FROM staff;
```

### Solution 19
```sql
SELECT 
    sale_date,
    LEAD(sale_date, 1) OVER (ORDER BY sale_date) AS next_sale_date,
    (LEAD(sale_date, 1) OVER (ORDER BY sale_date) - sale_date) AS days_until_next_sale
FROM sales_performance
WHERE staff_id = 101;
```

### Solution 20
```sql
WITH RECURSIVE Seq AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM Seq WHERE n < 10
)
SELECT n FROM Seq;
```

---

# Part 11: Production Enterprise Case Study: Customer Churn & Cohort Retention

### Business Challenge
At **NexusFintech**, leadership needs to measure customer transaction velocity. The data engineering team must compute:
1. Days elapsed between each customer's consecutive transactions.
2. Flagging customers who exhibit "churn risk" (defined as $> 45$ days between transactions).
3. Identifying customers with consecutive transaction amounts that increase 3 times in a row.

### Production Solution
```sql
WITH TransactionLags AS (
    SELECT 
        customer_id,
        txn_id,
        txn_date,
        amount,
        LAG(txn_date, 1) OVER (
            PARTITION BY customer_id 
            ORDER BY txn_date
        ) AS prior_txn_date,
        LAG(amount, 1) OVER (
            PARTITION BY customer_id 
            ORDER BY txn_date
        ) AS prior_amount
    FROM customer_transactions
),
Deltas AS (
    SELECT 
        customer_id,
        txn_id,
        txn_date,
        amount,
        COALESCE(txn_date - prior_txn_date, 0) AS days_since_last_txn,
        CASE WHEN (txn_date - prior_txn_date) > 45 THEN 1 ELSE 0 END AS is_churn_warning,
        CASE WHEN amount > prior_amount THEN 1 ELSE 0 END AS is_increasing_spend
    FROM TransactionLags
)
SELECT 
    customer_id,
    COUNT(*) AS total_txns,
    ROUND(AVG(days_since_last_txn), 1) AS avg_cadence_days,
    SUM(is_churn_warning) AS churn_warnings_count,
    MAX(amount) AS highest_single_purchase
FROM Deltas
GROUP BY customer_id
ORDER BY churn_warnings_count DESC, total_txns DESC;
```

---

# Part 12: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ADVANCED SQL & WINDOW FUNCTIONS CHEAT SHEET             │
├────────────────────────────────────────────────────────────────────────────┤
│ 1. SUBQUERY: Query inside query. Scalar (1x1), Multi-row (IN/ANY/ALL).     │
│ 2. CORRELATED SUBQUERY: References outer table; runs once per outer row.  │
│ 3. EXISTS vs IN: EXISTS terminates on first match; handles NULL safely.    │
│ 4. CTE: WITH Name AS (SELECT ...) -> Linear, readable, modular.           │
│ 5. RECURSIVE CTE: Anchor Query UNION ALL Recursive Query (Hierarchies).   │
├────────────────────────────────────────────────────────────────────────────┤
│                           WINDOW RANKING FUNCTIONS                         │
│  - ROW_NUMBER() : Unique sequential (1, 2, 3, 4). No ties.                │
│  - RANK()       : Ties share number; skips next (1, 2, 2, 4).              │
│  - DENSE_RANK() : Ties share number; DOES NOT skip (1, 2, 2, 3).          │
│  - NTILE(k)     : Distributes rows into k roughly equal buckets.           │
├────────────────────────────────────────────────────────────────────────────┤
│                           VALUE & OFFSET FUNCTIONS                         │
│  - LAG(col, n, def)  : Value n rows before current.                        │
│  - LEAD(col, n, def) : Value n rows after current.                         │
│  - FIRST_VALUE(col)  : First value in frame.                               │
│  - LAST_VALUE(col)   : Last value in frame (requires ROWS BETWEEN).        │
├────────────────────────────────────────────────────────────────────────────┤
│                               WINDOW FRAMES                                │
│  - ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW (Running Total)        │
│  - ROWS BETWEEN 2 PRECEDING AND CURRENT ROW (Moving Avg of 3)              │
│  - ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING (Full Partition)│
├────────────────────────────────────────────────────────────────────────────┤
│  CRITICAL RULE: Window functions execute in Step 5 (Projection time).      │
│  CANNOT be used directly in WHERE or HAVING clauses!                       │
└────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 13: 100-Mark Self-Assessment Examination

### Section A: Technical Conceptual (20 Marks — 4 marks each)

**Q1:** What is the primary difference between `RANK()` and `DENSE_RANK()` when three rows tie for 1st place?  
(a) `RANK()` assigns 1, 1, 1 and the next row is 2; `DENSE_RANK()` assigns 1, 1, 1 and the next row is 4.  
(b) `RANK()` assigns 1, 1, 1 and the next row is 4; `DENSE_RANK()` assigns 1, 1, 1 and the next row is 2.  
(c) `RANK()` assigns 1, 2, 3; `DENSE_RANK()` assigns 1, 1, 1.  
(d) `RANK()` throws an ambiguous tie error.  

**Q2:** Why does `SELECT * FROM tbl WHERE ROW_NUMBER() OVER (ORDER BY id) <= 10;` cause an execution failure?  
(a) `ROW_NUMBER` cannot be sorted by `id`.  
(b) Window functions are evaluated after `WHERE` in the logical execution order.  
(c) `LIMIT` must be used instead of `ROW_NUMBER`.  
(d) `OVER()` requires a `PARTITION BY` clause.  

**Q3:** What happens if `subquery` in `WHERE col NOT IN (SELECT subquery)` returns one or more `NULL` values?  
(a) The query ignores the NULLs and checks the remaining values.  
(b) The condition evaluates to UNKNOWN, returning zero rows.  
(c) A runtime null pointer exception is thrown.  
(d) All rows are returned.  

**Q4:** What is the termination requirement for a Recursive CTE?  
(a) An explicit `LIMIT` clause in the recursive query.  
(b) The recursive member must eventually return an empty result set.  
(c) A `MAX_RECURSION` global database variable must be set.  
(d) The anchor member must return exactly one row.  

**Q5:** Which window frame specification ensures `LAST_VALUE(val)` evaluates across the complete partition rather than terminating at the current row?  
(a) `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`  
(b) `RANGE BETWEEN CURRENT ROW AND CURRENT ROW`  
(c) `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`  
(d) `ROWS 1 PRECEDING`  

---

### Section B: Output Prediction (25 Marks)

#### Given Table `scores`:
| player_id | score |
| :--- | :--- |
| A | 100 |
| B | 90 |
| C | 90 |
| D | 80 |
| E | NULL |

**Q6 (10 Marks):** What are the exact values of `rnk` and `dense_rnk` for each player?
```sql
SELECT player_id, score,
    RANK() OVER (ORDER BY score DESC NULLS LAST) AS rnk,
    DENSE_RANK() OVER (ORDER BY score DESC NULLS LAST) AS dense_rnk
FROM scores;
```

**Q7 (15 Marks):** What is the exact value of `running_sum` for each row?
```sql
SELECT player_id, score,
    SUM(COALESCE(score, 0)) OVER (
        ORDER BY score DESC NULLS LAST 
        ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
    ) AS running_sum
FROM scores;
```

---

### Section C: Query Implementation (35 Marks)

**Q8 (15 Marks):** Write a query to find the 3rd highest distinct salary from an `employees` table without using `LIMIT` or `OFFSET`. If no 3rd highest salary exists, return `NULL`.

**Q9 (20 Marks):** Write a query using a Recursive CTE to traverse an employee hierarchy table `emp (emp_id, name, manager_id)` and display each employee's name along with their complete management hierarchy depth level (CEO = 1, their direct reports = 2, etc.).

---

### Section D: Architectural & Tuning Scenario (20 Marks)

**Q10 (20 Marks):** An ETL developer wrote a query using 4 nested subqueries in the `WHERE` clause that takes 35 minutes to complete on a 10M row table.
1. Explain why rewriting the subqueries as CTEs or Joins might allow the optimizer to produce a faster execution plan. (10 marks)
2. How does indexing the partition and order keys affect window function execution? (10 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** `RANK()` skips ($1, 1, 1, 4$); `DENSE_RANK()` does not skip ($1, 1, 1, 2$).
- **Q2:** **(b)** Window functions are evaluated in the SELECT projection stage, which occurs after WHERE filtering.
- **Q3:** **(b)** `NOT IN` with any `NULL` evaluates to `UNKNOWN`, producing an empty result set.
- **Q4:** **(b)** The recursive member must return 0 rows to terminate the loop.
- **Q5:** **(c)** `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

### Section B Solutions
**Q6 Output:**
| player_id | score | rnk | dense_rnk |
| :--- | :--- | :--- | :--- |
| A | 100 | 1 | 1 |
| B | 90 | 2 | 2 |
| C | 90 | 2 | 2 |
| D | 80 | 4 | 3 |
| E | NULL | 5 | 4 |

**Q7 Output:**
| player_id | score | running_sum | Explanation |
| :--- | :--- | :--- | :--- |
| A | 100 | 100 | 100 + 0 preceding = 100 |
| B | 90 | 190 | 90 + 100 = 190 |
| C | 90 | 180 | 90 + 90 = 180 |
| D | 80 | 170 | 80 + 90 = 170 |
| E | NULL | 80 | 0 + 80 = 80 |

### Section C Solutions
**Q8 Solution:**
```sql
WITH RankedSalaries AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT COALESCE(
    (SELECT DISTINCT salary FROM RankedSalaries WHERE rnk = 3),
    NULL
) AS third_highest_salary;
```

**Q9 Solution:**
```sql
WITH RECURSIVE EmpHierarchy AS (
    SELECT emp_id, name, manager_id, 1 AS depth_level
    FROM emp
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.emp_id, e.name, e.manager_id, h.depth_level + 1
    FROM emp e
    INNER JOIN EmpHierarchy h ON e.manager_id = h.emp_id
)
SELECT emp_id, name, depth_level
FROM EmpHierarchy
ORDER BY depth_level, emp_id;
```

### Section D Solutions
**Q10 Solution:**
1. Nested subqueries in `WHERE` often trigger repetitive nested-loop scans ($O(M \times N)$). Rewriting them as CTEs or explicit `INNER/LEFT JOIN`s enables the optimizer to use Hash Joins ($O(M+N)$) or materialize the subquery results once rather than re-evaluating row-by-row.
2. If an index covers `(partition_col, order_col)`, the storage engine can stream pre-sorted records directly into the window accumulator. Without an index, the query planner must execute an expensive disk or memory sort ($O(N \log N)$ Sort operator) for every window partition.
