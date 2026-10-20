# Day 14: Machine Learning Fundamentals & Model Mechanics
## Supervised vs. Unsupervised, Regression, Ensembles, Clustering, Cross-Validation & Data Leakage

---

# Part 1: Machine Learning Taxonomy & Problem Framing

```
                               ┌────────────────────────────────────────┐
                               │       MACHINE LEARNING PARADIGMS       │
                               └────────────────────────────────────────┘
                                    │                               │
                 ┌──────────────────┴─────────────┐  ┌──────────────┴────────────────┐
                 ▼                                │  ▼                               ▼
       SUPERVISED LEARNING                        │  UNSUPERVISED LEARNING      REINFORCEMENT
  (Labeled Data: Target y exists)                 │  (Unlabeled Data: Find X)   LEARNING (RL)
       │                       │                  │  │                  │       (Agent, State,
       ▼                       ▼                  │  ▼                  ▼       Reward, Action)
  REGRESSION            CLASSIFICATION            │  CLUSTERING    DIMENSIONALITY
  (Continuous Target)   (Discrete Target)         │  (K-Means,     REDUCTION
  - Linear Regression   - Logistic Regression     │   DBSCAN)      (PCA, t-SNE)
  - Random Forest       - Random Forest Classifier│
  - XGBoost             - Support Vector Machines │
```

---

# Part 2: Supervised Learning: Regression Mechanics

### 2.1 Linear Regression
Models the linear relationship between continuous dependent target $y$ and independent features $\mathbf{x} = [x_1, x_2, \dots, x_p]^T$:
$$\hat{y} = w_0 + w_1 x_1 + w_2 x_2 + \dots + w_p x_p = \mathbf{w}^T \mathbf{x}$$

#### Ordinary Least Squares (OLS) Closed-Form Solution:
Minimizes the Residual Sum of Squares (RSS). In matrix notation:
$$\mathbf{w} = (\mathbf{X}^T \mathbf{X})^{-1} \mathbf{X}^T \mathbf{y}$$
- **Computational Cost:** Inverting $(\mathbf{X}^T \mathbf{X})$ requires $O(p^3)$ time (where $p$ is feature count). When $p > 10,000$ or data does not fit in RAM, **Gradient Descent** is used instead.

#### Gradient Descent Optimization:
Iteratively updates weights in the opposite direction of the cost function gradient:
$$w_j := w_j - \alpha \frac{\partial}{\partial w_j} J(\mathbf{w})$$
- $\alpha$: Learning rate (too large causes divergence; too small causes slow convergence).

---

### 2.2 Classical Assumptions of OLS (Gauss-Markov Theorem)
For OLS to be BLUE (Best Linear Unbiased Estimator):
1. **Linearity:** Relationship between features and target must be linear in parameters.
2. **Homoscedasticity:** Variance of error terms $\epsilon_i$ must be constant across all levels of features (no "fan-shaped" residual plots).
3. **Independence:** Residuals must be uncorrelated (tested via Durbin-Watson statistic for time-series autocorrelation).
4. **Normality of Residuals:** Errors must be normally distributed: $\epsilon \sim \mathcal{N}(0, \sigma^2)$.
5. **No Multicollinearity:** Independent variables must not be highly correlated with each other (Variance Inflation Factor $\text{VIF} < 5$).

---

### 2.3 Regularization: Ridge (L2) vs. Lasso (L1) vs. ElasticNet

```
L2 RIDGE REGULARIZATION (Circular Constraint)
J(w) = MSE + λ ∑ w_j²
* Shrinks coefficients towards zero *
* NEVER sets coefficients exactly to zero *
* Handles multicollinearity *

L1 LASSO REGULARIZATION (Diamond/Corner Constraint)
J(w) = MSE + λ ∑ |w_j|
* Shrinks coefficients AND sets non-informative weights to EXACTLY ZERO *
* Performs AUTOMATIC FEATURE SELECTION *
```

