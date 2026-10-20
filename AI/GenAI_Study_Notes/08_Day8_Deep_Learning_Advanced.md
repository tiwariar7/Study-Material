# Day 8: Deep Learning — Activation Functions, Optimizers, Regularization

---

## Topics Covered
1. **Activation Functions** (Sigmoid, Tanh, ReLU, Leaky ReLU, ELU, GELU, Swish, Softmax)
2. **Optimizers** (SGD, Momentum, RMSProp, Adam, AdamW, Learning Rate Scheduling)
3. **Regularization** (Dropout, Batch Normalization, L1/L2, Early Stopping, Data Augmentation)

---

## 1. ACTIVATION FUNCTIONS

### 1.1 Why Activation Functions?

Without activation functions, a deep network is equivalent to a single linear transformation. Activations introduce **non-linearity**, enabling the network to learn complex patterns.

### 1.2 Comprehensive Comparison

| Function | Formula | Range | Derivative | Pros | Cons |
|----------|---------|-------|-----------|------|------|
| **Sigmoid** | σ(z)=1/(1+e⁻ᶻ) | (0,1) | σ(1-σ) | Probabilistic output | Vanishing gradient, not zero-centered |
| **Tanh** | (eᶻ-e⁻ᶻ)/(eᶻ+e⁻ᶻ) | (-1,1) | 1-tanh²(z) | Zero-centered | Vanishing gradient |
| **ReLU** | max(0,z) | [0,∞) | 0 or 1 | Fast, no vanishing gradient | Dying ReLU (dead neurons) |
| **Leaky ReLU** | max(αz,z), α=0.01 | (-∞,∞) | α or 1 | No dying neurons | α is hyperparameter |
| **ELU** | z if z>0; α(eᶻ-1) if z≤0 | (-α,∞) | 1 or α·eᶻ | Smooth, zero-centered | Slow (exp computation) |
| **GELU** | z·Φ(z) | (-0.17,∞) | Complex | Used in BERT/GPT | Slower than ReLU |
| **Swish** | z·σ(z) | (-0.28,∞) | Complex | Self-gated, smooth | Slower than ReLU |
| **Softmax** | eᶻⁱ/Σeᶻʲ | (0,1), Σ=1 | pᵢ(1-pᵢ) | Multi-class probabilities | Only for output layer |

### 1.3 Detailed Analysis

**Sigmoid:**
```
σ(z) = 1 / (1 + e⁻ᶻ)
σ'(z) = σ(z) × (1 - σ(z))    Maximum derivative = 0.25 at z=0
```
- **Vanishing gradient:** For |z| > 5, σ'(z) ≈ 0. In deep networks, multiplying many small derivatives → gradient ≈ 0.
- **Not zero-centered:** Output always positive → zigzag gradient updates.
- **Use case:** Binary classification output layer, gating mechanisms (LSTM).

**ReLU (Rectified Linear Unit):**
```
ReLU(z) = max(0, z)
ReLU'(z) = { 1 if z > 0, 0 if z ≤ 0 }
```
- **Advantages:** Derivative is 1 for positive values → no vanishing gradient. Sparse activation (many zeros → efficient). Very fast to compute.
- **Dying ReLU problem:** If z < 0 for all inputs, gradient is always 0 → neuron never updates → "dead." Can happen with large learning rates. ~10-40% of neurons may die.
- **Solution:** Leaky ReLU, PReLU, ELU.

**GELU (Gaussian Error Linear Unit):**
```
GELU(z) = z × Φ(z) ≈ 0.5z(1 + tanh(√(2/π)(z + 0.044715z³)))
```
Where Φ(z) is the Gaussian CDF.
- **Used in:** BERT, GPT-2, GPT-3, most modern transformers.
- **Intuition:** Smoothly gates the input — can be thought of as a smooth approximation of ReLU.
- **Why popular in transformers:** Smooth, non-monotonic near 0, provides better gradients than ReLU.

**Swish:**
```
Swish(z) = z × σ(z) = z / (1 + e⁻ᶻ)
```
- Self-gated activation (sigmoid gates the linear part).
- Outperforms ReLU on deep networks.
- Non-monotonic (slightly negative for negative inputs).

### 1.4 When to Use What

| Scenario | Recommended Activation |
|----------|----------------------|
| Hidden layers (general) | **ReLU** (default) |
| Deep networks with dying ReLU | Leaky ReLU, ELU |
| Transformers / LLMs | **GELU** |
| Binary classification output | Sigmoid |
| Multi-class classification output | Softmax |
| Regression output | Linear (no activation) |
| RNN hidden state | Tanh |
| LSTM gates | Sigmoid (gates), Tanh (cell state) |

