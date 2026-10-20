# Day 25: Data Engineering & Business Intelligence Comprehensive Revision
## End-to-End Modern Data Stack, Pipeline Blueprinting, Production Debugging & BI Architecture

---

# Part 1: The Modern Data Stack Architectural Blueprint

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  ENTERPRISE MODERN DATA STACK BLUEPRINT                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [EXTERNAL DATA SOURCES]
  ├── REST APIs (Stripe, Salesforce, Procore) ──> OAuth 2.0 / Cursor Pagination / Exponential Backoff
  └── Production OLTP DBs (PostgreSQL, Oracle) ──> Change Data Capture (Debezium / Binlog)
                                │
                                ▼
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 1. ORCHESTRATION & INGESTION (Apache Airflow / Azure Data Factory)                                    │
 │    - Self-Hosted IR (SHIR) securely bridges on-premise firewalls without opening inbound ports        │
 │    - Web Activity & Copy Data Activity stages raw data into Cloud Storage                             │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼ (Zero-Loss Raw Landing)
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 2. CLOUD OBJECT STORAGE (Bronze Data Lake - ADLS Gen2 / AWS S3)                                       │
 │    - Immutable, append-only raw JSON/Parquet storage partitioned by date: /bronze/year=2024/month=03/ │
 │    - Full historical fidelity for disaster recovery, compliance, and backfill replays                 │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼ (Snowflake COPY INTO / Snowpipe Event Loading)
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 3. CLOUD DATA WAREHOUSE (Snowflake Storage & Compute)                                                 │
 │    - Database Storage: Columnar, encrypted micro-partitions with automated min/max metadata pruning    │
 │    - Decoupled Compute: Independent Virtual Warehouses (ETL_WH vs. BI_WH) eliminating contention      │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼ (dbt Transformation as Code)
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 4. TRANSFORMATION AS CODE (dbt - Data Build Tool)                                                     │
 │    - Silver Layer: Cleansed, deduplicated 3NF tables; enforces dbt tests (unique, not_null)           │
 │    - Gold Layer: Star Schema Dimensional Models (fct_sales, dim_customers, dim_date)                  │
 │    - Lineage Graph: Directed Acyclic Graph compiled via {{ ref() }} macros                            │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                │
                                ▼ (VertiPaq DirectQuery or Import Refresh)
 ┌───────────────────────────────────────────────────────────────────────────────────────────────────────┐
 │ 5. BUSINESS INTELLIGENCE & REPORTING (Power BI & Modern Excel)                                        │
 │    - Star Schema: Single-direction 1:* relationships; denormalized dimensions                         │
 │    - DAX Analytical Measures: Dynamic Filter Context evaluation, CALCULATE(), Time Intelligence       │
 │    - Executive Cockpit: Real-time KPIs, Pareto distribution, and automated anomaly alerts             │
 └───────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

# Part 2: The 6-Step Systematic Pipeline Debugging Protocol

When an interviewer asks: *"Your nightly pipeline failed at 03:00 AM, and executive dashboards are displaying stale data. How do you investigate, fix, and communicate the incident?"*, follow this production protocol:

```
┌────────────────────────────────────────────────────────────────────────┐
│               THE 6-STEP PRODUCTION INCIDENT PROTOCOL                  │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 1: TRIAGE & STAKEHOLDER COMMUNICATION (0 - 5 mins)                │
│  - Post status notice to Slack #data-incidents. Set report banner.     │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 2: ISOLATE THE POINT OF FAILURE IN THE DAG (5 - 10 mins)          │
│  - Inspect Airflow Task Instance logs or ADF Activity run output.      │
│  - Classify failure: Ingestion vs. Storage vs. Transformation vs. BI.  │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 3: ANALYZE ERROR SIGNATURES & ROOT CAUSE (10 - 20 mins)           │
│  - Auth Error: Expired Key Vault secret or token refresh failure.      │
│  - Schema Drift: Upstream source dropped or renamed a critical column. │
│  - Memory/Spill: Sort operator exceeded work_mem; OOM kill on worker.  │
│  - Constraint Failure: dbt test failed on duplicate natural keys.      │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 4: APPLY DEFENSIVE REMEDIATION (20 - 35 mins)                     │
│  - Quarantine corrupted records to Dead Letter Queue (DLQ).            │
│  - Update dbt model or schema contract; hotfix configuration.          │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 5: IDEMPOTENT BACKFILL REPLAY (35 - 45 mins)                      │
│  - Re-run the specific failed task slice using Tumbling Window rerun.  │
│  - Verify MERGE statement upserted records without duplication.        │
├────────────────────────────────────────────────────────────────────────┤
│ STEP 6: POST-MORTEM & PERMANENT PREVENTION (Within 24 hours)           │
│  - Publish formal 5-Whys root cause analysis (RCA).                    │
│  - Implement automated dbt source freshness tests and alert rules.     │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 3: Cross-Cutting Data Engineering & BI Interview Bank

### Q1: Why does a Star Schema outperform a denormalized "One Big Table" (OBT) in Power BI, whereas OBT is often preferred in Snowflake/BigQuery?
**Answer:**
- **In Power BI (VertiPaq Engine):** VertiPaq uses columnar in-memory dictionary encoding. Storing descriptive strings (e.g., customer addresses) in a single normalized dimension table encodes the text once; repeating those strings across 50 million rows in an OBT destroys dictionary compression and balloons RAM usage. Furthermore, Star Schemas enable clean, intuitive DAX filter context propagation.
- **In Snowflake / BigQuery (Cloud OLAP):** Storage is cheap and compute is massively parallelized (MPP). Querying an OBT completely eliminates join overhead and join shuffle across distributed compute nodes, making OBT optimal for fast ad-hoc scanning.

### Q2: What is "Context Transition" in DAX, and how does it relate to SQL window functions?
**Answer:**
In DAX, Context Transition occurs when `CALCULATE()` is executed within an active Row Context (such as inside a calculated column or iterator like `SUMX`). `CALCULATE()` converts all column values of the current row into an equivalent active Filter Context.  
This is conceptually identical to a SQL window function `OVER (PARTITION BY current_row_values)`: it calculates an aggregated metric across the subset of records that share the current row's coordinates without collapsing the row itself.

### Q3: How do you handle late-arriving dimension records in a real-time data warehouse?
**Answer:**
When a transactional fact record arrives (e.g., a sale made by `customer_id = 999`), but the corresponding customer profile has not yet replicated from the CRM:
1. **Inferred Member Pattern:** The pipeline inserts a placeholder "dummy" record into `dim_customers` with `customer_id = 999`, setting attributes to `'Unknown / Pending'` and recording an `is_inferred = TRUE` flag.
2. The fact record joins successfully with the inferred dimension record, preserving referential integrity and preventing pipeline failure.
3. When the true customer record eventually arrives from the CRM, a `MERGE` statement overwrites the placeholder attributes while preserving the original surrogate key.

---

# Part 4: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   DATA ENGINEERING & BI CHEAT SHEET                    │
├────────────────────────────────────────────────────────────────────────┤
│ 1. PIPELINE BLUEPRINT:                                                 │
│    - API/CDC -> ADF/Airflow -> ADLS/S3 Bronze -> Snowflake Silver ->   │
│      dbt Gold Star Schema -> Power BI VertiPaq Dashboard.              │
├────────────────────────────────────────────────────────────────────────┤
│ 2. DEBUGGING CHECKLIST:                                                │
│    - Check Airflow task logs -> Identify error class (Auth, Schema,    │
│      Memory, Duplicates) -> Quarantine to DLQ -> Replay via MERGE.    │
├────────────────────────────────────────────────────────────────────────┤
│ 3. SNOWFLAKE vs POWER BI MODELING:                                     │
│    - Snowflake: OBT or Star Schema. Decoupled storage & compute.       │
│    - Power BI : STRICT STAR SCHEMA. Minimize column cardinality!       │
├────────────────────────────────────────────────────────────────────────┤
│ 4. ESSENTIAL DAX:                                                      │
│    - CALCULATE(): Only way to modify filter context.                   │
│    - ALL(): Strips filters (Grand Total).                              │
│    - ALLSELECTED(): Strips visual filters, preserves slicers.          │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 5: 100-Mark Self-Assessment Examination

### Section A: Architectural Foundations (25 Marks — 5 marks each)

**Q1:** What is the primary operational purpose of a Self-Hosted Integration Runtime (SHIR) in Azure Data Factory?  
(a) To execute Python scripts on GPUs.  
(b) To establish a secure bridge for copying data between private on-premises corporate networks and the Azure cloud without opening inbound firewall ports.  
(c) To host web APIs.  
(d) To replace Snowflake.  

**Q2:** In dbt, what does the macro `{{ source('raw_data', 'orders') }}` accomplish compared to hardcoding `raw_data.orders`?  
(a) Encrypts the table with AES-256.  
(b) Declares external source dependencies, enabling automated source freshness testing and graphical lineage tracing.  
(c) Converts the table into a CSV file.  
(d) Automatically deletes null values.  

**Q3:** In Power BI, why is bi-directional cross-filtering between Fact and Dimension tables strongly discouraged?  
(a) It causes syntax errors in DAX.  
(b) It introduces ambiguous relationship paths, degrades VertiPaq query performance, and can cause unexpected filter context propagation.  
(c) Power BI does not support bi-directional filtering.  
(d) It requires Excel licenses.  

**Q4:** In Snowflake, what happens to underlying storage costs when a 10 TB database is cloned using `CREATE DATABASE dev CLONE prod;`?  
(a) Storage cost immediately doubles (incurs an extra 10 TB charge).  
(b) Zero extra storage cost initially; both databases share the same immutable micro-partitions via metadata pointers until modifications occur (Copy-on-Write).  
(c) The production database is permanently deleted.  
(d) Cloning requires downloading data to local disk.  

**Q5:** What type of measure is a bank account balance in a Kimball Dimensional Model?  
(a) Fully Additive  
(b) Semi-Additive (cannot be summed across the Time dimension)  
(c) Non-Additive  
(d) Degenerate  

---

### Section B: Failure Debugging Scenarios (45 Marks)

**Q6 (25 Marks - The Broken Nightly Load):**  
At 06:00 AM, your team discovers that the automated nightly pipeline failed at the dbt transformation stage with the error:  
`dbt test failure: 14 failures found on unique_order_id in model stg_orders`.  
1. Explain how duplicate natural keys in the raw source system caused this failure. (10 marks)
2. Detail the exact deduplication SQL transformation using `ROW_NUMBER()` that must be implemented in the staging layer to resolve this issue. (15 marks)

**Q7 (20 Marks - Power BI Memory Degradation):**  
A Power BI report connected to Snowflake via Import Mode takes 25 minutes to refresh and consumes 4.2 GB of RAM on the Power BI Premium capacity. Investigation reveals a calculated column `Event_UUID` exists on the fact table. Explain why removing this single column will reduce model size by over 60%.

---

### Section C: Systems Integration Challenge (30 Marks)

**Q8 (30 Marks):** Draw the architectural component diagram and write the end-to-end data flow specifications for an **AEC Construction Equipment Telemetry Platform** at vConstruct:
1. IoT field equipment transmits GPS and engine idle-time metrics via REST API.
2. Azure Data Factory ingests data incrementally into an ADLS Gen2 Bronze lake.
3. Snowflake loads raw data via `COPY INTO` and dbt cleanses and models it into an equipment utilization Star Schema.
4. Power BI presents an executive dashboard displaying the **Equipment Utilization Rate %** and **Idle Fuel Waste Cost ($)**.

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** Secure bridge for on-premises to cloud data movement without inbound ports.
- **Q2:** **(b)** External source declaration, freshness testing, and lineage graph generation.
- **Q3:** **(b)** Ambiguous relationship paths and severe performance degradation.
- **Q4:** **(b)** Zero extra storage cost initially (Metadata Zero-Copy Cloning).
- **Q5:** **(b)** Semi-Additive.

### Section B Solutions
**Q6 Solution:**
1. **Root Cause:** Upstream source transactional systems updated order records without enforcing uniqueness in the extract window (e.g., an order was updated twice in the same batch, or network retries inserted duplicate payloads into the raw landing zone). When dbt executed its generic `unique` test on `order_id`, it detected multiple rows sharing the same key, halting the pipeline to prevent corrupted downstream financial metrics.
2. **Deduplication SQL Pattern:**
   ```sql
   WITH RankedRaw AS (
       SELECT 
           order_id,
           customer_id,
           order_date,
           total_amount,
           updated_at,
           ROW_NUMBER() OVER (
               PARTITION BY order_id 
               ORDER BY updated_at DESC, _ingested_at DESC
           ) AS rn
       FROM {{ source('raw', 'orders') }}
   )
   SELECT 
       order_id,
       customer_id,
       order_date,
       total_amount,
       updated_at
   FROM RankedRaw
   WHERE rn = 1;
   ```

**Q7 Solution:**
- VertiPaq memory compression depends strictly on **column cardinality** (the number of unique values).
- An `Event_UUID` on a 20-million row fact table contains 20 million unique strings.
- This completely destroys dictionary encoding (creating a 20-million-entry string hash table) and renders Run-Length Encoding (RLE) and bit-packing completely useless.
- Because a UUID is never used as an analytical dimension or aggregation metric in dashboards, dropping it eliminates the dictionary table, reclaiming 60–80% of model RAM.

### Section C Solutions
**Q8 Component Architecture & Data Flow:**
1. **Ingestion Layer:**
   - IoT Gateway buffers telemetry into Azure Event Hubs / REST API.
   - ADF pipeline triggers an hourly Tumbling Window copying raw JSON payloads into ADLS Gen2 Bronze (`/bronze/equipment_telemetry/year=2024/...`).
2. **Storage & Warehousing (Snowflake):**
   - Snowflake executes `COPY INTO bronze.raw_telemetry FROM @adls_telemetry_stage FILE_FORMAT = (TYPE = 'JSON');`.
3. **dbt Transformation (Silver $\to$ Gold):**
   - `stg_telemetry`: Extracts JSON keys (`equipment_id`, `engine_status`, `fuel_rate`, `timestamp`).
   - `fct_equipment_daily_activity`: Aggregates operating hours, idle hours, and fuel burned per equipment per day.
   - `dim_equipment`: Equipment metadata (Model, Trade, Job Site, Hourly Rental Cost).
4. **Power BI Executive Dashboard:**
   - Star Schema model (`fct_equipment_daily_activity` connected $1:*$ to `dim_equipment` and `dim_date`).
   - DAX Measures:
     ```dax
     Utilization Rate % = 
     DIVIDE(SUM(fct_equipment_daily_activity[operating_hours]), 
            SUM(fct_equipment_daily_activity[total_logged_hours]))

     Idle Fuel Cost ($) = 
     SUM(fct_equipment_daily_activity[idle_fuel_gallons]) * 4.25
     ```
