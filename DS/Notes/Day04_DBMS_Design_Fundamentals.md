# Day 4: Database Design, Normalization & Storage Internals
## Relational Schema Architecture, Keys, Normal Forms & Indexing

# Part 1: Relational Keys & Integrity Constraints

### 1.1 Types of Keys in Relational Modeling
A **key** is an attribute or set of attributes that uniquely identifies an entity within an entity set or enforces referential constraints between relations.

| Key Type | Formal Definition | Uniqueness | Nullable? | Role in System |
| :--- | :--- | :--- | :--- | :--- |
| **Super Key** | Any set of attributes that uniquely identifies a tuple. | Yes | Yes (if part of composite) | Theoretical superset of candidate keys. |
| **Candidate Key** | A **minimal** super key (no proper subset can uniquely identify the tuple). | Yes | **No** (by best practice) | Eligible to be chosen as Primary Key. |
| **Primary Key (PK)** | The chosen candidate key used by the RDBMS as the primary row identifier. | Yes | **Never** (`NOT NULL` enforced) | Establishes identity; determines clustered index order in MySQL/SQL Server. |
| **Alternate Key** | Candidate keys not selected as the primary key. | Yes | Generally No | Enforced via `UNIQUE` constraints. |
| **Foreign Key (FK)** | An attribute in a child table referencing a Primary or Candidate key in a parent table. | No (1:N) or Yes (1:1) | **Yes** (unless explicitly constrained) | Enforces Referential Integrity across relations. |
| **Composite Key** | A key composed of two or more attributes. | Yes | No | Identifies junction/bridge tables in M:N relationships. |
| **Surrogate Key** | A synthetic, system-generated identifier (e.g., `AUTO_INCREMENT`, `UUID`). | Yes | Never | Decouples internal identity from changing business attributes. |
| **Natural Key** | A business attribute with real-world meaning (e.g., SSN, ISBN, VIN). | Yes | Never | Subject to external business changes and data entry errors. |

---

### 1.2 Referential Integrity & Cascade Actions
A foreign key constraint ensures that a child row cannot reference a non-existent parent row. When a referenced parent row is updated or deleted, the database can trigger one of four behaviors:

1. **`ON DELETE RESTRICT` (or `NO ACTION`):** Prevents deletion of the parent row if any referencing child row exists. Raises a foreign key violation error. (Default behavior).
2. **`ON DELETE CASCADE`:** Automatically deletes all referencing child rows when the parent row is deleted.
   - *Example:* Deleting an `Order` cascades to delete all associated `OrderItems`.
3. **`ON DELETE SET NULL`:** Sets the foreign key column in all child rows to `NULL` when the parent row is deleted. (Requires foreign key column to be nullable).
   - *Example:* Deleting a `Department` sets `dept_id` to `NULL` for all affected `Employees`.
4. **`ON DELETE SET DEFAULT`:** Resets the foreign key column in child rows to a predefined default value.

```sql
CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) CHECK (unit_price >= 0),
    CONSTRAINT fk_order FOREIGN KEY (order_id) 
        REFERENCES orders(order_id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE,
    CONSTRAINT fk_product FOREIGN KEY (product_id) 
        REFERENCES products(product_id) 
        ON DELETE RESTRICT
);
```

---

# Part 2: Normalization & Normal Forms

