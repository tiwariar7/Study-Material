# Day 7: Deep Learning Foundations — Perceptron, MLPs, Forward & Backward Propagation

---

## Topics Covered
1. **Perceptron** (Single-Layer, Limitations)
2. **Multi-Layer Perceptrons (MLPs)** (Architecture, Hidden Layers)
3. **Forward Propagation** (Computation Flow)
4. **Backward Propagation** (Chain Rule, Gradient Computation)

---

## 1. PERCEPTRON

### 1.1 Definition & Fundamentals

**Definition:** A perceptron is the simplest neural network — a single artificial neuron that computes a weighted sum of inputs, adds a bias, applies a step function, and outputs a binary decision.

**Analogy:** Like a biological neuron: receives inputs (dendrites), processes them (cell body), fires or doesn't (axon).

**Mathematical Model:**
```
z = w₁x₁ + w₂x₂ + ... + wₙxₙ + b = wᵀx + b
y = step(z) = { 1 if z ≥ 0, 0 if z < 0 }
```

### 1.2 Perceptron Learning Algorithm

```
Initialize weights w = 0, bias b = 0
For each epoch:
    For each training example (xᵢ, yᵢ):
        ŷ = step(wᵀxᵢ + b)
        If ŷ ≠ yᵢ:
            w = w + α(yᵢ - ŷ)xᵢ
            b = b + α(yᵢ - ŷ)
```

**Convergence Theorem:** If data is **linearly separable**, the perceptron algorithm will converge in a finite number of steps. If not, it oscillates forever.

### 1.3 Limitations — The XOR Problem

A single perceptron can only learn **linearly separable** functions.

| Function | Linearly Separable? | Perceptron Can Learn? |
|----------|-------------------|--------------------|
| AND | Yes | ✓ |
| OR | Yes | ✓ |
| NOT | Yes | ✓ |
| **XOR** | **No** | **✗** |

**XOR truth table:** (0,0)→0, (0,1)→1, (1,0)→1, (1,1)→0. No single line can separate the classes.

**Solution:** Stack multiple perceptrons → Multi-Layer Perceptron (MLP).

---

## 2. MULTI-LAYER PERCEPTRONS (MLPs)

### 2.1 Definition

**Definition:** An MLP is a feedforward neural network with one or more **hidden layers** between the input and output layers. Each layer is **fully connected** (dense) to the next.

### 2.2 Architecture

```
Input Layer       Hidden Layer(s)      Output Layer
  x₁ ─────┐
  x₂ ─────┼──→ h₁ ──→ h₂ ──→ ... ──→ ŷ
  x₃ ─────┤
  ...      │    (with activation       (sigmoid for binary,
  xₙ ─────┘     functions)             softmax for multiclass,
                                        linear for regression)
```

**Components:**
- **Input layer:** Features (no computation, just passes data).
- **Hidden layer(s):** Each neuron computes z = wᵀh + b, then applies activation function.
- **Output layer:** Final prediction.
- **Weights (W):** Connections between neurons (learned parameters).
- **Biases (b):** Offset terms (one per neuron, learned).
- **Activation function:** Non-linearity applied after linear transformation (crucial — without it, MLP collapses to linear model).

### 2.3 Why Non-Linear Activations Are Essential

**Without activations:** Each layer is just a linear transformation. Stacking linear transformations = one linear transformation. The network can't learn non-linear patterns.

```
Layer 1: h = W₁x + b₁
Layer 2: y = W₂h + b₂ = W₂(W₁x + b₁) + b₂ = (W₂W₁)x + (W₂b₁ + b₂) = W'x + b'
```
Still linear! Activation functions break this.

### 2.4 Universal Approximation Theorem

**Statement:** A feedforward network with a single hidden layer containing a finite number of neurons can approximate any continuous function on a compact subset of ℝⁿ, given appropriate weights and a non-linear activation function.

**Practical caveat:** The theorem guarantees existence but doesn't say how many neurons are needed or how to find the weights. Deep networks (multiple layers) are more practical and efficient.

### 2.5 Counting Parameters

