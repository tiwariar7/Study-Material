# Day 8: NumPy & Pandas for High-Performance Data Analysis
## Vectorization, Indexing Internals, GroupBy, Merging, Pivoting, Data Cleaning & Time-Series

---

# Part 1: NumPy Internals & Vectorized Computing

### 1.1 The `ndarray` Memory Model
In standard Python, a `list` is an array of pointers to boxed heap objects. A NumPy `ndarray` is a contiguous block of homogeneous raw bytes in memory with an attached metadata header:
- **`dtype` (Data Type):** Tells NumPy how to interpret raw bytes (e.g., `int32`, `float64`).
- **`shape`:** A tuple representing array dimensions (e.g., `(1000, 4)`).
- **`strides`:** A tuple of bytes to step in each dimension when traversing memory.

```python
import numpy as np

arr = np.array([[10, 20, 30], [40, 50, 60]], dtype=np.int32)
print("Shape:", arr.shape)      # (2, 3)
print("Dtype:", arr.dtype)      # int32 (4 bytes per element)
print("Strides:", arr.strides)  # (12, 4) -> 12 bytes to jump to next row, 4 bytes to jump to next column
```

---

### 1.2 Vectorization vs. Python Loops
Vectorization eliminates the overhead of Python bytecode interpretation, dynamic type checking, and pointer chasing by executing operations directly in compiled C/Fortran code using modern CPU **SIMD (Single Instruction, Multiple Data)** vector registers.

```python
import time

size = 10_000_000
python_list = list(range(size))
numpy_array = np.arange(size, dtype=np.int64)

# Pure Python Loop:
t0 = time.perf_counter()
res_list = [x * 2 for x in python_list]
t1 = time.perf_counter()
print(f"Python list comprehension: {t1 - t0:.4f} seconds")  # ~0.65 seconds

# Vectorized NumPy:
t0 = time.perf_counter()
res_arr = numpy_array * 2
t1 = time.perf_counter()
print(f"NumPy vectorized multiplication: {t1 - t0:.4f} seconds")  # ~0.009 seconds (~70x speedup!)
```

---

### 1.3 The Rules of Array Broadcasting
Broadcasting allows NumPy to perform arithmetic operations on arrays of differing shapes without allocating unnecessary memory copies.

#### The Formal Broadcasting Rule:
Starting from the **trailing (right-most) dimensions** and moving leftward, two dimensions are compatible if:
1. They are **equal**, OR
2. One of them is **1**.

*Valid Example:*
```
Array A: (5, 1, 4)
Array B:    (3, 4)  --> Trailing dimensions match: 4 == 4; middle dimension matches: 1 and 3 (broadcasting expands 1 to 3); leading dimension expands 1 to 5.
Result:  (5, 3, 4)
```

```python
matrix = np.array([[1, 2, 3], [4, 5, 6]])  # Shape (2, 3)
row_vector = np.array([10, 20, 30])         # Shape (3,) -> broadcasts to (2, 3)
print(matrix + row_vector)
# Output:
# [[11, 22, 33],
#  [14, 25, 36]]
```

---

# Part 2: Pandas Core Architecture & Indexing Internals

### 2.1 Series & DataFrame Architecture
- **Series:** A 1D labeled array capable of holding any data type, composed of a NumPy array and an index label array.
- **DataFrame:** A 2D tabular data structure with labeled axes (rows and columns). Internally, columns of identical data types are grouped into unified memory arrays managed by the **BlockManager** (or 1D arrays in the newer **ArrayManager**).

---

### 2.2 Indexing: `loc` vs. `iloc` vs. `at` / `iat`

| Accessor | Basis of Indexing | Bounds Inclusion | Negative Indices? | Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **`.loc[]`** | **Label-based** (index names, column names) | **Both endpoints INCLUDED** (`'A':'C'` includes `'C'`) | Only if index contains negative integers | Production filtering by boolean masks or named labels |
| **`.iloc[]`** | **Integer-based** (0-indexed integer position) | Standard Python: **Stop bound EXCLUDED** (`0:3` gets 0, 1, 2) | Supported (counts from end) | Positional array slicing |
| **`.at[]`** | Label-based scalar lookup | Scalar only | No | Ultra-fast single-cell lookup |
| **`.iat[]`** | Position-based scalar lookup | Scalar only | Supported | Ultra-fast single-cell lookup by integer coordinates |

