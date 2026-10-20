# Day 2: SQL JOINs, Grouping & Aggregation
## Comprehensive Study Notes & Interview Preparation Guide

---

# Part 1: Relational Algebra & The Mechanics of JOINs

### 1.1 Definition & Relational Intuition
In relational database systems (RDBMS), data is normalized across multiple tables to reduce redundancy, prevent update anomalies, and enforce domain integrity. A **JOIN** is a binary relational operator that reconstructs relationships between tables by combining tuples (rows) from two or more relations based on a logical predicate known as the **join condition** (typically equality between primary keys and foreign keys).

### 1.2 Cartesian Product (CROSS JOIN)
Before exploring filtered joins, understand the theoretical foundation: the **Cartesian Product** ($R \times S$).
If Table $R$ has $M$ rows and Table $S$ has $N$ rows, the Cartesian Product produces a result relation containing $M \times N$ rows, with every row of $R$ concatenated with every row of $S$.

```sql
-- Explicit ANSI syntax
SELECT e.emp_name, d.dept_name
FROM employees e
CROSS JOIN departments d;

-- Implicit syntax (discouraged due to accidental omission of WHERE clause)
SELECT e.emp_name, d.dept_name
FROM employees e, departments d;
```

#### Practical Use Cases for CROSS JOIN:
1. **Generating Continuous Time/Date Grids:** Combining an active entity list with a complete calendar dimension table to detect zero-activity days (preventing missing rows in time-series reporting).
2. **Matrix Generation:** Testing all permutations of product sizes and colors for an inventory staging table.

---

# Part 2: Comprehensive Breakdown of Standard JOIN Types

### 2.1 The Reference Schema for Explanations
To ensure every join behavior is concrete and unambiguous, we use this exact reference dataset:

**Table: `departments` (`d`)**
| dept_id | dept_name | location |
| :--- | :--- | :--- |
| 10 | Engineering | New York |
| 20 | Analytics | San Francisco |
| 30 | Marketing | London |
| 40 | Operations | Dublin |

**Table: `employees` (`e`)**
| emp_id | emp_name | dept_id | salary | manager_id |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Alice | 10 | 120000 | NULL |
| 2 | Bob | 10 | 95000 | 1 |
| 3 | Charlie | 20 | 105000 | 1 |
| 4 | David | 20 | 88000 | 3 |
| 5 | Emma | NULL | 92000 | 1 |
| 6 | Frank | 50 | 75000 | 3 |

*Notice:*
- Emma has a `NULL` `dept_id` (Unassigned department).
- Frank has `dept_id = 50` which does **not** exist in `departments` (Orphan/Dangling foreign key).
- Departments 30 (Marketing) and 40 (Operations) have **no employees assigned**.

---

### 2.2 INNER JOIN
#### Definition & Logic
An `INNER JOIN` evaluates the join predicate (specified in the `ON` clause) and returns **only** the rows where the condition evaluates to `TRUE`. Rows with unmatched keys or `NULL` keys are completely excluded.

```sql
SELECT 
    e.emp_id, 
    e.emp_name, 
    d.dept_id, 
    d.dept_name
FROM employees e
INNER JOIN departments d 
    ON e.dept_id = d.dept_id;
```

#### Result Set:
| emp_id | emp_name | dept_id | dept_name |
| :--- | :--- | :--- | :--- |
| 1 | Alice | 10 | Engineering |
| 2 | Bob | 10 | Engineering |
| 3 | Charlie | 20 | Analytics |
| 4 | David | 20 | Analytics |

*Omissions Explained:*
- **Emma** is excluded because `NULL = 10`, `NULL = 20`, etc. evaluate to `UNKNOWN` in SQL Three-Valued Logic.
- **Frank** is excluded because `50` does not match any row in `departments`.
- **Departments 30 & 40** are excluded because no employee has `dept_id IN (30, 40)`.

---

### 2.3 LEFT OUTER JOIN (LEFT JOIN)
#### Definition & Logic
A `LEFT JOIN` returns **all rows from the left table**, along with matched rows from the right table. If no match exists in the right table for a given left row, the engine generates **NULL-padded** columns for all selected attributes from the right table.

```sql
SELECT 
    e.emp_id, 
    e.emp_name, 
    e.dept_id AS emp_dept, 
    d.dept_id AS dept_dept, 
    d.dept_name
FROM employees e
LEFT JOIN departments d 
    ON e.dept_id = d.dept_id;
```

#### Result Set:
| emp_id | emp_name | emp_dept | dept_dept | dept_name |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Alice | 10 | 10 | Engineering |
| 2 | Bob | 10 | 10 | Engineering |
| 3 | Charlie | 20 | 20 | Analytics |
| 4 | David | 20 | 20 | Analytics |
| 5 | Emma | NULL | NULL | NULL |
| 6 | Frank | 50 | NULL | NULL |

#### The Left Anti-Join Pattern (Finding Unmatched Records)
To find entities in the left table that have **no counterpart** in the right table, pair a `LEFT JOIN` with a `WHERE right_table.primary_key IS NULL`:

```sql
-- Employees who belong to an invalid or unassigned department
SELECT e.emp_id, e.emp_name, e.dept_id
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;
```
*Result:* Returns Emma and Frank.

---

### 2.4 RIGHT OUTER JOIN (RIGHT JOIN)
#### Definition & Logic
Symmetric counterpart to the `LEFT JOIN`. Returns all rows from the right table and matching rows from the left table. Any unmatched right table row is padded with `NULL`s for left-table attributes.

