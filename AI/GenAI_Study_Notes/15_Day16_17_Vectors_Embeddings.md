# Day 16–17: Vectors & Embeddings — Dense vs Sparse, ANN Indexing (HNSW, IVF, PQ) & Vector Databases

---

## Topics Covered
1. **Embedding Representations: Dense vs Sparse vs Learned Sparse**
   - Dense Embeddings (Bi-encoders: `text-embedding-3`, BGE, E5, Voyage)
   - Sparse Embeddings (BM25, SPLADE)
   - Hybrid Search & Reciprocal Rank Fusion (RRF)
2. **Vector Similarity & Distance Metrics**
   - Cosine Similarity, Dot Product, Euclidean ($L_2$), Manhattan ($L_1$)
   - Mathematical proof: $L_2$ equivalence to Cosine Distance on unit-normalized vectors
3. **Approximate Nearest Neighbor (ANN) Algorithms**:
   - Flat Search ($O(N \cdot d)$ exact baseline)
   - Inverted File Index (IVF)
   - **HNSW (Hierarchical Navigable Small World)** graphs (Multi-layer skip-list graph mechanics)
   - **Product Quantization (PQ)** & Vector Compression
4. **Bi-Encoders vs Cross-Encoders (Re-ranking Pipeline)**
5. **Vector Database Landscape** (Chroma, Pinecone, Weaviate, Qdrant, Milvus, pgvector)
6. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. EMBEDDING REPRESENTATIONS

```
                     TEXT EMBEDDING ARCHITECTURES
  +--------------------------------+--------------------------------+
  | DENSE EMBEDDINGS               | SPARSE EMBEDDINGS              |
  | • Dimension: 384 to 3072       | • Dimension: 30,000+ (Vocab)   |
  | • Most entries are non-zero    | • 99% entries are zero         |
  | • Captures deep SEMANTICS      | • Captures EXACT KEYWORD match |
  | • Examples: OpenAI, BGE, E5    | • Examples: BM25, TF-IDF       |
  +--------------------------------+--------------------------------+
                                  |
                                  ↓
  +-----------------------------------------------------------------+
  | LEARNED SPARSE: SPLADE (SParse Language AnD Encoder)            |
  | • Maps text into sparse vocabulary weights with query expansion |
  +-----------------------------------------------------------------+
```

### 1.1 Dense Embeddings (Bi-Encoder)
A deep transformer encoder compresses a text chunk of arbitrary length into a fixed-size dense vector $\vec{v} \in \mathbb{R}^d$.
- **Strengths:** Captures conceptual synonyms (e.g., matches `"automobile accident"` with `"car crash"`).
- **Weaknesses:** Fails on exact product codes, acronyms, or rare IDs (e.g., `"Error 0x80070005"` or `"SKU-9941"`).

### 1.2 Sparse Embeddings & BM25
Computes sparse frequency representations over fixed vocabularies:
$$\text{BM25}(D, Q) = \sum_{i=1}^n \text{IDF}(q_i) \cdot \frac{f(q_i, D) \cdot (k_1 + 1)}{f(q_i, D) + k_1 \cdot \left(1 - b + b \cdot \frac{|D|}{\text{avgdl}}\right)}$$
- Ideal for exact term matching, part numbers, and verbatim legal clauses.

### 1.3 Hybrid Search & Reciprocal Rank Fusion (RRF)
Modern enterprise search combines both Dense (semantic) and Sparse (BM25) search. To merge results from different scoring scales without score calibration issues, **Reciprocal Rank Fusion (RRF)** is used:

$$\text{RRF\_Score}(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
Where:
- $M$ is the set of retrieval systems (e.g., dense and sparse).
- $r_m(d)$ is the rank of document $d$ in system $m$ (1-indexed).
- $k$ is a smoothing constant (typically $k = 60$).

---

## 2. SIMILARITY & DISTANCE METRICS

| Metric | Mathematical Formula | Range | When to Use |
| :--- | :--- | :--- | :--- |
| **Cosine Similarity** | $\cos(\theta) = \frac{u \cdot v}{\|u\|_2 \|v\|_2} = \frac{\sum u_i v_i}{\sqrt{\sum u_i^2}\sqrt{\sum v_i^2}}$ | $[-1, 1]$ | Text embeddings of varying lengths |
| **Dot Product (Inner Product)** | $\langle u, v \rangle = \sum_{i=1}^d u_i v_i$ | $(-\infty, \infty)$ | When vectors are pre-normalized to unit length ($\|u\|_2 = 1$) |
| **Euclidean Distance ($L_2$)** | $d_{L2}(u, v) = \sqrt{\sum_{i=1}^d (u_i - v_i)^2}$ | $[0, \infty)$ | Geometric spatial clustering, physical coordinates |
| **Manhattan Distance ($L_1$)** | $d_{L1}(u, v) = \sum_{i=1}^d |u_i - v_i|$ | $[0, \infty)$ | Grid-like feature spaces, outlier-heavy data |

### Mathematical Proof: Equivalence of $L_2$ Distance and Cosine Similarity on Normalized Vectors
Let $u, v \in \mathbb{R}^d$ be unit vectors such that $\|u\|_2 = 1$ and $\|v\|_2 = 1$.
The squared $L_2$ distance is:
$$\|u - v\|_2^2 = (u - v) \cdot (u - v) = u \cdot u - 2(u \cdot v) + v \cdot v$$
Since $\|u\|_2^2 = u \cdot u = 1$ and $\|v\|_2^2 = v \cdot v = 1$:
$$\|u - v\|_2^2 = 1 - 2(u \cdot v) + 1 = 2 - 2(u \cdot v)$$
Because vectors are unit length, $u \cdot v = \cos(u, v)$:
$$\|u - v\|_2^2 = 2\left(1 - \cos(u, v)\right)$$
$$\|u - v\|_2 = \sqrt{2\left(1 - \text{CosineSimilarity}(u, v)\right)}$$
**Key Consequence:** When embeddings are normalized ($\ell_2$-norm = 1), optimizing Dot Product, Cosine Similarity, and Euclidean Distance yields the **exact same ranking order**, but Dot Product requires zero divisions or square roots, saving massive GPU/CPU compute!

---

## 3. APPROXIMATE NEAREST NEIGHBOR (ANN) INDEXING

Exact k-Nearest Neighbor search (Flat Index) requires comparing the query vector against every single vector in the database:
$$\text{Time Complexity} = O(N \cdot d)$$
For $N = 10,000,000$ documents and $d = 1,536$, a single query takes billions of FLOPs ($\sim 500\text{ms}$), making real-time search impossible. ANN algorithms sacrifice $\approx 1-3\%$ accuracy (recall) to achieve $\approx 100\times$ faster query latency ($O(\log N)$).

```
                      ANN ALGORITHM TAXONOMY
                                |
         +----------------------+----------------------+
         |                                             |
   Graph-Based                                   Clustering / Quantization
   (HNSW - State of the art)                     (IVF, Product Quantization)
         |                                             |
   High search speed, high recall,               Low memory footprint,
   larger memory footprint                       fast build time
```

### 3.1 Hierarchical Navigable Small World (HNSW) Graphs
HNSW (Malkov & Yashunin, 2018) builds a multi-layer graph inspired by **Skip-Lists**:

```
Layer 2 (Express):    [Node A] -------------------------------> [Node D]
                         |                                         |
Layer 1 (Regional):   [Node A] --------------> [Node B] ------> [Node D]
                         |                        |                |
Layer 0 (Dense/All):  [Node A] -> [Node C] -> [Node B] -> [E] -> [Node D]
```

- **Top Layers:** Sparse graphs with long-range connections. Search begins at the top layer, rapidly traversing large distances across vector space (greedy routing).
- **Bottom Layer (Layer 0):** Dense graph containing all vector nodes with short-range local connections.
- **Search Complexity:** $O(\log N)$ query time.
- **Key Parameters:**
  - $M$: Maximum number of bi-directional links per node (typically $16$ to $64$).
  - `efConstruction`: Size of dynamic candidate list during graph construction (higher = higher recall, slower build).
  - `efSearch`: Size of dynamic candidate list during query time (trades off query latency vs recall).

### 3.2 Product Quantization (PQ) Vector Compression
For a 10M vector database with $d=1536$ in FP32, storing raw vectors requires:
$$\text{RAM} = 10^7 \times 1536 \times 4 \text{ bytes} \approx 61.44 \text{ GB}$$
Product Quantization compresses vectors:
1. Splits a $d$-dimensional vector into $m$ distinct subvectors (e.g., $d=1536$, $m=64 \to 24$ dims each).
2. Runs k-means on each subvector space to create $k=256$ centroids.
3. Each subvector is replaced by an 8-bit integer ($2^8 = 256$) representing its centroid ID.
4. Total memory shrinks from $1536 \times 4 \text{ bytes} = 6144\text{ bytes}$ down to $64 \times 1 \text{ byte} = 64\text{ bytes}$—a **$96\times$ compression factor**!

---

## 4. BI-ENCODER VS CROSS-ENCODER

```
BI-ENCODER (Retrieval Stage)                CROSS-ENCODER (Re-ranking Stage)

Query   --> [Encoder A] --> [Vector Q] \     [Query + Document]
                                       > Dot Product          |
Document -> [Encoder B] --> [Vector D] /                      v
                                                    [Joint Cross-Encoder]
                                                              |
                                                              v
                                                    Relevance Score [0.0 - 1.0]
- Independent embeddings                     - Full cross-attention between every
- Documents pre-computed offline               query token and document token
- Extremely fast: O(1) dot product           - High latency: cannot precompute!
- Retrieves top-100 candidates               - Re-ranks top-100 down to top-5
```

---

## 5. VECTOR DATABASE COMPARISON MATRIX

| Vector DB | Architecture Type | Indexing Options | Best Used For |
| :--- | :--- | :--- | :--- |
| **Chroma** | Embedded / Lightweight | HNSW (hnswlib) | Prototyping, local development, Python-first |
| **Pinecone** | Fully Managed Cloud SaaS | Proprietary Graph / IVF | Enterprise zero-ops, managed autoscaling |
| **Qdrant** | Standalone (Rust-based) | HNSW with payload filtering | High-performance filtering, hybrid search |
| **Weaviate** | Standalone (Go-based) | HNSW, Dynamic PQ | Native vector + keyword search + GraphQL |
| **Milvus** | Distributed Cluster (Go/C++) | HNSW, IVF-PQ, ScaNN, GPU | Massive scale (100M to 1B+ vectors) |
| **pgvector** | PostgreSQL Extension | IVFFlat, HNSW | Storing vectors in existing relational database |

---

## 6. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: Why can't we use exact k-NN search in production vector databases?**
   - **A:** Exact k-NN computes pairwise similarity across all $N$ vectors ($O(N \cdot d)$). At 10 million vectors, every query performs tens of billions of floating point operations, taking hundreds of milliseconds. Approximate Nearest Neighbor (ANN) algorithms (like HNSW) trade away a negligible amount of recall ($1-2\%$) to traverse the index in $O(\log N)$ time, responding in milliseconds.

2. **Q: What is the relationship between Dot Product and Cosine Similarity?**
   - **A:** Cosine similarity is the dot product divided by the product of vector magnitudes: $\frac{u \cdot v}{\|u\|_2 \|v\|_2}$. If all vectors are pre-normalized to unit length ($\|u\|_2 = 1$), the denominator equals 1, making the dot product mathematically identical to cosine similarity while saving division and norm computations during search.

### Intermediate
3. **Q: How does the HNSW graph guarantee $O(\log N)$ search without getting trapped in local minima?**
   - **A:** HNSW utilizes a multi-layer hierarchical structure inspired by skip-lists. The highest layers have long-range links across distant clusters, allowing the greedy search algorithm to jump across the global vector space without getting stuck in local neighborhoods. As the search descends through progressively denser layers, it refines its search radius until reaching the dense base layer (Layer 0).

4. **Q: Why should production RAG systems use a Two-Stage Retrieval pipeline (Bi-Encoder + Cross-Encoder)?**
   - **A:** Bi-encoders compress documents independently into single vectors, enabling instantaneous ANN vector search across millions of documents, but they miss fine-grained token-level cross-interactions. A Cross-Encoder processes the query and candidate document together via full self-attention, capturing exact linguistic nuance, but is too computationally heavy to evaluate millions of candidates. Two-stage retrieval combines both: Bi-encoder retrieves top 50–100 candidates in milliseconds, and the Cross-Encoder re-ranks those top 50 candidates to select the final top 3–5 chunks with maximum precision.

### Advanced
5. **Q: What is Reciprocal Rank Fusion (RRF) and why is it superior to weighted score combination in Hybrid Search?**
   - **A:** Dense vector search outputs cosine similarities (typically in range $0.5 - 0.9$), while sparse search outputs unbounded BM25 scores (e.g., $0$ to $40+$). Combining raw scores requires complex normalization, which is unstable across different queries. RRF ignores absolute score magnitudes entirely, looking only at the integer rank position of documents across both lists: $\sum \frac{1}{60 + \text{rank}_i}$. This makes hybrid fusion invariant to score scale differences and robust against distribution shifts.

---

## 7. RAPID REVISION SUMMARY

- **Dense vs Sparse:** Dense captures semantic meaning; Sparse (BM25) captures exact keywords.
- **Metric Equivalence:** On $\ell_2$-normalized vectors, $\|u-v\|^2 = 2 - 2(u \cdot v)$. Inner product ordering matches cosine ordering.
- **HNSW:** Multi-layer skip-list graph. Top layers = long-range routing; bottom layer = local refinement. $O(\log N)$ search.
- **Product Quantization:** Divides vector into $m$ subvectors, maps to nearest centroids $\to$ 90%+ VRAM savings.
- **Two-Stage Retrieval:** Bi-encoder fast candidate retrieval (top 100) $\to$ Cross-encoder deep re-ranking (top 5).
- **RRF Formula:** $\sum \frac{1}{k + \text{rank}(d)}$, robustly merges dense and sparse results.

---

## 8. RECALL CHECKLIST

- [ ] Write the mathematical formulas for Cosine Similarity and Dot Product.
- [ ] Prove why Euclidean distance ordering equals Cosine similarity ordering on normalized vectors.
- [ ] Diagram the multi-layer HNSW graph search process.
- [ ] Explain how Product Quantization compresses a 1536-dimensional FP32 vector.
- [ ] Contrast Bi-Encoders and Cross-Encoders in terms of architecture, speed, and accuracy.
- [ ] Write the Reciprocal Rank Fusion (RRF) equation and explain the role of constant $k=60$.
- [ ] Compare pgvector, Qdrant, and Pinecone for enterprise deployments.
