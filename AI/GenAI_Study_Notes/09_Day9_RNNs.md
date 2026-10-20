# Day 9: RNNs — Architecture, LSTM & GRU Basics, Sequence Modeling

---

## Topics Covered
1. **Recurrent Neural Networks (RNNs)** — Architecture, Working, Limitations
2. **LSTM (Long Short-Term Memory)** — Gates, Cell State, Mechanism
3. **GRU (Gated Recurrent Unit)** — Simplified LSTM
4. **Sequence Modeling Applications**

---

## 1. RECURRENT NEURAL NETWORKS (RNNs)

### 1.1 Definition & Fundamentals

**Definition:** An RNN is a neural network designed for **sequential data** where the output at each step depends on the current input AND the previous hidden state. It has a **recurrent connection** that carries information across time steps.

**Plain English:** An RNN processes data one step at a time (like reading word by word), maintaining a "memory" (hidden state) that accumulates information from all previous steps.

**Why RNNs?** Standard MLPs can't handle variable-length sequences or temporal dependencies. A sentence has 5 words or 50 — RNNs process any length. Each output depends on the entire history, not just the current input.

### 1.2 Architecture

```
Unrolled RNN through time:

x₁ → [RNN Cell] → h₁ → [RNN Cell] → h₂ → [RNN Cell] → h₃
       ↑    ↓              ↑    ↓              ↑    ↓
       h₀   y₁             h₁   y₂             h₂   y₃

Same weights (Wₓₕ, Wₕₕ, Wₕᵧ) shared across all time steps!
```

### 1.3 Mathematical Foundation

**Hidden state update:**
```
hₜ = tanh(Wₓₕ · xₜ + Wₕₕ · hₜ₋₁ + bₕ)
```

**Output:**
```
yₜ = Wₕᵧ · hₜ + bᵧ
```

Where:
- **xₜ** = input at time t (e.g., word embedding)
- **hₜ** = hidden state at time t (the "memory")
- **h₀** = initial hidden state (usually zeros)
- **Wₓₕ** = input-to-hidden weights (shared across time)
- **Wₕₕ** = hidden-to-hidden weights (recurrent weights)
- **Wₕᵧ** = hidden-to-output weights
- **tanh** = activation function (squashes to [-1, 1])

**Weight sharing:** The same weights are used at every time step — this is what makes RNNs handle variable-length sequences.

### 1.4 Types of RNN Architectures

| Type | Input → Output | Example |
|------|---------------|---------|
| **One-to-One** | Single → Single | Standard feedforward (not really RNN) |
| **One-to-Many** | Single → Sequence | Image captioning |
| **Many-to-One** | Sequence → Single | Sentiment analysis, classification |
| **Many-to-Many** (same length) | Sequence → Sequence | POS tagging |
| **Many-to-Many** (diff length) | Sequence → Sequence | Machine translation (Encoder-Decoder) |

### 1.5 Backpropagation Through Time (BPTT)

**Definition:** BPTT unrolls the RNN across all time steps and applies standard backpropagation.

```
∂L/∂W = Σₜ ∂Lₜ/∂W

For ∂Lₜ/∂Wₕₕ at time t:
∂Lₜ/∂Wₕₕ = Σₖ₌₁ᵗ (∂Lₜ/∂hₜ) × (∏ⱼ₌ₖ₊₁ᵗ ∂hⱼ/∂hⱼ₋₁) × (∂hₖ/∂Wₕₕ)
```

The product ∏(∂hⱼ/∂hⱼ₋₁) involves **multiplying many terms** (one per time step). This leads to:

### 1.6 Vanishing/Exploding Gradients in RNNs

**Each term ∂hⱼ/∂hⱼ₋₁ involves multiplying by Wₕₕ and tanh derivative (≤1):**

- If ||Wₕₕ|| < 1 (or tanh derivative < 1): product → 0. **Vanishing gradient.** Can't learn long-range dependencies.
- If ||Wₕₕ|| > 1: product → ∞. **Exploding gradient.**

**Exploding gradient fix:** Gradient clipping — cap ||gradient|| at a threshold.

**Vanishing gradient fix:** **LSTM and GRU** — specifically designed to maintain gradients over long sequences.