For a network with layers [input=n₀, hidden₁=n₁, hidden₂=n₂, output=n₃]:
```
Parameters = (n₀×n₁ + n₁) + (n₁×n₂ + n₂) + (n₂×n₃ + n₃)
           = Σᵢ (nᵢ × nᵢ₊₁ + nᵢ₊₁)
```
Each connection = 1 weight. Each neuron in hidden/output = 1 bias.

**Example:** [784, 128, 64, 10] (MNIST)
```
(784×128+128) + (128×64+64) + (64×10+10) = 100,480 + 8,256 + 650 = 109,386 parameters
```

---

## 3. FORWARD PROPAGATION

### 3.1 Definition

**Definition:** Forward propagation is the process of computing the output of a neural network by passing the input through each layer sequentially — applying linear transformations and activation functions at each step.

### 3.2 Step-by-Step for a 2-Layer Network

Given input **x**, weights **W¹, W²**, biases **b¹, b²**, activation **σ**:

```
Layer 1 (hidden):
  z¹ = W¹x + b¹          (linear transformation)
  a¹ = σ(z¹)             (activation — e.g., ReLU)

Layer 2 (output):
  z² = W²a¹ + b²          (linear transformation)
  ŷ = a² = σ_out(z²)      (output activation — sigmoid/softmax)

Loss:
  L = Loss(ŷ, y)          (e.g., cross-entropy, MSE)
```

### 3.3 Matrix Dimensions

For batch of m samples, n₀ input features, n₁ hidden neurons, n₂ output neurons:
```
X:  (m × n₀)    Input batch
W¹: (n₀ × n₁)   Weights layer 1
b¹: (1 × n₁)    Bias layer 1 (broadcast)
Z¹: (m × n₁)    Pre-activation layer 1
A¹: (m × n₁)    Post-activation layer 1
W²: (n₁ × n₂)   Weights layer 2
Z²: (m × n₂)    Pre-activation output
Ŷ:  (m × n₂)    Output predictions
```

### 3.4 Implementation

```python
import numpy as np

def forward_pass(X, W1, b1, W2, b2):
    """Forward propagation for 2-layer network."""
    # Layer 1
    Z1 = X @ W1 + b1           # (m, n1)
    A1 = np.maximum(0, Z1)      # ReLU activation
    
    # Layer 2 (output)
    Z2 = A1 @ W2 + b2           # (m, n2)
    
    # Softmax for classification
    exp_Z2 = np.exp(Z2 - np.max(Z2, axis=1, keepdims=True))  # numerical stability
    Y_hat = exp_Z2 / exp_Z2.sum(axis=1, keepdims=True)
    
    cache = (Z1, A1, Z2, Y_hat)  # Save for backprop
    return Y_hat, cache
```

---

## 4. BACKWARD PROPAGATION (BACKPROPAGATION)

### 4.1 Definition

**Definition:** Backpropagation computes the gradient of the loss function with respect to each weight in the network by applying the **chain rule** of calculus, propagating the error backward from the output layer to the input layer.

**Plain English:** How much does each weight contribute to the total error? Backprop efficiently answers this by working backwards through the network.

### 4.2 The Chain Rule — Foundation of Backprop

For a composite function f(g(x)):
```
df/dx = df/dg × dg/dx
```

For a deep network: L depends on z² depends on a¹ depends on z¹ depends on W¹:
```
∂L/∂W¹ = ∂L/∂ŷ × ∂ŷ/∂z² × ∂z²/∂a¹ × ∂a¹/∂z¹ × ∂z¹/∂W¹
```

### 4.3 Step-by-Step Backprop for 2-Layer Network

**Setup:** Input x, hidden a¹ = σ(W¹x+b¹), output ŷ = σ(W²a¹+b²), loss L.

**Step 1: Output gradient**
```
For cross-entropy + softmax: δ² = ŷ - y     (elegant simplification)
For MSE + sigmoid: δ² = (ŷ - y) × σ'(z²)
```

**Step 2: Gradients for output weights**
```
∂L/∂W² = (a¹)ᵀ × δ²     (shape: n₁ × n₂)
∂L/∂b² = Σ δ²             (sum over batch)
```

**Step 3: Propagate error to hidden layer**
```
δ¹ = δ² × (W²)ᵀ ⊙ σ'(z¹)    (element-wise multiply with activation derivative)
```
For ReLU: σ'(z¹) = 1 if z¹ > 0, else 0.

