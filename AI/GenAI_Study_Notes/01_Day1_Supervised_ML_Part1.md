# Day 1: Supervised ML — Linear & Logistic Regression; Decision Trees & Random Forests

---

## Topics Covered
1. **Linear Regression** (Simple & Multiple)
2. **Logistic Regression** (Binary & Multiclass)
3. **Decision Trees** (Classification & Regression)
4. **Random Forests** (Ensemble / Bagging)

---

## 1. LINEAR REGRESSION

### 1.1 Definition & Fundamentals

**Definition:** Linear Regression is a supervised learning algorithm that models the relationship between one or more independent variables (features) and a continuous dependent variable (target) by fitting a linear equation.

**Plain English:** It draws the "best-fit straight line" through your data points so you can predict a numerical value (e.g., house price) from input features (e.g., area, bedrooms).

**What problem it solves:** Predicting a continuous outcome variable from input features. Examples: predicting salary from experience, house prices from square footage, temperature from altitude.

**Types:**
- **Simple Linear Regression:** One feature → one target. `y = mx + b`
- **Multiple Linear Regression:** Multiple features → one target. `y = b₀ + b₁x₁ + b₂x₂ + ... + bₙxₙ`

### 1.2 Mathematical Foundation

**Hypothesis function:**
```
ŷ = h(x) = θ₀ + θ₁x₁ + θ₂x₂ + ... + θₙxₙ
```
In matrix form: **ŷ = Xθ**

Where:
- **ŷ** = predicted value
- **θ₀** = bias/intercept (value of y when all features are 0)
- **θ₁...θₙ** = weights/coefficients (how much each feature contributes)
- **x₁...xₙ** = input features
- **X** = feature matrix (m × n), **θ** = parameter vector (n × 1)

**Cost Function (Mean Squared Error):**
```
J(θ) = (1/2m) Σᵢ₌₁ᵐ (ŷᵢ - yᵢ)²
```
Where:
- **m** = number of training examples
- **ŷᵢ** = predicted value for example i
- **yᵢ** = actual value for example i
- The factor 1/2 is for mathematical convenience (simplifies derivative)

**Why MSE?** It penalizes larger errors more heavily (squared), is differentiable everywhere (needed for gradient descent), and has a single global minimum (convex function for linear regression).

**Gradient Descent Update Rule:**
```
θⱼ := θⱼ - α · (∂J/∂θⱼ)

∂J/∂θⱼ = (1/m) Σᵢ₌₁ᵐ (ŷᵢ - yᵢ) · xⱼ⁽ⁱ⁾
```
Where:
- **α** = learning rate (step size; too large → overshoots, too small → slow convergence)
- All parameters are updated **simultaneously**

**Normal Equation (Closed-Form Solution):**
```
θ = (XᵀX)⁻¹ Xᵀy
```
- **No iteration needed**, gives exact solution.
- **Limitation:** O(n³) for matrix inversion — impractical when n (features) is very large (>10,000).
- **Fails if XᵀX is singular** (features are linearly dependent or n > m).

### 1.3 Gradient Descent vs Normal Equation

| Aspect | Gradient Descent | Normal Equation |
|--------|-----------------|-----------------|
| Iterative? | Yes | No |
| Learning rate needed? | Yes (α) | No |
| Feature scaling needed? | Yes | No |
| Complexity | O(kn²) per iteration | O(n³) for inversion |
| Large n (features) | Works well | Slow (matrix inversion) |
| Large m (samples) | Scales well | Scales well |
| Multiple minima risk | No (convex for linear) | N/A |

### 1.4 Assumptions of Linear Regression

1. **Linearity:** Relationship between X and y is linear.
2. **Independence:** Observations are independent of each other.
3. **Homoscedasticity:** Constant variance of residuals across all levels of X.
4. **Normality of residuals:** Errors are normally distributed (for valid inference, not prediction).
5. **No multicollinearity:** Features are not highly correlated with each other.
6. **No autocorrelation:** Residuals are not correlated with each other (especially for time series).

