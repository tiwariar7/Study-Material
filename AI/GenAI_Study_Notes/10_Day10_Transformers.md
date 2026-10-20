# Day 10: Transformer Architecture — Self-Attention, Multi-Head Attention, Encoder-Decoder

---

## Topics Covered
1. **Transformer Architecture Overview** ("Attention Is All You Need", Vaswani et al. 2017)
2. **Self-Attention Mechanism** (Query, Key, Value; Scaled Dot-Product Attention)
3. **Multi-Head Attention (MHA)** (Linear projections, parallel subspaces, aggregation)
4. **Positional Encoding** (Sinusoidal embeddings, Learned embeddings, RoPE overview)
5. **Encoder Architecture** (Self-Attention, FFN, Pre-LN vs Post-LN, Residual Connections)
6. **Decoder Architecture** (Masked Causal Attention, Cross-Attention / Encoder-Decoder Attention)
7. **Architectural Variants** (Encoder-only: BERT; Decoder-only: GPT; Encoder-Decoder: T5)
8. **From-Scratch PyTorch Implementation**
9. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. TRANSFORMER ARCHITECTURE OVERVIEW

### 1.1 Definition & Intuition
The **Transformer** is a neural network architecture introduced in 2017 by Vaswani et al. that replaces recurrent and convolutional operations entirely with **attention mechanisms**.

- **The Core Problem with RNNs:** RNNs process tokens sequentially ($O(N)$ sequential operations for a sequence of length $N$). This prevents massive GPU parallelization and causes information degradation over long distances (the vanishing gradient / bottleneck problem).
- **The Transformer Solution:** Processes all tokens simultaneously ($O(1)$ sequential operations). Pairwise interactions between every token in the sequence are computed directly, allowing immediate access to distant context regardless of position.

```
       TRANSFORMER ARCHITECTURE (Vaswani et al., 2017)

               Output Probabilities
                       ↑
                    Softmax
                       ↑
                  Linear Layer
                       ↑
             +--------------------+
             |  Decoder Block ×N  | <-------+
             +--------------------+          | Cross-Attention
                       ↑                     |
    +--------------------+                   |
    |  Encoder Block ×N  | ------------------+
    +--------------------+
               ↑                               ↑
     Positional Encoding             Positional Encoding
               ↑                               ↑
        Input Embeddings                Output Embeddings
               ↑                               ↑
          Input Tokens                   Target Tokens (shifted right)
```

### 1.2 Complexity Comparison Across Architectures

| Layer Type | Complexity per Layer | Sequential Operations | Maximum Path Length |
| :--- | :--- | :--- | :--- |
| **Self-Attention** | $O(N^2 \cdot d)$ | $O(1)$ | $O(1)$ |
| **Recurrent (RNN)** | $O(N \cdot d^2)$ | $O(N)$ | $O(N)$ |
| **Convolutional** | $O(k \cdot N \cdot d^2)$ | $O(1)$ | $O(\log_k(N))$ |

Where:
- $N$ = Sequence length
- $d$ = Representation / embedding dimension ($d_{model}$)
- $k$ = Convolutional kernel size

---

## 2. SELF-ATTENTION MECHANISM

### 2.1 The Concept of Query, Key, and Value
The attention mechanism is modeled after information retrieval systems:
- **Query ($Q$):** What the current token is looking for.
- **Key ($K$):** What each token advertises/contains.
- **Value ($V$):** The actual content or information payload delivered if there is a match between Query and Key.

For an input matrix of token representations $X \in \mathbb{R}^{N \times d_{model}}$:
$$Q = X W^Q, \quad K = X W^K, \quad V = X W^V$$
Where projection matrices are $W^Q, W^K \in \mathbb{R}^{d_{model} \times d_k}$ and $W^V \in \mathbb{R}^{d_{model} \times d_v}$.

### 2.2 Scaled Dot-Product Attention Equation
$$\text{Attention}(Q, K, V) = \text{Softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right) V$$

```
   Q Matrix (N x d_k)   x   K^T Matrix (d_k x N)
                    ↓
         Raw Attention Scores (N x N)
                    ↓
              Divide by √d_k
                    ↓
             Apply Mask (if causal)
                    ↓
         Softmax (row-wise sum = 1)
                    ↓
         Attention Weights A (N x N)   x   V Matrix (N x d_v)
                    ↓
         Contextualized Outputs (N x d_v)
```

