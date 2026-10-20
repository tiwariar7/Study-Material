# Day 28: Master Revision — Complete Syllabus Cheat Sheet, Master Formula Compendium & Top 50 Interview Bank

---

## 1. ONE-DAY EXECUTIVE FLASHCARD SYLLABUS

```
  DAY 1-2: SUPERVISED ML
  • Linear / Logistic Regression: MSE vs Binary Cross-Entropy; Log-odds link function.
  • Trees & Forests: Gini Impurity vs Entropy; Bagging reduces variance; Out-of-Bag (OOB) error.
  • SVM: Max-margin hyperplanes; Hinge loss; Kernel trick (RBF: K(x, z) = exp(-γ||x-z||²)).
  • Evaluation: Precision, Recall, F1-Score, ROC-AUC (invariant to class imbalance thresholding).

  DAY 3-4: UNSUPERVISED ML & DIMENSIONALITY REDUCTION
  • K-Means: Lloyd's algorithm; Voronoi partitions; Silhouette Score: (b - a) / max(a, b).
  • DBSCAN: Density reachability (Core, Border, Noise points); handles arbitrary cluster geometries.
  • PCA: Orthogonal projection onto eigenvectors of covariance matrix; captures maximum variance.

  DAY 5-6: NLP FOUNDATIONS & TEXT REPRESENTATION
  • Tokenization: Byte-Pair Encoding (BPE) merges frequent pairs; WordPiece; SentencePiece.
  • TF-IDF: TF * log(N / df); discounts ubiquitous stopwords, highlights rare salient words.
  • Word2Vec: CBOW (context -> target) vs Skip-Gram (target -> context) with Negative Sampling.

  DAY 7-8: DEEP LEARNING FOUNDATIONS
  • MLP: Forward propagation -> Loss computation -> Backpropagation via chain rule.
  • Activations: ReLU (dying neurons) -> GELU -> SwiGLU (Swish-gated linear units).
  • Optimizers: SGD + Momentum -> RMSProp -> Adam (1st & 2nd moments) -> AdamW (decoupled weight decay).

  DAY 9-10: SEQUENCE MODELING & TRANSFORMERS
  • RNN / LSTM: Sequential bottleneck; LSTM forget/input/output gates preserve gradient flow.
  • Self-Attention: Softmax((Q·K^T) / √d_k) · V. O(N²) complexity; scaled by √d_k to prevent saturation.
  • MHA vs GQA: Multi-Head (1:1) vs Grouped-Query (H:G) reduces KV cache memory by 8x.
  • Positional Encodings: Sinusoidal (relative offset) vs RoPE (rotates Q, K in complex plane).

  DAY 11-13: LLM FOUNDATIONS, SCALING LAWS & MoE
  • Training Pipeline: Pre-training (98% compute) -> SFT (format) -> Preference Alignment (values).
  • Chinchilla Optimal: Tokens D ≈ 20 * Parameters N. Production models overtrain for inference economics.
  • Mixture of Experts (MoE): Router selects top-k experts; Mixtral 8x7B (47B total, 13B active).
  • vLLM & PagedAttention: OS-style virtual memory paging eliminates KV cache fragmentation.

  DAY 14-17: PROMPT ENGINEERING, EMBEDDINGS & VECTOR STORES
  • Prompting: Few-shot, CoT ("Let's think step by step"), ReAct (Thought -> Action -> Observation).
  • Structured Output: Logit masking via Finite State Machines guarantees 100% valid JSON.
  • Embeddings: Bi-Encoder (fast retrieval) vs Cross-Encoder (accurate re-ranking).
  • HNSW: Multi-layer skip-list graph for O(log N) Approximate Nearest Neighbor search.
  • RRF: Reciprocal Rank Fusion: ∑ 1 / (60 + rank_i) combines BM25 and Vector search.

  DAY 18-20: FINE-TUNING & ADVANCED RAG
  • LoRA: ΔW = (α/r) · B · A. Freezes base weights; merges with zero inference latency overhead.
  • QLoRA: 4-bit NormalFloat (NF4) base weights + Double Quantization + Paged Optimizers.
  • Advanced RAG: Parent-Document chunking (Small-to-Big) + HyDE + Cross-Encoder Re-ranking.
  • Ragas Triad: Context Relevance (retrieval), Faithfulness (hallucination), Answer Relevance.

  DAY 21-27: AGENTS, GRAPHS & PRODUCTION DEPLOYMENT
  • LangGraph: Cyclic state machine with TypedDict state and reducers; supports Human-in-the-Loop.
  • Graph RAG: Microsoft GraphRAG uses Leiden community detection for global multi-hop summarization.
  • Inference SLA: TTFT (Prefill, compute-bound) vs TPOT (Decode, memory-bandwidth-bound).
  • Serving Optimizations: Continuous batching + Speculative decoding + Semantic caching.
```

