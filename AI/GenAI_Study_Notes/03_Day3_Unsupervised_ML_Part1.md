# Day 3: Unsupervised ML — K-Means & Hierarchical Clustering; PCA

---

## Topics Covered
1. **K-Means Clustering**
2. **Hierarchical Clustering** (Agglomerative & Divisive)
3. **Principal Component Analysis (PCA)**

---

## 1. K-MEANS CLUSTERING

### 1.1 Definition & Fundamentals

**Definition:** K-Means is an unsupervised partitioning algorithm that divides n data points into K clusters, where each point belongs to the cluster with the nearest centroid (mean). It minimizes within-cluster variance.

**Plain English:** Choose K cluster centers, assign each point to the nearest center, recompute centers as the average of assigned points, and repeat until stable.

**What problem it solves:** Grouping similar data points without labels — customer segmentation, image compression, document grouping, anomaly detection (points far from all centroids).

### 1.2 Algorithm (Lloyd's Algorithm)

```
1. Initialize: Randomly select K points as initial centroids (or use K-Means++).
2. Assignment Step: Assign each point xᵢ to the nearest centroid:
   cᵢ = argminₖ ||xᵢ - μₖ||²
3. Update Step: Recompute each centroid as the mean of its assigned points:
   μₖ = (1/|Cₖ|) Σ_{xᵢ ∈ Cₖ} xᵢ
4. Repeat steps 2-3 until centroids don't change (or max iterations reached).
```

**Convergence:** Guaranteed to converge (objective decreases monotonically), but to a **local minimum** (not necessarily global). The result depends on initialization.

### 1.3 Mathematical Foundation

**Objective Function (Within-Cluster Sum of Squares / Inertia):**
```
J = Σₖ₌₁ᴷ Σ_{xᵢ ∈ Cₖ} ||xᵢ - μₖ||²
```
- Minimize total squared distance from each point to its assigned centroid.
- Also called **inertia** in sklearn.
- Lower J = tighter clusters (but overfitting with too many clusters).

**This is an NP-hard problem.** K-Means uses a heuristic (alternating optimization) that finds local optima.

### 1.4 K-Means++ Initialization

**Problem with random initialization:** Can converge to poor local minima. Some initializations lead to terrible clustering.

**K-Means++ Algorithm:**
1. Choose the first centroid uniformly at random from data points.
2. For each remaining centroid:
   a. Compute D(x)² = distance from each point to its nearest existing centroid.
   b. Choose the next centroid with probability proportional to D(x)².
3. Points far from existing centroids are more likely to be chosen.

**Benefit:** O(log K)-competitive with optimal clustering. Much better initial centroids.
**Sklearn default:** `init='k-means++'`.

### 1.5 Choosing K — The Elbow Method & Silhouette Score

**Elbow Method:**
1. Run K-Means for K = 1, 2, 3, ..., 10.
2. Plot K vs Inertia (J).
3. The "elbow" point where inertia decrease slows significantly suggests optimal K.
- **Limitation:** Elbow can be ambiguous or non-existent.

**Silhouette Score:**
```
s(i) = (b(i) - a(i)) / max(a(i), b(i))
```
Where:
- **a(i)** = average distance from point i to all other points in the same cluster (intra-cluster).
- **b(i)** = average distance from point i to all points in the nearest different cluster (inter-cluster).
- **Range:** [-1, 1]. Higher is better. s ≈ 1: well-clustered. s ≈ 0: on boundary. s < 0: likely wrong cluster.

**Average Silhouette Score** across all points → overall clustering quality.

```python
from sklearn.metrics import silhouette_score

scores = []
for k in range(2, 11):
    km = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = km.fit_predict(X)
    scores.append(silhouette_score(X, labels))
    print(f"K={k}, Silhouette={scores[-1]:.4f}, Inertia={km.inertia_:.2f}")
```

### 1.6 Limitations of K-Means

1. **Must specify K in advance.**
2. **Assumes spherical, equally-sized clusters** (uses Euclidean distance).
3. **Sensitive to outliers** (outliers pull centroids).
4. **Sensitive to initialization** (mitigated by K-Means++, `n_init`).
5. **Fails on non-convex shapes** (e.g., concentric circles, moons).
6. **All features must be numerical** (use K-Modes for categorical).
7. **Sensitive to feature scaling** — always standardize.