- **ElasticNet:** Convex combination of L1 and L2 penalties ($r \cdot \text{L1} + \frac{1-r}{2} \cdot \text{L2}$). Ideal when multiple features are correlated (Lasso arbitrarily picks one; ElasticNet keeps groups).

---

# Part 3: Supervised Learning: Classification Mechanics

### 3.1 Logistic Regression & The Sigmoid Function
Used for binary classification ($y \in \{0, 1\}$). Rather than modeling continuous values, it models the **probability** $P(y=1 \mid \mathbf{x})$:

#### The Sigmoid (Logistic) Function:
$$\sigma(z) = \frac{1}{1 + e^{-z}} \quad \text{where } z = \mathbf{w}^T \mathbf{x} + b$$
- Maps any real number $z \in (-\infty, +\infty)$ into a valid probability interval $(0, 1)$.

#### Odds & Log-Odds (Logit):
$$\text{Odds} = \frac{p}{1-p} \implies \ln\left(\frac{p}{1-p}\right) = \mathbf{w}^T \mathbf{x}$$
- **Coefficient Interpretation:** For a 1-unit increase in feature $x_j$, the **log-odds** increase by $w_j$, meaning the **odds multiply by $e^{w_j}$**.

#### Binary Cross-Entropy Cost Function (Log-Loss):
$$J(\mathbf{w}) = -\frac{1}{m} \sum_{i=1}^m \left[ y^{(i)} \ln(\hat{y}^{(i)}) + (1 - y^{(i)}) \ln(1 - \hat{y}^{(i)}) \right]$$
- Penalizes confident incorrect predictions with infinite loss.

---

### 3.2 Decision Trees (CART - Classification and Regression Trees)
A non-parametric hierarchical model that recursively partitions the feature space into orthogonal rectangular regions.

#### Splitting Criteria:
1. **Gini Impurity (Default in Scikit-Learn):**
   $$I_G(t) = 1 - \sum_{i=1}^C p_i^2$$
   - Measures probability of misclassifying a randomly chosen element. (Gini = 0 for pure node).
2. **Entropy & Information Gain:**
   $$H(t) = -\sum_{i=1}^C p_i \log_2(p_i), \quad \text{Gain} = H(\text{Parent}) - \sum \frac{N_j}{N} H(\text{Child}_j)$$

#### Overfitting Mitigation (Hyperparameters):
- `max_depth`: Limits tree depth.
- `min_samples_split`: Minimum samples required to split an internal node.
- `min_samples_leaf`: Minimum samples required to exist at a leaf node.

---

### 3.3 Random Forests (Ensemble Bagging)
A Random Forest constructs an ensemble of $B$ independent Decision Trees to drastically reduce variance without increasing bias.

```
                  [ Training Dataset (N samples, P features) ]
                         │                    │
              Bootstrap Sample 1       Bootstrap Sample 2       ...
                 (with replacement)       (with replacement)
                         │                    │
                  Random Subspace         Random Subspace
                  (√P random features)    (√P random features)
                         │                    │
                         ▼                    ▼
                   Tree 1 (Deep)        Tree 2 (Deep)           ...
                         │                    │
                         └──────────┬─────────┘
                                    ▼
                         [ Majority Vote / Average ]
```

#### Why Random Forests Work (Decorrelation):
1. **Bootstrap Aggregating (Bagging):** Each tree is trained on a random sample of size $N$ drawn **with replacement** (each bootstrap sample contains $\approx 63.2\%$ of original data; the remaining $36.8\%$ is the **Out-of-Bag (OOB)** sample used for internal validation).
2. **Random Subspace Method:** At every single node split, only a random subset of features (typically $\sqrt{P}$ for classification, $\frac{P}{3}$ for regression) is considered. This **decorrelates** the trees, ensuring strong features don't dominate every tree.

---

# Part 4: Unsupervised Learning: K-Means Clustering

