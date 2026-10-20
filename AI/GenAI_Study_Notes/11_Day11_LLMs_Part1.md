# Day 11: Large Language Models (Part 1) — Definition, Modern Architecture & Scaling Laws

---

## Topics Covered
1. **LLM Fundamentals & Parameter Scales** (Compute, Data, Parameters)
2. **Modern LLM Architecture Components**:
   - Modern Normalization: **RMSNorm** vs LayerNorm
   - Modern Activations: **SwiGLU** vs GELU / ReLU
   - Positional Encodings: **RoPE** (Rotary Position Embedding) and **ALiBi**
   - Modern Attention Mechanisms: **MHA**, **MQA** (Multi-Query), **GQA** (Grouped-Query)
3. **KV Caching Mechanics & Memory Footprint Calculations**
4. **Subword Tokenization Algorithms** (BPE, Byte-level BPE, WordPiece, SentencePiece)
5. **Scaling Laws** (Kaplan et al. vs Hoffmann et al. / Chinchilla Optimal Ratios)
6. **Architectural Comparison: Llama 2/3 vs Mistral vs GPT-3/4**
7. **PyTorch Implementation: RMSNorm, SwiGLU, and GQA**
8. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. LLM FUNDAMENTALS & PARAMETER SCALES

### 1.1 Definition
A **Large Language Model (LLM)** is an autoregressive deep neural network—typically a decoder-only Transformer with billions to trillions of parameters—trained on massive corpora of tokenized text using the self-supervised objective of next-token prediction:
$$\mathcal{L}_{CLM}(\theta) = -\sum_{t=1}^{T} \log P(x_t \mid x_1, \dots, x_{t-1}; \theta)$$

### 1.2 Scale Milestones

| Model | Release | Parameters ($N$) | Training Tokens ($D$) | Attention Type | Context Window |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **GPT-2 (XL)** | 2019 | 1.5B | 40B | MHA | 1,024 |
| **GPT-3** | 2020 | 175B | 300B | MHA | 2,048 |
| **Chinchilla** | 2022 | 70B | 1.4T | MHA | 2,048 |
| **Llama 1** | 2023 | 65B | 1.4T | MHA + RoPE | 2,048 |
| **Llama 2** | 2023 | 70B | 2.0T | GQA (70B) + RoPE | 4,096 |
| **Mistral 7B**| 2023 | 7B | ~3T | GQA + Sliding Window | 8,192 / 32K |
| **Llama 3** | 2024 | 8B / 70B / 405B | 15T | GQA + RoPE + 128k Vocab | 8,192 / 128K |

---

## 2. MODERN LLM ARCHITECTURAL INNOVATIONS

The original 2017 Transformer has evolved dramatically in modern open-weight and proprietary LLMs.

```
       CLASSIC TRANSFORMER (2017)           MODERN LLM (Llama 3 / Mistral)
     +-----------------------------+       +-----------------------------+
     | Absolute Positional Embed   |  -->  | RoPE (Rotary Embeddings)    |
     | Post-LayerNorm              |  -->  | Pre-RMSNorm                 |
     | Standard MHA (Equal Q,K,V)  |  -->  | GQA (Grouped-Query Attn)    |
     | ReLU / GELU Activation      |  -->  | SwiGLU Activation           |
     | Small Vocab (32k, BPE)      |  -->  | Large Vocab (128k Tiktoken) |
     +-----------------------------+       +-----------------------------+
```

### 2.1 Normalization: RMSNorm (Root Mean Square Normalization)
Zhang & Sennrich (2019) demonstrated that the re-centering operation (mean subtraction) in standard LayerNorm is computationally redundant and provides no regularization benefit.

$$\text{RMSNorm}(x) = \frac{x}{\text{RMS}(x)} \odot \gamma, \quad \text{where } \text{RMS}(x) = \sqrt{\frac{1}{d} \sum_{i=1}^d x_i^2 + \epsilon}$$
- **Advantage:** Saves ~7% to 15% compute time per normalization layer with identical convergence stability.

### 2.2 Activation Function: SwiGLU (Gated Linear Units)
Shazeer (2020) introduced Gated Linear Units with Swish activation for the Transformer feed-forward block:
$$\text{SwiGLU}(x) = \left(\text{Swish}(x W_{gate}) \odot x W_{up}\right) W_{down}$$
Where $\text{Swish}(z) = z \cdot \sigma(\beta z) = \frac{z}{1 + e^{-\beta z}}$ (with $\beta = 1$).
- Instead of 2 matrices ($W_1, W_2$), SwiGLU uses 3 projection matrices ($W_{gate}, W_{up}, W_{down}$).
- To keep parameter count comparable to a standard FFN with hidden dimension $4d$, the hidden dimension of SwiGLU is typically set to:
$$d_{ff} \approx \frac{8}{3} d_{model}$$

