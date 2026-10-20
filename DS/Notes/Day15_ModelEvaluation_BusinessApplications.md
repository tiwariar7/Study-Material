# Day 15: Model Evaluation, Feature Engineering & Business Systems
## Classification & Regression Metrics, Imbalanced Learning, Recommendation Systems & Business Alignment

---

# Part 1: Classification Metrics & Cost-Utility Alignment

### 1.1 The Confusion Matrix & Derived Proportions

```
                              ACTUAL POSITIVE (1)         ACTUAL NEGATIVE (0)
 PREDICTED POSITIVE (1)       True Positive (TP)          False Positive (FP) [Type I]
 PREDICTED NEGATIVE (0)       False Negative (FN) [Type II] True Negative (TN)
```

| Metric | Formula | Plain English Meaning | Canonical Business Use Case |
| :--- | :--- | :--- | :--- |
| **Precision** | $\frac{TP}{TP + FP}$ | "Out of all instances we flagged as positive, how many were actually positive?" | Spam detection, Video recommendations, Legal document discovery. |
| **Recall (Sensitivity)** | $\frac{TP}{TP + FN}$ | "Out of all actual positive instances, how many did we successfully capture?" | Fraud detection, Cancer screening, Factory defect detection. |
| **Specificity** | $\frac{TN}{TN + FP}$ | "Out of all actual negative instances, how many did we correctly identify as negative?" | Drug clinical trials, minimizing false alarms in monitoring. |
| **$F_1$-Score** | $2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$ | Harmonic mean of Precision and Recall. Equal weighting. | General classification benchmark on moderately imbalanced data. |
| **$F_\beta$-Score** | $(1 + \beta^2) \frac{\text{Precision} \cdot \text{Recall}}{\beta^2 \text{Precision} + \text{Recall}}$ | Weighted harmonic mean. $\beta=2$ weights Recall $2\times$ higher than Precision. | Healthcare diagnostics ($F_2$), High-value fraud alert systems. |

---

### 1.2 ROC-AUC vs. Precision-Recall AUC (PR-AUC)

#### Receiver Operating Characteristic (ROC) Curve
Plots **True Positive Rate (Recall)** against **False Positive Rate ($\text{FPR} = \frac{FP}{FP + TN}$)** across all possible probability thresholds ($0.0 \to 1.0$).
- **ROC-AUC Score:** Represents the probability that the classifier will rank a randomly chosen positive instance higher than a randomly chosen negative instance. ($0.5 = \text{random guessing}$, $1.0 = \text{perfect}$).

#### The Severe Imbalance Trap: Why ROC-AUC Misleads
In severely imbalanced datasets (e.g., 99.9% Negative, 0.1% Positive in credit card fraud):
- Because $TN$ is massive (e.g., $999,000$), a large surge of $10,000$ False Positives yields a tiny $\text{FPR} = \frac{10000}{10000 + 999000} \approx 0.0099$. The ROC curve barely shifts, reporting an artificially optimistic AUC of $0.98$!
- **The Solution: PR-AUC (Precision-Recall Curve):**
  Plots **Precision** against **Recall**. Because neither metric includes $TN$ in its denominator, PR-AUC focuses exclusively on the minority class. In severe imbalance, **PR-AUC is mandatory**.

---

### 1.3 Cost-Utility Matrix & Decision Threshold Tuning
The standard decision threshold is $0.5$. In business applications, this is almost always suboptimal due to **asymmetric error costs**:

$$\text{Total Expected Cost} = C(FP) \cdot FP + C(FN) \cdot FN$$

*Example (Loan Default Risk):*
- Cost of $FN$ (Uncaught default): $\$50,000$ loss.
- Cost of $FP$ (Denying a creditworthy customer): $\$2,000$ lost interest margin.
- **Action:** Shift the classification threshold down from $0.50$ to **$0.18$**. This trades off higher False Positives (more manual reviews) to drastically minimize catastrophic False Negatives!

---

# Part 2: Regression Evaluation Metrics

### 2.1 The Metric Spectrum