---

## 2. MASTER FORMULA COMPENDIUM

### 2.1 Classical ML & Evaluation
- **Linear Regression Cost (MSE):**
  $$J(\theta) = \frac{1}{2m} \sum_{i=1}^m \left(h_\theta(x^{(i)}) - y^{(i)}\right)^2$$
- **Sigmoid Function:**
  $$\sigma(z) = \frac{1}{1 + e^{-z}}$$
- **Binary Cross-Entropy Loss (Log Loss):**
  $$\mathcal{L}_{BCE} = -\frac{1}{m} \sum_{i=1}^m \left[ y^{(i)} \log(\hat{y}^{(i)}) + (1 - y^{(i)}) \log(1 - \hat{y}^{(i)}) \right]$$
- **Gini Impurity & Entropy:**
  $$\text{Gini} = 1 - \sum_{i=1}^C p_i^2, \quad \text{Entropy} = -\sum_{i=1}^C p_i \log_2(p_i)$$
- **Precision, Recall, F1:**
  $$\text{Precision} = \frac{TP}{TP + FP}, \quad \text{Recall} = \frac{TP}{TP + FN}, \quad F_1 = \frac{2 \cdot P \cdot R}{P + R}$$
- **Silhouette Coefficient:**
  $$s(i) = \frac{b(i) - a(i)}{\max(a(i), b(i))}$$

---

### 2.2 Deep Learning & Optimization
- **Softmax Function:**
  $$\text{Softmax}(z)_i = \frac{e^{z_i}}{\sum_{j=1}^K e^{z_j}}$$
- **GELU (Gaussian Error Linear Unit):**
  $$\text{GELU}(x) = x \cdot \Phi(x) \approx 0.5x \left(1 + \tanh\left(\sqrt{2/\pi}(x + 0.044715x^3)\right)\right)$$
- **SwiGLU Activation:**
  $$\text{SwiGLU}(x) = \left(\text{Swish}(x W_{gate}) \odot x W_{up}\right) W_{down}$$
- **RMSNorm (Root Mean Square Normalization):**
  $$\text{RMSNorm}(x) = \frac{x}{\sqrt{\frac{1}{d}\sum_{i=1}^d x_i^2 + \epsilon}} \odot \gamma$$
- **AdamW Weight Update (Decoupled Decay):**
  $$m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t, \quad v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2$$
  $$\theta_t = \theta_{t-1} - \eta \left( \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} + \lambda \theta_{t-1} \right)$$

---

### 2.3 Attention & Transformers
- **Scaled Dot-Product Attention:**
  $$\text{Attention}(Q, K, V) = \text{Softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right) V$$
- **Multi-Head Attention:**
  $$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \dots, \text{head}_h) W^O, \quad \text{head}_i = \text{Attention}(Q W_i^Q, K W_i^K, V W_i^V)$$
- **Sinusoidal Positional Encoding:**
  $$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d}}\right), \quad PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d}}\right)$$
- **Distance Metric Equivalence on Unit Vectors ($\|u\|_2 = \|v\|_2 = 1$):**
  $$\|u - v\|_2^2 = 2 - 2(u \cdot v) = 2(1 - \cos(u, v))$$

---

### 2.4 LLMs, Scaling Laws & Alignment
- **Autoregressive Causal LM Loss:**
  $$\mathcal{L}_{CLM}(\theta) = -\sum_{t=1}^T \log P(x_t \mid x_{<t}; \theta)$$
- **Training Compute Approximation:**
  $$C \approx 6 \cdot N \cdot D \text{ FLOPs}$$
- **Chinchilla Compute-Optimal Ratio:**
  $$D \approx 20 \cdot N \text{ tokens}$$
- **KV Cache Memory Size (Bytes):**
  $$\text{Memory}_{KV} = 2 \times L \times B \times S \times N_{kv} \times d_{head} \times P$$
- **Bradley-Terry Preference Likelihood:**
  $$P(y_w \succ y_l \mid x) = \sigma(r(x, y_w) - r(x, y_l))$$
- **Direct Preference Optimization (DPO) Loss:**
  $$\mathcal{L}_{DPO}(\theta) = -\mathbb{E}_{(x, y_w, y_l)}\left[\log \sigma\left(\beta \log \frac{\pi_\theta(y_w \mid x)}{\pi_{ref}(y_w \mid x)} - \beta \log \frac{\pi_\theta(y_l \mid x)}{\pi_{ref}(y_l \mid x)}\right)\right]$$
