# Day 7: Python OOP, Iterators, Generators, File I/O & Phase 1 Checkpoint
## Object-Oriented Architecture, Memory Streaming, Complexity Analysis & 15 Checkpoint Challenges


# Part 1: Object-Oriented Programming (OOP) in Python

### 1.1 The Four Pillars of OOP in Python

#### 1. Encapsulation: Attribute Protection & `@property`
Encapsulation bundles data and the methods that operate on that data within a single class, restricting direct outside modification.
- **Public:** `self.name` (Freely accessible everywhere).
- **Protected:** `self._balance` (Convention signaling internal use; accessible but discouraged).
- **Private:** `self.__secret` (Triggers **Name Mangling**: Python internally rewrites the attribute name to `_ClassName__secret` to prevent accidental subclass override).

```python
class BankAccount:
    def __init__(self, owner: str, initial_balance: float):
        self.owner = owner
        self._balance = initial_balance  # Protected

    @property
    def balance(self) -> float:
        """Getter property: Allows controlled read access."""
        return self._balance

    @balance.setter
    def balance(self, value: float):
        """Setter property: Enforces validation rules."""
        if value < 0:
            raise ValueError("Account balance cannot be negative.")
        self._balance = value
```

---

#### 2. Inheritance & Method Resolution Order (MRO)
Inheritance allows a subclass to inherit attributes and methods from one or more parent classes.

```python
class BaseDatabaseConnector:
    def __init__(self, connection_str: str):
        self.connection_str = connection_str

    def connect(self):
        print(f"Connecting via {self.connection_str}...")

class PostgresConnector(BaseDatabaseConnector):
    def __init__(self, connection_str: str, schema: str):
        # super() delegates to parent class __init__
        super().__init__(connection_str)
        self.schema = schema

    def connect(self):
        super().connect()
        print(f"Setting search_path to schema: {self.schema}")
```

#### Multiple Inheritance & The Diamond Problem (C3 Linearization)
When class `D` inherits from `B` and `C`, and both inherit from `A`:
```
      A
     / \
    B   C
     \ /
      D
```
Python resolves method calls using the **C3 Linearization Algorithm** (Method Resolution Order - MRO).
You can inspect any class's resolution sequence via `ClassName.mro()`:
```python
class A:
    def ping(self): print("A")
class B(A):
    def ping(self): print("B"); super().ping()
class C(A):
    def ping(self): print("C"); super().ping()
class D(B, C):
    def ping(self): print("D"); super().ping()

d = D()
d.ping()
# Output: D -> B -> C -> A
# D.mro() -> [D, B, C, A, object]
```

---

#### 3. Polymorphism & Duck Typing
"If it walks like a duck and quacks like a duck, it's a duck."
Python does not require formal interface inheritance for polymorphism; it relies on **Duck Typing**: any object that implements the required method signature can be used interchangeably.

#### Operator Overloading (Magic / Dunder Methods)
| Dunder Method | Operation | Semantic Meaning |
| :--- | :--- | :--- |
| `__init__(self, ...)` | Constructor | Initializes instance attributes |
| `__repr__(self)` | `repr(obj)` | Unambiguous official string representation for developers |
| `__str__(self)` | `str(obj)`, `print(obj)` | Readable, user-friendly string |
| `__eq__(self, other)` | `a == b` | Value equality check |
| `__lt__(self, other)` | `a < b` | Less-than comparison (enables `sort()`) |
| `__len__(self)` | `len(obj)` | Container element count |
| `__getitem__(self, k)`| `obj[k]` | Subscript indexing |

```python
class Vector2D:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

    def __add__(self, other: 'Vector2D') -> 'Vector2D':
        return Vector2D(self.x + other.x, self.y + other.y)

    def __repr__(self) -> str:
        return f"Vector2D({self.x}, {self.y})"

v1 = Vector2D(2, 3)
v2 = Vector2D(5, 7)
print(v1 + v2)  # Vector2D(7, 10)
```

---