```sql
SELECT 
    e.emp_id, 
    e.emp_name, 
    d.dept_id, 
    d.dept_name
FROM employees e
RIGHT JOIN departments d 
    ON e.dept_id = d.dept_id;
```

#### Result Set:
| emp_id | emp_name | dept_id | dept_name |
| :--- | :--- | :--- | :--- |
| 1 | Alice | 10 | Engineering |
| 2 | Bob | 10 | Engineering |
| 3 | Charlie | 20 | Analytics |
| 4 | David | 20 | Analytics |
| NULL | NULL | 30 | Marketing |
| NULL | NULL | 40 | Operations |

> [!TIP]
> **Engineering Standard:** Almost all senior data engineers convert `RIGHT JOIN`s to `LEFT JOIN`s by swapping the table order (`FROM departments d LEFT JOIN employees e`). This maintains a standard left-to-right reading flow and improves code review ergonomics.

---

### 2.5 FULL OUTER JOIN
#### Definition & Logic
Returns all records when there is a match in either left or right table records. Unmatched rows from either side are returned with `NULL` in the opposing table's attributes.

```sql
-- Standard ANSI SQL (PostgreSQL, SQL Server, Snowflake, Oracle)
SELECT 
    e.emp_id, 
    e.emp_name, 
    d.dept_id, 
    d.dept_name
FROM employees e
FULL OUTER JOIN departments d 
    ON e.dept_id = d.dept_id;
```

#### Result Set:
| emp_id | emp_name | dept_id | dept_name |
| :--- | :--- | :--- | :--- |
| 1 | Alice | 10 | Engineering |
| 2 | Bob | 10 | Engineering |
| 3 | Charlie | 20 | Analytics |
| 4 | David | 20 | Analytics |
| 5 | Emma | NULL | NULL |
| 6 | Frank | NULL | NULL |
| NULL | NULL | 30 | Marketing |
| NULL | NULL | 40 | Operations |

#### Dialect Note: Emulating FULL OUTER JOIN in MySQL
MySQL does not support native `FULL OUTER JOIN`. The standard emulation uses `UNION` (which performs distinct set deduplication) combining a `LEFT JOIN` and a `RIGHT JOIN`:

```sql
SELECT e.emp_id, e.emp_name, d.dept_id, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
UNION
SELECT e.emp_id, e.emp_name, d.dept_id, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

---

# Part 3: Critical Join Gotchas & Execution Internals

### 3.1 The ON Clause Filter vs. The WHERE Clause Filter in OUTER JOINs
This is one of the most frequently failed senior SQL interview questions.

#### Scenario A: Filtering in the `ON` clause
```sql
SELECT e.emp_name, d.dept_name
FROM departments d
LEFT JOIN employees e 
    ON d.dept_id = e.dept_id 
    AND e.salary > 100000;
```
- **How it executes:** The condition `e.salary > 100000` is evaluated **during the join phase**. If an employee has salary $\le 100000$, the join predicate is false for that employee. However, because it is a `LEFT JOIN` on `departments`, **all departments are still preserved**.
- **Output:** Every department is listed. Departments without employees making $> 100k$ will still appear, showing `NULL` for `emp_name`.

#### Scenario B: Filtering in the `WHERE` clause
```sql
SELECT e.emp_name, d.dept_name
FROM departments d
LEFT JOIN employees e 
    ON d.dept_id = e.dept_id 
WHERE e.salary > 100000;
```
- **How it executes:** The outer join happens first. Then, the `WHERE` clause filters the unified intermediate table. For unmatched departments (e.g., Marketing, Operations), `e.salary` is `NULL`. The predicate `NULL > 100000` evaluates to `UNKNOWN` and is discarded by `WHERE`.
- **Consequence:** **The LEFT JOIN is silently converted into an INNER JOIN!** Departments 30 and 40 are wiped out.

> [!CAUTION]
> Placing a predicate on the right table in the `WHERE` clause of a `LEFT JOIN` will convert it into an `INNER JOIN` unless you explicitly account for nulls (e.g., `WHERE (e.salary > 100000 OR e.salary IS NULL)`).

---

### 3.2 Join Cardinality & Row Explosion
When joining two tables on non-unique keys, the output row count can expand exponentially.
- If key `K` appears $A$ times in Table 1 and $B$ times in Table 2, an equijoin on $K$ produces $A \times B$ rows.
- **Real-world danger:** If an analytics pipeline joins an `Orders` table ($10^6$ rows) with a `Promotions` table on `promo_code`, and `promo_code = 'WELCOME'` is duplicated 5 times in the promotions dimension due to poor ETL data quality, the query produces $5 \times 10^6$ rows, inflating financial revenue metrics by $500\%$.

---

### 3.3 Join Algorithms Under the Hood (Engine Internals)
Understanding how query planners physically execute joins is essential for query tuning:

1. **Nested Loop Join:**
   - **Mechanism:** For each row in the outer table, scan the inner table.
   - **Cost:** $O(M \times N)$ without index; $O(M \log N)$ if inner table has an index on join key.
   - **Best for:** Small outer table and indexed inner table.
2. **Hash Join:**
   - **Mechanism:** Builds an in-memory hash table on the smaller table's join keys (Build phase). Then streams the larger table and probes the hash table (Probe phase).
   - **Cost:** $O(M + N)$ time complexity; requires memory proportional to the build table.
   - **Best for:** Large tables with equijoin predicates without usable indexes.
3. **Sort-Merge Join:**
   - **Mechanism:** Both inputs are sorted by the join key (if not already indexed/clustered) and traversed in lockstep.
   - **Cost:** $O(M \log M + N \log N)$ for sorting, then $O(M + N)$ for the merge.
   - **Best for:** Medium to large datasets where data is pre-sorted or join conditions include inequalities ($<$ or $>$).

---

# Part 4: Self Joins & Hierarchical Data

### 4.1 Definition & Syntax
A **Self Join** is a standard join in which a table is joined with itself. It requires distinct table aliases to distinguish between the two roles the table plays in the query.

### 4.2 Use Case 1: Employee-Manager Hierarchy
In our `employees` table, `manager_id` references `emp_id` in the very same table:

```sql
SELECT 
    emp.emp_name AS employee,
    COALESCE(mgr.emp_name, 'Top Level Executive') AS manager
