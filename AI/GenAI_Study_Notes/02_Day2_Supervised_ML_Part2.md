# Day 2: Supervised ML — SVM & K-NN; Model Evaluation Metrics

---

## Topics Covered
1. **Support Vector Machines (SVM)** — Linear, Kernel, Soft Margin
2. **K-Nearest Neighbors (K-NN)**
3. **Model Evaluation Metrics** — Accuracy, Precision, Recall, F1, AUC-ROC, Confusion Matrix

---

## 1. SUPPORT VECTOR MACHINES (SVM)

### 1.1 Definition & Fundamentals

**Definition:** SVM is a supervised learning algorithm that finds the **optimal hyperplane** which separates data points of different classes with the **maximum margin**. It can handle both linear and non-linear classification (and regression via SVR).

**Plain English:** Imagine drawing a line (or surface in higher dimensions) between two groups of points, positioning it so it's as far as possible from the nearest points of both groups. Those nearest points are the "support vectors."

**Key Terminology:**
- **Hyperplane:** A decision boundary that separates classes. In 2D, it's a line; in 3D, a plane; in nD, a hyperplane.
- **Margin:** The distance between the hyperplane and the nearest data points from each class. SVM maximizes this.
- **Support Vectors:** The data points closest to the hyperplane. These are the critical elements — the hyperplane is determined entirely by them.
- **Hard Margin:** No data point is allowed inside the margin or on the wrong side. Only works for linearly separable data.
- **Soft Margin:** Allows some misclassifications via slack variables. Handles non-separable data.

### 1.2 Mathematical Foundation

**Linear SVM — Hard Margin:**

The hyperplane: **wᵀx + b = 0**

Classification rule:
```
wᵀx + b ≥ +1   → class +1
wᵀx + b ≤ -1   → class -1
```

The margin width = **2 / ||w||**

**Optimization Problem:**
```
Minimize:    (1/2) ||w||²
Subject to:  yᵢ(wᵀxᵢ + b) ≥ 1  for all i
```
Minimizing ||w||² maximizes the margin 2/||w||.

**Soft Margin SVM (C-SVM):**

Introduce **slack variables ξᵢ** to allow misclassifications:
```
Minimize:    (1/2) ||w||² + C Σᵢ ξᵢ
Subject to:  yᵢ(wᵀxᵢ + b) ≥ 1 - ξᵢ,  ξᵢ ≥ 0
```

Where:
- **C** = regularization parameter (penalty for misclassification)
- **Large C:** Low tolerance for misclassification → narrow margin → may overfit
- **Small C:** High tolerance → wide margin → may underfit
- **ξᵢ = 0:** Point is correctly classified outside margin
- **0 < ξᵢ < 1:** Point is inside margin but on correct side
- **ξᵢ ≥ 1:** Point is misclassified

### 1.3 The Kernel Trick

**Problem:** Real-world data is often not linearly separable in the original feature space.

**Solution:** Map data to a higher-dimensional space where it becomes linearly separable, then find a hyperplane there.

**Key Insight (Kernel Trick):** Instead of explicitly computing the transformation φ(x), compute only the **dot product** in the higher-dimensional space using a **kernel function** K(xᵢ, xⱼ) = φ(xᵢ)ᵀφ(xⱼ). This is computationally efficient because we never compute φ(x) directly.

**Common Kernels:**

| Kernel | Formula | Use Case |
|--------|---------|----------|
| **Linear** | K(x,y) = xᵀy | Linearly separable, high-dimensional (text) |
| **Polynomial** | K(x,y) = (γxᵀy + r)ᵈ | Moderate non-linearity |
| **RBF (Gaussian)** | K(x,y) = exp(-γ\|\|x-y\|\|²) | General purpose, most commonly used |
| **Sigmoid** | K(x,y) = tanh(γxᵀy + r) | Similar to neural network |

**RBF Kernel — γ Parameter:**
- **Large γ:** Each point has small influence → complex, wiggly boundary → overfitting
- **Small γ:** Each point has large influence → smooth boundary → underfitting
- γ is like the inverse of the radius of influence of each support vector.

### 1.4 SVM for Regression (SVR)

- Instead of maximizing margin around a hyperplane, SVR fits a tube of width ε around the regression line.
- Points inside the tube have zero loss; points outside are penalized.
- ε controls the tube width (more tolerance = wider tube = simpler model).

### 1.5 Advantages & Disadvantages