### 1.5 Implementation

```python
import torch
import torch.nn as nn

# Built-in activations
relu = nn.ReLU()
leaky_relu = nn.LeakyReLU(negative_slope=0.01)
elu = nn.ELU(alpha=1.0)
gelu = nn.GELU()
silu = nn.SiLU()  # Swish
sigmoid = nn.Sigmoid()
tanh = nn.Tanh()
softmax = nn.Softmax(dim=-1)

# In a model
class Model(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 256)
        self.fc2 = nn.Linear(256, 10)
        self.gelu = nn.GELU()
    
    def forward(self, x):
        x = self.gelu(self.fc1(x))  # GELU in hidden layer
        x = self.fc2(x)              # No activation (CrossEntropyLoss handles softmax)
        return x
```

---

## 2. OPTIMIZERS

### 2.1 Gradient Descent Variants

**Batch Gradient Descent:**
```
θ = θ - α × (1/m) Σᵢ₌₁ᵐ ∇_θ L(xᵢ, yᵢ)
```
- Uses **entire dataset** per update. Slow, exact gradient.

**Stochastic Gradient Descent (SGD):**
```
θ = θ - α × ∇_θ L(xᵢ, yᵢ)
```
- Uses **one sample** per update. Noisy but fast. Can escape local minima.

**Mini-Batch SGD (most common):**
```
θ = θ - α × (1/B) Σᵢ₌₁ᴮ ∇_θ L(xᵢ, yᵢ)
```
- Uses **batch of B samples** (typically 32, 64, 128, 256). Best of both worlds.

### 2.2 SGD with Momentum

**Problem with vanilla SGD:** Oscillates in ravines (high curvature), slow convergence.

**Solution:** Accumulate past gradients to build "velocity":
```
v_t = β × v_{t-1} + (1 - β) × g_t     (β typically 0.9)
θ = θ - α × v_t
```

**Intuition:** Like a ball rolling downhill — builds momentum in consistent gradient directions, dampens oscillations in inconsistent directions.

**Nesterov Momentum:** "Look ahead" — compute gradient at the anticipated next position:
```
v_t = β × v_{t-1} + α × ∇_θ L(θ - β × v_{t-1})
θ = θ - v_t
```
Slightly better convergence than standard momentum.

### 2.3 RMSProp (Root Mean Square Propagation)

**Problem:** Same learning rate for all parameters, but some gradients are large and some are small.

**Solution:** Adapt learning rate per-parameter using running average of squared gradients:
```
s_t = β × s_{t-1} + (1 - β) × g_t²          (β typically 0.999)
θ = θ - α × g_t / (√s_t + ε)
```

**Intuition:** Parameters with large gradients get smaller effective learning rates (divided by large √s). Parameters with small gradients get larger effective learning rates.

### 2.4 Adam (Adaptive Moment Estimation)

**Adam = Momentum + RMSProp.** The most popular optimizer.

```
m_t = β₁ × m_{t-1} + (1 - β₁) × g_t        # First moment (mean of gradients)
v_t = β₂ × v_{t-1} + (1 - β₂) × g_t²       # Second moment (mean of squared gradients)

# Bias correction (important in early steps when m, v are biased toward 0)
m̂_t = m_t / (1 - β₁ᵗ)
v̂_t = v_t / (1 - β₂ᵗ)

# Update
θ = θ - α × m̂_t / (√v̂_t + ε)
```

**Default hyperparameters:** α=0.001, β₁=0.9, β₂=0.999, ε=10⁻⁸.

**Why Adam works well:**
- Momentum (m_t) → accelerates in consistent gradient directions.
- Adaptive learning rate (v_t) → per-parameter adjustment.
- Bias correction → prevents zero-biased estimates in early training.

### 2.5 AdamW (Weight Decay)

**Problem with Adam + L2 regularization:** L2 regularization in Adam doesn't work as intended because the adaptive learning rates interfere with the regularization term.

**AdamW solution:** Decouple weight decay from the gradient:
```
θ = θ - α × (m̂_t / (√v̂_t + ε) + λ × θ)
```
Instead of adding L2 penalty to the loss (which Adam modifies), directly decay the weights.

**Used in:** All modern transformer training (BERT, GPT, etc.).

### 2.6 Optimizer Comparison

| Optimizer | Adaptive LR? | Momentum? | When to Use |
|-----------|-------------|-----------|-------------|
| **SGD** | No | Optional | Good generalization, CNNs, final fine-tuning |
| **SGD+Momentum** | No | Yes | Faster SGD convergence |
| **RMSProp** | Yes | No | RNNs, non-stationary problems |
| **Adam** | Yes | Yes | Default for most cases |
| **AdamW** | Yes | Yes | **Transformers, LLMs** (recommended) |

