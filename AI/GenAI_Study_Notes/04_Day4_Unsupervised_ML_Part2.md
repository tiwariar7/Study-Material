# Day 4: Unsupervised ML — Anomaly Detection & DBSCAN; Dimensionality Reduction

---

## Topics Covered
1. **Anomaly Detection** (Isolation Forest, Statistical Methods, One-Class SVM)
2. **DBSCAN** (Density-Based Spatial Clustering)
3. **Dimensionality Reduction** (t-SNE, UMAP, LDA — beyond PCA)

---

## 1. ANOMALY DETECTION

### 1.1 Definition & Fundamentals

**Definition:** Anomaly detection (outlier detection) identifies data points that deviate significantly from the majority of the data. These unusual observations may indicate errors, fraud, faults, or novel patterns.

**Types of Anomalies:**
- **Point anomalies:** A single data point is anomalous (e.g., a fraudulent transaction).
- **Contextual anomalies:** Anomalous in a specific context (e.g., 40°C is normal in summer but anomalous in winter).
- **Collective anomalies:** A collection of data points is anomalous together (e.g., unusual sequence of network packets).

**Approaches:**
| Approach | Method | When to Use |
|----------|--------|-------------|
| **Statistical** | Z-score, IQR, Gaussian models | Data follows known distribution |
| **Distance-based** | K-NN distance, LOF | Density varies |
| **Density-based** | Local Outlier Factor (LOF) | Clusters of varying density |
| **Isolation-based** | Isolation Forest | General purpose, scalable |
| **Model-based** | One-Class SVM, Autoencoders | Complex patterns, high dimensions |

### 1.2 Statistical Methods

**Z-Score Method:**
```
z = (x - μ) / σ
```
Points with |z| > 3 (or 2.5) are considered outliers.
- **Assumption:** Data is approximately normally distributed.
- **Limitation:** Only works for univariate, roughly Gaussian data.

**IQR Method:**
```
IQR = Q3 - Q1
Lower bound = Q1 - 1.5 × IQR
Upper bound = Q3 + 1.5 × IQR
```
Points outside [lower, upper] are outliers.
- **Advantage:** No assumption of normality.
- **Used in:** Box plots.

### 1.3 Isolation Forest

**Intuition:** Anomalies are few and different — they are easier to isolate (separate from the rest). Normal points require many random splits to isolate; anomalies need very few.

**Algorithm:**
1. Build a forest of random binary trees (Isolation Trees).
2. For each tree: randomly select a feature and a random split value within that feature's range.
3. Recursively split until each point is isolated (in its own leaf).
4. **Anomaly score** = average path length to isolation across all trees.
   - **Short path** → anomaly (easily isolated).
   - **Long path** → normal (hard to isolate).

**Score normalization:**
```
s(x, n) = 2^(-E(h(x)) / c(n))
```
Where:
- E(h(x)) = average path length for point x
- c(n) = average path length of unsuccessful search in BST (normalization constant)
- s → 1: anomaly. s → 0.5: no clear decision. s → 0: normal.

**Advantages:**
- Linear time complexity O(n × t × log n) where t = number of trees.
- No distance computation needed.
- Works well in high dimensions.
- No need to profile "normal" data distribution.

```python
from sklearn.ensemble import IsolationForest
import numpy as np

# -1 = anomaly, 1 = normal
iso = IsolationForest(
    n_estimators=100,       # Number of trees
    contamination=0.05,     # Expected proportion of outliers
    random_state=42
)
predictions = iso.fit_predict(X)
anomaly_scores = iso.decision_function(X)  # Lower = more anomalous

anomalies = X[predictions == -1]
print(f"Detected {len(anomalies)} anomalies out of {len(X)} points")
```

### 1.4 Local Outlier Factor (LOF)

**Intuition:** A point is an outlier if it's in a region of significantly lower density than its neighbors. LOF compares each point's density to its neighbors' densities.

**Key Concepts:**
- **k-distance(x):** Distance to the k-th nearest neighbor.
- **Reachability distance:** max(k-distance(neighbor), actual_distance).
- **Local Reachability Density (LRD):** Inverse of average reachability distance.
- **LOF score:** Ratio of a point's LRD to the average LRD of its neighbors.

**LOF ≈ 1:** Similar density to neighbors (normal). **LOF >> 1:** Much lower density (outlier).