| Advantages | Disadvantages |
|-----------|--------------|
| Effective in high-dimensional spaces | Slow on large datasets O(n²) to O(n³) |
| Memory efficient (only support vectors matter) | Poor with noisy data / overlapping classes |
| Versatile via different kernels | Doesn't directly provide probability estimates |
| Works well when n_features > n_samples | Sensitive to feature scaling |
| Robust to overfitting in high dimensions | Kernel and hyperparameter selection is non-trivial |

### 1.6 Implementation

```python
from sklearn.svm import SVC, SVR
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.model_selection import GridSearchCV
from sklearn.datasets import load_iris

iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42
)

# SVM REQUIRES feature scaling (margin and kernel depend on distances)
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('svm', SVC(kernel='rbf', C=1.0, gamma='scale', probability=True))
])
pipe.fit(X_train, y_train)
print(f"Accuracy: {pipe.score(X_test, y_test):.4f}")

# Get probabilities (requires probability=True, uses Platt scaling internally)
probs = pipe.predict_proba(X_test)

# Hyperparameter Tuning
param_grid = {
    'svm__C': [0.1, 1, 10, 100],
    'svm__gamma': ['scale', 'auto', 0.01, 0.1],
    'svm__kernel': ['rbf', 'linear', 'poly']
}
grid = GridSearchCV(pipe, param_grid, cv=5, scoring='accuracy', n_jobs=-1)
grid.fit(X_train, y_train)
print(f"Best: {grid.best_params_}, Score: {grid.best_score_:.4f}")

# Support Vectors
svm_model = pipe.named_steps['svm']
print(f"Number of support vectors per class: {svm_model.n_support_}")
```

**Critical Implementation Notes:**
- **Always scale features** before SVM (distances matter).
- `gamma='scale'` sets γ = 1/(n_features × var(X)) — good default.
- `probability=True` enables `predict_proba()` but slows training (Platt scaling with internal 5-fold CV).

---

## 2. K-NEAREST NEIGHBORS (K-NN)

### 2.1 Definition & Fundamentals

**Definition:** K-NN is a **non-parametric, instance-based (lazy)** supervised learning algorithm that classifies a new data point based on the majority class of its K nearest neighbors in the training data.

**Plain English:** To predict a new point, look at the K closest training points and take a vote. No model is actually "trained" — the entire training set IS the model.

**Key Properties:**
- **Lazy learning:** No explicit training phase. All computation happens at prediction time.
- **Non-parametric:** Makes no assumptions about data distribution.
- **Instance-based:** Stores all training examples and compares new instances to them.

### 2.2 How K-NN Works

**Algorithm:**
1. Store all training data.
2. For a new point x:
   a. Compute the distance from x to every training point.
   b. Select the K nearest neighbors.
   c. **Classification:** Return the majority class among the K neighbors (optionally weighted by distance).
   d. **Regression:** Return the mean (or weighted mean) of the K neighbors' target values.

### 2.3 Distance Metrics

| Metric | Formula | When to Use |
|--------|---------|-------------|
| **Euclidean (L2)** | √(Σ(xᵢ - yᵢ)²) | Default, continuous features |
| **Manhattan (L1)** | Σ\|xᵢ - yᵢ\| | High-dimensional, sparse data |
| **Minkowski** | (Σ\|xᵢ - yᵢ\|ᵖ)^(1/p) | Generalized: p=1→Manhattan, p=2→Euclidean |
| **Hamming** | Proportion of differing features | Categorical features |
| **Cosine** | 1 - (x·y)/(||x||·||y||) | Text, when magnitude doesn't matter |

**Curse of Dimensionality:**
In high dimensions, distances become increasingly similar (all points are roughly equidistant). K-NN degrades severely in high-dimensional spaces.

### 2.4 Choosing K

- **K = 1:** Complex boundary, overfitting, sensitive to noise.
- **K = n (total samples):** Always predicts majority class — underfitting.
- **Odd K** for binary classification avoids ties.
- **Optimal K:** Use cross-validation. Typical range: 3–20.
- **Rule of thumb:** K ≈ √n (square root of training set size).

**Bias-Variance Tradeoff:**
| Small K | Large K |
|---------|---------|
| Low bias | High bias |
| High variance | Low variance |
| Complex boundary | Smooth boundary |
| Sensitive to noise | Robust to noise |

### 2.5 Weighted K-NN

- **Uniform:** All K neighbors vote equally. Default.
- **Distance-weighted:** Closer neighbors have more influence. Weight = 1/distance.
- Use `weights='distance'` in sklearn.
- Better when classes have unequal density.

