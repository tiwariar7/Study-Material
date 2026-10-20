# Day 21: Python, Analytics & ML 45-Minute Mock Interview Simulation
## Live Algorithmic Coding, Pandas Wrangling, Applied Statistics & ML System Design

---

# Part 1: Phase 1 — Algorithmic Coding (12 Minutes)

### The Challenge: Minimum Window Substring
**Problem Statement:**  
Given two strings `s` and `t` of lengths $m$ and $n$ respectively, return the minimum window substring of `s` such that every character in `t` (including duplicates) is included in the window. If there is no such substring, return the empty string `""`.

**Candidate Spoken Thought Process:**
> *"I'll solve this using the **Dynamic Sliding Window** pattern with two frequency hash maps: `t_counts` to store the target character frequencies, and `window_counts` to track the current window in `s`.  
> I'll track two integer variables: `required`, which is the number of unique characters in `t`, and `formed`, which tracks how many unique characters currently meet the target frequency in the window.  
> As the `right` pointer expands the window, whenever `window_counts[char] == t_counts[char]`, I increment `formed`. Once `formed == required`, the window is valid. Then, I contract the `left` pointer to minimize the window size while preserving validity.  
> This ensures an optimal **$O(m + n)$ time complexity** because both pointers advance at most $m$ times, with $O(1)$ auxiliary space since the character set is bounded by standard ASCII."*

**Production Solution:**
```python
from collections import Counter

def min_window(s: str, t: str) -> str:
    if not s or not t or len(s) < len(t):
        return ""

    t_counts = Counter(t)
    required = len(t_counts)
    
    window_counts = {}
    formed = 0
    
    # ans tuple: (window_length, left_index, right_index)
    ans = (float("inf"), None, None)
    left = 0

    for right, char in enumerate(s):
        window_counts[char] = window_counts.get(char, 0) + 1
        
        if char in t_counts and window_counts[char] == t_counts[char]:
            formed += 1

        # Contract window from left as long as it remains valid
        while left <= right and formed == required:
            current_char = s[left]
            
            # Update smallest window found so far
            if (right - left + 1) < ans[0]:
                ans = (right - left + 1, left, right)

            window_counts[current_char] -= 1
            if current_char in t_counts and window_counts[current_char] < t_counts[current_char]:
                formed -= 1

            left += 1

    return "" if ans[0] == float("inf") else s[ans[1]:ans[2] + 1]

# Verification
assert min_window("ADOBECODEBANC", "ABC") == "BANC"
assert min_window("a", "a") == "a"
assert min_window("a", "aa") == ""
```

---

# Part 2: Phase 2 — Production Pandas Wrangling (12 Minutes)

### The Challenge: Customer Sessionization & Conversion Metrics
**Problem Statement:**  
You are given a raw clickstream DataFrame `events` with columns: `['user_id', 'event_timestamp', 'event_type', 'item_id']`.  
An active shopping session is defined as a sequence of events for a user where the gap between consecutive events is **less than 30 minutes**.
1. Sessionize the data: assign a unique sequential `session_id` to each user session.
2. Produce an executive summary table showing:
   - Total sessions per user.
   - Average session duration in minutes.
   - Session conversion rate (percentage of sessions containing at least one `'purchase'` event).

**Candidate Spoken Thought Process:**
> *"First, I ensure `event_timestamp` is parsed as a datetime and sort by `user_id` and `event_timestamp`.  
> To detect session boundaries, I use `.groupby('user_id')['event_timestamp'].diff()` to find the time delta between consecutive events.  
> Whenever `delta > 30 minutes` or is null (the first event), a new session starts. I convert this boolean condition into a cumulative sum (`cumsum()`), which generates a unique sequential `session_id` per user.  
> Finally, I group by `user_id` and `session_id` to compute duration and purchase flags, followed by a second aggregation to produce the user summary metrics."*