| Metric | Formula | Sensitivity to Outliers | Interpretable in Target Units? | Mathematical Penalty |
| :--- | :--- | :--- | :--- | :--- |
| **Mean Absolute Error (MAE)** | $\frac{1}{n}\sum |y_i - \hat{y}_i|$ | **Low** (Robust) | **Yes** (Exact same units as $y$) | Linear penalty: a \$100 error is worth exactly 10 $\times$ a \$10 error. |
| **Root Mean Squared Error (RMSE)** | $\sqrt{\frac{1}{n}\sum (y_i - \hat{y}_i)^2}$ | **Very High** (Penalizes extremes) | **Yes** (Exact same units as $y$) | Quadratic penalty: a \$100 error is worth 100 $\times$ a \$10 error! |
| **Mean Absolute Percentage Error (MAPE)** | $\frac{100\%}{n}\sum \left|\frac{y_i - \hat{y}_i}{y_i}\right|$ | High when $y_i$ is small | **Yes** (Percentage %) | Fails / divides by zero if actual $y_i = 0$. |
| **$R^2$ (Coefficient of Determination)** | $1 - \frac{\sum (y_i - \hat{y}_i)^2}{\sum (y_i - \bar{y})^2}$ | High | **No** (Dimensionless ratio $\le 1.0$) | Fraction of target variance explained by features. |

#### $R^2$ vs. Adjusted $R^2$:
Adding any new feature to a regression model (even random noise) will **never decrease $R^2$**.  
**Adjusted $R^2$** penalizes models for adding non-informative features:
$$R^2_{\text{adj}} = 1 - \left[ \frac{(1 - R^2)(n - 1)}{n - p - 1} \right]$$
- $n$: Number of observations, $p$: Number of predictors.

---

# Part 3: Advanced Feature Engineering

### 3.1 Categorical Encoding Strategies
1. **One-Hot Encoding:** Creates binary columns for each unique category.
   - *Danger:* High-cardinality features (e.g., `zip_code` with 10,000 values) cause extreme dimensionality explosion and sparse matrices, slowing tree training.
2. **Target (Mean) Encoding:** Replaces each categorical value with the average target value for that category:
   $$\hat{x}_k = \bar{y}_k$$
   - *Danger:* Severe Data Leakage & Overfitting!
   - *Remediation:* Must apply **Out-of-Fold (OOF) Target Encoding** with m-estimate smoothing:
     $$S_i = \frac{n_i \cdot \bar{y}_i + m \cdot \bar{y}_{\text{global}}}{n_i + m}$$

---

