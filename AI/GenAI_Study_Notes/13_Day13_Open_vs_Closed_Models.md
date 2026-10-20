# Day 13: Open-Source vs Closed-Source Models — Llama, Mistral, GPT-4 & Mixture-of-Experts (MoE)

---

## Topics Covered
1. **Open-Weight vs Closed-Source (Proprietary) Paradigms**
2. **Deep-Dive on Major Frontier Models**:
   - Closed-Source: GPT-4/4o, Claude 3/3.5, Gemini 1.5 Pro/Flash
   - Open-Weight: Llama 3/3.1/3.2, Mistral/Mixtral, Qwen 2.5, DeepSeek V2/V3
3. **Mixture-of-Experts (MoE) Architecture Deep-Dive**:
   - Sparse vs Dense models
   - Top-$k$ Router Gating mechanism & routing formulations
   - Expert starvation & auxiliary load-balancing loss
   - Memory vs Compute footprint (Total vs Active parameters)
4. **Multi-Head Latent Attention (MLA)** overview (DeepSeek innovation)
5. **Trade-Off Matrix: Latency, Privacy, Control, Hardware & Cost**
6. **Self-Hosting Frameworks: vLLM, Ollama, TensorRT-LLM, TGI**
7. **Licensing Landscapes (Apache 2.0, MIT, Llama Community License)**
8. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. OPEN-WEIGHT VS CLOSED-SOURCE PARADIGMS

```
               THE SPECTRUM OF LLM ACCESSIBILITY
  Proprietary (Closed)           Open-Weight                  Fully Open-Source
   (API-Only Access)         (Weights + Code)          (Weights + Code + Data)
           |                         |                            |
  OpenAI GPT-4o / o1           Meta Llama 3.1                   OLMo (AI2)
  Anthropic Claude 3.5         Mistral / Mixtral            Pythia (EleutherAI)
  Google Gemini 1.5 Pro        Qwen 2.5, DeepSeek V3         BLOOM (BigScience)
```

| Dimension | Proprietary Models (e.g., GPT-4o, Claude 3.5) | Open-Weight Models (e.g., Llama 3.1 70B, Qwen 2.5) |
| :--- | :--- | :--- |
| **Access Model** | Black-box REST API | Direct model weights (safetensors / GGUF) |
| **Data Privacy** | Data sent over wire to vendor; relies on compliance agreements | 100% on-premise execution; zero data leakage risk |
| **Customizability** | Prompting, fine-tuning API (limited), system prompts | Full fine-tuning (LoRA, full weights), custom heads, pruning |
| **Cost Dynamics** | Variable OPEX (pay per input/output token) | Fixed CAPEX/OPEX (GPU server lease/purchase, vLLM cluster) |
| **Reproducibility** | Zero guarantee (vendors update models silently) | 100% deterministic & frozen forever |
| **Vendor Lock-in** | High | None (weights can be run across any cloud or local rig) |

---

## 2. MIXTURE-OF-EXPERTS (MoE) ARCHITECTURE

### 2.1 Dense vs Sparse Neural Networks
- **Dense Model:** Every input token activates 100% of the model's parameters in every layer (e.g., Llama 3 70B uses all 70B weights for every token).
- **Sparse MoE Model:** Replaces standard feed-forward networks (FFN) with multiple parallel "expert" FFNs. A lightweight **router network** dynamically routes each token to only the top-$k$ experts.

```
                    INPUT TOKEN x
                          |
             +------------+------------+
             |                         |
             ↓                         ↓
     [ROUTER GATING]        [Top-k Selection (e.g., k=2)]
             |                         |
    +--------+--------+                |
    | Scores:         |                |
    | Exp 1: 0.05     |                |
    | Exp 2: 0.65 --->|----------------+--> Routes x to Expert 2
    | Exp 3: 0.02     |                |
    | Exp 4: 0.28 --->|----------------+--> Routes x to Expert 4
    +-----------------+
             |
             +-----------------------+
             |                       |
             ↓                       ↓
      [Expert 2 (FFN)]        [Expert 4 (FFN)]
             |                       |
             \                       /
              +----------+----------+
                         |
           Weighted Sum: 0.65*E2(x) + 0.28*E4(x)
                         ↓
                    OUTPUT TOKEN
```

### 2.2 Mathematical Formulation
For $E$ experts $\{FFN_1, FFN_2, \dots, FFN_E\}$, the router produces gating logits via a learned weight matrix $W_g$:
$$H(x) = x \cdot W_g$$
Top-$k$ gating selects the top $k$ values and sets all others to $-\infty$:
$$\text{KeepTopK}(v, k)_i = \begin{cases} v_i & \text{if } v_i \text{ is in top } k \\ -\infty & \text{otherwise} \end{cases}$$
The final routing weights are obtained via Softmax:
$$G(x) = \text{Softmax}(\text{KeepTopK}(H(x), k))$$
The output is the weighted sum of only the active experts:
$$y = \sum_{i \in \text{TopK}} G(x)_i \cdot FFN_i(x)$$

