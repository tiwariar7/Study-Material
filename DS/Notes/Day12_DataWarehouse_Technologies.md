# Day 12: Modern Data Warehouses, Orchestration & Transformation
## Snowflake Architecture, T-SQL vs. PL/SQL, Apache Airflow, dbt & The Medallion Architecture

---

# Part 1: Snowflake Architecture & Cloud Data Warehousing

```
┌────────────────────────────────────────────────────────────────────────┐
│                        SNOWFLAKE ARCHITECTURE                          │
├────────────────────────────────────────────────────────────────────────┤
│ 1. CLOUD SERVICES LAYER                                                │
│    - Query Optimization, Authentication, Metadata, Access Control     │
├────────────────────────────────────────────────────────────────────────┤
│ 2. QUERY PROCESSING LAYER (Virtual Warehouses)                         │
│    - Decoupled Independent Compute Clusters (X-Small to 4X-Large)     │
│    - Dedicated Compute per Workload (ETL vs BI vs Data Science)        │
├────────────────────────────────────────────────────────────────────────┤
│ 3. DATABASE STORAGE LAYER (Centralized Hybrid Columnar)               │
│    - Immutable Micro-partitions (50MB–500MB uncompressed)              │
│    - Cloud Blob Storage (AWS S3 / Azure Blob / GCS)                    │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 The Three Decoupled Architectural Layers

#### Layer 1: Centralized Database Storage
- Snowflake stores all table data in an optimized, encrypted, columnar format split into immutable **Micro-partitions** (typically 50MB to 500MB of uncompressed data).
- Storage resides in cloud object stores (S3, ADLS Gen2, GCS). Users pay only for raw compressed cloud storage volume ($/TB/month).

#### Layer 2: Virtual Compute Warehouses (Query Processing)
- Compute is **completely decoupled** from storage. A "Virtual Warehouse" is an independent cluster of CPU/RAM nodes (powered by EC2 or Azure VMs).
- **Zero Resource Contention:** An ETL warehouse running a heavy batch load does not slow down an executive running a Power BI dashboard on a separate BI warehouse!
- **Auto-Suspend & Auto-Resume:** Warehouses automatically spin down after $N$ seconds of inactivity, billing compute strictly per-second.

#### Layer 3: Cloud Services (The Brain)
- A multi-tenant, globally managed layer that handles authentication, access control (RBAC), metadata tracking, ACID transaction coordination, and query compilation/optimization.

---

### 1.2 Revolutionary Snowflake Capabilities

#### Micro-Partitioning & Pruning
Tables are automatically partitioned based on natural insertion order. For every micro-partition, Snowflake automatically tracks **column-level min/max metadata**. During query execution, the optimizer uses this metadata to completely skip (prune) non-matching micro-partitions without reading them from disk!

#### Zero-Copy Cloning
Creates an instantaneous metadata clone of a database, schema, or table without copying underlying storage bytes:
```sql
CREATE DATABASE analytics_dev CLONE analytics_prod;
```
- Both databases initially point to the exact same immutable micro-partitions ($0 extra storage cost).
- Writes to either database write new micro-partitions independently (Copy-on-Write).

#### Time Travel & Fail-safe
- **Time Travel (Up to 90 Days):** Allows querying or restoring data as it existed at any historical timestamp or query ID:
  ```sql
  SELECT * FROM orders AT(OFFSET => -60*15); -- 15 minutes ago
  UNDROP TABLE accidental_deleted_table;
  ```
- **Fail-safe (7 Days):** Disaster recovery storage accessible only by Snowflake support.

---

# Part 2: Enterprise Database Engines: T-SQL vs. PL/SQL

Both Microsoft SQL Server and Oracle are battle-tested relational engines, but their procedural extensions differ:

| Feature | T-SQL (Transact-SQL - Microsoft SQL Server) | PL/SQL (Procedural Language - Oracle) |
| :--- | :--- | :--- |
| **Block Structure** | Implicit or explicit `BEGIN ... END`. | Formal block: `DECLARE ... BEGIN ... EXCEPTION ... END;`. |
| **Variable Declaration**| `DECLARE @var INT = 10;` | `var_name NUMBER := 10;` (Inside DECLARE block). |
| **Error Handling** | Structured `BEGIN TRY ... BEGIN CATCH ... END CATCH`. | `EXCEPTION WHEN NO_DATA_FOUND THEN ...`. |
| **Cursors** | Declared, opened, fetched, closed, deallocated. | Native cursor FOR loops (`FOR r IN (SELECT ...) LOOP`). |
| **Identity / Autoincrement**| `IDENTITY(1,1)` column property. | `SEQUENCE` objects (e.g., `seq.NEXTVAL`) or `GENERATED AS IDENTITY`. |

```sql
-- T-SQL Stored Procedure Example (SQL Server)
CREATE PROCEDURE dbo.GetEmployeeBonus
    @EmpId INT,
    @BonusPct DECIMAL(4,2)
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        UPDATE dbo.Employees 
        SET Bonus = Salary * @BonusPct 
        WHERE EmployeeID = @EmpId;
    END TRY
    BEGIN CATCH
        SELECT ERROR_NUMBER() AS ErrorNum, ERROR_MESSAGE() AS ErrorMsg;
    END CATCH
