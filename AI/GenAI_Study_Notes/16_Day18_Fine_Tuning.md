# Day 18: Fine-Tuning LLMs — LoRA, QLoRA, PEFT & Quantization

---

## Topics Covered
1. **Decision Matrix: Prompting vs RAG vs Fine-Tuning**
2. **Full Fine-Tuning vs PEFT (Parameter-Efficient Fine-Tuning)**
   - Catastrophic Forgetting & VRAM Memory Math (AdamW overhead)
3. **LoRA (Low-Rank Adaptation — Hu et al., 2021)**:
   - Mathematical derivation: $W = W_0 + \frac{\alpha}{r}(B \cdot A)$
   - Rank $r$, scaling factor $\alpha$, and target module selection
   - Zero-latency weight merging at inference time
4. **QLoRA (Quantized Low-Rank Adaptation — Dettmers et al., 2023)**:
   - 4-bit NormalFloat (NF4) data type
   - Double Quantization (DQ)
   - Paged Optimizers (VRAM spike protection via CPU paging)
5. **Quantization Foundations**:
   - Precision types: FP32, BF16/FP16, INT8, INT4
   - Post-Training Quantization (PTQ: AWQ, GPTQ, GGUF) vs Quantization-Aware Training (QAT)
6. **Implementation with Hugging Face (`trl`, `peft`, `bitsandbytes`)**
7. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. DECISION MATRIX: PROMPTING VS RAG VS FINE-TUNING

```
                         DO YOU NEED NEW KNOWLEDGE OR NEW BEHAVIOR/STYLE?
                                                |
                 +------------------------------+------------------------------+
                 |                                                             |
        NEW FACTUAL KNOWLEDGE                                          NEW BEHAVIOR / SYNTAX / STYLE
                 |                                                             |
      Frequent updates? Docs?                                       Fixed format, specialized syntax,
                 |                                                  or custom tone?
                 ↓                                                             ↓
         USE RAG (Retrieval)                                        USE FINE-TUNING (LoRA/QLoRA)
      (Dynamic, traceable facts,                                   (Format compliance, domain style,
       zero weight modifications)                                   drastically reduces prompt tokens)
```

| Dimension | Prompting (Few-Shot) | Retrieval-Augmented Generation (RAG) | Fine-Tuning (PEFT / LoRA) |
| :--- | :--- | :--- | :--- |
| **Primary Purpose** | Guiding general reasoning | Accessing dynamic/proprietary data | Adapting style, tone, format, or task syntax |
| **Model Weights** | Unchanged | Unchanged | Modified (or adapter added) |
| **Hallucination Risk**| Moderate to High | Low (grounded in retrieved context) | Moderate (can still hallucinate facts) |
| **Training Compute** | None | None | Moderate (single GPU for LoRA/QLoRA) |
| **Token Cost** | High (tokens repeated in every call) | High (retrieved chunks add prompt tokens) | Low (knowledge/format baked into weights) |

---

## 2. VRAM MATHEMATICS: WHY FULL FINE-TUNING IS PROHIBITIVE

To train a model with $N$ parameters in 16-bit precision using the standard AdamW optimizer:

$$\text{Total VRAM per Parameter} \approx 16 \text{ to } 20 \text{ Bytes}$$

### Breakdown per Parameter:
1. **Model Weights (FP16/BF16):** 2 bytes
2. **Gradients (FP16/BF16):** 2 bytes
3. **AdamW Optimizer States (FP32):**
   - Momentum (1st moment vector): 4 bytes
   - Variance (2nd moment vector): 4 bytes
   - Master weights (FP32 copy for precision): 4 bytes
   - *Total Optimizer states = 12 bytes!*
4. **Activations & Temp Buffers:** $\approx 2$ to $4$ bytes

**Concrete Example: Llama 3 70B Full Fine-Tuning**
$$\text{VRAM} = 70 \times 10^9 \times 16 \text{ bytes} \approx 1,120 \text{ GB of VRAM}$$
Requires a cluster of at least **16 $\times$ NVIDIA A100/H100 80GB GPUs** just to hold the optimizer state and weights!

---

## 3. LoRA (LOW-RANK ADAPTATION)

### 3.1 Intuition & Hypothesis
Aghajanyan et al. (2020) demonstrated that pre-trained language models have a very low **intrinsic dimension**—meaning the parameter updates $\Delta W$ during task adaptation reside in a low-dimensional subspace.

Instead of updating the full frozen weight matrix $W_0 \in \mathbb{R}^{d \times k}$, LoRA decomposes the weight update $\Delta W$ into two low-rank matrices:
$$\Delta W = B \cdot A$$
Where:
- $W_0 \in \mathbb{R}^{d \times k}$ (Frozen original weights)
- $A \in \mathbb{R}^{r \times k}$ (Initialized with Gaussian random noise $\mathcal{N}(0, \sigma^2)$)
- $B \in \mathbb{R}^{d \times r}$ (Initialized with zeros, so $\Delta W = 0$ at the start of training)
- $r \ll \min(d, k)$ (Rank, typically $r \in [8, 64]$)