#### 4. Abstraction (`abc.ABC` & `@abstractmethod`)
Abstract Base Classes define contracts that subclasses **must** implement before they can be instantiated.

```python
from abc import ABC, abstractmethod

class BaseMLModel(ABC):
    @abstractmethod
    def train(self, X, y):
        """Must be implemented by concrete subclasses."""
        pass

    @abstractmethod
    def predict(self, X):
        pass

class LinearRegressor(BaseMLModel):
    def train(self, X, y):
        print("Training linear regression coefficients...")
    def predict(self, X):
        return [0.0] * len(X)

# model = BaseMLModel() # TypeError: Can't instantiate abstract class BaseMLModel with abstract methods
```

---

### 1.2 Instance vs. Class vs. Static Methods

```python
class Employee:
    # Class Attribute (Shared across all instances)
    base_bonus_multiplier = 1.15

    def __init__(self, name: str, salary: float):
        self.name = name          # Instance attribute
        self.salary = salary      # Instance attribute

    # 1. Instance Method (Takes 'self', accesses instance and class state)
    def calculate_total_comp(self) -> float:
        return self.salary * self.base_bonus_multiplier

    # 2. Class Method (Takes 'cls', acts as an alternative factory constructor)
    @classmethod
    def from_string(cls, data_str: str):
        """Factory: creates an Employee from 'Name,Salary' string."""
        name, salary = data_str.split(",")
        return cls(name.strip(), float(salary.strip()))

    # 3. Static Method (Takes neither 'self' nor 'cls', pure utility function)
    @staticmethod
    def is_valid_salary(salary: float) -> bool:
        return salary >= 30000.0
```

---

# Part 2: Advanced Protocols: Iterators & Generators

### 2.1 The Iterator Protocol
An **Iterable** is an object capable of returning its members one at a time (implements `__iter__()`).
An **Iterator** is an object representing a stream of data that produces successive items via repeated calls to `__next__()`, raising `StopIteration` when depleted.

```python
class CountdownIterator:
    """Custom iterator that counts down from start to 1."""
    def __init__(self, start: int):
        self.current = start

    def __iter__(self):
        return self  # An iterator must return itself from __iter__

    def __next__(self) -> int:
        if self.current <= 0:
            raise StopIteration
        val = self.current
        self.current -= 1
        return val

for num in CountdownIterator(3):
    print(num)  # 3, 2, 1
```

---

### 2.2 Generators & The `yield` Keyword
A **Generator Function** uses `yield` instead of `return`. When called, it does not execute the code immediately; it returns a **generator object** that pauses and resumes execution state across calls, consuming $O(1)$ memory regardless of dataset size!

```python
import sys

def large_range_list(n: int):
    return [i for i in range(n)]

def large_range_gen(n: int):
    for i in range(n):
        yield i

# Memory Comparison for 1,000,000 items:
list_obj = large_range_list(1_000_000)
gen_obj = large_range_gen(1_000_000)

print("List memory:", sys.getsizeof(list_obj), "bytes")     # ~8,448,728 bytes (~8.4 MB)
print("Generator memory:", sys.getsizeof(gen_obj), "bytes") # ~200 bytes! (42,000x leaner)
```

#### Generator Pipelines (Composing Data Transformations):
```python
def read_lines(file_path):
    with open(file_path, "r") as f:
        for line in f:
            yield line.strip()

def filter_comments(lines):
    for line in lines:
        if not line.startswith("#") and line:
            yield line

def parse_csv(lines):
    for line in lines:
        yield line.split(",")

# Chained together lazily: Zero memory overhead
pipeline = parse_csv(filter_comments(read_lines("large_dataset.csv")))
```

---

# Part 3: Robust File I/O & JSON Serialization

### 3.1 Context Managers (`with` Statement)
Context managers guarantee that resources (file descriptors, database connections, locks) are safely released, even if exceptions occur.

```python
# Implementing a custom Context Manager via class
class TimedExecution:
    import time
    def __enter__(self):
        self.start = self.time.perf_counter()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.duration = self.time.perf_counter() - self.start
        print(f"Executed block in {self.duration:.4f} seconds.")
        return False  # Propagate exceptions if any
```