```python
from sklearn.neighbors import LocalOutlierFactor

lof = LocalOutlierFactor(n_neighbors=20, contamination=0.05)
predictions = lof.fit_predict(X)  # -1 = outlier, 1 = inlier
scores = lof.negative_outlier_factor_  # More negative = more outlier
```

### 1.5 One-Class SVM

- Learns a boundary around "normal" data.
- Points outside the boundary = anomalies.
- Uses kernel trick for non-linear boundaries.
- Good when you only have normal data for training.

```python
from sklearn.svm import OneClassSVM

ocsvm = OneClassSVM(kernel='rbf', gamma='scale', nu=0.05)  # nu ≈ contamination
ocsvm.fit(X_train_normal)  # Train only on normal data
predictions = ocsvm.predict(X_test)  # -1 = anomaly
```

### 1.6 Comparison of Anomaly Detection Methods

| Method | Strengths | Weaknesses | Complexity |
|--------|-----------|------------|------------|
| Z-score/IQR | Simple, interpretable | Univariate, assumes distribution | O(n) |
| Isolation Forest | Scalable, high-dim, no density estimation | contamination param needed | O(n log n) |
| LOF | Handles varying densities | Sensitive to k, slow for large n | O(n² k) |
| One-Class SVM | Non-linear boundaries | Slow, kernel selection | O(n²) to O(n³) |
| Autoencoders | Complex patterns, deep learning | Needs lots of data, training | Depends |

---

## 2. DBSCAN

### 2.1 Definition & Fundamentals

**Definition:** DBSCAN (Density-Based Spatial Clustering of Applications with Noise) is a density-based clustering algorithm that groups together points that are closely packed, marks low-density points as outliers, and can find arbitrarily shaped clusters.

**Plain English:** A cluster is a dense region of points separated from other dense regions by sparse areas. Points in sparse areas are noise/outliers.

**Key Parameters:**
- **ε (eps):** The maximum distance between two points to be considered neighbors.
- **MinPts (min_samples):** The minimum number of points within ε distance to form a dense region (core point).

### 2.2 Point Classification

| Point Type | Definition | Role |
|-----------|-----------|------|
| **Core Point** | Has ≥ MinPts neighbors within ε | Heart of a cluster |
| **Border Point** | Within ε of a core point but has < MinPts neighbors | Edge of a cluster |
| **Noise Point** | Not within ε of any core point | Outlier (label = -1) |

### 2.3 Algorithm

```
1. For each point p:
   a. Find all points within distance ε (ε-neighborhood).
   b. If |ε-neighborhood| ≥ MinPts → mark p as CORE point.

2. Connect core points that are within ε of each other → form clusters.

3. For each non-core point:
   a. If within ε of a core point → assign to that cluster (BORDER point).
   b. Otherwise → mark as NOISE.

More precisely (with density-reachability):
1. Pick an unvisited point p.
2. Find its ε-neighborhood.
3. If |neighborhood| < MinPts, mark as noise (may later become border).
4. If |neighborhood| ≥ MinPts:
   a. Create a new cluster C, add p to C.
   b. For each point q in p's neighborhood:
      - If q is unvisited, find q's ε-neighborhood.
      - If q is a core point, add q's neighbors to the expansion set.
      - If q is not yet in any cluster, add q to C.
5. Repeat until all points are visited.
```

### 2.4 Choosing Parameters

**ε (eps):**
- **k-distance plot method:** For each point, compute distance to k-th nearest neighbor (k = MinPts). Sort in ascending order and plot. The "elbow" suggests ε.
- Too small → many noise points. Too large → clusters merge.

**MinPts:**
- Rule of thumb: MinPts ≥ d + 1 (d = dimensionality). Often MinPts = 2×d.
- Minimum: 3. Common default: 5.
- Larger MinPts → smoother clusters, more noise points.

```python
from sklearn.neighbors import NearestNeighbors
import matplotlib.pyplot as plt

# K-distance plot for choosing eps
nn = NearestNeighbors(n_neighbors=5)  # k = MinPts
nn.fit(X_scaled)
distances, _ = nn.kneighbors(X_scaled)
distances = np.sort(distances[:, -1])  # k-th neighbor distance

plt.plot(distances)
plt.xlabel('Points (sorted)')
plt.ylabel('5th Nearest Neighbor Distance')
plt.title('K-Distance Plot')
plt.show()
# Choose eps at the elbow point
```