```python
import pandas as pd

df = pd.DataFrame(
    {"name": ["Alice", "Bob", "Charlie"], "salary": [90000, 110000, 125000]},
    index=["emp_101", "emp_102", "emp_103"]
)

# loc: Label-based (Includes endpoint!)
print(df.loc["emp_101":"emp_102", "name":"salary"])

# iloc: Position-based (Excludes endpoint!)
print(df.iloc[0:2, 0:2])
```

---

### 2.3 The `SettingWithCopyWarning` Demystified
One of the most common pitfalls in Pandas is **Chained Indexing**:

```python
# THE BUGGY PATTERN (Chained Indexing)
df[df["salary"] > 100000]["salary"] = 105000
# Pandas triggers: SettingWithCopyWarning: A value is trying to be set on a copy of a slice from a DataFrame.
```
#### Why does this happen?
`df[df["salary"] > 100000]` executes first and returns an intermediate object. Pandas cannot guarantee whether this intermediate object is a **view** (sharing memory with `df`) or an isolated **copy**. Modifying it may update `df` or modify a throwaway temporary copy!

#### The Bulletproof Solution:
Always use `.loc` to perform filtering and assignment in a **single atomic operation**:
```python
df.loc[df["salary"] > 100000, "salary"] = 105000  # Guaranteed to modify original df in place!
```

---

# Part 3: Advanced Grouping & Relational Operations

### 3.1 The Split-Apply-Combine Pattern
When calling `df.groupby('dept')`:
1. **Split:** Data is partitioned into discrete groups based on key values without building full intermediate DataFrames.
2. **Apply:** An aggregation, transformation, or filter function is applied to each group.
3. **Combine:** Results are concatenated into an output DataFrame.

```python
df_sales = pd.DataFrame({
    "dept": ["HR", "IT", "IT", "HR", "IT"],
    "emp": ["Alice", "Bob", "Charlie", "David", "Evan"],
    "salary": [70000, 120000, 110000, 80000, 140000]
})

# 1. Multi-Aggregate (Named Aggregation)
dept_summary = df_sales.groupby("dept").agg(
    headcount=("emp", "count"),
    avg_salary=("salary", "mean"),
    max_salary=("salary", "max")
).reset_index()
```

---

### 3.2 `transform()`: Emulating SQL Window Functions
Unlike `.agg()` (which collapses $N$ rows into 1 row per group), `.transform()` calculates the group metric and **broadcasts it back to the original DataFrame's row dimensions**.

```python
# Calculate each employee's salary difference from their department average
df_sales["dept_avg"] = df_sales.groupby("dept")["salary"].transform("mean")
df_sales["salary_diff"] = df_sales["salary"] - df_sales["dept_avg"]
# Identical to SQL: salary - AVG(salary) OVER (PARTITION BY dept)
```

---

### 3.3 `filter()`: Emulating SQL `HAVING`
`.filter()` discards entire groups that do not satisfy a group-level boolean predicate.

```python
# Keep only departments with at least 3 employees
large_depts = df_sales.groupby("dept").filter(lambda g: len(g) >= 3)
```

---

### 3.4 Relational Joins with `pd.merge()`
```python
customers = pd.DataFrame({"cust_id": [1, 2, 3], "name": ["Acme", "Beta", "Cyber"]})
orders = pd.DataFrame({"order_id": [101, 102], "cust_id": [1, 2], "val": [500, 750]})

# Production-grade merge with explicit cardinality validation:
merged = pd.merge(
    customers, 
    orders, 
    on="cust_id", 
    how="left", 
    validate="one_to_many"  # Asserts customers has unique cust_id!
)
```

---

# Part 4: Reshaping Data: Pivoting, Melting & Stacking