**What happens when assumptions are violated:**
- Violated linearity → biased predictions, use polynomial features or non-linear models
- Multicollinearity → unstable coefficients, large standard errors → use VIF, PCA, or regularization
- Heteroscedasticity → inefficient estimates → use weighted least squares or log transform

### 1.5 Regularization in Linear Regression

**Why needed?** To prevent overfitting, handle multicollinearity, and improve generalization.

**Ridge Regression (L2):**
```
J(θ) = (1/2m) Σ(ŷᵢ - yᵢ)² + λ Σⱼ₌₁ⁿ θⱼ²
```
- Adds **sum of squared weights** as penalty.
- **Shrinks coefficients** but never makes them exactly zero.
- Good when many features contribute a little.

**Lasso Regression (L1):**
```
J(θ) = (1/2m) Σ(ŷᵢ - yᵢ)² + λ Σⱼ₌₁ⁿ |θⱼ|
```
- Adds **sum of absolute weights** as penalty.
- **Can shrink coefficients to exactly zero** → automatic feature selection.
- Good when you suspect only a few features are important.

**Elastic Net:**
```
J(θ) = (1/2m) Σ(ŷᵢ - yᵢ)² + λ₁ Σ|θⱼ| + λ₂ Σθⱼ²
```
- Combines L1 and L2. Best of both worlds.

| Aspect | Ridge (L2) | Lasso (L1) | Elastic Net |
|--------|-----------|-----------|-------------|
| Penalty | Sum of θ² | Sum of |θ| | Both |
| Feature selection | No | Yes | Yes |
| Handles multicollinearity | Yes | Partially | Yes |
| Solution uniqueness | Always unique | May not be unique | Unique |
| When to use | All features matter | Sparse features | Groups of correlated features |

### 1.6 Evaluation Metrics for Regression

| Metric | Formula | Interpretation |
|--------|---------|----------------|
| **MAE** | (1/m) Σ\|yᵢ - ŷᵢ\| | Average absolute error. Robust to outliers. |
| **MSE** | (1/m) Σ(yᵢ - ŷᵢ)² | Average squared error. Penalizes large errors more. |
| **RMSE** | √MSE | Same unit as target. Most commonly used. |
| **R² Score** | 1 - (SS_res / SS_tot) | Proportion of variance explained. 1 = perfect, 0 = baseline, <0 = worse than mean. |
| **Adjusted R²** | 1 - [(1-R²)(m-1)/(m-n-1)] | Penalizes adding irrelevant features. Use for multiple regression. |

Where: SS_res = Σ(yᵢ - ŷᵢ)², SS_tot = Σ(yᵢ - ȳ)²

### 1.7 Implementation

```python
import numpy as np
from sklearn.linear_model import LinearRegression, Ridge, Lasso
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, r2_score
from sklearn.preprocessing import StandardScaler

# --- Sample Data ---
np.random.seed(42)
X = np.random.rand(100, 3)  # 100 samples, 3 features
y = 3*X[:, 0] + 5*X[:, 1] - 2*X[:, 2] + np.random.randn(100) * 0.5

# --- Train/Test Split ---
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# --- Feature Scaling (important for regularization & gradient descent) ---
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)  # fit on train, transform train
X_test_scaled = scaler.transform(X_test)          # only transform test (no fit!)

# --- Linear Regression ---
lr = LinearRegression()
lr.fit(X_train, y_train)
y_pred = lr.predict(X_test)

print(f"Coefficients: {lr.coef_}")       # [≈3.0, ≈5.0, ≈-2.0]
print(f"Intercept: {lr.intercept_:.4f}")
print(f"R² Score: {r2_score(y_test, y_pred):.4f}")
print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.4f}")

# --- Ridge Regression ---
ridge = Ridge(alpha=1.0)  # alpha = λ (regularization strength)
ridge.fit(X_train_scaled, y_train)
print(f"Ridge R²: {ridge.score(X_test_scaled, y_test):.4f}")

# --- Lasso Regression ---
lasso = Lasso(alpha=0.1)
lasso.fit(X_train_scaled, y_train)
print(f"Lasso R²: {lasso.score(X_test_scaled, y_test):.4f}")
print(f"Lasso Coefficients: {lasso.coef_}")  # Some may be exactly 0
```