---

### 3.2 JSON Serialization & Custom Encoders
The `json` module natively serializes basic types (`dict`, `list`, `str`, `int`, `float`, `bool`, `None`). Complex types (`datetime`, `UUID`, custom class instances) throw `TypeError: Object of type X is not JSON serializable`.

```python
import json
from datetime import datetime

class UserEvent:
    def __init__(self, user_id: int, action: str, timestamp: datetime):
        self.user_id = user_id
        self.action = action
        self.timestamp = timestamp

class CustomJSONEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, UserEvent):
            return {
                "user_id": obj.user_id,
                "action": obj.action,
                "timestamp": obj.timestamp.isoformat()
            }
        return super().default(obj)

event = UserEvent(42, "PAYMENT_SUBMITTED", datetime.utcnow())
serialized_json = json.dumps(event, cls=CustomJSONEncoder, indent=2)
print(serialized_json)
```

---

# Part 4: Computational Complexity Masterclass (Big-O)

Understanding time and space complexity is non-negotiable for Data Science Engineer interviews.

```
Runtime Complexity Hierarchy (Fastest to Slowest):
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)
```

### 4.1 Detailed Breakdown with Data Engineering Examples

| Big-O Notation | Name | Practical Description | Canonical Data Science Example |
| :--- | :--- | :--- | :--- |
| **$O(1)$** | Constant | Runtime does not depend on dataset size $N$. | Hash map lookup (`dict[k]`), array index seek (`arr[0]`), stack push/pop. |
| **$O(\log n)$** | Logarithmic | Problem size is divided by a constant factor in each step. | Binary Search in sorted array, B-Tree index lookup. |
| **$O(n)$** | Linear | Examines each element a constant number of times. | Linear scan (`min()`, `max()`), single pass aggregation, Two-Sum hash map. |
| **$O(n \log n)$** | Linearithmic | Efficient divide-and-conquer sorting. | Timsort (`sorted()`), Merge Sort, QuickSort average case. |
| **$O(n^2)$** | Quadratic | Nested loops over the same dataset. | Brute force pair search, bubble sort, unindexed nested loop join. |
| **$O(2^n)$** | Exponential | Generating all subsets (Power Set). | Exhaustive combinatorial feature subset selection. |

---

# Part 5: Phase 1 Checkpoint Challenge: 15 Core Problems

Attempt these 15 problems independently before checking the solutions in Part 6!

### Part A: 10 Mixed SQL Interview Problems

**SQL Problem 1 (Aggregation & Grouping):**  
Write a query on `orders (order_id, customer_id, order_date, amount, status)` to find customers who placed at least 3 orders in 2024, where their total completed spend exceeds $1,000, and show their completed spend percentage relative to their total spend.

**SQL Problem 2 (Anti-Join):**  
Given `departments (dept_id, dept_name)` and `employees (emp_id, emp_name, dept_id)`. Find all departments that have **no employees assigned**, using two completely different SQL techniques (`LEFT JOIN` and `NOT EXISTS`).

**SQL Problem 3 (Self Join & Precedence):**  
Given `employee_salaries (emp_id, emp_name, dept_id, salary)`. Write a query to find all pairs of employees in the same department where Employee A earns at least $20,000 more than Employee B. Avoid duplicate inverted pairs.

**SQL Problem 4 (Second-Highest Salary):**  
Write an ANSI SQL query to find the 2nd highest distinct salary from `employees`. If fewer than 2 distinct salaries exist, return `NULL` (ensure scalar null return).

**SQL Problem 5 (Top-N per Group):**  
Given `products (product_id, category_id, product_name, price)`. Find the top 2 most expensive products in each category. If there is a tie for 2nd place, include all tied products.

**SQL Problem 6 (Window Cumulative Metrics):**  
Given `daily_revenue (revenue_date, daily_amount)`. Calculate:
1. The running cumulative revenue from the start of the year.
2. A 7-day rolling average of daily revenue (current day and prior 6 days).