### 2.1 The Purpose of Normalization
Normalization is a systematic technique for organizing data in a database to achieve two primary objectives:
1. **Eliminate Redundant Data:** Storing the same fact in multiple places wastes storage and increases I/O overhead.
2. **Eliminate Data Modification Anomalies:**
   - **Insertion Anomaly:** Inability to record a fact about an entity without artificially fabricating an unrelated entity (e.g., cannot add a new Department until an Employee is hired into it).
   - **Update Anomaly:** Updating a repeated fact in one row while missing it in another leads to internal data inconsistency (e.g., changing a department's location in 3 rows while 2 rows still show the old location).
   - **Deletion Anomaly:** Deleting an entity unintentionally purges critical information about another entity (e.g., deleting the last employee in a department deletes the department record itself).

---

### 2.2 Functional Dependency (FD)
Let $R$ be a relation schema, and let $X$ and $Y$ be subsets of attributes of $R$.
A **Functional Dependency** $X \rightarrow Y$ ($X$ functionally determines $Y$) states that for any two valid tuples $t_1$ and $t_2$ in $R$:
$$\text{If } t_1[X] = t_2[X], \text{ then } t_1[Y] = t_2[Y]$$
- $X$ is called the **Determinant**.
- $Y$ is called the **Dependent**.

---

### 2.3 Step-by-Step Walkthrough of the Normal Forms

#### First Normal Form (1NF)
**Rule:**
1. Each column must contain only **atomic (indivisible) values**. No arrays, comma-separated lists, or nested tables.
2. Each column must contain values of a single consistent data type.
3. Each record must be uniquely identifiable (a primary key must exist).

*Violation Example:*
| order_id | customer_name | product_ids |
| :--- | :--- | :--- |
| 101 | Alice | 1, 4, 7 |

*1NF Compliant Solution:*
Split multivalued rows so that each row contains exactly one product:
| order_id | product_id | customer_name |
| :--- | :--- | :--- |
| 101 | 1 | Alice |
| 101 | 4 | Alice |
| 101 | 7 | Alice |

---

#### Second Normal Form (2NF)
**Rule:**
1. The relation must be in **1NF**.
2. **No Partial Dependencies:** Every non-prime attribute (an attribute not part of any candidate key) must be **fully functionally dependent** on the **entire** candidate key, not on a proper subset of it.
*(Note: If a table's primary key consists of a single column, the table is automatically in 2NF if it is in 1NF).*

*Violation Example:*
- Primary Key: `(order_id, product_id)` (Composite)
- Attributes: `quantity`, `product_name`, `product_price`
- Functional Dependencies:
  - `(order_id, product_id) -> quantity` (Full dependency: quantity requires both)
  - `product_id -> product_name, product_price` (**Partial Dependency!** Product name depends only on `product_id`, which is a subset of the composite PK).

*2NF Decomposition:*
Split into two tables:
1. `order_items (order_id, product_id, quantity)` (PK: `order_id, product_id`)
2. `products (product_id, product_name, product_price)` (PK: `product_id`)

---

#### Third Normal Form (3NF)
**Rule:**
1. The relation must be in **2NF**.
2. **No Transitive Dependencies:** No non-prime attribute may be functionally dependent on another non-prime attribute ($X \rightarrow Y$ and $Y \rightarrow Z$, where $X$ is the primary key and $Y, Z$ are non-prime attributes).
- *Formal Definition:* For every non-trivial functional dependency $X \rightarrow Y$, either:
  - $X$ is a **Super Key**, OR
  - $Y$ is a **Prime Attribute** (part of a candidate key).

*Violation Example:*
- Table: `employees (emp_id, emp_name, dept_id, dept_name, dept_location)`
- Primary Key: `emp_id`
- Functional Dependencies:
  - `emp_id -> dept_id`
  - `dept_id -> dept_name, dept_location`
  - Hence, `emp_id -> dept_name` is a **transitive dependency** through `dept_id`. If all employees in Marketing leave, the information that Marketing is located in London is deleted (Deletion Anomaly).

*3NF Decomposition:*
Split into two tables:
1. `employees (emp_id, emp_name, dept_id)` (PK: `emp_id`, FK: `dept_id`)
2. `departments (dept_id, dept_name, dept_location)` (PK: `dept_id`)

---

#### Boyce-Codd Normal Form (BCNF) — "3.5NF"
**Rule:**
A relation is in BCNF if and only if for **every** non-trivial functional dependency $X \rightarrow Y$, **$X$ is a Super Key**.
- BCNF eliminates the second condition of 3NF ("$Y$ is a prime attribute"). Under BCNF, *every determinant must be a candidate key*.

*When does 3NF differ from BCNF?*
When a relation has:
1. Multiple overlapping composite candidate keys.

*Classic BCNF Violation Example:*
- Schema: `StudentAdvisor (student_id, subject, advisor)`
- Business Rules:
  - Each student can have multiple advisors, but only one advisor per subject: `(student_id, subject) -> advisor`
  - Each advisor advises only **one** subject: `advisor -> subject`
- Candidate Keys: `(student_id, subject)` and `(student_id, advisor)`
- Is it in 3NF?
  - For `advisor -> subject`: `advisor` is not a super key, but `subject` is a **prime attribute** (part of Candidate Key 1). Therefore, it passes 3NF!
- Is it in BCNF?
  - **No.** Because in `advisor -> subject`, the determinant `advisor` is **not a super key**.
  - *Anomaly:* You cannot add a new advisor and their subject until at least one student is assigned to that advisor.

*BCNF Decomposition:*
1. `AdvisorSubject (advisor, subject)` (PK: `advisor`)
2. `StudentAdvisor (student_id, advisor)` (PK: `student_id, advisor`, FK: `advisor`)

---

# Part 3: Entity-Relationship (ER) Modeling & Relationships

```
┌──────────────────┐               ┌──────────────────┐
│     Customer     │1             N│      Orders      │
│──────────────────│───────────────│──────────────────│
│ customer_id (PK) │  places       │ order_id (PK)    │
│ customer_name    │               │ customer_id (FK) │
└──────────────────┘               └──────────────────┘
                                            │1
                                            │
                                            │contains
                                            │
                                            │N
                                   ┌──────────────────┐
                                   │   Order_Items    │
                                   │──────────────────│
                                   │ item_id (PK)     │
                                   │ order_id (FK)    │
                                   │ product_id (FK)  │
                                   │ quantity         │
                                   └──────────────────┘
                                            │N
                                            │
                                            │references
                                            │
                                            │1
                                   ┌──────────────────┐
                                   │     Products     │
                                   │──────────────────│
                                   │ product_id (PK)  │
                                   │ product_name     │
                                   │ category_id (FK) │
                                   └──────────────────┘
```

### 3.1 Relationship Cardinalities
1. **One-to-One (1:1):**
   - *Example:* `User` to `UserProfile`.
   - *Implementation:* Place the PK of `User` as a FK in `UserProfile` with a `UNIQUE` constraint.
2. **One-to-Many (1:N):**
   - *Example:* `Department` to `Employees`.
   - *Implementation:* Place the parent's PK (`dept_id`) as a FK in the child table (`employees`).
3. **Many-to-Many (M:N):**
   - *Example:* `Students` to `Courses`, or `Orders` to `Products`.
   - *Implementation:* Cannot be implemented directly in relational engines. Requires an intermediate **Junction / Bridge / Associative Entity** (`OrderItems`, `Enrollments`) containing foreign keys to both entities.

---

# Part 4: Views vs. Materialized Views

| Feature | Standard View (Virtual Table) | Materialized View (Physical Snapshot) |
| :--- | :--- | :--- |
| **Storage** | Schema metadata only (No disk data). | Stored physically on disk as a concrete table. |
| **Execution Cost** | Evaluated dynamically on every query. | Read directly from disk ($O(1)$ initial read). |
| **Data Freshness** | Always 100% real-time and up-to-date. | Stale until refreshed (manual, scheduled, or trigger). |
| **Write Impact** | Zero impact on `INSERT`/`UPDATE` to underlying tables. | Overhead during refresh cycles. |
| **Indexability** | Cannot be indexed (except SQL Server Indexed Views). | Can be indexed like any physical table (B-Trees, etc.). |
| **Ideal Use Case** | Enforcing column-level security, simplifying joins. | Heavy analytical queries, aggregations, reporting dashboards. |

```sql
-- Creating a Materialized View in PostgreSQL
CREATE MATERIALIZED VIEW mv_monthly_sales_summary AS
SELECT 
    DATE_TRUNC('month', order_date) AS sales_month,
    category,
    COUNT(order_id) AS total_orders,
    SUM(total_amount) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY DATE_TRUNC('month', order_date), category;

-- Refreshing the Materialized View
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_sales_summary;
```

---

# Part 5: Storage Internals & Indexing Architecture

### 5.1 What is an Index?
An index is an auxiliary data structure (most commonly a self-balancing **B-Tree** or **B+ Tree**) that enables the database engine to find specific rows in $O(\log N)$ time, avoiding expensive full table scans ($O(N)$).

### 5.2 B+ Tree Architecture
In a B+ Tree:
- **Internal Nodes:** Store search key values and child pointers to direct query traversal.
- **Leaf Nodes:** Stored on disk pages linked sequentially as a **doubly-linked list**. Leaf nodes contain either the actual data rows (Clustered) or pointers to data rows (Non-Clustered).
- **Height:** Typically 3 to 4 levels even for millions of rows. A search requires only 3 to 4 page I/O reads.

```
                  [ 50 | 100 ]            <-- Root Node
                 /     |      \
     [ 20 | 35 ]   [ 65 | 80 ]  [ 120 | 150 ] <-- Internal Nodes
       /    \        /    \       /     \
    [Leaves: Doubly-Linked List at Base Layer]
    [Page 1] <---> [Page 2] <---> [Page 3] <---> [Page 4]
```

---

### 5.3 Clustered vs. Non-Clustered Indexes

| Dimension | Clustered Index | Non-Clustered Index (Secondary) |
| :--- | :--- | :--- |
| **Physical Order** | Dictates the physical sort order of rows on disk. | Logical structure separate from physical row storage. |
| **Leaf Node Content**| Contains the **actual data pages** (the full row). | Contains the index key columns + a **Row Locator** (pointer/PK). |
| **Count per Table** | **Strictly 1 per table** (data can only be sorted one way).| **Multiple per table** (typically up to hundreds). |
| **Lookup Cost** | Direct lookup: Traverse B-Tree to leaf node $\rightarrow$ Data retrieved. | Two-step: Traverse secondary B-tree $\rightarrow$ get PK $\rightarrow$ lookup in clustered index (**Key Lookup / Bookmark Lookup**). |
| **Default Creation**| Primary Key automatically creates clustered index (MySQL InnoDB / SQL Server). | `CREATE INDEX` or `UNIQUE` constraint creates non-clustered index. |

---

### 5.4 The Composite Index Left-Prefix Rule
When an index is created on multiple columns: `CREATE INDEX idx_emp_dept_sal ON employees (dept_id, salary, hire_date);`

The B+ tree is sorted strictly by `dept_id` first, then by `salary` within identical `dept_id`s, and then by `hire_date`.

#### Which queries can use this index?
- `WHERE dept_id = 10` $\rightarrow$ **YES** (Uses prefix).
- `WHERE dept_id = 10 AND salary > 80000` $\rightarrow$ **YES** (Uses both).
- `WHERE dept_id = 10 AND salary = 90000 AND hire_date = '2022-01-01'` $\rightarrow$ **YES** (Uses all 3).
- `WHERE salary = 90000` $\rightarrow$ **NO** (Violates left-prefix; cannot seek).
- `WHERE hire_date = '2022-01-01'` $\rightarrow$ **NO** (Violates left-prefix).
- `WHERE dept_id = 10 AND hire_date = '2022-01-01'` $\rightarrow$ **PARTIAL** (Uses `dept_id` to seek; filters `hire_date` in memory).

---

### 5.5 Covering Indexes (`INCLUDE` Columns)
A **Covering Index** contains all columns requested by a query in its leaf nodes. When an index covers a query, the engine satisfies the query entirely from the index without executing an expensive **Key Lookup** back to the clustered table!

```sql
-- Creating an index with INCLUDE columns (SQL Server / PostgreSQL)
CREATE INDEX idx_orders_customer_covering 
ON orders (customer_id, order_date) 
INCLUDE (total_amount, order_status);

-- This query is 100% "Covered" (Index-Only Scan)
SELECT customer_id, order_date, total_amount, order_status
FROM orders
WHERE customer_id = 101 AND order_date >= '2024-01-01';
```

---

# Part 6: Core Comparisons Guide

### Comparison 1: 3NF vs. BCNF
| Feature | Third Normal Form (3NF) | Boyce-Codd Normal Form (BCNF) |
| :--- | :--- | :--- |
| **Determinant Rule** | For $X \rightarrow Y$, $X$ is super key OR $Y$ is prime attribute. | For $X \rightarrow Y$, $X$ **must** be a super key. |
| **Strictness** | Less strict. Tolerates functional dependencies into prime attributes. | Stricter. |
| **Dependency Preservation** | Always preserves functional dependencies. | May occasionally lose functional dependencies during decomposition. |
| **Industry Practice** | Standard target for almost all OLTP database designs. | Applied in specialized scenarios with overlapping composite keys. |

### Comparison 2: OLTP vs. OLAP Schema Design
| Attribute | OLTP (Online Transaction Processing) | OLAP (Online Analytical Processing) |
| :--- | :--- | :--- |
| **Primary Goal** | Fast, reliable transaction processing (`INSERT`, `UPDATE`). | Fast analytical querying and aggregation over historical data. |
| **Data Model** | Highly Normalized (3NF / BCNF). | Denormalized (Star Schema, Snowflake Schema, One Big Table). |
| **Tables** | Many narrow tables with many foreign keys. | Few wide tables (Fact tables surrounded by Dimension tables). |
| **Redundancy** | Minimized to prevent update anomalies. | Intentionally introduced to eliminate expensive joins. |
| **Write/Read Ratio** | High volume of single-row writes and updates. | Bulk append loads; read-intensive aggregate queries. |

---

# Part 7: Structured Interview Question Bank

### Q1: What is the difference between a Surrogate Key and a Natural Key? When should you choose one over the other?
**Answer:**
A Natural Key is an attribute that exists in the real world and has business meaning (such as a National Identification Number, Vehicle VIN, or Email address). A Surrogate Key is a synthetic, database-generated identifier (such as an auto-incrementing integer or UUID) that has no external meaning.
- **Choose Surrogate Keys** when natural keys are long strings (costly for foreign key indexing), prone to changing business rules (e.g., users changing email addresses), or when privacy regulations prohibit storing sensitive identifiers in secondary index logs.
- **Choose Natural Keys** for reference lookup tables (e.g., ISO 2-letter country codes like `'US'`, `'GB'`) where the code is permanently standardized and eliminates an unnecessary join.

### Q2: Why does an excessive number of indexes degrade database performance?
**Answer:**
While indexes accelerate `SELECT` queries by providing fast B-tree traversal, they impose direct write amplification costs during `INSERT`, `UPDATE`, and `DELETE` operations. Every time a row is inserted or deleted, or an indexed column is modified, the database engine must synchronously update every associated secondary B-tree, manage page splits, rebalance trees, and write corresponding write-ahead log (WAL) entries. Furthermore, unused indexes consume valuable buffer pool memory, evicting frequently queried data pages.

### Q3: What is a "Key Lookup" (or Bookmark Lookup), and why is it detrimental to high-scale query performance?
**Answer:**
When a query uses a secondary (non-clustered) index to filter rows, the leaf node of that secondary index provides the clustered key (or physical tuple ID). If the `SELECT` clause requests additional columns that are not part of the secondary index, the engine must perform an extra point-lookup into the clustered index for each matching row. If the query matches thousands of rows, this results in thousands of random I/O operations, which can be slower than a full table sequential scan.

### Q4: Can a table in 3NF have insertion anomalies?
**Answer:**
Yes. A table in 3NF can still exhibit insertion anomalies if it has overlapping candidate keys and a functional dependency $X \rightarrow Y$ where $X$ is not a super key (which violates BCNF). In such cases, one cannot insert a fact about $X \rightarrow Y$ without supplying a value for the other candidate key attributes.

### Q5: Explain the difference between `CHAR` and `VARCHAR` from a storage and performance perspective.
**Answer:**
- `CHAR(n)` is fixed-length; it always reserves $n$ bytes on disk and right-pads shorter strings with spaces. It avoids row migration overhead during updates because the column size never expands. Ideal for fixed-length codes (UUIDs, MD5 hashes, country codes).
- `VARCHAR(n)` is variable-length; it stores only the actual characters plus a 1- or 2-byte length prefix. It conserves substantial disk and buffer cache space for variable-length text (names, descriptions), reducing overall I/O.

---

# Part 8: Complete Production DDL Database Design Exercise

### Enterprise E-Commerce Platform Architecture
**Entities Covered:** Customers, Categories, Products, Orders, Order_Items, Payments.

```sql
-- 1. Customers Dimension
CREATE TABLE customers (
    customer_id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone_number VARCHAR(20),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE NOT NULL
);

-- 2. Product Categories (Supports self-referencing hierarchy)
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    parent_category_id INT REFERENCES categories(category_id) ON DELETE SET NULL,
    category_name VARCHAR(100) NOT NULL,
    slug VARCHAR(120) NOT NULL UNIQUE
);

-- 3. Products Catalog
CREATE TABLE products (
    product_id BIGSERIAL PRIMARY KEY,
    category_id INT NOT NULL REFERENCES categories(category_id) ON DELETE RESTRICT,
    sku VARCHAR(64) NOT NULL UNIQUE,
    product_name VARCHAR(255) NOT NULL,
    unit_price DECIMAL(12, 2) NOT NULL CHECK (unit_price >= 0.00),
    stock_quantity INT NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    is_discontinued BOOLEAN DEFAULT FALSE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 4. Orders Header
CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(customer_id) ON DELETE RESTRICT,
    order_date TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    order_status VARCHAR(30) NOT NULL CHECK (order_status IN ('Pending', 'Processing', 'Shipped', 'Delivered', 'Cancelled', 'Refunded')),
    subtotal_amount DECIMAL(12, 2) NOT NULL DEFAULT 0.00 CHECK (subtotal_amount >= 0.00),
    tax_amount DECIMAL(12, 2) NOT NULL DEFAULT 0.00 CHECK (tax_amount >= 0.00),
    shipping_amount DECIMAL(12, 2) NOT NULL DEFAULT 0.00 CHECK (shipping_amount >= 0.00),
    total_amount DECIMAL(12, 2) NOT NULL CHECK (total_amount >= 0.00)
);

-- 5. Order Line Items (Composite Business Relationship)
CREATE TABLE order_items (
    order_item_id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(product_id) ON DELETE RESTRICT,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price_at_sale DECIMAL(12, 2) NOT NULL CHECK (unit_price_at_sale >= 0.00),
    total_line_amount DECIMAL(12, 2) GENERATED ALWAYS AS (quantity * unit_price_at_sale) STORED,
    CONSTRAINT uq_order_product UNIQUE (order_id, product_id)
);

-- 6. Payments
CREATE TABLE payments (
    payment_id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(order_id) ON DELETE RESTRICT,
    payment_method VARCHAR(50) NOT NULL CHECK (payment_method IN ('Credit Card', 'Debit Card', 'PayPal', 'Wire Transfer', 'Stripe')),
    payment_status VARCHAR(30) NOT NULL CHECK (payment_status IN ('Authorized', 'Captured', 'Failed', 'Refunded')),
    transaction_reference VARCHAR(100) NOT NULL UNIQUE,
    amount_paid DECIMAL(12, 2) NOT NULL CHECK (amount_paid > 0.00),
    payment_timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- INDEXING STRATEGY FOR HIGH CONCURRENCY
-- FK lookup indexes to prevent full scans during join resolution
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_payments_order_id ON payments(order_id);

-- Covering Index for Customer Order History Lookup
CREATE INDEX idx_orders_cust_date_covering 
ON orders (customer_id, order_date DESC) 
INCLUDE (order_status, total_amount);
```

---

# Part 9: End-to-End Enterprise Case Study: Legacy Schema Normalization

### Scenario & Business Problem
A fast-growing fintech startup logged all transactions and merchant settlements in a single denormalized table `raw_settlements` originating from flat CSV exports.

**The Unnormalized Relation:**
`raw_settlements (settlement_id, merchant_name, merchant_tax_id, merchant_city, merchant_bank_acc, batch_id, batch_date, txn_id, txn_amount, txn_fee, txn_status)`

**Symptoms Experienced:**
1. **Update Anomalies:** When a merchant changed their bank account, past batch records showed differing bank details.
2. **Deletion Anomalies:** Deleting a batch to clear failed transactions wiped out the merchant's business profile.
3. **Storage Bloat:** Merchant address and tax IDs were repeated 1.4 million times across daily transactions.

### Decomposition into 3NF Relational Architecture
- **Identified Functional Dependencies:**
  - `settlement_id -> all attributes`
  - `merchant_tax_id -> merchant_name, merchant_city, merchant_bank_acc` (Independent entity)
  - `batch_id -> batch_date` (Independent entity)
  - `txn_id -> batch_id, merchant_tax_id, txn_amount, txn_fee, txn_status`

#### Normalized Target Schemas:
1. `merchants (merchant_id, tax_id [UQ], name, city, bank_account)`
2. `settlement_batches (batch_id, batch_date, created_at)`
3. `settlement_transactions (txn_id, batch_id [FK], merchant_id [FK], amount, fee, status)`

**Result:** Eliminated 78% storage redundancy, guaranteed single-point merchant profile updates, and eradicated deletion anomalies completely.

---

# Part 10: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   DBMS DESIGN & NORMALIZATION CHEAT SHEET              │
├────────────────────────────────────────────────────────────────────────┤
│ 1. PRIMARY KEY : Unique + NOT NULL. Determines physical cluster order. │
│ 2. FOREIGN KEY : Enforces referential integrity to parent PK/Unique.   │
│ 3. CASCADE     : Automatically propagates parent DELETE/UPDATE to child│
├────────────────────────────────────────────────────────────────────────┤
│                          NORMAL FORMS SUMMARY                          │
│  - 1NF : Atomic attributes. No arrays/lists. PK defined.               │
│  - 2NF : 1NF + No Partial Dependencies (non-key relies on whole PK).   │
│  - 3NF : 2NF + No Transitive Dependencies (non-key -> non-key).        │
│  - BCNF: Strict 3NF: For every X -> Y, X MUST be a Super Key.         │
├────────────────────────────────────────────────────────────────────────┤
│                          INDEXING ESSENTIALS                           │
│  - CLUSTERED INDEX     : 1 per table. Holds actual data pages.         │
│  - NON-CLUSTERED INDEX : Secondary B-Tree holding keys + row pointer.  │
│  - LEFT-PREFIX RULE    : Composite index (A, B, C) works for A, (A,B), │
│                          (A,B,C), but NOT for B or C alone.            │
│  - COVERING INDEX      : Index contains all selected columns (INCLUDE).│
│                          Eliminates expensive Key Lookups.             │
├────────────────────────────────────────────────────────────────────────┤
│                     VIEWS vs MATERIALIZED VIEWS                        │
│  - VIEW              : Virtual. Zero storage. Runs underlying query.   │
│  - MATERIALIZED VIEW : Physical snapshot on disk. Super fast reads.    │
│                        Must be refreshed periodically.                 │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 11: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (25 Marks — 5 marks each)

**Q1:** A table has a composite primary key `(student_id, course_id)`. The column `course_fee` depends solely on `course_id`. Which normal form does this table violate?  
(a) 1NF  
(b) 2NF  
(c) 3NF  
(d) BCNF  

**Q2:** What happens when an `ON DELETE SET NULL` constraint is triggered, but the foreign key column is defined as `NOT NULL`?  
(a) The engine silently changes the column to allow NULLs.  
(b) The deletion succeeds and sets the column to 0.  
(c) The delete operation fails and throws an integrity constraint violation error.  
(d) The parent record is deleted and child rows remain unchanged.  

**Q3:** How many Clustered Indexes can exist on a single database table in SQL Server or MySQL InnoDB?  
(a) 0  
(b) Strictly 1  
(c) Up to 16  
(d) Unlimited  

**Q4:** In a B+ Tree index of depth 3, approximately how many disk page reads are required to locate an individual record by its primary key?  
(a) 1  
(b) 3 to 4  
(c) Hundreds of sequential reads  
(d) Proportional to $N$  

**Q5:** Given a composite index on `(country, state, city)`, which of the following queries CANNOT leverage the index via an index seek?  
(a) `WHERE country = 'USA'`  
(b) `WHERE country = 'USA' AND state = 'CA'`  
(c) `WHERE state = 'CA' AND city = 'Los Angeles'`  
(d) `WHERE country = 'USA' AND state = 'CA' AND city = 'Los Angeles'`  

---

### Section B: Dependency & Normalization Analysis (25 Marks)

**Q6 (10 Marks):** Consider the relation $R(A, B, C, D, E)$ with functional dependencies:
- $A \rightarrow B, C$
- $C, D \rightarrow E$
- $B \rightarrow D$
1. Determine the candidate key(s) of $R$.
2. State the highest normal form of $R$ with formal justification.

**Q7 (15 Marks):** Decompose the relation $R(A, B, C, D)$ with functional dependencies $A \rightarrow B, C$ and $B \rightarrow D$ into 3NF relations. State the primary keys and foreign keys of the resulting relations.

---

### Section C: Database Design Implementation (30 Marks)

**Q8 (30 Marks):** Design the relational schema for an **Airline Flight Booking System**.
Write production-grade PostgreSQL DDL statements for:
1. `airports (airport_code [3-char PK], name, city, country)`
2. `flights (flight_id, flight_number, departure_airport [FK], arrival_airport [FK], scheduled_departure, scheduled_arrival)`
3. `passengers (passenger_id, passport_number [UQ], first_name, last_name, email)`
4. `bookings (booking_id, flight_id [FK], passenger_id [FK], seat_number, price, booking_status)`
*Include all necessary primary keys, foreign keys with appropriate cascade rules, and at least one composite unique constraint to prevent double-booking the same seat on the same flight.*

---

### Section D: Architectural & Performance Scenario (20 Marks)

**Q9 (20 Marks):** An analytics team queries a 40-million row `transactions` table using:
```sql
SELECT customer_id, txn_date, amount
FROM transactions
WHERE customer_id = 89412
ORDER BY txn_date DESC;
```
1. If the table has only a clustered index on `txn_id`, describe the execution plan and performance bottleneck. (7 marks)
2. Design the optimal non-clustered composite index to make this query execute in under 5 milliseconds. Explain how it eliminates the sort and key lookup operations. (13 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** 2NF. `course_fee` depends on part of the composite primary key (`course_id`), which is a partial dependency.
- **Q2:** **(c)** Integrity violation error. You cannot set a `NOT NULL` column to `NULL`.
- **Q3:** **(b)** Strictly 1. Physical data rows can only be ordered on disk in one sequence.
- **Q4:** **(b)** 3 to 4 page I/O reads (1 read per B-Tree node level from root to leaf).
- **Q5:** **(c)** `WHERE state = 'CA' AND city = 'Los Angeles'` violates the left-prefix rule because `country` is omitted.

### Section B Solutions
**Q6 Solution:**
1. Candidate Key:
   - Compute closure of $A$: $A^+ = \{A, B, C, D, E\}$ (since $A \rightarrow B, C$, $B \rightarrow D$, and $C, D \rightarrow E$).
   - Therefore, **$A$ is the sole candidate key**.
2. Highest Normal Form:
   - Is it in 1NF? Yes.
   - Is it in 2NF? Yes, because candidate key $A$ is a single attribute (no partial dependencies possible).
   - Is it in 3NF? Check $B \rightarrow D$: $B$ is not a super key, and $D$ is not a prime attribute.
   - **Highest Normal Form is 2NF.** (Violates 3NF due to transitive dependency $A \rightarrow B \rightarrow D$).

**Q7 Solution:**
- Canonical FDs: $A \rightarrow B, C$ and $B \rightarrow D$.
- 3NF Decomposition:
  1. $R_1(A, B, C)$ with Primary Key $A$.
  2. $R_2(B, D)$ with Primary Key $B$, and $B$ in $R_1$ acting as Foreign Key referencing $R_2$.

### Section C Solutions (DDL Implementation)
```sql
CREATE TABLE airports (
    airport_code CHAR(3) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    city VARCHAR(100) NOT NULL,
    country VARCHAR(100) NOT NULL
);

CREATE TABLE flights (
    flight_id BIGSERIAL PRIMARY KEY,
    flight_number VARCHAR(10) NOT NULL,
    departure_airport CHAR(3) NOT NULL REFERENCES airports(airport_code) ON DELETE RESTRICT,
    arrival_airport CHAR(3) NOT NULL REFERENCES airports(airport_code) ON DELETE RESTRICT,
    scheduled_departure TIMESTAMP WITH TIME ZONE NOT NULL,
    scheduled_arrival TIMESTAMP WITH TIME ZONE NOT NULL,
    CHECK (scheduled_arrival > scheduled_departure)
);

CREATE TABLE passengers (
    passenger_id BIGSERIAL PRIMARY KEY,
    passport_number VARCHAR(20) NOT NULL UNIQUE,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE bookings (
    booking_id BIGSERIAL PRIMARY KEY,
    flight_id BIGINT NOT NULL REFERENCES flights(flight_id) ON DELETE CASCADE,
    passenger_id BIGINT NOT NULL REFERENCES passengers(passenger_id) ON DELETE RESTRICT,
    seat_number VARCHAR(5) NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0.00),
    booking_status VARCHAR(20) NOT NULL DEFAULT 'Confirmed',
    CONSTRAINT uq_flight_seat UNIQUE (flight_id, seat_number) -- Prevents double booking
);
```

### Section D Solutions
**Q9 Solution:**
1. Without an index on `customer_id`, the engine must perform a **Full Table Scan** reading all 40 million rows ($O(N)$ I/O cost). Once matching rows are located, an in-memory or TempDB **Sort operator** ($O(K \log K)$) must physically order the rows by `txn_date DESC`.
2. **Optimal Index:**
   ```sql
   CREATE INDEX idx_txns_cust_date ON transactions (customer_id, txn_date DESC) INCLUDE (amount);
   ```
   - **Mechanism:**
     - The index seeks directly to `customer_id = 89412` ($O(\log N)$).
     - Because the secondary index leaf nodes are already sorted physically by `txn_date DESC`, rows are streamed in exact requested order, **completely eliminating the Sort operator**.
     - Because `amount` is stored in the leaf nodes via `INCLUDE`, the index **covers** the entire query, **completely eliminating Key Lookups** to the clustered table.

---