### 2.6 Advantages & Disadvantages

| Advantages | Disadvantages |
|-----------|--------------|
| Simple, intuitive | Slow prediction O(mn) — compute distance to all n points |
| No training time | High memory (stores entire dataset) |
| Non-parametric (flexible) | Curse of dimensionality |
| Works for any number of classes | Sensitive to feature scaling |
| Good baseline model | Sensitive to irrelevant features |

### 2.7 Optimizations

- **KD-Tree:** Spatial data structure for fast nearest neighbor search in low dimensions. O(log n) per query. Degrades in high dimensions.
- **Ball Tree:** Works better than KD-Tree in higher dimensions. Uses hyperspheres.
- **Locality-Sensitive Hashing (LSH):** Approximate nearest neighbors. O(1) amortized. Used for very large datasets.
- Sklearn automatically selects the best algorithm: `algorithm='auto'`.

### 2.8 Implementation

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.model_selection import cross_val_score

# Always scale features for K-NN!
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('knn', KNeighborsClassifier(n_neighbors=5, weights='uniform', metric='minkowski', p=2))
])
pipe.fit(X_train, y_train)
print(f"Accuracy: {pipe.score(X_test, y_test):.4f}")

# Finding optimal K
k_range = range(1, 31)
k_scores = []
for k in k_range:
    knn = Pipeline([('scaler', StandardScaler()), ('knn', KNeighborsClassifier(n_neighbors=k))])
    scores = cross_val_score(knn, X_train, y_train, cv=5, scoring='accuracy')
    k_scores.append(scores.mean())

best_k = k_range[k_scores.index(max(k_scores))]
print(f"Best K: {best_k}, Accuracy: {max(k_scores):.4f}")
```

### 2.9 SVM vs K-NN

| Aspect | SVM | K-NN |
|--------|-----|------|
| Type | Parametric (learns w, b) | Non-parametric (lazy) |
| Training | Slow (optimization) | None (stores data) |
| Prediction | Fast (support vectors only) | Slow (distance to all points) |
| Memory | Low (support vectors) | High (entire dataset) |
| High dimensions | Excellent | Poor (curse of dimensionality) |
| Decision boundary | Optimal margin hyperplane | Complex, local |
| Feature scaling | Required | Required |

---

## 3. MODEL EVALUATION METRICS

### 3.1 Confusion Matrix

For binary classification (Positive = 1, Negative = 0):

```
                    Predicted
                  Pos       Neg
Actual  Pos    TP (True+)  FN (False-)
        Neg    FP (False+) TN (True-)