**SQL Problem 7 (Consecutive Events / Gaps & Islands):**  
Given `user_logins (user_id, login_date)`. Identify all users who logged in for at least 3 consecutive calendar days.

**SQL Problem 8 (Conditional Pivot):**  
Given `hospital_visits (visit_id, doctor_id, patient_id, visit_type, fee)`. Where `visit_type` is in `('In-Person', 'Telehealth', 'Emergency')`. Pivot the data to display `doctor_id`, total visits, total in-person fees, total telehealth fees, and total emergency fees in a single row per doctor.

**SQL Problem 9 (Correlated Subquery):**  
Given `sales (sale_id, salesperson_id, amount, region)`. Find all sales transactions whose amount is strictly greater than the average transaction amount of that same region.

**SQL Problem 10 (Join Cardinality & Duplicates):**  
Table A has 4 rows with key `1`. Table B has 3 rows with key `1` and 2 rows with `NULL`. How many rows are returned by:
1. `INNER JOIN ON A.key = B.key`
2. `LEFT JOIN ON A.key = B.key`
3. `FULL OUTER JOIN ON A.key = B.key`

---

### Part B: 5 Production Python Coding Challenges

**Python Problem 1 (OOP Architecture):**  
Design a `Shape` class hierarchy with an abstract base class `Shape` having abstract methods `area()` and `perimeter()`. Implement concrete classes `Rectangle(width, height)` and `Circle(radius)`. Add operator overloading so that two shapes can be compared by area using `<` (`__lt__`).

**Python Problem 2 (Custom Streaming Iterator):**  
Write a custom iterator class `FibonacciIterator(max_count)` that produces the first `max_count` numbers of the Fibonacci sequence ($0, 1, 1, 2, 3, 5, \dots$) in $O(1)$ memory.

**Python Problem 3 (Binary Search):**  
Implement `binary_search(nums: list[int], target: int) -> int` that returns the index of `target` in a strictly sorted ascending array in $O(\log n)$ time and $O(1)$ space. Return `-1` if absent.

**Python Problem 4 (Frequency & Anagram Substring):**  
Write a function `find_anagram_indices(s: str, p: str) -> list[int]` that finds all start indices of `p`'s anagrams in `s` using a sliding window and hash map/counter in $O(|s|)$ time.

**Python Problem 5 (File Parsing & JSON Generator):**  
Write a generator function `stream_error_logs(file_path: str)` that opens a log file lazily, finds lines with status `[ERROR]`, and yields a parsed dictionary `{"timestamp": ..., "service": ..., "message": ...}` one line at a time.

---

# Part 6: Complete Solutions to Phase 1 Checkpoint

### Solutions to SQL Problems (1–10)

#### Solution 1:
```sql
SELECT 
    customer_id,
    COUNT(order_id) AS total_orders_2024,
    SUM(CASE WHEN status = 'Completed' THEN amount ELSE 0 END) AS completed_spend,
    ROUND(
        100.0 * SUM(CASE WHEN status = 'Completed' THEN amount ELSE 0 END) / SUM(amount),
        2
    ) AS completed_spend_pct
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2024
GROUP BY customer_id
HAVING COUNT(order_id) >= 3 
   AND SUM(CASE WHEN status = 'Completed' THEN amount ELSE 0 END) > 1000;
```

#### Solution 2:
```sql
-- Technique A: LEFT JOIN with Anti-Join predicate
SELECT d.dept_id, d.dept_name
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
WHERE e.emp_id IS NULL;

-- Technique B: NOT EXISTS (Short-circuit optimization)
SELECT d.dept_id, d.dept_name
FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id
);
```

#### Solution 3:
```sql
SELECT 
    e1.emp_name AS high_earner,
    e2.emp_name AS low_earner,
    e1.dept_id,
    e1.salary AS high_sal,
    e2.salary AS low_sal,
    (e1.salary - e2.salary) AS salary_difference
FROM employee_salaries e1
INNER JOIN employee_salaries e2 
    ON e1.dept_id = e2.dept_id 
    AND e1.salary >= e2.salary + 20000;
```