- **LoRA Forward Pass:**
  $$h = W_0 x + \frac{\alpha}{r}(B \cdot A) x$$

---

### 2.5 RAG, Search & Production Serving
- **BM25 Scoring:**
  $$\text{BM25}(D, Q) = \sum_{i=1}^n \text{IDF}(q_i) \cdot \frac{f(q_i, D)(k_1 + 1)}{f(q_i, D) + k_1(1 - b + b \cdot \frac{|D|}{\text{avgdl}})}$$
- **Reciprocal Rank Fusion (RRF):**
  $$\text{RRF\_Score}(d) = \sum_{m \in M} \frac{1}{k + r_m(d)} \quad (k \approx 60)$$
- **Maximal Marginal Relevance (MMR):**
  $$\text{MMR} = \arg\max_{d_i \in R \setminus S} \left[ \lambda \cdot \text{Sim}_1(d_i, Q) - (1 - \lambda) \max_{d_j \in S} \text{Sim}_2(d_i, d_j) \right]$$
- **End-to-End Inference Latency:**
  $$\text{Latency}_{E2E} = \text{TTFT} + (N_{out} - 1) \times \text{TPOT}$$

---

## 3. ULTIMATE 100-CONCEPT ONE-LINER CHEAT SHEET

1. **Bias-Variance Tradeoff:** High bias underfits; high variance overfits; total error = bias² + variance + irreducible error.
2. **L1 vs L2 Regularization:** L1 (Lasso) induces sparsity; L2 (Ridge) shrinks weights smoothly without zeroing them out.
3. **ROC-AUC:** Probability that a classifier ranks a random positive instance higher than a random negative instance.
4. **Ensemble Bagging:** Trains models in parallel on bootstrapped subsets; reduces variance (Random Forest).
5. **Ensemble Boosting:** Trains models sequentially, focusing on past errors; reduces bias (XGBoost, LightGBM).
6. **Curse of Dimensionality:** As dimensions increase, data points become equidistant and space becomes exponentially sparse.
7. **PCA:** Finds orthogonal principal components along directions of maximum variance via eigendecomposition.
8. **DBSCAN:** Density-based clustering that discovers arbitrary non-spherical clusters and marks isolated points as noise.
9. **K-Means++:** Initializes centroids far apart to guarantee $O(\log k)$ competitive clustering bounds.
10. **Word2Vec Negative Sampling:** Replaces expensive full-vocabulary softmax with binary logistic regression against $k$ noise words.
11. **Subword Tokenization (BPE):** Iteratively merges most frequent byte/character pairs, handling unseen words with zero OOV.
12. **Vanishing Gradient:** Gradients exponentially decay to zero in deep networks due to repeated multiplication by fractional derivatives.
13. **Exploding Gradient:** Gradients exponentially multiply to NaN/infinity; mitigated via Gradient Clipping.
14. **Residual Connections (ResNet):** Identity shortcut $x + F(x)$ provides an unobstructed highway for gradient backpropagation.
15. **Batch Normalization vs Layer Normalization:** BatchNorm normalizes across the batch; LayerNorm normalizes across feature dimensions independently per sample.
16. **Pre-LN Transformer:** Normalizes inputs before sub-layers; ensures stable residual streams without learning rate warmup.
17. **Scaled Dot-Product Attention:** Measures query-key similarity normalized by $\sqrt{d_k}$ to prevent softmax saturation.
18. **Multi-Head Attention:** Projects queries, keys, and values into $h$ parallel subspaces to capture diverse linguistic relations.
19. **RoPE (Rotary Position Embedding):** Rotates Q and K in 2D planes, ensuring dot products encode relative distances naturally.
20. **ALiBi:** Replaces positional embeddings with static negative linear bias slopes proportional to token distances.
21. **SwiGLU:** Gated linear unit using Swish activation; standard FFN replacement in Llama and modern LLMs.
22. **RMSNorm:** Eliminates mean centering from LayerNorm, reducing compute overhead while retaining normalization stability.
23. **Autoregressive Generation:** Generating text token-by-token where each step conditions on all previously generated tokens.
24. **Causal Masking:** Upper-triangular matrix set to $-\infty$ preventing attention from looking ahead at future tokens.
25. **KV Cache:** Retains past key and value vectors in VRAM, turning autoregressive decoding from $O(T^2)$ to $O(T)$ compute.
26. **Grouped-Query Attention (GQA):** Multiple query heads share a single key-value head, slashing KV cache memory by $8\times$.
27. **Multi-Query Attention (MQA):** All query heads share exactly one key head and one value head.
28. **Chinchilla Scaling Law:** Model parameters and training tokens should scale 1:1; compute-optimal ratio is $D \approx 20N$.
29. **Fill-In-the-Middle (FIM):** Pre-training transformation that teaches models to generate missing middle code conditioned on prefix and suffix.
30. **Emergent Abilities:** Sudden jumps in capabilities at critical parameter scale; often exaggerated by non-linear metrics.
31. **Base Model:** Pure next-token prediction model trained on internet text; lacks conversational instruction-following alignment.
32. **Supervised Fine-Tuning (SFT):** Training on curated prompt-response pairs to teach the model assistant formatting and tone.
33. **RLHF:** Optimizes policy against a learned reward model via PPO with a KL penalty to prevent reward hacking.
34. **Bradley-Terry Model:** Mathematical preference formulation: $P(y_w \succ y_l) = \sigma(r(y_w) - r(y_l))$.
35. **DPO (Direct Preference Optimization):** Mathematically cancels out the reward model, optimizing policy directly on preference pairs.
36. **Mixture of Experts (MoE):** Replaces dense FFN with multiple expert networks routed dynamically by a gating network.
37. **Active vs Total Parameters:** Total parameters dictate VRAM requirements; active parameters dictate forward-pass compute latency.
38. **MoE Auxiliary Loss:** Penalizes non-uniform expert routing to prevent expert collapse/starvation during pre-training.
39. **vLLM PagedAttention:** Manages KV cache in non-contiguous memory blocks like OS virtual paging, ending memory fragmentation.
40. **Continuous Batching:** Schedules requests dynamically at iteration/token level, replacing completed sequences immediately.
41. **Speculative Decoding:** Small draft model proposes $K$ tokens; large target model validates in 1 parallel forward pass.
42. **TTFT (Time To First Token):** Latency during prefill phase; compute-bound on GPU Tensor Cores.
43. **TPOT (Time Per Output Token):** Inter-token generation latency; memory-bandwidth-bound by GPU HBM bus speed.
44. **Catastrophic Forgetting:** Overwriting generalized pre-trained representations when fine-tuning heavily on a narrow task.
45. **PEFT:** Parameter-Efficient Fine-Tuning; updates $<1\%$ of weights to prevent forgetting and slash VRAM usage.
46. **LoRA:** Represents weight updates as low-rank decomposition $\Delta W = B \cdot A$; merges with zero inference latency.
47. **LoRA Rank ($r$):** The inner dimension of adapter matrices; controls capacity and expressiveness (typically 8 to 64).
48. **LoRA Alpha ($\alpha$):** Scaling hyperparameter; keeps adaptation magnitude stable across different rank choices.
49. **QLoRA:** Fine-tunes 4-bit NormalFloat quantized base models with 16-bit LoRA adapters and paged optimizers.
50. **NormalFloat4 (NF4):** Information-theoretically optimal 4-bit quantile mapping for normally distributed neural weights.
51. **Double Quantization:** Quantizes the quantization constants themselves, saving 0.37 bits per parameter.
52. **Paged Optimizers:** Uses CUDA Unified Memory to page memory spikes from GPU VRAM to CPU RAM, preventing OOM crashes.
53. **AWQ (Activation-aware Weight Quantization):** Protects the top 1% salient weight channels to maintain accuracy in 4-bit PTQ.
54. **GPTQ:** One-shot post-training quantization using inverse Hessian approximations.
55. **GGUF:** Binary format for running quantized models efficiently on CPU and edge devices via llama.cpp.
56. **Zero-Shot CoT:** Eliciting multi-step reasoning by appending `"Let's think step by step"`.
57. **Self-Consistency:** Generates multiple reasoning paths at high temperature and selects answer via majority voting.
58. **Least-to-Most Prompting:** Decomposes complex problems into sub-problems, solving them sequentially.
59. **Tree-of-Thoughts (ToT):** Explores tree of intermediate thoughts using BFS/DFS with heuristic self-evaluations.
60. **ReAct Pattern:** Interleaves Thought, Action (Tool Call), and Observation in an iterative loop.
61. **Constrained Decoding:** Uses Finite State Machines to mask invalid vocabulary logits, guaranteeing 100% valid JSON schemas.
62. **Direct Prompt Injection:** User inputs designed to override system prompt directives.
63. **Indirect Prompt Injection:** Malicious instructions embedded in external web/PDF data ingested by RAG or autonomous agents.
64. **Lost in the Middle:** Phenomenon where LLMs recall information at the start and end of context far better than the center.
65. **Bi-Encoder:** Embeds query and documents independently into dense vectors; enables fast ANN retrieval.
66. **Cross-Encoder:** Jointly attends to query and document simultaneously; computationally slow but yields superior re-ranking accuracy.
67. **HNSW:** Hierarchical Navigable Small World; multi-layer skip-list graph for $O(\log N)$ approximate nearest neighbor search.
68. **Product Quantization (PQ):** Splits vectors into subvectors and maps to centroids, achieving 90%+ VRAM savings.
69. **BM25:** Probabilistic lexical search algorithm scoring documents based on term frequency and inverted document frequency.
70. **SPLADE:** Learned sparse representation model performing term expansion in vocabulary space.
71. **Hybrid Search:** Combines dense semantic search with sparse lexical search via Reciprocal Rank Fusion.
72. **Reciprocal Rank Fusion (RRF):** $\sum \frac{1}{60 + \text{rank}_i}$; rank-based combination invariant to score distribution scales.
73. **Small-to-Big Retrieval:** Indexes small child chunks for sharp vector search; passes larger parent chunk to LLM for rich context.
74. **Anthropic Contextual Retrieval:** Prepends document-level summary context to each chunk prior to embedding.
75. **HyDE:** Generates a hypothetical answer to embed, bridging the syntactic gap between questions and documents.
76. **Maximal Marginal Relevance (MMR):** Balances query relevance against redundancy penalty among retrieved documents.
77. **Context Relevance (Ragas):** Ratio of relevant sentences in retrieved context to total retrieved sentences.
78. **Faithfulness / Groundedness (Ragas):** Percentage of claims in generated answer that are directly verifiable from retrieved context.
79. **Answer Relevance (Ragas):** Degree to which the output directly addresses the question asked.
80. **LCEL:** LangChain Expression Language; declarative composition protocol supporting streaming, batching, and pipes (`|`).
81. **LangGraph:** Stateful, cyclic multi-agent orchestration framework modeling workflows as graph state machines.
82. **LangGraph State Reducer:** Functions like `operator.add` that dictate how state updates append to existing lists instead of overwriting.
83. **Checkpointer (LangGraph):** Persists state snapshots at every node boundary to enable human-in-the-loop and conversational persistence.
84. **Human-in-the-Loop (`interrupt_before`):** Pauses graph execution before dangerous node execution to await human review.
85. **LlamaIndex Node:** Atomic unit of data in LlamaIndex preserving parent, child, and sequential graph relationships.
86. **Sentence-Window Retrieval:** Embeds single sentences; dynamically swaps in surrounding sentence window before LLM synthesis.
87. **Auto-Merging Retriever:** Automatically merges leaf nodes into their parent chunk if sufficient sibling leaves match.
88. **LlamaIndex `compact` Mode:** Stuffs maximum retrieved chunks into prompt to minimize API calls.
89. **Knowledge Graph Triplet:** Fundamental relational unit: $(Subject, Predicate, Object)$.
90. **Labeled Property Graph:** Nodes and directed edges containing labels and key-value attribute maps.
91. **Cypher:** Declarative pattern-matching graph query language used in Neo4j (`MATCH (a)-[:REL]->(b)`).
92. **Microsoft GraphRAG:** Ingests text $\to$ builds KG $\to$ Leiden community clustering $\to$ hierarchical community summary search.
93. **Multi-Hop Reasoning:** Query requiring traversal across multiple disjoint documents connected by intermediate entities.
94. **Reflexion:** Agentic verbal self-correction; failure critiques stored in episodic memory to condition future attempts.
95. **Multi-Agent Supervisor Pattern:** High-level coordinator delegating specialized tasks to worker agents with restricted toolsets.
96. **Multi-Agent Debate:** Diverse agent personas critiquing candidate answers across rounds to converge on truth.
97. **Semantic Caching:** Vector similarity search over past query embeddings to return instant cached responses for similar queries.
98. **Model Cascading:** Routing simple queries to cheap 8B models and complex queries to 70B/frontier models.
99. **Prompt Caching:** Provider-level KV cache reuse for static prompt prefixes, saving up to 90% cost and latency.
100. **LLMOps Observability:** Tracking latency traces, token usage, cost, and LLM-as-a-judge scores in production (Langfuse).