FROM employees emp
LEFT JOIN employees mgr 
    ON emp.manager_id = mgr.emp_id;
```

#### Result:
| employee | manager |
| :--- | :--- |
| Alice | Top Level Executive |
| Bob | Alice |
| Charlie | Alice |
| David | Charlie |
| Emma | Alice |
| Frank | Charlie |

*Why LEFT JOIN?* The CEO/President has `manager_id IS NULL`. An `INNER JOIN` would drop the top-level executive from the output.

---

### 4.3 Use Case 2: Finding Inter-Row Comparisons (Pairs & Precedence)
Finding all pairs of employees working in the same department:

```sql
SELECT 
    e1.emp_name AS employee_1,
    e2.emp_name AS employee_2,
    e1.dept_id
FROM employees e1
INNER JOIN employees e2 
    ON e1.dept_id = e2.dept_id
    AND e1.emp_id < e2.emp_id; -- Strict inequality prevents self-pairing and duplicate reverse pairs
```
- Using `e1.emp_id != e2.emp_id` would output both `(Alice, Bob)` and `(Bob, Alice)`.
- Using `e1.emp_id < e2.emp_id` outputs strictly unique pairings.

---

# Part 5: Aggregation & Grouping Mechanics

### 5.1 The `GROUP BY` Clause Mechanics
`GROUP BY` transforms a detailed row-level relation into a collapsed summary relation.
- Rows sharing identical values across the specified grouping attributes are placed into discrete buckets.
- All non-grouping attributes selected in the `SELECT` list **must** be wrapped in aggregate functions (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`).

```sql
SELECT 
    dept_id, 
    COUNT(*) AS total_employees,
    AVG(salary) AS avg_salary,
    MAX(salary) AS highest_salary
FROM employees
WHERE dept_id IS NOT NULL
GROUP BY dept_id;
```

#### Grouping by Expressions:
You can group by expressions, date truncations, or conditional buckets:
```sql
SELECT 
    CASE 
        WHEN salary >= 100000 THEN 'High Band'
        WHEN salary >= 80000 THEN 'Mid Band'
        ELSE 'Entry Band'
    END AS salary_tier,
    COUNT(*) AS emp_count
FROM employees
GROUP BY 
    CASE 
        WHEN salary >= 100000 THEN 'High Band'
        WHEN salary >= 80000 THEN 'Mid Band'
        ELSE 'Entry Band'
    END;
```

---

### 5.2 The Crucial Distinction: `WHERE` vs `HAVING`

| Dimension | `WHERE` Clause | `HAVING` Clause |
| :--- | :--- | :--- |
| **Stage of Execution** | Step 3: Evaluated **before** grouping occurs. | Step 6: Evaluated **after** aggregation and grouping. |
| **Target of Filtering** | Operates on individual, base records (rows). | Operates on grouped, aggregated summary buckets. |
| **Aggregate Functions** | **Cannot** contain aggregate functions (`WHERE SUM(x) > 10` is an error). | **Can and usually does** contain aggregate functions (`HAVING COUNT(*) > 2`). |
| **Index Utilization** | Can directly utilize B-tree indexes to prune rows early. | Cannot leverage standard indexes directly for aggregate results. |
| **Engine Efficiency** | High: Reduces volume of rows sent to grouping buffer. | Lower: Applied after expensive hashing or sorting for grouping. |

#### Demonstration of Co-existence:
```sql
SELECT 
    dept_id, 
    AVG(salary) AS avg_sal,
    COUNT(*) AS active_staff
FROM employees
WHERE salary >= 80000          -- Step 1: Discard any employee making under $80k BEFORE grouping
GROUP BY dept_id              -- Step 2: Form groups of remaining employees
HAVING COUNT(*) >= 2;          -- Step 3: Keep only groups that have 2 or more such employees
```

---

# Part 6: Advanced Aggregation Patterns

### 6.1 Conditional Aggregation (`CASE` inside Aggregates)
Conditional aggregation allows you to perform multi-pass aggregation in a single scan of the dataset.

#### Pattern 1: Horizontal Pivoting (Rows to Columns)
Given sales records with order statuses, calculate counts per month:
```sql
SELECT 
    DATE_TRUNC('month', order_date) AS order_month,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
    SUM(CASE WHEN status = 'REFUNDED' THEN 1 ELSE 0 END) AS refunded_orders,
    SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders
FROM orders
GROUP BY DATE_TRUNC('month', order_date);
```