### 1.7 Variants

| Variant | Difference from K-Means |
|---------|------------------------|
| **K-Means++** | Better initialization (default in sklearn) |
| **Mini-Batch K-Means** | Uses random subsets per iteration → faster for large data |
| **K-Medoids (PAM)** | Uses actual data points as centers → robust to outliers |
| **K-Modes** | For categorical data (uses mode instead of mean) |
| **K-Prototypes** | Mixed numerical + categorical |

### 1.8 Implementation

```python
from sklearn.cluster import KMeans, MiniBatchKMeans
from sklearn.preprocessing import StandardScaler
import numpy as np

# Generate sample data
from sklearn.datasets import make_blobs
X, y_true = make_blobs(n_samples=300, centers=4, cluster_std=0.6, random_state=42)

# Always scale for K-Means
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# K-Means
km = KMeans(
    n_clusters=4,
    init='k-means++',    # Smart initialization
    n_init=10,           # Run 10 times, pick best (lowest inertia)
    max_iter=300,        # Max iterations per run
    random_state=42
)
labels = km.fit_predict(X_scaled)

print(f"Inertia: {km.inertia_:.2f}")
print(f"Centroids shape: {km.cluster_centers_.shape}")  # (4, n_features)
print(f"Iterations: {km.n_iter_}")
print(f"Silhouette: {silhouette_score(X_scaled, labels):.4f}")

# Mini-Batch K-Means (for large datasets)
mbkm = MiniBatchKMeans(n_clusters=4, batch_size=100, random_state=42)
mbkm.fit(X_scaled)
```

### 1.9 Time & Space Complexity

| Aspect | Complexity |
|--------|-----------|
| Time per iteration | O(n × K × d) where n=samples, K=clusters, d=features |
| Total time | O(n × K × d × I × R) where I=iterations, R=n_init restarts |
| Space | O(n × d + K × d) |

---

## 2. HIERARCHICAL CLUSTERING

### 2.1 Definition & Fundamentals

**Definition:** Hierarchical Clustering builds a hierarchy (tree) of clusters, either bottom-up (**agglomerative**: each point starts as its own cluster and merges) or top-down (**divisive**: all points start as one cluster and split).

**Plain English:** Start with each point as its own cluster, then repeatedly merge the two closest clusters until only one remains. The tree of merges (dendrogram) lets you choose any number of clusters.

**Advantages over K-Means:**
- No need to specify K upfront.
- Produces a dendrogram (visualize hierarchy).
- Can find non-spherical clusters (with appropriate linkage).
- Deterministic (no random initialization).

### 2.2 Agglomerative Algorithm

```
1. Start: Each data point is its own cluster (n clusters).
2. Compute distance between all pairs of clusters.
3. Merge the two closest clusters.
4. Update distances.
5. Repeat until one cluster remains (or desired K clusters).
```

### 2.3 Linkage Criteria (How to Measure Inter-Cluster Distance)

| Linkage | Distance Between Clusters A and B | Characteristics |
|---------|----------------------------------|----------------|
| **Single** | min distance between any two points | Finds elongated/chain shapes. Sensitive to noise. |
| **Complete** | max distance between any two points | Finds compact, spherical clusters. Robust. |
| **Average** | mean distance between all pairs | Compromise between single and complete. |
| **Ward's** | Increase in total within-cluster variance after merging | Tends to create equal-sized, spherical clusters. Most common. |

**Ward's Method:** Minimize the total within-cluster variance at each merge step. Mathematically similar to K-Means objective. **Default in sklearn.**

### 2.4 Dendrogram

- **X-axis:** Data points (or cluster labels).
- **Y-axis:** Distance at which clusters merge.
- **Horizontal line at height h:** Cutting here gives clusters. Taller vertical lines = bigger inter-cluster distances.
- **How to choose K:** Cut where there's the largest vertical gap (clusters are most distinct).

