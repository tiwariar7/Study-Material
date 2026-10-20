# Day 17: Python Algorithmic Coding & Data Transformation
## Hash Maps, Two Pointers, Sliding Window, Binary Search, Prefix Sums & Pandas Transformations

---

# Part 1: Algorithmic Pattern Blueprints

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CORE ALGORITHMIC BLUEPRINTS                     │
├────────────────────────────────────────────────────────────────────────┤
│ 1. TWO POINTERS (Sorted Sequences)                                     │
│    - Left at 0, Right at N-1. Move inwards based on target comparison. │
│    - Eliminates O(N²) nested loops down to O(N) time and O(1) space.   │
├────────────────────────────────────────────────────────────────────────┤
│ 2. SLIDING WINDOW (Contiguous Subarrays / Substrings)                  │
│    - Right pointer expands window to satisfy condition.                │
│    - Left pointer contracts window to maintain validity / minimize.   │
│    - Time: O(N) because each pointer visits each element at most once. │
├────────────────────────────────────────────────────────────────────────┤
│ 3. PREFIX SUM + HASH MAP (Subarray Sums)                               │
│    - Running sum S_j. Subarray sum between i and j is S_j - S_{i-1}.   │
│    - If (current_sum - target) is in seen hash map, valid subarray!    │
├────────────────────────────────────────────────────────────────────────┤
│ 4. MODIFIED BINARY SEARCH (Rotated / Boundary Search)                  │
│    - Midpoint splits array into at least one strictly sorted half.     │
│    - Check if target falls within the sorted half; adjust boundaries.  │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 2: The 8 Production-Grade Coding Problems

---

### Problem 1: Two Sum II — Input Array Is Sorted
**Problem Statement:** Given a 1-indexed array of integers `numbers` that is already sorted in non-decreasing order, find two numbers such that they add up to a specific `target` number. Return the indices of the two numbers added by one.

**Constraints & Edge Cases:** Must use strictly $O(1)$ auxiliary space. Array may contain negative numbers.

```python
def two_sum_sorted(numbers: list[int], target: int) -> list[int]:
    """
    Two Pointers Pattern.
    Time Complexity: O(N) - single pass.
    Space Complexity: O(1) - constant auxiliary memory.
    """
    left = 0
    right = len(numbers) - 1

    while left < right:
        current_sum = numbers[left] + numbers[right]
        if current_sum == target:
            return [left + 1, right + 1]  # 1-indexed requirement
        elif current_sum < target:
            left += 1   # Need larger sum
        else:
            right -= 1  # Need smaller sum

    return []

# Test assertions:
assert two_sum_sorted([2, 7, 11, 15], 9) == [1, 2]
assert two_sum_sorted([-3, 3, 4, 90], 0) == [1, 2]
```

---

### Problem 2: Longest Substring Without Repeating Characters
**Problem Statement:** Given a string `s`, find the length of the longest substring without duplicate characters.

**Constraints & Edge Cases:** Empty string, all identical characters (`"bbbbb"`), alternating strings.

```python
def length_of_longest_substring(s: str) -> int:
    """
    Dynamic Sliding Window with Hash Map (Last Seen Index).
    Time Complexity: O(N) - string traversed once.
    Space Complexity: O(min(N, M)) where M is character alphabet size.
    """
    last_seen = {}
    max_len = 0
    left = 0

    for right, char in enumerate(s):
        # If char seen and inside current window, jump left pointer past it
        if char in last_seen and last_seen[char] >= left:
            left = last_seen[char] + 1

        last_seen[char] = right
        max_len = max(max_len, right - left + 1)

    return max_len

# Test assertions:
assert length_of_longest_substring("abcabcbb") == 3  # "abc"
assert length_of_longest_substring("bbbbb") == 1     # "b"
assert length_of_longest_substring("pwwkew") == 3    # "wke"
assert length_of_longest_substring("") == 0
```

---

### Problem 3: Group Anagrams
**Problem Statement:** Given an array of strings `strs`, group the anagrams together. You can return the answer in any order.

**Constraints & Edge Cases:** Empty array, single characters, strings with different lengths.

```python
from collections import defaultdict

def group_anagrams(strs: list[str]) -> list[list[str]]:
    """
    Hash Map with 26-element character count tuple as canonical key.
    Time Complexity: O(N * K) where N is number of strings, K is max string length.
    Space Complexity: O(N * K).
    """
    anagram_map = defaultdict(list)

    for word in strs:
        # Build frequency array of size 26
        count = [0] * 26
        for char in word:
            count[ord(char) - ord('a')] += 1
            
        # Tuple of counts is hashable and serves as canonical key
        canonical_key = tuple(count)
        anagram_map[canonical_key].append(word)

    return list(anagram_map.values())

# Test assertions:
res = group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"])
assert len(res) == 3
```