### 2.3 Total Parameters vs Active Parameters
**Case Study: Mixtral 8x7B**
- Has 8 expert FFNs per layer, selecting $k = 2$ experts per token.
- Self-attention weights and layer norms are **shared** across all experts.
- Total parameters = $\approx 46.7\text{B}$
- Active parameters per token = $\approx 12.9\text{B}$
- **Takeaway:** Offers the reasoning capacity and knowledge capacity of a ~47B model while running at the inference latency and compute cost of a 13B model!

### 2.4 Load Balancing & Auxiliary Loss
Without intervention, the router tends to suffer from **expert collapse** (routing all tokens to the same 1 or 2 popular experts, leaving the remaining experts completely untrained).
To enforce uniform distribution across all experts, an auxiliary loss $\mathcal{L}_{aux}$ is added to the training objective:
$$\mathcal{L}_{aux} = \alpha \cdot E \sum_{i=1}^E f_i \cdot P_i$$
Where:
- $f_i$ is the fraction of tokens routed to expert $i$: $f_i = \frac{1}{T} \sum_{t=1}^T \mathbb{I}(\text{token } t \text{ selects expert } i)$
- $P_i$ is the average routing probability assigned to expert $i$: $P_i = \frac{1}{T} \sum_{t=1}^T G(x_t)_i$
- $\alpha$ is a balancing coefficient (typically 0.01).
- The loss is minimized when tokens are evenly distributed ($f_i = 1/E$ and $P_i = 1/E$).

---

## 3. FRONTIER MODEL ECOSYSTEM BREAKDOWN

```
                        FRONTIER MODEL LANDSCAPE
    +-------------------------------------------------------------+
    | PROPRIETARY                                                 |
    | • OpenAI: GPT-4o (multimodal), o1 (test-time reasoning)     |
    | • Anthropic: Claude 3.5 Sonnet (coding, writing, vision)    |
    | • Google: Gemini 1.5 Pro (2M token context, native multi)   |
    +-------------------------------------------------------------+
                                  |
    +-------------------------------------------------------------+
    | OPEN-WEIGHT DENSE                                           |
    | • Meta: Llama 3.1 (8B, 70B, 405B - trained on 15T tokens)   |
    | • Alibaba: Qwen 2.5 (0.5B to 72B - premier coding & math)   |
    | • Google: Gemma 2 (9B, 27B)                                 |
    +-------------------------------------------------------------+
                                  |
    +-------------------------------------------------------------+
    | OPEN-WEIGHT MoE                                             |
    | • Mistral AI: Mixtral 8x7B, Mixtral 8x22B                   |
    | • DeepSeek: DeepSeek-V3 (671B MoE, 37B active, MLA)         |
    +-------------------------------------------------------------+
```

### DeepSeek Architectural Highlight: Multi-Head Latent Attention (MLA)
Standard MHA/GQA stores uncompressed KV matrices in memory.
DeepSeek compresses Key and Value vectors into a low-dimensional latent space ($c^{KV}$) during inference:
$$c^{KV} = X W_{DKV}$$
Only the compressed latent vector $c^{KV}$ is retained in the KV cache, slashing KV cache memory consumption by **$93\%$** compared to standard MHA without degrading retrieval fidelity!

---

## 4. SELF-HOSTING OPEN-WEIGHT INFERENCE ENGINES

When deploying open-weight models in production, standard PyTorch (`model.generate()`) is up to $20\times$ slower than optimized serving engines.

| Serving Engine | Primary Advantage | Best Use Case |
| :--- | :--- | :--- |
| **vLLM** | **PagedAttention** (eliminates memory fragmentation in KV cache); continuous batching. | Production REST APIs handling concurrent requests |
| **Ollama** | Single binary installer; native GGUF support with automatic CPU/GPU offloading. | Local development, macOS, laptops, offline desktops |
| **TensorRT-LLM** | NVIDIA hardware-specific kernel fusion and FP8/FP4 optimization. | Enterprise high-throughput dedicated NVIDIA clusters |
| **TGI (Text Generation Inference)** | Hugging Face ecosystem integration, built-in Prometheus metrics, token streaming. | Kubernetes deployments on cloud providers |