```
WIDE FORMAT (Human-friendly / Dashboards)
| Date       | Electronics | Fashion | Home |
| 2024-01-01 | 15000       | 8000    | 4000 |

               ▲                          │
               │                          │
        pivot_table()                   melt()
               │                          │
               │                          ▼
LONG FORMAT (Tidy Data / Analytics & Machine Learning)
| Date       | Category    | Revenue |
| 2024-01-01 | Electronics | 15000   |
| 2024-01-01 | Fashion     | 8000    |
| 2024-01-01 | Home        | 4000    |
```

### 4.1 `pivot_table()` (Long to Wide with Aggregation)
```python
df_orders = pd.DataFrame({
    "region": ["East", "East", "West", "West"],
    "quarter": ["Q1", "Q2", "Q1", "Q2"],
    "revenue": [50000, 60000, 45000, 55000]
})

pivot = df_orders.pivot_table(
    index="region", 
    columns="quarter", 
    values="revenue", 
    aggfunc="sum", 
    fill_value=0,
    margins=True  # Appends 'All' Total row and column!
)
```

### 4.2 `melt()` (Wide to Long)
```python
wide_df = pd.DataFrame({
    "country": ["USA", "Germany"],
    "year_2022": [100, 80],
    "year_2023": [115, 85]
})

tidy_df = pd.melt(
    wide_df,
    id_vars=["country"],
    value_vars=["year_2022", "year_2023"],
    var_name="year",
    value_name="gdp_index"
)
```

---

# Part 5: Data Cleaning & Quality Operations

### 5.1 Missing Data Strategies
- `df.isna().sum()`: Measures null distribution.
- `df.dropna(subset=['critical_col'])`: Drops rows with nulls in critical attributes.
- Imputation Techniques:
  ```python
  # Mean / Median Imputation
  df["salary"] = df["salary"].fillna(df["salary"].median())

  # Forward / Backward Fill for Time-Series
  df["price"] = df["price"].ffill().bfill()
  ```

---

### 5.2 Outlier Detection Techniques

#### 1. Interquartile Range (IQR) Method:
```python
def remove_outliers_iqr(df: pd.DataFrame, column: str) -> pd.DataFrame:
    q1 = df[column].quantile(0.25)
    q3 = df[column].quantile(0.75)
    iqr = q3 - q1
    lower_bound = q1 - 1.5 * iqr
    upper_bound = q3 + 1.5 * iqr
    return df[(df[column] >= lower_bound) & (df[column] <= upper_bound)]
```

#### 2. Z-Score Method:
```python
def flag_outliers_zscore(df: pd.DataFrame, column: str, threshold: float = 3.0) -> pd.Series:
    mean = df[column].mean()
    std = df[column].std()
    z_scores = (df[column] - mean) / std
    return z_scores.abs() > threshold
```

---

### 5.3 Memory Optimization via Categorical Types
Converting low-cardinality string columns (`gender`, `status`, `country`) to `category` dtypes replaces repeated strings with integer lookup keys, often reducing DataFrame RAM footprint by $80\%$:
```python
df["status"] = df["status"].astype("category")
```

---

# Part 6: Time-Series & Datetime Analytics

```python
df["order_timestamp"] = pd.to_datetime(df["order_timestamp"])

# 1. Component Extraction via .dt accessor
df["year"] = df["order_timestamp"].dt.year
df["month_name"] = df["order_timestamp"].dt.month_name()
df["day_of_week"] = df["order_timestamp"].dt.day_name()
df["is_weekend"] = df["order_timestamp"].dt.dayofweek.isin([5, 6])

# 2. Resampling (Time-based GroupBy)
monthly_sales = df.set_index("order_timestamp")["revenue"].resample("M").sum()

# 3. Rolling Moving Windows
df["7d_rolling_avg"] = df.set_index("order_timestamp")["revenue"].rolling("7D").mean().values
```

---

# Part 7: Required Deliverable: Messy CSV Data Ingestion & Executive Summary Pipeline

### Business Scenario
You are given a raw, corrupted sales log CSV file `raw_sales_data.csv`. The data contains:
- Missing customer IDs and negative quantities.
- String currency fields formatted like `"$1,250.00"`.
- Inconsistent date formats.
- Duplicate order IDs.