```

- **TP (True Positive):** Correctly predicted positive.
- **TN (True Negative):** Correctly predicted negative.
- **FP (False Positive / Type I Error):** Predicted positive, actually negative. "False alarm."
- **FN (False Negative / Type II Error):** Predicted negative, actually positive. "Missed detection."

### 3.2 Core Classification Metrics

**Accuracy:**
```
Accuracy = (TP + TN) / (TP + TN + FP + FN)
```
- Overall correctness. **Misleading on imbalanced data.**
- Example: 95% negative class → always predicting negative gives 95% accuracy.

**Precision (Positive Predictive Value):**
```
Precision = TP / (TP + FP)
```
- "Of everything I predicted positive, how many actually are?"
- High precision = few false positives.
- **Important when FP is costly:** spam filter (don't mark real email as spam), search results.

**Recall (Sensitivity / True Positive Rate):**
```
Recall = TP / (TP + FN)
```
- "Of all actual positives, how many did I catch?"
- High recall = few false negatives.
- **Important when FN is costly:** disease detection (don't miss sick patients), fraud detection.

**F1 Score:**
```
F1 = 2 × (Precision × Recall) / (Precision + Recall)
```
- **Harmonic mean** of precision and recall.
- Balanced metric when both FP and FN matter.
- Range: [0, 1]. F1 = 1 is perfect.
- **Why harmonic mean?** It punishes extreme imbalances. If either P or R is 0, F1 is 0.

**F-beta Score:**
```
Fβ = (1 + β²) × (P × R) / (β²P + R)
```
- β > 1: Weighs recall higher (e.g., F2 for disease detection).
- β < 1: Weighs precision higher (e.g., F0.5 for spam filtering).

### 3.3 Specificity and Related Metrics

**Specificity (True Negative Rate):**
```
Specificity = TN / (TN + FP)
```
- "Of all actual negatives, how many did I correctly identify?"

**False Positive Rate:**
```
FPR = FP / (FP + TN) = 1 - Specificity
```

### 3.4 ROC Curve and AUC

**ROC (Receiver Operating Characteristic) Curve:**
- Plots **TPR (Recall)** vs **FPR** at various classification thresholds.
- Each point on the curve = one threshold value.
- Diagonal line = random classifier (AUC = 0.5).

**AUC (Area Under the ROC Curve):**
- AUC = 1.0 → perfect classifier.
- AUC = 0.5 → random classifier (no discrimination).
- AUC < 0.5 → worse than random (flip predictions).
- **Interpretation:** Probability that the model ranks a random positive example higher than a random negative example.
- **Advantage:** Threshold-independent, works well for ranking problems.
- **Limitation:** Can be misleading on highly imbalanced datasets (prefer PR-AUC instead).

**Precision-Recall Curve:**
- Plots Precision vs Recall at various thresholds.
- **PR-AUC** is preferred over ROC-AUC for **imbalanced datasets**.
- Baseline = proportion of positive class (not 0.5).

### 3.5 Multi-Class Metrics

**Averaging strategies for precision, recall, F1:**

| Strategy | Formula | When to Use |
|----------|---------|-------------|
| **Macro** | Average per-class metrics equally | All classes equally important |
| **Weighted** | Weight by class support (n samples) | Account for class imbalance |
| **Micro** | Compute globally (total TP, FP, FN) | Overall performance |

```python
from sklearn.metrics import f1_score
f1_score(y_test, y_pred, average='macro')     # Equal weight per class
f1_score(y_test, y_pred, average='weighted')   # Weight by support
f1_score(y_test, y_pred, average='micro')      # Global TP/(TP+FP+FN)
```

### 3.6 Regression Metrics (Review from Day 1)

| Metric | Formula | Notes |
|--------|---------|-------|
| **MAE** | (1/m)Σ\|y-ŷ\| | Robust to outliers |
| **MSE** | (1/m)Σ(y-ŷ)² | Penalizes large errors |
| **RMSE** | √MSE | Same unit as target |
| **R²** | 1-(SS_res/SS_tot) | Proportion of variance explained |
| **MAPE** | (100/m)Σ\|y-ŷ\|/\|y\| | Percentage error, undefined when y=0 |

### 3.7 Cross-Validation

**Why?** A single train/test split may not be representative. CV gives a more reliable estimate of model performance.

**K-Fold Cross-Validation:**
1. Split data into K equal folds.
2. For each fold i: train on K-1 folds, validate on fold i.
3. Average the K validation scores.
- **K = 5 or 10** is standard.
- **K = n (Leave-One-Out):** Lowest bias, highest variance, expensive.

**Stratified K-Fold:** Preserves class distribution in each fold. **Always use for classification.**

```python
from sklearn.model_selection import cross_val_score, StratifiedKFold

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=cv, scoring='f1_macro')
print(f"CV F1: {scores.mean():.4f} ± {scores.std():.4f}")
```

### 3.8 Overfitting vs Underfitting Diagnosis

| Scenario | Training Score | Validation Score | Diagnosis | Solution |
|----------|---------------|-----------------|-----------|----------|
| High / High | 0.95 | 0.93 | Good fit | ✓ |
| High / Low | 0.99 | 0.70 | Overfitting | Regularize, more data, simpler model |
| Low / Low | 0.65 | 0.60 | Underfitting | More features, complex model, less regularization |
| Low / High | Unusual | — | Possible data leakage | Check pipeline |

### 3.9 Complete Evaluation Implementation

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    confusion_matrix, classification_report,
    roc_auc_score, roc_curve, precision_recall_curve,
    average_precision_score
)
import numpy as np

# Assume y_test, y_pred (labels), y_prob (probabilities for positive class)

# All metrics at once
print(classification_report(y_test, y_pred, target_names=['Class 0', 'Class 1']))

# Individual metrics
acc = accuracy_score(y_test, y_pred)
prec = precision_score(y_test, y_pred)
rec = recall_score(y_test, y_pred)
f1 = f1_score(y_test, y_pred)
auc = roc_auc_score(y_test, y_prob)

# Confusion Matrix
cm = confusion_matrix(y_test, y_pred)
# cm[0][0]=TN, cm[0][1]=FP, cm[1][0]=FN, cm[1][1]=TP

# ROC Curve data
fpr, tpr, thresholds_roc = roc_curve(y_test, y_prob)

# Precision-Recall Curve data
precision_vals, recall_vals, thresholds_pr = precision_recall_curve(y_test, y_prob)
pr_auc = average_precision_score(y_test, y_prob)

# Optimal threshold (Youden's J statistic for ROC)
j_scores = tpr - fpr
optimal_idx = np.argmax(j_scores)
optimal_threshold = thresholds_roc[optimal_idx]
print(f"Optimal Threshold: {optimal_threshold:.4f}")
```