---

## 4. TOP 50 MASTER TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Group A: Machine Learning & NLP Fundamentals
1. **Q: Why does Logistic Regression use Log Loss instead of Mean Squared Error?**
   - **A:** Squaring the difference of a Sigmoid function produces a non-convex loss surface riddled with local minima. Log Loss (derived from Maximum Likelihood Estimation) produces a smooth, strictly convex loss surface, guaranteeing that gradient descent converges to the global minimum.

2. **Q: How does Random Forest reduce variance while Boosting reduces bias?**
   - **A:** Random Forest trains many deep, low-bias trees independently on bootstrapped data (Bagging) and averages them; averaging $B$ uncorrelated estimators reduces variance by $\approx 1/B$ without increasing bias. Boosting trains shallow, high-bias trees sequentially, each learning to predict the residuals of preceding trees, progressively reducing systematic bias.

3. **Q: Explain the mathematical intuition behind Word2Vec Negative Sampling.**
   - **A:** The standard Skip-Gram objective calculates softmax over the entire vocabulary $V$ ($\sum_{w \in V} \exp$), requiring hundreds of thousands of dot products per step. Negative Sampling reformulates the task as binary logistic regression: distinguish the true target word (label 1) from $k$ randomly drawn noise words (label 0). This slashes computational complexity from $O(|V|)$ to $O(k)$ per step.