**Step 4: Gradients for hidden weights**
```
∂L/∂W¹ = xᵀ × δ¹
∂L/∂b¹ = Σ δ¹
```

**Step 5: Update weights**
```
W² = W² - α × ∂L/∂W²
b² = b² - α × ∂L/∂b²
W¹ = W¹ - α × ∂L/∂W¹
b¹ = b¹ - α × ∂L/∂b¹
```

### 4.4 Computational Graph View

```
x → [W¹x+b¹] → z¹ → [σ] → a¹ → [W²a¹+b²] → z² → [σ_out] → ŷ → [Loss] → L
                                                                          ↑
                                                                          y (true)

Forward: left → right (compute outputs)
Backward: right → left (compute gradients via chain rule)
```

### 4.5 Complete Implementation from Scratch

```python
import numpy as np

class SimpleNeuralNetwork:
    def __init__(self, input_size, hidden_size, output_size, lr=0.01):
        # Xavier initialization (good for sigmoid/tanh)
        self.W1 = np.random.randn(input_size, hidden_size) * np.sqrt(2.0 / input_size)
        self.b1 = np.zeros((1, hidden_size))
        self.W2 = np.random.randn(hidden_size, output_size) * np.sqrt(2.0 / hidden_size)
        self.b2 = np.zeros((1, output_size))
        self.lr = lr
    
    def relu(self, z):
        return np.maximum(0, z)
    
    def relu_derivative(self, z):
        return (z > 0).astype(float)
    
    def softmax(self, z):
        exp_z = np.exp(z - np.max(z, axis=1, keepdims=True))
        return exp_z / exp_z.sum(axis=1, keepdims=True)
    
    def cross_entropy_loss(self, y_hat, y):
        m = y.shape[0]
        # y is one-hot encoded
        return -np.sum(y * np.log(y_hat + 1e-8)) / m
    
    def forward(self, X):
        self.Z1 = X @ self.W1 + self.b1
        self.A1 = self.relu(self.Z1)
        self.Z2 = self.A1 @ self.W2 + self.b2
        self.Y_hat = self.softmax(self.Z2)
        return self.Y_hat
    
    def backward(self, X, y):
        m = X.shape[0]
        
        # Output layer gradients
        dZ2 = self.Y_hat - y                    # (m, output_size)
        dW2 = (self.A1.T @ dZ2) / m             # (hidden, output)
        db2 = np.sum(dZ2, axis=0, keepdims=True) / m
        
        # Hidden layer gradients
        dA1 = dZ2 @ self.W2.T                    # (m, hidden)
        dZ1 = dA1 * self.relu_derivative(self.Z1)  # (m, hidden)
        dW1 = (X.T @ dZ1) / m                    # (input, hidden)
        db1 = np.sum(dZ1, axis=0, keepdims=True) / m
        
        # Update weights
        self.W2 -= self.lr * dW2
        self.b2 -= self.lr * db2
        self.W1 -= self.lr * dW1
        self.b1 -= self.lr * db1
    
    def train(self, X, y, epochs=1000):
        for epoch in range(epochs):
            y_hat = self.forward(X)
            loss = self.cross_entropy_loss(y_hat, y)
            self.backward(X, y)
            if epoch % 100 == 0:
                print(f"Epoch {epoch}, Loss: {loss:.4f}")

# XOR Example
X = np.array([[0,0], [0,1], [1,0], [1,1]])
y = np.array([[1,0], [0,1], [0,1], [1,0]])  # One-hot: [0,1]=XOR=1

nn = SimpleNeuralNetwork(2, 8, 2, lr=0.5)
nn.train(X, y, epochs=5000)
predictions = np.argmax(nn.forward(X), axis=1)
print(f"Predictions: {predictions}")  # [0, 1, 1, 0] ← XOR learned!
```

### 4.6 Weight Initialization