END;

-- PL/SQL Stored Procedure Example (Oracle)
CREATE OR REPLACE PROCEDURE calculate_bonus(
    p_emp_id IN NUMBER,
    p_bonus_pct IN NUMBER
) IS
BEGIN
    UPDATE employees 
    SET bonus = salary * p_bonus_pct 
    WHERE employee_id = p_emp_id;
    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(-20001, 'Bonus calculation failed: ' || SQLERRM);
END calculate_bonus;
```

---

# Part 3: Workflow Orchestration: Apache Airflow Fundamentals

Apache Airflow orchestrates complex computational workflows as **Directed Acyclic Graphs (DAGs)** authored entirely as Python code.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        APACHE AIRFLOW DAG WORKFLOW                     │
│                                                                        │
│   [extract_api] ──> [validate_schema] ──> [load_to_snowflake]          │
│                                                   │                    │
│                                                   ▼                    │
│                                            [run_dbt_models]            │
│                                                   │                    │
│                                                   ▼                    │
│                                            [send_slack_alert]          │
└────────────────────────────────────────────────────────────────────────┘
```

### 3.1 Core Airflow Building Blocks
- **DAG (Directed Acyclic Graph):** The complete collection of tasks with directional dependencies and no circular loops.
- **Operator:** The template for a single unit of work:
  - `BashOperator`: Executes bash scripts.
  - `PythonOperator`: Calls an arbitrary Python function.
  - `SnowflakeOperator` / `SQLExecuteQueryOperator`: Runs native SQL commands.
- **Task Instance:** A specific task assigned to an execution date/run (`dag_run`).
- **Dependencies (`>>` and `<<`):** Bitshift operators defining the execution sequence (`task_a >> task_b`).

### 3.2 Production Airflow DAG Implementation
```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.snowflake.operators.snowflake import SQLExecuteQueryOperator

default_args = {
    "owner": "data_engineering",
    "depends_on_past": False,
    "email_on_failure": True,
    "email": ["data_alerts@company.com"],
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
}

with DAG(
    dag_id="daily_sales_pipeline",
    default_args=default_args,
    description="Extracts sales data, loads to Snowflake, and executes dbt models",
    schedule="0 2 * * *",  # Daily at 02:00 UTC
    start_date=datetime(2024, 1, 1),
    catchup=False,          # Prevents backfilling every day since start_date!
    max_active_runs=1,
) as dag:

    def extract_external_sales(**context):
        print("Fetching API data and staging to cloud bucket...")

    task_extract = PythonOperator(
        task_id="extract_api_sales",
        python_callable=extract_external_sales,
    )

    task_copy_snowflake = SQLExecuteQueryOperator(
        task_id="copy_into_bronze_snowflake",
        conn_id="snowflake_default",
        sql="""
            COPY INTO bronze_db.raw.sales 
            FROM @bronze_db.stages.s3_sales_stage 
            FILE_FORMAT = (TYPE = 'PARQUET');
        """,
    )

    task_extract >> task_copy_snowflake
```

---

# Part 4: Modern Transformation as Code: dbt (Data Build Tool)

dbt enables analytics engineers to transform data directly inside cloud warehouses using standard `SELECT` statements, treating SQL transformations with software engineering rigor (version control, automated testing, documentation, CI/CD).

