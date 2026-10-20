# Day 1 — SQL Fundamentals and Query Execution

## Sample Database Schema

All examples in these notes use the following schema. Save these CREATE and INSERT statements and run them in any SQL environment (MySQL, PostgreSQL, SQL Server, SQLite) to follow along.

### Table: `employees`

```sql
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    first_name  VARCHAR(50),
    last_name   VARCHAR(50),
    department  VARCHAR(50),
    salary      DECIMAL(10, 2),
    hire_date   DATE,
    manager_id  INT,
    city        VARCHAR(50),
    commission  DECIMAL(10, 2)
);

INSERT INTO employees VALUES
(1,  'Alice',   'Johnson',  'Engineering',  95000.00, '2019-03-15', NULL,  'New York',    NULL),
(2,  'Bob',     'Smith',    'Engineering',  87000.00, '2020-07-01', 1,     'New York',    NULL),
(3,  'Charlie', 'Brown',    'Sales',        72000.00, '2018-11-20', 5,     'Chicago',     5000.00),
(4,  'Diana',   'Lee',      'Sales',        68000.00, '2021-01-10', 5,     'Chicago',     7500.00),
(5,  'Eve',     'Davis',    'Sales',        91000.00, '2017-06-05', NULL,  'Chicago',     3000.00),
(6,  'Frank',   'Wilson',   'HR',           64000.00, '2022-02-28', 8,     'San Francisco', NULL),
(7,  'Grace',   'Taylor',   'HR',           61000.00, '2023-04-15', 8,     'San Francisco', NULL),
(8,  'Henry',   'Martinez', 'HR',           78000.00, '2016-09-01', NULL,  'San Francisco', NULL),
(9,  'Irene',   'Anderson', 'Engineering',  105000.00,'2015-05-20', 1,     'New York',    NULL),
(10, 'Jack',    'Thomas',   'Marketing',    73000.00, '2020-08-12', NULL,  'Boston',      2000.00),
(11, 'Karen',   'White',    'Marketing',    69000.00, '2021-03-30', 10,    'Boston',      1500.00),
(12, 'Leo',     'Harris',   'Engineering',  92000.00, '2019-10-10', 1,     'New York',    NULL),
(13, 'Mary',    'Clark',    'Sales',        68000.00, '2022-06-18', 5,     'Chicago',     4000.00),
(14, 'Nathan',  'Lewis',    'Engineering',  87000.00, '2021-09-05', 1,     'Boston',      NULL),
(15, 'Olivia',  'Walker',   'HR',           64000.00, '2023-01-20', 8,     NULL,          NULL);
```

### Table: `orders`

```sql
CREATE TABLE orders (
    order_id     INT PRIMARY KEY,
    customer_id  INT,
    order_date   DATE,
    amount       DECIMAL(10, 2),
    status       VARCHAR(20),
    region       VARCHAR(50)
);

INSERT INTO orders VALUES
(101, 1001, '2024-01-05', 250.00,  'Completed', 'East'),
(102, 1002, '2024-01-07', 450.00,  'Completed', 'West'),
(103, 1001, '2024-01-12', 125.00,  'Cancelled', 'East'),
(104, 1003, '2024-01-15', 800.00,  'Completed', 'North'),
(105, 1002, '2024-01-20', NULL,    'Pending',   'West'),
(106, 1004, '2024-02-01', 320.00,  'Completed', 'East'),
(107, 1005, '2024-02-05', 670.00,  'Completed', 'South'),
(108, 1003, '2024-02-10', 150.00,  'Cancelled', 'North'),
(109, 1001, '2024-02-15', 990.00,  'Completed', 'East'),
(110, 1006, '2024-02-20', 0.00,    'Completed', 'West'),
(111, 1004, '2024-03-01', 410.00,  'Pending',   'East'),
(112, 1007, '2024-03-05', 550.00,  'Completed', 'South');
```

### Quick Reference: Table Contents

**employees (15 rows)**

| emp_id | first_name | last_name | department  | salary    | hire_date  | manager_id | city          | commission |
|--------|-----------|-----------|-------------|-----------|------------|------------|---------------|------------|
| 1      | Alice     | Johnson   | Engineering | 95000.00  | 2019-03-15 | NULL       | New York      | NULL       |
| 2      | Bob       | Smith     | Engineering | 87000.00  | 2020-07-01 | 1          | New York      | NULL       |
| 3      | Charlie   | Brown     | Sales       | 72000.00  | 2018-11-20 | 5          | Chicago       | 5000.00    |
| 4      | Diana     | Lee       | Sales       | 68000.00  | 2021-01-10 | 5          | Chicago       | 7500.00    |
| 5      | Eve       | Davis     | Sales       | 91000.00  | 2017-06-05 | NULL       | Chicago       | 3000.00    |
| 6      | Frank     | Wilson    | HR          | 64000.00  | 2022-02-28 | 8          | San Francisco | NULL       |
| 7      | Grace     | Taylor    | HR          | 61000.00  | 2023-04-15 | 8          | San Francisco | NULL       |
| 8      | Henry     | Martinez  | HR          | 78000.00  | 2016-09-01 | NULL       | San Francisco | NULL       |
| 9      | Irene     | Anderson  | Engineering | 105000.00 | 2015-05-20 | 1          | New York      | NULL       |
| 10     | Jack      | Thomas    | Marketing   | 73000.00  | 2020-08-12 | NULL       | Boston        | 2000.00    |
| 11     | Karen     | White     | Marketing   | 69000.00  | 2021-03-30 | 10         | Boston        | 1500.00    |
| 12     | Leo       | Harris    | Engineering | 92000.00  | 2019-10-10 | 1          | New York      | NULL       |
| 13     | Mary      | Clark     | Sales       | 68000.00  | 2022-06-18 | 5          | Chicago       | 4000.00    |
| 14     | Nathan    | Lewis     | Engineering | 87000.00  | 2021-09-05 | 1          | Boston        | NULL       |
| 15     | Olivia    | Walker    | HR          | 64000.00  | 2023-01-20 | 8          | NULL          | NULL       |

---

## Part 1: SELECT Statement

### 1.1 Definition

`SELECT` is the SQL clause that specifies which columns (or expressions) to include in the result set. It is the most fundamental SQL statement and the starting point of every query.

### 1.2 Intuition

Think of a database table as a spreadsheet. `SELECT` is you telling the database "I want to see these specific columns." You're not modifying data — you're requesting a view of it.

### 1.3 Purpose

- Retrieve specific columns instead of the entire table.
- Compute derived values (expressions, calculations).
- Rename columns for readability (aliasing).
- Serve as the projection operation in relational algebra.

### 1.4 Syntax

```sql
-- Select specific columns
SELECT column1, column2 FROM table_name;

-- Select all columns
SELECT * FROM table_name;

-- Select with expression
SELECT column1, column2 * 1.1 AS adjusted_value FROM table_name;

-- Select with alias
SELECT first_name AS "First Name", salary AS annual_pay FROM table_name;
```

### 1.5 Worked Examples

**Example 1: Select specific columns**
```sql
SELECT first_name, department, salary
FROM employees;
```

**Result (15 rows, showing first 5):**

| first_name | department  | salary    |
|-----------|-------------|-----------|
| Alice     | Engineering | 95000.00  |
| Bob       | Engineering | 87000.00  |
| Charlie   | Sales       | 72000.00  |
| Diana     | Sales       | 68000.00  |
| Eve       | Sales       | 91000.00  |

**Example 2: Select with computed column and alias**
```sql
SELECT 
    first_name,
    salary,
    salary * 12 AS annual_salary,
    salary / 160 AS hourly_rate
FROM employees;
```

| first_name | salary    | annual_salary  | hourly_rate |
|-----------|-----------|----------------|-------------|
| Alice     | 95000.00  | 1140000.00     | 593.75      |
| Bob       | 87000.00  | 1044000.00     | 543.75      |

**Example 3: String concatenation (dialect-dependent)**
```sql
-- MySQL / SQLite
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;

-- PostgreSQL
SELECT first_name || ' ' || last_name AS full_name FROM employees;

-- SQL Server
SELECT first_name + ' ' + last_name AS full_name FROM employees;
```

### 1.6 SELECT * — When to Use and When Not To

| Scenario | Use `SELECT *`? | Reason |
|----------|----------------|--------|
| Quick exploration / debugging | ✅ Yes | Convenient for ad-hoc investigation |
| Production queries | ❌ No | Fragile if schema changes; wastes bandwidth |
| Application code | ❌ No | Unpredictable column order; performance overhead |
| Views | ❌ No | Schema changes may silently break the view |

### 1.7 Common Mistakes

1. **Using SELECT * in production:** If a column is added or removed, queries and application code may break silently.
2. **Forgetting aliases for expressions:** `SELECT salary * 12 FROM employees` returns a column with an auto-generated name (varies by DBMS), making results hard to use.
3. **Quoting aliases inconsistently:** Use double quotes (`"Full Name"`) for aliases with spaces in standard SQL; SQL Server allows square brackets (`[Full Name]`); MySQL allows backticks (`` `Full Name` ``).

### 1.8 Interview Explanation

> "SELECT specifies the columns or expressions to include in the result set. It's the projection operation — it determines the shape of the output. In production, I always name specific columns rather than using SELECT * to avoid fragile dependencies on table schema and to reduce network overhead."

---

## Part 2: DISTINCT

### 2.1 Definition

`DISTINCT` is a keyword used with `SELECT` that eliminates duplicate rows from the result set. Two rows are considered duplicates if all selected columns have identical values.

### 2.2 Intuition

If you ask "which departments exist in the company?" you want each department listed once, not repeated for every employee. DISTINCT removes the repeats.

### 2.3 Purpose

- Identify unique values in a column or combination of columns.
- Data profiling: understanding the cardinality (number of unique values) of a column.
- Deduplication in reporting.

### 2.4 Syntax

```sql
SELECT DISTINCT column1 FROM table_name;
SELECT DISTINCT column1, column2 FROM table_name;
-- DISTINCT applies to the ENTIRE row of selected columns
```

### 2.5 Worked Examples

**Example 1: Unique departments**
```sql
SELECT DISTINCT department FROM employees;
```

| department  |
|-------------|
| Engineering |
| Sales       |
| HR          |
| Marketing   |

**4 rows** returned (from 15 total employees).

**Example 2: Unique department-city combinations**
```sql
SELECT DISTINCT department, city FROM employees;
```

| department  | city          |
|-------------|---------------|
| Engineering | New York      |
| Engineering | Boston        |
| Sales       | Chicago       |
| HR          | San Francisco |
| HR          | NULL          |
| Marketing   | Boston        |

**6 rows.** Note: DISTINCT treats `(HR, San Francisco)` and `(HR, NULL)` as different because the city values differ. Also note that DISTINCT considers two NULLs as equal to each other for deduplication purposes (this is an exception to the general rule that NULL ≠ NULL).

### 2.6 DISTINCT vs GROUP BY

Both can produce unique values, but they serve different purposes:

| Aspect | DISTINCT | GROUP BY |
|--------|----------|----------|
| Primary purpose | Remove duplicate rows | Create groups for aggregation |
| Can aggregate? | No (just deduplicates) | Yes (used with COUNT, SUM, etc.) |
| Performance | Typically similar for simple dedup | Same or better when combined with aggregation |
| Readability | Clearer intent for "unique values" | Clearer intent for "aggregated groups" |

```sql
-- These produce identical results for listing unique departments:
SELECT DISTINCT department FROM employees;
SELECT department FROM employees GROUP BY department;
```

### 2.7 Common Mistakes

1. **Thinking DISTINCT applies per column:** `SELECT DISTINCT dept, city` does NOT return distinct departments AND distinct cities separately — it returns distinct (dept, city) pairs.
2. **Using DISTINCT to hide a JOIN problem:** If a JOIN produces unexpected duplicates, the fix is to correct the JOIN logic, not to slap on DISTINCT.
3. **Performance trap:** DISTINCT requires sorting or hashing all rows. On large tables without proper indexing, it can be very expensive.

### 2.8 Interview Explanation

> "DISTINCT eliminates duplicate rows from the result set based on all selected columns. It's useful for data profiling and reporting unique values. However, if I find myself needing DISTINCT to 'fix' duplicate results from a JOIN, that's usually a signal that my JOIN logic needs correction rather than masking with DISTINCT."

---

## Part 3: WHERE Clause

### 3.1 Definition

`WHERE` is the clause that filters rows from the base table(s) before any grouping or aggregation occurs. Only rows that satisfy the condition(s) are passed to subsequent processing stages.

### 3.2 Intuition

WHERE is your filter. Before the database does anything else with the data, it checks each row: "Does this row meet the condition?" If yes, keep it. If no, discard it.

### 3.3 Purpose

- Filter rows based on column values, comparisons, patterns, or ranges.
- Reduce the data the database needs to process (improves performance).
- Implement business logic (e.g., "only active customers," "orders above $500").

### 3.4 Syntax and Operators

```sql
SELECT columns FROM table WHERE condition;
```

**Comparison operators:**

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Equal to | `WHERE department = 'Sales'` |
| `<>` or `!=` | Not equal to | `WHERE status <> 'Cancelled'` |
| `<` | Less than | `WHERE salary < 70000` |
| `>` | Greater than | `WHERE salary > 90000` |
| `<=` | Less than or equal | `WHERE hire_date <= '2020-12-31'` |
| `>=` | Greater than or equal | `WHERE amount >= 500` |

**Logical operators:**

| Operator | Meaning | Example |
|----------|---------|---------|
| `AND` | Both conditions true | `WHERE dept = 'Sales' AND salary > 70000` |
| `OR` | Either condition true | `WHERE dept = 'Sales' OR dept = 'HR'` |
| `NOT` | Negates condition | `WHERE NOT department = 'HR'` |

### 3.5 Operator Precedence

SQL evaluates logical operators in this order:
1. `NOT` (highest precedence)
2. `AND`
3. `OR` (lowest precedence)

**Critical interview trap:**

```sql
-- What does this return?
SELECT * FROM employees 
WHERE department = 'Sales' OR department = 'HR' AND salary > 70000;
```

Because `AND` binds tighter than `OR`, this is evaluated as:
```sql
WHERE department = 'Sales' OR (department = 'HR' AND salary > 70000)
```

This returns **all** Sales employees (regardless of salary) plus only HR employees earning above 70000.

**Fix with parentheses:**
```sql
WHERE (department = 'Sales' OR department = 'HR') AND salary > 70000
```

This returns employees from Sales or HR who also earn above 70000.

### 3.6 Worked Examples

**Example 1: Simple filter**
```sql
SELECT first_name, department, salary 
FROM employees 
WHERE department = 'Engineering';
```

| first_name | department  | salary    |
|-----------|-------------|-----------|
| Alice     | Engineering | 95000.00  |
| Bob       | Engineering | 87000.00  |
| Irene     | Engineering | 105000.00 |
| Leo       | Engineering | 92000.00  |
| Nathan    | Engineering | 87000.00  |

**Example 2: Multiple conditions with AND**
```sql
SELECT first_name, salary, department
FROM employees
WHERE department = 'Sales' AND salary >= 70000;
```

| first_name | salary   | department |
|-----------|----------|------------|
| Charlie   | 72000.00 | Sales      |
| Eve       | 91000.00 | Sales      |

**Example 3: OR with parentheses**
```sql
SELECT first_name, department, salary
FROM employees
WHERE (department = 'HR' OR department = 'Marketing') AND salary > 65000;
```

| first_name | department | salary   |
|-----------|-----------|----------|
| Henry     | HR        | 78000.00 |
| Jack      | Marketing | 73000.00 |
| Karen     | Marketing | 69000.00 |

### 3.7 Common Mistakes

1. **Forgetting operator precedence (AND before OR):** Always use parentheses to make intent explicit.
2. **Using `= NULL` instead of `IS NULL`:** `WHERE commission = NULL` always evaluates to UNKNOWN (returns no rows). Use `WHERE commission IS NULL`.
3. **String comparison case sensitivity:** Behavior varies by DBMS and collation. MySQL with `utf8_general_ci` collation is case-insensitive; PostgreSQL is case-sensitive by default.

---

## Part 4: ORDER BY

### 4.1 Definition

`ORDER BY` sorts the result set by one or more columns or expressions, in ascending (`ASC`, the default) or descending (`DESC`) order. It is the **only** clause that guarantees the order of rows in the output.

### 4.2 Key Fact for Interviews

> Without ORDER BY, SQL does NOT guarantee any particular row order. Even if results appear sorted, that's an implementation artifact, not a guarantee. Relying on implicit ordering is a bug.

### 4.3 Syntax

```sql
SELECT columns FROM table
ORDER BY column1 [ASC|DESC], column2 [ASC|DESC];

-- Order by column position (not recommended for production)
SELECT first_name, salary FROM employees ORDER BY 2 DESC;

-- Order by alias
SELECT first_name, salary * 12 AS annual FROM employees ORDER BY annual DESC;

-- Order by expression
SELECT first_name, salary FROM employees ORDER BY salary + COALESCE(commission, 0) DESC;
```

### 4.4 Worked Examples

**Example 1: Single column sort**
```sql
SELECT first_name, salary FROM employees ORDER BY salary DESC;
```

| first_name | salary    |
|-----------|-----------|
| Irene     | 105000.00 |
| Alice     | 95000.00  |
| Leo       | 92000.00  |
| Eve       | 91000.00  |
| Bob       | 87000.00  |
| Nathan    | 87000.00  |
| ... | ... |

**Example 2: Multi-column sort**
```sql
SELECT department, first_name, salary 
FROM employees 
ORDER BY department ASC, salary DESC;
```

This sorts by department alphabetically first, then within each department by salary highest-first:

| department  | first_name | salary    |
|-------------|-----------|-----------|
| Engineering | Irene     | 105000.00 |
| Engineering | Alice     | 95000.00  |
| Engineering | Leo       | 92000.00  |
| Engineering | Bob       | 87000.00  |
| Engineering | Nathan    | 87000.00  |
| HR          | Henry     | 78000.00  |
| HR          | Frank     | 64000.00  |
| HR          | Olivia    | 64000.00  |
| HR          | Grace     | 61000.00  |
| Marketing   | Jack      | 73000.00  |
| Marketing   | Karen     | 69000.00  |
| Sales       | Eve       | 91000.00  |
| Sales       | Charlie   | 72000.00  |
| Sales       | Diana     | 68000.00  |
| Sales       | Mary      | 68000.00  |

### 4.5 NULL Sorting Behavior

NULLs sort differently across databases:

| DBMS | NULL position in ASC | NULL position in DESC |
|------|---------------------|----------------------|
| MySQL | First (smallest) | Last |
| SQL Server | First (smallest) | Last |
| PostgreSQL | Last (largest) | First |
| Oracle | Last (largest) | First |

PostgreSQL and Oracle support `NULLS FIRST` / `NULLS LAST`:
```sql
SELECT first_name, commission FROM employees 
ORDER BY commission DESC NULLS LAST;
```

### 4.6 Common Mistakes

1. **Assuming result order without ORDER BY:** Never do this. The optimizer can return rows in any order.
2. **Sorting by column position in production:** `ORDER BY 2` is fragile — if columns are reordered in the SELECT list, the sort changes silently.
3. **Performance on large tables:** ORDER BY requires sorting. On millions of rows, sorting is expensive unless an index covers the sort column(s).

### 4.7 Interview Explanation

> "ORDER BY is the only SQL clause that guarantees row order in the result set. Without it, results may appear sorted incidentally but that's not reliable. I always specify ASC or DESC explicitly for clarity, and I'm aware that NULL sort behavior varies by database — PostgreSQL puts NULLs last in ASC, while MySQL puts them first."

---

## Part 5: LIMIT and TOP

### 5.1 Definition

`LIMIT` (MySQL, PostgreSQL, SQLite) and `TOP` (SQL Server) restrict the number of rows returned by a query. They are applied **after** sorting (if ORDER BY is present).

### 5.2 Purpose

- Pagination: showing results page by page.
- "Top N" queries: highest salaries, most recent orders.
- Performance: quickly sample data from a large table.
- Preventing accidental full-table dumps.

### 5.3 Syntax by Dialect

```sql
-- MySQL / PostgreSQL / SQLite
SELECT columns FROM table ORDER BY col LIMIT n;
SELECT columns FROM table ORDER BY col LIMIT n OFFSET m;

-- SQL Server
SELECT TOP n columns FROM table ORDER BY col;
SELECT TOP n PERCENT columns FROM table ORDER BY col;

-- Oracle (12c+)
SELECT columns FROM table ORDER BY col FETCH FIRST n ROWS ONLY;

-- Standard SQL (ANSI SQL:2008)
SELECT columns FROM table ORDER BY col 
OFFSET m ROWS FETCH NEXT n ROWS ONLY;
```

### 5.4 Worked Examples

**Example 1: Top 3 highest-paid employees**
```sql
-- MySQL / PostgreSQL
SELECT first_name, salary FROM employees ORDER BY salary DESC LIMIT 3;

-- SQL Server
SELECT TOP 3 first_name, salary FROM employees ORDER BY salary DESC;
```

| first_name | salary    |
|-----------|-----------|
| Irene     | 105000.00 |
| Alice     | 95000.00  |
| Leo       | 92000.00  |

**Example 2: Pagination — rows 6–10**
```sql
-- MySQL / PostgreSQL: OFFSET is 0-based
SELECT first_name, salary FROM employees 
ORDER BY salary DESC 
LIMIT 5 OFFSET 5;
```

| first_name | salary   |
|-----------|----------|
| Nathan    | 87000.00 |
| Henry     | 78000.00 |
| Jack      | 73000.00 |
| Charlie   | 72000.00 |
| Karen     | 69000.00 |

### 5.5 Common Mistakes