**Practical limitation:** Standard RNNs can only capture dependencies of ~10-20 time steps. Beyond that, earlier inputs have negligible influence.

### 1.7 Simple RNN Implementation

```python
import torch
import torch.nn as nn

class SimpleRNN(nn.Module):
    def __init__(self, input_size, hidden_size, output_size, num_layers=1):
        super().__init__()
        self.rnn = nn.RNN(
            input_size=input_size,     # Feature dimension of input
            hidden_size=hidden_size,    # Size of hidden state
            num_layers=num_layers,      # Stacked RNN layers
            batch_first=True,           # Input: (batch, seq_len, features)
            bidirectional=False
        )
        self.fc = nn.Linear(hidden_size, output_size)
    
    def forward(self, x):
        # x: (batch, seq_len, input_size)
        out, h_n = self.rnn(x)
        # out: (batch, seq_len, hidden_size) — all hidden states
        # h_n: (num_layers, batch, hidden_size) — last hidden state
        
        # For many-to-one: use last hidden state
        out = self.fc(h_n[-1])  # (batch, output_size)
        return out
```

---

## 2. LSTM (LONG SHORT-TERM MEMORY)

### 2.1 Definition & Fundamentals

**Definition:** LSTM is a specialized RNN architecture that uses **gates** (forget, input, output) and a **cell state** to selectively remember or forget information over long sequences, solving the vanishing gradient problem.

**Key Innovation:** The **cell state (Cₜ)** — a highway that runs through the entire sequence with only minor linear interactions. Information can flow unchanged through many time steps, preserving long-term dependencies.

### 2.2 Architecture — The Three Gates + Cell State

```
         ┌─────────── Cell State (Cₜ₋₁ → Cₜ) ──────────────┐
         │              ↑ forget        ↑ input               │
         │              ×               +                      │
         │           ┌──┘           ┌───┘                      │
         │     ┌─────┴─────┐ ┌─────┴─────┐  ┌─────────────┐  │
hₜ₋₁ ───┤─→ │ Forget Gate │ │ Input Gate │  │ Output Gate │  │
         │   │   fₜ = σ()   │ │  iₜ = σ()  │  │  oₜ = σ()   │  │
xₜ ──────┤   └─────────────┘ └───────────┘  └──────┬──────┘  │
         │         ↑                ↑               │         │
         │    [hₜ₋₁, xₜ]    ┌──────────┐           │         │
         │                   │Candidate │     ↓         │
         │                   │C̃ₜ=tanh() │   tanh(Cₜ)×oₜ = hₜ
         │                   └──────────┘                     │
         └─────────────────────────────────────────────────────┘
```

### 2.3 Mathematical Equations

**Step 1: Forget Gate** — What to remove from cell state
```
fₜ = σ(Wf · [hₜ₋₁, xₜ] + bf)
```
Output: values between 0 (completely forget) and 1 (completely keep).

**Step 2: Input Gate** — What new information to store
```
iₜ = σ(Wi · [hₜ₋₁, xₜ] + bi)          # How much to update
C̃ₜ = tanh(Wc · [hₜ₋₁, xₜ] + bc)       # Candidate values
```

**Step 3: Update Cell State**
```
Cₜ = fₜ ⊙ Cₜ₋₁ + iₜ ⊙ C̃ₜ
```
- **fₜ ⊙ Cₜ₋₁:** Keep (or forget) old cell state.
- **iₜ ⊙ C̃ₜ:** Add new relevant information.
- ⊙ = element-wise multiplication.

**Step 4: Output Gate** — What to output from cell state
```
oₜ = σ(Wo · [hₜ₋₁, xₜ] + bo)
hₜ = oₜ ⊙ tanh(Cₜ)
```

### 2.4 Why LSTM Solves Vanishing Gradients

**Key insight:** Gradient flows through the cell state Cₜ with only **additive** and **element-wise multiplicative** operations (no matrix multiplication by Wₕₕ like in vanilla RNN):

```
∂Cₜ/∂Cₜ₋₁ = fₜ    (forget gate value, between 0 and 1)
```

When fₜ ≈ 1 (keep everything), gradient flows unimpeded. The network learns what to forget (fₜ ≈ 0) and what to keep (fₜ ≈ 1), maintaining useful gradients over long sequences.