### 3.2 Numerical Transformations
- **Log Transformation ($y' = \ln(1 + x)$):** Compresses long right tails, converting multiplicative relationships into additive ones and stabilizing variance (homoscedasticity).
- **Scaling Techniques:**
  - `StandardScaler`: Transforms to $\mu=0, \sigma=1$. Sensitive to outliers.
  - `RobustScaler`: Centers using Median and scales by IQR ($Q_3 - Q_1$). **Immune to extreme outliers**.

---

# Part 4: Production Class Imbalance Remediation

### 4.1 Resampling Protocols
- **Random Undersampling:** Discards majority class rows. (Fast, but wastes valuable training data).
- **SMOTE (Synthetic Minority Over-sampling Technique):** Synthesizes new minority examples by interpolating between nearest neighbors in feature space:
  $$x_{\text{new}} = x_i + \lambda (x_{zi} - x_i) \quad \text{where } \lambda \sim U(0, 1)$$

> [!CAUTION]
> **Production Rule:** Never apply SMOTE or oversampling to the test or validation datasets! Resampling must be applied **strictly inside training folds**.

---

### 4.2 Cost-Sensitive Learning (`class_weight='balanced'`)
Instead of artificially generating synthetic rows, modify the loss function during gradient descent:
$$w_c = \frac{N}{K \cdot N_c}$$
- Minorities receive exponentially higher loss penalties when misclassified, forcing the decision boundary to protect them.

---

# Part 5: Recommendation Systems Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                     RECOMMENDATION SYSTEM PARADIGMS                    │
├────────────────────────────────────────────────────────────────────────┤
│ 1. CONTENT-BASED FILTERING                                             │
│    - Recommends items similar to what user liked in the past.          │
│    - Features: Item metadata (Genre, Actor, Keywords, TF-IDF).         │
│    - Metric: Cosine Similarity between Item Vector and User Profile.   │
│    - Pros: No cold-start for new items; transparent explanations.      │
│    - Cons: Cannot recommend outside user's existing bubble (serendipity)│
├────────────────────────────────────────────────────────────────────────┤
│ 2. COLLABORATIVE FILTERING (CF)                                        │
│    - "Users who agreed in the past will agree in the future."          │
│    - Memory-Based: User-User CF vs. Item-Item CF.                      │
│    - Model-Based: Matrix Factorization (SVD, Alternating Least Squares)│
│      R ≈ U (User Latent Matrix) × V^T (Item Latent Matrix)            │
│    - Pros: Learns latent patterns; domain-independent.                 │
│    - Cons: Severe Cold-Start Problem (Cannot score new users/items).   │
└────────────────────────────────────────────────────────────────────────┘
```

#### Cold-Start Remediation (Hybrid Architectures):
1. **New User:** Collect explicit preferences during onboarding survey; fall back to popularity-based top items.
2. **New Item:** Use Content-Based metadata embeddings until sufficient interaction clicks are logged.

---

# Part 6: Required Deliverable: Executive Decision Framework for Model & Metric Selection

### Step-by-Step Selection Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                MODEL & METRIC SELECTION DECISION TREE                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                    │
                                   What is the nature of the Target (y)?
                                                    │
                    ┌───────────────────────────────┴───────────────────────────────┐
                    ▼                                                               ▼
             CONTINUOUS TARGET                                              DISCRETE / CATEGORICAL
               (REGRESSION)                                                    (CLASSIFICATION)
                    │                                                               │
     Are extreme outliers costly?                                     Is the dataset severely imbalanced?
     ┌──────────────┴──────────────┐                                  ┌─────────────┴─────────────┐
     ▼                             ▼                                  ▼                           ▼
    YES                            NO                                YES (Fraud, Ads)             NO (Balanced)
   RMSE / Huber Loss              MAE                                PR-AUC & Recall              ROC-AUC & Accuracy
                    │                                                               │
     Which Model Family?                                             What is the primary business risk?
     - Baseline: Ridge / Lasso                                       ┌──────────────┴──────────────┐
     - Tabular: XGBoost / LightGBM                                   ▼                             ▼
                                                             FALSE NEGATIVE COST        FALSE POSITIVE COST
                                                             (Cancer, Fraud)            (Spam, Promotions)
                                                             Prioritize RECALL          Prioritize PRECISION
                                                             Tune F2-Score (β=2)        Tune F0.5-Score (β=0.5)
```

---

# Part 7: Structured Interview Question Bank

### Q1: In an email spam classification model, why is Precision far more important than Recall?
**Answer:**
- A False Positive ($FP$) means a critical business email (e.g., job offer, client invoice) is misclassified as spam and moved to the spam folder, causing immediate business harm.
- A False Negative ($FN$) means an unwanted spam email lands in the primary inbox, causing minor inconvenience.
- Because the cost of a False Positive is orders of magnitude higher than a False Negative, the spam classifier must maximize **Precision** (e.g., $> 99.5\%$) by operating at a high decision threshold, ensuring that an email is only marked as spam if the model is virtually certain.

### Q2: Why is the harmonic mean used for the $F_1$-score instead of the simple arithmetic mean?
**Answer:**
The arithmetic mean treats Precision and Recall independently; a classifier that predicts everything as positive on a dataset with 50% positives would get Recall = 1.0 and Precision = 0.5, giving an arithmetic mean of $0.75$. The **harmonic mean** ($\frac{2}{\frac{1}{P} + \frac{1}{R}}$) penalizes extreme trade-offs heavily. If either Precision or Recall approaches 0, the harmonic mean collapses towards 0, ensuring that a high $F_1$-score is achieved only when **both** precision and recall are robust.

### Q3: What is Matrix Factorization in Collaborative Filtering, and how does Singular Value Decomposition (SVD) work?
**Answer:**
Given a sparse User-Item interaction rating matrix $R$ of shape $(m \times n)$, Matrix Factorization decomposes $R$ into the product of two lower-rank dense matrices:
$$R \approx U \cdot V^T$$
- $U$: User latent factor matrix of shape $(m \times k)$.
- $V$: Item latent factor matrix of shape $(n \times k)$.
- $k$: Latent dimensionality (e.g., $k = 50$).
The dot product of user vector $\mathbf{u}_i$ and item vector $\mathbf{v}_j$ predicts the rating for unobserved pairs. SVD minimizes the reconstruction error over observed ratings while adding L2 regularization to prevent overfitting on unobserved ratings.

---

# Part 8: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   MODEL EVALUATION & METRICS CHEAT SHEET               │
├────────────────────────────────────────────────────────────────────────┤
│ 1. CLASSIFICATION:                                                     │
│    - Precision: TP / (TP + FP). Cost of FP is high.                    │
│    - Recall   : TP / (TP + FN). Cost of FN is high (Fraud/Health).     │
│    - F1-Score : Harmonic mean: 2 * (P * R) / (P + R).                  │
│    - Imbalance: Use PR-AUC instead of ROC-AUC.                         │
│    - Threshold Tuning: Optimize threshold = argmin(C_FP*FP + C_FN*FN). │
├────────────────────────────────────────────────────────────────────────┤
│ 2. REGRESSION:                                                         │
│    - MAE  : Linear penalty; robust to outliers; interpretable units.   │
│    - RMSE : Quadratic penalty; heavily punishes large errors.          │
│    - Adj R²: Penalizes useless feature additions: 1 - [(1-R²)(n-1)/(n-p-1)]│
├────────────────────────────────────────────────────────────────────────┤
│ 3. IMBALANCE:                                                          │
│    - class_weight='balanced': Cost-sensitive loss weighting.           │
│    - SMOTE: Synthesizes minority rows. NEVER apply to test set!        │
├────────────────────────────────────────────────────────────────────────┤
│ 4. RECOMMENDERS:                                                       │
│    - Content-Based: Cosine similarity on item metadata. No item cold-start.│
│    - Collaborative: User-Item interactions. SVD Matrix Factorization.  │
│      Suffers from user/item cold-start problem.                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 9: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (20 Marks — 4 marks each)

**Q1:** What is the primary reason for preferring PR-AUC over ROC-AUC when evaluating a fraud detection model with 0.05% positive cases?  
(a) ROC-AUC is computationally too slow to calculate.  
(b) The massive number of True Negatives artificially depresses the False Positive Rate, masking poor precision in ROC-AUC.  
(c) PR-AUC does not require a confusion matrix.  
(d) ROC-AUC can only be evaluated on binary features.  

**Q2:** If a model achieves a Mean Absolute Percentage Error (MAPE) of undefined / infinite error, what was the most likely characteristic of the target test data?  
(a) The target feature contained negative numbers.  
(b) The target feature contained one or more true values equal to zero ($y_i = 0$).  
(c) The test set was too large.  
(d) The model was a decision tree.  

**Q3:** Which metric would you optimize when designing a factory assembly line machine vision system where stopping the line for a false defect ($FP$) costs \$5, but letting a broken part pass ($FN$) causes a \$50,000 catastrophic engine failure?  
(a) Precision  
(b) Specificity  
(c) Recall (or $F_2$-Score)  
(d) Accuracy  

**Q4:** In Collaborative Filtering recommendation systems, what does the "Cold-Start Problem" refer to?  
(a) Slow server startup times during winter months.  
(b) Inability to generate accurate recommendations for brand-new users or brand-new items with zero historical interaction data.  
(c) Running out of memory during SVD matrix factorization.  
(d) Converting categorical features to one-hot encoding.  

**Q5:** What is the fundamental difference between MAE and RMSE in terms of error penalization?  
(a) MAE penalizes errors exponentially; RMSE penalizes errors linearly.  
(b) MAE penalizes errors linearly; RMSE squares errors, penalizing large outliers much more severely.  
(c) RMSE is always smaller than MAE.  
(d) MAE requires calculating logarithms.  

---

### Section B: Mathematics & Metric Computation (25 Marks)

**Q6 (10 Marks):** A diagnostic cancer detection model evaluated on 1,000 biopsy samples produces the following confusion matrix:
- True Positives ($TP$): $80$
- False Positives ($FP$): $20$
- False Negatives ($FN$): $10$
- True Negatives ($TN$): $890$
1. Calculate the **Precision** of the model. (3 marks)
2. Calculate the **Recall** of the model. (3 marks)
3. Calculate the **$F_1$-Score** of the model. (4 marks)

**Q7 (15 Marks):** Calculate the $R^2$ score for a regression model given:
- Total Sum of Squares ($SS_{\text{tot}} = \sum (y_i - \bar{y})^2$): $500$
- Residual Sum of Squares ($SS_{\text{res}} = \sum (y_i - \hat{y}_i)^2$): $125$
Explain the exact business interpretation of this $R^2$ value.

---

### Section C: Implementation & Pipeline Challenges (35 Marks)

**Q8 (20 Marks):** Write a Python function `find_optimal_threshold(y_true, y_probs, cost_fp, cost_fn) -> float` that evaluates candidate classification probability thresholds from $0.05$ to $0.95$ in increments of $0.05$ and returns the threshold that **minimizes total financial loss**.

**Q9 (15 Marks):** Write a Python snippet demonstrating **Out-of-Fold (OOF) Target Encoding** using `KFold` on a training dataset to encode a high-cardinality categorical column `merchant_id` without data leakage.

---

### Section D: Enterprise Architectural Scenario (20 Marks)

**Q10 (20 Marks):** A streaming video entertainment company (like Netflix) wants to build an automated recommendation engine.
1. Design a **Hybrid Recommendation Architecture** combining Content-Based and Collaborative Filtering that mitigates the Cold-Start problem for new movies and new subscribers. (10 marks)
2. Define two offline metrics and two online A/B testing metrics to measure recommender success. (10 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(b)** Massive True Negatives artificially suppress FPR, making ROC-AUC deceptively high.
- **Q2:** **(b)** Division by zero ($y_i = 0$) in the MAPE denominator.
- **Q3:** **(c)** Recall / $F_2$-Score (Minimizing catastrophic False Negatives).
- **Q4:** **(b)** Inability to score new users or new items lacking interaction history.
- **Q5:** **(b)** MAE is linear; RMSE is quadratic, penalizing outliers heavily.

### Section B Solutions
**Q6 Solution:**
1. Precision: $\frac{TP}{TP + FP} = \frac{80}{80 + 20} = \frac{80}{100} = 0.80$ (80.0%).
2. Recall: $\frac{TP}{TP + FN} = \frac{80}{80 + 10} = \frac{80}{90} \approx 0.8889$ (88.89%).
3. $F_1$-Score: $2 \cdot \frac{P \cdot R}{P + R} = 2 \cdot \frac{0.80 \cdot 0.8889}{0.80 + 0.8889} = \frac{1.4222}{1.6889} \approx 0.8421$ (84.21%).

**Q7 Solution:**
- $R^2 = 1 - \frac{SS_{\text{res}}}{SS_{\text{tot}}} = 1 - \frac{125}{500} = 1 - 0.25 = 0.75$ (75.0%).
- **Business Interpretation:** Exactly $75\%$ of the total variance in the dependent target variable is successfully explained and captured by the independent features in the model, with the remaining $25\%$ attributable to unexplained variance or noise.

### Section C Solutions
**Q8 Solution:**
```python
import numpy as np
from sklearn.metrics import confusion_matrix

def find_optimal_threshold(y_true, y_probs, cost_fp, cost_fn) -> float:
    best_threshold = 0.5
    min_cost = float('inf')
    
    for thresh in np.arange(0.05, 1.00, 0.05):
        y_pred = (y_probs >= thresh).astype(int)
        tn, fp, fn, tp = confusion_matrix(y_true, y_pred).ravel()
        total_cost = (fp * cost_fp) + (fn * cost_fn)
        
        if total_cost < min_cost:
            min_cost = total_cost
            best_threshold = round(thresh, 2)
            
    return best_threshold
```

**Q9 Solution:**
```python
import pandas as pd
from sklearn.model_selection import KFold

def oof_target_encode(df, cat_col, target_col, n_splits=5):
    kf = KFold(n_splits=n_splits, shuffle=True, random_state=42)
    df[f"{cat_col}_encoded"] = 0.0
    global_mean = df[target_col].mean()

    for train_idx, val_idx in kf.split(df):
        train_fold = df.iloc[train_idx]
        val_fold = df.iloc[val_idx]

        # Calculate mean strictly on train fold
        means = train_fold.groupby(cat_col)[target_col].mean()
        # Map back to validation fold with fallback to global mean
        df.iloc[val_idx, df.columns.get_loc(f"{cat_col}_encoded")] = (
            val_fold[cat_col].map(means).fillna(global_mean)
        )
    return df
```

### Section D Solutions
**Q10 Solution:**
1. **Hybrid Architecture:**
   - **Offline Model:** Train an implicit Matrix Factorization model (e.g., SVD / ALS) on user viewing history to capture latent collaborative affinities.
   - **Content-Based Fallback:** For new movies (Item Cold-Start), generate text/video embeddings (genre, actors, summary) and compute cosine similarity against existing popular catalog items.
   - **Onboarding Survey:** For new subscribers (User Cold-Start), present 5 favorite genre cards; recommend top-rated movies in selected genres until 5 stream events are logged, then transition the user into the Collaborative Filtering pipeline.
2. **Metrics:**
   - **Offline Metrics:** MAP@K (Mean Average Precision at K), NDCG@K (Normalized Discounted Cumulative Gain).
   - **Online A/B Testing Metrics:** Click-Through Rate (CTR) on recommended rows, 30-Day Subscriber Retention Rate, Total Streaming Watch Time (hours).