4. **Q: Why does AdamW outperform standard Adam when using L2 weight decay?**
   - **A:** In standard Adam, L2 regularization is added directly to the gradient vector ($g_t + \lambda \theta$). Because Adam divides gradients by the second moment $\sqrt{v_t}$, weights with large historical gradients experience less weight decay, while weights with infrequent gradients experience more decay. AdamW decouples weight decay entirely from the adaptive gradient update: $\theta_t = \theta_{t-1} - \eta \hat{m}_t / (\sqrt{\hat{v}_t} + \epsilon) - \eta \lambda \theta_{t-1}$, restoring true weight shrinkage.

---

### Group B: Transformers & Modern Attention
5. **Q: Why do Transformers divide the dot product by $\sqrt{d_k}$ in Scaled Dot-Product Attention?**
   - **A:** If components of $Q$ and $K$ are independent with variance 1, their dot product has variance $d_k$ and standard deviation $\sqrt{d_k}$. For large $d_k$, dot products become very large, pushing the Softmax function into saturation regions where gradients approach zero. Dividing by $\sqrt{d_k}$ scales variance back to 1.0, preserving healthy gradient flow during backpropagation.

6. **Q: Why does Pre-LN train more stably than Post-LN in deep Transformers?**
   - **A:** In Post-LN ($x_{l+1} = \text{LN}(x_l + F(x_l))$), gradients must pass through LayerNorm at every residual step, which can scale down or distort early layer gradients in deep networks. In Pre-LN ($x_{l+1} = x_l + F(\text{LN}(x_l))$), the residual identity stream $x_{l+1} = x_l + \dots$ has an unimpeded linear connection from output to input, enabling smooth gradient backpropagation without requiring warmup schedules.