```python
from scipy.cluster.hierarchy import dendrogram, linkage, fcluster
import matplotlib.pyplot as plt

# Compute linkage matrix
Z = linkage(X_scaled, method='ward', metric='euclidean')

# Plot dendrogram
plt.figure(figsize=(12, 6))
dendrogram(Z, truncate_mode='level', p=5)  # Show only top 5 levels
plt.xlabel('Sample Index')
plt.ylabel('Distance')
plt.title('Dendrogram')
plt.show()

# Cut at specific number of clusters
labels = fcluster(Z, t=4, criterion='maxclust')  # 4 clusters
```

### 2.5 Agglomerative Clustering in Sklearn

```python
from sklearn.cluster import AgglomerativeClustering

agg = AgglomerativeClustering(
    n_clusters=4,           # Or None with distance_threshold
    linkage='ward',         # 'ward', 'complete', 'average', 'single'
    # distance_threshold=10, # If n_clusters=None, cut at this distance
)
labels = agg.fit_predict(X_scaled)
print(f"Silhouette: {silhouette_score(X_scaled, labels):.4f}")
```

### 2.6 Agglomerative vs Divisive

| Aspect | Agglomerative (Bottom-Up) | Divisive (Top-Down) |
|--------|--------------------------|-------------------|
| Start | Each point = cluster | All points = 1 cluster |
| Process | Merge closest pairs | Split farthest points |
| Complexity | O(n²log n) or O(n³) | O(2ⁿ) (exponential) |
| Common? | Yes (widely used) | Rarely (expensive) |
| Implementation | scipy, sklearn | Less common |

### 2.7 K-Means vs Hierarchical Clustering

| Aspect | K-Means | Hierarchical |
|--------|---------|-------------|
| K specification | Required upfront | Not required (dendrogram) |
| Scalability | O(nKdI) — fast | O(n²) or O(n³) — slow for large n |
| Cluster shape | Spherical | Any (depends on linkage) |
| Deterministic | No (random init) | Yes |
| Result | Flat clusters | Hierarchy (dendrogram) |
| Outlier sensitivity | High | Depends on linkage |
| Memory | O(nK) | O(n²) (distance matrix) |

### 2.8 Time & Space Complexity

| Algorithm | Time | Space |
|-----------|------|-------|
| Agglomerative (naive) | O(n³) | O(n²) |
| Agglomerative (optimized, single linkage) | O(n² log n) | O(n²) |
| Sklearn AgglomerativeClustering | O(n² log n) with connectivity | O(n²) |

---

## 3. PRINCIPAL COMPONENT ANALYSIS (PCA)

### 3.1 Definition & Fundamentals

**Definition:** PCA is an unsupervised linear dimensionality reduction technique that transforms data into a new coordinate system of **principal components** — orthogonal directions that capture the maximum variance in the data.

**Plain English:** PCA finds the directions (axes) along which your data varies the most. By keeping only the top few directions, you reduce the number of features while retaining most of the information.

**What problem it solves:**
- Reduce dimensions for visualization (2D/3D).
- Speed up ML algorithms.
- Remove multicollinearity.
- Reduce overfitting (fewer features).
- Noise reduction (discard low-variance components).

### 3.2 Intuition & Working Mechanism

**Step-by-step:**
1. **Standardize** the data (mean=0, std=1 for each feature). **Critical** because PCA is variance-based; features with larger scales dominate.
2. **Compute the covariance matrix** C = (1/(n-1)) XᵀX (where X is centered).
3. **Compute eigenvalues and eigenvectors** of C.
4. **Sort eigenvectors** by eigenvalue (descending). Eigenvector with largest eigenvalue = PC1 (direction of max variance).
5. **Select top k eigenvectors** to form the projection matrix W (d×k).
6. **Transform data:** Z = XW (project onto k-dimensional space).

**Why eigenvectors?** They are the directions along which the data has maximum variance. The eigenvalue = amount of variance captured by that direction.

### 3.3 Mathematical Foundation

**Covariance Matrix:**
```
C = (1/(n-1)) XᵀX    (X is centered: subtract mean from each feature)
```
C is a d×d symmetric matrix where Cᵢⱼ = covariance between feature i and feature j.

**Eigendecomposition:**
```
Cv = λv
```
Where:
- **v** = eigenvector (principal component direction)
- **λ** = eigenvalue (variance explained by this direction)