#### Solution 4:
```sql
WITH RankedSalaries AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT COALESCE(
    (SELECT DISTINCT salary FROM RankedSalaries WHERE rnk = 2), 
    NULL
) AS second_highest_salary;
```

#### Solution 5:
```sql
WITH RankedProducts AS (
    SELECT 
        category_id,
        product_name,
        price,
        DENSE_RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS price_rank
    FROM products
)
SELECT category_id, product_name, price, price_rank
FROM RankedProducts
WHERE price_rank <= 2
ORDER BY category_id, price_rank;
```

#### Solution 6:
```sql
SELECT 
    revenue_date,
    daily_amount,
    -- 1. Cumulative Running Total
    SUM(daily_amount) OVER (
        ORDER BY revenue_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_yearly_total,
    -- 2. 7-Day Rolling Moving Average
    ROUND(
        AVG(daily_amount) OVER (
            ORDER BY revenue_date 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ), 
        2
    ) AS rolling_7day_avg
FROM daily_revenue;
```

#### Solution 7:
```sql
WITH DistinctLogins AS (
    SELECT DISTINCT user_id, CAST(login_date AS DATE) AS login_date
    FROM user_logins
),
Sequenced AS (
    SELECT 
        user_id,
        login_date,
        login_date - CAST(ROW_NUMBER() OVER(PARTITION BY user_id ORDER BY login_date) AS INT) * INTERVAL '1 day' AS island_id
    FROM DistinctLogins
)
SELECT 
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS consecutive_active_days
FROM Sequenced
GROUP BY user_id, island_id
HAVING COUNT(*) >= 3;
```

#### Solution 8:
```sql
SELECT 
    doctor_id,
    COUNT(*) AS total_visits,
    SUM(CASE WHEN visit_type = 'In-Person' THEN fee ELSE 0 END) AS in_person_fees,
    SUM(CASE WHEN visit_type = 'Telehealth' THEN fee ELSE 0 END) AS telehealth_fees,
    SUM(CASE WHEN visit_type = 'Emergency' THEN fee ELSE 0 END) AS emergency_fees
FROM hospital_visits
GROUP BY doctor_id;
```

#### Solution 9:
```sql
SELECT s.sale_id, s.salesperson_id, s.region, s.amount
FROM sales s
WHERE s.amount > (
    SELECT AVG(sub.amount)
    FROM sales sub
    WHERE sub.region = s.region
);
```

#### Solution 10:
- **1. INNER JOIN:** $4 \times 3 = 12$ rows. (Keys with 1 match; NULLs do not match).
- **2. LEFT JOIN:** $4 \times 3 = 12$ rows. (All 4 rows from A find 3 matches in B).
- **3. FULL OUTER JOIN:** $12 + 2 = 14$ rows. (12 matched rows + 2 unmatched NULL rows from Table B).

---

### Solutions to Python Problems (1–5)

#### Solution 1 (OOP Shape Hierarchy):
```python
import math
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        pass

    @abstractmethod
    def perimeter(self) -> float:
        pass

    def __lt__(self, other: 'Shape') -> bool:
        if not isinstance(other, Shape):
            return NotImplemented
        return self.area() < other.area()

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

    def perimeter(self) -> float:
        return 2 * (self.width + self.height)

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self) -> float:
        return math.pi * (self.radius ** 2)

    def perimeter(self) -> float:
        return 2 * math.pi * self.radius

# Validation:
r = Rectangle(4, 5)  # Area = 20
c = Circle(3)        # Area = 28.27
assert r < c         # True!
```

#### Solution 2 (Custom Iterator):
```python
class FibonacciIterator:
    def __init__(self, max_count: int):
        self.max_count = max_count
        self.count = 0
        self.a, self.b = 0, 1

    def __iter__(self):
        return self

    def __next__(self) -> int:
        if self.count >= self.max_count:
            raise StopIteration
        val = self.a
        self.a, self.b = self.b, self.a + self.b
        self.count += 1
        return val

assert list(FibonacciIterator(6)) == [0, 1, 1, 2, 3, 5]
```