### 2.5 LSTM Parameter Count

For input_size=d, hidden_size=h:
```
Each gate has: W[h×(h+d)] + b[h]  parameters
4 gates (f, i, C̃, o): 4 × (h(h+d) + h) = 4h(h+d) + 4h = 4h² + 4hd + 4h
```
**~4x more parameters than vanilla RNN** — but much more capable.

---

## 3. GRU (GATED RECURRENT UNIT)

### 3.1 Definition

**Definition:** GRU is a simplified version of LSTM with only **two gates** (reset, update) and **no separate cell state**. It merges the cell state into the hidden state.

### 3.2 Mathematical Equations

**Reset Gate** — How much past information to forget:
```
rₜ = σ(Wr · [hₜ₋₁, xₜ] + br)
```

**Update Gate** — How much to update hidden state:
```
zₜ = σ(Wz · [hₜ₋₁, xₜ] + bz)
```

**Candidate Hidden State:**
```
h̃ₜ = tanh(Wh · [rₜ ⊙ hₜ₋₁, xₜ] + bh)
```

**Final Hidden State:**
```
hₜ = (1 - zₜ) ⊙ hₜ₋₁ + zₜ ⊙ h̃ₜ
```

**Intuition:** zₜ acts as both forget and input gate. When zₜ ≈ 0: keep old state (memory). When zₜ ≈ 1: use new candidate (update).

### 3.3 LSTM vs GRU

| Aspect | LSTM | GRU |
|--------|------|-----|
| Gates | 3 (forget, input, output) | 2 (reset, update) |
| States | Hidden state + Cell state | Hidden state only |
| Parameters | More (~4h(h+d)) | Fewer (~3h(h+d)) |
| Performance | Better on complex/long sequences | Often comparable |
| Training speed | Slower | Faster |
| Memory | More | Less |
| Interpretability | Gates more explicit | More coupled |

**When to use which:** Start with GRU (faster, fewer parameters). Switch to LSTM if GRU underperforms on long sequences. In practice, performance difference is often small.

### 3.4 Implementation

```python
import torch
import torch.nn as nn

class LSTMModel(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size, output_size, num_layers=2):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(
            input_size=embed_dim,
            hidden_size=hidden_size,
            num_layers=num_layers,
            batch_first=True,
            bidirectional=True,     # Process forward and backward
            dropout=0.3             # Between LSTM layers (not last)
        )
        # Bidirectional → hidden_size * 2
        self.fc = nn.Linear(hidden_size * 2, output_size)
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x):
        # x: (batch, seq_len) — token indices
        embedded = self.dropout(self.embedding(x))  # (batch, seq_len, embed_dim)
        
        # LSTM output
        lstm_out, (h_n, c_n) = self.lstm(embedded)
        # lstm_out: (batch, seq_len, hidden*2)  — all time steps
        # h_n: (num_layers*2, batch, hidden)    — last hidden state
        # c_n: (num_layers*2, batch, hidden)    — last cell state
        
        # Concatenate forward and backward final hidden states
        hidden = torch.cat((h_n[-2], h_n[-1]), dim=1)  # (batch, hidden*2)
        output = self.fc(self.dropout(hidden))
        return output

class GRUModel(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_size, output_size):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.gru = nn.GRU(embed_dim, hidden_size, num_layers=2,
                          batch_first=True, bidirectional=True, dropout=0.3)
        self.fc = nn.Linear(hidden_size * 2, output_size)
    
    def forward(self, x):
        embedded = self.embedding(x)
        gru_out, h_n = self.gru(embedded)
        # GRU has no cell state (c_n)
        hidden = torch.cat((h_n[-2], h_n[-1]), dim=1)
        return self.fc(hidden)
```

---

## 4. SEQUENCE MODELING APPLICATIONS

### 4.1 Common Tasks

| Task | Architecture | Input → Output |
|------|-------------|---------------|
| Sentiment Analysis | Many-to-One | Sentence → Positive/Negative |
| Named Entity Recognition | Many-to-Many | Tokens → Entity labels |
| Machine Translation | Encoder-Decoder | Source sentence → Target sentence |
| Text Generation | Many-to-Many | Prompt → Next tokens |
| Time Series Forecasting | Many-to-One/Many | Past values → Future values |
| Speech Recognition | Many-to-Many | Audio frames → Text |