### 2.3 Why Scale by $\sqrt{d_k}$?
**Mathematical Proof / Derivation:**
Assume the elements of $Q$ and $K$ are independent random variables with zero mean and unit variance:
$$\mathbb{E}[q_i] = 0, \quad \text{Var}(q_i) = 1, \quad \mathbb{E}[k_i] = 0, \quad \text{Var}(k_i) = 1$$
The dot product between two single vectors $q, k \in \mathbb{R}^{d_k}$ is:
$$z = q \cdot k = \sum_{i=1}^{d_k} q_i k_i$$
The expectation of the dot product:
$$\mathbb{E}[z] = \sum_{i=1}^{d_k} \mathbb{E}[q_i] \mathbb{E}[k_i] = 0$$
The variance of the dot product:
$$\text{Var}(z) = \sum_{i=1}^{d_k} \text{Var}(q_i k_i) = \sum_{i=1}^{d_k} \left(\text{Var}(q_i)\text{Var}(k_i) + \text{Var}(q_i)\mathbb{E}[k_i]^2 + \text{Var}(k_i)\mathbb{E}[q_i]^2\right) = \sum_{i=1}^{d_k} (1 \cdot 1) = d_k$$
Standard deviation is $\sqrt{d_k}$.
- If $d_k$ is large (e.g., $d_k = 64$ or $128$), the dot products grow large in magnitude.
- The softmax function: $\text{softmax}(z)_i = \frac{e^{z_i}}{\sum_j e^{z_j}}$. When inputs have high variance, softmax pushes towards one-hot vectors ($e^{z_{max}} \gg \sum e^{z_j}$).
- Softmax gradient in saturation regions approaches zero ($\frac{\partial s_i}{\partial z_j} = s_i(\delta_{ij} - s_j) \approx 0$).
- Dividing by $\sqrt{d_k}$ normalizes the variance back to $1.0$, preventing vanishing gradients during backpropagation.

---

## 3. MULTI-HEAD ATTENTION (MHA)

### 3.1 Intuition
A single self-attention head averages information across all tokens, which can dilute distinct linguistic relationships. Multi-Head Attention enables the model to jointly attend to information from different representation subspaces at different positions:
- Head 1 might focus on subject-verb agreements.
- Head 2 might focus on direct object relations.
- Head 3 might focus on prepositional phrases.
- Head 4 might focus on pronoun coreference resolution.

### 3.2 Formal Equation
$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \dots, \text{head}_h) W^O$$
$$\text{where } \text{head}_i = \text{Attention}(Q W_i^Q, K W_i^K, V W_i^V)$$

Dimensions:
- $h$: Number of heads (typically 8 to 96)
- $d_{model}$: Hidden dimension (e.g., 512, 768, 4096)
- $d_k = d_v = d_{model} / h$ (e.g., $512 / 8 = 64$)
- $W_i^Q \in \mathbb{R}^{d_{model} \times d_k}$
- $W_i^K \in \mathbb{R}^{d_{model} \times d_k}$
- $W_i^V \in \mathbb{R}^{d_{model} \times d_v}$
- $W^O \in \mathbb{R}^{(h \cdot d_v) \times d_{model}} = \mathbb{R}^{d_{model} \times d_{model}}$

---

## 4. POSITIONAL ENCODINGS

Self-attention is **permutation-invariant**: $\text{Attention}(P X, P X, P X) = P \cdot \text{Attention}(X, X, X)$ for any permutation matrix $P$. Without position information, the sentence *"Dog bites man"* is mathematically indistinguishable from *"Man bites dog"*.

### 4.1 Sinusoidal Positional Encoding (Vaswani et al., 2017)
Added directly to input token embeddings: $X_{input} = \text{Embedding}(X) + PE$.

$$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{\frac{2i}{d_{model}}}}\right)$$
$$PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{\frac{2i}{d_{model}}}}\right)$$

Where:
- $pos \in [0, N-1]$ is the token position in the sequence.
- $i \in [0, \frac{d_{model}}{2}-1]$ is the dimension index.
- Wavelengths form a geometric progression from $2\pi$ to $10000 \cdot 2\pi$.