**Explained Variance Ratio:**
```
EVR(k) = λₖ / Σᵢ λᵢ
```
- Proportion of total variance captured by the k-th component.
- **Cumulative EVR:** Sum of top k ratios = total variance retained.

**Choosing k:** Select k such that cumulative EVR ≥ 95% (or 90%, depending on application).

### 3.4 PCA via SVD

In practice, PCA is computed using **Singular Value Decomposition (SVD)** rather than eigendecomposition:
```
X = UΣVᵀ
```
Where:
- **U** = left singular vectors (n×n)
- **Σ** = diagonal matrix of singular values (n×d)
- **V** = right singular vectors (d×d) — **these are the principal components**

Relationship: σᵢ² / (n-1) = λᵢ (singular values relate to eigenvalues).

**Why SVD?** More numerically stable than eigendecomposition, especially for large or ill-conditioned matrices.

### 3.5 Worked Example

```
Data (2 features): (1,2), (3,4), (5,6), (7,8)

1. Center: mean = (4, 5). Centered: (-3,-3), (-1,-1), (1,1), (3,3)

2. Covariance matrix:
   C = (1/3) × [(-3)²+(-1)²+1²+3²    (-3)(-3)+(-1)(-1)+1·1+3·3]
                [same                    (-3)²+(-1)²+1²+3²        ]
   C = (1/3) × [20  20] = [6.67  6.67]
                [20  20]   [6.67  6.67]

3. Eigenvalues: λ₁ = 13.33, λ₂ = 0
   Eigenvectors: v₁ = [1/√2, 1/√2], v₂ = [1/√2, -1/√2]

4. EVR: PC1 captures 100% of variance! (data lies on a perfect line)

5. Transform: Z = X × v₁ → reduces to 1D
```

### 3.6 Important Properties & Limitations

**Properties:**
- PCs are **orthogonal** (uncorrelated).
- PC1 has max variance, PC2 has max remaining variance perpendicular to PC1, etc.
- Total variance is preserved: Σλᵢ = Σvar(features).
- **Reconstruction:** X_approx = ZWᵀ + μ (can approximately reconstruct original data from reduced representation).

**Limitations:**
1. **Linear only** — cannot capture non-linear relationships (use Kernel PCA for that).
2. **Sensitive to scaling** — must standardize first.
3. **Interpretability lost** — PCs are linear combinations of original features; individual components lose semantic meaning.
4. **Assumes variance = information** — high variance ≠ always useful (could be noise).
5. **All components computed from all features** — no feature selection (contrast with Lasso).

### 3.7 PCA vs Feature Selection

| Aspect | PCA | Feature Selection |
|--------|-----|------------------|
| Method | Creates new features (combinations) | Selects existing features |
| Interpretability | Low (new abstract axes) | High (original features) |
| Information loss | Minimal (captures max variance) | Potentially more |
| Multicollinearity | Removes (PCs are orthogonal) | May remain |
| Works with | Numerical features | Any features |

### 3.8 Implementation

```python
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler
import numpy as np

# CRITICAL: Always standardize before PCA
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# --- Full PCA (to analyze explained variance) ---
pca_full = PCA()
pca_full.fit(X_scaled)

# Explained variance ratio
print("Explained variance per component:", pca_full.explained_variance_ratio_)
print("Cumulative variance:", np.cumsum(pca_full.explained_variance_ratio_))

# --- Choose k components (e.g., 95% variance) ---
pca = PCA(n_components=0.95)  # Keep enough for 95% variance
X_reduced = pca.fit_transform(X_scaled)
print(f"Reduced from {X_scaled.shape[1]} to {X_reduced.shape[1]} dimensions")
print(f"Variance retained: {sum(pca.explained_variance_ratio_):.4f}")

# --- Or specify exact number of components ---
pca_2d = PCA(n_components=2)
X_2d = pca_2d.fit_transform(X_scaled)

# --- For visualization ---
import matplotlib.pyplot as plt
plt.scatter(X_2d[:, 0], X_2d[:, 1], c=y, cmap='viridis')
plt.xlabel(f'PC1 ({pca_2d.explained_variance_ratio_[0]:.1%} var)')
plt.ylabel(f'PC2 ({pca_2d.explained_variance_ratio_[1]:.1%} var)')
plt.title('PCA Projection')
plt.show()

# --- Reconstruction ---
X_reconstructed = pca.inverse_transform(X_reduced)
reconstruction_error = np.mean((X_scaled - X_reconstructed) ** 2)
print(f"Reconstruction MSE: {reconstruction_error:.4f}")

# --- Component loadings (how much each original feature contributes) ---
# pca.components_ is (n_components, n_features)
for i, comp in enumerate(pca_2d.components_):
    print(f"PC{i+1} loadings: {comp}")
```