### 2.3 Attention Variants: MHA vs MQA vs GQA
During autoregressive generation, reading Key-Value representations from GPU high-bandwidth memory (HBM) to cache is the primary bottleneck.

```
MHA (Multi-Head Attention)     MQA (Multi-Query Attention)     GQA (Grouped-Query Attention)
  Q Q Q Q Q Q Q Q                Q Q Q Q Q Q Q Q                 Q Q   Q Q   Q Q   Q Q
  | | | | | | | |                \ \ \ / / / / /                 \ /   \ /   \ /   \ /
  K K K K K K K K                       K                               K     K     K     K
  V V V V V V V V                       V                               V     V     V     V
  8 Q heads, 8 K-V heads         8 Q heads, 1 K-V head           8 Q heads, 4 K-V heads (groups=4)
```

- **Multi-Head Attention (MHA):** $H$ query heads, $H$ key heads, $H$ value heads. Maximum capacity, highest memory bandwidth consumption during inference.
- **Multi-Query Attention (MQA) (Shazeer, 2019):** $H$ query heads share a single key head and a single value head. $H\times$ KV-cache reduction, but can suffer quality degradation.
- **Grouped-Query Attention (GQA) (Ainslie et al., 2023):** Divides $H$ query heads into $G$ groups. Each group shares 1 key head and 1 value head.
  - Ratio $r = H / G$ (e.g., Llama 3 70B has 64 Q heads and 8 KV heads $\to$ ratio 8:1).
  - Matches MHA quality while achieving MQA-level decoding speed and memory reduction.

---

## 3. KV CACHING & MEMORY FOOTPRINT

### 3.1 Motivation
In autoregressive generation, step $t$ computes attention between token $t$ and all past tokens $1 \dots t-1$.
Without KV Caching, generating token $T$ requires recomputing $K$ and $V$ for all preceding tokens, scaling as $O(T^2)$ compute.
With KV Caching, past keys and values are retained in GPU memory. Only the current single token's $Q_t, K_t, V_t$ are computed, and $K_t, V_t$ are appended to the cache ($O(T)$ total compute).

### 3.2 Exact KV Cache Size Equation
For a model with:
- $L$ = Number of layers
- $N_{kv}$ = Number of KV heads per layer (equals $n_{heads}$ in MHA, $G$ in GQA, $1$ in MQA)
- $d_{head} = d_{model} / n_{heads}$
- $B$ = Batch size
- $S$ = Sequence length (context length + generated tokens)
- $P$ = Precision in bytes (2 bytes for FP16/BF16, 1 byte for FP8, 0.5 bytes for INT4)

$$\text{KV Cache Size (Bytes)} = 2 \times L \times B \times S \times N_{kv} \times d_{head} \times P$$
*(The factor of 2 accounts for both Keys and Values).*

#### Example Calculation:
For Llama 2 70B (FP16, $P=2$):
- $L = 80$, $n_{heads} = 64$, $N_{kv} = 8$ (GQA), $d_{head} = 128$.
- Batch size $B = 1$, Sequence length $S = 4,096$:
$$\text{KV Cache} = 2 \times 80 \times 1 \times 4096 \times 8 \times 128 \times 2 \text{ bytes} \approx 1,342,177,280 \text{ bytes} \approx 1.34 \text{ GB}$$
If it used standard MHA ($N_{kv} = 64$):
$$\text{KV Cache (MHA)} = 8 \times 1.34 \text{ GB} = 10.74 \text{ GB per stream!}$$

---

## 4. TOKENIZATION MECHANISMS

Tokenizers convert raw text strings into discrete integers ($ID \in [0, V-1]$).

```
Raw Text: "unbelievable"
  Character-level: ['u', 'n', 'b', 'e', 'l', 'i', 'e', 'v', 'a', 'b', 'l', 'e']  (Too long, weak semantics)
  Word-level:      ["unbelievable"]  (Out-of-vocabulary nightmare for unseen words)
  Subword (BPE):   ["un", "believ", "able"]  (Optimal balance: finite vocab, handles novel words)
```

### 4.1 Comparison of Tokenizers