**Why Sinusoids?**
By trigonometric identity, for any fixed offset $k$, $PE_{pos+k}$ can be represented as a linear function of $PE_{pos}$:
$$\begin{pmatrix} \sin(\omega (pos+k)) \\ \cos(\omega (pos+k)) \end{pmatrix} = \begin{pmatrix} \cos(\omega k) & \sin(\omega k) \\ -\sin(\omega k) & \cos(\omega k) \end{pmatrix} \begin{pmatrix} \sin(\omega pos) \\ \cos(\omega pos) \end{pmatrix}$$
This allows the model to easily learn relative position comparisons.

### 4.2 Learned Absolute Positional Embeddings (BERT, GPT-2)
- A lookup table $E_{pos} \in \mathbb{R}^{N_{max} \times d_{model}}$.
- Directly optimized with backpropagation.
- Limitation: Hard cap at sequence length $N_{max}$. Cannot generalize to lengths unseen during training.

### 4.3 Rotary Positional Embedding (RoPE - Su et al., 2021)
Used in modern LLMs (Llama 1/2/3, Mistral, Qwen):
- Rotates the Query and Key vectors in the 2D complex plane by an angle proportional to their position:
$$\tilde{q}_m = R_{\Theta, m}^d q_m, \quad \tilde{k}_n = R_{\Theta, n}^d k_n$$
- The inner product satisfies: $\langle \tilde{q}_m, \tilde{k}_n \rangle = g(q, k, m-n)$. The attention score depends purely on relative distance $(m - n)$ rather than absolute positions $m$ and $n$.

---

## 5. ENCODER ARCHITECTURE

An Encoder block consists of two primary sub-layers:
1. **Multi-Head Self-Attention**
2. **Position-wise Feed-Forward Network (FFN)**

Both sub-layers are wrapped with **Residual Connections** and **Layer Normalization**.

```
                Encoder Layer
                      ↑
             Add & LayerNorm
                      ↑
         Feed-Forward Network (FFN)
                      ↑
             Add & LayerNorm
                      ↑
         Multi-Head Self-Attention
                      ↑
                    Input
```

### 5.1 Feed-Forward Network (FFN)
Applied to each position separately and identically:
$$\text{FFN}(x) = \max(0, x W_1 + b_1) W_2 + b_2$$
- Inner dimension $d_{ff} = 4 \times d_{model}$ (e.g., $512 \to 2048 \to 512$).
- Modern LLMs replace ReLU with **SwiGLU**: $\text{SwiGLU}(x) = (\text{Swish}(x W_{gate}) \odot x W_{up}) W_{down}$.

### 5.2 LayerNorm: Pre-LN vs Post-LN

| Aspect | Post-LN (Original Transformer) | Pre-LN (Modern Standard: GPT-2, Llama) |
| :--- | :--- | :--- |
| **Formula** | $x = \text{LayerNorm}(x + \text{Sublayer}(x))$ | $x = x + \text{Sublayer}(\text{LayerNorm}(x))$ |
| **Stability** | Gradients vanish/explode in deep nets without warm-up | Extremely stable gradients through residual stream |
| **Warm-up** | Requires strict learning rate warm-up schedule | Can train directly without sensitive warm-up |
| **Final Layer** | Not needed | Requires a final LayerNorm before prediction head |

---

## 6. DECODER ARCHITECTURE

The Decoder generates tokens autoregressively (one at a time from left to right). It contains three sub-layers:
1. **Masked Multi-Head Self-Attention (Causal Attention)**
2. **Encoder-Decoder Cross-Attention**
3. **Position-wise Feed-Forward Network (FFN)**

### 6.1 Causal Masking (Look-Ahead Mask)
To prevent position $i$ from attending to subsequent positions $j > i$:
$$M_{ij} = \begin{cases} 0 & \text{if } j \le i \\ -\infty & \text{if } j > i \end{cases}$$
$$\text{Attention}(Q, K, V) = \text{Softmax}\left(\frac{Q K^T}{\sqrt{d_k}} + M\right) V$$
Since $e^{-\infty} = 0$, tokens cannot gather information from the future.