**Key Implementation Notes:**
- `fit_transform` on training data, `transform` only on test data (prevents data leakage).
- `alpha` in sklearn = λ in formulas.
- Feature scaling is **essential** for Ridge/Lasso (penalty is scale-dependent).
- `LinearRegression` uses Normal Equation internally (via SVD decomposition).

### 1.8 Worked Example (Manual Calculation)

**Simple Linear Regression:** Given points (1,2), (2,4), (3,5), (4,4), (5,5):

```
n=5, Σx=15, Σy=20, Σxy=67, Σx²=55

θ₁ (slope) = (nΣxy - ΣxΣy) / (nΣx² - (Σx)²)
            = (5×67 - 15×20) / (5×55 - 225)
            = (335 - 300) / (275 - 225) = 35/50 = 0.7

θ₀ (intercept) = ȳ - θ₁x̄ = 4 - 0.7×3 = 1.9

Equation: ŷ = 1.9 + 0.7x

Predict for x=6: ŷ = 1.9 + 0.7(6) = 6.1
```

---

## 2. LOGISTIC REGRESSION

### 2.1 Definition & Fundamentals

**Definition:** Logistic Regression is a supervised classification algorithm that models the probability of a binary outcome using the **sigmoid (logistic) function**. Despite its name, it is used for **classification**, not regression.

**Plain English:** It predicts the probability that an input belongs to a particular class (e.g., spam or not spam) by squashing a linear combination of features through a sigmoid function to output values between 0 and 1.

**Why not use Linear Regression for classification?**
- Linear regression outputs can be < 0 or > 1 (not valid probabilities).
- The decision boundary is sensitive to outliers.
- MSE loss for classification is non-convex → many local minima.

### 2.2 Mathematical Foundation

**Sigmoid Function:**
```
σ(z) = 1 / (1 + e⁻ᶻ)
```
Properties:
- Output range: (0, 1) — interpretable as probability.
- σ(0) = 0.5, σ(+∞) → 1, σ(-∞) → 0.
- **Derivative:** σ'(z) = σ(z) · (1 - σ(z)) — elegant, used in backprop.

**Hypothesis:**
```
h(x) = σ(θᵀx) = 1 / (1 + e^(-θᵀx))
```
Output = P(y=1 | x; θ) = probability that class is 1.

**Decision Rule:**
```
ŷ = 1  if h(x) ≥ 0.5  (i.e., θᵀx ≥ 0)
ŷ = 0  if h(x) < 0.5   (i.e., θᵀx < 0)
```
**Decision boundary:** θᵀx = 0 (a hyperplane in feature space).

**Cost Function (Binary Cross-Entropy / Log Loss):**
```
J(θ) = -(1/m) Σᵢ₌₁ᵐ [yᵢ · log(ŷᵢ) + (1 - yᵢ) · log(1 - ŷᵢ)]
```
**Why not MSE?** For sigmoid output, MSE creates a non-convex loss surface with many local minima. Log loss is **convex** → guaranteed global minimum.

**Intuition of log loss:**
- When y=1: cost = -log(ŷ). If ŷ→1, cost→0. If ŷ→0, cost→∞. (Heavily penalizes confident wrong predictions.)
- When y=0: cost = -log(1-ŷ). If ŷ→0, cost→0. If ŷ→1, cost→∞.

**Gradient (same form as linear regression, different h(x)):**
```
∂J/∂θⱼ = (1/m) Σᵢ₌₁ᵐ (h(xᵢ) - yᵢ) · xⱼ⁽ⁱ⁾
```

### 2.3 Multiclass Logistic Regression

**One-vs-Rest (OvR):** Train K binary classifiers (one per class). For class k, treat class k as positive and all others as negative. Predict the class with highest probability.