### 2.7 Learning Rate Scheduling

| Schedule | Description | Use Case |
|----------|-------------|----------|
| **Step Decay** | Reduce LR by factor every N epochs | Simple, predictable |
| **Cosine Annealing** | LR follows cosine curve to near-zero | Transformers (common) |
| **Warmup + Decay** | Linearly increase LR, then decay | LLM training, transformers |
| **ReduceLROnPlateau** | Reduce when metric stops improving | General training |
| **One Cycle** | Increase then decrease LR within one cycle | Fast convergence |

```python
import torch.optim as optim
from torch.optim.lr_scheduler import CosineAnnealingLR, StepLR

optimizer = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)

# Cosine Annealing (used in transformer training)
scheduler = CosineAnnealingLR(optimizer, T_max=100, eta_min=1e-6)

# Linear warmup + cosine decay (common for LLM training)
from transformers import get_linear_schedule_with_warmup
scheduler = get_linear_schedule_with_warmup(
    optimizer, num_warmup_steps=1000, num_training_steps=10000
)

# In training loop:
for epoch in range(epochs):
    train_one_epoch()
    scheduler.step()  # Update learning rate
```

---

## 3. REGULARIZATION

### 3.1 What is Regularization?

**Definition:** Techniques to prevent overfitting and improve generalization by constraining the model's capacity.

**Signs of overfitting:** Training loss decreases but validation loss increases (or plateaus).

### 3.2 Dropout

**Definition:** During training, randomly set a fraction p of neuron activations to zero at each forward pass. At test time, use all neurons but scale by (1-p).

**How it works:**
```
Training: mask = Bernoulli(1-p)    # Random binary mask
          a_dropout = a × mask / (1-p)   # Inverted dropout (scale up)

Testing:  a_test = a              # Use all neurons, no scaling needed
```

**Why it works:**
- Forces the network to not rely on any single neuron → redundant representations.
- Acts like training an ensemble of sub-networks (each forward pass uses a different subset).
- Provides noise regularization.

**Typical values:** p = 0.2-0.5 (drop 20-50% of neurons).
- **Higher p = more regularization** (more neurons dropped).
- Fully connected layers: p=0.5 typical. Convolutional layers: p=0.2-0.3. **Never apply to output layer.**

```python
class Model(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 256)
        self.dropout = nn.Dropout(p=0.5)
        self.fc2 = nn.Linear(256, 10)
    
    def forward(self, x):
        x = torch.relu(self.fc1(x))
        x = self.dropout(x)  # Applied only during training (model.train())
        x = self.fc2(x)
        return x

model.train()  # Dropout active
model.eval()   # Dropout disabled (all neurons used)
```

### 3.3 Batch Normalization

**Definition:** Normalize the inputs to each layer by subtracting the batch mean and dividing by the batch standard deviation, then scale and shift using learned parameters.

**Formula:**
```
μ_B = (1/B) Σᵢ xᵢ                    # Batch mean
σ²_B = (1/B) Σᵢ (xᵢ - μ_B)²         # Batch variance
x̂ᵢ = (xᵢ - μ_B) / √(σ²_B + ε)       # Normalize
yᵢ = γ × x̂ᵢ + β                       # Scale and shift (learnable!)
```

Where:
- **γ (gamma):** Learnable scale parameter. Initialized to 1.
- **β (beta):** Learnable shift parameter. Initialized to 0.
- **ε:** Small constant for numerical stability (typically 10⁻⁵).
- During **training:** uses batch statistics (μ_B, σ²_B).
- During **inference:** uses running average of batch statistics accumulated during training.

**Why it works:**
1. **Reduces internal covariate shift:** Each layer's input distribution stays stable.
2. **Allows higher learning rates:** Normalization stabilizes training.
3. **Mild regularization effect:** Batch statistics add noise (different each mini-batch).
4. **Reduces sensitivity to initialization.**

**Where to place BatchNorm:** After the linear layer, **before** the activation (most common):
```
x → Linear → BatchNorm → ReLU → next layer
```
Some practitioners place it after activation; both work in practice.

```python
class Model(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 256)
        self.bn1 = nn.BatchNorm1d(256)    # For fully connected
        self.fc2 = nn.Linear(256, 10)
    
    def forward(self, x):
        x = self.fc1(x)
        x = self.bn1(x)          # Normalize
        x = torch.relu(x)        # Activate
        x = self.fc2(x)
        return x

# For convolutional layers: nn.BatchNorm2d(num_channels)
```