**Production Solution:**
```python
import pandas as pd
import numpy as np

def analyze_user_sessions(events: pd.DataFrame) -> pd.DataFrame:
    # 1. Type Enforcement & Sorting
    df = events.copy()
    df["event_timestamp"] = pd.to_datetime(df["event_timestamp"])
    df = df.sort_values(["user_id", "event_timestamp"]).reset_index(drop=True)

    # 2. Compute Intersite Inactivity Deltas
    time_diff = df.groupby("user_id")["event_timestamp"].diff()
    
    # 3. New Session Flag: True if first event or gap >= 30 minutes
    new_session_flag = time_diff.isna() | (time_diff >= pd.Timedelta(minutes=30))
    
    # 4. Generate User-Specific Session ID via Cumulative Sum
    df["session_seq"] = df.groupby("user_id")[new_session_flag.name].cumsum()
    df["session_id"] = df["user_id"].astype(str) + "_s" + df["session_seq"].astype(str)

    # 5. Session-Level Aggregation
    session_summary = df.groupby(["user_id", "session_id"]).agg(
        session_start=("event_timestamp", "min"),
        session_end=("event_timestamp", "max"),
        has_purchase=("event_type", lambda x: (x == "purchase").any())
    ).reset_index()

    session_summary["duration_minutes"] = (
        (session_summary["session_end"] - session_summary["session_start"]).dt.total_seconds() / 60.0
    )

    # 6. User-Level Executive Summary
    user_report = session_summary.groupby("user_id").agg(
        total_sessions=("session_id", "count"),
        avg_session_duration_mins=("duration_minutes", "mean"),
        conversion_rate_pct=("has_purchase", lambda x: round(100.0 * x.mean(), 2))
    ).reset_index()

    return user_report
```

---

# Part 3: Phase 3 — Applied Statistics & Probability (10 Minutes)

### The Challenge: A/B Testing Early-Stopping Paradox & Bayesian Fraud
**Interviewer:**  
1. *"An analyst runs an A/B test for 3 days. They observe $p = 0.038$ on day 3 and immediately declare Variant B the winner and stop the test. What statistical error was committed, and what is the consequence?"*  
2. *"A high-value payment gateway processes transactions where $0.2\%$ are fraudulent. Our fraud model has an $80\%$ Sensitivity (Recall) and a $1\%$ False Positive Rate. A transaction is flagged by the model. What is the exact probability that this transaction is actually fraudulent?"*

**Candidate Spoken Thought Process:**
> *"1. The analyst committed the **Continuous Peeking Problem (Alpha Inflation)**. By checking the $p$-value repeatedly and stopping as soon as it crossed $0.05$, the true False Positive Rate is not 5%, but can inflate up to 30–40%. The observed lift is likely random noise (winner's curse). The test should have run for a predetermined fixed sample horizon based on power calculations.  
> 2. For the second question, this is a classic application of **Bayes' Theorem** illustrating the **Base Rate Fallacy**."*

#### Mathematical Proof Walkthrough:
$$P(\text{Fraud}) = 0.002, \quad P(\text{Legit}) = 0.998$$
$$P(\text{Flag} \mid \text{Fraud}) = 0.80, \quad P(\text{Flag} \mid \text{Legit}) = 0.01$$

$$\text{Marginal } P(\text{Flag}) = (0.80 \times 0.002) + (0.01 \times 0.998) = 0.0016 + 0.00998 = 0.01158$$

$$P(\text{Fraud} \mid \text{Flag}) = \frac{0.80 \times 0.002}{0.01158} = \frac{0.0016}{0.01158} \approx 0.1382 \quad (\mathbf{13.82\%})$$

**Spoken Conclusion:**
> *"Even with an 80% sensitive model and a low 1% false positive rate, **only 13.8% of flagged transactions are actual fraud**. Over 86% of the flags are false alarms due to the extremely rare 0.2% base rate. This is why automated blocking would be disastrous; instead, flagged transactions should route to human review or trigger multi-factor authentication (SMS/2FA)."*

---

# Part 4: Phase 4 — End-to-End Machine Learning Business Scenario (11 Minutes)

### The Challenge: E-Commerce Chargeback Prevention Engine
**Interviewer:** *"We are building an automated transaction risk engine for a merchant processing \$50M monthly. If a fraudulent charge slips through, we lose the \$100 product value plus a \$25 bank chargeback fee (\$125 total loss). If we falsely reject a good customer, we lose the \$15 profit margin and cause user friction. How do you formulate, build, evaluate, and tune this model?"*