### 4.1 How dbt Works
In dbt, an engineer writes pure `SELECT` queries in `.sql` files. dbt automatically compiles these files into DDL/DML statements (`CREATE TABLE AS SELECT` or `CREATE VIEW AS SELECT`), managing table creation, dependency resolution, and incremental merges.

```sql
-- models/marts/fct_daily_sales.sql
{{ config(
    materialized='incremental',
    unique_key='sale_id'
) }}

WITH source_data AS (
    -- The {{ ref() }} macro builds the automated lineage DAG!
    SELECT * FROM {{ ref('stg_sales') }}
    {% if is_incremental() %}
        -- Only query records newer than the current table's max timestamp
        WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
    {% endif %}
)
SELECT 
    sale_id,
    customer_id,
    sale_amount,
    updated_at
FROM source_data;
```

---

### 4.2 Materialization Strategies in dbt
1. **`view`:** Fast to build; zero warehouse storage; evaluated on-the-fly when queried.
2. **`table`:** Physical table created via `CTAS`; fast query response; incurs compute time on build.
3. **`incremental`:** Modifies only newly added or modified rows since the last dbt execution using `MERGE`. Essential for multi-million row fact tables.
4. **`ephemeral`:** Interpolated as an inline Common Table Expression (CTE) into downstream models; zero database footprint.

---

### 4.3 Automated Data Testing in dbt
dbt includes four out-of-the-box generic tests defined in `schema.yml`:
```yaml
version: 2
models:
  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: order_status
        tests:
          - accepted_values:
              values: ['Pending', 'Completed', 'Cancelled', 'Refunded']
      - name: customer_id
        tests:
          - relationships:
              to: ref('stg_customers')
              field: customer_id
```

---

# Part 5: Required Deliverable: From Raw Ingestion to Gold Analytics (The Medallion Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   THE MEDALLION ARCHITECTURE PIPELINE                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [External APIs / OLTP DBs]
             │
             ▼ (Raw Extract via Airflow / ADF)
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 1. BRONZE LAYER (Raw Zone)                                                                            │
 │    - Schema: bronze.raw_orders                                                                        │
 │    - Characteristics: Append-only, untransformed raw JSON/Parquet directly from sources               │
 │    - Retains original source schema, audit metadata (_ingested_at, _source_file)                      │
 │    - Zero schema enforcement; full historical record for disaster recovery & reprocessing             │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
             │
             ▼ (dbt Silver Models: Cleaning, Casting, Deduplication, Normalization)
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 2. SILVER LAYER (Conformed / Cleaned Zone)                                                            │
 │    - Schema: silver.stg_orders, silver.dim_customers                                                  │
 │    - Characteristics: Cleaned, type-casted, deduplicated enterprise tables                            │
 │    - Deduplication: Uses ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC)           │
 │    - Data Quality: Null checks enforced; invalid values quarantined to DLQ                            │
 │    - Third Normal Form (3NF) relational structure                                                     │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
             │
             ▼ (dbt Gold Models: Star Schema Dimensional Modeling & Aggregations)
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 3. GOLD LAYER (Curated / Business Marts Zone)                                                         │
 │    - Schema: gold.fct_daily_sales, gold.dim_customer_360                                              │
 │    - Characteristics: Highly optimized Star Schema (Fact and Dimension tables) or One Big Table (OBT)│
 │    - Pre-aggregated business KPIs (Customer LTV, Monthly Recurring Revenue, Churn Risk)               │
 │    - Power BI / Tableau dashboards and ML feature stores connect directly here                        │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 6: Structured Interview Question Bank

### Q1: How does Snowflake achieve zero-copy cloning without duplicating storage data?
**Answer:**
Snowflake's storage layer consists of immutable micro-partitions stored in cloud object storage, with the Cloud Services metadata layer maintaining pointers to these files. When `CLONE` is invoked, Snowflake simply duplicates the metadata catalog pointers to the existing micro-partitions without copying any underlying data blocks. If a user subsequently modifies or updates data in the cloned table, Snowflake writes brand-new micro-partitions (Copy-on-Write) and updates the clone's metadata pointers, ensuring that unchanged data is shared while modified data remains isolated.