1. **Using LIMIT without ORDER BY:** The rows returned are non-deterministic. You might get different results on different runs.
2. **OFFSET performance on large datasets:** `LIMIT 10 OFFSET 1000000` forces the database to scan and skip 1 million rows. For deep pagination, use keyset (seek) pagination instead.
3. **Ties in Top-N:** If the 3rd and 4th employees have the same salary, `LIMIT 3` arbitrarily excludes one. SQL Server's `TOP 3 WITH TIES` handles this; standard SQL uses window functions.

### 5.6 LIMIT vs TOP Comparison

| Feature | LIMIT (MySQL/PG) | TOP (SQL Server) |
|---------|------------------|------------------|
| Position in query | End of query | After SELECT keyword |
| OFFSET support | Built-in: `LIMIT n OFFSET m` | Requires `OFFSET...FETCH` (SQL Server 2012+) |
| WITH TIES | Not available | `TOP n WITH TIES` |
| Percentage | Not available | `TOP n PERCENT` |

---

## Part 6: BETWEEN, IN, LIKE

### 6.1 BETWEEN

**Definition:** Tests whether a value falls within an inclusive range.

```sql
-- Syntax
WHERE column BETWEEN low AND high
-- Equivalent to:
WHERE column >= low AND column <= high
```

**Critical: BETWEEN is INCLUSIVE on both ends.**

**Example:**
```sql
SELECT first_name, salary FROM employees 
WHERE salary BETWEEN 70000 AND 90000;
```

| first_name | salary   |
|-----------|----------|
| Bob       | 87000.00 |
| Charlie   | 72000.00 |
| Henry     | 78000.00 |
| Jack      | 73000.00 |
| Nathan    | 87000.00 |