### 3.4 Layer Normalization

**Difference from Batch Normalization:** Normalize across features (within a single sample) instead of across the batch.

```
LayerNorm: normalize over features (per sample)
BatchNorm: normalize over batch (per feature)
```

**When to use:**
- **BatchNorm:** CNNs, fixed batch sizes, image tasks.
- **LayerNorm:** **Transformers, RNNs**, variable sequence lengths. Doesn't depend on batch size.

```python
# LayerNorm (used in Transformers)
self.layer_norm = nn.LayerNorm(hidden_size)

# In Transformer block:
x = x + self.attention(self.layer_norm(x))   # Pre-norm (GPT-style)
# or
x = self.layer_norm(x + self.attention(x))   # Post-norm (original Transformer)
```

### 3.5 Other Regularization Techniques

| Technique | Description |
|-----------|-------------|
| **L1/L2 Regularization** | Add penalty to loss: L1 (sparsity), L2 (weight decay). In Adam, use AdamW for proper L2. |
| **Early Stopping** | Stop training when validation loss starts increasing. Monitor val_loss with patience. |
| **Data Augmentation** | Artificially increase training data: rotations, flips, crops, noise (images); synonym replacement, back-translation (text). |
| **Weight Tying** | Share weights between layers (e.g., encoder-decoder embedding in transformers). |
| **Label Smoothing** | Instead of hard labels [0,1], use soft labels [ε/(K-1), 1-ε]. Prevents overconfidence. |

```python
# Early Stopping (manual implementation)
best_val_loss = float('inf')
patience, counter = 10, 0

for epoch in range(epochs):
    train_loss = train_one_epoch()
    val_loss = validate()
    
    if val_loss < best_val_loss:
        best_val_loss = val_loss
        counter = 0
        torch.save(model.state_dict(), 'best_model.pth')
    else:
        counter += 1
        if counter >= patience:
            print("Early stopping!")
            break
```

### 3.6 Dropout vs Batch Normalization

| Aspect | Dropout | Batch Normalization |
|--------|---------|-------------------|
| Mechanism | Randomly zero neurons | Normalize activations |
| Applied during | Training only | Training + Inference (different stats) |
| Regularization | Strong | Mild |
| Speedup | No (actually slower) | Yes (allows higher LR) |
| Common in | FC layers, attention | CNNs, Transformers |
| Interaction | **Can conflict** — dropout changes batch stats, confusing BN |

**Best practice:** In modern architectures, BatchNorm/LayerNorm is preferred. Dropout is still used in fully connected layers and attention (attention dropout). Don't use both on the same layer without careful consideration.

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Forgetting `model.eval()`** during inference — dropout stays active, batch norm uses wrong stats.
2. **Using sigmoid in hidden layers** — use ReLU/GELU. Sigmoid causes vanishing gradients.
3. **Using Adam with L2 regularization instead of AdamW** — the adaptive LR interferes with L2. Use AdamW.
4. **Learning rate too high** — loss diverges. Too low — training is slow. Use warmup + decay.
5. **Not using learning rate scheduling** — a constant LR is rarely optimal.
6. **Applying batch norm to small batch sizes** — noisy statistics. Use LayerNorm or GroupNorm instead.

---

## Interview Preparation

### Direct Interview Answers

**Q: What is the dying ReLU problem?**
> "When a neuron's input is always negative, ReLU outputs 0 and has 0 gradient, so the weights never update — the neuron is permanently 'dead.' This can happen with large learning rates pushing weights into a negative regime. Solutions: Leaky ReLU (small slope for negative values), ELU, or careful initialization (He initialization). In practice, ~10-40% of neurons may die but the network still works."

**Q: Explain Batch Normalization.**
> "BatchNorm normalizes layer inputs by subtracting the batch mean and dividing by batch standard deviation, then applies learnable scale (γ) and shift (β) parameters. This stabilizes training, allows higher learning rates, reduces sensitivity to initialization, and provides mild regularization. During inference, it uses running averages accumulated during training."

### Interview Q&A Bank

**Basic:**
1. **Q:** Why is ReLU preferred over sigmoid? **A:** ReLU has gradient of 1 for positive inputs (no vanishing gradient), is computationally simple (max(0,z)), creates sparse activations, and converges faster. Sigmoid gradients vanish for large/small inputs and output isn't zero-centered.

2. **Q:** What is dropout and why does it work? **A:** Randomly zeros neuron activations during training. Forces redundant representations (no single neuron is critical), acts like ensemble of sub-networks, provides noise regularization. Disabled during inference.