7. **Q: What is Rotary Position Embedding (RoPE) and why is it preferred over learned absolute positional embeddings?**
   - **A:** RoPE rotates Query and Key vectors in 2D chunks by an angle proportional to their sequence index ($m\theta$). The dot product between rotated vectors $\langle R_m q, R_n k \rangle$ becomes a direct function of their relative distance $(m - n)$ rather than absolute positions. This improves relative position modeling and allows length extrapolation techniques (like YaRN or RoPE scaling) to extend context windows beyond training limits.

8. **Q: Contrast Multi-Head Attention (MHA), Multi-Query Attention (MQA), and Grouped-Query Attention (GQA).**
   - **A:** MHA uses $H$ query, $H$ key, and $H$ value heads ($1:1:1$), maximizing expressive power but creating huge KV caches during decoding. MQA shares 1 key and 1 value head across all $H$ query heads ($H:1:1$), minimizing KV cache size but occasionally hurting accuracy. GQA clusters $H$ query heads into $G$ groups ($H:G:G$, typically 8 queries per KV head), slashing KV cache memory by $87.5\%$ while retaining near-identical modeling capacity to MHA.

---

### Group C: LLMs, Training & Alignment
9. **Q: What is the Chinchilla scaling law and its core implication?**
   - **A:** Hoffmann et al. demonstrated that for compute-optimal training, parameter count $N$ and token count $D$ should scale proportionally: $D \approx 20N$. A 70B model requires $\approx 1.4\text{T}$ tokens. Models like GPT-3 (175B on 300B tokens) were severely undertrained. However, for inference efficiency, modern models intentionally overtrain smaller architectures (e.g., Llama 3 8B on 15T tokens) to minimize per-token serving cost.

10. **Q: What is the mathematical connection between RLHF with PPO and Direct Preference Optimization (DPO)?**
    - **A:** In the KL-constrained RLHF objective, the optimal policy $\pi^*$ analytically satisfies $r(x,y) = \beta \log(\pi^*(y|x)/\pi_{ref}(y|x)) + \beta \log Z(x)$. By substituting this expression directly into the Bradley-Terry preference loss, the normalizing constant $Z(x)$ cancels out. DPO optimizes the policy directly on preferred vs dispreferred completions using binary cross-entropy, eliminating the need to train a separate reward model or run PPO reinforcement learning.