**5 rows.** (70000 ≤ salary ≤ 90000. Leo's 92000 > 90000, so excluded.)

**Date BETWEEN example:**
```sql
SELECT first_name, hire_date FROM employees
WHERE hire_date BETWEEN '2020-01-01' AND '2021-12-31';
```

| first_name | hire_date  |
|-----------|------------|
| Bob       | 2020-07-01 |
| Diana     | 2021-01-10 |
| Jack      | 2020-08-12 |
| Karen     | 2021-03-30 |
| Nathan    | 2021-09-05 |

**Common mistake with date BETWEEN:**
```sql
-- DANGER: If hire_date includes time component
WHERE hire_date BETWEEN '2020-01-01' AND '2020-01-31'
-- This EXCLUDES '2020-01-31 15:30:00' because it's > '2020-01-31 00:00:00'
-- Fix: use < for the upper bound
WHERE hire_date >= '2020-01-01' AND hire_date < '2020-02-01'
```

### 6.2 IN

**Definition:** Tests whether a value matches any value in a specified list or subquery result.

```sql
WHERE column IN (value1, value2, value3)
-- Equivalent to:
WHERE column = value1 OR column = value2 OR column = value3
```

**Example:**
```sql
SELECT first_name, department FROM employees
WHERE department IN ('Sales', 'Marketing');
```

| first_name | department |
|-----------|-----------|
| Charlie   | Sales     |
| Diana     | Sales     |
| Eve       | Sales     |
| Jack      | Marketing |
| Karen     | Marketing |
| Mary      | Sales     |

**NOT IN — beware of NULLs:**
```sql
-- This returns nothing if the subquery/list contains NULL!
SELECT * FROM employees WHERE department NOT IN ('Sales', NULL);
```

**Why?** `NOT IN` expands to `dept <> 'Sales' AND dept <> NULL`. Since `dept <> NULL` evaluates to UNKNOWN, the entire AND chain becomes UNKNOWN, and no rows are returned.

**Fix:** Use `NOT IN` only with NULL-free lists, or use `NOT EXISTS` instead.

### 6.3 LIKE

**Definition:** Pattern matching for string columns using wildcard characters.

| Wildcard | Meaning | Example |
|----------|---------|---------|
| `%` | Zero or more characters | `'A%'` matches Alice, Anderson |
| `_` | Exactly one character | `'_ob'` matches Bob |

```sql
-- Names starting with 'A'
SELECT first_name FROM employees WHERE first_name LIKE 'A%';
-- Result: Alice

-- Names containing 'ar'
SELECT first_name FROM employees WHERE first_name LIKE '%ar%';
-- Result: Charlie, Karen, Mary

-- Names with exactly 3 characters
SELECT first_name FROM employees WHERE first_name LIKE '___';
-- Result: Bob, Eve, Leo

-- Escape a literal %
SELECT * FROM products WHERE name LIKE '%\%%' ESCAPE '\';
```

**Performance warning:** `LIKE '%text'` (leading wildcard) cannot use an index and forces a full table scan. Only `LIKE 'text%'` (trailing wildcard) can leverage a B-tree index.

### 6.4 BETWEEN vs IN vs LIKE Comparison

| Operator | Use Case | Can Use Index? | NULL Behavior |
|----------|----------|---------------|---------------|
| BETWEEN | Continuous ranges | Yes (range scan) | NULL BETWEEN x AND y → UNKNOWN |
| IN | Discrete set of values | Yes (multiple equality checks) | NULL IN (...) → UNKNOWN |
| LIKE | Pattern matching | Only with trailing wildcard | NULL LIKE '%x' → UNKNOWN |

---

## Part 7: NULL Handling

### 7.1 Definition

`NULL` in SQL represents the **absence of a value** — not zero, not an empty string, not false. It means "unknown" or "not applicable."

### 7.2 The Three-Valued Logic

SQL uses three-valued logic: TRUE, FALSE, and UNKNOWN. Any comparison involving NULL yields UNKNOWN.

```
NULL = NULL     → UNKNOWN  (NOT TRUE!)
NULL <> NULL    → UNKNOWN
NULL > 5        → UNKNOWN
NULL = 0        → UNKNOWN
NULL = ''       → UNKNOWN
```

**This is the single most important concept for SQL interviews regarding NULLs.**

### 7.3 Testing for NULL

```sql
-- Correct
WHERE column IS NULL
WHERE column IS NOT NULL

-- WRONG (always returns no rows)
WHERE column = NULL
WHERE column <> NULL
```

### 7.4 NULL in Boolean Logic

| Expression | Result |
|-----------|--------|
| TRUE AND UNKNOWN | UNKNOWN |
| FALSE AND UNKNOWN | FALSE |
| TRUE OR UNKNOWN | TRUE |
| FALSE OR UNKNOWN | UNKNOWN |
| NOT UNKNOWN | UNKNOWN |

### 7.5 COALESCE and IFNULL/ISNULL

`COALESCE` returns the first non-NULL value from its arguments. It is ANSI standard and works in all major databases.

```sql
SELECT 
    first_name,
    commission,
    COALESCE(commission, 0) AS effective_commission
FROM employees;
```

| first_name | commission | effective_commission |
|-----------|-----------|---------------------|
| Alice     | NULL      | 0.00                |
| Bob       | NULL      | 0.00                |
| Charlie   | 5000.00   | 5000.00             |
| Diana     | 7500.00   | 7500.00             |
| Eve       | 3000.00   | 3000.00             |

**Dialect-specific alternatives:**

```sql
-- MySQL
IFNULL(commission, 0)

-- SQL Server
ISNULL(commission, 0)

-- Oracle
NVL(commission, 0)

-- COALESCE (standard, all databases)
COALESCE(commission, 0)
```

**COALESCE with multiple fallbacks:**
```sql
SELECT COALESCE(phone_work, phone_home, phone_mobile, 'No phone') AS contact_number
FROM contacts;
```

### 7.6 NULL in Aggregations

| Function | NULL Behavior |
|----------|--------------|
| COUNT(*) | Counts all rows including NULLs |
| COUNT(column) | Counts only non-NULL values |
| SUM(column) | Ignores NULLs |
| AVG(column) | Ignores NULLs (divides by non-NULL count) |
| MIN/MAX | Ignore NULLs |

**Critical interview example:**

```sql
SELECT 
    COUNT(*) AS total_rows,
    COUNT(commission) AS with_commission,
    AVG(commission) AS avg_commission
FROM employees;
```

Non-NULL commissions from the data:
- Charlie: 5000, Diana: 7500, Eve: 3000, Jack: 2000, Karen: 1500, Mary: 4000
- Count = 6, Sum = 23000

| total_rows | with_commission | avg_commission |
|-----------|----------------|----------------|
| 15        | 6              | 3833.33        |

**The trap:** AVG ignores NULLs. If you want to treat NULLs as 0, use `AVG(COALESCE(commission, 0))` which gives 23000/15 = 1533.33 — a very different answer!

### 7.7 Common Mistakes

1. **Using `= NULL` instead of `IS NULL`:** The most common beginner mistake.
2. **Forgetting NULL in NOT IN:** `WHERE x NOT IN (1, 2, NULL)` returns nothing.
3. **AVG ignoring NULLs:** Can lead to misleading business metrics if not accounted for.
4. **Concatenation with NULL:** In most databases, `'Hello' || NULL` = NULL (not 'Hello').
5. **ORDER BY with NULLs:** Different databases sort NULLs differently (see ORDER BY section).

### 7.8 Interview Explanation

> "NULL represents the absence of a value, not zero or empty string. SQL uses three-valued logic where any comparison with NULL yields UNKNOWN, which is why we must use IS NULL / IS NOT NULL rather than = NULL. Aggregate functions like SUM, AVG, MIN, MAX ignore NULLs, while COUNT(*) counts all rows but COUNT(column) counts only non-NULL values. This distinction between COUNT(*) and COUNT(column) is a common source of incorrect results in analytics."

---

## Part 8: CASE WHEN

### 8.1 Definition

`CASE WHEN` is SQL's conditional expression. It evaluates conditions in order and returns a value for the first condition that is true. It is analogous to `if/elif/else` in Python.

### 8.2 Syntax

**Simple CASE (compares one expression to multiple values):**
```sql
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ELSE default_result
END
```

**Searched CASE (evaluates independent boolean conditions):**
```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END
```

### 8.3 Worked Examples

**Example 1: Salary bands**
```sql
SELECT 
    first_name,
    salary,
    CASE
        WHEN salary >= 100000 THEN 'Senior'
        WHEN salary >= 80000  THEN 'Mid-Level'
        WHEN salary >= 65000  THEN 'Junior'
        ELSE 'Entry'
    END AS salary_band
FROM employees
ORDER BY salary DESC;
```

| first_name | salary    | salary_band |
|-----------|-----------|-------------|
| Irene     | 105000.00 | Senior      |
| Alice     | 95000.00  | Mid-Level   |
| Leo       | 92000.00  | Mid-Level   |
| Eve       | 91000.00  | Mid-Level   |
| Bob       | 87000.00  | Mid-Level   |
| Nathan    | 87000.00  | Mid-Level   |
| Henry     | 78000.00  | Junior      |
| Jack      | 73000.00  | Junior      |
| Charlie   | 72000.00  | Junior      |
| Karen     | 69000.00  | Junior      |
| Diana     | 68000.00  | Junior      |
| Mary      | 68000.00  | Junior      |
| Frank     | 64000.00  | Entry       |
| Olivia    | 64000.00  | Entry       |
| Grace     | 61000.00  | Entry       |

**Example 2: CASE with aggregation (pivot-style counting)**
```sql
SELECT 
    department,
    COUNT(*) AS total,
    COUNT(CASE WHEN salary >= 80000 THEN 1 END) AS high_earners,
    COUNT(CASE WHEN salary < 80000 THEN 1 END) AS standard_earners
FROM employees
GROUP BY department;
```

| department  | total | high_earners | standard_earners |
|-------------|-------|-------------|-----------------|
| Engineering | 5     | 4           | 1               |
| HR          | 4     | 0           | 4               |
| Marketing   | 2     | 0           | 2               |
| Sales       | 4     | 1           | 3               |

**How this works:** `CASE WHEN salary >= 80000 THEN 1 END` returns 1 for matching rows and NULL otherwise. `COUNT(...)` counts non-NULL values, effectively counting only the matching rows.

**Example 3: CASE in ORDER BY**
```sql
-- Custom sort: Sales first, then Engineering, then everything else
SELECT first_name, department FROM employees
ORDER BY 
    CASE department
        WHEN 'Sales' THEN 1
        WHEN 'Engineering' THEN 2
        ELSE 3
    END;
```

**Example 4: CASE for NULL-safe comparison**
```sql
SELECT 
    first_name,
    salary + COALESCE(commission, 0) AS total_compensation,
    CASE 
        WHEN commission IS NOT NULL THEN 'Commission-based'
        ELSE 'Fixed salary'
    END AS pay_type
FROM employees;
```

### 8.4 Nested CASE

```sql
SELECT 
    first_name, department, salary,
    CASE 
        WHEN department = 'Engineering' THEN
            CASE 
                WHEN salary > 100000 THEN 'Principal Engineer'
                WHEN salary > 85000  THEN 'Senior Engineer'
                ELSE 'Engineer'
            END
        WHEN department = 'Sales' THEN
            CASE 
                WHEN salary > 80000 THEN 'Senior Sales'
                ELSE 'Sales Rep'
            END
        ELSE 'General Staff'
    END AS title
FROM employees;
```

While nested CASE works, deeply nested conditions reduce readability. Consider using a lookup table for complex mappings in production.

### 8.5 Common Mistakes

1. **Missing ELSE:** If no WHEN condition matches and ELSE is omitted, the result is NULL (not an error). Always include ELSE unless NULL is the intended default.
2. **Condition order matters:** Conditions are evaluated top-to-bottom, and the first match wins. If you put `WHEN salary >= 65000` before `WHEN salary >= 100000`, everything above 65000 gets caught by the first condition.
3. **Using CASE WHERE instead of CASE WHEN:** Common syntax error.

### 8.6 Interview Explanation

> "CASE WHEN is SQL's conditional expression — the equivalent of if/else in programming. It evaluates conditions sequentially, returns the result for the first true condition, and falls through to ELSE if none match. I use it for data categorization, conditional aggregation, pivot-style transformations, and custom sorting. A key detail is that condition order matters — the first matching WHEN wins."

---

## Part 9: Aggregate Functions

### 9.1 Overview

Aggregate functions compute a single result from a set of rows. They are the foundation of data analysis in SQL.

| Function | Returns | NULL Handling |
|----------|---------|---------------|
| `COUNT(*)` | Number of rows | Counts ALL rows (incl. NULLs) |
| `COUNT(col)` | Number of non-NULL values | Ignores NULLs |
| `COUNT(DISTINCT col)` | Number of unique non-NULL values | Ignores NULLs |
| `SUM(col)` | Total of values | Ignores NULLs |
| `AVG(col)` | Mean of values | Ignores NULLs (divides by non-NULL count) |
| `MIN(col)` | Smallest value | Ignores NULLs |
| `MAX(col)` | Largest value | Ignores NULLs |

### 9.2 Worked Examples

**Example 1: Basic aggregation**
```sql
SELECT 
    COUNT(*) AS total_employees,
    COUNT(commission) AS employees_with_commission,
    COUNT(DISTINCT department) AS num_departments,
    SUM(salary) AS total_payroll,
    AVG(salary) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees;
```

Calculations:
- COUNT(*) = 15
- COUNT(commission) = 6 (Charlie, Diana, Eve, Jack, Karen, Mary)
- COUNT(DISTINCT department) = 4 (Engineering, Sales, HR, Marketing)
- SUM(salary) = 95000+87000+72000+68000+91000+64000+61000+78000+105000+73000+69000+92000+68000+87000+64000 = 1,174,000
- AVG(salary) = 1,174,000/15 = 78,266.67
- MIN(salary) = 61000
- MAX(salary) = 105000

| total_employees | employees_with_commission | num_departments | total_payroll | avg_salary | min_salary | max_salary |
|----------------|--------------------------|----------------|---------------|-----------|------------|------------|
| 15             | 6                        | 4              | 1174000.00    | 78266.67  | 61000.00   | 105000.00  |

**Example 2: Aggregation with GROUP BY**
```sql
SELECT 
    department,
    COUNT(*) AS headcount,
    ROUND(AVG(salary), 2) AS avg_salary,
    SUM(salary) AS dept_payroll
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;
```

| department  | headcount | avg_salary | dept_payroll |
|-------------|-----------|-----------|-------------|
| Engineering | 5         | 93200.00  | 466000.00   |
| Sales       | 4         | 74750.00  | 299000.00   |
| Marketing   | 2         | 71000.00  | 142000.00   |
| HR          | 4         | 66750.00  | 267000.00   |

### 9.3 COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)

This distinction comes up constantly in interviews:

```sql
SELECT
    COUNT(*) AS count_star,              -- 15 (all rows)
    COUNT(commission) AS count_col,      -- 6  (non-NULL commissions)
    COUNT(DISTINCT city) AS count_dist   -- 4  (New York, Chicago, San Francisco, Boston — NULL excluded)
FROM employees;
```

| count_star | count_col | count_dist |
|-----------|----------|------------|
| 15        | 6        | 4          |

**Key insight:** `COUNT(DISTINCT city)` returns 4, not 5, because Olivia's NULL city is not counted. NULLs are excluded from COUNT(column) and COUNT(DISTINCT column), but NOT from COUNT(*).

### 9.4 SUM and AVG with NULLs — The Danger

```sql
-- Scenario: What is the average commission?

-- Approach 1: AVG ignores NULLs
SELECT AVG(commission) FROM employees;
-- = (5000+7500+3000+2000+1500+4000) / 6 = 23000/6 = 3833.33

-- Approach 2: Treat NULL as 0
SELECT AVG(COALESCE(commission, 0)) FROM employees;
-- = 23000 / 15 = 1533.33
```

**Business implication:** Approach 1 answers "What is the average commission among those who earn commission?" Approach 2 answers "What is the average commission across all employees?" The correct choice depends on the business question.

### 9.5 Aggregation Rule: SELECT and GROUP BY

**Rule:** When using GROUP BY, every column in SELECT must either:
1. Be in the GROUP BY clause, OR
2. Be inside an aggregate function.

```sql
-- ✅ Correct
SELECT department, AVG(salary) FROM employees GROUP BY department;

-- ❌ Error (first_name is not grouped or aggregated)
SELECT department, first_name, AVG(salary) FROM employees GROUP BY department;
-- MySQL with ONLY_FULL_GROUP_BY disabled will run this but return arbitrary first_name values!
```

### 9.6 SUM of Empty Set vs SUM of NULLs

```sql
-- No matching rows
SELECT SUM(salary) FROM employees WHERE department = 'Legal';
-- Result: NULL (not 0!)

-- All matching values are NULL
SELECT SUM(commission) FROM employees WHERE department = 'Engineering';
-- Result: NULL (all Engineering commissions are NULL)

-- Safe version
SELECT COALESCE(SUM(salary), 0) FROM employees WHERE department = 'Legal';
-- Result: 0
```

### 9.7 Common Mistakes

1. **Assuming SUM returns 0 for no rows:** It returns NULL. Use `COALESCE(SUM(x), 0)`.
2. **Confusing COUNT(*) and COUNT(column):** Leads to wrong counts when NULLs are present.
3. **AVG denominator confusion:** AVG uses the count of non-NULL values, not the count of all rows.
4. **Adding non-aggregated columns to SELECT with GROUP BY:** In strict SQL mode, this raises an error. MySQL in permissive mode silently returns arbitrary values, which is worse.
5. **Integer division in AVG:** In some databases, `AVG(integer_column)` performs integer division. Cast to DECIMAL first: `AVG(CAST(salary AS DECIMAL))`.

---

## Part 10: SQL Logical Execution Order

### 10.1 Why This Matters

The order in which you write SQL clauses is NOT the order in which the database processes them. Understanding logical execution order explains:
- Why you can't use a SELECT alias in WHERE.
- Why HAVING can filter on aggregates but WHERE cannot.
- Why ORDER BY can reference aliases.

### 10.2 Writing Order vs Execution Order

```
Writing Order:          Logical Execution Order:
─────────────          ───────────────────────
1. SELECT              1. FROM        ← Identify the table(s)
2. FROM                2. WHERE       ← Filter individual rows
3. WHERE               3. GROUP BY    ← Group rows
4. GROUP BY            4. HAVING      ← Filter groups
5. HAVING              5. SELECT      ← Choose columns/expressions
6. ORDER BY            6. DISTINCT    ← Remove duplicates
7. LIMIT               7. ORDER BY    ← Sort results
                       8. LIMIT       ← Restrict output rows
```

### 10.3 ASCII Diagram

```
┌──────────────────────────────────────────────────┐
│                  SQL Query Flow                   │
├──────────────────────────────────────────────────┤
│                                                   │
│  1. FROM / JOIN     → Combine source tables       │
│         ↓                                         │
│  2. WHERE           → Filter rows (before groups) │
│         ↓                                         │
│  3. GROUP BY        → Create groups               │
│         ↓                                         │
│  4. HAVING          → Filter groups               │
│         ↓                                         │
│  5. SELECT          → Evaluate expressions/aliases│
│         ↓                                         │
│  6. DISTINCT        → Remove duplicate rows       │
│         ↓                                         │
│  7. ORDER BY        → Sort the result set         │
│         ↓                                         │
│  8. LIMIT / TOP     → Return subset of rows       │
│                                                   │
└──────────────────────────────────────────────────┘
```

### 10.4 Practical Implications

**Why you can't use a SELECT alias in WHERE:**
```sql
-- ❌ Error: alias not yet defined when WHERE executes
SELECT salary * 12 AS annual_salary FROM employees WHERE annual_salary > 100000;

-- ✅ Fix: repeat the expression
SELECT salary * 12 AS annual_salary FROM employees WHERE salary * 12 > 100000;

-- ✅ Alternative: use a subquery
SELECT * FROM (SELECT salary * 12 AS annual_salary FROM employees) t WHERE annual_salary > 100000;
```

Note: MySQL allows alias references in WHERE as an extension, but this is non-standard.

**Why ORDER BY CAN use aliases:**
```sql
-- ✅ This works because ORDER BY executes AFTER SELECT
SELECT salary * 12 AS annual_salary FROM employees ORDER BY annual_salary DESC;
```

**Why HAVING filters groups, not rows:**
```sql
-- HAVING runs after GROUP BY, so it can filter on aggregates
SELECT department, AVG(salary) AS avg_sal
FROM employees
GROUP BY department
HAVING AVG(salary) > 75000;
```

| department  | avg_sal  |
|-------------|----------|
| Engineering | 93200.00 |

### 10.5 Interview Explanation

> "SQL has a logical execution order that differs from the writing order. It goes: FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT. This explains why you can't reference a SELECT alias in WHERE (WHERE runs before SELECT), but you can in ORDER BY (ORDER BY runs after SELECT). It also explains why WHERE filters individual rows before grouping, while HAVING filters after grouping and can use aggregate results."

---

## Part 11: Comparisons and Commonly Confused Concepts

### 11.1 WHERE vs HAVING

| Aspect | WHERE | HAVING |
|--------|-------|--------|
| **When it runs** | Before GROUP BY | After GROUP BY |
| **Filters** | Individual rows | Groups (aggregated results) |
| **Can use aggregates?** | ❌ No | ✅ Yes |
| **Can use non-aggregated columns?** | ✅ Yes | ✅ Yes (if in GROUP BY) |
| **Performance** | Faster (reduces rows before grouping) | Slower (groups formed first, then filtered) |

```sql
-- WHERE: filter rows before grouping
SELECT department, AVG(salary) 
FROM employees 
WHERE hire_date >= '2020-01-01'  -- Filter rows first
GROUP BY department;

-- HAVING: filter groups after aggregation
SELECT department, AVG(salary) AS avg_sal
FROM employees 
GROUP BY department
HAVING AVG(salary) > 75000;     -- Filter groups

-- Both: filter rows, then filter groups
SELECT department, AVG(salary) AS avg_sal
FROM employees 
WHERE hire_date >= '2018-01-01'  -- Keep only employees hired from 2018
GROUP BY department
HAVING COUNT(*) >= 3;            -- Keep only departments with 3+ such employees
```

**Interview trap:** "Can you use WHERE instead of HAVING?" Only if you're filtering on non-aggregated columns. You CANNOT write `WHERE COUNT(*) > 3` — you must use HAVING.

### 11.2 COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)

| Expression | What It Counts | NULL Handling |
|-----------|---------------|---------------|
| `COUNT(*)` | All rows, period | Includes NULLs |
| `COUNT(col)` | Non-NULL values in column | Excludes NULLs |
| `COUNT(DISTINCT col)` | Unique non-NULL values | Excludes NULLs |

### 11.3 BETWEEN vs >= AND <=

| Aspect | BETWEEN | >= AND <= |
|--------|---------|-----------|
| Readability | More readable for ranges | More explicit |
| Inclusivity | Always inclusive both ends | You control with < or <= |
| Negation | NOT BETWEEN excludes endpoints | More flexible negation |
| Date safety | Can miss times within a day | Allows `< next_day` pattern |

**Best practice for date ranges:** Use `>= start_date AND < day_after_end_date` for safety with datetime columns.

### 11.4 COALESCE vs IFNULL vs ISNULL

| Function | Standard? | Arguments | Database |
|----------|-----------|-----------|----------|
| COALESCE | ✅ ANSI SQL | Any number | All |
| IFNULL | ❌ | Exactly 2 | MySQL |
| ISNULL | ❌ | Exactly 2 | SQL Server |
| NVL | ❌ | Exactly 2 | Oracle |

**Recommendation:** Use `COALESCE` everywhere for portability.

---

## Part 12: Interview Question Bank

### Essential Interview Questions

**Q1: What is the difference between COUNT(*) and COUNT(column_name)?**

> `COUNT(*)` counts all rows including those with NULLs. `COUNT(column_name)` counts only rows where that column is not NULL. For example, if a table has 100 rows and 20 have NULL in the `email` column, `COUNT(*)` returns 100 but `COUNT(email)` returns 80.

**Q2: What does NULL mean in SQL? Can you compare NULL with = ?**

> NULL represents the absence of a value — it means unknown or not applicable. You cannot use `= NULL` because any comparison with NULL returns UNKNOWN, not TRUE or FALSE. You must use `IS NULL` or `IS NOT NULL`. This is because SQL uses three-valued logic: TRUE, FALSE, and UNKNOWN.

**Q3: Explain the SQL logical execution order.**

> FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT. This means WHERE filters rows before grouping (so it can't use aggregates), HAVING filters after grouping (so it can use aggregates), and ORDER BY runs after SELECT (so it can reference aliases). This order also explains why you generally can't use a SELECT alias in a WHERE clause.

**Q4: What happens if you use LIMIT without ORDER BY?**

> The result is non-deterministic. The database may return any arbitrary set of rows, and the set may differ between executions. Always use ORDER BY before LIMIT to ensure predictable results.

**Q5: Write a query to find employees who earn more than the average salary.**

```sql
SELECT first_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

Note: You cannot write `WHERE salary > AVG(salary)` because WHERE runs before aggregation. You need a subquery (covered in detail on Day 3).

**Q6: What is the difference between WHERE and HAVING?**

> WHERE filters individual rows before grouping and cannot use aggregate functions. HAVING filters groups after GROUP BY and can use aggregate functions. For performance, filter as much as possible in WHERE to reduce the data that needs to be grouped.

**Q7: What does `SELECT DISTINCT department, city FROM employees` return?**

> It returns unique combinations of (department, city), not unique departments AND unique cities separately. DISTINCT applies to the entire row of selected columns.

**Q8: What does BETWEEN do and is it inclusive?**

> BETWEEN tests if a value falls within a range. It is inclusive on BOTH ends. `WHERE x BETWEEN 5 AND 10` is equivalent to `WHERE x >= 5 AND x <= 10`. A common pitfall is using BETWEEN with datetime columns — `BETWEEN '2024-01-01' AND '2024-01-31'` can miss records with times after midnight on Jan 31.

**Q9: Output prediction — what does this query return?**

```sql
SELECT department, COUNT(*) 
FROM employees 
WHERE salary > 70000
GROUP BY department
HAVING COUNT(*) >= 2
ORDER BY COUNT(*) DESC;
```

**Step-by-step:**
1. FROM: all 15 rows
2. WHERE salary > 70000: filters to rows with salary > 70000
   - Alice(95k), Bob(87k), Charlie(72k), Eve(91k), Henry(78k), Jack(73k), Irene(105k), Leo(92k), Nathan(87k) = 9 employees
3. GROUP BY department:
   - Engineering: Alice, Bob, Irene, Leo, Nathan = 5
   - Sales: Charlie, Eve = 2
   - HR: Henry = 1
   - Marketing: Jack = 1
4. HAVING COUNT(*) >= 2: keeps Engineering(5) and Sales(2)
5. ORDER BY COUNT(*) DESC:

| department  | COUNT(*) |
|-------------|----------|
| Engineering | 5        |
| Sales       | 2        |

**Q10: What happens when you do `SUM(column)` on an empty result set?**

> It returns NULL, not 0. This is a common source of bugs. Use `COALESCE(SUM(column), 0)` to return 0 instead.

### Advanced / Follow-up Questions

**Q11: Why is `NOT IN` dangerous with NULLs?**

> If the list in `NOT IN` contains NULL, the entire expression evaluates to UNKNOWN for every row, so no rows are returned. For example, `WHERE dept NOT IN ('Sales', NULL)` expands to `dept <> 'Sales' AND dept <> NULL`. Since `dept <> NULL` is always UNKNOWN, the AND makes the entire condition UNKNOWN. Use `NOT EXISTS` or filter NULLs from the list.

**Q12: Does SQL Server support LIMIT?**

> No. SQL Server uses `TOP n` (placed after SELECT). For pagination with offset, SQL Server 2012+ supports `OFFSET n ROWS FETCH NEXT m ROWS ONLY`, which is also the ANSI SQL standard syntax.

**Q13: Can you use a column alias in WHERE?**

> Not in standard SQL. WHERE is evaluated before SELECT, so aliases defined in SELECT are not yet available. Most databases enforce this. MySQL is an exception and allows it as an extension, but relying on this is non-portable.

**Q14: What is the result of `NULL + 5`?**

> NULL. Any arithmetic operation involving NULL returns NULL. Use COALESCE to substitute NULLs before arithmetic: `COALESCE(column, 0) + 5`.

**Q15: Trick question — what is the output?**

```sql
SELECT COUNT(*), SUM(amount), AVG(amount)
FROM orders
WHERE amount > 1000;
```

> If no orders have amount > 1000 (looking at our data, the max is 990), then COUNT(*) = 0, SUM(amount) = NULL, AVG(amount) = NULL. COUNT(*) returns 0 for empty sets, but SUM and AVG return NULL.

---

## Part 13: Hands-On Practice

### Instructions

Below are 20 practice queries. All use the `employees` and `orders` tables defined at the beginning of these notes. Try each exercise on your own before checking the answer key.

---

### Exercise 1
**List all employees showing their full name (first + last), department, and salary. Sort by salary descending.**

### Exercise 2
**Find all unique cities where employees work.**

### Exercise 3
**List employees in the Engineering department who earn more than $90,000.**

### Exercise 4
**Find employees whose first name starts with the letter 'M' or 'J'.**

### Exercise 5
**List employees hired between January 1, 2020 and December 31, 2021 (inclusive). Show their name, department, and hire date. Sort by hire date.**

### Exercise 6
**Find all employees who do NOT have a commission (commission is NULL).**

### Exercise 7
**Calculate the total, average, minimum, and maximum salary for the entire company.**

### Exercise 8
**Show each employee's name, salary, commission, and total compensation (salary + commission). If commission is NULL, treat it as 0.**

### Exercise 9
**Categorize employees into salary bands:**
- **'High'** if salary >= 90,000
- **'Medium'** if salary >= 70,000
- **'Low'** if salary < 70,000

**Show the employee name, salary, and band.**

### Exercise 10
**Find the number of employees in each department. Sort by headcount descending.**

### Exercise 11
**Find the average salary per department. Only show departments where the average salary exceeds $72,000.**

### Exercise 12
**List the top 5 highest-paid employees.**

### Exercise 13
**Find all orders that are not 'Cancelled'. Show order_id, customer_id, amount, and status. Sort by amount descending.**

### Exercise 14
**How many orders have a NULL amount? How many have a non-NULL amount?**

### Exercise 15
**For each order status, find the total number of orders and the total revenue (sum of amount). Sort by total revenue descending.**

### Exercise 16
**List employees from either 'New York' or 'Boston' who earn more than $85,000.**

### Exercise 17
**Find employees whose last name contains the letter 'a' (case-insensitive). Show their full name and department.**

### Exercise 18
**For each department, count how many employees earn commission and how many do not, using CASE WHEN and aggregation.**

### Exercise 19
**Find all employees whose city is unknown (NULL). List their name, department, and use COALESCE to display 'Not Assigned' for the city.**

### Exercise 20
**Calculate the average order amount per region, but only for completed orders. Exclude regions where the average is below $400.**

---

## Answer Key

---

### Solution 1

```sql
SELECT 
    CONCAT(first_name, ' ', last_name) AS full_name,  -- MySQL
    -- first_name || ' ' || last_name AS full_name,    -- PostgreSQL
    department,
    salary
FROM employees
ORDER BY salary DESC;
```

**Expected output (first 5 rows):**

| full_name       | department  | salary    |
|----------------|-------------|-----------|
| Irene Anderson | Engineering | 105000.00 |
| Alice Johnson  | Engineering | 95000.00  |
| Leo Harris     | Engineering | 92000.00  |
| Eve Davis      | Sales       | 91000.00  |
| Bob Smith      | Engineering | 87000.00  |

---

### Solution 2

```sql
SELECT DISTINCT city FROM employees;
```

**Expected output:**

| city          |
|---------------|
| New York      |
| Chicago       |
| San Francisco |
| Boston        |
| NULL          |

**5 rows** (NULL is a distinct value for DISTINCT purposes).

---

### Solution 3

```sql
SELECT first_name, last_name, salary 
FROM employees 
WHERE department = 'Engineering' AND salary > 90000;
```

| first_name | last_name | salary    |
|-----------|-----------|-----------|
| Alice     | Johnson   | 95000.00  |
| Irene     | Anderson  | 105000.00 |
| Leo       | Harris    | 92000.00  |

---

### Solution 4

```sql
SELECT first_name, last_name FROM employees
WHERE first_name LIKE 'M%' OR first_name LIKE 'J%';
```

| first_name | last_name |
|-----------|-----------|
| Jack      | Thomas    |
| Mary      | Clark     |

**Alternative using IN with SUBSTRING:**
```sql
SELECT first_name, last_name FROM employees
WHERE LEFT(first_name, 1) IN ('M', 'J');
```

---

### Solution 5

```sql
SELECT first_name, department, hire_date
FROM employees
WHERE hire_date BETWEEN '2020-01-01' AND '2021-12-31'
ORDER BY hire_date;
```

| first_name | department  | hire_date  |
|-----------|-------------|------------|
| Bob       | Engineering | 2020-07-01 |
| Jack      | Marketing   | 2020-08-12 |
| Diana     | Sales       | 2021-01-10 |
| Karen     | Marketing   | 2021-03-30 |
| Nathan    | Engineering | 2021-09-05 |

---

### Solution 6

```sql
SELECT first_name, last_name, department
FROM employees
WHERE commission IS NULL;
```

| first_name | last_name | department  |
|-----------|-----------|-------------|
| Alice     | Johnson   | Engineering |
| Bob       | Smith     | Engineering |
| Frank     | Wilson    | HR          |
| Grace     | Taylor    | HR          |
| Henry     | Martinez  | HR          |
| Irene     | Anderson  | Engineering |
| Leo       | Harris    | Engineering |
| Nathan    | Lewis     | Engineering |
| Olivia    | Walker    | HR          |

**9 rows** (15 total - 6 with commission = 9 without).

---

### Solution 7

```sql
SELECT 
    SUM(salary) AS total_salary,
    ROUND(AVG(salary), 2) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees;
```

| total_salary | avg_salary | min_salary | max_salary |
|-------------|-----------|------------|------------|
| 1174000.00  | 78266.67  | 61000.00   | 105000.00  |

---

### Solution 8

```sql
SELECT 
    first_name,
    salary,
    commission,
    salary + COALESCE(commission, 0) AS total_compensation
FROM employees
ORDER BY total_compensation DESC;
```

| first_name | salary    | commission | total_compensation |
|-----------|-----------|------------|-------------------|
| Irene     | 105000.00 | NULL       | 105000.00         |
| Alice     | 95000.00  | NULL       | 95000.00          |
| Eve       | 91000.00  | 3000.00    | 94000.00          |
| Leo       | 92000.00  | NULL       | 92000.00          |
| Bob       | 87000.00  | NULL       | 87000.00          |
| Nathan    | 87000.00  | NULL       | 87000.00          |
| Henry     | 78000.00  | NULL       | 78000.00          |
| Charlie   | 72000.00  | 5000.00    | 77000.00          |
| Diana     | 68000.00  | 7500.00    | 75500.00          |
| Jack      | 73000.00  | 2000.00    | 75000.00          |
| Mary      | 68000.00  | 4000.00    | 72000.00          |
| Karen     | 69000.00  | 1500.00    | 70500.00          |
| Frank     | 64000.00  | NULL       | 64000.00          |
| Olivia    | 64000.00  | NULL       | 64000.00          |
| Grace     | 61000.00  | NULL       | 61000.00          |

---

### Solution 9

```sql
SELECT 
    first_name,
    salary,
    CASE
        WHEN salary >= 90000 THEN 'High'
        WHEN salary >= 70000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_band
FROM employees
ORDER BY salary DESC;
```

| first_name | salary    | salary_band |
|-----------|-----------|-------------|
| Irene     | 105000.00 | High        |
| Alice     | 95000.00  | High        |
| Leo       | 92000.00  | High        |
| Eve       | 91000.00  | High        |
| Bob       | 87000.00  | Medium      |
| Nathan    | 87000.00  | Medium      |
| Henry     | 78000.00  | Medium      |
| Jack      | 73000.00  | Medium      |
| Charlie   | 72000.00  | Medium      |
| Karen     | 69000.00  | Low         |
| Diana     | 68000.00  | Low         |
| Mary      | 68000.00  | Low         |
| Frank     | 64000.00  | Low         |
| Olivia    | 64000.00  | Low         |
| Grace     | 61000.00  | Low         |

---

### Solution 10

```sql
SELECT department, COUNT(*) AS headcount
FROM employees
GROUP BY department
ORDER BY headcount DESC;
```

| department  | headcount |
|-------------|-----------|
| Engineering | 5         |
| Sales       | 4         |
| HR          | 4         |
| Marketing   | 2         |

---

### Solution 11

```sql
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 72000
ORDER BY avg_salary DESC;
```

| department  | avg_salary |
|-------------|-----------|
| Engineering | 93200.00  |
| Sales       | 74750.00  |

(HR avg = 66750 and Marketing avg = 71000, both at or below 72000.)

---

### Solution 12

```sql
-- PostgreSQL / MySQL
SELECT first_name, last_name, salary 
FROM employees 
ORDER BY salary DESC 
LIMIT 5;

-- SQL Server
SELECT TOP 5 first_name, last_name, salary 
FROM employees 
ORDER BY salary DESC;
```

| first_name | last_name | salary    |
|-----------|-----------|-----------|
| Irene     | Anderson  | 105000.00 |
| Alice     | Johnson   | 95000.00  |
| Leo       | Harris    | 92000.00  |
| Eve       | Davis     | 91000.00  |
| Bob       | Smith     | 87000.00  |

**Note:** Nathan also earns 87000. The 5th position is arbitrary between Bob and Nathan. This is a tie-handling issue — discussed in Day 3 with window functions.

---

### Solution 13

```sql
SELECT order_id, customer_id, amount, status
FROM orders
WHERE status <> 'Cancelled'
ORDER BY amount DESC;
```

| order_id | customer_id | amount  | status    |
|---------|-------------|---------|-----------|
| 109     | 1001        | 990.00  | Completed |
| 104     | 1003        | 800.00  | Completed |
| 107     | 1005        | 670.00  | Completed |
| 112     | 1007        | 550.00  | Completed |
| 102     | 1002        | 450.00  | Completed |
| 111     | 1004        | 410.00  | Pending   |
| 106     | 1004        | 320.00  | Completed |
| 101     | 1001        | 250.00  | Completed |
| 110     | 1006        | 0.00    | Completed |
| 105     | 1002        | NULL    | Pending   |

**Note:** Order 105 has NULL amount and is not cancelled, so it appears. NULL sorts at the end in DESC for most databases.

---

### Solution 14

```sql
SELECT 
    COUNT(*) - COUNT(amount) AS null_amount_count,
    COUNT(amount) AS non_null_amount_count
FROM orders;
```

| null_amount_count | non_null_amount_count |
|------------------|----------------------|
| 1                | 11                   |

**Alternative:**
```sql
SELECT 
    SUM(CASE WHEN amount IS NULL THEN 1 ELSE 0 END) AS null_count,
    SUM(CASE WHEN amount IS NOT NULL THEN 1 ELSE 0 END) AS non_null_count
FROM orders;
```

---

### Solution 15

```sql
SELECT 
    status,
    COUNT(*) AS order_count,
    COALESCE(SUM(amount), 0) AS total_revenue
FROM orders
GROUP BY status
ORDER BY total_revenue DESC;
```

Calculations:
- Completed: 250+450+800+320+670+990+0+550 = 4030
- Cancelled: 125+150 = 275
- Pending: NULL + 410 = 410

| status    | order_count | total_revenue |
|-----------|------------|---------------|
| Completed | 8          | 4030.00       |
| Pending   | 2          | 410.00        |
| Cancelled | 2          | 275.00        |

---

### Solution 16

```sql
SELECT first_name, last_name, city, salary
FROM employees
WHERE city IN ('New York', 'Boston') AND salary > 85000;
```

| first_name | last_name | city     | salary    |
|-----------|-----------|----------|-----------|
| Alice     | Johnson   | New York | 95000.00  |
| Bob       | Smith     | New York | 87000.00  |
| Irene     | Anderson  | New York | 105000.00 |
| Leo       | Harris    | New York | 92000.00  |
| Nathan    | Lewis     | Boston   | 87000.00  |

---

### Solution 17

```sql
-- Portable (force case-insensitive)
SELECT 
    CONCAT(first_name, ' ', last_name) AS full_name, 
    department
FROM employees
WHERE LOWER(last_name) LIKE '%a%';
```

**Expected output:**

| full_name        | department  |
|-----------------|-------------|
| Eve Davis       | Sales       |
| Grace Taylor    | HR          |
| Henry Martinez  | HR          |
| Irene Anderson  | Engineering |
| Jack Thomas     | Marketing   |
| Leo Harris      | Engineering |
| Mary Clark      | Sales       |
| Olivia Walker   | HR          |

**8 rows.** (Last names containing 'a': Davis, Taylor, Martinez, Anderson, Thomas, Harris, Clark, Walker)

---

### Solution 18

```sql
SELECT 
    department,
    COUNT(CASE WHEN commission IS NOT NULL THEN 1 END) AS with_commission,
    COUNT(CASE WHEN commission IS NULL THEN 1 END) AS without_commission
FROM employees
GROUP BY department;
```

| department  | with_commission | without_commission |
|-------------|----------------|-------------------|
| Engineering | 0              | 5                 |
| HR          | 0              | 4                 |
| Marketing   | 2              | 0                 |
| Sales       | 4              | 0                 |

---

### Solution 19

```sql
SELECT 
    first_name,
    last_name,
    department,
    COALESCE(city, 'Not Assigned') AS city
FROM employees
WHERE city IS NULL;
```

| first_name | last_name | department | city         |
|-----------|-----------|------------|--------------|
| Olivia    | Walker    | HR         | Not Assigned |

---

### Solution 20

```sql
SELECT 
    region,
    ROUND(AVG(amount), 2) AS avg_amount
FROM orders
WHERE status = 'Completed'
GROUP BY region
HAVING AVG(amount) >= 400
ORDER BY avg_amount DESC;
```

Completed orders by region:
- East: 250 + 320 + 990 = 1560, count=3, avg=520.00
- West: 450 + 0 = 450, count=2, avg=225.00
- North: 800, count=1, avg=800.00
- South: 670 + 550 = 1220, count=2, avg=610.00

HAVING AVG >= 400 keeps: North(800), South(610), East(520).

| region | avg_amount |
|--------|-----------|
| North  | 800.00    |
| South  | 610.00    |
| East   | 520.00    |

---

## Part 14: Case Study — Sales Performance Analysis

### 1. Business Problem

A retail company needs a weekly report summarizing employee sales performance. Management wants to understand:
- Total compensation by department (salary + commission).
- How many employees earn commission vs. fixed salary.
- Which employees are the top earners.
- Salary distribution across departments.

### 2. Requirements and Assumptions

- Use the `employees` table.
- Commission represents variable pay tied to sales performance.
- NULL commission means the employee is not in a commission-eligible role.
- "Total compensation" = salary + COALESCE(commission, 0).
- Management wants results sorted by total compensation descending.

### 3. Input Data

The `employees` table with 15 rows, as defined in our schema.

### 4. Proposed Approach

Build a single report query that:
1. Calculates total compensation per employee.
2. Classifies pay type (Commission-based vs. Fixed).
3. Assigns salary bands (High / Medium / Low).
4. Aggregates metrics per department.

### 5. Implementation

**Step 1: Individual employee report**

```sql
SELECT 
    CONCAT(first_name, ' ', last_name) AS employee_name,
    department,
    salary,
    COALESCE(commission, 0) AS commission,
    salary + COALESCE(commission, 0) AS total_compensation,
    CASE 
        WHEN commission IS NOT NULL THEN 'Commission-based'
        ELSE 'Fixed salary'
    END AS pay_type,
    CASE
        WHEN salary >= 90000 THEN 'High'
        WHEN salary >= 70000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_band
FROM employees
ORDER BY total_compensation DESC;
```

**Step 2: Department-level summary**

```sql
SELECT 
    department,
    COUNT(*) AS headcount,
    ROUND(AVG(salary), 2) AS avg_salary,
    SUM(salary + COALESCE(commission, 0)) AS total_dept_compensation,
    COUNT(CASE WHEN commission IS NOT NULL THEN 1 END) AS commission_earners,
    COUNT(CASE WHEN commission IS NULL THEN 1 END) AS fixed_salary_earners,
    COUNT(CASE WHEN salary >= 90000 THEN 1 END) AS high_band,
    COUNT(CASE WHEN salary >= 70000 AND salary < 90000 THEN 1 END) AS medium_band,
    COUNT(CASE WHEN salary < 70000 THEN 1 END) AS low_band,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees
GROUP BY department
ORDER BY total_dept_compensation DESC;
```

### 6. Expected Results

**Department Summary:**

| department  | headcount | avg_salary | total_dept_compensation | commission_earners | fixed_salary_earners | high | med | low | min_salary | max_salary |
|-------------|-----------|-----------|------------------------|-------------------|---------------------|------|-----|-----|------------|------------|
| Engineering | 5         | 93200.00  | 466000.00              | 0                 | 5                   | 3    | 2   | 0   | 87000      | 105000     |
| Sales       | 4         | 74750.00  | 318500.00              | 4                 | 0                   | 1    | 1   | 2   | 68000      | 91000      |
| HR          | 4         | 66750.00  | 267000.00              | 0                 | 4                   | 0    | 1   | 3   | 61000      | 78000      |
| Marketing   | 2         | 71000.00  | 145500.00              | 2                 | 0                   | 0    | 1   | 1   | 69000      | 73000      |

### 7. Testing and Validation

- **Row count check:** SUM of headcounts should equal 15. (5+4+4+2 = 15 ✅)
- **Salary cross-check:** SUM of total_dept_compensation should equal company-wide total.
  - 466000 + 318500 + 267000 + 145500 = 1,197,000
  - Company total salary = 1,174,000 + total commission (23,000) = 1,197,000 ✅
- **NULL handling check:** Verify commission_earners + fixed_salary_earners = headcount per department.

### 8. Limitations and Trade-offs

- Commission data is a snapshot; it doesn't reflect YTD performance.
- Salary bands are hardcoded — better to use a lookup table for maintainability.
- No time dimension: this is a point-in-time analysis, not a trend.
- No adjustment for role level or experience.

### 9. Likely Interview Follow-ups

1. "How would you handle this if the company had 10,000 employees across 50 departments?" → Index on department, consider materialized views.
2. "What if commission is paid quarterly, not monthly?" → Need a time-series table of commission payments.
3. "How would you add year-over-year salary growth?" → Need historical salary data and window functions (Day 3 topic).
4. "What if some employees belong to multiple departments?" → The schema would need a junction table (many-to-many relationship, Day 4 topic).

---

## Part 15: Compact Revision Sheet

### Essential Definitions

| Term | Definition |
|------|-----------|
| SELECT | Specifies columns/expressions in the result set (projection) |
| DISTINCT | Eliminates duplicate rows across ALL selected columns |
| WHERE | Filters individual rows BEFORE grouping |
| ORDER BY | Sorts the result set; only guarantee of row order |
| LIMIT/TOP | Restricts number of output rows |
| BETWEEN | Inclusive range test: `x BETWEEN a AND b` = `x >= a AND x <= b` |
| IN | Set membership test: `x IN (a, b, c)` = `x=a OR x=b OR x=c` |
| LIKE | Pattern matching: `%` = any chars, `_` = exactly one char |
| NULL | Absence of value; comparison with NULL → UNKNOWN |
| CASE WHEN | Conditional expression; first true condition wins |
| COALESCE | Returns first non-NULL argument |

### Syntax Templates

```sql
-- Basic query
SELECT col1, col2, expr AS alias
FROM table
WHERE condition
ORDER BY col1 DESC
LIMIT n;

-- Aggregation
SELECT group_col, COUNT(*), SUM(col), AVG(col)
FROM table
WHERE row_filter
GROUP BY group_col
HAVING aggregate_filter;

-- CASE WHEN
CASE WHEN cond1 THEN val1 WHEN cond2 THEN val2 ELSE default END

-- NULL handling
WHERE col IS NULL | IS NOT NULL
COALESCE(col, fallback_value)
```

### Critical Rules

1. **Execution order:** FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
2. **NULL ≠ NULL:** Use `IS NULL`, never `= NULL`
3. **COUNT(*)** counts all rows; **COUNT(col)** counts non-NULLs only
4. **AVG** ignores NULLs (divides by non-NULL count)
5. **SUM of empty set** returns NULL, not 0
6. **NOT IN with NULL** returns no rows — use NOT EXISTS
7. **BETWEEN** is inclusive on both ends
8. **LIKE '%text'** forces full table scan (no index use)
9. **AND** has higher precedence than **OR** — use parentheses
10. **LIMIT without ORDER BY** is non-deterministic

### Key Comparisons

| Concept A | Concept B | Key Difference |
|-----------|-----------|----------------|
| WHERE | HAVING | WHERE: before GROUP BY, no aggregates. HAVING: after, can aggregate |
| COUNT(*) | COUNT(col) | * counts all rows; col skips NULLs |
| COALESCE | IFNULL | COALESCE is standard SQL, multi-arg; IFNULL is MySQL-only, 2-arg |
| BETWEEN | >= AND <= | Identical; BETWEEN is syntactic sugar |
| LIMIT | TOP | LIMIT: MySQL/PG, end of query. TOP: SQL Server, after SELECT |

### Common Mistakes Checklist

- [ ] Used `= NULL` instead of `IS NULL`
- [ ] Used `NOT IN` with a list that may contain NULL
- [ ] Forgot `ORDER BY` before `LIMIT`
- [ ] Used `SELECT *` in production code
- [ ] Referenced a SELECT alias in WHERE
- [ ] Confused AVG behavior with NULLs (AVG ignores them)
- [ ] Forgot parentheses with mixed AND/OR
- [ ] Used BETWEEN with datetime without considering time component

---

## Part 16: Closed-Book Self-Assessment (100 Marks)

### Instructions

- Time limit: 60 minutes.
- No notes, no AI assistance.
- Write your answers on paper or in a separate document.
- Check the answer key only after completing all questions.

---

### Section A: Conceptual Questions (30 marks)

**Q1 (3 marks):** What is the SQL logical execution order? List all 8 steps.

**Q2 (3 marks):** Explain the difference between `COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)`. Give an example where all three return different values.

**Q3 (2 marks):** What does NULL represent in SQL? Why can't you use `WHERE column = NULL`?

**Q4 (3 marks):** Explain the difference between WHERE and HAVING. Give one example where you must use HAVING and cannot use WHERE.

**Q5 (2 marks):** Is `BETWEEN 10 AND 20` inclusive or exclusive at the boundaries? What is the equivalent expression using comparison operators?

**Q6 (2 marks):** What happens when you use `NOT IN` with a list that contains NULL? Why?

**Q7 (3 marks):** What is the result of `AVG(commission)` vs `AVG(COALESCE(commission, 0))` if a table has 10 employees, only 4 of whom have non-NULL commissions totaling 20000?

**Q8 (2 marks):** Can you reference a SELECT alias in the WHERE clause? In the ORDER BY clause? Explain why.

**Q9 (3 marks):** What is the difference between `LIMIT` and `TOP`? Name one feature that TOP supports but LIMIT does not.

**Q10 (2 marks):** In which databases are NULLs sorted first in ascending order, and in which are they sorted last?

**Q11 (3 marks):** What does `LIKE '%abc'` match? Can it use a B-tree index? Why or why not?

**Q12 (2 marks):** What does `SUM(salary)` return when no rows match the WHERE clause?

---

### Section B: Output Prediction (25 marks)

Use the `employees` table from the notes.

**Q13 (4 marks):** Predict the output:
```sql
SELECT COUNT(*), COUNT(commission), COUNT(DISTINCT department)
FROM employees;
```

**Q14 (4 marks):** Predict the output:
```sql
SELECT department, MAX(salary) - MIN(salary) AS salary_range
FROM employees
GROUP BY department
HAVING MAX(salary) - MIN(salary) > 20000;
```

**Q15 (4 marks):** Predict the output:
```sql
SELECT first_name, salary
FROM employees
WHERE department = 'HR' OR department = 'Marketing' AND salary > 70000
ORDER BY salary DESC;
```

(Pay careful attention to operator precedence.)

**Q16 (4 marks):** Predict the output:
```sql
SELECT COALESCE(NULL, NULL, 'hello', 'world');
```

**Q17 (5 marks):** Predict the exact output (all rows):
```sql
SELECT department, COUNT(*) AS cnt
FROM employees
WHERE city IS NOT NULL
GROUP BY department
HAVING COUNT(*) >= 3
ORDER BY cnt DESC;
```

**Q18 (4 marks):** What does this return? Explain why.
```sql
SELECT AVG(commission) FROM employees WHERE department = 'Engineering';
```

---

### Section C: Write the Query (30 marks)

**Q19 (5 marks):** Write a query to find all employees whose salary is above the company average. Show name, department, and salary. Sort by salary descending.

*Hint: You need a subquery.*

**Q20 (5 marks):** Write a query to display each department with: the number of employees, the average salary (rounded to 2 decimals), the number of employees earning commission, and the number without commission. Sort by average salary descending.

**Q21 (5 marks):** Write a query to categorize orders into:
- 'High Value' if amount >= 500
- 'Medium Value' if amount >= 200
- 'Low Value' if amount < 200
- 'Unknown' if amount is NULL

Show order_id, amount, category, and status. Sort by amount descending (NULLs last).

**Q22 (5 marks):** Write a query to find regions where the total revenue from completed orders exceeds $500.

**Q23 (5 marks):** Write a query to display the top 3 departments by total compensation (salary + commission, treating NULL commission as 0). Show department name, headcount, and total compensation.

**Q24 (5 marks):** Write a query that shows each employee's first name, city, and a column called `location_status` that shows:
- 'Local' if city is 'New York'
- 'Regional' if city is 'Chicago' or 'Boston'
- 'Remote' if city is 'San Francisco'
- 'Unassigned' if city is NULL

Sort by department, then salary descending.

---

### Section D: Scenario-Based (15 marks)

**Q25 (5 marks):** A junior analyst writes this query and gets unexpected results:
```sql
SELECT department, first_name, AVG(salary)
FROM employees
GROUP BY department;
```
What is wrong? How would you fix it? What would different databases do with this query?

**Q26 (5 marks):** Your team lead asks: "What is the average commission across the company?" You run `SELECT AVG(commission) FROM employees` and get 3833.33. Your lead says "That can't be right, we have 15 employees and total commission is only 23000. The average should be about 1533."

Who is correct? Explain the discrepancy. Write queries that produce both answers and explain when each is appropriate.

**Q27 (5 marks):** You need to retrieve all employees who are NOT in the Sales or HR departments. You write:
```sql
SELECT * FROM employees WHERE department NOT IN ('Sales', 'HR');
```
This works correctly. But a colleague uses a subquery:
```sql
SELECT * FROM employees 
WHERE department NOT IN (SELECT department FROM departments WHERE active = 0);
```
Under what condition would this query return zero rows even if there are valid employees? How would you fix it?

---

### Assessment Answer Key

---

**A1:** FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET

**A2:** `COUNT(*)` counts all rows including NULLs. `COUNT(column)` counts only non-NULL values. `COUNT(DISTINCT column)` counts unique non-NULL values. Example: If a column has values [A, A, B, NULL], then COUNT(*) = 4, COUNT(col) = 3, COUNT(DISTINCT col) = 2.

**A3:** NULL represents the absence of a value (unknown/not applicable). `= NULL` returns UNKNOWN (not TRUE) because SQL uses three-valued logic. NULL is not equal to anything, not even itself. Use `IS NULL` instead.

**A4:** WHERE filters individual rows before GROUP BY and cannot use aggregate functions. HAVING filters groups after GROUP BY and can use aggregates. Example requiring HAVING: `SELECT department, COUNT(*) FROM employees GROUP BY department HAVING COUNT(*) > 3` — you cannot write `WHERE COUNT(*) > 3`.

**A5:** Inclusive on both ends. `x BETWEEN 10 AND 20` is equivalent to `x >= 10 AND x <= 20`.

**A6:** It returns zero rows. `NOT IN (a, b, NULL)` expands to `x<>a AND x<>b AND x<>NULL`. Since `x<>NULL` evaluates to UNKNOWN, the entire AND chain evaluates to UNKNOWN, and the row is excluded.

**A7:** `AVG(commission)` = 20000/4 = 5000. `AVG(COALESCE(commission, 0))` = 20000/10 = 2000. The first ignores NULLs (divides by 4 non-NULL values); the second treats NULLs as 0 (divides by all 10 rows).

**A8:** No, you cannot reference a SELECT alias in WHERE because WHERE executes before SELECT. Yes, you can in ORDER BY because ORDER BY executes after SELECT. (MySQL allows aliases in WHERE as a non-standard extension.)

**A9:** LIMIT is used in MySQL/PostgreSQL/SQLite and appears at the end of the query. TOP is used in SQL Server and appears after SELECT. TOP supports `WITH TIES` (includes tied rows) and `PERCENT` (return a percentage of rows), which LIMIT does not.

**A10:** MySQL and SQL Server sort NULLs first in ASC. PostgreSQL and Oracle sort NULLs last in ASC.

**A11:** `LIKE '%abc'` matches any string ending in 'abc'. It cannot use a B-tree index because the wildcard is at the beginning — the database cannot do a prefix-based lookup and must scan every value.

**A12:** NULL (not 0). Use `COALESCE(SUM(salary), 0)` to return 0 instead.

**A13:**

| COUNT(*) | COUNT(commission) | COUNT(DISTINCT department) |
|----------|------------------|---------------------------|
| 15       | 6                | 4                         |

**A14:**
- Engineering: 105000-87000 = 18000 (excluded: not > 20000)
- Sales: 91000-68000 = 23000 ✅
- HR: 78000-61000 = 17000 (excluded)
- Marketing: 73000-69000 = 4000 (excluded)

| department | salary_range |
|------------|-------------|
| Sales      | 23000.00    |

**A15:** Due to AND having higher precedence than OR, this is evaluated as:
`department = 'HR' OR (department = 'Marketing' AND salary > 70000)`

This returns ALL HR employees plus Marketing employees earning > 70000.

| first_name | salary   |
|-----------|----------|
| Henry     | 78000.00 |
| Jack      | 73000.00 |
| Frank     | 64000.00 |
| Olivia    | 64000.00 |
| Grace     | 61000.00 |

(All 4 HR employees + Jack from Marketing. Karen earns 69000 which is not > 70000, so she's excluded.)

**A16:** `'hello'`. COALESCE returns the first non-NULL argument.

**A17:**
First, filter `WHERE city IS NOT NULL` removes Olivia (emp_id 15).
Remaining 14 employees grouped by department:
- Engineering: 5 (all have non-NULL cities)
- Sales: 4
- HR: 3 (Olivia excluded, leaving Frank, Grace, Henry)
- Marketing: 2

HAVING COUNT(*) >= 3 keeps Engineering(5), Sales(4), HR(3).

| department  | cnt |
|-------------|-----|
| Engineering | 5   |
| Sales       | 4   |
| HR          | 3   |

**A18:** Returns NULL. All 5 Engineering employees have NULL commission. AVG of all NULLs returns NULL (there are zero non-NULL values to average).

**A19:**
```sql
SELECT first_name, department, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees)
ORDER BY salary DESC;
```
(Average salary is approximately 78266.67, so this returns employees earning more than that.)

**A20:**
```sql
SELECT 
    department,
    COUNT(*) AS headcount,
    ROUND(AVG(salary), 2) AS avg_salary,
    COUNT(CASE WHEN commission IS NOT NULL THEN 1 END) AS with_commission,
    COUNT(CASE WHEN commission IS NULL THEN 1 END) AS without_commission
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;
```

**A21:**
```sql
SELECT 
    order_id,
    amount,
    CASE
        WHEN amount IS NULL THEN 'Unknown'
        WHEN amount >= 500 THEN 'High Value'
        WHEN amount >= 200 THEN 'Medium Value'
        ELSE 'Low Value'
    END AS category,
    status