### Production Solution Script
```python
import pandas as pd
import numpy as np

def clean_and_summarize_sales(file_path: str) -> pd.DataFrame:
    """
    Ingests dirty transaction data, validates integrity, removes anomalies,
    and produces an executive summary table by Product Category.
    """
    # 1. Ingestion
    df = pd.read_csv(file_path)
    
    # 2. Deduplication on Business Key
    initial_rows = len(df)
    df = df.drop_duplicates(subset=["order_id"], keep="first")
    
    # 3. Clean Currency Strings to Numeric Float
    df["unit_price"] = (
        df["unit_price"]
        .astype(str)
        .str.replace("$", "", regex=False)
        .str.replace(",", "", regex=False)
        .str.strip()
    )
    df["unit_price"] = pd.to_numeric(df["unit_price"], errors="coerce")
    
    # 4. Standardize Dates
    df["order_date"] = pd.to_datetime(df["order_date"], errors="coerce")
    
    # 5. Handle Integrity Anomalies (Negative quantity, null dates, null prices)
    valid_mask = (
        df["order_date"].notna() &
        df["unit_price"].notna() &
        (df["unit_price"] > 0) &
        (df["quantity"] > 0)
    )
    df_clean = df.loc[valid_mask].copy()
    
    # 6. Feature Engineering
    df_clean["gross_revenue"] = df_clean["quantity"] * df_clean["unit_price"]
    
    # 7. Executive Summary Table
    summary = df_clean.groupby("category").agg(
        total_orders=("order_id", "count"),
        total_units_sold=("quantity", "sum"),
        total_revenue=("gross_revenue", "sum"),
        avg_order_value=("gross_revenue", "mean"),
        max_single_sale=("gross_revenue", "max")
    ).reset_index()
    
    # Format currency for executive presentation
    summary["total_revenue"] = summary["total_revenue"].round(2)
    summary["avg_order_value"] = summary["avg_order_value"].round(2)
    
    # Append Company-Wide Total Row
    total_row = pd.DataFrame([{
        "category": "TOTAL / COMPANY",
        "total_orders": summary["total_orders"].sum(),
        "total_units_sold": summary["total_units_sold"].sum(),
        "total_revenue": summary["total_revenue"].sum(),
        "avg_order_value": round(df_clean["gross_revenue"].mean(), 2),
        "max_single_sale": summary["max_single_sale"].max()
    }])
    
    final_executive_table = pd.concat([summary, total_row], ignore_index=True)
    return final_executive_table
```

---

# Part 8: Structured Interview Question Bank

### Q1: What is the difference between `df['col']` vs `df[['col']]`?
**Answer:**
- `df['col']` extracts a 1-dimensional **Pandas Series**.
- `df[['col']]` extracts a 2-dimensional **Pandas DataFrame** containing a single column. This distinction matters because DataFrame methods and Series methods differ, and passing a list of column names preserves DataFrame structure.

### Q2: Why is iterating over a DataFrame with `for index, row in df.iterrows():` strongly discouraged in production?
**Answer:**
`iterrows()` is notoriously slow because for every single row, it boxes the data into a brand-new Pandas Series object and checks data types dynamically, completely bypassing NumPy's vectorization and C-level memory speed. Operations using `iterrows()` are typically $100\times$ to $1000\times$ slower than vectorized operations, `.apply()`, or list comprehensions over NumPy arrays (`df['col'].to_numpy()`).

### Q3: Explain the difference between `merge()` and `concat()` in Pandas.
**Answer:**
- `pd.merge()` performs database-style relational joins based on common key columns or index labels (equivalent to SQL `INNER`, `LEFT`, `RIGHT`, `OUTER` joins).
- `pd.concat()` stitches DataFrames together along an axis (vertically `axis=0` like SQL `UNION ALL`, or horizontally `axis=1`) purely based on index alignment, without evaluating key join predicates.

### Q4: How does Pandas handle missing values under the hood for integer and boolean columns?
**Answer:**
Historically, Pandas coerced integer columns containing missing values to `float64` because standard NumPy arrays did not support integer `NaN`. In modern Pandas (versions 1.0+), Pandas introduced nullable data types (`Int64`, `boolean`, `string`) backed by a separate boolean bitmask that tracks null status without altering the underlying numerical data type.