### 2.5 Implementation

```python
from sklearn.cluster import DBSCAN
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import silhouette_score

# Always scale!
X_scaled = StandardScaler().fit_transform(X)

db = DBSCAN(
    eps=0.5,            # Neighborhood radius
    min_samples=5,      # MinPts
    metric='euclidean'  # Can also use 'manhattan', 'cosine', etc.
)
labels = db.fit_predict(X_scaled)

n_clusters = len(set(labels)) - (1 if -1 in labels else 0)
n_noise = list(labels).count(-1)

print(f"Clusters: {n_clusters}")
print(f"Noise points: {n_noise}")

# Silhouette score (exclude noise points)
if n_clusters > 1:
    mask = labels != -1
    sil = silhouette_score(X_scaled[mask], labels[mask])
    print(f"Silhouette (excl. noise): {sil:.4f}")
```

### 2.6 DBSCAN vs K-Means

| Aspect | DBSCAN | K-Means |
|--------|--------|---------|
| Number of clusters | Automatic | Must specify K |
| Cluster shape | Arbitrary | Spherical only |
| Outlier handling | Labels as noise | Assigns to nearest centroid |
| Parameter sensitivity | ε and MinPts | K |
| Varying density | Struggles | Struggles |
| Deterministic | Yes | No (random init) |
| Scalability | O(n²) or O(n log n) with index | O(nKdI) |

### 2.7 HDBSCAN (Improved DBSCAN)

**HDBSCAN** = Hierarchical DBSCAN. Automatically selects ε from a hierarchy:
- No need to choose ε.
- Handles varying density clusters.
- Only one parameter: `min_cluster_size`.
- Generally preferred over DBSCAN in practice.

```python
# pip install hdbscan
import hdbscan

clusterer = hdbscan.HDBSCAN(min_cluster_size=15)
labels = clusterer.fit_predict(X_scaled)
print(f"Clusters: {len(set(labels)) - (1 if -1 in labels else 0)}")
```

### 2.8 Complexity

| Algorithm | Time | Space |
|-----------|------|-------|
| DBSCAN (naive) | O(n²) | O(n) |
| DBSCAN (with spatial index, e.g., KD-tree) | O(n log n) | O(n) |
| K-Means | O(nKdI) | O(nK) |

---

## 3. DIMENSIONALITY REDUCTION (BEYOND PCA)

### 3.1 Overview

PCA (covered in Day 3) is linear. For non-linear data, we need other methods:

| Method | Linear? | Preserves | Best For |
|--------|---------|-----------|----------|
| **PCA** | Yes | Global variance | General reduction, preprocessing |
| **t-SNE** | No | Local structure | 2D/3D visualization |
| **UMAP** | No | Local + global | Visualization + downstream ML |
| **LDA** | Yes | Class separability | Supervised reduction |

### 3.2 t-SNE (t-distributed Stochastic Neighbor Embedding)

**Definition:** t-SNE is a non-linear dimensionality reduction technique primarily used for **visualization** of high-dimensional data in 2D or 3D.

**How it works:**
1. Compute pairwise similarities in high-dimensional space using Gaussian distribution.
2. Define pairwise similarities in low-dimensional space using **Student's t-distribution** (heavier tails).
3. Minimize the KL divergence between the two distributions using gradient descent.

**Key parameter — Perplexity:**
- Roughly the effective number of nearest neighbors.
- Typical range: 5-50. Default: 30.
- Low perplexity → focus on very local structure (may see artificial clusters).
- High perplexity → more global structure.

**Critical Properties:**
- **Not deterministic** (random initialization).
- **Different runs produce different results.**
- **Distances between clusters are NOT meaningful** — only within-cluster structure is.
- **Cannot project new data** (no transform method). Fit + transform together.
- **Slow** for large datasets: O(n²) naive, O(n log n) with Barnes-Hut approximation.
- **Only for visualization** — do NOT use t-SNE embeddings as features for ML models.

```python
from sklearn.manifold import TSNE

tsne = TSNE(
    n_components=2,
    perplexity=30,
    learning_rate='auto',  # or 200
    n_iter=1000,
    random_state=42
)
X_tsne = tsne.fit_transform(X_scaled)  # No separate transform!

plt.scatter(X_tsne[:, 0], X_tsne[:, 1], c=y, cmap='viridis', alpha=0.7)
plt.title('t-SNE Visualization')
plt.show()
```