| Method | Formula | Best For |
|--------|---------|----------|
| **Zero** | W = 0 | **Never** — all neurons learn same thing (symmetry breaking problem) |
| **Random** | W ~ N(0, 0.01) | Basic, but can cause vanishing/exploding gradients |
| **Xavier/Glorot** | W ~ N(0, √(2/(nᵢₙ + nₒᵤₜ))) | Sigmoid, Tanh |
| **He** | W ~ N(0, √(2/nᵢₙ)) | **ReLU** (accounts for half being zeroed) |

**Why initialization matters:** Poor initialization → vanishing gradients (weights too small, signals die) or exploding gradients (weights too large, signals blow up).

### 4.7 Vanishing & Exploding Gradients

**Vanishing Gradients:**
- Gradients shrink exponentially as they propagate backward through many layers.
- Deeper layers barely learn while shallow layers stagnate.
- **Caused by:** Sigmoid/tanh (derivatives < 1), poor initialization.
- **Solutions:** ReLU activation, He initialization, skip connections (ResNet), batch normalization, LSTM/GRU.

**Exploding Gradients:**
- Gradients grow exponentially → weights update wildly.
- **Solutions:** Gradient clipping (cap gradient magnitude), proper initialization, batch normalization.

```python
# Gradient clipping
max_grad_norm = 1.0
for param in model.parameters():
    torch.nn.utils.clip_grad_norm_(param, max_grad_norm)
```

### 4.8 PyTorch Implementation

```python
import torch
import torch.nn as nn
import torch.optim as optim

class MLP(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super().__init__()
        self.fc1 = nn.Linear(input_size, hidden_size)
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(hidden_size, output_size)
    
    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = self.fc2(x)
        return x

# Setup
model = MLP(784, 128, 10)  # MNIST: 28×28 input, 10 classes
criterion = nn.CrossEntropyLoss()  # Includes softmax internally
optimizer = optim.Adam(model.parameters(), lr=0.001)

# Training loop
for epoch in range(10):
    for batch_X, batch_y in dataloader:
        # Forward
        outputs = model(batch_X)
        loss = criterion(outputs, batch_y)
        
        # Backward
        optimizer.zero_grad()   # Clear old gradients
        loss.backward()         # Compute gradients (autograd)
        optimizer.step()        # Update weights
    
    print(f"Epoch {epoch}, Loss: {loss.item():.4f}")
```

---

## Comparisons & Common Mistakes

### Common Mistakes
1. **Zero initialization** — all neurons learn the same thing (symmetry problem). Always use random initialization (Xavier/He).
2. **Using sigmoid in hidden layers** — causes vanishing gradients. Use ReLU.
3. **Forgetting `optimizer.zero_grad()`** in PyTorch — gradients accumulate by default.
4. **Not understanding why non-linearity is needed** — without it, deep network = linear model.
5. **Confusing number of parameters with number of neurons** — parameters = weights + biases.

---

## Interview Preparation

### Direct Interview Answers

**Q: What is backpropagation?**
> "Backpropagation computes the gradient of the loss with respect to each weight using the chain rule. The error signal flows backward from the output through each layer, computing how much each weight contributed to the error. These gradients are then used by an optimizer (e.g., SGD, Adam) to update weights and minimize the loss."

**Q: Why can't a single perceptron learn XOR?**
> "A perceptron learns a linear decision boundary (a straight line in 2D). XOR requires a non-linear boundary — points (0,1) and (1,0) are class 1, while (0,0) and (1,1) are class 0. No straight line can separate these. An MLP with a hidden layer introduces non-linearity, creating the curved boundary needed for XOR."

### Interview Q&A Bank

**Basic:**
1. **Q:** What is the role of the activation function? **A:** It introduces non-linearity. Without it, a deep network collapses to a single linear transformation. Non-linearity enables learning complex patterns.

2. **Q:** What is forward propagation? **A:** The process of passing input through the network layer by layer — applying linear transformations (Wx+b) and activation functions — to produce an output prediction.

**Intermediate:**
3. **Q:** Explain the chain rule in backpropagation with an example. **A:** If loss L depends on z which depends on w, then ∂L/∂w = ∂L/∂z × ∂z/∂w. For a deeper chain: ∂L/∂W¹ = ∂L/∂ŷ × ∂ŷ/∂z² × ∂z²/∂a¹ × ∂a¹/∂z¹ × ∂z¹/∂W¹. Each term is a local derivative, and we multiply them along the path.