### 4.2 Bidirectional RNN/LSTM

- Process sequence in **both directions** simultaneously.
- Forward pass: read left → right. Backward pass: read right → left.
- Final hidden state = concatenation of both directions.
- **Advantage:** Each output considers BOTH past and future context.
- **Limitation:** Can't be used for autoregressive generation (can't look into the future during generation).

### 4.3 Sequence-to-Sequence (Seq2Seq) with Attention

**Encoder-Decoder Architecture:**
1. **Encoder** (LSTM/GRU): Reads entire input sequence, produces context vector.
2. **Decoder** (LSTM/GRU): Generates output sequence one token at a time.

**Problem:** Single context vector is a bottleneck for long sequences.

**Solution — Attention:** At each decoder step, attend to all encoder hidden states and create a weighted context vector.

```
Attention weights: αᵢⱼ = softmax(score(hᵢ_dec, hⱼ_enc))
Context: cᵢ = Σⱼ αᵢⱼ · hⱼ_enc
```
This was the precursor to the Transformer's self-attention (Day 10).

### 4.4 RNN vs Transformer (Preview)

| Aspect | RNN/LSTM | Transformer |
|--------|----------|-------------|
| Processing | Sequential (one step at a time) | Parallel (all positions at once) |
| Long-range dependencies | LSTM better than RNN, but still limited | Excellent (self-attention) |
| Training speed | Slow (sequential) | Fast (parallelizable) |
| Memory | O(sequence_length) | O(sequence_length²) for attention |
| Current status | Legacy (still used in some cases) | **Dominant** for NLP and beyond |

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Using vanilla RNN for long sequences** — vanishing gradients. Use LSTM or GRU.
2. **Forgetting `batch_first=True`** in PyTorch — default is (seq_len, batch, features), which is confusing.
3. **Not applying gradient clipping** for RNN training — exploding gradients are common.
4. **Ignoring bidirectional option** when context from both directions is available.
5. **Using RNN for tasks where a Transformer would be much better** — modern NLP should prefer Transformers.

---

## Interview Preparation

### Direct Interview Answers

**Q: What is the vanishing gradient problem in RNNs and how does LSTM solve it?**
> "In vanilla RNNs, gradients are multiplied by the weight matrix Wₕₕ and tanh derivative at each time step during backpropagation. Over many steps, these products either shrink to zero (vanishing) or explode. LSTM solves this with a cell state that flows through the network with additive updates — gradients flow through the cell state without repeated matrix multiplication. The forget gate controls how much gradient passes through, learning to keep gradients alive for relevant long-range dependencies."

**Q: Explain the three gates of an LSTM.**
> "Forget gate (σ): decides what to remove from the cell state — how much of the previous memory to keep. Input gate (σ + tanh): decides what new information to add to the cell state — the gate controls how much, and the tanh creates candidate values. Output gate (σ): decides what part of the cell state to output as the hidden state — filters the cell state before output."

### Interview Q&A Bank

**Basic:**
1. **Q:** What is an RNN? **A:** A neural network with recurrent connections that maintain a hidden state across time steps, allowing it to process sequential data. The same weights are shared across all time steps.

2. **Q:** What is the difference between LSTM and GRU? **A:** LSTM has 3 gates and a separate cell state; GRU has 2 gates (reset, update) and merges cell state into hidden state. GRU is simpler and faster; LSTM can be better for long sequences.

**Intermediate:**
3. **Q:** What is BPTT? **A:** Backpropagation Through Time. The RNN is unrolled across all time steps, and standard backpropagation is applied. The loss gradient at each time step flows backward through all previous time steps, which is why vanishing/exploding gradients occur.

4. **Q:** What is bidirectional LSTM and when would you use it? **A:** Processes the sequence in both forward and backward directions, producing hidden states that capture context from the entire sequence. Use for tasks like NER, sentiment analysis, or any task where the full context is available. Don't use for autoregressive generation.