### 6.2 Cross-Attention
- **Query ($Q$):** Projected from the previous decoder layer.
- **Key ($K$) & Value ($V$):** Projected from the final output representations of the **Encoder**.
- Decouples decoding state from input context, allowing generated tokens to query source document tokens.

---

## 7. TAXONOMY OF TRANSFORMER ARCHITECTURES

```
                           TRANSFORMER FAMILY
                                   |
         +-------------------------+-------------------------+
         |                         |                         |
   Encoder-Only              Decoder-Only             Encoder-Decoder
   (Autoencoding)          (Autoregressive)           (Sequence-to-Sequence)
         |                         |                         |
    BERT, RoBERTa             GPT-1/2/3/4                Original 2017
       DeBERTa               Llama, Mistral                 T5, BART
         |                         |                         |
 Best for: Classification,   Best for: Open-ended     Best for: Translation,
 NER, Embedding Extraction    Generation, Chat, Code    Summarization
```

| Type | Attention Type | Training Objective | Strengths |
| :--- | :--- | :--- | :--- |
| **Encoder-Only** | Bidirectional | Masked Language Modeling (MLM), Next Sentence Prediction | Deep contextual representations for understanding tasks |
| **Decoder-Only** | Causal (Unidirectional) | Next-Token Prediction (Causal LM: $\max \sum \log P(x_t \mid x_{<t})$) | Generative tasks, in-context learning, scaling efficiency |
| **Encoder-Decoder** | Bidirectional (Enc) + Causal (Dec) + Cross | Masked span corruption (T5), Seq2Seq translation | Cross-lingual translation, structured text transformation |

---

## 8. COMPLETE PYTORCH IMPLEMENTATION FROM SCRATCH

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class ScaledDotProductAttention(nn.Module):
    def __init__(self, d_k: int):
        super().__init__()
        self.scale = 1.0 / math.sqrt(d_k)

    def forward(self, q, k, v, mask=None):
        # q, k, v shapes: [batch_size, n_heads, seq_len, d_k]
        scores = torch.matmul(q, k.transpose(-2, -1)) * self.scale
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)
        attn_weights = F.softmax(scores, dim=-1)
        output = torch.matmul(attn_weights, v)
        return output, attn_weights

class MultiHeadAttention(nn.Module):
    def __init__(self, d_model: int, n_heads: int):
        super().__init__()
        assert d_model % n_heads == 0, "d_model must be divisible by n_heads"
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_k = d_model // n_heads

        self.w_q = nn.Linear(d_model, d_model)
        self.w_k = nn.Linear(d_model, d_model)
        self.w_v = nn.Linear(d_model, d_model)
        self.w_o = nn.Linear(d_model, d_model)

        self.attention = ScaledDotProductAttention(self.d_k)

    def forward(self, q, k, v, mask=None):
        batch_size = q.size(0)

        # 1. Linear projections and reshape to [B, heads, seq_len, d_k]
        q = self.w_q(q).view(batch_size, -1, self.n_heads, self.d_k).transpose(1, 2)
        k = self.w_k(k).view(batch_size, -1, self.n_heads, self.d_k).transpose(1, 2)
        v = self.w_v(v).view(batch_size, -1, self.n_heads, self.d_k).transpose(1, 2)

        # 2. Scaled Dot-Product Attention
        out, attn_weights = self.attention(q, k, v, mask=mask)

        # 3. Concatenate heads and project out
        out = out.transpose(1, 2).contiguous().view(batch_size, -1, self.d_model)
        return self.w_o(out)

class TransformerEncoderBlock(nn.Module):
    """Pre-LN Transformer Encoder Block"""
    def __init__(self, d_model: int, n_heads: int, d_ff: int, dropout: float = 0.1):
        super().__init__()
        self.ln1 = nn.LayerNorm(d_model)
        self.mha = MultiHeadAttention(d_model, n_heads)
        self.dropout1 = nn.Dropout(dropout)

        self.ln2 = nn.LayerNorm(d_model)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.GELU(),
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),
            nn.Dropout(dropout)
        )

    def forward(self, x, mask=None):
        # Pre-LN Residual Connection 1
        x = x + self.dropout1(self.mha(self.ln1(x), self.ln1(x), self.ln1(x), mask))
        # Pre-LN Residual Connection 2
        x = x + self.ffn(self.ln2(x))
        return x