```
                       LoRA FORWARD PASS
                        Input Vector x (1 x d)
                                |
             +------------------+------------------+
             |                                     |
             ↓ (Frozen)                            ↓ (Trainable)
     Original Weights W_0                     Matrix A (k x r)
          (d x k)                                  |
             |                                     ↓
             |                              Matrix B (r x d)
             |                                     |
             |                              Multiply by α / r
             |                                     |
             ↓                                     ↓
          W_0 · x               +             (α/r) · B · A · x
             \                                     /
              +-----------------+-----------------+
                                |
                                ↓
                        Output Vector h (1 x k)
```

### 3.2 Scaling Factor ($\alpha$)
The forward pass is:
$$h = W_0 x + \frac{\alpha}{r} (B A) x$$
Where $\alpha$ is a constant scaling hyperparameter.
- Setting $\alpha = 2r$ is standard practice (e.g., $r=16, \alpha=32$).
- The scaling factor $\frac{\alpha}{r}$ keeps the learning dynamics stable when adjusting rank $r$, avoiding retuning the learning rate.

### 3.3 Zero Inference Overhead: Weight Merging
At serving time, we do not need to compute two separate matrix multiplications! Because matrix multiplication is distributive:
$$W_{merged} = W_0 + \frac{\alpha}{r}(B \cdot A)$$
The trained adapter weights can be permanently added back into the base model weights, resulting in **zero added inference latency or memory overhead**.

---

## 4. QLoRA (QUANTIZED LOW-RANK ADAPTATION)

QLoRA (Dettmers et al., 2023) reduces the memory requirement of fine-tuning so drastically that a **70B parameter model can be fine-tuned on a single 48GB GPU**, or a **7B model on a single 16GB consumer GPU (RTX 4090)**.

```
       LoRA (16-bit Base + 16-bit Adapters)     --> ~160 GB VRAM (70B)
       QLoRA (4-bit Base + 16-bit Adapters)    --> ~48 GB VRAM (70B)
```

### 4.1 Three Breakthrough Innovations in QLoRA:
1. **NF4 (NormalFloat4) Quantization:**
   - Standard integers (INT4) space quantiles uniformly. However, pre-trained neural network weights follow a normal distribution $\mathcal{N}(0, \sigma^2)$.
   - NF4 assigns information-theoretically optimal quantile bin intervals to normal distributions, yielding significantly higher precision than standard 4-bit integer quantization.
2. **Double Quantization (DQ):**
   - Quantization maps FP16 weights to 4-bit integers by computing a quantization constant (scale factor $c_1$) for blocks of 64 parameters ($32 \text{ bits} / 64 = 0.5 \text{ bits/param}$).
   - Double Quantization quantizes those scale factors $c_1$ themselves into 8-bit integers, reducing memory from $0.5$ bits/param to $0.127$ bits/param (saving 0.37 bits/param $\approx 3$ GB on a 65B model).
3. **Paged Optimizers:**
   - Leverages CUDA Unified Memory to automatically page memory from GPU VRAM to CPU RAM during memory-intensive activation spikes, preventing CUDA Out-of-Memory (OOM) crashes.

---

## 5. QUANTIZATION SCHEMES COMPARISON

```
+--------+--------------------+---------------------+-----------------------------+
| Type   | Bits per Weight    | Memory for 7B Model | Primary Execution Use Case  |
+--------+--------------------+---------------------+-----------------------------+
| FP32   | 32 bits (4 bytes)  | ~28 GB              | Baseline research training  |
| BF16   | 16 bits (2 bytes)  | ~14 GB              | Standard training & serving |
| INT8   | 8 bits (1 byte)    | ~7 GB               | BitsAndBytes / SmoothQuant  |
| INT4   | 4 bits (0.5 byte)  | ~3.5 GB             | QLoRA, AWQ, GPTQ, GGUF      |
+--------+--------------------+---------------------+-----------------------------+
```

### Post-Training Quantization (PTQ) Methods:
- **AWQ (Activation-aware Weight Quantization):** Protects the top 1% most salient weight channels based on activation magnitudes, quantizing the remaining 99% to 4-bit with near-zero perplexity loss.
- **GPTQ:** Uses second-order Taylor expansion (inverse Hessian) to quantize weights layer by layer, adjusting remaining weights to compensate for quantization error.
- **GGUF (llama.cpp):** Optimized for CPU and Apple Metal GPU execution, supporting integer and k-quant hybrid schemes (e.g., `Q4_K_M`).

---

## 6. COMPLETE QLoRA FINE-TUNING CODE WITH PEFT & TRL

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer, SFTConfig