11. **Q: Why is an auxiliary load-balancing loss needed during Mixture-of-Experts (MoE) pre-training?**
    - **A:** Without intervention, the router network can suffer from expert collapse: a few experts receive slightly higher weights early on, causing the router to route all tokens to them. These experts receive all the training updates while the remaining experts get starved of gradients and stay untrained. The auxiliary loss penalizes non-uniform token distributions, ensuring all experts learn useful representations.

12. **Q: How does vLLM's PagedAttention eliminate memory waste in KV caching?**
    - **A:** Traditional serving engines pre-allocate contiguous memory blocks for each request up to maximum context length, wasting up to $80\%$ of VRAM on ungenerated tokens and memory fragmentation. PagedAttention divides the KV cache into fixed-size virtual pages stored non-contiguously in physical VRAM (like an operating system virtual memory manager), reducing memory waste to under $4\%$ and allowing $2-4\times$ higher concurrency.

---

### Group D: Prompting & Fine-Tuning
13. **Q: How does grammar-based constrained decoding guarantee valid JSON outputs?**
    - **A:** It converts the target JSON schema or Pydantic model into a Finite State Machine (FSM). At each token generation step, the FSM determines the set of grammatically legal continuation tokens. Any token not in this set has its logit masked to $-\infty$ prior to Softmax, making syntax errors mathematically impossible.

14. **Q: What is the LoRA decomposition formula and why does it have zero added latency during inference?**
    - **A:** $\Delta W = \frac{\alpha}{r}(B \cdot A)$, where $W_0 \in \mathbb{R}^{d \times k}$, $B \in \mathbb{R}^{d \times r}$, $A \in \mathbb{R}^{r \times k}$, with $r \ll \min(d, k)$. Because matrix multiplication is distributive, the trained adapter can be merged directly into the base weights prior to deployment: $W_{final} = W_0 + \frac{\alpha}{r}(B \cdot A)$. At runtime, the model executes a single standard linear forward pass with zero added latency.

15. **Q: Explain the three core innovations of QLoRA.**
    - **A:** (1) **NF4 Quantization:** Information-theoretically optimal 4-bit quantiles for normally distributed weights. (2) **Double Quantization:** Quantizes quantization constants to save 0.37 bits/parameter. (3) **Paged Optimizers:** Uses CUDA Unified Memory to page memory spikes to CPU RAM, preventing out-of-memory errors.

16. **Q: When should you choose RAG over Fine-Tuning?**
    - **A:** Choose RAG when the problem requires dynamic, frequently changing knowledge, access to private document archives, verifiable citations, or zero hallucination tolerance. Choose Fine-Tuning when you need to adapt tone, style, specialized domain syntax, or strict structural formatting without stuffing heavy exemplars into prompt context.

---

### Group E: RAG, Embeddings & Vector Search
17. **Q: Why does Euclidean distance ordering match Cosine similarity ordering on normalized vectors?**
    - **A:** For unit vectors $\|u\| = \|v\| = 1$, $\|u - v\|_2^2 = \|u\|^2 + \|v\|^2 - 2(u \cdot v) = 2 - 2\cos(u, v)$. Thus, minimizing Euclidean distance is mathematically equivalent to maximizing Cosine similarity, allowing systems to use dot products for faster ranking.

18. **Q: Explain how HNSW graphs achieve $O(\log N)$ search complexity.**
    - **A:** HNSW creates a multi-layer graph inspired by skip-lists. Top layers contain sparse graphs with long-range links for rapid global routing across vector space. Bottom layers contain dense graphs with short-range links for local cluster refinement. The search greedily traverses top layers before descending, finding nearest neighbors in logarithmic time.

19. **Q: What is Parent-Document (Small-to-Big) chunking in RAG?**
    - **A:** Small chunks (100 tokens) produce sharp, focused vector embeddings that match search queries accurately, but lack sufficient context for comprehensive generation. Parent-document chunking indexes small child chunks for vector retrieval, but retrieves and supplies the larger parent document (1,000 tokens) to the LLM during answer synthesis.

20. **Q: How does HyDE (Hypothetical Document Embeddings) bridge the semantic gap in retrieval?**
    - **A:** Raw questions and document answers have different grammatical and stylistic structures. HyDE instructs an LLM to generate a hypothetical answer passage first. Embedding this hypothetical answer produces a vector that lands much closer to real document passages in semantic space than the original question vector did.

21. **Q: What are the three pillars of the RAG Triad in Ragas?**
    - **A:** (1) **Context Relevance:** Is the retrieved context relevant and free of distracting noise? (2) **Faithfulness (Groundedness):** Are all claims in the generated answer strictly derived from the context? (3) **Answer Relevance:** Does the response directly address the question asked?