**Softmax Regression (Multinomial Logistic Regression):**
```
P(y = k | x) = e^(θₖᵀx) / Σⱼ₌₁ᴷ e^(θⱼᵀx)
```
- Outputs a probability distribution over K classes.
- Loss: **Categorical Cross-Entropy.**
- Sklearn: `LogisticRegression(multi_class='multinomial', solver='lbfgs')`

### 2.4 Threshold Tuning & Class Imbalance

- **Default threshold = 0.5** — not always optimal.
- **Lowering threshold** (e.g., 0.3): Increases recall (catches more positives), decreases precision (more false positives). Good when missing positives is costly (e.g., cancer detection).
- **Raising threshold** (e.g., 0.7): Increases precision, decreases recall. Good when false positives are costly (e.g., spam filter).
- **How to choose:** Use **Precision-Recall curve** or **ROC curve** to find the optimal threshold for your business requirement.

### 2.5 Implementation

```python
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, confusion_matrix, roc_auc_score

# Load data
data = load_breast_cancer()
X_train, X_test, y_train, y_test = train_test_split(
    data.data, data.target, test_size=0.2, random_state=42
)

# Train
model = LogisticRegression(max_iter=10000, solver='lbfgs')
model.fit(X_train, y_train)

# Predict
y_pred = model.predict(X_test)
y_prob = model.predict_proba(X_test)[:, 1]  # Probabilities for class 1

# Evaluate
print(classification_report(y_test, y_pred))
print(f"AUC-ROC: {roc_auc_score(y_test, y_prob):.4f}")
print(f"Confusion Matrix:\n{confusion_matrix(y_test, y_pred)}")

# Coefficients interpretation
# Positive coefficient → feature increases log-odds of class 1
# Negative coefficient → feature decreases log-odds of class 1
```

### 2.6 Linear vs Logistic Regression

| Aspect | Linear Regression | Logistic Regression |
|--------|------------------|-------------------|
| Task | Regression (continuous) | Classification (discrete) |
| Output | Any real number | Probability [0, 1] |
| Function | Linear: θᵀx | Sigmoid: σ(θᵀx) |
| Loss | MSE | Binary Cross-Entropy |
| Decision boundary | N/A | Hyperplane (θᵀx = 0) |
| Assumptions | Linear, normal residuals | Linear decision boundary |
| Regularization | Ridge, Lasso | L1, L2 (via `penalty` param) |

---

## 3. DECISION TREES

### 3.1 Definition & Fundamentals

**Definition:** A Decision Tree is a supervised learning algorithm that partitions the feature space into rectangular regions by recursively splitting data based on feature values, forming a tree-like structure of decisions.

**Plain English:** It creates a flowchart of yes/no questions about your data's features, splitting the data at each step until it reaches a prediction (leaf node).

**Key Terminology:**
- **Root Node:** The topmost node; represents the entire dataset.
- **Internal/Decision Node:** A node that splits on a feature.
- **Leaf/Terminal Node:** A node that gives the final prediction.
- **Branch:** A path from one node to another.
- **Depth:** The length of the longest path from root to a leaf.
- **Splitting:** Dividing a node into sub-nodes based on a condition.
- **Pruning:** Removing branches to reduce overfitting.

### 3.2 How Splitting Works

**Key question:** Which feature and which value should we split on at each node?

**For Classification — Impurity Measures:**

**Gini Impurity:**
```
Gini(t) = 1 - Σₖ₌₁ᴷ pₖ²
```
Where pₖ = proportion of class k in node t.
- Gini = 0 → pure node (all same class).
- Gini = 0.5 → maximum impurity for binary (50-50 split).
- **Preferred by CART algorithm** (sklearn default).

**Entropy (Information Gain):**
```
Entropy(t) = -Σₖ₌₁ᴷ pₖ · log₂(pₖ)
```
- Entropy = 0 → pure node.
- Entropy = 1 → max impurity for binary.

