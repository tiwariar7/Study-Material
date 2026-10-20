# Day 13: Power BI, DAX & Modern Excel Analytics
## Dimensional Modeling, Power Query (M), DAX Evaluation Contexts, VertiPaq & Dashboard Engineering

---

# Part 1: Dimensional Modeling & The Star Schema

### 1.1 Facts vs. Dimensions (Kimball Methodology)

```
                       ┌──────────────────────┐
                       │    dim_customers     │
                       ├──────────────────────┤
                       │ customer_key (PK)    │
                       │ customer_name        │
                       │ segment, country     │
                       └──────────────────────┘
                                  │1
                                  │
                                  │*
┌─────────────────────┐*        ┌──────────────────────┐        *┌─────────────────────┐
│      dim_date       │─────────│      fct_orders      │─────────│    dim_products     │
├─────────────────────┤1        ├──────────────────────┤1        ├─────────────────────┤
│ date_key (PK)       │         │ order_id             │         │ product_key (PK)    │
│ calendar_date       │         │ customer_key (FK)    │         │ product_name        │
│ month, quarter, year│         │ date_key (FK)        │         │ category, unit_cost │
└─────────────────────┘         │ product_key (FK)     │         └─────────────────────┘
                                │ quantity, net_amount │
                                └──────────────────────┘
```

#### Fact Tables
Record quantitative measurements, events, or transactional metrics resulting from a business process.
- **Additive Measures:** Can be summed across all dimensions (e.g., `Sales Amount`, `Quantity Sold`).
- **Semi-Additive Measures:** Can be summed across some dimensions, but not time (e.g., `Bank Account Balance`, `Warehouse Inventory Count` — summing balances across days makes no sense; must use end-of-period snapshot).
- **Non-Additive Measures:** Cannot be meaningfully summed across any dimension (e.g., `Unit Price`, `Profit Margin %`, `Ratios`).

#### Dimension Tables
Contain qualitative context (the "who, what, where, when, why") that filters, slices, and categorizes facts. Characterized by wide tables with descriptive text attributes.

---

### 1.2 Star Schema vs. Snowflake Schema