### 3.3 UMAP (Uniform Manifold Approximation and Projection)

**Advantages over t-SNE:**
- **Faster** (especially for large datasets).
- **Preserves more global structure** (cluster-to-cluster relationships more meaningful).
- **Can transform new data** (has a `transform` method).
- **Can be used for general dimensionality reduction** (not just visualization).
- **More parameters but sensible defaults.**

**Key parameters:**
- `n_neighbors`: Local vs global balance (like perplexity). Default: 15.
- `min_dist`: How tightly points are packed. Default: 0.1.
- `n_components`: Output dimensions. Default: 2.

```python
# pip install umap-learn
import umap

reducer = umap.UMAP(n_components=2, n_neighbors=15, min_dist=0.1, random_state=42)
X_umap = reducer.fit_transform(X_scaled)

# Can transform new data!
X_new_umap = reducer.transform(X_new)
```

### 3.4 LDA (Linear Discriminant Analysis)

**Definition:** LDA is a **supervised** dimensionality reduction technique that finds linear combinations of features that best separate classes.

**Key difference from PCA:** PCA maximizes variance (unsupervised). LDA maximizes class separability (supervised — uses labels).

**Mathematics:**
```
Maximize: J(w) = (wᵀ Sᵦ w) / (wᵀ Sᵤ w)
```
Where:
- **Sᵦ** = between-class scatter matrix (how far class means are from overall mean)
- **Sᵤ** = within-class scatter matrix (how spread out each class is)
- Maximize ratio → maximize separation / minimize overlap.

**Maximum components:** min(n_classes - 1, n_features). For K classes, LDA gives at most K-1 components.

```python
from sklearn.discriminant_analysis import LinearDiscriminantAnalysis

lda = LinearDiscriminantAnalysis(n_components=2)
X_lda = lda.fit_transform(X_scaled, y)  # Needs labels!

# Can transform new data
X_new_lda = lda.transform(X_new)
```

### 3.5 PCA vs t-SNE vs UMAP vs LDA

| Aspect | PCA | t-SNE | UMAP | LDA |
|--------|-----|-------|------|-----|
| Type | Linear | Non-linear | Non-linear | Linear |
| Supervised? | No | No | No (can be) | Yes |
| Preserves | Global variance | Local structure | Local + global | Class separability |
| Transform new data | Yes | No | Yes | Yes |
| Scalability | Fast | Slow O(n²) | Fast | Fast |
| Use for ML features? | Yes | No | Yes | Yes |
| Deterministic | Yes | No | No | Yes |
| Max components | min(n, d) | 2-3 | Any | K-1 |

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Using t-SNE embeddings as features for ML** — t-SNE distorts distances and is non-parametric. Only use for visualization.
2. **Interpreting t-SNE cluster distances** — distance between clusters has no meaning. Only within-cluster proximity matters.
3. **Choosing ε for DBSCAN arbitrarily** — always use k-distance plot.
4. **Not labeling DBSCAN noise points separately** — they are NOT in any cluster (label = -1).
5. **Using LOF/Isolation Forest without proper contamination** — wrong contamination dramatically affects results.
6. **Confusing LDA (discriminant analysis) with LDA (topic modeling)** — completely different algorithms with the same abbreviation.

---

## Interview Preparation

### Direct Interview Answers

**Q: What is DBSCAN and how does it differ from K-Means?**
> "DBSCAN is a density-based clustering algorithm that groups nearby points into clusters and marks isolated points as noise. Unlike K-Means, it doesn't require specifying the number of clusters, can find arbitrarily shaped clusters, and has built-in outlier detection. However, it struggles with clusters of varying density and requires careful parameter selection (ε and MinPts)."

**Q: When would you use t-SNE vs PCA?**
> "PCA for preprocessing and dimensionality reduction before ML — it's fast, linear, and produces features usable for training. t-SNE for visualization of high-dimensional data — it reveals local structure and clusters in 2D. But never use t-SNE features for ML: it doesn't preserve global structure, can't transform new data, and distorts distances."

### Interview Q&A Bank