**Information Gain:**
```
IG(feature) = Entropy(parent) - Σ (|child_j| / |parent|) × Entropy(child_j)
```
**Choose the feature that maximizes Information Gain** (= maximizes reduction in entropy).

**For Regression — Variance Reduction:**
```
Split criterion = Var(parent) - Σ (|child_j| / |parent|) × Var(child_j)
```
The leaf prediction = **mean** of target values in that region.

### 3.3 Worked Example

Dataset: Predict "Play Tennis" from Outlook, Temperature, Humidity, Wind.

```
Total: 14 examples (9 Yes, 5 No)
Entropy(S) = -(9/14)log₂(9/14) - (5/14)log₂(5/14) = 0.940

Split on Outlook:
  Sunny:     5 examples (2 Yes, 3 No) → Entropy = 0.971
  Overcast:  4 examples (4 Yes, 0 No) → Entropy = 0.0 ← Pure!
  Rainy:     5 examples (3 Yes, 2 No) → Entropy = 0.971

IG(Outlook) = 0.940 - (5/14)(0.971) - (4/14)(0) - (5/14)(0.971) = 0.247

Compare with IG of other features → pick the highest IG → Outlook becomes root.
```

### 3.4 Overfitting & Regularization (Pruning)

**Decision Trees overfit easily** — they can grow until every leaf is pure (memorizing the training data).

**Pre-pruning (stop growing early):**
| Hyperparameter | What it does |
|---------------|-------------|
| `max_depth` | Maximum depth of tree |
| `min_samples_split` | Minimum samples needed to split a node |
| `min_samples_leaf` | Minimum samples required in a leaf |
| `max_features` | Number of features to consider at each split |
| `max_leaf_nodes` | Maximum number of leaf nodes |

**Post-pruning (Cost-Complexity Pruning):**
```
Total cost = Training error + α × |T|
```
Where |T| = number of leaves, α = complexity parameter (ccp_alpha in sklearn). Larger α → more pruning.

### 3.5 Decision Trees for Regression (CART)

- Splits feature space into rectangular regions.
- Prediction = mean of target values in that region.
- Split criterion: minimize MSE (or MAE) in resulting children.
- Same pruning techniques apply.

### 3.6 Advantages & Disadvantages

| Advantages | Disadvantages |
|-----------|--------------|
| Easy to interpret & visualize | Prone to overfitting |
| No feature scaling needed | Unstable (small data changes → different tree) |
| Handles both numerical & categorical | Biased toward features with more levels |
| Handles non-linear relationships | Greedy splits (locally optimal, not globally) |
| Built-in feature importance | High variance |
| Fast inference | Cannot extrapolate beyond training data range |

### 3.7 Implementation

```python
from sklearn.tree import DecisionTreeClassifier, DecisionTreeRegressor, export_text
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

# Classification
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42
)

dt = DecisionTreeClassifier(
    criterion='gini',           # 'gini' or 'entropy'
    max_depth=3,                # Pre-pruning
    min_samples_split=5,        # Need ≥5 samples to split
    min_samples_leaf=2,         # Each leaf needs ≥2 samples
    random_state=42
)
dt.fit(X_train, y_train)
print(f"Accuracy: {dt.score(X_test, y_test):.4f}")
print(f"Feature Importances: {dict(zip(iris.feature_names, dt.feature_importances_))}")

# Print tree structure
print(export_text(dt, feature_names=iris.feature_names))

# Cost-Complexity Pruning
path = dt.cost_complexity_pruning_path(X_train, y_train)
ccp_alphas = path.ccp_alphas  # Array of alpha values
# Try different alphas, pick best via cross-validation
```

---

## 4. RANDOM FORESTS

### 4.1 Definition & Fundamentals

**Definition:** Random Forest is an **ensemble learning** method that constructs multiple decision trees during training and outputs the **mode** (classification) or **mean** (regression) of the individual trees' predictions.

**Plain English:** Instead of relying on one decision tree (which can overfit), build many trees on random subsets of data and features, then let them "vote" on the answer. The wisdom of the crowd reduces errors.