### 3.9 Kernel PCA (Non-Linear Extension)

- Standard PCA only captures linear relationships.
- **Kernel PCA** applies the kernel trick to project data into a higher-dimensional space, then performs PCA.
- Common kernels: RBF, polynomial (same as SVM kernels).
- Useful for non-linear dimensionality reduction (e.g., Swiss roll).

```python
from sklearn.decomposition import KernelPCA

kpca = KernelPCA(n_components=2, kernel='rbf', gamma=0.1)
X_kpca = kpca.fit_transform(X_scaled)
```

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Not standardizing before K-Means or PCA** — both are distance/variance-based.
2. **Choosing K arbitrarily** for K-Means — always use elbow + silhouette.
3. **Assuming K-Means handles non-spherical clusters** — it doesn't. Use DBSCAN or spectral clustering.
4. **Confusing PCA with feature selection** — PCA creates new composite features, doesn't select originals.
5. **Applying PCA after train/test split but fitting on test data** — fit PCA on training data only, then transform both.
6. **Over-reducing dimensions with PCA** — always check cumulative variance retained.

### Key Comparison: Clustering Algorithms

| Aspect | K-Means | Hierarchical | DBSCAN (Day 4) |
|--------|---------|-------------|----------------|
| K required | Yes | No (dendrogram) | No |
| Cluster shape | Spherical | Any (linkage-dependent) | Any shape |
| Outlier handling | Poor | Linkage-dependent | Marks as noise |
| Scalability | O(nKdI) — good | O(n²) — poor | O(n²) medium |
| Deterministic | No | Yes | Yes |

---

## Interview Preparation

### Direct Interview Answers

**Q: Explain K-Means clustering.**
> "K-Means partitions data into K clusters by alternating between assigning each point to the nearest centroid and recomputing centroids as cluster means. It minimizes within-cluster sum of squares. Key considerations: uses K-Means++ for initialization, requires specifying K (use elbow/silhouette), assumes spherical clusters, and requires feature scaling."

**Q: What is PCA and when would you use it?**
> "PCA is a dimensionality reduction technique that finds orthogonal directions (principal components) of maximum variance. It works by eigendecomposing the covariance matrix and projecting data onto the top k eigenvectors. I'd use it to reduce dimensions before ML (faster, less overfitting), remove multicollinearity, visualize high-dimensional data, or compress images."

### Interview Q&A Bank

**Basic:**
1. **Q:** What is the difference between supervised and unsupervised learning? **A:** Supervised uses labeled data (input→output mapping); unsupervised finds patterns in unlabeled data (clustering, dimensionality reduction). K-Means, PCA are unsupervised; SVM, Random Forest are supervised.

2. **Q:** How does K-Means initialize centroids? **A:** K-Means++ (default): first centroid is random, subsequent ones are chosen with probability proportional to squared distance from nearest existing centroid. This spreads centroids and avoids poor local minima.

3. **Q:** What is eigenvalue in PCA? **A:** The eigenvalue of a principal component equals the variance of the data projected onto that component. Larger eigenvalue = more information captured.

**Intermediate:**
4. **Q:** How do you choose K in K-Means? **A:** Two main methods: (1) Elbow method — plot K vs inertia, pick the elbow. (2) Silhouette score — measures how similar points are to their own cluster vs other clusters. Higher is better. Also consider domain knowledge.

5. **Q:** Explain Ward's linkage. **A:** Ward's minimizes the increase in total within-cluster variance when merging two clusters. At each step, it merges the pair that least increases the total variance. Produces compact, spherical clusters similar to K-Means.