FROM orders
ORDER BY CASE WHEN amount IS NULL THEN 1 ELSE 0 END, amount DESC;
```

Note: The NULL check must come before the numeric comparisons to avoid NULLs falling through to 'Low Value'. For NULLS LAST in databases that support it: `ORDER BY amount DESC NULLS LAST`.

**A22:**
```sql
SELECT region, SUM(amount) AS total_revenue
FROM orders
WHERE status = 'Completed'
GROUP BY region
HAVING SUM(amount) > 500;
```

Expected: East(1560), South(1220), North(800). West total = 450, excluded.

**A23:**
```sql
SELECT 
    department,
    COUNT(*) AS headcount,
    SUM(salary + COALESCE(commission, 0)) AS total_compensation
FROM employees
GROUP BY department
ORDER BY total_compensation DESC
LIMIT 3;
```

| department  | headcount | total_compensation |
|-------------|-----------|-------------------|
| Engineering | 5         | 466000.00         |
| Sales       | 4         | 318500.00         |
| HR          | 4         | 267000.00         |

**A24:**
```sql
SELECT 
    first_name,
    department,
    salary,
    city,
    CASE
        WHEN city = 'New York' THEN 'Local'
        WHEN city IN ('Chicago', 'Boston') THEN 'Regional'
        WHEN city = 'San Francisco' THEN 'Remote'
        WHEN city IS NULL THEN 'Unassigned'
    END AS location_status
