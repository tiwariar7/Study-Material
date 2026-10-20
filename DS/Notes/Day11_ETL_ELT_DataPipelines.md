# Day 11: ETL/ELT Architectures & Production Data Pipelines
## Batch vs. Streaming, Idempotency, Incremental Loading, Observability & Azure Data Factory (ADF)

---

# Part 1: The Architectural Paradigm Shift: ETL vs. ELT

```
TRADITIONAL ETL (Extract -> Transform -> Load)
[ Source Systems ] ──> [ External ETL Server ] ──> [ Data Warehouse ]
                          (Informatica / SSIS)        (Pre-transformed tables)
                       * Transform happens BEFORE load *
                       * Heavy CPU load on staging server *

MODERN CLOUD ELT (Extract -> Load -> Transform)
[ Source Systems ] ──> [ Cloud Raw Storage ] ──> [ Cloud Data Warehouse ]
                       (S3 / ADLS Gen2 Bronze)    (Snowflake / BigQuery / Fabric)
                                                  * Raw data loaded first *
                                                  * SQL/dbt transforms inside warehouse *
```

### 1.1 Architectural Comparison Matrix

| Dimension | Traditional ETL | Modern Cloud ELT |
| :--- | :--- | :--- |
| **Transform Location** | Dedicated intermediary server/memory buffer. | Inside the target distributed cloud warehouse. |
| **Data Fidelity** | Raw data is discarded; only transformed aggregated data lands. | **100% Raw data is preserved permanently** (Bronze/Data Lake). |
| **Scalability** | Bottlenecked by the hardware limits of the ETL engine. | Elastic: Leverages decoupled cloud compute (Snowflake, BigQuery). |
| **Schema Flexibility** | Schema-on-Write: Changes require modifying ETL pipeline code. | Schema-on-Read: Raw data lands as JSON/Parquet; transformed on demand. |
| **Compute Cost** | High specialized licensing (Informatica, Ab Initio). | Low storage cost; pay-per-second warehouse compute. |
| **Ideal Workload** | On-premise systems with strict compliance prohibiting cloud raw storage. | Modern enterprise analytics, BI, and machine learning platforms. |

---

# Part 2: Processing Paradigms: Batch vs. Streaming

### 2.1 Batch Processing
- **Mechanics:** Processes bounded chunks of historical data at scheduled periodic intervals (e.g., hourly, daily at 02:00 UTC).
- **Latency:** Minutes to hours.
- **Throughput:** Maximum efficiency. Optimizers can sort, compress, and write millions of rows in bulk columnar blocks.
- **Technologies:** Apache Spark, Azure Data Factory, dbt, SQL Server Integration Services (SSIS).

### 2.2 Streaming Processing
- **Mechanics:** Processes unbounded streams of events one-by-one or in micro-batches as they occur.
- **Latency:** Sub-second to seconds.
- **Throughput:** Lower per-event efficiency; high real-time responsiveness.
- **Technologies:** Apache Kafka, Azure Event Hubs, Apache Flink, Spark Structured Streaming.

---

### 2.3 System Architectures: Lambda vs. Kappa

#### Lambda Architecture (Dual-Path)
Combines a **Batch Layer** (for accurate, comprehensive historical data processing) with a **Speed Layer** (streaming low-latency views for recent events), unified by a **Serving Layer**.
- *Weakness:* Massive code duplication (must maintain logic twice: once in Spark/SQL for batch, once in Flink/Kafka for streaming).

#### Kappa Architecture (Single Streaming Path)
Replaces the dual paths with a **single stream-processing engine**. All data is treated as an immutable log stream. Historical reprocessing is achieved by rewinding the stream offset from the beginning of the log.

---

# Part 3: Critical Pipeline Engineering Principles

### 3.1 Idempotency & Determinism
A pipeline is **idempotent** if running it multiple times with the same input data produces the exact same final state without creating duplicate records or side effects.

#### How to Guarantee Idempotency:
1. **Atomic Partition Overwrites:**
   ```sql
   -- Idempotent daily batch load: Overwrite partition instead of blind append
   INSERT OVERWRITE TABLE daily_sales PARTITION (sale_date = '2024-03-15')
   SELECT * FROM staging_sales WHERE sale_date = '2024-03-15';
   ```