#### Pattern 2: Calculating KPI Ratios & Conversion Rates
```sql
SELECT 
    dept_id,
    ROUND(
        100.0 * SUM(CASE WHEN salary >= 100000 THEN 1 ELSE 0 END) / COUNT(*), 
        2
    ) AS pct_six_figure_earners
FROM employees
WHERE dept_id IS NOT NULL
GROUP BY dept_id;
```

> [!TIP]
> Notice the usage of `100.0` (floating-point literal) instead of `100` to prevent integer division truncation in databases like SQL Server and PostgreSQL.

---

### 6.2 Duplicate Detection & Deduplication Strategies

#### Finding Duplicate Values in a Single Column:
```sql
SELECT email, COUNT(*) AS occurrences
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;
```

#### Finding Exact Duplicate Rows Across Composite Keys:
```sql
SELECT first_name, last_name, date_of_birth, COUNT(*) AS occurrences
FROM patients
GROUP BY first_name, last_name, date_of_birth
HAVING COUNT(*) > 1;
```

#### Isolating All Fields of Duplicate Records:
```sql
WITH DuplicateKeys AS (
    SELECT email
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
)
SELECT u.*
FROM users u
INNER JOIN DuplicateKeys d ON u.email = d.email
ORDER BY u.email, u.created_at;
```

---

# Part 7: Complete Logical Query Execution Order with Joins

Recall the foundational SQL execution pipeline. Today we integrate `JOIN` and `HAVING`:

```
1. FROM        (Identify base tables and Cartesian product)
2. ON          (Apply join filters for each row pair)
3. JOIN        (Retain matches and append outer NULL-padded rows)
4. WHERE       (Filter combined intermediate rows)
5. GROUP BY    (Collapse rows into groups based on key values)
6. WITH CUBE / ROLLUP (Optional multi-dimensional aggregations)
7. HAVING      (Filter aggregate group buckets)
8. SELECT      (Evaluate projection list, expressions, aliases)
9. DISTINCT    (Eliminate duplicate output rows)
10. ORDER BY   (Sort output; can reference SELECT aliases and unselected cols)
11. OFFSET / LIMIT / TOP (Return requested slice of rows)
```

---

# Part 8: Essential Interview Comparisons

### Comparison 1: INNER JOIN vs LEFT JOIN vs CROSS JOIN
| Attribute | INNER JOIN | LEFT JOIN | CROSS JOIN |
| :--- | :--- | :--- | :--- |
| **Output size min** | $0$ rows | Number of rows in Left Table | $M \times N$ rows |
| **Output size max** | $M \times N$ rows | $\max(M, M \times N)$ | $M \times N$ rows exactly |
| **Null generation** | Never generates NULLs | Generates NULLs for unmatched Right rows | Never generates NULLs |
| **Predicate** | Requires join condition (`ON`) | Requires join condition (`ON`) | No condition (or trivial `1=1`) |
| **Semantic Meaning** | "Intersection of relationships" | "All primary entities with optional details" | "All possible combinations" |

### Comparison 2: ON vs WHERE in Joins
| Scenario | ON Clause | WHERE Clause |
| :--- | :--- | :--- |
| **INNER JOIN** | Logically equivalent to WHERE (filtered during join). | Logically equivalent to ON (filtered after join). |
| **LEFT JOIN (Right Table)** | Controls which right rows attach. Preserves left rows. | Filters post-join. Eliminates left rows with NULL right attributes! |
| **LEFT JOIN (Left Table)** | Unintuitive: evaluating left predicates in ON does not discard left rows. | Standard: strictly filters out unwanted left rows. |

---

# Part 9: Comprehensive Technical Interview Question Bank

### Q1: What happens if you join two tables on a column where both contain NULL values?
**Answer:**
In SQL, `NULL = NULL` evaluates to `UNKNOWN` in accordance with three-valued logic. Because standard join predicates require the condition to evaluate to `TRUE` to produce a match, rows with `NULL` keys will **never** match each other in an equijoin (`ON a.key = b.key`). To treat `NULL` as equal for joining, one must explicitly use `IS NOT DISTINCT FROM` (PostgreSQL) or `ON a.key = b.key OR (a.key IS NULL AND b.key IS NULL)`.

### Q2: What is the maximum and minimum number of rows returned by a LEFT JOIN between Table A (100 rows) and Table B (50 rows)?
**Answer:**
- **Minimum:** 100 rows. A `LEFT JOIN` guarantees that every row from the left table appears at least once. If zero rows in Table B match Table A, all 100 rows appear with `NULL`s for B's columns.
- **Maximum:** $100 \times 50 = 5,000$ rows. If every single row in Table A matches all 50 rows in Table B (for instance, all rows have identical join keys or `ON 1=1`), the result is the full Cartesian product.

### Q3: Why is `SELECT dept_id, emp_name, AVG(salary) FROM employees GROUP BY dept_id;` invalid in ANSI SQL?
**Answer:**
Because `emp_name` is neither enclosed within an aggregate function nor specified in the `GROUP BY` clause. Since grouping collapses multiple employees of a department into a single output row, the database engine cannot arbitrarily choose which employee's name to display alongside the departmental average salary. (Note: SQLite and older MySQL versions allowed this by returning an arbitrary non-deterministic value, but it is illegal in standard SQL).

### Q4: Explain the difference between Semi-Join and Anti-Join. How are they written in SQL?
**Answer:**
- **Semi-Join:** Returns rows from the left table that have at least one match in the right table, without duplicating left rows and without returning right columns. Written via `EXISTS` or `IN`:
  ```sql
  SELECT * FROM departments d
  WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);
  ```