#### Solution 3 (Binary Search):
```python
def binary_search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1

assert binary_search([1, 3, 5, 7, 9, 11], 7) == 3
assert binary_search([1, 3, 5, 7, 9, 11], 4) == -1
```

#### Solution 4 (Sliding Window Anagrams):
```python
from collections import Counter

def find_anagram_indices(s: str, p: str) -> list[int]:
    p_len, s_len = len(p), len(s)
    if p_len > s_len:
        return []

    p_count = Counter(p)
    s_count = Counter(s[:p_len])
    result = []

    if s_count == p_count:
        result.append(0)

    for i in range(p_len, s_len):
        # Slide window: Add right char, remove left char
        s_count[s[i]] += 1
        left_char = s[i - p_len]
        s_count[left_char] -= 1
        if s_count[left_char] == 0:
            del s_count[left_char]

        if s_count == p_count:
            result.append(i - p_len + 1)

    return result

assert find_anagram_indices("cbaebabacd", "abc") == [0, 6]
```

#### Solution 5 (File Parsing Generator):
```python
import re

LOG_REGEX = re.compile(r'\[(?P<timestamp>[^\]]+)\] \[(?P<level>[A-Z]+)\] \[(?P<service>[^\]]+)\] (?P<message>.*)')

def stream_error_logs(file_path: str):
    with open(file_path, "r", encoding="utf-8") as f:
        for line in f:
            if "[ERROR]" in line:
                match = LOG_REGEX.match(line.strip())
                if match:
                    data = match.groupdict()
                    yield {
                        "timestamp": data["timestamp"],
                        "service": data["service"],
                        "message": data["message"]
                    }
```

---

# Part 7: Enterprise Case Study: Extensible Plugin-Based ETL Framework

### Architectural Problem Statement
At **DataSphere Analytics**, daily batch pipelines ingested files from multiple heterogeneous sources (Local CSVs, S3 Buckets, REST APIs). Junior developers wrote monolithic scripts with tangled file-handling, parsing, and transformation code.

You must design an extensible, object-oriented ETL framework where:
1. `BaseExtractor`, `BaseTransformer`, and `BaseLoader` are abstract contracts.
2. Data streams lazily as generator records between phases to handle arbitrary file sizes with $O(1)$ memory.
3. Concrete implementations can be plugged in without modifying the core pipeline driver.

### Production Solution
```python
from abc import ABC, abstractmethod
from typing import Generator, Any

class BaseExtractor(ABC):
    @abstractmethod
    def extract(self) -> Generator[dict, None, None]:
        pass

class BaseTransformer(ABC):
    @abstractmethod
    def transform(self, records: Generator[dict, None, None]) -> Generator[dict, None, None]:
        pass

class BaseLoader(ABC):
    @abstractmethod
    def load(self, records: Generator[dict, None, None]) -> int:
        pass

# Concrete Extractor: Simulating CSV Streaming
class MockCSVExtractor(BaseExtractor):
    def __init__(self, data: list[dict]):
        self.data = data

    def extract(self) -> Generator[dict, None, None]:
        for row in self.data:
            yield row

# Concrete Transformer: Cleaning & Normalizing Data
class CleanCurrencyTransformer(BaseTransformer):
    def transform(self, records: Generator[dict, None, None]) -> Generator[dict, None, None]:
        for row in records:
            # Transformation logic
            cleaned_amount = float(str(row["amount"]).replace("$", "").strip())
            yield {
                "txn_id": row["txn_id"],
                "clean_amount": round(cleaned_amount, 2),
                "is_flagged": cleaned_amount > 10000.0
            }

# Concrete Loader: Target Stager
class ConsoleLoader(BaseLoader):
    def load(self, records: Generator[dict, None, None]) -> int:
        count = 0
        for record in records:
            print(f"Loading record into warehouse: {record}")
            count += 1
        return count

# Pipeline Orchestrator Driver
class ETLPipeline:
    def __init__(self, extractor: BaseExtractor, transformer: BaseTransformer, loader: BaseLoader):
        self.extractor = extractor
        self.transformer = transformer
        self.loader = loader

    def run(self) -> int:
        raw_stream = self.extractor.extract()
        transformed_stream = self.transformer.transform(raw_stream)
        loaded_count = self.loader.load(transformed_stream)
        return loaded_count

# Execution Test
sample_raw_data = [
    {"txn_id": "T1", "amount": "$450.50"},
    {"txn_id": "T2", "amount": "$12500.00"},
    {"txn_id": "T3", "amount": "$89.99"}
]

pipeline = ETLPipeline(
    extractor=MockCSVExtractor(sample_raw_data),
    transformer=CleanCurrencyTransformer(),
    loader=ConsoleLoader()
)

records_processed = pipeline.run()
print(f"Pipeline successfully completed. Loaded {records_processed} rows.")
```