model_id = "meta-llama/Meta-Llama-3-8B"

# 1. Configure 4-bit NF4 Quantization
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True
)

# 2. Load Base Model in 4-bit
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    quantization_config=bnb_config,
    device_map="auto"
)
model = prepare_model_for_kbit_training(model)

# 3. Configure LoRA Parameters
peft_config = LoraConfig(
    r=16,                           # Rank
    lora_alpha=32,                  # Scaling factor
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"]
)
model = get_peft_model(model, peft_config)
model.print_trainable_parameters()
# Output: trainable params: ~41M || all params: ~8B || trainable%: ~0.5%

# 4. Train with SFTTrainer
training_args = SFTConfig(
    output_dir="./lora_output",
    learning_rate=2e-4,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    max_seq_length=2048,
    logging_steps=10,
    fp16=False,
    bf16=True
)
# trainer = SFTTrainer(model=model, args=training_args, peft_config=peft_config, ...)
# trainer.train()
```

---

## 7. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: Why does fine-tuning on a small dataset often degrade the general capabilities of an LLM (Catastrophic Forgetting)?**
   - **A:** When updating all base model weights on a narrow domain dataset, the optimization moves the weights towards a local minimum specific to the new task, overwriting the broadly generalized representations learned during pre-training over trillions of diverse tokens. PEFT (LoRA) mitigates this by freezing the original weights $W_0$ entirely, preserving core language capabilities.

2. **Q: Why is Matrix $B$ initialized to zero while Matrix $A$ is initialized to Gaussian random noise in LoRA?**
   - **A:** If both $A$ and $B$ were initialized with random values, $\Delta W = B \cdot A$ would initially be non-zero, corrupting the pre-trained weights from step 0 and harming model performance before training even begins. Initializing $B=0$ guarantees $\Delta W = 0 \cdot A = 0$ at step 0, ensuring training starts smoothly from the exact pre-trained baseline.

### Intermediate
3. **Q: What is the purpose of the $\alpha$ scaling factor in LoRA?**
   - **A:** $\alpha$ scales the magnitude of the adapter's contribution ($\frac{\alpha}{r} \Delta W$). When experimenting with different ranks $r$ (e.g., increasing $r$ from 8 to 64 to add expressiveness), keeping $\alpha$ fixed or scaling proportionally ensures that the overall magnitude of the parameter updates remains stable, preventing the need to re-tune learning rates across rank changes.

4. **Q: How does NF4 (NormalFloat4) outperform standard INT4 quantization?**
   - **A:** Standard INT4 uses evenly spaced intervals between minimum and maximum values. However, weights in pre-trained neural networks are distributed normally (Gaussian) with high density near 0 and thin tails. NF4 spaces quantization bins so that each bin contains an equal expected number of weights, maximizing information entropy and minimizing quantization error.

### Advanced
5. **Q: Explain how QLoRA executes backpropagation through a 4-bit base model weight matrix.**
   - **A:** During the forward pass, 4-bit NF4 base model weights $W_0$ are dequantized on-the-fly into 16-bit BF16 numbers in GPU registers/SRAM, multiplied by the input $x$, and immediately discarded. During the backward pass, gradients are computed in BF16 and passed only to the trainable LoRA adapter matrices ($A$ and $B$). The 4-bit weights themselves are frozen and never receive gradient updates, keeping VRAM requirements strictly bound to the small adapter parameters and optimizer states.

---

## 8. RAPID REVISION SUMMARY

- **PEFT / LoRA:** Decomposes weight updates $\Delta W = B \cdot A$ with rank $r \ll d$. Reduces trainable params by $99\%+$.
- **Inference Merging:** $W = W_0 + \frac{\alpha}{r}(BA) \to$ zero latency penalty at runtime.
- **QLoRA:** 4-bit NF4 base weights + Double Quantization + Paged Optimizers $\to$ fine-tunes 70B on 48GB GPU.
- **AdamW Memory Overhead:** 12 bytes per parameter in optimizer state alone; full fine-tuning takes $\approx 16-20$ bytes/param.
- **Quantization:** AWQ / GPTQ for fast 4-bit inference; GGUF for edge/CPU inference.

---

## 9. RECALL CHECKLIST

- [ ] Calculate the VRAM required to fully fine-tune a 7B model using AdamW in FP16.
- [ ] Write the LoRA forward pass equation including scaling parameter $\alpha$.
- [ ] Explain why matrix $B$ is initialized to zero and matrix $A$ to Gaussian random values.
- [ ] Explain how LoRA weights are merged back into base weights for zero-overhead inference.
- [ ] List the three core innovations of QLoRA (NF4, Double Quantization, Paged Optimizers).
- [ ] Explain why NF4 provides lower quantization error than uniform INT4 for neural net weights.
- [ ] Compare PTQ (Post-Training Quantization) methods: AWQ vs GPTQ vs GGUF.