- **Anti-Join:** Returns rows from the left table that have zero matches in the right table. Written via `NOT EXISTS` or `LEFT JOIN ... WHERE right.key IS NULL`:
  ```sql
  SELECT * FROM departments d
  WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);
  ```

### Q5: Can `HAVING` be used without a `GROUP BY` clause?
**Answer:**
Yes. If `HAVING` is used without an explicit `GROUP BY`, the entire table is treated as a single composite group. For example:
```sql
SELECT AVG(salary) 
FROM employees 
HAVING COUNT(*) > 5;
```
If the table has 5 or fewer rows, this query returns an empty set (0 rows). If it has more than 5 rows, it returns 1 row containing the overall average salary.

---

# Part 10: Hands-On Practice Lab (20 Problems)

### Database Initialization Script for Practice
Run this schema in PostgreSQL, MySQL, SQL Server, or SQLite to follow along:

```sql
DROP TABLE IF EXISTS order_items;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(50),
    city VARCHAR(50),
    segment VARCHAR(20)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(50),
    category VARCHAR(30),
    unit_price DECIMAL(10, 2)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    order_status VARCHAR(20),
    total_amount DECIMAL(10, 2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    price_per_unit DECIMAL(10, 2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Seed Data
INSERT INTO customers VALUES
(101, 'Acme Corp', 'New York', 'Corporate'),
(102, 'Beta LLC', 'San Francisco', 'Consumer'),
(103, 'Cyberdyne', 'New York', 'Corporate'),
(104, 'Delta Inc', 'Chicago', 'Enterprise'),
(105, 'Epsilon Co', 'Austin', 'Consumer'),
(106, 'Foxtrot Ltd', 'Chicago', 'Enterprise');

INSERT INTO products VALUES
(1, 'Cloud Database', 'Software', 500.00),
(2, 'Analytics Suite', 'Software', 300.00),
(3, 'Security Gateway', 'Hardware', 800.00),
(4, 'Server Rack', 'Hardware', 1200.00),
(5, 'Consulting Hour', 'Services', 150.00);

INSERT INTO orders VALUES
(1001, 101, '2024-01-15', 'Delivered', 1600.00),
(1002, 101, '2024-02-10', 'Delivered', 800.00),
(1003, 102, '2024-02-14', 'Cancelled', 500.00),
(1004, 103, '2024-03-01', 'Delivered', 2400.00),
(1005, 104, '2024-03-15', 'Pending', 600.00),
(1006, 101, '2024-04-05', 'Delivered', 300.00);
-- Customer 105 and 106 have placed no orders!

INSERT INTO order_items VALUES
(1, 1001, 1, 2, 500.00),
(2, 1001, 2, 2, 300.00),
(3, 1002, 3, 1, 800.00),
(4, 1003, 1, 1, 500.00),
(5, 1004, 4, 2, 1200.00),
(6, 1005, 2, 2, 300.00),
(7, 1006, 2, 1, 300.00);
```

---

### Practice Problem Statements (Try Before Viewing Solutions!)

1. **Inner Match:** Display customer name, order ID, and order date for all delivered orders.
2. **Customer Order Audit:** Display all customer names and their order IDs, including customers who have never placed an order.
3. **Unmatched Customers:** Identify all customers who have never placed an order using a Left Anti-Join.
4. **Product Purchase Check:** Find all products that have never been ordered.
5. **Customer Spending Summary:** Calculate total amount spent by each customer across delivered orders only. Include customer name.
6. **High Spenders Only:** Display customer names and total spending for customers whose delivered orders exceed $1,500.
7. **Department Order Metrics:** Show total orders, total delivered orders, and total cancelled orders per customer segment.
8. **Item-Level Sales Reconciliation:** For each order, display `order_id`, header `total_amount`, and sum of `quantity * price_per_unit` from `order_items` to check for discrepancies.
9. **Zero-Order Cities:** List cities where not a single customer has placed a delivered order.
10. **Category Breakdown:** Calculate total units sold and total revenue per product category.
11. **Self Join - City Peers:** Find pairs of distinct customers located in the same city. Avoid duplicate reverse pairs.
12. **Multi-Item Orders:** Find all order IDs that contain more than 1 distinct product.
13. **Product Popularity Rank:** List product names alongside the count of unique customers who bought them.
14. **Customer Multi-Order Frequency:** List customers who have placed 2 or more delivered orders.
15. **Conditional Aggregation - Status Split:** Display customer ID and two columns: `delivered_spend` and `non_delivered_spend`.
16. **Average Order Value per Segment:** Find the average delivered order amount grouped by customer segment.
17. **Duplicate Detection:** In the `customers` table, write a query to detect if duplicate `customer_name` entries exist.
18. **Cross Join Combinations:** Generate all possible combinations of customer segments and product categories.
19. **Orders Containing Hardware:** List customer names that have ordered at least one item from the 'Hardware' category.
20. **Comprehensive Reconciliation:** For each customer, show customer name, count of total orders, count of delivered orders, and total net delivered revenue (default to 0.00 if none).

---

# Part 11: Solutions & Explanations to Practice Problems

### Solution 1
```sql
SELECT c.customer_name, o.order_id, o.order_date
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'Delivered';
```
*Explanation:* Standard inner join filtered by status. Eliminates non-purchasing customers and non-delivered orders.