FROM employees
ORDER BY department, salary DESC;
```

**A25:** The query is invalid in standard SQL. `first_name` is in SELECT but not in GROUP BY and not inside an aggregate function. In strict SQL mode (PostgreSQL, SQL Server), this raises an error. In MySQL with `ONLY_FULL_GROUP_BY` disabled, it runs but returns an arbitrary `first_name` for each department — this is unpredictable and wrong. Fix: Either add `first_name` to GROUP BY (changes the grouping) or remove it from SELECT, or use an aggregate like `MAX(first_name)`.

**A26:** Both are technically correct — they answer different questions.
- `AVG(commission)` = 23000/6 = 3833.33 → Average commission among commission-earning employees (ignores NULLs).
- `AVG(COALESCE(commission, 0))` = 23000/15 = 1533.33 → Average commission across all employees (treats NULL as 0).

The lead's interpretation (divide by 15) answers "What is the average commission cost per employee?" The query result (divide by 6) answers "What is the average commission among those who earn one?" The right answer depends on the business question.

```sql
-- Lead's answer
SELECT AVG(COALESCE(commission, 0)) FROM employees;  -- 1533.33

-- Query's answer
SELECT AVG(commission) FROM employees;  -- 3833.33
```

**A27:** If the subquery `SELECT department FROM departments WHERE active = 0` returns any NULL value (e.g., a row where `department` column is NULL), then `NOT IN` returns zero rows for every employee (because `NOT IN` with NULL always evaluates to UNKNOWN). Fix: Add `WHERE department IS NOT NULL` to the subquery, or use `NOT EXISTS` instead:

```sql
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d 
    WHERE d.active = 0 AND d.department = e.department
);
```