---

### Problem 4: Container With Most Water
**Problem Statement:** Given an integer array `height` of length $n$, where each element represents a vertical line at coordinate $(i, \text{height}[i])$. Find two lines that together with the x-axis form a container that holds the most water. Return the maximum amount of water a container can store.

**Constraints & Edge Cases:** $n \ge 2$, strictly $O(1)$ space required.

```python
def max_area(height: list[int]) -> int:
    """
    Two Pointers (Greedy Inward Contraction).
    Area = (right - left) * min(height[left], height[right])
    Always shift the shorter line inward because the width decreases,
    so only a taller line can increase total area.
    
    Time Complexity: O(N).
    Space Complexity: O(1).
    """
    left = 0
    right = len(height) - 1
    max_water = 0

    while left < right:
        width = right - left
        h = min(height[left], height[right])
        max_water = max(max_water, width * h)

        if height[left] < height[right]:
            left += 1
        else:
            right -= 1

    return max_water

# Test assertions:
assert max_area([1, 8, 6, 2, 5, 4, 8, 3, 7]) == 49
assert max_area([1, 1]) == 1
```

---

### Problem 5: Subarray Sum Equals K
**Problem Statement:** Given an array of integers `nums` and an integer `k`, return the total number of continuous subarrays whose sum equals `k`.

**Constraints & Edge Cases:** Array may contain negative numbers (which breaks standard two-pointer sliding window!).

```python
from collections import defaultdict

def subarray_sum(nums: list[int], k: int) -> int:
    """
    Prefix Sum + Hash Map.
    S_j - S_i = k  ===> S_i = S_j - k.
    
    Time Complexity: O(N).
    Space Complexity: O(N).
    """
    prefix_counts = defaultdict(int)
    prefix_counts[0] = 1  # Base case: empty prefix has sum 0
    
    running_sum = 0
    count = 0

    for num in nums:
        running_sum += num
        target_complement = running_sum - k
        
        # If running_sum - k was seen previously, add its frequency
        if target_complement in prefix_counts:
            count += prefix_counts[target_complement]

        prefix_counts[running_sum] += 1

    return count

# Test assertions:
assert subarray_sum([1, 1, 1], 2) == 2
assert subarray_sum([1, -1, 0], 0) == 3
```

---

### Problem 6: Search in Rotated Sorted Array
**Problem Statement:** An integer array `nums` sorted in ascending order is rotated at an unknown pivot index. Given `nums` and a `target`, return the index of `target` if it is in `nums`, or `-1` if it is not in `nums`. Your algorithm must run in strictly $O(\log n)$ runtime.

**Constraints & Edge Cases:** All elements are unique.

```python
def search_rotated_array(nums: list[int], target: int) -> int:
    """
    Modified Binary Search.
    Time Complexity: O(log N).
    Space Complexity: O(1).
    """
    left = 0
    right = len(nums) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if nums[mid] == target:
            return mid

        # Check if Left Half is Sorted
        if nums[left] <= nums[mid]:
            # Check if target lies within the sorted left half
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # Otherwise, Right Half is Sorted
        else:
            # Check if target lies within the sorted right half
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1

# Test assertions:
assert search_rotated_array([4, 5, 6, 7, 0, 1, 2], 0) == 4
assert search_rotated_array([4, 5, 6, 7, 0, 1, 2], 3) == -1
```

---

### Problem 7: Top K Frequent Elements
**Problem Statement:** Given an integer array `nums` and an integer `k`, return the $k$ most frequent elements. You may return the answer in any order. The solution must be better than $O(n \log n)$ time.

```python
from collections import Counter

def top_k_frequent(nums: list[int], k: int) -> list[int]:
    """
    Bucket Sort Algorithm.
    Index of bucket represents frequency (from 0 to N).
    Time Complexity: O(N) linear time.
    Space Complexity: O(N).
    """
    counts = Counter(nums)
    n = len(nums)
    
    # Buckets where index = frequency, value = list of elements with that frequency
    buckets = [[] for _ in range(n + 1)]
    for num, freq in counts.items():
        buckets[freq].append(num)

    result = []
    # Traverse buckets in reverse order (highest frequency first)
    for freq in range(n, 0, -1):
        for num in buckets[freq]:
            result.append(num)
            if len(result) == k:
                return result

    return result

# Test assertions:
assert sorted(top_k_frequent([1, 1, 1, 2, 2, 3], 2)) == [1, 2]
assert top_k_frequent([1], 1) == [1]
```