### Solution 2
```sql
SELECT c.customer_name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```
*Explanation:* Preserves all customers (Epsilon and Foxtrot output with `NULL` for `order_id`).

### Solution 3
```sql
SELECT c.customer_id, c.customer_name, c.city
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```
*Explanation:* Anti-join pattern. Evaluates primary key of right table for `NULL`.

### Solution 4
```sql
SELECT p.product_id, p.product_name
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
WHERE oi.item_id IS NULL;
```
*Explanation:* Identifies Consulting Hour (product 5) which has zero sales rows.

### Solution 5
```sql
SELECT c.customer_name, SUM(o.total_amount) AS total_spent
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'Delivered'
GROUP BY c.customer_id, c.customer_name;
```
*Explanation:* Includes `c.customer_id` in `GROUP BY` to ensure proper grouping if two customers share a name.

### Solution 6
```sql
SELECT c.customer_name, SUM(o.total_amount) AS total_spent
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'Delivered'
GROUP BY c.customer_id, c.customer_name
HAVING SUM(o.total_amount) > 1500;
```
*Explanation:* `HAVING` filters aggregate sum after groups are constructed.

### Solution 7
```sql
SELECT 
    c.segment,
    COUNT(o.order_id) AS total_orders,
    SUM(CASE WHEN o.order_status = 'Delivered' THEN 1 ELSE 0 END) AS delivered_orders,
    SUM(CASE WHEN o.order_status = 'Cancelled' THEN 1 ELSE 0 END) AS cancelled_orders
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.segment;
```
*Explanation:* Conditional aggregation counts order statuses across customer segments.

### Solution 8
```sql
SELECT 
    o.order_id,
    o.total_amount AS header_amount,
    SUM(oi.quantity * oi.price_per_unit) AS calculated_items_amount,
    (o.total_amount - SUM(oi.quantity * oi.price_per_unit)) AS discrepancy
FROM orders o
INNER JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.total_amount;
```
*Explanation:* Validates relational data integrity between header and line items.

### Solution 9
```sql
SELECT DISTINCT c.city
FROM customers c
WHERE c.city NOT IN (
    SELECT DISTINCT c2.city
    FROM customers c2
    INNER JOIN orders o ON c2.customer_id = o.customer_id
    WHERE o.order_status = 'Delivered'
);
```
*Alternative via Joins:*
```sql
SELECT c.city
FROM customers c
LEFT JOIN orders o 
    ON c.customer_id = o.customer_id 
    AND o.order_status = 'Delivered'
GROUP BY c.city
HAVING COUNT(o.order_id) = 0;
```

### Solution 10
```sql
SELECT 
    p.category,
    COALESCE(SUM(oi.quantity), 0) AS total_units_sold,
    COALESCE(SUM(oi.quantity * oi.price_per_unit), 0.00) AS total_revenue
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.category;
```

### Solution 11
```sql
SELECT 
    c1.customer_name AS customer_a,
    c2.customer_name AS customer_b,
    c1.city
FROM customers c1
INNER JOIN customers c2 
    ON c1.city = c2.city 
    AND c1.customer_id < c2.customer_id;
```
*Explanation:* Strictly ordered ID comparison prevents duplicate permutations.

### Solution 12
```sql
SELECT order_id, COUNT(DISTINCT product_id) AS unique_products
FROM order_items
GROUP BY order_id
HAVING COUNT(DISTINCT product_id) > 1;
```

### Solution 13
```sql
SELECT 
    p.product_name,
    COUNT(DISTINCT o.customer_id) AS unique_buyers
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id
GROUP BY p.product_id, p.product_name;
```

### Solution 14
```sql
SELECT c.customer_id, c.customer_name, COUNT(o.order_id) AS delivered_count
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'Delivered'
GROUP BY c.customer_id, c.customer_name
HAVING COUNT(o.order_id) >= 2;
```

### Solution 15
```sql
SELECT 
    customer_id,
    SUM(CASE WHEN order_status = 'Delivered' THEN total_amount ELSE 0 END) AS delivered_spend,
    SUM(CASE WHEN order_status != 'Delivered' THEN total_amount ELSE 0 END) AS non_delivered_spend
FROM orders
GROUP BY customer_id;
```

### Solution 16
```sql
SELECT 
    c.segment,
    ROUND(AVG(o.total_amount), 2) AS avg_order_value
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_status = 'Delivered'
GROUP BY c.segment;
```

### Solution 17
```sql
SELECT customer_name, COUNT(*) AS occurrences
FROM customers
GROUP BY customer_name
HAVING COUNT(*) > 1;
```

### Solution 18
```sql
SELECT DISTINCT c.segment, p.category
FROM customers c
CROSS JOIN products p;
```

### Solution 19
```sql
SELECT DISTINCT c.customer_name
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id
WHERE p.category = 'Hardware';
```

### Solution 20
```sql
SELECT 
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) AS total_orders,
    COUNT(CASE WHEN o.order_status = 'Delivered' THEN 1 END) AS delivered_orders,
    COALESCE(SUM(CASE WHEN o.order_status = 'Delivered' THEN o.total_amount ELSE 0 END), 0.00) AS net_delivered_revenue
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY net_delivered_revenue DESC;
```

---

# Part 12: End-to-End Enterprise Case Study: E-Commerce Revenue Reconciliation & Audit