### 4.1 The K-Means Algorithm (Lloyd's Algorithm)
Partitions $N$ unlabeled observations into $K$ predefined, non-overlapping clusters:
1. **Initialization:** Randomly pick $K$ initial centroid positions $\mu_1, \dots, \mu_K$.
2. **Assignment Step:** Assign each data point $x_i$ to its nearest centroid based on Euclidean distance:
   $$c^{(i)} = \arg\min_j \|x^{(i)} - \mu_j\|^2$$
3. **Update Step:** Recalculate centroids as the mathematical mean of all assigned points:
   $$\mu_j = \frac{1}{|S_j|} \sum_{i \in S_j} x^{(i)}$$
4. **Convergence:** Repeat steps 2 and 3 until centroids stop moving or reach maximum iterations.

- **Objective Function (Inertia / WCSS):**
  $$\text{WCSS} = \sum_{j=1}^K \sum_{i \in S_j} \|x^{(i)} - \mu_j\|^2$$

---

### 4.2 Finding the Optimal $K$

1. **The Elbow Method:** Plot WCSS (Inertia) against $K$. Look for the "elbow" point where adding more clusters yields diminishing returns in variance reduction.
2. **Silhouette Coefficient:**
   $$s(i) = \frac{b(i) - a(i)}{\max(a(i), b(i))} \quad (-1 \le s(i) \le +1)$$
   - $a(i)$: Mean intra-cluster distance (cohesion).
   - $b(i)$: Mean nearest-cluster distance (separation).
   - Score near $+1$ indicates well-separated, dense clusters; score near $0$ indicates overlapping clusters.

> [!WARNING]
> **K-Means Limitations:** Assumes clusters are spherical and have similar variances. Extremely sensitive to outliers and requires all features to be standardized (`StandardScaler`) beforehand!

---

# Part 5: Validation Governance, Overfitting & Data Leakage

### 5.1 The Bias-Variance Decomposition
$$\text{Expected Test Error} = \text{Bias}^2 + \text{Variance} + \sigma^2 \text{ (Irreducible Error)}$$
- **High Bias (Underfitting):** Model is too simplistic (e.g., fitting a straight line to quadratic data). High error on both training and test data.
- **High Variance (Overfitting):** Model memorizes training noise (e.g., deep unpruned decision tree). Low training error, but high test error.

---

### 5.2 Cross-Validation Protocols
- **$K$-Fold Cross-Validation:** Dataset is partitioned into $K$ equal folds. Model is trained on $K-1$ folds and evaluated on the remaining fold, rotating $K$ times.
- **Stratified $K$-Fold:** Guarantees that each fold preserves the exact class proportion as the full dataset. **Mandatory for imbalanced classification!**
- **Time-Series Split (Walk-Forward):** Never shuffle time-series data! Folds must respect temporal order (Train on past, test on future).

---

### 5.3 Data Leakage: The Silent Model Killer
Data Leakage occurs when information from outside the training dataset is inadvertently used to create the model, producing unrealistically high test metrics that collapse in production.

#### Common Leakage Sources:
1. **Preprocessing Leakage (Global Scaling):** Fitting `StandardScaler()` or `SimpleImputer()` on the **entire dataset** before splitting!
   - *Fix:* Fit preprocessing transformers **strictly on the training fold only**, then transform the validation/test folds.
2. **Target Leakage:** Including a feature that is a proxy for the target or only recorded *after* the event occurs (e.g., including `customer_refund_timestamp` to predict customer returns).

#### The Scikit-Learn Solution: `Pipeline`
```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import RandomForestClassifier

# Encapsulating scaler + model inside Pipeline guarantees scaler fits ONLY on train folds during cross-validation!
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('model', RandomForestClassifier(n_estimators=100, random_state=42))
])
```

---

# Part 6: Production Python Machine Learning Implementation