---

### Problem 8: Advanced Python & Pandas Data Transformation Challenge
**Problem Statement:** Given a messy sales transaction dataset with semi-structured JSON line items, write a Pandas transformation function that:
1. Explodes the line items into individual transaction rows.
2. Extracts item metadata (sku, quantity, unit price).
3. Pivots the data into an executive summary showing Total Revenue and Average Quantity by Product Category across Sales Regions.

```python
import pandas as pd
import json

raw_transactions = pd.DataFrame({
    "order_id": [101, 102],
    "region": ["West", "East"],
    "items_json": [
        json.dumps([
            {"sku": "SKU_A", "category": "Tech", "qty": 2, "price": 500.0},
            {"sku": "SKU_B", "category": "Office", "qty": 5, "price": 20.0}
        ]),
        json.dumps([
            {"sku": "SKU_C", "category": "Tech", "qty": 1, "price": 1200.0}
        ])
    ]
})

def transform_nested_sales(df: pd.DataFrame) -> pd.DataFrame:
    """Production Pandas pipeline handling JSON parsing, exploding, and pivoting."""
    # 1. Parse JSON strings into Python lists
    df["items_parsed"] = df["items_json"].apply(json.loads)
    
    # 2. Explode nested lists into individual rows
    df_exploded = df.explode("items_parsed").reset_index(drop=True)
    
    # 3. Extract JSON keys into individual columns
    items_df = pd.json_normalize(df_exploded["items_parsed"])
    
    # 4. Join back with parent order attributes
    cleaned_df = pd.concat([df_exploded[["order_id", "region"]], items_df], axis=1)
    
    # 5. Compute gross sales
    cleaned_df["line_total"] = cleaned_df["qty"] * cleaned_df["price"]
    
    # 6. Pivot Table Summary
    summary_pivot = cleaned_df.pivot_table(
        index="region",
        columns="category",
        values=["line_total", "qty"],
        aggfunc={"line_total": "sum", "qty": "mean"},
        fill_value=0
    )
    
    return summary_pivot

# Execute transformation:
pivot_result = transform_nested_sales(raw_transactions)
print(pivot_result)
```

---

# Part 3: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   PYTHON CODING INTERVIEW CHEAT SHEET                  │
├────────────────────────────────────────────────────────────────────────┤
│ 1. TWO POINTERS:                                                       │
│    - Sorted Array: left = 0, right = N-1. Check sum vs target. O(N).   │
│    - Container with Water: Always move the shorter height pointer!     │
├────────────────────────────────────────────────────────────────────────┤
│ 2. SLIDING WINDOW:                                                     │
│    - Longest substring without repeats: Track last_seen[char].         │
│      left = max(left, last_seen[char] + 1). O(N) time.                 │
├────────────────────────────────────────────────────────────────────────┤
│ 3. PREFIX SUMS:                                                        │
│    - Subarray Sum = K: Running sum - K in prefix_counts map.           │
│      Handles negative numbers seamlessly where two-pointers fail!      │
├────────────────────────────────────────────────────────────────────────┤
│ 4. MODIFIED BINARY SEARCH:                                             │
│    - Rotated sorted array: At least one half is ALWAYS sorted.         │
│      nums[left] <= nums[mid] -> Left half is sorted; check bounds!     │
├────────────────────────────────────────────────────────────────────────┤
│ 5. TOP-K FREQUENT:                                                     │
│    - Bucket sort achieves O(N) linear time using frequency as index.   │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 4: 100-Mark Self-Assessment Examination

### Section A: Algorithmic Complexity & Tracing (25 Marks — 5 marks each)

**Q1:** What is the worst-case time complexity of checking if a target exists in a Hash Map (Python dictionary) of $N$ items?  
(a) $O(1)$  
(b) $O(\log n)$  
(c) $O(n)$ (when all keys collide into the same bucket)  
(d) $O(n^2)$  

**Q2:** Why can the standard Two-Pointer sliding window technique NOT be used to find subarrays that sum to $K$ when the array contains negative numbers?  
(a) Negative numbers cause integer overflow.  
(b) The monotonicity property is broken: expanding the right window no longer guarantees the sum increases, and contracting the left no longer guarantees it decreases.  
(c) Python lists do not support negative indexing.  
(d) Two pointers only works on strings.  