---

# Part 8: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                      PYTHON OOP & COMPLEXITY CHEAT SHEET               │
├────────────────────────────────────────────────────────────────────────┤
│ 1. 4 PILLARS OF OOP:                                                   │
│    - Encapsulation: Use @property for validated getters/setters.       │
│    - Inheritance  : super().__init__(). C3 MRO diamond resolution.     │
│    - Polymorphism : Duck typing & Dunder methods (__add__, __lt__).    │
│    - Abstraction  : abc.ABC + @abstractmethod contracts.               │
├────────────────────────────────────────────────────────────────────────┤
│ 2. METHOD TYPES:                                                       │
│    - Instance Method: (self) -> Acts on instance state.                │
│    - Class Method   : @classmethod (cls) -> Factory constructors.      │
│    - Static Method  : @staticmethod () -> Isolated utility functions.  │
├────────────────────────────────────────────────────────────────────────┤
│ 3. STREAMING & ITERATION:                                              │
│    - Iterator : Implements __iter__() and __next__(); raises           │
│                 StopIteration.                                         │
│    - Generator: Uses 'yield'. Pauses state. O(1) memory footprint.     │
├────────────────────────────────────────────────────────────────────────┤
│ 4. COMPUTATIONAL COMPLEXITY (BIG-O):                                   │
│    - O(1)     : Dict/Set lookup, array index.                          │
│    - O(log n) : Binary Search.                                         │
│    - O(n)     : Linear search, Two-Sum hash map.                       │
│    - O(n log n): Timsort, Merge Sort.                                  │
│    - O(n²)    : Nested loops, unindexed nested loop join.              │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 9: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (20 Marks — 4 marks each)

**Q1:** What is the Method Resolution Order (MRO) in Python?  
(a) The order in which module imports are parsed at startup.  
(b) The algorithmic sequence (C3 Linearization) Python uses to look up attributes and methods in class hierarchies.  
(c) The sequence in which variables are cleared by the garbage collector.  
(d) The order of execution of functions in a thread pool.  

**Q2:** What happens when an instantiated class attempts to call an `@abstractmethod` that was not overridden by its subclass?  
(a) A warning is printed and the method returns `None`.  
(b) Python raises `TypeError: Can't instantiate abstract class with abstract methods`.  
(c) The method calls the parent object's base implementation automatically.  
(d) The Python interpreter crashes.  

**Q3:** What is the space complexity of a generator that yields numbers from 1 to 100 million?  
(a) $O(n)$  
(b) $O(\log n)$  
(c) $O(1)$  
(d) $O(n \log n)$  

**Q4:** In Python name mangling, how is a private attribute `__salary` defined inside class `Employee` stored internally?  
(a) `__salary`  
(b) `_salary`  
(c) `_Employee__salary`  
(d) `Employee.__salary`  

**Q5:** What is the worst-case time complexity of Python's built-in `list.sort()`?  
(a) $O(n^2)$  
(b) $O(n \log n)$  
(c) $O(n)$  
(d) $O(\log n)$  

---