### Q2: What is the `{{ ref() }}` macro in dbt, and why is it superior to hardcoding table names?
**Answer:**
The `{{ ref('model_name') }}` macro accomplishes two essential engineering functions:
1. **Automated Lineage & Dependency Graph:** It tells dbt that the current model depends on `model_name`. dbt automatically computes the Directed Acyclic Graph (DAG) and executes models in the correct topological order.
2. **Environment Isolation:** It interpolates the correct database and schema based on the active target environment (e.g., querying `dev_schema.stg_orders` in local development, but compiling to `prod_schema.stg_orders` in production CI/CD).

### Q3: What is `catchup=False` in an Apache Airflow DAG, and what catastrophic issue does it prevent?
**Answer:**
By default, Airflow's `catchup` parameter is `True`. If a DAG is defined with a `start_date` of two months ago and enabled today, Airflow will immediately schedule and execute separate `dag_run` instances for every single missed interval between the start date and the current date (e.g., 60 back-to-back runs). This can overload source APIs, crash databases, and incur massive cloud computing costs. Setting `catchup=False` instructs the scheduler to execute strictly the most recent interval.

---

# Part 7: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│               DATA WAREHOUSE & ORCHESTRATION CHEAT SHEET               │
├────────────────────────────────────────────────────────────────────────┤
│ 1. SNOWFLAKE:                                                          │
│    - Decoupled 3 Tiers: Storage (Micro-partitions) | Compute (Virtual  │
│      Warehouses) | Cloud Services (Metadata & Optimizer).              │
│    - Zero-Copy Cloning: Duplicates metadata pointers; 0 storage cost.  │
│    - Time Travel: Query historical state up to 90 days.                │
├────────────────────────────────────────────────────────────────────────┤
│ 2. APACHE AIRFLOW:                                                     │
│    - DAG: Directed Acyclic Graph (Python-authored workflow).           │
│    - Operators: PythonOperator, BashOperator, SQLExecuteQueryOperator. │
│    - Dependencies: task_a >> task_b.                                   │
│    - Always set catchup=False unless historical backfill is intended!  │
├────────────────────────────────────────────────────────────────────────┤
│ 3. DBT (DATA BUILD TOOL):                                              │
│    - In-warehouse SQL transformations (ELT).                           │
│    - {{ ref('model') }}: Automatic DAG lineage and environment routing.│
│    - Materializations: view, table, incremental, ephemeral.           │
│    - Testing: unique, not_null, accepted_values, relationships.        │
├────────────────────────────────────────────────────────────────────────┤
│ 4. MEDALLION ARCHITECTURE:                                             │
│    - Bronze (Raw)   : Append-only full historical fidelity.            │
│    - Silver (Clean) : Deduplicated, 3NF cleaned conformed tables.      │
│    - Gold (Marts)   : Star schema, business KPIs, reporting-ready.     │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 8: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (20 Marks — 4 marks each)

**Q1:** In Snowflake's three-tier architecture, where are query compilation, metadata management, and transaction locks handled?  
(a) Database Storage Layer  
(b) Virtual Warehouse Compute Layer  
(c) Cloud Services Layer  
(d) Local SSD Cache  

**Q2:** What happens when an Airflow DAG contains a circular dependency (`task_1 >> task_2 >> task_1`)?  
(a) Airflow executes the tasks in an infinite loop.  
(b) The Airflow scheduler detects a DAG cycle and raises an `AirflowDagCycleException`.  
(c) The tasks are automatically executed in parallel.  
(d) Airflow converts the cycle into a linear queue.  

**Q3:** Which dbt materialization creates a temporary Common Table Expression (CTE) that is interpolated into downstream models rather than existing as a database object?  
(a) `view`  
(b) `table`  
(c) `incremental`  
(d) `ephemeral`  

**Q4:** In Snowflake, what is the default maximum duration for Time Travel on an Enterprise Edition account?  
(a) 1 day  
(b) 7 days  
(c) 90 days  
(d) 365 days  

**Q5:** In a Medallion Architecture, which layer is characterized by Star Schema dimensional models (Fact and Dimension tables) optimized for BI dashboards?  
(a) Bronze Layer  
(b) Silver Layer  
(c) Gold Layer  
(d) Landing Zone  

---

### Section B: Architecture & Execution Tracing (25 Marks)

**Q6 (10 Marks):** Explain how Snowflake's micro-partition min/max metadata enables "Query Pruning". How does sorting/clustering data impact pruning efficiency?