**Core Idea — Bagging (Bootstrap Aggregating):**
1. Create B bootstrap samples (random sampling **with replacement**) from the training data.
2. Train one decision tree on each bootstrap sample.
3. Aggregate predictions: **majority vote** (classification) or **average** (regression).

**What Random Forest adds beyond Bagging:**
At each split, consider only a **random subset of features** (typically √n for classification, n/3 for regression). This **decorrelates the trees** — if there's one very strong feature, not every tree will split on it first.

### 4.2 Why Random Forest Works — Bias-Variance Tradeoff

- Individual decision trees have **low bias** (flexible) but **high variance** (overfit).
- Averaging many trees **reduces variance** while keeping bias low.
- The feature randomization ensures trees are **diverse** (not all identical copies).
- Mathematically: Var(average of B trees) = σ² × ρ/B + σ²(1-ρ) where ρ = correlation between trees. Lower ρ → lower variance.

### 4.3 Out-of-Bag (OOB) Error

Each bootstrap sample uses ~63.2% of training data (on average). The remaining ~36.8% is **Out-of-Bag** data.
- Use OOB data to evaluate each tree on data it hasn't seen.
- **OOB score** ≈ cross-validation accuracy without needing a separate validation set.
- Set `oob_score=True` in sklearn.

### 4.4 Feature Importance

Two methods:

1. **Mean Decrease in Impurity (MDI):** Sum of impurity reductions (Gini/Entropy) weighted by the number of samples reaching that node, across all trees. **Default in sklearn.** Bias: favors high-cardinality features.

2. **Permutation Importance:** Randomly shuffle one feature → measure accuracy drop. More reliable, slower. Use `sklearn.inspection.permutation_importance`.

### 4.5 Key Hyperparameters

| Parameter | Description | Typical Values |
|-----------|-------------|----------------|
| `n_estimators` | Number of trees | 100-500 (more = better, diminishing returns) |
| `max_depth` | Maximum tree depth | None (grow fully) or 10-30 |
| `max_features` | Features per split | `'sqrt'` (classification), `'log2'`, or fraction |
| `min_samples_split` | Min samples to split | 2-10 |
| `min_samples_leaf` | Min samples in leaf | 1-5 |
| `bootstrap` | Use bootstrap sampling? | True (default) |
| `oob_score` | Calculate OOB score? | False (set True for validation) |
| `n_jobs` | Parallel trees | -1 (use all cores) |

### 4.6 Implementation

```python
from sklearn.ensemble import RandomForestClassifier, RandomForestRegressor
from sklearn.model_selection import cross_val_score, GridSearchCV
from sklearn.datasets import load_breast_cancer

data = load_breast_cancer()
X_train, X_test, y_train, y_test = train_test_split(
    data.data, data.target, test_size=0.2, random_state=42
)

# Train
rf = RandomForestClassifier(
    n_estimators=200,
    max_depth=None,         # Grow full trees
    max_features='sqrt',    # √n features per split
    min_samples_leaf=1,
    oob_score=True,
    n_jobs=-1,
    random_state=42
)
rf.fit(X_train, y_train)

print(f"Test Accuracy: {rf.score(X_test, y_test):.4f}")
print(f"OOB Score: {rf.oob_score_:.4f}")

# Feature Importance
importances = rf.feature_importances_
top_features = sorted(zip(data.feature_names, importances), key=lambda x: -x[1])[:5]
print("Top 5 Features:", top_features)

# Cross-Validation
cv_scores = cross_val_score(rf, data.data, data.target, cv=5, scoring='accuracy')
print(f"CV Accuracy: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

# Hyperparameter Tuning
param_grid = {
    'n_estimators': [100, 200, 300],
    'max_depth': [None, 10, 20],
    'max_features': ['sqrt', 'log2'],
}
grid = GridSearchCV(RandomForestClassifier(random_state=42), param_grid, cv=3, n_jobs=-1)
grid.fit(X_train, y_train)
print(f"Best Params: {grid.best_params_}")
```

### 4.7 Decision Tree vs Random Forest