### Business Context & Problem Statement
At **DataScale Retail**, finance discovered that the reported monthly GMV (Gross Merchandise Value) calculated from the `orders` header table does not match the sum of individual line items in the `order_items` table. Furthermore, customer lifetime value (LTV) models failed because customers without orders were excluded from cohort retention curves.

As a Senior Data Engineer, you must build an audit query that:
1. Reconciles every order header with its itemized sum.
2. Identifies phantom orders (orders with no items) and orphan items (items with invalid `order_id`).
3. Produces a consolidated customer revenue table accurately representing all customers, including non-buyers.

### Implementation Solution
```sql
-- Comprehensive Financial Integrity & Reconciliation Model
WITH OrderReconciliation AS (
    SELECT 
        o.order_id,
        o.customer_id,
        o.order_status,
        o.total_amount AS reported_header_amount,
        COALESCE(SUM(oi.quantity * oi.price_per_unit), 0.00) AS calculated_items_amount,
        (o.total_amount - COALESCE(SUM(oi.quantity * oi.price_per_unit), 0.00)) AS discrepancy
    FROM orders o
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.order_id, o.customer_id, o.order_status, o.total_amount
)
SELECT 
    c.customer_id,
    c.customer_name,
    c.segment,
    COUNT(r.order_id) AS total_placed_orders,
    SUM(CASE WHEN r.order_status = 'Delivered' THEN 1 ELSE 0 END) AS valid_delivered_orders,
    COALESCE(SUM(CASE WHEN r.order_status = 'Delivered' THEN r.calculated_items_amount ELSE 0 END), 0.00) AS audited_net_revenue,
    SUM(CASE WHEN ABS(r.discrepancy) > 0.01 THEN 1 ELSE 0 END) AS orders_with_discrepancies
FROM customers c
LEFT JOIN OrderReconciliation r ON c.customer_id = r.customer_id
GROUP BY c.customer_id, c.customer_name, c.segment
ORDER BY audited_net_revenue DESC;
```

---

# Part 13: 10-Minute Interview Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SQL JOINS & AGGREGATION CHEAT SHEET             │
├────────────────────────────────────────────────────────────────────────┤
│ 1. INNER JOIN : Returns rows with matching keys in BOTH tables.        │
│ 2. LEFT JOIN  : All left rows; matching right rows or NULLs.           │
│ 3. RIGHT JOIN : All right rows; matching left rows or NULLs.          │
│ 4. FULL JOIN  : All rows from both; NULL-padded where unmatched.       │
│ 5. CROSS JOIN : Cartesian product (M x N rows). No ON clause.          │
│ 6. ANTI-JOIN  : LEFT JOIN ... WHERE right.key IS NULL                  │
│                 (Finds unmatched records).                             │
│ 7. SEMI-JOIN  : WHERE EXISTS (SELECT 1 FROM ...)                       │
│                 (Filters without expanding duplicate matches).         │
├────────────────────────────────────────────────────────────────────────┤
│                          WHERE vs HAVING                               │
│  - WHERE filters base rows BEFORE grouping. No aggregate functions.   │
│  - HAVING filters group summaries AFTER aggregation.                  │
├────────────────────────────────────────────────────────────────────────┤
│                       CONDITIONAL AGGREGATION                          │
│  SUM(CASE WHEN status = 'Delivered' THEN revenue ELSE 0 END)           │
│  COUNT(CASE WHEN flag = 1 THEN 1 END) -- NULLs ignored by COUNT!       │
├────────────────────────────────────────────────────────────────────────┤
│                          COMMON PITFALLS                               │
│  - Filtering right table in WHERE clause converts LEFT JOIN to INNER.  │
│  - NULL = NULL is UNKNOWN (joins will never match on NULL).            │
│  - Duplicate join keys cause row explosion (M x N multiplication).     │
│  - Selecting unaggregated columns not in GROUP BY is invalid SQL.      │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 14: 100-Mark Self-Assessment Examination

### Section A: Technical Multiple Choice & Conceptual (25 Marks — 5 marks each)

**Q1:** Given Table A with 5 rows (all keys = 1) and Table B with 4 rows (all keys = 1). How many rows does an `INNER JOIN ON A.key = B.key` return?  
(a) 5  
(b) 4  
(c) 9  
(d) 20  

**Q2:** A query has `FROM A LEFT JOIN B ON A.id = B.id WHERE B.status = 'ACTIVE'`. What is the practical consequence of this query?  
(a) All rows from A are preserved, and only active rows from B attach.  
(b) The query behaves as an `INNER JOIN`, discarding all rows of A with no match in B.  
(c) The query throws a compilation syntax error.  
(d) Unmatched rows from A are assigned status 'ACTIVE'.  

**Q3:** What is the result of `COUNT(CASE WHEN score >= 50 THEN NULL ELSE 1 END)` when evaluated against 10 students where 6 have score $\ge 50$ and 4 have score $< 50$?  
(a) 10  
(b) 6  
(c) 4  
(d) 0  

**Q4:** In SQL logical execution order, which clause is executed immediately before `HAVING`?  
(a) WHERE  
(b) SELECT  
(c) GROUP BY  
(d) ORDER BY  

**Q5:** Why does `ON A.id = B.id` fail to match rows where `A.id` is `NULL` and `B.id` is `NULL`?  
(a) SQL requires primary keys to be unique.  
(b) `NULL = NULL` evaluates to `UNKNOWN` in SQL 3-valued logic.  
(c) Query optimizer skips NULL values for indexing.  
(d) NULL is converted to 0 during join execution.  

