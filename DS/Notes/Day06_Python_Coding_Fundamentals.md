# Day 6: Python Coding Fundamentals & Data Structures
## Memory Model, Mutability, Collections, Algorithmic Complexity & Pythonic Idioms


# Part 1: The Python Object Model, Memory & Mutability

### 1.1 Everything is an Object & Names are Labels
In Python, variables do **not** hold values directly; variables are **names (pointers/references) bound to objects in memory**.
Every Python object consists of at least three core fields in CPython:
1. **Type Pointer:** Indicates the object's class/type (e.g., `PyTypeObject`).
2. **Reference Count (`ob_refcnt`):** Tracks how many variable names or containers reference this object (used by CPython's garbage collector).
3. **Value / Data Payload:** The actual underlying data.

```python
a = [1, 2, 3]
b = a          # Both 'a' and 'b' reference the exact same memory address
print(id(a) == id(b))  # True
b.append(4)
print(a)               # [1, 2, 3, 4] -> Mutating through 'b' affects 'a'!
```

---

### 1.2 Identity (`is`) vs. Equality (`==`)
- **`==` (Value Equality):** Evaluates whether two objects contain the same data value (invokes the object's `__eq__()` magic method).
- **`is` (Reference Identity):** Evaluates whether two variables point to the **exact same memory address** (`id(a) == id(b)`).

```python
x = [1, 2, 3]
y = [1, 2, 3]
print(x == y)  # True (Identical values)
print(x is y)  # False (Distinct heap-allocated objects)

# Small Integer Interning: Python pre-allocates integers from -5 to 256
p = 100
q = 100
print(p is q)  # True (Both point to the cached integer block)

m = 1000
n = 1000
print(m is n)  # False (Exceeds intern cache; separate allocations in REPL)
```

> [!IMPORTANT]
> **Interview Rule:** Always use `is` when comparing against singleton objects like `None` (`if val is None:`), because `None` is a guaranteed unique singleton in Python memory, and `==` can be maliciously overridden by user-defined classes.

---

### 1.3 Mutability vs. Immutability
- **Immutable Types:** `int`, `float`, `bool`, `str`, `tuple`, `frozenset`, `bytes`. Once created, their memory contents can **never** be modified in place. Any "modification" allocates an entirely new object in memory.
- **Mutable Types:** `list`, `dict`, `set`, `bytearray`. Their memory contents can be modified in place without changing their memory address (`id()`).

```
Immutable Tuple with a Mutable Element (The Classic Trap):
t = (1, 2, [3, 4])
# t[0] = 99        --> TypeError: 'tuple' object does not support item assignment
t[2].append(5)     --> SUCCESS!
print(t)           --> (1, 2, [3, 4, 5])
```
*Why did this work?* The tuple's internal pointers are immutable (it permanently references index 0, index 1, and index 2). However, the object at index 2 is a mutable `list`, which can mutate internally while its identity remains unchanged!

---

### 1.4 Shallow Copy vs. Deep Copy

```python
import copy

original = [[1, 2, 3], [4, 5, 6]]

# 1. Assignment (Zero copy - shared pointer)
ref = original

# 2. Shallow Copy (Creates new outer container, but inner elements are shared references)
shallow = copy.copy(original) # or original.copy() or original[:]
shallow[0].append(99)
print(original[0])  # [1, 2, 3, 99] -> Modified!

# 3. Deep Copy (Recursively clones every nested container and object)
deep = copy.deepcopy(original)
deep[0].append(888)
print(original[0])  # [1, 2, 3, 99] -> Completely isolated!
```

---

### 1.5 The Mutable Default Argument Trap
In Python, default function arguments are evaluated **once at function definition time**, not every time the function is called.

```python
# THE ANTI-PATTERN (Buggy)
def append_to_list(val, target_list=[]):
    target_list.append(val)
    return target_list

print(append_to_list(1))  # [1]
print(append_to_list(2))  # [1, 2] -- The default list persisted across calls!

# THE PYTHONIC IDIOM (Correct)
def append_to_list_safe(val, target_list=None):
    if target_list is None:
        target_list = []
    target_list.append(val)
    return target_list
```

---

# Part 2: Core Data Structures & Algorithmic Complexities

### 2.1 Big-O Complexity Matrix for Python Built-ins

| Data Structure | Operation | Average Case | Worst Case | Internal Implementation |
| :--- | :--- | :--- | :--- | :--- |
| **`list`** | `append(x)` | $O(1)$ amortized | $O(n)$ (array resize) | Contiguous dynamic array of object pointers |
| | `insert(0, x)` / `pop(0)` | $O(n)$ | $O(n)$ | Requires shifting all $n$ subsequent pointers |
| | `pop()` (from end) | $O(1)$ | $O(1)$ | No memory shifting required |
| | `x in my_list` | $O(n)$ | $O(n)$ | Linear scan |
| | Slicing `lst[a:b]` | $O(k)$ where $k=b-a$ | $O(k)$ | Allocates new list and copies pointers |
| **`dict`** | Get `d[k]` / Set `d[k]=v` | $O(1)$ | $O(n)$ (hash collisions) | Compact hash table with open addressing |
| | Delete `del d[k]` | $O(1)$ | $O(n)$ | Marks dummy tombstone |
| | `k in d` (membership) | $O(1)$ | $O(n)$ | Hashes key, looks up index in table |
| **`set`** | Add `s.add(x)` / Remove | $O(1)$ | $O(n)$ | Hash table (dictionary with dummy values) |
| | Union `s1 \| s2` | $O(\text{len}(s1) + \text{len}(s2))$ | $O(n)$ | Combines hash sets |
| | Intersection `s1 & s2` | $O(\min(\text{len}(s1), \text{len}(s2)))$ | $O(n)$ | Traverses smaller set, checks hash in larger |
| **`deque`** | `appendleft(x)` / `popleft()`| $O(1)$ | $O(1)$ | Doubly linked list of fixed memory blocks |

---

### 2.2 Python Dictionaries Under the Hood
Since Python 3.7, dictionaries are guaranteed to maintain **insertion order**.
CPython achieves this using a **two-array compact architecture**:
1. **Indices Array:** A sparse array mapping hash buckets to indices in the entries array.
2. **Entries Array:** A dense array storing `[hash_code, key_pointer, value_pointer]` in the exact sequence they were inserted.
- **Space Reduction:** Reduces memory consumption by 30–40% compared to legacy sparse hash tables.
- **Key Requirement:** A dictionary key **must be hashable** (must implement `__hash__()` and `__eq__()`). Mutable objects (`list`, `dict`, `set`) are unhashable and will throw `TypeError: unhashable type`.

---

# Part 3: Comprehensions & Advanced Slicing

### 3.1 Python Slicing Internals: `sequence[start:stop:step]`
- **Indices:** 0-based. Negative indices count backward from the end (`-1` is the last element).
- **Omission Rules:**
  - `lst[:]` creates a shallow copy.
  - `lst[::-1]` reverses the sequence in $O(n)$ time.
  - `lst[::2]` extracts every second element.

### 3.2 List, Dict, and Set Comprehensions
Comprehensions are not merely syntactic sugar; they run at C-speed inside the CPython bytecode interpreter by avoiding repeated attribute lookups (`list.append`).

```python
# 1. Filtered and Transformed List Comprehension
squares = [x**2 for x in range(10) if x % 2 == 0]

# 2. Dictionary Comprehension (Inverting a map)
char_map = {'a': 1, 'b': 2, 'c': 3}
inv_map = {v: k for k, v in char_map.items()}

# 3. Set Comprehension
unique_lengths = {len(word) for word in ["apple", "banana", "fig", "apple"]}

# 4. Nested Comprehension (Flattening a 2D Matrix)
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flattened = [val for row in matrix for val in row]
# Equivalent to:
# for row in matrix:
#     for val in row:
#         flattened.append(val)
```

---

# Part 4: Functions, Variadic Arguments & Scope

### 4.1 Argument Unpacking: `*args` and `**kwargs`
- `*args`: Collects extra positional arguments into an immutable **tuple**.
- `**kwargs`: Collects extra keyword arguments into a mutable **dictionary**.

```python
def flexible_logger(level, *args, **kwargs):
    print(f"[{level}] Positional: {args}")
    print(f"[{level}] Metadata: {kwargs}")

flexible_logger("INFO", "User login", "IP: 127.0.0.1", user_id=42, latency_ms=12.5)
# args   -> ('User login', 'IP: 127.0.0.1')
# kwargs -> {'user_id': 42, 'latency_ms': 12.5}
```

#### Keyword-Only and Positional-Only Parameters:
```python
# '/' forces preceding arguments to be positional-only
# '*' forces following arguments to be keyword-only
def configure_engine(api_key, /, timeout=30, *, retries=3, debug=False):
    pass

# configure_engine(api_key="secret")   --> ERROR! api_key must be positional
configure_engine("secret", timeout=10, retries=5, debug=True) # Valid
```

---

### 4.2 The LEGB Scope Resolution Rule
When a variable name is referenced, Python searches namespaces in this strict order:
1. **L (Local):** Names assigned inside the current function.
2. **E (Enclosing):** Names in the local scope of any enclosing/outer functions (closures).
3. **G (Global):** Names assigned at the top-level of the module.
4. **B (Built-in):** Predefined Python names (`len`, `range`, `ValueError`).

```python
counter = 0  # Global

def outer():
    counter = 10  # Enclosing
    def inner():
        nonlocal counter  # Modifies enclosing scope
        counter += 1
        print("Inner:", counter)
    inner()
    print("Outer:", counter)

outer()
print("Global:", counter)
# Output:
# Inner: 11
# Outer: 11
# Global: 0
```

---

# Part 5: Python's Sorting Engine: Timsort

Python's built-in `sort()` (in-place) and `sorted()` (returns a new list) use **Timsort**, an adaptive, stable, hybrid sorting algorithm derived from Merge Sort and Insertion Sort:
- **Worst-case Time Complexity:** $O(n \log n)$
- **Best-case Time Complexity:** $O(n)$ (when input is already sorted or nearly sorted)
- **Space Complexity:** $O(n)$
- **Stability:** **Guaranteed**. If two records share identical sort keys, their original relative order is strictly preserved.

### Custom Multi-Criteria Sorting via Keys:
```python
students = [
    {"name": "Alice", "grade": 88, "age": 20},
    {"name": "Bob", "grade": 95, "age": 22},
    {"name": "Charlie", "grade": 88, "age": 19},
    {"name": "David", "grade": 95, "age": 20}
]

# Sort primarily by grade DESCENDING, then by age ASCENDING:
sorted_students = sorted(
    students, 
    key=lambda s: (-s["grade"], s["age"])
)
# Result: Bob (95, 22), David (95, 20), Charlie (88, 19), Alice (88, 20)
```

---

# Part 6: Production Implementation of Required Coding Tasks

### Task 1: Character & Word Frequency Counting
```python
from collections import Counter

def analyze_frequencies(text: str) -> dict:
    """Computes character frequency ignoring whitespace, and top words."""
    cleaned_chars = [c.lower() for c in text if c.isalnum()]
    char_counts = Counter(cleaned_chars)
    
    words = text.lower().split()
    word_counts = Counter(words)
    
    return {
        "char_frequencies": dict(char_counts),
        "most_common_char": char_counts.most_common(1)[0] if char_counts else None,
        "most_common_word": word_counts.most_common(1)[0] if word_counts else None
    }

# Complexity: O(N) Time, O(U) Space where U is unique characters/words.
```

---

### Task 2: Valid Anagrams (Two Approaches)
```python
def is_anagram_hashmap(s: str, t: str) -> bool:
    """Optimal O(N) time and O(1) space (at most 26 alphabetic keys)."""
    if len(s) != len(t):
        return False
    
    counts = {}
    for char in s:
        counts[char] = counts.get(char, 0) + 1
    for char in t:
        if char not in counts or counts[char] == 0:
            return False
        counts[char] -= 1
        
    return True

def is_anagram_sorting(s: str, t: str) -> bool:
    """O(N log N) time, O(N) space."""
    return sorted(s) == sorted(t)
```

---

### Task 3: Order-Preserving Duplicate Removal
```python
def remove_duplicates_ordered(seq: list) -> list:
    """
    Deduplicates a list while strictly maintaining initial element order.
    Uses dict.fromkeys() which runs in C-speed and preserves insertion order.
    """
    return list(dict.fromkeys(seq))

def remove_unhashable_duplicates(seq: list) -> list:
    """Handles lists containing unhashable objects (e.g. dicts)."""
    seen = []
    for item in seq:
        if item not in seen:
            seen.append(item)
    return seen  # Note: O(N^2) due to linear search in seen list
```

---

### Task 4: Two-Sum Problem
```python
def two_sum_optimal(nums: list[int], target: int) -> tuple[int, int] | None:
    """
    Finds the two indices whose values sum to target in O(N) time.
    Uses a hash map to store value -> index mapping.
    """
    seen = {}  # complement -> index
    for current_idx, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return (seen[complement], current_idx)
        seen[num] = current_idx
    return None

# Test: two_sum_optimal([2, 7, 11, 15], 9) -> (0, 1)
```

---

### Task 5: Matrix Transposition & Deep Transformations
```python
def transpose_matrix(matrix: list[list[int]]) -> list[list[int]]:
    """
    Transposes an M x N matrix into an N x M matrix using zip(*) idiom.
    """
    if not matrix or not matrix[0]:
        return []
    return [list(col) for col in zip(*matrix)]

# Example:
# matrix = [[1, 2, 3], [4, 5, 6]]
# transpose_matrix(matrix) -> [[1, 4], [2, 5], [3, 6]]
```

---

# Part 7: Core Comparisons Guide

### Comparison 1: `list` vs `tuple` vs `array.array`
| Feature | `list` | `tuple` | `array.array` |
| :--- | :--- | :--- | :--- |
| **Mutability** | Mutable | **Immutable** | Mutable |
| **Element Types** | Heterogeneous (Any object) | Heterogeneous (Any object) | **Strictly Homogeneous** (C data types) |
| **Memory Footprint** | Largest (Pointers + over-allocated buffer) | Lean (Exact size allocated) | Smallest (Raw unboxed C values) |
| **Dictionary Key?** | **No** (Unhashable) | **Yes** (If elements are immutable) | No |
| **Primary Use Case** | Dynamic general-purpose sequences | Fixed records, coordinates, dict keys | High-performance numeric data buffers |

---

# Part 8: Structured Technical Interview Question Bank

### Q1: What is the output of `[i for i in range(3)]` vs `(i for i in range(3))`?
**Answer:**
- `[i for i in range(3)]` is a **List Comprehension**: It immediately evaluates all elements and returns a fully materialized list in memory: `[0, 1, 2]`.
- `(i for i in range(3))` is a **Generator Expression**: It returns a lazy generator object `<generator object ...>` that produces items on-demand using the iterator protocol (`next()`), consuming $O(1)$ memory.

### Q2: Why is inserting at index 0 of a standard Python list slow ($O(n)$), and which data structure should you use instead?
**Answer:**
A Python `list` is internally represented as a contiguous dynamic array of object pointers. Inserting an element at index 0 requires shifting every existing pointer in the array one position to the right in memory, resulting in $O(n)$ time complexity. For workloads requiring frequent $O(1)$ insertions or deletions at both ends, `collections.deque` (a doubly-linked block array) should be used.

### Q3: How does Python manage memory for variables, and how does the Garbage Collector reclaim memory?
**Answer:**
Python manages memory through:
1. **Reference Counting:** Every object tracks how many references point to it (`ob_refcnt`). When the count reaches 0, its memory is immediately deallocated.
2. **Generational Cyclic Garbage Collector:** Reference counting cannot detect **reference cycles** (e.g., Object A references Object B, and Object B references Object A, while both are unreachable from root scopes). Python's cyclic GC organizes objects into three generations (Gen 0, 1, 2) based on survival age and periodically runs cycle-detection algorithms to detect and reclaim isolated cyclic islands.

### Q4: Explain the difference between `dict.get(key)` and `dict[key]`.
**Answer:**
- `dict[key]` performs a direct lookup and raises a `KeyError` exception if the key does not exist.
- `dict.get(key, default=None)` safely checks for the key; if present, it returns the value, and if absent, it returns the specified default value (`None` if omitted) without throwing an exception.

---

# Part 9: End-to-End Enterprise Case Study: Streaming Server Log Aggregator

### Problem Statement
A data engineering pipeline receives millions of raw Web Server HTTP access logs formatted as raw strings. You must write an in-memory aggregation module that:
1. Parses IP addresses, HTTP status codes, and response times.
2. Identifies top-3 attacking IPs (IPs with status code `401` or `403`).
3. Computes the average response time per unique endpoint, discarding corrupted records gracefully.

### Production Solution
```python
import re
from collections import defaultdict, Counter

LOG_PATTERN = re.compile(r'(?P<ip>\S+) \[(?P<endpoint>\S+)\] (?P<status>\d{3}) (?P<latency>\d+ms)')

raw_logs = [
    "192.168.1.10 [/api/v1/login] 401 120ms",
    "10.0.0.5 [/api/v1/users] 200 45ms",
    "192.168.1.10 [/api/v1/login] 401 95ms",
    "172.16.0.4 [/api/v1/checkout] 500 450ms",
    "192.168.1.10 [/api/v1/login] 403 110ms",
    "10.0.0.5 [/api/v1/users] 200 50ms",
    "CORRUPTED LOG LINE WITHOUT VALID SCHEMA",
    "10.0.0.99 [/api/v1/login] 401 80ms"
]

def parse_and_aggregate_logs(logs: list[str]) -> dict:
    endpoint_latencies = defaultdict(list)
    failed_auth_ips = Counter()
    corrupted_count = 0
    
    for line in logs:
        match = LOG_PATTERN.match(line)
        if not match:
            corrupted_count += 1
            continue
            
        data = match.groupdict()
        ip = data["ip"]
        endpoint = data["endpoint"]
        status = int(data["status"])
        latency = int(data["latency"].replace("ms", ""))
        
        # Accumulate metrics
        endpoint_latencies[endpoint].append(latency)
        if status in (401, 403):
            failed_auth_ips[ip] += 1
            
    # Calculate averages
    avg_latencies = {
        endpoint: round(sum(latencies) / len(latencies), 2)
        for endpoint, latencies in endpoint_latencies.items()
    }
    
    return {
        "top_suspicious_ips": failed_auth_ips.most_common(3),
        "endpoint_avg_latency_ms": avg_latencies,
        "corrupted_records_dropped": corrupted_count
    }

# Execute
results = parse_and_aggregate_logs(raw_logs)
# Output:
# 'top_suspicious_ips': [('192.168.1.10', 3), ('10.0.0.99', 1)]
# 'endpoint_avg_latency_ms': {'/api/v1/login': 101.25, '/api/v1/users': 47.5, '/api/v1/checkout': 450.0}
# 'corrupted_records_dropped': 1
```

---

# Part 10: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   PYTHON CODING FUNDAMENTALS CHEAT SHEET               │
├────────────────────────────────────────────────────────────────────────┤
│ 1. MUTABILITY:                                                         │
│    - Immutable: int, float, str, tuple, frozenset.                     │
│    - Mutable  : list, dict, set. (Unhashable; cannot be dict keys).    │
│ 2. IDENTITY vs EQUALITY:                                               │
│    - 'is' checks memory pointer address (id(a) == id(b)).              │
│    - '==' checks semantic value equality (__eq__).                     │
├────────────────────────────────────────────────────────────────────────┤
│                          TIME COMPLEXITIES                             │
│  - List: append O(1), pop() O(1), insert(0) O(N), search O(N).         │
│  - Dict/Set: lookup, insert, delete are all AVERAGE O(1).              │
│  - Timsort: O(N log N) worst case, O(N) best case (stable sort).       │
├────────────────────────────────────────────────────────────────────────┤
│                          ESSENTIAL IDIOMS                              │
│  - Safe Defaults: def f(x=None): if x is None: x = []                  │
│  - Reverse: seq[::-1]                                                  │
│  - Transpose: [list(c) for c in zip(*matrix)]                          │
│  - Deduplicate Preserving Order: list(dict.fromkeys(seq))              │
│  - Unpack: def f(*args, **kwargs)                                      │
├────────────────────────────────────────────────────────────────────────┤
│                           SCOPE (LEGB RULE)                            │
│  Local -> Enclosing -> Global -> Built-in.                             │
│  Use 'nonlocal' for enclosing, 'global' for module-level variables.    │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 11: 100-Mark Self-Assessment Examination

### Section A: Conceptual & Output Prediction (25 Marks — 5 marks each)

**Q1:** What is the output of the following code?
```python
a = [1, 2, 3]
b = a
b += [4]
print(a)
```
(a) `[1, 2, 3]`  
(b) `[1, 2, 3, 4]`  
(c) `TypeError`  
(d) `None`  

**Q2:** What happens when trying to use a tuple containing a list as a dictionary key?
```python
d = {}
k = (1, 2, [3, 4])
d[k] = "test"
```
(a) Succeeds; tuples are always hashable.  
(b) `TypeError: unhashable type: 'list'`  
(c) Key is inserted with value `"test"`.  
(d) List is automatically converted to a tuple.  

**Q3:** What is the average time complexity of checking membership (`x in collection`) for a Python `list` versus a Python `set` of $N$ elements?  
(a) List: $O(1)$, Set: $O(1)$  
(b) List: $O(n)$, Set: $O(n)$  
(c) List: $O(n)$, Set: $O(1)$  
(d) List: $O(\log n)$, Set: $O(1)$  

**Q4:** What is the output of this code snippet?
```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])
```
(a) `[0, 1, 2]`  
(b) `[2, 2, 2]`  
(c) `[0, 0, 0]`  
(d) `[None, None, None]`  

**Q5:** Which Python built-in module provides a double-ended queue with $O(1)$ append and pop operations at both ends?  
(a) `queue.Queue`  
(b) `collections.deque`  
(c) `itertools.cycle`  
(d) `heapq`  

---

### Section B: Algorithmic Complexity & Tracing (25 Marks)

**Q6 (10 Marks):** Analyze the exact time and space complexity of:
```python
def find_pairs(nums: list[int], k: int) -> int:
    nums_set = set(nums)
    count = 0
    for x in nums_set:
        if x + k in nums_set:
            count += 1
    return count
```
State the average and worst-case time complexity, and the auxiliary space complexity.

**Q7 (15 Marks):** Explain why `funcs = [lambda: i for i in range(3)]` outputs `[2, 2, 2]` (Late Binding of Closures). How do you fix the function definition so that it correctly outputs `[0, 1, 2]`?

---

### Section C: Code Implementation (30 Marks)

**Q8 (15 Marks):** Write a Python function `first_unique_char(s: str) -> int` that returns the 0-based index of the first non-repeating character in a string. If every character repeats, return `-1`. Your solution must run in strictly $O(n)$ time.

**Q9 (15 Marks):** Write a function `group_anagrams(words: list[str]) -> list[list[str]]` that groups anagrams together.  
*Example:* `["eat", "tea", "tan", "ate", "nat", "bat"]` $\rightarrow$ `[["eat", "tea", "ate"], ["tan", "nat"], ["bat"]]`. Explain the time complexity of your approach.

---

### Section D: Real-World Architecture & Performance (20 Marks)

**Q10 (20 Marks):** You are processing a 10 GB CSV dataset in Python on a machine with 8 GB of RAM.
1. Why does `data = open("data.csv").readlines()` cause an `OutOfMemoryError`? (6 marks)
2. How do Generator Functions (`yield`) resolve this problem? Write an idiomatic streaming generator that yields one cleaned row at a time. (14 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** `[1, 2, 3, 4]`. The in-place operator `+=` on lists invokes `__iadd__()`, which extends the existing list in-place rather than allocating a new object.
- **Q2:** **(b)** `TypeError: unhashable type: 'list'`. A tuple is only hashable if all of its constituent elements are hashable.
- **Q3:** **(c)** List: $O(n)$ (linear scan); Set: $O(1)$ (hash table lookup).
- **Q4:** **(b)** `[2, 2, 2]`. Closures bind to the variable name `i`, not the value at creation time. By the time the lambdas execute, `i` has completed the loop with value `2`.
- **Q5:** **(b)** `collections.deque`.

### Section B Solutions
**Q6 Solution:**
- Building `nums_set`: $O(n)$ time.
- Iterating over unique elements: $U$ iterations, each doing an $O(1)$ hash set lookup.
- **Average Time Complexity:** $O(n)$.
- **Worst-case Time Complexity:** $O(n^2)$ if severe hash collisions occur in the set.
- **Auxiliary Space Complexity:** $O(n)$ to store elements in `nums_set`.

**Q7 Solution:**
- Python closures look up variables in enclosing scopes at runtime (Late Binding). The variable `i` is looked up in the enclosing scope when `f()` is called, at which point the loop has finished and `i = 2`.
- **Fix (Default Argument Binding):**
  ```python
  funcs = [lambda x=i: x for i in range(3)]
  # Default arguments are evaluated at definition time!
  ```

### Section C Solutions
**Q8 Solution:**
```python
from collections import Counter

def first_unique_char(s: str) -> int:
    counts = Counter(s)  # O(N) time
    for idx, char in enumerate(s):  # O(N) time
        if counts[char] == 1:
            return idx
    return -1
```

**Q9 Solution:**
```python
from collections import defaultdict

def group_anagrams(words: list[str]) -> list[list[str]]:
    anagram_map = defaultdict(list)
    for word in words:
        # Tuple of character counts or sorted string as canonical key
        canonical_key = "".join(sorted(word))  # O(K log K)
        anagram_map[canonical_key].append(word)
    return list(anagram_map.values())
```
*Complexity:* $O(N \cdot K \log K)$ where $N$ is word count and $K$ is maximum word length.

### Section D Solutions
**Q10 Solution:**
1. `.readlines()` loads the entire 10 GB file into memory at once as a list of strings, exceeding the 8 GB system RAM and causing an immediate OOM crash.
2. **Streaming Generator Solution:**
   ```python
   def stream_large_csv(file_path: str):
       with open(file_path, "r", encoding="utf-8") as f:
           header = next(f) # Read header
           for line in f:   # Iterating over file object streams line by line (O(1) memory)
               yield line.strip().split(",")

   # Consuming stream with constant memory
   for record in stream_large_csv("data.csv"):
       process_record(record)
   ```

---