| Algorithm | Used By | Vocabulary Construction Mechanism | Handling of Unknowns |
| :--- | :--- | :--- | :--- |
| **Byte-Pair Encoding (BPE)** | GPT-2, RoBERTa | Starts with characters; iteratively merges the most frequent adjacent symbol pair. | Fails on unseen chars unless byte-level |
| **Byte-Level BPE (BBPE)** | GPT-4, Llama 2/3 | Base alphabet is all 256 individual raw bytes (0x00 to 0xFF). No unknown tokens possible. | Zero OOV (Every string reducible to bytes) |
| **WordPiece** | BERT | Starts with chars; merges pairs that maximize the likelihood of the language model on training data. | Subwords prefixed with `##` (e.g. `affable` $\to$ `af`, `##fable`) |
| **Unigram / SentencePiece**| T5, Llama 1, ALBERT | Starts with large candidate set; iteratively removes tokens that minimally increase training loss. | Treats whitespace as normal char (`_`) |

---

## 5. SCALING LAWS: COMPUTE-OPTIMAL TRAINING

Training compute $C$ (in FLOPs) is approximated by:
$$C \approx 6 \cdot N \cdot D$$
Where $N$ = Model parameters (non-embedding), $D$ = Total training tokens.
*(Each forward pass takes $\approx 2ND$ FLOPs; backward pass takes $\approx 4ND$ FLOPs).*

### 5.1 Kaplan et al. (OpenAI, 2020) vs Chinchilla (DeepMind, 2022)

```
                       COMPUTE ALLOCATION FOR EQUAL BUDGET
    Kaplan et al. (2020):
       Scale Parameters Faster: N ∝ C^0.73,  Tokens: D ∝ C^0.27
       Result: Led to severely undertrained models (e.g., GPT-3 175B trained on only 300B tokens).
       
    Hoffmann et al. / Chinchilla (2022):
       Scale Parameters and Tokens Equally: N ∝ C^0.50,  D ∝ C^0.50
       Result: For compute-optimal loss, tokens must scale 1:1 with parameters!
```

### 5.2 The Chinchilla Rule
$$\text{Tokens } D \approx 20 \times \text{Parameters } N$$
- A **70B parameter model** requires at least **1.4 Trillion tokens** to be compute-optimal.
- **Inference-Optimal vs Compute-Optimal:**
  - Modern open models (e.g., Llama 3 8B trained on 15T tokens $\to 1875\times$) intentionally overtrain far beyond the Chinchilla compute-optimal point.
  - Reason: Once trained, a smaller model (8B) trained on 15T tokens outperforms a 70B model trained on 1.4T tokens at drastically lower inference serving cost!

---

## 6. CODE IMPLEMENTATION: RMSNORM, SWIGLU & GQA

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class RMSNorm(nn.Module):
    def __init__(self, dim: int, eps: float = 1e-6):
        super().__init__()
        self.eps = eps
        self.weight = nn.Parameter(torch.ones(dim))

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # x: [batch, seq_len, dim]
        variance = x.pow(2).mean(-1, keepdim=True)
        return x * torch.rsqrt(variance + self.eps) * self.weight

class SwiGLU(nn.Module):
    def __init__(self, d_model: int, hidden_dim: int):
        super().__init__()
        self.w_gate = nn.Linear(d_model, hidden_dim, bias=False)
        self.w_up = nn.Linear(d_model, hidden_dim, bias=False)
        self.w_down = nn.Linear(hidden_dim, d_model, bias=False)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # F.silu(x) = x * sigmoid(x) (Swish)
        return self.w_down(F.silu(self.w_gate(x)) * self.w_up(x))

class GroupedQueryAttention(nn.Module):
    def __init__(self, d_model: int, n_heads: int, n_kv_heads: int):
        super().__init__()
        assert n_heads % n_kv_heads == 0
        self.d_model = d_model
        self.n_heads = n_heads
        self.n_kv_heads = n_kv_heads
        self.num_groups = n_heads // n_kv_heads
        self.head_dim = d_model // n_heads

        self.q_proj = nn.Linear(d_model, n_heads * self.head_dim, bias=False)
        self.k_proj = nn.Linear(d_model, n_kv_heads * self.head_dim, bias=False)
        self.v_proj = nn.Linear(d_model, n_kv_heads * self.head_dim, bias=False)
        self.out_proj = nn.Linear(d_model, d_model, bias=False)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        B, S, _ = x.shape

        q = self.q_proj(x).view(B, S, self.n_heads, self.head_dim).transpose(1, 2)
        k = self.k_proj(x).view(B, S, self.n_kv_heads, self.head_dim).transpose(1, 2)
        v = self.v_proj(x).view(B, S, self.n_kv_heads, self.head_dim).transpose(1, 2)

        # Expand K and V heads to match Q heads across groups
        k = k.repeat_interleave(self.num_groups, dim=1) # [B, n_heads, S, head_dim]
        v = v.repeat_interleave(self.num_groups, dim=1)

        # Scaled dot-product attention
        scores = torch.matmul(q, k.transpose(-2, -1)) / (self.head_dim ** 0.5)
        # Apply causal mask
        causal_mask = torch.triu(torch.full((S, S), float("-inf"), device=x.device), diagonal=1)
        scores = scores + causal_mask
        attn = F.softmax(scores, dim=-1)
        out = torch.matmul(attn, v) # [B, n_heads, S, head_dim]

        out = out.transpose(1, 2).contiguous().view(B, S, self.d_model)
        return self.out_proj(out)