### What is PagedAttention (vLLM)?
Traditional KV caches require contiguous virtual memory allocations for each sequence's maximum possible length ($S_{max}$). This wastes 60–80% of VRAM due to internal and external memory fragmentation.
PagedAttention treats GPU VRAM like the virtual memory manager of an Operating System:
- Divides KV cache into fixed-size **pages** (e.g., 16 tokens per block).
- Pages can be located anywhere in physical VRAM and dynamically allocated as tokens are generated.
- Allows sharing of KV cache pages between parallel generations (e.g., parallel sampling or beam search).

---

## 5. LICENSING & COMPLIANCE

```
     PERMISSIVE                   RESTRICTED COMMERCIAL              CLOSED
     (Apache 2.0 / MIT)            (Community Licenses)           (Proprietary)
          |                                  |                          |
   Mistral 7B, Qwen 2.5             Meta Llama 3.1 License         OpenAI / Anthropic
          |                                  |                          |
   Use for any purpose            Free commercial use UNLESS       No access to weights.
   including training             MAU > 700 Million users;         Terms prohibit using
   competing models.              Cannot use outputs to train      outputs to train 
                                  competing foundation models.     competing models.
```

---

## 6. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: Why does an MoE model with 47B total parameters run faster than a dense 47B model?**
   - **A:** In an MoE model like Mixtral 8x7B, while all 47B parameters reside in GPU memory (VRAM), the router activates only $k=2$ experts per token. This means only $\approx 13\text{B}$ parameters perform mathematical FLOPs for any given token forward pass, resulting in the computational speed of a 13B dense model.

2. **Q: What is the main operational difference between paying for API tokens vs hosting Llama 3 locally?**
   - **A:** API models represent variable operating cost (OPEX)—zero fixed hardware, pay-per-token, zero infrastructure maintenance, but data leaves your firewall and vendor may deprecate endpoints. Local hosting requires fixed infrastructure (CAPEX/GPU leases), engineering overhead for scaling and fault tolerance, but provides fixed predictability, complete data privacy, and custom model control.

### Intermediate
3. **Q: What is "expert starvation" in MoE models and how is it mitigated during training?**
   - **A:** Expert starvation occurs when a few experts randomly receive slightly higher weights early in training, causing the router to send all tokens to them. These few experts get continuously updated while the remaining experts never receive gradients and remain untrained. It is mitigated by adding an auxiliary load-balancing loss that penalizes routers when token routing probabilities deviate from a uniform distribution across all experts.

4. **Q: How does vLLM's PagedAttention dramatically increase serving throughput?**
   - **A:** Without PagedAttention, GPU memory must pre-allocate contiguous chunks for each request up to maximum context length, wasting up to 80% of VRAM on ungenerated tokens and fragmented blocks. PagedAttention divides the KV cache into non-contiguous physical memory blocks (similar to OS paging). This brings memory waste down to under 4%, allowing systems to run $2\times$ to $4\times$ larger concurrent batch sizes on the exact same GPU hardware.

### Advanced
5. **Q: Under what circumstances should an enterprise choose open-weight over frontier proprietary models?**
   - **A:**
     1. **Strict Data Residency & Privacy:** Healthcare (HIPAA), Finance, or Defense where data cannot cross third-party boundaries.
     2. **Fine-Tuning on Domain Vocabulary:** When standard prompting is insufficient and deep domain alignment via LoRA/SFT is mandatory.
     3. **Deterministic Latency & SLA:** Eliminating third-party API rate limits and throttling.
     4. **Cost at Scale:** When processing billions of tokens monthly, where self-hosting on leased GPUs is substantially cheaper than paying $15-$30 per million proprietary tokens.

---

## 7. RAPID REVISION SUMMARY

- **MoE Mechanism:** Router network scores all $E$ experts $\to$ selects top-$k$ $\to$ weighted sum of expert FFNs.
- **Compute vs Memory:** MoE total weights dictate VRAM footprint; active weights dictate forward-pass compute latency.
- **Auxiliary Loss:** Prevents expert collapse by penalizing non-uniform token routing.
- **vLLM PagedAttention:** Breaks KV cache into non-contiguous blocks $\to$ eliminates VRAM fragmentation $\to$ massively boosts batch concurrency.
- **Licenses:** Apache 2.0 (fully open), Llama Community License (commercial allowed up to 700M monthly active users).

---

## 8. RECALL CHECKLIST

- [ ] Explain the difference between Dense and Sparse neural network architectures.
- [ ] Write down the Top-$k$ gating routing formula for an MoE layer.
- [ ] Calculate total vs active parameters for Mixtral 8x7B.
- [ ] Explain why the auxiliary load-balancing loss is necessary during MoE pre-training.
- [ ] Describe how PagedAttention solves KV-cache memory fragmentation.
- [ ] Compare vLLM, Ollama, and TensorRT-LLM use cases.
- [ ] State the commercial restriction in the Meta Llama 3 license.