```python
import numpy as np
import pandas as pd
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split, StratifiedKFold, cross_val_score
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import RandomForestClassifier
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report, roc_auc_score

# 1. Generate Synthetic Imbalanced Dataset (e.g., Customer Churn)
X, y = make_classification(
    n_samples=5000, 
    n_features=15, 
    n_informative=10, 
    weights=[0.85, 0.15], # 15% minority class
    random_state=42
)

# 2. Strict Stratified Train-Test Split (Holdout Test Set)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.20, stratify=y, random_state=42
)

# 3. Leak-Free Production Pipeline
clf_pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('rf', RandomForestClassifier(
        n_estimators=150, 
        max_depth=8, 
        min_samples_leaf=5, 
        class_weight='balanced', # Mitigates class imbalance
        random_state=42
    ))
])

# 4. Stratified K-Fold Cross-Validation on Training Set Only
cv_strategy = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
cv_scores = cross_val_score(clf_pipeline, X_train, y_train, cv=cv_strategy, scoring='roc_auc')

print(f"5-Fold CV ROC-AUC: {np.mean(cv_scores):.4f} (+/- {np.std(cv_scores):.4f})")

# 5. Final Evaluation on Holdout Test Set
clf_pipeline.fit(X_train, y_train)
y_pred = clf_pipeline.predict(X_test)
y_pred_proba = clf_pipeline.predict_proba(X_test)[:, 1]

print("\nHoldout Test Classification Report:")
print(classification_report(y_test, y_pred, target_names=['Retained', 'Churned']))
print(f"Holdout ROC-AUC Score: {roc_auc_score(y_test, y_pred_proba):.4f}")
```

---

# Part 7: Structured Interview Question Bank

### Q1: Why is feature scaling (e.g., StandardScaler) required for Logistic Regression and K-Means, but completely unnecessary for Decision Trees and Random Forests?
**Answer:**
- **Distance- & Gradient-Based Models (Logistic Regression, K-Means, SVM, Neural Networks):** Calculate Euclidean distances or use gradient descent optimization. If feature $A$ is on scale $[0, 1]$ and feature $B$ is on scale $[0, 1000000]$, feature $B$ completely dominates distance metrics and causes elongated, oscillating gradient updates.
- **Tree-Based Models (Decision Trees, Random Forests, XGBoost):** Are scale-invariant because splits evaluate **monotonic rank order** along individual axes ($x_j \le \text{threshold}$). Multiplying a feature by 1,000 does not alter the relative ordering of values, so tree splits remain 100% identical.

### Q2: How does L1 (Lasso) regularization perform feature selection, whereas L2 (Ridge) does not?
**Answer:**
Geometrically, L1 regularization adds a penalty proportional to the absolute values of weights ($\sum |w_j|$), which creates a **diamond-shaped constraint boundary** with sharp corners on the coordinate axes. The elliptical contours of the MSE loss function frequently intersect the L1 constraint region directly at these axis corners, setting non-informative feature weights **strictly to zero**. In contrast, L2 regularization creates a smooth **circular constraint** ($\sum w_j^2$); intersections occur along curves, shrinking weights close to zero but never reaching exact zero.

### Q3: What is the "Out-of-Bag" (OOB) error in a Random Forest, and how does it save computational time?
**Answer:**
Because each tree in a Random Forest is trained on a bootstrap sample drawn with replacement, approximately $1 - 1/e \approx 63.2\%$ of the original training instances are included in any given tree's training set. The remaining $\approx 36.8\%$ of instances are **Out-of-Bag (OOB)** for that tree. By passing each training instance through all trees for which it was OOB and computing the ensemble prediction, the model obtains an unbiased validation score without needing to execute a separate, computationally expensive $K$-fold cross-validation loop.

---

# Part 8: 10-Minute Rapid Revision Sheet