**Basic:**
1. **Q:** What is an anomaly? **A:** A data point that deviates significantly from the majority. Types: point (single outlier), contextual (depends on context), collective (group that's anomalous together). Applications: fraud, manufacturing defects, network intrusion.

2. **Q:** What does DBSCAN stand for and what are its two parameters? **A:** Density-Based Spatial Clustering of Applications with Noise. Parameters: ε (eps) = maximum neighborhood distance, MinPts (min_samples) = minimum points to form a core point.

**Intermediate:**
3. **Q:** How does Isolation Forest detect anomalies? **A:** It builds random trees by randomly selecting features and split values. Anomalies, being few and different, are isolated in fewer splits (shorter path length). The anomaly score is based on average path length — shorter path = more anomalous.

4. **Q:** What are core, border, and noise points in DBSCAN? **A:** Core: has ≥ MinPts neighbors within ε (cluster interior). Border: within ε of a core point but has < MinPts neighbors (cluster edge). Noise: not within ε of any core point (outlier, labeled -1).

**Advanced:**
5. **Q:** What are the limitations of DBSCAN? How does HDBSCAN address them? **A:** DBSCAN struggles with varying density clusters (one ε can't fit all), requires careful parameter selection, and is O(n²). HDBSCAN builds a hierarchy of clusters across all ε values, automatically selects the most stable clusters, handles varying density, and only requires min_cluster_size.

6. **Q:** Why does t-SNE use Student's t-distribution in low-dimensional space instead of Gaussian? **A:** The "crowding problem": in high dimensions, many points are at moderate distances, but low-dimensional space doesn't have room for all of them. The t-distribution's heavier tails allow dissimilar points to be pushed further apart, preventing crowding in the center.

---

## Rapid Revision Summary

- **Anomaly Detection:** Z-score/IQR (simple), Isolation Forest (scalable, random splits, short path = anomaly), LOF (density-based), One-Class SVM (boundary learning).
- **DBSCAN:** Density-based. Core/Border/Noise points. ε + MinPts. Arbitrary shapes. Auto-detects K and outliers. Use k-distance plot for ε.
- **t-SNE:** Non-linear, visualization only. Perplexity ≈ effective neighbors. Don't use for ML features. Distances between clusters meaningless.
- **UMAP:** Faster t-SNE alternative. Preserves more global structure. Can transform new data. Usable for ML.
- **LDA:** Supervised dimensionality reduction. Maximizes class separation. Max K-1 components.

---

## Self-Assessment

1. Explain how Isolation Forest scores anomalies without computing distances.
2. What is the difference between core, border, and noise points in DBSCAN?
3. How would you choose ε for DBSCAN?
4. Why can't you use t-SNE embeddings as features for classification?
5. What does UMAP preserve that t-SNE doesn't?
6. When would you use LDA over PCA?

**Answers:**
1. It builds random isolation trees. Anomalies are isolated with fewer splits (shorter average path). Score = 2^(-avg_path / normalization). No distance computation; purely based on tree structure.
2. Core: ≥ MinPts neighbors in ε-ball. Border: in ε of a core point, < MinPts neighbors. Noise: not in ε of any core point. Only core points form cluster seeds; border points are cluster edges.
3. Use k-distance plot: compute k-th nearest neighbor distance for each point (k = MinPts), sort ascending, plot. ε = the distance at the "elbow" (where the curve bends sharply).
4. t-SNE is non-parametric (can't transform new data), doesn't preserve global distances, is non-deterministic, and optimizes for local neighborhood preservation only. The embedding positions are meaningless as feature values.
5. UMAP preserves more global structure (relative positions of clusters are more meaningful) and can transform new data points (parametric mapping).
6. When you have labels and want to reduce dimensions specifically to maximize class separability. PCA maximizes variance regardless of classes; LDA explicitly uses class information.

---

## Recall Checklist

- [ ] List and explain 3+ anomaly detection methods
- [ ] Explain Isolation Forest algorithm and scoring
- [ ] Describe LOF and its key concepts
- [ ] Explain DBSCAN algorithm step by step
- [ ] Define core, border, and noise points
- [ ] Use k-distance plot to choose ε
- [ ] Explain t-SNE algorithm conceptually (high-dim similarities → low-dim → minimize KL divergence)
- [ ] List t-SNE limitations and why it's visualization-only
- [ ] Compare UMAP vs t-SNE
- [ ] Explain LDA objective (maximize between-class / within-class scatter ratio)
- [ ] Distinguish PCA (unsupervised, variance) from LDA (supervised, separability)