**Q3:** What is the average time complexity of finding the Top $K$ frequent elements in an array of $N$ numbers using Bucket Sort?  
(a) $O(n \log n)$  
(b) $O(n \log k)$  
(c) $O(n)$  
(d) $O(k)$  

**Q4:** In a binary search algorithm, what bug does the formula `mid = left + (right - left) // 2` prevent compared to `mid = (left + right) // 2`?  
(a) Zero division error.  
(b) 32-bit integer overflow in languages like C++/Java when `left + right` exceeds $2^{31}-1$.  
(c) Floating point rounding error.  
(d) Syntax error in Python.  

**Q5:** What is the space complexity of `list(set(nums))` used to deduplicate a list of $N$ integers?  
(a) $O(1)$  
(b) $O(n)$  
(c) $O(\log n)$  
(d) $O(n^2)$  

---

### Section B: Problem Implementation (50 Marks)

**Q6 (25 Marks - Minimum Size Subarray Sum):**  
Given an array of positive integers `nums` and a positive integer `target`, return the minimal length of a contiguous subarray of which the sum is greater than or equal to `target`. If there is no such subarray, return `0`. Your solution must run in strictly $O(n)$ time using the **Sliding Window** pattern.

**Q7 (25 Marks - 3Sum):**  
Given an integer array `nums`, return all the triplets `[nums[i], nums[j], nums[k]]` such that $i \neq j$, $i \neq k$, and $j \neq k$, and `nums[i] + nums[j] + nums[k] == 0`. Notice that the solution set must not contain duplicate triplets. Your solution must run in $O(n^2)$ time.

---

### Section C: Live Coding Optimization Scenario (25 Marks)

**Q8 (25 Marks):** A candidate writes the following code to find the first non-repeating character in a 10-million character string:
```python
def first_unique(s: str) -> str:
    for char in s:
        if s.count(char) == 1:
            return char
    return ""
```
1. Calculate the exact Big-O time complexity of this code and explain why it hangs in production. (10 marks)
2. Rewrite the function to run in strictly $O(n)$ time and $O(1)$ auxiliary space (assuming standard ASCII/lowercase English alphabet). (15 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(c)** $O(n)$ during severe hash collision degradation.
- **Q2:** **(b)** Monotonicity is broken; sum changes non-directionally.
- **Q3:** **(c)** $O(n)$ linear time.
- **Q4:** **(b)** Prevents integer overflow.
- **Q5:** **(b)** $O(n)$ memory to allocate the set and list.

### Section B Solutions
**Q6 Solution:**
```python
def min_sub_array_len(target: int, nums: list[int]) -> int:
    left = 0
    current_sum = 0
    min_len = float('inf')

    for right in range(len(nums)):
        current_sum += nums[right]

        while current_sum >= target:
            min_len = min(min_len, right - left + 1)
            current_sum -= nums[left]
            left += 1

    return min_len if min_len != float('inf') else 0
```

**Q7 Solution:**
```python
def three_sum(nums: list[int]) -> list[list[int]]:
    nums.sort()  # O(N log N)
    result = []
    n = len(nums)

    for i in range(n - 2):
        # Skip duplicate values for the first element
        if i > 0 and nums[i] == nums[i - 1]:
            continue

        left = i + 1
        right = n - 1

        while left < right:
            total = nums[i] + nums[left] + nums[right]
            if total == 0:
                result.append([nums[i], nums[left], nums[right]])
                # Skip duplicate values for left and right
                while left < right and nums[left] == nums[left + 1]:
                    left += 1
                while left < right and nums[right] == nums[right - 1]:
                    right -= 1
                left += 1
                right -= 1
            elif total < 0:
                left += 1
            else:
                right -= 1

    return result
```

### Section C Solutions
**Q8 Solution:**
1. **Complexity:** `s.count(char)` scans the entire string of length $N$ ($O(N)$). Since it is called inside a `for char in s` loop that runs $N$ times, the overall time complexity is **$O(N^2)$**. For $N = 10,000,000$, $N^2 = 10^{14}$ operations, which takes days to complete.
2. **$O(n)$ Optimized Solution:**
   ```python
   from collections import Counter

   def first_unique_optimal(s: str) -> str:
       counts = Counter(s)  # O(N) single pass
       for char in s:        # O(N) second pass
           if counts[char] == 1:
               return char
       return ""
   ```
   - **Time Complexity:** $O(N)$ (two sequential passes).
   - **Space Complexity:** $O(1)$ auxiliary memory (at most 26 keys for English alphabet or 256 for ASCII). Executes in $\approx 0.15$ seconds on 10 million characters.