---

# Part 9: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                      NUMPY & PANDAS CHEAT SHEET                        │
├────────────────────────────────────────────────────────────────────────┤
│ 1. NUMPY:                                                              │
│    - Homogeneous memory buffer. Strides define byte steps.             │
│    - Broadcasting: Trailing dims must match or be 1.                   │
├────────────────────────────────────────────────────────────────────────┤
│ 2. INDEXING:                                                           │
│    - .loc[] : Label-based (INCLUDES endpoint).                         │
│    - .iloc[]: Integer position-based (EXCLUDES endpoint).              │
│    - Avoid Chained Indexing: df.loc[mask, 'col'] = val.                │
├────────────────────────────────────────────────────────────────────────┤
│ 3. GROUPBY PATTERNS:                                                   │
│    - .agg()      : Collapses rows into summary metrics.                │
│    - .transform(): Broadcasts group metrics back to row granularity.   │
│    - .filter()   : Discards groups failing group-level condition.      │
├────────────────────────────────────────────────────────────────────────┤
│ 4. RESHAPING:                                                          │
│    - pivot_table(): Long to Wide (Aggregates values).                  │
│    - melt()       : Wide to Long (Unpivots columns into rows).         │
├────────────────────────────────────────────────────────────────────────┤
│ 5. CLEANING:                                                           │
│    - IQR Bounds   : [Q1 - 1.5*IQR, Q3 + 1.5*IQR]                       │
│    - Categoricals : df['col'].astype('category') saves up to 80% RAM.  │
│    - Datetime     : df['dt'].dt.month, df.resample('M').sum()          │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 10: 100-Mark Self-Assessment Examination

### Section A: Conceptual & API Nuances (25 Marks — 5 marks each)

**Q1:** What is the result of slicing a DataFrame with `.loc[1:3]` when the DataFrame's index contains integer labels `[1, 2, 3, 4, 5]`?  
(a) Rows at integer positions 1 and 2 (labels 2 and 3).  
(b) Rows with labels 1, 2, and 3.  
(c) Rows with labels 1 and 2 only.  
(d) `KeyError`.  

**Q2:** When merging two DataFrames with duplicate keys on both sides ($M$ matches on left, $N$ matches on right), how many rows does `pd.merge(how='inner')` generate for that key?  
(a) $M + N$  
(b) $\max(M, N)$  
(c) $M \times N$  
(d) 0  

**Q3:** Which method converts a wide DataFrame with columns `['ID', 'Q1_Sales', 'Q2_Sales']` into a long-form DataFrame?  
(a) `.pivot()`  
(b) `.melt()`  
(c) `.stack()`  
(d) `.explode()`  

**Q4:** What is the primary purpose of `validate='one_to_many'` in `pd.merge()`?  
(a) Enforces that the right table contains only unique keys.  
(b) Enforces that the left table contains only unique keys, raising an exception if duplicates exist.  
(c) Converts the output DataFrame to a one-dimensional Series.  
(d) Automatically fills null values with 0.  

**Q5:** Why does converting a low-cardinality string column to dtype `category` drastically reduce memory consumption?  
(a) Strings are compressed using gzip in memory.  
(b) Unique strings are stored once in a dictionary, and the column is replaced by small integer codes (e.g., `int8`).  
(c) All uppercase letters are converted to lowercase.  
(d) It drops missing values automatically.  

---

### Section B: Code Analysis & Output Tracing (25 Marks)

**Q6 (10 Marks):** Given the following DataFrame:
```python
df = pd.DataFrame({
    'dept': ['Sales', 'Sales', 'IT', 'IT'],
    'bonus': [1000, 2000, 3000, 4000]
})
df['avg_bonus'] = df.groupby('dept')['bonus'].transform('mean')
```
State the exact contents of the resulting DataFrame (columns: `dept`, `bonus`, `avg_bonus`).