22. **Q: What is Reciprocal Rank Fusion (RRF) and why is it used in Hybrid Search?**
    - **A:** Vector search produces bounded cosine similarities, while BM25 produces unbounded lexical scores. Normalizing and combining these raw scores is fragile. RRF ignores score magnitudes and combines results based purely on their rank positions: $\sum \frac{1}{60 + \text{rank}_i}$, providing robust, scale-invariant hybrid fusion.

---

### Group F: Agents, Orchestration & Production Deployment
23. **Q: Why are Directed Acyclic Graphs (DAGs) insufficient for complex autonomous agents?**
    - **A:** Real agents require iterative cycles: generating code, running unit tests, observing errors, and looping back to self-correct. DAGs enforce strictly forward linear execution without loops. State machine frameworks like LangGraph allow cycles, dynamic conditional branching, and checkpointed persistence.

24. **Q: How does LangGraph implement Human-in-the-Loop workflows?**
    - **A:** LangGraph uses persistent checkpointers (e.g., `PostgresSaver`) to record state snapshots at every node. Using `interrupt_before=["dangerous_action_node"]`, the graph pauses execution before executing a sensitive tool. A human can inspect state, approve or edit variables, and resume execution using the matching `thread_id`.

25. **Q: What is the difference between TTFT and TPOT, and what hardware bottlenecks govern each?**
    - **A:** TTFT (Time to First Token) measures the prefill phase where all prompt tokens are processed in parallel; it is compute-bound by GPU Tensor Cores. TPOT (Time Per Output Token) measures inter-token decoding latency where tokens are emitted autoregressively one by one; it is memory-bandwidth-bound by the speed of reading weights from GPU High-Bandwidth Memory (HBM).

26. **Q: How does Speculative Decoding accelerate inference without altering output quality?**
    - **A:** A small draft model generates $K$ candidate tokens sequentially. The large target model evaluates all $K$ tokens in a single parallel forward pass. Rejection sampling accepts tokens matching the target model's distribution. This delivers a $2-3\times$ speedup while mathematically guaranteeing identical generation distributions.

27. **Q: How does Microsoft GraphRAG handle global dataset summarization?**
    - **A:** It extracts entities and relationships, applies the Leiden clustering algorithm to group nodes into hierarchical communities, and pre-generates summary reports for each community. Global queries are evaluated across these community summaries in parallel and reduced into a holistic answer, solving the global aggregation failure of vector RAG.

28. **Q: What is Semantic Caching and how does it optimize LLM serving costs?**
    - **A:** Rather than caching exact string matches, semantic caching embeds incoming user queries and searches a vector cache of past queries. If similarity exceeds a threshold ($>0.96$), the precomputed answer is returned instantly ($<10\text{ms}$), reducing latency to near zero and eliminating upstream API costs.

29. **Q: Explain the Reflexion architecture for agentic self-correction.**
    - **A:** An Actor agent executes a task; an Evaluator scores the outcome; if it fails, a Reflection model analyzes the failure trajectory and synthesizes an explicit verbal critique. This critique is stored in episodic memory and prepended to future attempts, enabling the agent to learn from mistakes without weight updates.

30. **Q: How do you protect a production RAG application against Indirect Prompt Injection?**
    - **A:** Encapsulate untrusted retrieved context inside strict XML delimiters (`<context>...</context>`); instruct the model that content inside these tags represents passive data rather than executable instructions; deploy a dual-model architecture where an untrusted extractor retrieves facts while a separate privileged controller formulates responses; and run output guardrails (e.g., Llama Guard) to inspect for command injection or data exfiltration attempts.

---

## 5. FINAL EXAM RECALL CHECKLIST

- [ ] Can you derive the Scaled Dot-Product Attention equation and explain the $\sqrt{d_k}$ scaling factor?
- [ ] Can you write down the complete AdamW weight update equation with decoupled weight decay?
- [ ] Can you explain the difference between Pre-LN and Post-LN and why modern models use Pre-LN?
- [ ] Can you calculate the exact KV Cache size for a model given $L, B, S, N_{kv}$, and $d_{head}$?
- [ ] Can you state the Chinchilla optimal token-to-parameter ratio ($D \approx 20N$)?
- [ ] Can you write the LoRA adapter update equation and explain how it merges into base weights?
- [ ] Can you explain the 3 pillars of the Ragas RAG Triad?
- [ ] Can you explain how HNSW graphs achieve $O(\log N)$ ANN vector search?
- [ ] Can you contrast TTFT (compute-bound prefill) vs TPOT (memory-bandwidth-bound decode)?
- [ ] Can you diagram the LangGraph cyclic state machine and explain state reducers?