2. **Merge / Upsert on Business Keys:**
   ```sql
   MERGE INTO dim_customer target
   USING staging_customer source
   ON target.customer_id = source.customer_id
   WHEN MATCHED THEN UPDATE SET target.email = source.email, target.updated_at = source.updated_at
   WHEN NOT MATCHED THEN INSERT (customer_id, email, updated_at) VALUES (source.customer_id, source.email, source.updated_at);
   ```

---

### 3.2 Incremental Loading Strategies

```
Method 1: High-Watermark (Timestamp Polling)
Source Table: [WHERE updated_at > :last_pipeline_run_timestamp]
* Requires reliable updated_at column on source *
* Misses hard DELETE operations *

Method 2: Change Data Capture (CDC)
Source Database WAL / Binlog ──> Debezium / Kafka Connect ──> Target Warehouse
* Zero source query overhead *
* Captures INSERTS, UPDATES, and DELETES perfectly *
```

---

### 3.3 Handling Schema Drift & The Dead Letter Queue (DLQ)
When upstream sources add, drop, or alter column data types unexpectedly, production pipelines must not crash silently.
- **Quarantine Pattern:** Malformed rows that fail schema validation rules (e.g., negative prices, unparseable dates) are routed to a **Dead Letter Queue (DLQ)** or an `errors_staging` table for engineer inspection, allowing valid rows to proceed to the warehouse uninterrupted.

---

# Part 4: Azure Data Factory (ADF) Architecture