**Candidate Spoken Architecture Plan:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                   FRAUD ENGINE DESIGN ARCHITECTURE                     │
├────────────────────────────────────────────────────────────────────────┤
│ 1. PROBLEM FORMULATION:                                                │
│    - Binary Classification with Severe Class Imbalance (~0.5% fraud).  │
│    - Cost Asymmetry: Cost(FN) = $125 | Cost(FP) = $15 (8.3x ratio).    │
├────────────────────────────────────────────────────────────────────────┤
│ 2. EVALUATION METRIC:                                                  │
│    - DO NOT USE ACCURACY OR ROC-AUC!                                   │
│    - Optimize Precision-Recall AUC (PR-AUC) and Cost-Utility Function: │
│      Total Cost = ($15 * FP) + ($125 * FN).                            │
├────────────────────────────────────────────────────────────────────────┤
│ 3. MODEL FAMILY & LEAKAGE PREVENTION:                                  │
│    - XGBoost / LightGBM with scale_pos_weight = 8.3                    │
│    - Time-Series Split (Walk-Forward validation).                      │
│    - Group features strictly into backward-looking sliding windows     │
│      (e.g., velocity: transactions in last 10 mins).                   │
├────────────────────────────────────────────────────────────────────────┤
│ 4. THRESHOLD OPTIMIZATION:                                             │
│    - Default 0.5 threshold will miss too many $125 losses.             │
│    - Sweep operational threshold from 0.05 to 0.95 to find minimum     │
│      dollar loss point (typically around 0.12 - 0.18).                 │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 5: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   ANALYTICS & ML MOCK CHEAT SHEET                      │
├────────────────────────────────────────────────────────────────────────┤
│ 1. MINIMUM WINDOW SUBSTRING:                                           │
│    - Dynamic sliding window. Track formed == required unique chars.    │
│    - Contract left pointer while valid. O(M + N) time, O(1) space.     │
├────────────────────────────────────────────────────────────────────────┤
│ 2. PANDAS SESSIONIZATION:                                              │
│    - diff() > 30 mins -> boolean flag -> cumsum() = session_id.        │
├────────────────────────────────────────────────────────────────────────┤
│ 3. APPLIED STATISTICS:                                                 │
│    - Base Rate Fallacy: P(A|B) is low when P(A) is rare, even with     │
│      high test sensitivity! P(Fraud|Flag) = 13.8%.                     │
│    - Peeking Problem: Early stopping inflates alpha up to 30-40%.      │
├────────────────────────────────────────────────────────────────────────┤
│ 4. BUSINESS ML COST ALIGNMENT:                                         │
│    - Threshold = argmin[ Cost(FP)*FP + Cost(FN)*FN ].                  │
│    - When Cost(FN) >> Cost(FP), lower threshold from 0.50 to ~0.15.    │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 6: 100-Mark Self-Assessment Examination

### Section A: Algorithmic & Pandas Mechanics (25 Marks — 5 marks each)

**Q1:** What is the time complexity of the Minimum Window Substring sliding window algorithm on string `s` of length $M$ and target `t` of length $N$?  
(a) $O(M \times N)$  
(b) $O(M + N)$  
(c) $O(M \log N)$  
(d) $O(M^2)$  

**Q2:** In Pandas, what is the effect of applying `.cumsum()` to a boolean Series `[True, False, False, True, False]`?  
(a) `[1, 0, 0, 1, 0]`  
(b) `[1, 1, 1, 2, 2]`  
(c) `[True, False]`  
(d) `TypeError`  

**Q3:** In a binary classification problem where the positive class is 0.1% of the data, what will a naive dummy classifier that always predicts 0 achieve in accuracy?  
(a) 0.1%  
(b) 50.0%  
(c) 99.9%  
(d) 100.0%  

**Q4:** In Bayes' Theorem, what term represents the prior probability of evidence $P(B)$?  
(a) Posterior  
(b) Likelihood  
(c) Marginal Evidence  
(d) Prior  

**Q5:** Why does standard $K$-Fold cross-validation cause optimistic data leakage when applied to time-series forecasting datasets?  
(a) It shuffles future data into the training set, allowing the model to peek into the future.  
(b) It drops missing values.  
(c) It requires too much memory.  
(d) Time-series data cannot be divided into $K$ parts.  