**Advanced:**
5. **Q:** How does the LSTM cell state enable long-range gradient flow? **A:** The cell state update is Cₜ = fₜ⊙Cₜ₋₁ + iₜ⊙C̃ₜ. The gradient ∂Cₜ/∂Cₜ₋₁ = fₜ (element-wise, values between 0-1). Unlike vanilla RNN where gradients multiply by the full weight matrix, LSTM's gradient only multiplies by the forget gate value. When fₜ≈1, gradient passes through unchanged, enabling learning over hundreds of time steps.

6. **Q:** Why have RNNs/LSTMs been largely replaced by Transformers? **A:** (1) RNNs process sequentially — can't parallelize within a sequence, making training slow. (2) Despite LSTM improvements, long-range dependencies still degrade with distance. Transformers compute all-pairs attention in parallel, directly connecting any two positions. (3) Transformers scale better with data and compute, enabling the large-scale training that powers modern LLMs.

---

## Rapid Revision Summary

- **RNN:** hₜ = tanh(Wₓₕxₜ + Wₕₕhₜ₋₁ + b). Sequential processing. Shared weights across time. Vanishing gradient for long sequences.
- **LSTM:** 3 gates (forget, input, output) + cell state. fₜ=σ(), iₜ=σ(), C̃ₜ=tanh(), Cₜ=fₜ⊙Cₜ₋₁+iₜ⊙C̃ₜ, oₜ=σ(), hₜ=oₜ⊙tanh(Cₜ). Cell state enables long-range gradient flow.
- **GRU:** 2 gates (reset, update). Simpler, fewer params. hₜ=(1-zₜ)⊙hₜ₋₁+zₜ⊙h̃ₜ.
- **Bidirectional:** Process both directions → full context. Can't use for generation.
- **Seq2Seq + Attention:** Encoder reads input → decoder generates output, attending to all encoder states.
- **RNN → Transformer:** Sequential → parallel. Limited context → unlimited context.

---

## Self-Assessment

1. Write the hidden state update equation for a vanilla RNN.
2. Explain each LSTM gate with its equation and purpose.
3. Why does vanilla RNN suffer from vanishing gradients but LSTM doesn't?
4. How many parameters does an LSTM layer have with input_size=100 and hidden_size=256?
5. What is the key difference between LSTM and GRU architecturally?
6. When would you use a bidirectional LSTM vs a unidirectional one?

**Answers:**
1. hₜ = tanh(Wₓₕ·xₜ + Wₕₕ·hₜ₋₁ + bₕ). Linear combo of current input and previous hidden state, squashed by tanh.
2. Forget: fₜ=σ(Wf[hₜ₋₁,xₜ]+bf) — controls what to remove from cell. Input: iₜ=σ(Wi[hₜ₋₁,xₜ]+bi), C̃ₜ=tanh(Wc[hₜ₋₁,xₜ]+bc) — controls what to add. Output: oₜ=σ(Wo[hₜ₋₁,xₜ]+bo) — controls what to output from cell.
3. RNN: gradient multiplies by Wₕₕ at each step → exponential shrinking/growth. LSTM: gradient through cell state only multiplies by forget gate fₜ (scalar near 1) → gradient preserved.
4. 4 × (256 × (256+100) + 256) = 4 × (91,136 + 256) = 4 × 91,392 = 365,568 parameters.
5. LSTM has separate cell state (Cₜ) and hidden state (hₜ) with 3 gates. GRU merges both into one hidden state with 2 gates. GRU: hₜ=(1-zₜ)⊙hₜ₋₁+zₜ⊙h̃ₜ.
6. Bidirectional: when full sequence is available (classification, NER, encoding). Unidirectional: when generating/predicting sequentially (text generation, language modeling).

---

## Recall Checklist

- [ ] Draw and explain RNN architecture (unrolled)
- [ ] Write RNN hidden state equation
- [ ] Explain BPTT and vanishing gradients in RNNs
- [ ] Draw and explain LSTM architecture with all gates
- [ ] Write all LSTM equations (forget, input, cell update, output)
- [ ] Explain why cell state preserves gradients
- [ ] Write GRU equations and compare with LSTM
- [ ] Implement LSTM in PyTorch with embedding layer
- [ ] Explain bidirectional vs unidirectional
- [ ] Describe Seq2Seq with attention
- [ ] Compare RNN/LSTM with Transformers