| Aspect | Decision Tree | Random Forest |
|--------|--------------|---------------|
| Overfitting | High risk | Low (averaging reduces variance) |
| Bias | Low | Low |
| Variance | High | Low |
| Interpretability | High (single tree) | Lower (ensemble) |
| Training time | Fast | Slower (many trees) |
| Feature importance | Available | More robust (averaged) |
| Handles noise | Poorly | Well |
| Missing values | Handled (sklearn impute) | Handled |

### 4.8 Bagging vs Boosting (Preview)

| Aspect | Bagging (Random Forest) | Boosting (e.g., XGBoost) |
|--------|------------------------|-------------------------|
| Strategy | Parallel, independent trees | Sequential, correcting errors |
| Goal | Reduce variance | Reduce bias |
| Tree correlation | Low (by design) | N/A (sequential) |
| Overfitting risk | Lower | Higher (if not tuned) |
| Performance | Good | Often better (competitive) |
| Sensitivity to noise | Robust | More sensitive |

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Not scaling features** for Linear/Logistic Regression with regularization.
2. **Using MSE loss for classification** — always use cross-entropy for classification.
3. **Not tuning Decision Tree depth** — leads to memorization (100% train, poor test).
4. **Confusing "Logistic Regression" with regression** — it's a classifier.
5. **Ignoring multicollinearity** in Linear Regression — coefficients become unreliable.
6. **Using accuracy on imbalanced data** — use F1, AUC-ROC instead.
7. **Not using cross-validation** — single train/test split can be misleading.
8. **Data leakage from scaling** — never fit_transform on test data.

---

## Interview Preparation

### Direct Interview Answers

**Q: What is Linear Regression?**
> "Linear Regression models the relationship between input features and a continuous target variable by fitting a linear equation that minimizes the sum of squared errors. It uses gradient descent or the normal equation to find optimal weights. It assumes linearity, independent errors, and constant variance of residuals."

**Q: Why can't we use Linear Regression for classification?**
> "Linear Regression outputs unbounded values, which aren't valid probabilities. The decision boundary is sensitive to outliers, and the MSE loss function for binary targets creates a non-convex surface. Logistic Regression solves this by applying a sigmoid function to bound output between 0 and 1, and uses log-loss which is convex."

**Q: Explain the Random Forest algorithm.**
> "Random Forest is a bagging ensemble of decision trees. Each tree is trained on a bootstrap sample (random subset with replacement) and at each split, only a random subset of features is considered. The final prediction is the majority vote for classification or mean for regression. This reduces variance by decorrelating the trees and leveraging the law of large numbers."

### Interview Q&A Bank

**Basic:**
1. **Q:** What is the cost function of Linear Regression? **A:** Mean Squared Error: J(θ) = (1/2m) Σ(ŷ - y)². It measures average squared difference between predicted and actual values.

2. **Q:** What is the sigmoid function? **A:** σ(z) = 1/(1+e⁻ᶻ). It maps any real number to (0,1), used as the activation in Logistic Regression to output probabilities.

3. **Q:** What is Gini Impurity? **A:** Gini = 1 - Σpₖ², measures the probability of incorrectly classifying a randomly chosen element. Used by CART to choose splits.

**Intermediate:**
4. **Q:** Difference between Ridge and Lasso? **A:** Ridge adds L2 penalty (sum of squared weights), shrinks coefficients but doesn't zero them. Lasso adds L1 penalty (sum of absolute weights), can zero out coefficients for automatic feature selection. Use Ridge when all features matter, Lasso for sparse models.

5. **Q:** Why does Random Forest use random feature subsets? **A:** To decorrelate the trees. If one feature is very strong, all bagged trees would split on it first, making them highly correlated. Random feature selection forces diversity, reducing ensemble variance.

6. **Q:** How does a Decision Tree handle continuous features? **A:** It evaluates every possible threshold for each continuous feature, calculates the impurity reduction (Gini or Entropy) for each, and picks the threshold that maximizes information gain.