```

---

## 9. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: Why does self-attention have $O(N^2)$ computational complexity?**
   - **A:** For a sequence of length $N$, every single token calculates a dot product with every other token in the sequence to compute pairwise compatibility scores ($Q \cdot K^T$). This results in an $N \times N$ matrix multiplication, requiring $O(N^2 \cdot d)$ operations and $O(N^2)$ memory for the attention matrix.

2. **Q: What is the purpose of Positional Encodings?**
   - **A:** The dot-product self-attention operation is order-invariant (permutation-symmetric). Without positional encodings, the representations for shuffled sentences are identical. Adding positional encodings injects information about the absolute or relative sequence positions of tokens.

### Intermediate
3. **Q: What is the exact difference between Self-Attention and Cross-Attention?**
   - **A:** In self-attention, $Q$, $K$, and $V$ all originate from the same sequence representation (e.g., encoder input or decoder input). In cross-attention, $Q$ is derived from the decoder's current representation, while $K$ and $V$ come from the encoder's output representations.

4. **Q: Why does Pre-LN train more stably than Post-LN?**
   - **A:** In Post-LN, gradients passing through the residual connection must pass through LayerNorm at every block. As depth increases, variance and gradient norms at early layers can degrade, requiring strict warmup schedules. In Pre-LN, the residual identity path $x_{l+1} = x_l + F(LN(x_l))$ has an unobstructed path directly from final layer to input layer, ensuring clean gradient propagation without vanishing.

### Advanced / Staff Level
5. **Q: Why is Decoder-only preferred over Encoder-Decoder for modern Foundation LLMs?**
   - **A:**
     1. **Compute Efficiency:** Decoder-only uses causal language modeling where every token prediction contributes to training loss. In Seq2Seq, encoder token positions do not generate loss directly.
     2. **KV-Cache Optimization:** Autoregressive generation needs only to append a new Key and Value vector per step, keeping generation memory and computation minimal.
     3. **Sample Efficiency & In-Context Learning:** Unifying prompt prefix and generation into a single continuous causal stream simplifies multi-task prompting and few-shot conditioning without requiring separate encoder-decoder parameter synchronization.

6. **Q: How does FlashAttention optimize the $O(N^2)$ memory wall without approximating attention?**
   - **A:** Standard attention materializes the intermediate $N \times N$ attention matrix in slow GPU High Bandwidth Memory (HBM). FlashAttention uses **tiling** to compute softmax incrementally (using the online softmax trick) in fast GPU on-chip SRAM. It computes attention block by block and recomputes intermediate attention scores in backward pass, reducing memory transfers from HBM by $4\times$ to $8\times$ with zero loss in mathematical precision.

---

## 10. RAPID REVISION SUMMARY

- **Core Formula:** $\text{Attention}(Q,K,V) = \text{Softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$.
- **Scale Factor:** Normalizes dot-product variance from $d_k$ back to 1 to avoid softmax gradient saturation.
- **Multi-Head:** Splits $d_{model}$ into $h$ heads of dimension $d_k = d_{model}/h$. Allows distinct subspace modeling.
- **Positional Encoding:** Injects order into permutation-invariant attention (Sinusoidal, Learned, RoPE).
- **Encoder vs Decoder:** Encoder uses bidirectional attention; Decoder uses causal masking ($-\infty$ upper triangle) + cross-attention.
- **Architectures:** BERT (Encoder-only, understanding), GPT (Decoder-only, generation), T5 (Encoder-Decoder, translation/transform).

---

## 11. RECALL CHECKLIST

- [ ] Write the Scaled Dot-Product Attention formula from memory.
- [ ] Prove why the dot product variance equals $d_k$ for standard normal inputs.
- [ ] Explain why $\sqrt{d_k}$ scaling prevents vanishing gradients in softmax.
- [ ] Diagram Multi-Head Attention splitting, attention calculation, and output projection.
- [ ] State the difference between Pre-LN and Post-LN and why modern models use Pre-LN.
- [ ] Write down the causal mask matrix structure for a 4-token sequence.
- [ ] Compare Encoder-only, Decoder-only, and Encoder-Decoder architectures.
- [ ] Explain RoPE (Rotary Position Embedding) and why relative positional encoding is beneficial.