```
┌────────────────────────────────────────────────────────────────────────┐
│                   MACHINE LEARNING FUNDAMENTALS CHEAT SHEET            │
├────────────────────────────────────────────────────────────────────────┤
│ 1. REGRESSION & CLASSIFICATION:                                        │
│    - Linear Reg : Continuous target. OLS minimizes MSE. Gauss-Markov.  │
│    - Logistic   : Binary target. Sigmoid σ(z) maps to (0, 1). Log-Loss.│
│    - Ridge (L2) : Shrinks weights; handles collinearity.               │
│    - Lasso (L1) : Sets weights to zero; feature selection.             │
├────────────────────────────────────────────────────────────────────────┤
│ 2. ENSEMBLES & TREES:                                                  │
│    - Decision Tree: CART uses Gini or Entropy. High variance.          │
│    - Random Forest: Bagging + Random Subspace (√P features).           │
│                     Reduces variance; decorrelates trees; OOB score.   │
├────────────────────────────────────────────────────────────────────────┤
│ 3. UNSUPERVISED:                                                       │
│    - K-Means: Minimizes WCSS (Inertia). Requires feature scaling!      │
│    - Elbow Method & Silhouette Score (-1 to +1) find optimal K.        │
├────────────────────────────────────────────────────────────────────────┤
│ 4. VALIDATION GOVERNANCE:                                              │
│    - High Bias = Underfitting | High Variance = Overfitting.           │
│    - Stratified K-Fold: Preserves minority class proportions.          │
│    - Data Leakage: NEVER scale/impute on full dataset! Use Pipeline.   │
└────────────────────────────────────────────────────────────────────────┘
```

---

# Part 9: 100-Mark Self-Assessment Examination

### Section A: Conceptual Foundations (20 Marks — 4 marks each)

**Q1:** Which of the following models is inherently invariant to monotonic feature scaling?  
(a) Logistic Regression  
(b) Support Vector Machines (SVM)  
(c) Random Forest Classifier  
(d) K-Means Clustering  

**Q2:** What is the primary mathematical mechanism by which Lasso (L1) regularization sets coefficients to zero?  
(a) Zero-division error in gradient descent.  
(b) Intersection of the loss function contours with the sharp corners of the L1 diamond constraint region on the axes.  
(c) Removing features with negative correlation.  
(d) Applying a softmax filter across all weights.  

**Q3:** In a binary classification problem with severe class imbalance (98% negative, 2% positive), which metric should be prioritized during model selection?  
(a) Overall Accuracy  
(b) Precision-Recall AUC (PR-AUC) or F1-Score  
(c) Mean Squared Error (MSE)  
(d) R-Squared ($R^2$)  

**Q4:** What percentage of training data is included, on average, in a single bootstrap sample of size $N$ drawn with replacement?  
(a) 50.0%  
(b) 63.2%  
(c) 80.0%  
(d) 100.0%  

**Q5:** In K-Means clustering, what does a Silhouette Coefficient of $+0.85$ indicate?  
(a) The model is severely overfitted.  
(b) The clusters are dense and well-separated from neighboring clusters.  
(c) The optimal number of clusters is $K = 1$.  
(d) All data points are equidistant from the origin.  

---

### Section B: Mathematics & Algorithmic Tracing (25 Marks)

**Q6 (10 Marks):** Given a Logistic Regression model with intercept $w_0 = -2.0$ and feature weight $w_1 = 0.5$ for feature $x_1$:
1. Write down the equation for the predicted probability $\hat{p} = P(y=1 \mid x_1)$.
2. Calculate the exact predicted probability $\hat{p}$ when $x_1 = 4.0$. (Note: $e^0 = 1$).
3. With a standard classification threshold of $0.5$, what is the predicted class?

**Q7 (15 Marks):** Calculate the **Gini Impurity** for a decision tree node containing 40 samples of Class A and 10 samples of Class B. Show all calculation steps.

---

### Section C: Implementation Challenges (35 Marks)

**Q8 (20 Marks):** Write a Python function using Scikit-Learn that creates an end-to-end `Pipeline` containing:
1. `SimpleImputer` using the `'median'` strategy.
2. `StandardScaler`.
3. `LogisticRegression` with L2 penalty and `class_weight='balanced'`.
Perform a 5-fold Stratified Cross-Validation on given `(X, y)` and return the mean ROC-AUC score.