```

---

## 7. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: Why is next-token prediction considered a self-supervised objective?**
   - **A:** It requires no manual human annotation. The raw text sequence itself provides both the input features ($x_1 \dots x_{t-1}$) and the ground truth supervision label ($x_t$). Any text document in existence acts as natural supervision data.

2. **Q: How does RMSNorm differ mathematically from LayerNorm, and why use it?**
   - **A:** LayerNorm subtracts the mean ($\mu$) and divides by standard deviation ($\sigma$): $\frac{x-\mu}{\sqrt{\sigma^2+\epsilon}}\gamma + \beta$. RMSNorm assumes zero-mean distribution and scales only by root mean square: $\frac{x}{\sqrt{\frac{1}{d}\sum x_i^2 + \epsilon}}\gamma$. It eliminates mean computation and additive bias $\beta$, reducing FLOPs and kernel execution overhead by ~10% without affecting model accuracy.

### Intermediate
3. **Q: Explain how Grouped-Query Attention (GQA) achieves a middle ground between MHA and MQA.**
   - **A:** MHA maintains independent Key and Value heads for every Query head ($1:1$), maximizing expressive power but causing high memory traffic during KV cache retrieval. MQA shares 1 Key and 1 Value head across all Query heads ($H:1$), reducing KV cache memory by $H\times$ but occasionally degrading nuanced multi-agent or coding performance. GQA groups Query heads into $G$ clusters ($H:G$, typically $8:1$), saving ~87.5% of KV-cache memory while retaining near-identical modeling performance to MHA.

4. **Q: What is the Chinchilla scaling law and why was GPT-3 considered sub-optimally trained?**
   - **A:** Hoffmann et al. demonstrated that for optimal compute budget allocation, parameter count ($N$) and token volume ($D$) should scale equally ($D \approx 20N$). GPT-3 had 175B parameters but was trained on only 300B tokens (a ratio of only $1.7\times$). According to Chinchilla, a 175B model should have been trained on $\approx 3.5\text{T}$ tokens. Chinchilla (70B model trained on 1.4T tokens) outperformed GPT-3 despite having less than half the parameters.

### Advanced
5. **Q: Why do production deployments overtrain models beyond Chinchilla optimality?**
   - **A:** Chinchilla optimality calculates the minimum compute to train a model for a specific target loss. However, it completely ignores downstream **inference cost**. If an organization serves billions of tokens daily, inference compute quickly dwarfs training compute. Training a compact 8B model on 15T tokens (drastically beyond Chinchilla) takes extra training compute up front, but amortizes across billions of subsequent API calls because running 8B inference is $10\times$ cheaper than running 70B inference.

---

## 8. RAPID REVISION SUMMARY

- **Autoregressive Objective:** Maximize log-likelihood $\sum \log P(x_t \mid x_{<t})$.
- **Modern LLM Backbone:** Decoder-only + Pre-RMSNorm + SwiGLU + RoPE + GQA.
- **Compute Calculation:** Training FLOPs $C \approx 6ND$.
- **Chinchilla Optimal:** $D \approx 20N$ tokens. For high-volume serving, overtraining smaller models yields superior inference economics.
- **KV Cache Memory:** $2 \times L \times B \times S \times N_{kv} \times d_{head} \times P$ bytes. GQA reduces KV cache size by factor of $H/N_{kv}$.
- **Tokenization:** Modern models use Byte-level BPE with large vocabularies (128k in Llama 3) to improve compression ratio and cross-lingual efficiency.

---

## 9. RECALL CHECKLIST

- [ ] Write the autoregressive language modeling objective mathematically.
- [ ] Write the RMSNorm formula and contrast with standard LayerNorm.
- [ ] Explain SwiGLU and how its parameter count is matched to standard FFN.
- [ ] Diagram MHA, MQA, and GQA head structures.
- [ ] Calculate the KV Cache size for a model given $L, B, S, N_{kv}, d_{head}$, and precision.
- [ ] State the Chinchilla optimal token-to-parameter ratio.
- [ ] Explain why Byte-level BPE eliminates out-of-vocabulary (<UNK>) tokens.
- [ ] Implement RMSNorm and GroupedQueryAttention in PyTorch from memory.