---

### Section B: Analytical Implementation (50 Marks)

**Q6 (25 Marks - Pandas Rolling Active Users):**  
Given a DataFrame `user_logins (user_id, login_timestamp)`. Write a production Pandas script to compute the **7-Day Rolling Active Users (7D-RAU)** for each calendar day: the count of unique `user_id`s who logged in within the 7-day window ending on that day.

**Q7 (25 Marks - Optimal Threshold Computation):**  
Given a cost matrix where a False Negative costs \$500 and a False Positive costs \$20. You have an array of true labels `y_true` (10,000 samples with 100 positives) and predicted probabilities `y_probs`. Write a Python function to find the exact threshold between $0.05$ and $0.50$ that minimizes total financial loss.

---

### Section C: Business Problem Architecture (25 Marks)

**Q8 (25 Marks):** You are designing a recommendation system for an online grocery delivery app.
1. Why is an Item-Item collaborative filtering model preferred over User-User collaborative filtering for grocery baskets? (10 marks)
2. How do you incorporate item expiration / shelf-life into the recommendation ranking score? (15 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** $O(M + N)$ linear time.
- **Q2:** **(b)** `[1, 1, 1, 2, 2]`.
- **Q3:** **(c)** 99.9% (demonstrating why accuracy is useless for imbalanced data).
- **Q4:** **(c)** Marginal Evidence.
- **Q5:** **(a)** Future data leaks into the past during random shuffling.

### Section B Solutions
**Q6 Solution:**
```python
import pandas as pd

def calculate_7d_rolling_active_users(logins: pd.DataFrame) -> pd.DataFrame:
    df = logins.copy()
    df["login_date"] = pd.to_datetime(df["login_timestamp"]).dt.floor("D")
    
    # 1. Deduplicate user logins per calendar day
    daily_unique = df.drop_duplicates(["login_date", "user_id"])
    
    # 2. Complete calendar date range
    all_dates = pd.date_range(daily_unique["login_date"].min(), daily_unique["login_date"].max(), freq="D")
    
    # 3. Compute 7-day rolling unique count
    records = []
    for current_date in all_dates:
        start_date = current_date - pd.Timedelta(days=6)
        active_in_window = daily_unique[
            (daily_unique["login_date"] >= start_date) & 
            (daily_unique["login_date"] <= current_date)
        ]["user_id"].nunique()
        records.append({"date": current_date, "rolling_7d_active_users": active_in_window})
        
    return pd.DataFrame(records)
```

**Q7 Solution:**
```python
import numpy as np
from sklearn.metrics import confusion_matrix

def find_optimal_loss_threshold(y_true, y_probs):
    best_thresh = 0.50
    min_loss = float('inf')
    
    for thresh in np.arange(0.05, 0.51, 0.01):
        y_pred = (y_probs >= thresh).astype(int)
        tn, fp, fn, tp = confusion_matrix(y_true, y_pred).ravel()
        loss = (fp * 20) + (fn * 500)
        
        if loss < min_loss:
            min_loss = loss
            best_thresh = round(thresh, 2)
            
    return best_thresh, min_loss
```

### Section C Solutions
**Q8 Solution:**
1. **Item-Item CF Preference:**
   - In grocery platforms, user tastes vary dramatically over time, but item relationships are static and stable (e.g., people who buy milk frequently buy cereal or coffee).
   - The number of unique catalog items ($10,000$) is orders of magnitude smaller than the number of active users ($5,000,000$). Computing and caching the Item-Item similarity matrix requires minimal memory ($10,000 \times 10,000$), whereas User-User similarity is computationally intractable.
2. **Shelf-Life Ranking Score:**
   - Formulate the final ranking score as a composite objective:
     $$\text{FinalScore} = w_1 \cdot \text{RelevanceScore} + w_2 \cdot \text{UrgencyPenalty}$$
   - Where $\text{UrgencyPenalty} = \exp\left(-\frac{\text{DaysUntilExpiration}}{\lambda}\right)$. Items nearing expiration receive a boost in the carousel ranking (or are offered via dynamic discount bundles), minimizing warehouse food waste while maintaining consumer relevance.