**Intermediate:**
3. **Q:** Explain Adam optimizer. **A:** Adam combines momentum (exponentially weighted average of gradients) with RMSProp (exponentially weighted average of squared gradients). It adapts learning rates per-parameter and uses bias correction for early steps. Default: lr=0.001, β₁=0.9, β₂=0.999. Most popular general-purpose optimizer.

4. **Q:** What is the difference between BatchNorm and LayerNorm? **A:** BatchNorm normalizes across the batch dimension (per feature). LayerNorm normalizes across the feature dimension (per sample). LayerNorm doesn't depend on batch size and works with variable-length sequences, making it the standard for Transformers and RNNs.

**Advanced:**
5. **Q:** Why does AdamW work better than Adam + L2 for transformers? **A:** In standard Adam, L2 regularization is added to the gradient, but the adaptive learning rates scale it differently for each parameter, effectively applying different regularization strengths. AdamW decouples weight decay from gradient-based update, applying it directly and uniformly to weights. This gives more consistent regularization.

6. **Q:** What activation function do modern LLMs (GPT, BERT) use and why? **A:** GELU (Gaussian Error Linear Unit): z × Φ(z). It's smooth, non-monotonic near zero, and provides better gradient flow than ReLU. The probabilistic interpretation (stochastic gating based on input magnitude) helps with transformer attention patterns. Some newer models use SwiGLU (Swish-gated linear unit) for even better performance.

---

## Rapid Revision Summary

- **Activations:** ReLU (default hidden), GELU (transformers), Sigmoid (binary output/gates), Softmax (multiclass output).
- **Dying ReLU:** Neurons stuck at 0. Fix: Leaky ReLU, He init.
- **SGD:** Simple, good generalization. **Adam:** Adaptive LR + momentum, most popular. **AdamW:** Decoupled weight decay, for transformers.
- **LR schedule:** Warmup + cosine decay for transformers. ReduceLROnPlateau for general training.
- **Dropout:** Random neuron zeroing during training. p=0.1-0.5. Ensemble effect.
- **BatchNorm:** Normalize per-feature across batch. Learnable γ, β. Allows higher LR.
- **LayerNorm:** Normalize per-sample across features. Used in Transformers.
- **Early Stopping:** Stop when val_loss increases for `patience` epochs.

---

## Self-Assessment

1. Compare sigmoid, tanh, and ReLU — give the formula, range, derivative, and one key advantage/disadvantage.
2. What is the dying ReLU problem? Give two solutions.
3. Write the Adam update equations and explain each component.
4. Why is LayerNorm used in Transformers instead of BatchNorm?
5. Explain dropout's inverted scaling — why multiply by 1/(1-p) during training?
6. What is early stopping and how do you implement it?

**Answers:**
1. Sigmoid: 1/(1+e⁻ᶻ), (0,1), σ(1-σ), pro: probability output, con: vanishing gradient. Tanh: (eᶻ-e⁻ᶻ)/(eᶻ+e⁻ᶻ), (-1,1), 1-tanh², pro: zero-centered, con: vanishing gradient. ReLU: max(0,z), [0,∞), 0 or 1, pro: no vanishing gradient, con: dying neurons.
2. Neurons stuck at 0 output with 0 gradient → never update. Solutions: Leaky ReLU (small negative slope), He initialization (prevents initial values from being all negative), ELU.
3. m=β₁m+(1-β₁)g (momentum), v=β₂v+(1-β₂)g² (adaptive LR), bias correct both, update θ=θ-α·m̂/√(v̂+ε).
4. BatchNorm depends on batch statistics — in Transformers, batch sizes vary and sequences have different lengths. LayerNorm normalizes per-sample across features, independent of batch size.
5. During training, if we drop 50% of neurons, the expected sum of activations is halved. Multiplying by 1/(1-p) compensates, so expected output magnitude matches inference (when all neurons are active). This is "inverted dropout" — scales at training time so no change needed at inference.
6. Monitor validation loss. If it doesn't improve for `patience` epochs, stop training and reload the best model checkpoint.

---

## Recall Checklist

- [ ] List 6+ activation functions with formulas, ranges, and use cases
- [ ] Explain dying ReLU and its solutions
- [ ] Explain GELU and why it's used in transformers
- [ ] Compare SGD, Momentum, RMSProp, Adam, AdamW
- [ ] Write Adam's update equations
- [ ] Explain learning rate warmup and cosine annealing
- [ ] Explain dropout mechanism and inverted scaling
- [ ] Write BatchNorm formula and explain γ, β
- [ ] Compare BatchNorm vs LayerNorm
- [ ] Implement dropout and batch norm in PyTorch
- [ ] List 5+ regularization techniques
- [ ] Implement early stopping