**Q7 (15 Marks):** What does this NumPy broadcasting operation output?
```python
a = np.array([1, 2, 3]).reshape(3, 1)
b = np.array([10, 20])
c = a + b
```
State `c.shape` and write down the resulting 2D matrix.

---

### Section C: Implementation & Data Cleaning Challenges (30 Marks)

**Q8 (15 Marks):** Write a Python function `impute_missing_by_group(df: pd.DataFrame, group_col: str, target_col: str) -> pd.DataFrame` that fills missing values in `target_col` with the **median** of that column within each respective group in `group_col`.

**Q9 (15 Marks):** You are given an e-commerce DataFrame `orders` with columns `['customer_id', 'order_date', 'amount']`. Write a query in Pandas to find:
- Top 5 customers by total spending.
- For each of these 5 customers, calculate their 3-order rolling average spend ordered by `order_date`.

---

### Section D: Architectural & Performance Scenario (20 Marks)

**Q10 (20 Marks):** A data science pipeline reads a 12 GB CSV file on a machine with 16 GB RAM using `df = pd.read_csv('data.csv')`. The machine freezes due to RAM exhaustion.
1. Explain why a 12 GB CSV file expands to 25–35 GB in RAM when loaded into standard Pandas. (8 marks)
2. Detail three concrete techniques in Pandas to successfully process this dataset without running out of memory. (12 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** `.loc` is label-based and includes both endpoints (labels 1, 2, and 3).
- **Q2:** **(c)** Cartesian multiplication on matching keys: $M \times N$.
- **Q3:** **(b)** `.melt()`.
- **Q4:** **(b)** Asserts that the left table's merge keys are strictly unique.
- **Q5:** **(b)** Encodes unique strings as an integer categorical dictionary.

### Section B Solutions
**Q6 Output:**
| dept | bonus | avg_bonus |
| :--- | :--- | :--- |
| Sales | 1000 | 1500.0 |
| Sales | 2000 | 1500.0 |
| IT | 3000 | 3500.0 |
| IT | 4000 | 3500.0 |

**Q7 Output:**
- `c.shape`: `(3, 2)`
- Matrix:
  ```
  [[11, 21],
   [12, 22],
   [13, 23]]
  ```

### Section C Solutions
**Q8 Solution:**
```python
def impute_missing_by_group(df: pd.DataFrame, group_col: str, target_col: str) -> pd.DataFrame:
    df_copy = df.copy()
    group_medians = df_copy.groupby(group_col)[target_col].transform("median")
    df_copy[target_col] = df_copy[target_col].fillna(group_medians)
    return df_copy
```

**Q9 Solution:**
```python
# 1. Top 5 customers by spend
top5_custs = (
    orders.groupby("customer_id")["amount"]
    .sum()
    .nlargest(5)
    .index
)

# 2. Filter, sort, and calculate rolling 3-order average
top5_df = orders[orders["customer_id"].isin(top5_custs)].sort_values(["customer_id", "order_date"]).copy()
top5_df["rolling_3_avg"] = (
    top5_df.groupby("customer_id")["amount"]
    .rolling(window=3, min_periods=1)
    .mean()
    .reset_index(level=0, drop=True)
)
```

### Section D Solutions
**Q10 Solution:**
1. **Memory Inflation:** CSVs store raw text. In Pandas, strings default to Python object pointers (8 bytes per pointer + 50–80 bytes per boxed Python string), 64-bit numerical types are allocated by default (`int64`, `float64`), and BlockManager metadata causes $2\times$ to $3\times$ memory bloat.
2. **Remediation Techniques:**
   - **Chunking:** Process the file in streaming batches: `pd.read_csv('data.csv', chunksize=100_000)`.
   - **Column Pruning (`usecols`):** Load only necessary columns: `pd.read_csv(..., usecols=['id', 'date', 'amount'])`.
   - **Downcasting Dtypes:** Specify downcasted types: `dtype={'status': 'category', 'qty': 'int16', 'amount': 'float32'}`.
   - **Use Polars / PyArrow Engine:** `pd.read_csv(..., engine='pyarrow')` or Apache Arrow backed string types to eliminate string pointer boxing.