**Q7 (15 Marks):** Consider the Airflow DAG schedule: `schedule="0 0 * * *"` with `start_date=2024-01-01`. When does the DAG run for execution date `2024-01-01` actually trigger, and why does Airflow use this scheduling paradigm?

---

### Section C: Implementation Challenges (35 Marks)

**Q8 (20 Marks):** Write a production dbt model `fct_orders.sql` configured as an **incremental** table using `order_id` as the unique key. Ensure it references the upstream model `stg_orders` using the `ref` macro, and only queries rows where `updated_at` is greater than the maximum `updated_at` currently in the target table.

**Q9 (15 Marks):** Write an Airflow Python DAG snippet defining three tasks: `extract_task`, `transform_task`, and `load_task`. Configure the DAG so that `transform_task` runs only if `extract_task` succeeds, and `load_task` runs only if `transform_task` succeeds, with an automated 2-retry policy.

---

### Section D: Enterprise Failure Scenario (20 Marks)

**Q10 (20 Marks):** A daily incremental dbt model fails with an error:
`Duplicate key violation: duplicate key value found in column 'order_id'`.
1. Explain how upstream late-arriving updates or duplicated raw records caused this failure. (10 marks)
2. Provide the SQL deduplication pattern using `ROW_NUMBER()` in the staging layer to permanently eliminate this vulnerability. (10 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(c)** Cloud Services Layer.
- **Q2:** **(b)** Airflow detects the cycle and refuses to parse the DAG (`AirflowDagCycleException`).
- **Q3:** **(d)** `ephemeral`.
- **Q4:** **(c)** 90 days.
- **Q5:** **(c)** Gold Layer.

### Section B Solutions
**Q6 Solution:**
- Every micro-partition automatically records the minimum and maximum values of every column in the Cloud Services metadata catalog.
- When a query executes with `WHERE order_date BETWEEN '2024-01-01' AND '2024-01-05'`, the optimizer checks the min/max values of all micro-partitions. Partitions where `min > '2024-01-05'` or `max < '2024-01-01'` are pruned entirely without reading any data from storage.
- Clustering keys ensure that rows with correlated values are physically packed into the same micro-partitions, minimizing the min/max range overlaps and maximizing the number of skipped partitions.

**Q7 Solution:**
- In Airflow, a data interval run for date $D$ executes **at the end of the interval** ($D + \text{duration}$). Therefore, the run for `2024-01-01` triggers on **2024-01-02 at 00:00:00 UTC**.
- **Reasoning:** A daily data pipeline cannot process all records belonging to January 1st until January 1st has fully completed!

### Section C Solutions
**Q8 Solution:**
```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

WITH source_data AS (
    SELECT * FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
        WHERE updated_at > (SELECT COALESCE(MAX(updated_at), '1970-01-01') FROM {{ this }})
    {% endif %}
)
SELECT 
    order_id,
    customer_id,
    order_status,
    total_amount,
    updated_at
FROM source_data;
```

**Q9 Solution:**
```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.empty import EmptyOperator

default_args = {
    "retries": 2,
    "retry_delay": timedelta(minutes=3),
}

with DAG(
    dag_id="linear_pipeline",
    default_args=default_args,
    schedule="0 0 * * *",
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:
    extract = EmptyOperator(task_id="extract_task")
    transform = EmptyOperator(task_id="transform_task")
    load = EmptyOperator(task_id="load_task")

    extract >> transform >> load
```

### Section D Solutions
**Q10 Solution:**
1. **Root Cause:** If the upstream source system produces two records with the same `order_id` in the same incremental window (e.g., an order created at 10:00 and updated at 10:05), the staging query selects both records. When dbt executes the `MERGE INTO`, it attempts to join on `order_id` and encounters multiple matching source rows, violating relational merge constraints.
2. **Deduplication Pattern:**
   ```sql
   WITH RankedSource AS (
       SELECT 
           order_id,
           customer_id,
           total_amount,
           updated_at,
           ROW_NUMBER() OVER (
               PARTITION BY order_id 
               ORDER BY updated_at DESC, _ingested_at DESC
           ) AS dedupe_rank
       FROM raw.orders
   )
   SELECT 
       order_id,
       customer_id,
       total_amount,
       updated_at
   FROM RankedSource
   WHERE dedupe_rank = 1;
   ```