---

### Section B: Output Prediction (25 Marks)

#### Schema for Prediction:
```sql
CREATE TABLE t1 (id INT, val VARCHAR(10));
CREATE TABLE t2 (id INT, val VARCHAR(10));

INSERT INTO t1 VALUES (1, 'A'), (2, 'B'), (NULL, 'C');
INSERT INTO t2 VALUES (1, 'X'), (1, 'Y'), (3, 'Z'), (NULL, 'W');
```

**Q6 (10 Marks):** Write down the exact output rows (id and val from both tables) for:
```sql
SELECT t1.id AS t1_id, t2.id AS t2_id, t1.val, t2.val
FROM t1
INNER JOIN t2 ON t1.id = t2.id;
```

**Q7 (15 Marks):** Write down the exact output rows for:
```sql
SELECT t1.id AS t1_id, t2.id AS t2_id, t1.val, t2.val
FROM t1
LEFT JOIN t2 ON t1.id = t2.id;
```

---

### Section C: Query Writing & Code Implementation (30 Marks)

**Q8 (15 Marks):** Given a table `transactions (txn_id, user_id, txn_date, amount, status)`, write a single query to compute:
- `user_id`
- Total transaction count
- Total amount of 'SUCCESS' transactions
- Total amount of 'FAILED' transactions
- Success rate as a percentage (`100.0 * success_count / total_count`) rounded to 2 decimal places.
Only include users who have at least 3 total transactions.

**Q9 (15 Marks):** Given an `employees (emp_id, emp_name, salary, manager_id)` table, write a query using a `SELF JOIN` to find all employees who earn strictly more than their direct manager. Display `employee_name`, `employee_salary`, `manager_name`, and `manager_salary`.

---

### Section D: Real-World Scenario & Optimization (20 Marks)

**Q10 (20 Marks):** You are investigating an ETL pipeline that runs every night. The query joins a 50-million-row `events` table with a 50,000-row `users` table on `user_id`.
1. Which join physical algorithm (Nested Loop, Hash Join, Sort-Merge) will the query planner most likely select, and why? (7 marks)
2. The query runtime recently spiked from 2 minutes to 45 minutes after an index was dropped. Explain how an index on `events.user_id` vs `users.user_id` affects performance. (7 marks)
3. If `users` contains duplicate `user_id` records due to an upstream failure, describe the exact impact on the event count metrics. (6 marks)

---

# Assessment Answer Key & Scoring Guide

### Section A Solutions
- **Q1:** **(d)** 20. Cartesian match on keys: $5 \times 4 = 20$.
- **Q2:** **(b)** The query behaves as an `INNER JOIN`. When unmatched, `B.status` is `NULL`. `WHERE NULL = 'ACTIVE'` evaluates to `UNKNOWN` and is discarded.
- **Q3:** **(c)** 4. `COUNT(column)` ignores `NULL`s. For the 6 students with score $\ge 50$, the `CASE` returns `NULL`. For the other 4, it returns `1`. `COUNT` of four `1`s and six `NULL`s is 4.
- **Q4:** **(c)** `GROUP BY`.
- **Q5:** **(b)** `NULL = NULL` yields `UNKNOWN`, not `TRUE`.

### Section B Solutions
**Q6 Output:**
| t1_id | t2_id | val | val |
| :--- | :--- | :--- | :--- |
| 1 | 1 | A | X |
| 1 | 1 | A | Y |
*(2 rows total. NULLs and unmatched keys 2 and 3 do not match).*

**Q7 Output:**
| t1_id | t2_id | val | val |
| :--- | :--- | :--- | :--- |
| 1 | 1 | A | X |
| 1 | 1 | A | Y |
| 2 | NULL | B | NULL |
| NULL | NULL | C | NULL |
*(4 rows total. Left table rows 2 and NULL are preserved with NULL padding).*

### Section C Solutions
**Q8 Solution:**
```sql
SELECT 
    user_id,
    COUNT(*) AS total_txns,
    COALESCE(SUM(CASE WHEN status = 'SUCCESS' THEN amount ELSE 0 END), 0) AS success_amount,
    COALESCE(SUM(CASE WHEN status = 'FAILED' THEN amount ELSE 0 END), 0) AS failed_amount,
    ROUND(
        100.0 * SUM(CASE WHEN status = 'SUCCESS' THEN 1 ELSE 0 END) / COUNT(*),
        2
    ) AS success_rate_pct
FROM transactions
GROUP BY user_id
HAVING COUNT(*) >= 3;
```

**Q9 Solution:**
```sql
SELECT 
    e.emp_name AS employee_name,
    e.salary AS employee_salary,
    m.emp_name AS manager_name,
    m.salary AS manager_salary
FROM employees e
INNER JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

### Section D Solutions
**Q10 Solution:**
1. **Hash Join:** The planner will build an in-memory hash table on the small 50k-row `users` table and stream the 50M-row `events` table to probe keys ($O(M+N)$).
2. If `events.user_id` is queried in an indexed loop without hash join capability, missing an index causes a full sequential scan for every probe. Conversely, having an index on `events.user_id` allows rapid indexed lookups if queries filter by a specific subset of users.
3. If `users` has duplicates on `user_id`, the join key becomes $1 \text{ to } N$ rather than $1 \text{ to } 1$. Every event associated with that user will be duplicated $N$ times, severely inflating metrics like total pageviews, revenue, and active session counts.