**Q9 (15 Marks):** Explain how "Target Leakage" occurs in a customer churn prediction model if a feature `days_since_cancellation_survey` is included. How does this corrupt model performance when deployed?

---

### Section D: Enterprise Failure & Tuning Scenario (20 Marks)

**Q10 (20 Marks):** A Data Science Engineer builds a Random Forest model on 500,000 credit card applications to predict credit default. The model achieves 99.4% accuracy on training data, but drops to 84.1% on the test set.
1. Diagnose the exact machine learning problem (Bias vs. Variance). (6 marks)
2. Identify three specific hyperparameters of `RandomForestClassifier` that should be tuned to resolve this issue, explaining the mechanism of each. (14 marks)

---

# Assessment Solutions & Scoring Guide

### Section A Solutions
- **Q1:** **(c)** Random Forest Classifier (Tree splits depend strictly on rank ordering).
- **Q2:** **(b)** Diamond constraint intersection with loss contours on axes.
- **Q3:** **(b)** PR-AUC or F1-Score (Accuracy is meaningless on imbalanced data).
- **Q4:** **(b)** $\approx 63.2\%$ ($1 - 1/e$).
- **Q5:** **(b)** Dense, well-separated clusters.

### Section B Solutions
**Q6 Solution:**
1. $z = -2.0 + 0.5(x_1) \implies \hat{p} = \frac{1}{1 + e^{-(-2.0 + 0.5 x_1)}}$.
2. For $x_1 = 4.0$: $z = -2.0 + 0.5(4.0) = -2.0 + 2.0 = 0$.
   - $\hat{p} = \frac{1}{1 + e^0} = \frac{1}{1 + 1} = \frac{1}{2} = 0.50$ (50%).
3. With threshold $0.5$, boundary cases typically default to Class 1 (or depend on implementation; $\hat{p} \ge 0.5 \to$ Class 1).

**Q7 Solution:**
- Total samples $N = 40 + 10 = 50$.
- $p_A = \frac{40}{50} = 0.8$, $p_B = \frac{10}{50} = 0.2$.
- $\text{Gini} = 1 - (p_A^2 + p_B^2) = 1 - (0.8^2 + 0.2^2) = 1 - (0.64 + 0.04) = 1 - 0.68 = 0.32$.

### Section C Solutions
**Q8 Solution:**
```python
from sklearn.pipeline import Pipeline
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import StratifiedKFold, cross_val_score
import numpy as np

def evaluate_leak_free_pipeline(X, y):
    pipe = Pipeline([
        ('imputer', SimpleImputer(strategy='median')),
        ('scaler', StandardScaler()),
        ('classifier', LogisticRegression(penalty='l2', class_weight='balanced', max_iter=1000))
    ])
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    scores = cross_val_score(pipe, X, y, cv=cv, scoring='roc_auc')
    return np.mean(scores)
```

**Q9 Solution:**
- `days_since_cancellation_survey` is populated only *after* a customer initiates account cancellation. In production, when the model scores active, current customers to prevent churn, this feature does not yet exist (or is NULL).
- In offline training, the model learns that having a non-null survey date guarantees churn ($100\%$ correlation), ignoring legitimate early-warning signals. In production, the model fails completely.

### Section D Solutions
**Q10 Solution:**
1. **Diagnosis:** **Severe Overfitting (High Variance)**. The trees were allowed to grow unconstrained, memorizing training noise and failing to generalize to unseen test instances.
2. **Hyperparameter Remediations:**
   - **`max_depth`:** Constrain maximum tree depth (e.g., set to 8–12) to prevent trees from isolating single noisy training samples.
   - **`min_samples_leaf`:** Increase minimum leaf samples (e.g., set to 10–50), forcing leaves to contain generalizable sample sizes.
   - **`max_features`:** Reduce the feature subspace fraction (e.g., set to `'sqrt'` or `0.3`) to increase tree decorrelation across the ensemble.