### 3.10 Which Metric to Use?

| Scenario | Recommended Metric |
|----------|-------------------|
| Balanced dataset | Accuracy, F1 |
| Imbalanced dataset | F1, PR-AUC, Recall |
| Cost of FP is high (spam, recommendation) | Precision |
| Cost of FN is high (disease, fraud) | Recall |
| Ranking/scoring task | AUC-ROC |
| Severely imbalanced | PR-AUC, F1 |
| Regression | RMSE (penalize large errors), MAE (robust) |
| Comparing models across thresholds | AUC-ROC, PR-AUC |

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Using accuracy on imbalanced data** — 99% accuracy means nothing if 99% is one class.
2. **Forgetting to scale features** for SVM and K-NN — both are distance-based.
3. **Using K=1 in K-NN** without checking — almost always overfits.
4. **Confusing precision and recall** — remember: Precision = "of my predictions, how many are right?" Recall = "of all actual positives, how many did I find?"
5. **Not using stratified CV** for classification — can have folds with no positive samples.
6. **Evaluating model on training data** — always use held-out test or CV.
7. **Selecting features/hyperparameters on test data** — this is data leakage. Use a validation set or CV.

### Supervised ML Algorithm Selection Guide

| Algorithm | Best For | Scaling? | Interpretable? | Handles Non-linear? |
|-----------|----------|----------|---------------|-------------------|
| Linear Regression | Continuous target, linear relationships | For regularized | Yes | No (add poly features) |
| Logistic Regression | Binary/multiclass, baseline | For regularized | Yes | No (add poly features) |
| Decision Tree | Non-linear, mixed features, explainability | No | Very | Yes |
| Random Forest | General purpose, robust, feature importance | No | Moderate | Yes |
| SVM | High-dimensional, clear margin | Yes | Low | Yes (kernels) |
| K-NN | Simple baseline, low dimensions | Yes | Low | Yes (inherently) |

---

## Interview Preparation

### Direct Interview Answers

**Q: Explain SVM in simple terms.**
> "SVM finds the hyperplane that separates classes with the maximum margin. The key insight is that only the points closest to the boundary — support vectors — matter. For non-linear data, the kernel trick maps data to a higher-dimensional space where linear separation is possible, without explicitly computing the transformation."

**Q: What is the difference between precision and recall?**
> "Precision answers 'of all my positive predictions, how many are correct?' while Recall answers 'of all actual positives, how many did I find?' They trade off: increasing one typically decreases the other. The choice depends on cost: high precision when false alarms are costly (spam filter), high recall when missing positives is costly (cancer screening)."

### Interview Q&A Bank

**Basic:**
1. **Q:** What does K-NN stand for and how does it work? **A:** K-Nearest Neighbors. It stores training data and classifies new points by majority vote of K closest training points using a distance metric (usually Euclidean). No explicit training phase.

2. **Q:** What is a confusion matrix? **A:** A 2×2 (for binary) table showing TP, TN, FP, FN counts. It reveals not just overall accuracy but what types of errors the model makes.

3. **Q:** What is AUC-ROC? **A:** Area Under the Receiver Operating Characteristic curve. ROC plots TPR vs FPR at various thresholds. AUC summarizes this as a single number: 1 = perfect, 0.5 = random. It measures how well the model separates classes.

**Intermediate:**
4. **Q:** Explain the kernel trick. **A:** Instead of explicitly mapping data to high-dimensional space (expensive), kernel functions compute the dot product in that space directly. K(x,y) = φ(x)ᵀφ(y). RBF kernel maps to infinite-dimensional space but computes in O(n) time.

5. **Q:** Why is F1 score the harmonic mean and not arithmetic mean? **A:** Harmonic mean penalizes extreme differences. If Precision=1.0 and Recall=0.01, arithmetic mean = 0.505 (misleadingly high), but harmonic mean ≈ 0.02 (reflects poor recall). F1 ensures both metrics must be reasonably high.