### Section B: Code Analysis & Output Tracing (25 Marks)

**Q6 (10 Marks):** Trace and state the exact output of:
```python
class Parent:
    def greet(self):
        print("Hello from Parent")

class Child(Parent):
    def greet(self):
        super().greet()
        print("Hello from Child")

c = Child()
c.greet()
```

**Q7 (15 Marks):** What does this generator produce, and what happens on the 4th call to `next()`?
```python
def gen():
    yield 10
    yield 20
    yield 30

g = gen()
print(next(g))
print(next(g))
print(next(g))
print(next(g))
```

---

### Section C: Implementation Challenges (35 Marks)

**Q8 (15 Marks):** Write a Python class `Stack` implementing a LIFO stack with methods `push(item)`, `pop() -> item`, and `peek() -> item`. All operations must run in strictly $O(1)$ amortized time. Raise an `IndexError` when popping from an empty stack.

**Q9 (20 Marks):** Write a Python generator function `sliding_window(seq: list, window_size: int)` that takes a list and yields successive overlapping windows of size `window_size` as tuples.  
*Example:* `sliding_window([1, 2, 3, 4, 5], 3)` yields `(1, 2, 3)`, `(2, 3, 4)`, `(3, 4, 5)`.

---

### Section D: Architectural & Complexity Scenario (20 Marks)

**Q10 (20 Marks):** You are hired to optimize a Python batch process that compares 100,000 incoming customer transaction records against 500,000 existing fraudulent account IDs.
1. The original code uses:
   ```python
   flagged = [txn for txn in transactions if txn["account_id"] in fraud_list]
   # where fraud_list is a Python list of integers.
   ```
   Calculate the exact Big-O time complexity of this code and explain why it takes over 15 minutes to run. (10 marks)
2. Rewrite the code to complete in under 1 second. Explain the data structure change and analyze the new time and space complexity. (10 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** C3 Linearization order for attribute lookup.
- **Q2:** **(b)** Python prevents class instantiation and throws `TypeError`.
- **Q3:** **(c)** $O(1)$ memory; values are generated lazily one at a time.
- **Q4:** **(c)** `_Employee__salary`.
- **Q5:** **(b)** $O(n \log n)$ (Timsort guarantee).

### Section B Solutions
**Q6 Output:**
```
Hello from Parent
Hello from Child
```

**Q7 Output:**
```
10
20
30
StopIteration Exception is raised on the 4th next() call!
```

### Section C Solutions
**Q8 Solution:**
```python
class Stack:
    def __init__(self):
        self._items = []

    def push(self, item):
        self._items.append(item)  # O(1) amortized

    def pop(self):
        if not self._items:
            raise IndexError("pop from empty stack")
        return self._items.pop()   # O(1) from end

    def peek(self):
        if not self._items:
            raise IndexError("peek from empty stack")
        return self._items[-1]     # O(1)
```

**Q9 Solution:**
```python
def sliding_window(seq: list, window_size: int):
    if window_size <= 0 or window_size > len(seq):
        return
    for i in range(len(seq) - window_size + 1):
        yield tuple(seq[i:i + window_size])
```

### Section D Solutions
**Q10 Solution:**
1. **Complexity:** Checking `txn["account_id"] in fraud_list` performs a linear scan $O(M)$ over the 500,000 elements in `fraud_list`. With $N = 100,000$ transactions, the total runtime is $O(N \times M) = 100,000 \times 500,000 = 5 \times 10^{10}$ operations, taking 15–20 minutes.
2. **Optimized Solution:**
   ```python
   # Convert list to hash set upfront: O(M) time, O(M) space
   fraud_set = set(fraud_list)
   
   # Set membership lookup is O(1) average:
   flagged = [txn for txn in transactions if txn["account_id"] in fraud_set]
   ```
   - **New Time Complexity:** $O(M + N) \approx 600,000$ operations. Executes in $\approx 50$ milliseconds (over 20,000x speedup!).
   - **Space Complexity:** $O(M)$ to store the hash set in memory.

---