Azure Data Factory is Microsoft's cloud-native, serverless data integration and workflow orchestration service.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        AZURE DATA FACTORY (ADF)                        │
│                                                                        │
│  [ Trigger ] ──> [ Pipeline (Workflow DAG) ]                           │
│                     │                                                  │
│                     ├── [ Lookup Activity ]                            │
│                     ├── [ Copy Data Activity ] ──> [ Integration RT ]  │
│                     └── [ Databricks / Web Activity ]                  │
│                                                                        │
│  [ Datasets ] ──────────── Points to schema/files                      │
│  [ Linked Services ] ───── Connection strings & credentials (Key Vault)│
└────────────────────────────────────────────────────────────────────────┘
```

### 4.1 The Five Core Building Blocks of ADF

1. **Linked Services:** The connection configuration (similar to a connection string) that defines how ADF connects to external resources (e.g., Azure SQL, Blob Storage, AWS S3, REST API), typically referencing credentials stored securely in **Azure Key Vault**.
2. **Datasets:** Named views of data structures that point to specific tables, folders, or files within a Linked Service (e.g., `ParquetFile_ADLS`, `SalesTable_AzureSQL`).
3. **Activities:** The execution steps within a pipeline:
   - *Data Movement:* **Copy Data Activity** (High-throughput data movement).
   - *Data Transformation:* **Mapping Data Flows**, Databricks Notebook, Synapse Stored Procedure.
   - *Control Flow:* **Lookup**, **Get Metadata**, **ForEach**, **If Condition**, **Until**, **Web Activity**.
4. **Pipelines:** A logical grouping of activities that perform a unit of work (a directed acyclic graph - DAG).
5. **Triggers:** Determine when a pipeline execution is initiated:
   - **Schedule Trigger:** Standard wall-clock time (e.g., daily at midnight).
   - **Tumbling Window Trigger:** Non-overlapping, contiguous time intervals; natively supports backfilling historical windows and handling pipeline dependencies.
   - **Event-Based Trigger:** Fires in response to storage events (e.g., Blob created or deleted).

---

### 4.2 Integration Runtimes (IR)
The compute infrastructure that ADF uses to execute activities:
- **Azure IR:** Fully managed, serverless, auto-scaling compute inside Azure for cloud-to-cloud data movement.
- **Self-Hosted IR (SHIR):** Installed on-premises or on a private virtual network to securely copy data from behind corporate firewalls to the cloud without opening inbound firewall ports.
- **Azure-SSIS IR:** Dedicated virtual machine cluster to execute legacy SQL Server Integration Services (SSIS) packages natively in the cloud.

---

# Part 5: Required Deliverable: Complete API-to-Database Pipeline Blueprint

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION API-TO-WAREHOUSE INCREMENTAL ELT PIPELINE                             │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [External REST API]
         │ (HTTPS / OAuth 2.0 Bearer Token)
         ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 1. INGESTION ENGINE (ADF Web Activity / Python Azure Function)                                          │
 │    - Reads High-Watermark (last_synced_at) from Metadata Control Table                                  │
 │    - GET /api/v1/orders?modified_after={high_watermark} with Keyset Pagination                          │
 │    - Implements Exponential Backoff with Jitter for HTTP 429 / 503 errors                              │
 └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ Raw JSON Payload
 ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 2. RAW STORAGE / DATA LAKE (ADLS Gen2 - Bronze Layer)                                                   │
 │    - Path: /bronze/orders/year=2024/month=03/orders_20240315_120000.json.gz                            │
 │    - Immutable, append-only raw landing zone for full auditability & replayability                      │
 └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼ Event Notification Trigger / ADF Copy Activity
 ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 3. DATA QUALITY & QUARANTINE VALIDATION                                                                 │
 │    - Validates schema against JSON Schema contract                                                      │
 │    ├── Valid Records   ──> Land into Staging Table (staging_orders)                                     │
 │    └── Invalid Records ──> Land into Quarantine Dead Letter Queue (DLQ) + Alert Slack/Ops               │
 └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 4. CLOUD DATA WAREHOUSE (Snowflake / Azure Synapse - Silver Layer)                                      │
 │    - Executes Atomic Idempotent Upsert (MERGE INTO target USING staging):                               │
 │        WHEN MATCHED AND target.updated_at < source.updated_at THEN UPDATE SET ...                       │
 │        WHEN NOT MATCHED THEN INSERT (...)                                                               │
 └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 5. CONTROL TABLE WATERMARK UPDATE & OBSERVABILITY                                                       │
 │    - UPDATE pipeline_metadata SET last_synced_at = MAX(source.updated_at)                               │
 │    - Emit metrics to Azure Monitor: rows_ingested, latency_seconds, error_count                         │
 └─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 6: Structured Interview Question Bank

### Q1: Why is Tumbling Window Trigger preferred over Schedule Trigger for historical backfills in Azure Data Factory?
**Answer:**
A Schedule Trigger is purely wall-clock dependent; if a pipeline fails or is disabled for three days, re-enabling it only fires the current scheduled time, skipping missed days. In contrast, a **Tumbling Window Trigger** produces a contiguous, non-overlapping series of discrete time slices (e.g., 01:00–02:00, 02:00–03:00). It natively passes `@trigger().outputs.windowStartTime` parameters to the pipeline, supports built-in dependency chains (Window $N$ waits for Window $N-1$ to succeed), and allows one-click automated backfilling of past historical slices.

### Q2: What is Change Data Capture (CDC), and why is it superior to timestamp polling for large OLTP database extraction?
**Answer:**
Timestamp polling (`WHERE updated_at > :last_run`) requires executing periodic, expensive table scans against the production OLTP database, competing with user transactions for buffer memory and CPU. Crucially, timestamp polling **cannot detect hard physical deletions** (`DELETE FROM table`). CDC reads the database's internal transaction log (Write-Ahead Log in Postgres, Binlog in MySQL) asynchronously with zero impact on query execution, capturing every `INSERT`, `UPDATE`, and `DELETE` event with full pre- and post-images.

### Q3: How do you achieve end-to-end data pipeline observability?
**Answer:**
End-to-end observability requires tracking the **Four Pillars of Data Health**:
1. **Freshness:** Is the data landing within the agreed Service Level Objective (SLO)?
2. **Volume:** Did today's partition receive 10,000 rows as expected, or an anomalous 50 rows?
3. **Schema:** Did upstream changes drop a column or change an `INT` to `VARCHAR`?
4. **Lineage & Quality:** Tracking data flow from source to target, asserting null-rate checks and business invariants via automated test frameworks (e.g., Great Expectations, dbt tests).

---

# Part 7: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                     ETL/ELT & PIPELINES CHEAT SHEET                    │
├────────────────────────────────────────────────────────────────────────┤
│ 1. ETL vs ELT:                                                         │
│    - ETL: Pre-warehouse transforms. Raw data discarded. Scale limits.  │
│    - ELT: Load raw data first (Bronze). Transform inside cloud warehouse│
│           (Silver/Gold) using SQL/dbt. Scalable and re-playable.       │
├────────────────────────────────────────────────────────────────────────┤
│ 2. PIPELINE PRINCIPLES:                                                │
│    - Idempotency : Re-running pipeline produces identical state.       │
│                    Achieved via MERGE/Upsert or INSERT OVERWRITE.      │
│    - Watermarking: Polling records updated_at > last_watermark.        │
│    - CDC         : Reads DB transaction logs (Captures DELETES).       │
│    - DLQ         : Quarantines malformed records without halting DAG.  │
├────────────────────────────────────────────────────────────────────────┤
│ 3. AZURE DATA FACTORY (ADF):                                           │
│    - Linked Service: Connection string + credentials (Key Vault).      │
│    - Dataset       : Pointer to specific table/file structure.         │
│    - Activity      : Copy Data, Lookup, Web, Notebook, ForEach.        │
│    - Triggers      : Schedule, Tumbling Window (Backfills), Event.     │
│    - Integration RT: Azure IR (Cloud), Self-Hosted IR (On-Premises).   │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 8: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (20 Marks — 4 marks each)

**Q1:** What is the primary architectural advantage of modern ELT over traditional ETL?  
(a) ELT eliminates the need for database storage.  
(b) Raw data is preserved, and transformations leverage scalable, decoupled cloud warehouse compute.  
(c) ELT requires zero network bandwidth.  
(d) ELT transforms data in-memory on the source server.  

**Q2:** Which Azure Data Factory component securely manages connection strings and credentials to external databases?  
(a) Dataset  
(b) Linked Service  
(c) Pipeline  
(d) Integration Runtime  

**Q3:** Which ADF trigger type natively supports continuous, non-overlapping historical window intervals with automated backfilling?  
(a) Schedule Trigger  
(b) Tumbling Window Trigger  
(c) Blob Event Trigger  
(d) Webhook Trigger  

**Q4:** What is the fundamental limitation of timestamp-based watermarking (`WHERE updated_at > :last_watermark`)?  
(a) It cannot process more than 100 rows.  
(b) It cannot detect hard row deletions (`DELETE`) on the source table.  
(c) It requires disabling primary keys.  
(d) It only works on JSON files.  

**Q5:** In a resilient production pipeline, what is a "Dead Letter Queue" (DLQ)?  
(a) A deleted Kafka topic.  
(b) An isolated staging storage area where corrupted/unparseable records are quarantined for review without halting the pipeline.  
(c) A queue used exclusively for database backup archives.  
(d) A slow query log in PostgreSQL.  

---

### Section B: Architectural & Failure Diagnostics (25 Marks)

**Q6 (10 Marks):** Define **Idempotency** in the context of data pipelines. Explain why a pipeline that executes `INSERT INTO warehouse.sales SELECT * FROM staging.sales;` is NOT idempotent, and provide the SQL rewrite to make it fully idempotent.

**Q7 (15 Marks):** Compare the **Lambda Architecture** and **Kappa Architecture**. Why has the industry largely shifted towards Kappa architecture for modern real-time data platforms?

---

### Section C: ADF Design Implementation (35 Marks)

**Q8 (20 Marks):** You need to copy data from an on-premises Oracle database behind a corporate firewall into Azure Data Lake Storage Gen2.
1. Which type of Integration Runtime must be installed, and where? (5 marks)
2. Detail the exact ADF building blocks (Linked Services, Datasets, Activities) required to execute this data movement. (15 marks)

**Q9 (15 Marks):** Write a SQL `MERGE` statement that idempotently updates an active customer dimension `dim_customers (cust_id, email, address, updated_at)` from an incoming incremental staging table `stg_customers`, inserting new customers and updating existing customers only if `stg_customers.updated_at > dim_customers.updated_at`.

---

### Section D: Enterprise Failure Recovery Scenario (20 Marks)

**Q10 (20 Marks):** An ADF incremental pipeline runs every night at 02:00 UTC. On Thursday morning, you discover that the pipeline failed on Tuesday and Wednesday due to an expired API secret in Azure Key Vault.
1. Outline the exact step-by-step procedure to rotate the credentials and safely recover the two missed days of data without causing duplicates. (10 marks)
2. Explain how configuring Tumbling Window Triggers simplifies this backfill recovery process. (10 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** Raw data preservation + decoupled cloud compute transformation.
- **Q2:** **(b)** Linked Service.
- **Q3:** **(b)** Tumbling Window Trigger.
- **Q4:** **(b)** Inability to capture hard row deletions.
- **Q5:** **(b)** Isolated quarantine for corrupted records.

### Section B Solutions
**Q6 Solution:**
- **Idempotency:** The property where executing an operation multiple times produces the exact same outcome as executing it once.
- **Why Non-Idempotent:** Running `INSERT INTO warehouse.sales` twice inserts the same batch twice, causing duplicate rows and inflated revenue metrics.
- **Idempotent Rewrite:**
  ```sql
  -- Strategy A: Partition Overwrite
  INSERT OVERWRITE TABLE warehouse.sales PARTITION (batch_date = '2024-03-15')
  SELECT * FROM staging.sales WHERE batch_date = '2024-03-15';
  
  -- Strategy B: Truncate-Load Staging Swap
  MERGE INTO warehouse.sales target
  USING staging.sales source
  ON target.sale_id = source.sale_id
  WHEN MATCHED THEN UPDATE SET ...
  WHEN NOT MATCHED THEN INSERT ...;
  ```

**Q7 Solution:**
- **Lambda:** Dual-path (Speed Layer in streaming + Batch Layer in MapReduce/Spark + Serving Layer). Forces engineers to maintain two duplicate codebases in different frameworks for the same business logic.
- **Kappa:** Eliminates the batch layer. Treats all data as an immutable event stream (Kafka). Historical reprocessing is achieved simply by spinning up a new consumer group and replaying the stream from offset 0. Reduces code maintenance by 50% and eliminates synchronization bugs.

### Section C Solutions
**Q8 Solution:**
1. **Integration Runtime:** A **Self-Hosted Integration Runtime (SHIR)** must be installed on a Windows Server or VM located inside the corporate on-premises network that can connect to the Oracle database.
2. **ADF Components:**
   - **Linked Service 1:** `LS_OnPrem_Oracle` (uses SHIR, connects to Oracle via connection string/credentials).
   - **Linked Service 2:** `LS_Azure_ADLSGen2` (uses Azure IR, connects via Managed Identity).
   - **Dataset 1:** `DS_Oracle_SourceTable` (points to `LS_OnPrem_Oracle`).
   - **Dataset 2:** `DS_ADLS_ParquetSink` (points to `LS_Azure_ADLSGen2`, format Parquet).
   - **Activity:** **Copy Data Activity** with source set to `DS_Oracle_SourceTable` and sink set to `DS_ADLS_ParquetSink`.

**Q9 Solution:**
```sql
MERGE INTO dim_customers AS target
USING stg_customers AS source
ON target.cust_id = source.cust_id
WHEN MATCHED AND source.updated_at > target.updated_at THEN
    UPDATE SET 
        target.email = source.email,
        target.address = source.address,
        target.updated_at = source.updated_at
WHEN NOT MATCHED THEN
    INSERT (cust_id, email, address, updated_at)
    VALUES (source.cust_id, source.email, source.address, source.updated_at);
```

### Section D Solutions
**Q10 Solution:**
1. **Recovery Steps:**
   - Update the secret in Azure Key Vault; ADF Linked Service will automatically pull the new secret.
   - Test connectivity via the ADF authoring canvas.
   - Identify the earliest failed window start time (Tuesday 02:00 UTC).
   - Ensure the downstream warehouse merge statement is idempotent to prevent duplicating Tuesday's partial data.
   - Trigger the pipeline with `windowStartTime = Tuesday 02:00 UTC` and `windowEndTime = Thursday 02:00 UTC`.
2. **Tumbling Window Advantage:**
   - Tumbling Window triggers break execution into discrete, tracked temporal chunks (`T1: Tue-Wed`, `T2: Wed-Thu`).
   - In the ADF Monitoring UI, simply select the two failed window slices and click **Rerun Window**. The trigger automatically supplies the correct historical start and end timestamps to each run, executing them sequentially without manual script hacking.