4. **Q:** What is the vanishing gradient problem? **A:** In deep networks, gradients shrink exponentially as they propagate backward through layers with sigmoid/tanh activations (whose derivatives are < 1). Early layers get near-zero gradients and barely learn. Solutions: ReLU (derivative = 1 for positive values), skip connections, proper initialization.

**Advanced:**
5. **Q:** Compare Xavier and He initialization. When would you use each? **A:** Xavier (Glorot): W ~ N(0, √(2/(fan_in + fan_out))). Designed for sigmoid/tanh where the derivative range is balanced. He: W ~ N(0, √(2/fan_in)). Designed for ReLU where half the neurons output 0, so remaining ones need larger variance. Use He for ReLU networks (most modern networks).

6. **Q:** What is the Universal Approximation Theorem? What are its practical implications? **A:** A single hidden layer with enough neurons can approximate any continuous function. But "enough neurons" may be impractically many. Deep networks are more parameter-efficient — they build hierarchical representations (edges → textures → objects). The theorem justifies neural networks theoretically but doesn't solve practical optimization.

---

## Rapid Revision Summary

- **Perceptron:** Single neuron, step function, linear boundary only. Can't learn XOR.
- **MLP:** Multiple layers + non-linear activations → can learn any function (Universal Approximation).
- **Forward prop:** x → Wx+b → activation → next layer → ... → prediction → loss.
- **Backprop:** Loss → chain rule → compute ∂L/∂W for all layers → update weights.
- **Key gradients:** δ_output = ŷ-y, δ_hidden = (δ_next × W_next) ⊙ σ'(z). dW = xᵀ × δ.
- **Initialization:** Xavier for sigmoid/tanh, He for ReLU. Never zero.
- **Vanishing gradients:** Sigmoid derivatives < 1, gradients → 0. Solution: ReLU, skip connections, batch norm.

---

## Self-Assessment

1. Why does a perceptron fail on XOR? What is the minimum architecture to solve XOR?
2. How many parameters does a network with layers [100, 64, 32, 10] have?
3. Compute the forward pass for a single neuron: inputs=[1, 2, 3], weights=[0.5, -0.3, 0.8], bias=0.1, activation=ReLU.
4. Write the chain rule expression for ∂L/∂W¹ in a 3-layer network.
5. Why is zero initialization problematic? What happens?
6. Explain the difference between vanishing and exploding gradients.

**Answers:**
1. Perceptron has a linear decision boundary; XOR is not linearly separable. Minimum: 2 input, 2 hidden neurons, 1 output (with non-linear activations).
2. (100×64+64) + (64×32+32) + (32×10+10) = 6,464 + 2,080 + 330 = 8,874 parameters.
3. z = 1(0.5) + 2(-0.3) + 3(0.8) + 0.1 = 0.5 - 0.6 + 2.4 + 0.1 = 2.4. ReLU(2.4) = 2.4.
4. ∂L/∂W¹ = ∂L/∂a³ × ∂a³/∂z³ × ∂z³/∂a² × ∂a²/∂z² × ∂z²/∂a¹ × ∂a¹/∂z¹ × ∂z¹/∂W¹.
5. All neurons in a layer receive the same gradients and update identically → they remain identical (symmetry). The network acts as if it has one neuron per layer. Random initialization breaks this symmetry.
6. Vanishing: gradients → 0 in early layers (slow/no learning). Exploding: gradients → ∞ (unstable training, NaN). Both are caused by repeated multiplication of values < 1 or > 1 through many layers.

---

## Recall Checklist

- [ ] Define perceptron and its learning algorithm
- [ ] Explain why perceptron can't learn XOR
- [ ] Describe MLP architecture with all components
- [ ] Explain why non-linear activations are essential
- [ ] State the Universal Approximation Theorem
- [ ] Count parameters for a given network architecture
- [ ] Walk through forward propagation step by step
- [ ] Walk through backpropagation with chain rule
- [ ] Implement a simple neural network from scratch (NumPy)
- [ ] Implement an MLP in PyTorch
- [ ] Compare Xavier and He initialization
- [ ] Explain vanishing and exploding gradient problems and solutions