6. **Q:** Can PCA be used for feature selection? **A:** No. PCA creates new features (linear combinations of originals), so interpretability is lost. For feature selection, use Lasso, mutual information, or tree-based importance. PCA is dimensionality reduction, not feature selection.

**Advanced:**
7. **Q:** Why is K-Means guaranteed to converge but may not find the global optimum? **A:** Each step (assignment, update) decreases or maintains the objective. Since there are finite partitions, it must terminate. But the objective is non-convex with multiple local minima. Different initializations → different solutions. Mitigation: run n_init times with K-Means++.

8. **Q:** What is the relationship between PCA and SVD? **A:** PCA can be computed via eigendecomposition of the covariance matrix (XᵀX) or via SVD (X = UΣVᵀ). The right singular vectors V are the principal components, and σᵢ²/(n-1) equals the eigenvalue λᵢ. SVD is preferred for numerical stability.

9. **Q:** When would PCA hurt model performance? **A:** When variance ≠ information. If the target variable is related to low-variance directions, PCA discards useful information. Also when data has non-linear structure (use Kernel PCA). And when interpretability is needed (PCs are abstract combinations).

---

## Rapid Revision Summary

- **K-Means:** Initialize K centroids (K-Means++), assign points to nearest centroid, recompute centroids, repeat. Minimizes inertia. Spherical clusters only. Scale features. Choose K via elbow/silhouette.
- **Hierarchical Clustering:** Bottom-up (agglomerative): merge closest clusters. Dendrogram shows full hierarchy. Ward's linkage = compact clusters. O(n²) space and time.
- **PCA:** Standardize → covariance matrix → eigendecomposition → project onto top k eigenvectors. Captures max variance. Cumulative EVR ≥ 95% typical. Linear only (use Kernel PCA for non-linear).

---

## Self-Assessment

1. Walk through the K-Means algorithm step by step. What is the objective function?
2. Why is K-Means++ better than random initialization?
3. Compute the silhouette score conceptually for a point in a well-separated cluster.
4. What is Ward's linkage and why is it the most common?
5. Explain how to read a dendrogram and choose K.
6. Why must you standardize before PCA? What goes wrong if you don't?
7. Given eigenvalues [4.5, 2.0, 0.3, 0.1, 0.1], how many components would you keep for 95% variance?

**Answers:**
1. Initialize centroids → assign each point to nearest centroid → recompute centroids as cluster means → repeat until convergence. Objective: minimize J = Σ||xᵢ - μₖ||² (within-cluster sum of squares).
2. K-Means++ spreads initial centroids by choosing subsequent ones with probability proportional to D(x)². This avoids placing all centroids in one cluster region, leading to better local minima.
3. If a(i) ≈ 0 (very close to own cluster) and b(i) is large (far from nearest other cluster), then s(i) = (b-a)/max(a,b) ≈ 1 — well-clustered.
4. Ward's minimizes the increase in total within-cluster variance when merging. It produces compact, balanced clusters and is mathematically similar to K-Means' objective.
5. Y-axis = merge distance. Large gaps between merge levels = natural cluster boundaries. Cut at the largest gap to determine K.
6. PCA maximizes variance. Without standardization, features with larger scales (e.g., income in thousands vs age in decades) dominate the first few PCs, regardless of actual importance.
7. Total variance = 7.0. Cumulative: PC1=64.3%, PC1+2=92.9%, PC1+2+3=97.1% ≥ 95%. Keep **3 components**.

---

## Recall Checklist

- [ ] Describe K-Means algorithm and its objective function
- [ ] Explain K-Means++ initialization
- [ ] Calculate and interpret silhouette score
- [ ] List K-Means limitations (shape, K, outliers, scaling)
- [ ] Describe agglomerative hierarchical clustering algorithm
- [ ] Explain 4 linkage criteria and when to use each
- [ ] Read a dendrogram to determine optimal K
- [ ] Derive PCA steps: standardize → covariance → eigen → project
- [ ] Explain explained variance ratio and how to choose k
- [ ] Distinguish PCA from feature selection
- [ ] Implement K-Means, AgglomerativeClustering, and PCA in sklearn
- [ ] Explain Kernel PCA and when it's needed