| Feature | Star Schema | Snowflake Schema |
| :--- | :--- | :--- |
| **Structure** | Fact table at center connected directly to denormalized Dimension tables. | Dimensions are normalized into sub-tables (e.g., `Product` $\to$ `SubCategory` $\to$ `Category`). |
| **Number of Joins**| Minimal (1 join between Fact and any Dimension). | Multiple joins required to traverse normalized hierarchy. |
| **VertiPaq Engine Performance** | **Optimal** (Power BI's columnar engine is designed specifically for Star Schemas). | Sub-optimal (Extra relationship traversal adds CPU overhead). |
| **Maintenance** | Simpler DAX formulas and clearer relationship modeling for business users. | Complex relationship paths; potential ambiguity in cross-filtering. |

> [!IMPORTANT]
> **Power BI Best Practice:** Always flatten normalized hierarchies into a single denormalized Dimension table in Power Query before loading into the Power BI data model.

---

### 1.3 Relationship Cardinality & Cross-Filter Direction
- **Cardinality:**
  - **One-to-Many ($1:*$):** The standard golden pattern (Primary key of Dimension table to Foreign key of Fact table).
  - **Many-to-Many ($*:*$):** Dangerous; causes ambiguous paths and performance degradation. Should be resolved using a bridge table.
- **Cross-Filter Direction:**
  - **Single (Default):** Filters flow downstream from the "One" side (Dimension) to the "Many" side (Fact).
  - **Both (Bi-directional):** Filters flow in both directions. Avoid unless strictly required, as it creates ambiguous relationship paths and severe performance bottlenecks in large models.

---

# Part 2: Power Query & The M Language

### 2.1 The M Engine & Applied Steps
Power Query executes transformations sequentially as an immutable dependency pipeline recorded in the **M Language**. Each step takes the previous table as an input and returns a new transformed table.

### 2.2 Query Folding (The #1 Performance Factor)
**Query Folding** is the ability of Power Query to translate user interface transformation steps (filtering, grouping, joins, column removal) into a **single native SQL query** executed directly on the source relational database.

#### What folds:
- Filtering rows (`WHERE`), selecting columns (`SELECT`), joins (`INNER/LEFT JOIN`), grouping (`GROUP BY`).
#### What breaks folding:
- Complex custom M functions, changing data types to non-native types, merging with non-relational sources (Excel + SQL), or using index-based steps.

> [!TIP]
> Always verify Query Folding: Right-click the last Applied Step in Power Query. If **"View Native Query"** is enabled, folding is active, meaning the source database handles the compute rather than your desktop RAM!

---

# Part 3: DAX (Data Analysis Expressions) Masterclass

### 3.1 Calculated Columns vs. Measures

| Attribute | Calculated Column | Measure |
| :--- | :--- | :--- |
| **Evaluation Timing** | During data refresh / model load. | Dynamically at query time when visual renders. |
| **Storage** | Stored physically in RAM (compressed by VertiPaq). | Zero RAM storage (computed on-the-fly in CPU cache). |
| **Context** | Evaluated under **Row Context** (current row). | Evaluated under **Filter Context** (visual slicers, filters). |
| **Use Case** | Slicer fields, row-level categories, matrix row headers. | Numerical aggregations, KPIs, ratios, percentages. |

---

### 3.2 Evaluation Contexts: Row Context vs. Filter Context

#### 1. Row Context
The concept of "the current row". Present automatically in Calculated Columns and DAX iterator functions (`SUMX`, `AVERAGEX`, `FILTER`).
- Does **not** know about filters applied by slicers or matrix visuals!

#### 2. Filter Context
The combination of all active filters applied to the data model:
- Visual filters (Row/Column headers in a Matrix).
- Slicers selected on the report page.
- Page-level and report-level filters.

---

### 3.3 The Queen of DAX: `CALCULATE()` & Context Transition
`CALCULATE(<expression>, <filter1>, <filter2>, ...)` is the **only function in DAX that can modify the Filter Context**.

#### Context Transition:
When a Measure is called inside a Row Context (or when `CALCULATE()` wraps an expression inside a row context), it triggers **Context Transition**: it automatically converts the current row's values into an equivalent Filter Context!

```dax
-- 1. Base Measure
Total Revenue = SUM(fct_orders[net_amount])

-- 2. Modifying Filter Context (Overriding Visual Filters)
Tech Sales = 
CALCULATE(
    [Total Revenue],
    dim_products[category] = "Technology"
)

-- 3. Calculating % of Total using ALL() to remove Category Filters
% Category Contribution = 
DIVIDE(
    [Total Revenue],
    CALCULATE([Total Revenue], ALL(dim_products[category]))
)
```

---

### 3.4 Essential Time Intelligence DAX Patterns
Requires a contiguous, marked **Date Dimension Table** (`dim_date`):

```dax
-- Year-to-Date (YTD) Revenue
Revenue YTD = 
TOTALYTD([Total Revenue], dim_date[calendar_date])

-- Same Period Last Year (SPLY)
Revenue SPLY = 
CALCULATE(
    [Total Revenue],
    SAMEPERIODLASTYEAR(dim_date[calendar_date])
)

-- Month-over-Month (MoM) Growth %
MoM Revenue Growth % = 
VAR CurrentRevenue = [Total Revenue]
VAR PriorMonthRevenue = CALCULATE([Total Revenue], DATEADD(dim_date[calendar_date], -1, MONTH))
RETURN
    DIVIDE(CurrentRevenue - PriorMonthRevenue, PriorMonthRevenue)
```

---

# Part 4: Advanced Modern Excel Modeling

### 4.1 `XLOOKUP` vs. Legacy `VLOOKUP`
```excel
=XLOOKUP(lookup_value, lookup_array, return_array, [if_not_found], [match_mode], [search_mode])
```
- **Why XLOOKUP wins:**
  1. Searches to the left and right (no index column offset required).
  2. Defaults to exact match (`match_mode = 0`).
  3. Safe against column insertion/deletion (uses range references, not static column numbers).
  4. Built-in error handling (`if_not_found` replaces `IFERROR`).

### 4.2 Dynamic Array Formulas
- **`FILTER()`:** `=FILTER(A2:C100, (B2:B100="Corporate") * (C2:C100>5000), "No Records")`
- **`UNIQUE()`:** `=UNIQUE(A2:A100)`
- **`SORT()`:** `=SORT(FILTER(A2:C100, B2:B100="West"), 3, -1)` (Sorts filtered array by 3rd column descending).

---

# Part 5: Required Deliverable: Executive Sales Performance Dashboard Specification

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   EXECUTIVE SALES PERFORMANCE COCKPIT                                    │
├──────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ [ Filter Slicers: Year | Quarter | Region | Sales Segment ]                                              │
├──────────────────┬──────────────────┬──────────────────┬──────────────────┬──────────────────────────────┤
│ TOTAL REVENUE    │ GROSS PROFIT     │ PROFIT MARGIN    │ TOTAL ORDERS     │ MoM REVENUE GROWTH           │
│ $14,250,000      │ $4,560,000       │ 32.0%            │ 24,180           │ +12.4% ▲                     │
├──────────────────┴──────────────────┴──────────────────┴──────────────────┴──────────────────────────────┤
│ ┌───────────────────────────────────────────────┐  ┌───────────────────────────────────────────────────┐ │
│ │ MONTHLY REVENUE vs. SPLY (Line Chart)         │  │ CATEGORY PERFORMANCE PARETO (Bar + Line)          │ │
│ │  - Current Year Revenue (Solid Blue)          │  │  - Category Revenue (Horizontal Bars)             │ │
│ │  - Prior Year Revenue SPLY (Dashed Grey)      │  │  - Cumulative % Contribution (Line on Secondary)  │ │
│ └───────────────────────────────────────────────┘  └───────────────────────────────────────────────────┘ │
├────────────────────────────────────────────────────┬─────────────────────────────────────────────────────┤
│ REGIONAL SALES MATRIX (Drillable Dimension)        │ TOP 10 ENTERPRISE CUSTOMERS BY LTV                  │
│ North: $5.2M | South: $3.8M | West: $5.25M         │ Customer Name | Total Spend | Orders | Margin %     │
└────────────────────────────────────────────────────┴─────────────────────────────────────────────────────┘
```

### Core DAX Measures Powering the Dashboard:
```dax
-- 1. Base Revenue
Total Revenue = SUM(fct_orders[net_amount])

-- 2. Total Cost
Total Cost = SUMX(fct_orders, fct_orders[quantity] * RELATED(dim_products[unit_cost]))

-- 3. Gross Profit
Gross Profit = [Total Revenue] - [Total Cost]

-- 4. Gross Profit Margin %
Profit Margin % = DIVIDE([Gross Profit], [Total Revenue], 0)

-- 5. Prior Year Revenue (SPLY)
Revenue SPLY = CALCULATE([Total Revenue], SAMEPERIODLASTYEAR(dim_date[calendar_date]))

-- 6. Year-over-Year Growth %
YoY Revenue Growth % = DIVIDE([Total Revenue] - [Revenue SPLY], [Revenue SPLY])
```

---

# Part 6: Structured Interview Question Bank

### Q1: What is the difference between `ALL()` and `ALLSELECTED()` in DAX?
**Answer:**
- `ALL(table_or_column)` removes **all filters** from the specified column or table, regardless of where they originated (visual headers, slicers, page filters). It evaluates against the grand total of the entire dataset.
- `ALLSELECTED(table_or_column)` removes filters applied inside the visual (e.g., matrix row/column headers) while **retaining filters applied by external slicers** and page-level filters. It is essential for calculating dynamic "% of Sliced Total" in dashboards.

### Q2: Why is the VertiPaq engine so sensitive to column cardinality, and how do you optimize Power BI model memory?
**Answer:**
VertiPaq is an in-memory columnar database that compresses data using **Dictionary Encoding, Run-Length Encoding (RLE), and Bit-Packing**.
- The memory consumed by a column is directly proportional to its **Cardinality (Number of Unique Values)**.
- High-cardinality columns (such as transaction IDs, UUIDs, or timestamps with hours/minutes/seconds) destroy dictionary compression, consuming gigabytes of RAM.
- **Optimization:** Split datetime columns into separate `Date` and `Time` columns, remove unused surrogate IDs from fact tables, and replace calculated columns with dynamic measures.

### Q3: What is Context Transition in DAX, and what dangerous bug can occur if used improperly?
**Answer:**
Context Transition occurs when an expression is wrapped in `CALCULATE()` inside a Row Context (such as inside a calculated column or `SUMX`). `CALCULATE()` converts all column values of the current row into an active filter context.
- **The Bug:** If the table lacks a unique primary key, and multiple rows share identical attribute values, Context Transition will filter the table down to **all identical rows**, multiplying the aggregate calculation and producing severely inflated results.

---

# Part 7: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                      POWER BI & EXCEL CHEAT SHEET                      │
├────────────────────────────────────────────────────────────────────────┤
│ 1. MODELING:                                                           │
│    - Kimball Star Schema: Central Fact Table + Denormalized Dimensions.│
│    - Single Cross-Filter (1:*) is standard. Avoid bi-directional!     │
├────────────────────────────────────────────────────────────────────────┤
│ 2. DAX CONTEXT:                                                        │
│    - Row Context   : Evaluates current row (Calculated Columns, SUMX). │
│    - Filter Context: Visual coordinates + Slicers + Page filters.      │
│    - CALCULATE()   : ONLY way to modify Filter Context. Triggers       │
│                      Context Transition.                               │
│    - ALL()         : Clears filters (Grand Total).                     │
│    - ALLSELECTED() : Clears visual filters, keeps slicers.             │
├────────────────────────────────────────────────────────────────────────┤
│ 3. POWER QUERY (M):                                                    │
│    - Query Folding : Pushes transformations down to SQL source DB.     │
├────────────────────────────────────────────────────────────────────────┤
│ 4. MODERN EXCEL:                                                       │
│    - XLOOKUP: Searches any direction; exact match default.             │
│    - Dynamic Arrays: =FILTER(), =UNIQUE(), =SORT().                   │
│    - Multi-criteria: SUMIFS(sum_range, crit_range1, crit1, ...).       │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 8: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (20 Marks — 4 marks each)

**Q1:** What is the primary difference between a Calculated Column and a Measure in Power BI?  
(a) Calculated columns are evaluated dynamically at query time; Measures are computed during refresh.  
(b) Calculated columns consume RAM storage; Measures are computed on-the-fly and consume zero model storage.  
(c) Measures can only return text strings.  
(d) Calculated columns cannot use DAX formulas.  

**Q2:** In a Kimball Dimensional Model, what type of measure is a bank account daily balance?  
(a) Fully Additive  
(b) Semi-Additive  
(c) Non-Additive  
(d) Degenerate  

**Q3:** What is "Query Folding" in Power Query?  
(a) Compressing data into a zip file before loading.  
(b) The automated translation of transformation steps into a native SQL query executed by the source database.  
(c) Merging two Excel spreadsheets into a single sheet.  
(d) Converting rows into columns.  

**Q4:** Which DAX function removes internal visual filters while respecting external slicers selected by the user?  
(a) `ALL()`  
(b) `ALLSELECTED()`  
(c) `ALLEXCEPT()`  
(d) `REMOVEFILTERS()`  

**Q5:** What happens to the VertiPaq engine's memory compression when a column contains a unique GUID/UUID for every row?  
(a) Compression reaches maximum efficiency.  
(b) Dictionary encoding bloats, causing massive memory consumption.  
(c) The column is automatically deleted.  
(d) VertiPaq converts the GUID to an integer.  

---

### Section B: Context & DAX Tracing (25 Marks)

**Q6 (10 Marks):** Explain what the following DAX measure calculates and how `ALL(dim_products[category])` modifies the filter context:
```dax
Category Sales % = 
DIVIDE(
    SUM(fct_sales[amount]),
    CALCULATE(SUM(fct_sales[amount]), ALL(dim_products[category]))
)
```

**Q7 (15 Marks):** Consider a table `orders` with columns `[customer_id]`, `[order_id]`, and `[amount]`. What is the fundamental difference in execution and result between:
- Expression A: `SUM(orders[amount])`
- Expression B: `SUMX(orders, orders[amount] * (1 - orders[discount_pct]))`

---

### Section C: Implementation Challenges (35 Marks)

**Q8 (20 Marks):** Write production DAX formulas for the following business requirements:
1. `YTD Sales`: Year-to-date sales based on the Date table.
2. `Top Category Sales`: Total sales filtered strictly to products where `category = 'Luxury'`, ignoring any visual slicers filtering that category.
3. `Customer 30-Day Churn Risk Flag`: A calculated column in `dim_customers` that flags `'At Risk'` if their `days_since_last_purchase > 30`, otherwise `'Active'`.

**Q9 (15 Marks):** Write the modern Excel Dynamic Array formula using `=FILTER()` and `=SORT()` to extract all rows from `A2:D100` where Region (Column B) is `"West"` and Revenue (Column D) is greater than `$10,000`, sorted by Revenue descending.

---

### Section D: Real-World Performance Tuning Scenario (20 Marks)

**Q10 (20 Marks):** A Power BI report with 20 million rows takes 18 seconds to refresh each visual. Diagnostic investigation reveals:
1. Two calculated columns exist on the fact table: `Customer_Full_Address` (strings) and `Order_Timestamp_Seconds` (DateTime with second-level precision).
2. Bi-directional cross-filtering is enabled between `dim_customers` and `fct_orders`.
3. Power Query applied steps include a Python script step that breaks Query Folding.

Explain how addressing these three bottlenecks will reduce model size by over 70% and drop visual rendering times to under 1 second.

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** Calculated columns consume RAM; Measures are query-time calculations consuming 0 model storage.
- **Q2:** **(b)** Semi-Additive (cannot be summed across time).
- **Q3:** **(b)** Translation of M transformation steps into native source SQL.
- **Q4:** **(b)** `ALLSELECTED()`.
- **Q5:** **(b)** High cardinality destroys dictionary compression, causing severe RAM bloat.

### Section B Solutions
**Q6 Solution:**
- **What it calculates:** The percentage contribution of the currently selected product category relative to the total sales across all categories.
- **Filter Modification:** When viewed in a category visual (e.g., Category = "Electronics"), the numerator evaluates sales for Electronics. `CALCULATE(..., ALL(dim_products[category]))` strips the "Electronics" filter off the denominator, calculating the grand total across all categories.

**Q7 Solution:**
- **Expression A:** Standard simple aggregation. Directly aggregates the compressed columnar vector in VertiPaq via vectorized CPU instructions ($O(1)$ memory).
- **Expression B:** Iterator function (`SUMX`). Creates an active **Row Context**, iterating row-by-row across the table, computing the discounted amount per row, and accumulating the sum.

### Section C Solutions
**Q8 Solution:**
```dax
-- 1. YTD Sales
YTD Sales = TOTALYTD(SUM(fct_sales[amount]), dim_date[date_key])

-- 2. Top Category Sales
Top Category Sales = 
CALCULATE(
    SUM(fct_sales[amount]),
    ALL(dim_products[category]),
    dim_products[category] = "Luxury"
)

-- 3. Customer Calculated Column
Customer Churn Flag = 
IF(dim_customers[days_since_last_purchase] > 30, "At Risk", "Active")
```

**Q9 Solution:**
```excel
=SORT(FILTER(A2:D100, (B2:B100="West") * (D2:D100>10000), "No Records"), 4, -1)
```

### Section D Solutions
**Q10 Solution:**
1. **Eliminate High-Cardinality Calculated Columns:**
   - Drop `Customer_Full_Address` from the fact table; keep it in `dim_customers` where it is stored once per customer.
   - Split `Order_Timestamp_Seconds` into a separate `Date` key and an integer `Hour` column. Truncating seconds reduces cardinality from millions of unique timestamps to 24 unique hours, enabling 90%+ VertiPaq dictionary compression.
2. **Disable Bi-Directional Cross-Filtering:**
   - Change relationship back to **Single Direction** (`dim_customers (1) -> fct_orders (*)`). This eliminates ambiguity loops and prevents the engine from generating multi-table joining bitmaps on every click.
3. **Restore Query Folding in Power Query:**
   - Remove the Python script step or move it to upstream ETL/warehouse staging. When folding is restored, filters and joins execute directly inside the source database engine, pulling only clean, aggregated data into Power BI.