**Advanced:**
7. **Q:** What is the OOB error in Random Forest and why is it useful? **A:** Each bootstrap sample excludes ~36.8% of training data. These out-of-bag samples are used to evaluate each tree. Aggregating OOB predictions gives an unbiased error estimate without needing cross-validation, saving computation.

8. **Q:** Derive the gradient of logistic regression's cost function. **A:** Starting from J = -[y·log(σ(z)) + (1-y)·log(1-σ(z))], using chain rule and σ'(z) = σ(z)(1-σ(z)): ∂J/∂θⱼ = (σ(θᵀx) - y) · xⱼ = (ŷ - y) · xⱼ. Elegant: same form as linear regression but with sigmoid activation.

9. **Q:** When would Random Forest perform poorly? **A:** On very high-dimensional sparse data (e.g., text with 100K features), extrapolation tasks (can't predict beyond training range), when interpretability is critical, or when the dataset is very small (not enough for diverse bootstraps). Gradient boosting or linear models often perform better in these cases.

---

## Rapid Revision Summary

- **Linear Regression:** ŷ = θᵀx, minimizes MSE, uses gradient descent or normal equation. Regularize with Ridge (L2) or Lasso (L1).
- **Logistic Regression:** σ(θᵀx) → probability, uses log loss, decision boundary at θᵀx=0. Threshold tuning for precision-recall tradeoff.
- **Decision Tree:** Recursive splitting using Gini/Entropy (classification) or variance reduction (regression). Prune to prevent overfitting.
- **Random Forest:** Bagging + random feature selection → many decorrelated trees → majority vote/mean. OOB error for validation.

---

## Self-Assessment

1. Write the cost function for linear regression and explain each term.
2. Why is log loss used instead of MSE for logistic regression? Explain with convexity.
3. Calculate Gini impurity for a node with 70% class A and 30% class B.
4. Explain why feature scaling is important for Ridge but not for Decision Trees.
5. What is the difference between Information Gain and Gini Impurity? When would you prefer one?
6. How does Random Forest reduce the variance of Decision Trees? Explain both mechanisms.
7. Given a dataset with 100 features where only 5 are relevant, would you use Ridge or Lasso? Why?

**Answers:**
1. J(θ) = (1/2m) Σ(ŷᵢ - yᵢ)². m = samples, ŷᵢ = predicted, yᵢ = actual. 1/2 for derivative convenience.
2. Sigmoid + MSE → non-convex surface with many local minima. Log loss is convex → guaranteed convergence to global minimum.
3. Gini = 1 - (0.7² + 0.3²) = 1 - (0.49 + 0.09) = 0.42
4. Ridge penalty depends on coefficient magnitudes — large-scale features dominate. Trees split on thresholds, scale doesn't affect comparison.
5. Both measure impurity; Gini is faster to compute, Entropy considers log probabilities. Gini favors equal-sized partitions; Entropy favors purer subsets. In practice, results are very similar.
6. (a) Bagging: averaging reduces variance by factor of 1/B. (b) Feature randomization: decorrelates trees, reducing effective correlation ρ in Var formula.
7. Lasso — it can set 95 irrelevant coefficients to exactly zero, performing automatic feature selection.

---

## Recall Checklist

- [ ] Define and write the hypothesis function for Linear Regression
- [ ] Write the MSE cost function and compute its gradient
- [ ] Explain Normal Equation, its formula, and when it fails
- [ ] Distinguish Ridge, Lasso, and Elastic Net
- [ ] Write the sigmoid function and its derivative
- [ ] Write the binary cross-entropy loss function
- [ ] Explain why log loss is preferred over MSE for classification
- [ ] Calculate Gini Impurity and Information Gain by hand
- [ ] List Decision Tree hyperparameters for controlling overfitting
- [ ] Explain the Random Forest algorithm step by step
- [ ] Explain OOB error and its relationship to cross-validation
- [ ] Compare Bagging vs Boosting
- [ ] Implement LinearRegression, LogisticRegression, DecisionTreeClassifier, RandomForestClassifier in sklearn