6. **Q:** How does the C parameter in SVM work? **A:** C controls the trade-off between maximizing margin and minimizing classification error. Large C → small margin, fits training data closely (may overfit). Small C → large margin, more misclassifications allowed (may underfit). Found via cross-validation.

**Advanced:**
7. **Q:** When would you choose PR-AUC over ROC-AUC? **A:** On highly imbalanced datasets. ROC-AUC can be misleadingly high because TN inflates the denominator of FPR. PR-AUC focuses only on the positive class performance. Example: 1% positive class — a model catching 50% of positives while having 100 false positives per true positive might still have AUC-ROC > 0.9 but poor PR-AUC.

8. **Q:** What is the curse of dimensionality and how does it affect K-NN? **A:** In high dimensions, all points become approximately equidistant — the ratio of nearest to farthest distance approaches 1. K-NN relies on meaningful distance differences, so it fails. Solutions: dimensionality reduction (PCA), feature selection, or use algorithms like SVM that handle high dimensions better.

9. **Q:** How do you handle imbalanced datasets? **A:** Multiple approaches: (1) Resample: oversample minority (SMOTE) or undersample majority. (2) Use class weights (`class_weight='balanced'` in sklearn). (3) Use appropriate metrics (F1, PR-AUC, not accuracy). (4) Cost-sensitive learning. (5) Ensemble methods (BalancedRandomForest). (6) Threshold tuning.

---

## Rapid Revision Summary

- **SVM:** Maximum margin classifier. Hard margin (separable) vs Soft margin (C parameter). Kernel trick for non-linearity. RBF most common. Scale features. Support vectors determine boundary.
- **K-NN:** Lazy learner. Stores all data. Distance-based. K controls complexity. Scale features. Curse of dimensionality. Good baseline.
- **Confusion Matrix:** TP, TN, FP, FN → derive Accuracy, Precision, Recall, F1.
- **Precision:** TP/(TP+FP) — few false alarms. **Recall:** TP/(TP+FN) — miss nothing.
- **F1:** Harmonic mean of P and R. Use when both matter.
- **AUC-ROC:** TPR vs FPR across thresholds. Threshold-independent. 0.5 = random, 1 = perfect.
- **Imbalanced data:** Use F1/PR-AUC, not accuracy. Use stratified CV.

---

## Self-Assessment

1. Explain the difference between hard margin and soft margin SVM.
2. What is the role of γ in the RBF kernel? What happens when it's too large or too small?
3. Why does K-NN fail in high dimensions? Name two solutions.
4. Calculate precision, recall, and F1 given: TP=80, FP=20, FN=10, TN=890.
5. When would you prefer PR-AUC over ROC-AUC? Give an example.
6. What is the effect of K on the bias-variance tradeoff in K-NN?
7. How do support vectors affect the SVM model? What happens if you remove non-support vectors?

**Answers:**
1. Hard margin: no misclassifications allowed, only works on linearly separable data. Soft margin: allows misclassifications via slack variables ξ, controlled by C parameter.
2. γ controls the radius of influence of each support vector. Large γ → narrow influence → complex boundary → overfitting. Small γ → wide influence → smooth boundary → underfitting.
3. Curse of dimensionality: all distances converge. Solutions: PCA/dimensionality reduction, feature selection, use algorithms better suited for high dimensions.
4. Precision = 80/(80+20) = 0.80. Recall = 80/(80+10) = 0.889. F1 = 2(0.80×0.889)/(0.80+0.889) = 0.842.
5. On imbalanced datasets. Example: fraud detection with 0.1% fraud rate — ROC-AUC can be misleadingly high because TN dominate, while PR-AUC exposes poor precision.
6. Small K: low bias, high variance (overfits). Large K: high bias, low variance (underfits). Cross-validate to find optimal.
7. Support vectors completely determine the hyperplane. Removing non-support vectors doesn't change the model at all.

---

## Recall Checklist

- [ ] Explain SVM's optimization objective (maximize margin)
- [ ] Write the soft margin formulation with C and slack variables
- [ ] Explain the kernel trick and list 4 kernels
- [ ] Describe K-NN's algorithm step by step
- [ ] Explain the curse of dimensionality
- [ ] Draw and explain a confusion matrix
- [ ] Calculate Precision, Recall, F1 from TP/FP/FN/TN
- [ ] Explain ROC-AUC and PR-AUC and when to use each
- [ ] Explain macro vs micro vs weighted averaging
- [ ] Implement SVM with proper scaling in sklearn
- [ ] Implement K-NN with optimal K selection via CV
- [ ] List which metrics to use for imbalanced datasets
