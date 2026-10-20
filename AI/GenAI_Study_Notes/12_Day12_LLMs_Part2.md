# Day 12: Large Language Models (Part 2) — Emergent Abilities, Alignment (SFT, RLHF, DPO) & Chat Templates

---

## Topics Covered
1. **Emergent Abilities in LLMs** (Phase transitions, In-Context Learning, Are they "mirages"?)
2. **Base Models vs Instruction / Chat Models**
3. **The Complete LLM Training Pipeline**:
   - Stage 1: **Pre-training** (CLM, FIM - Fill-In-the-Middle)
   - Stage 2: **Supervised Fine-Tuning (SFT)** (Instruction formatting, prompt-response pairs)
   - Stage 3: **Preference Alignment**:
     - **RLHF** (Reward Model via Bradley-Terry + PPO + KL-divergence penalty)
     - **DPO** (Direct Preference Optimization — derivation & mechanics)
     - **KTO** and **ORPO** overviews
4. **Chat Templates & Special Token Mechanics** (Jinja2, Llama 3 format, ChatML)
5. **Evaluation Benchmarks** (MMLU, GSM8K, HumanEval, MT-Bench, LMSYS Chatbot Arena)
6. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. EMERGENT ABILITIES IN LLMS

### 1.1 Definition
Wei et al. (2022) defined an **emergent ability** as an ability that is *not present in smaller models but is present in larger models*, appearing abruptly as model scale crosses a critical threshold.

```
Accuracy (%)
  100 |                                           * (Threshold reached!)
      |                                         *
      |                                       *
      |                                     *
   20 |-----------------------------------*
    0 +-------------------------------------------> Scale (Compute / Parameters)
         1B               10B            100B
```

Key emergent abilities include:
- Multi-step arithmetic & symbolic reasoning (GSM8K, MATH)
- In-Context Few-Shot Learning (adapting to new tasks purely via prompt context without weight updates)
- Chain-of-Thought (CoT) reasoning
- Instruction following and translation of low-resource languages

### 1.2 Are Emergent Abilities Real or "Mirages"? (Schaeffer et al., NeurIPS 2023)
- **The Finding:** Emergence is often an artifact of **discontinuous, non-linear evaluation metrics** (e.g., exact string match accuracy $0/1$) rather than fundamental changes in the model's internal representation.
- **Example:** If solving a 5-step problem requires all 5 steps to be correct:
  - If per-token probability increases linearly from $0.5 \to 0.7 \to 0.9$:
  - Exact match probability: $0.5^5 = 3.1\% \to 0.7^5 = 16.8\% \to 0.9^5 = 59.0\%$.
  - When plotted on a continuous metric (like Brier score or cross-entropy loss), improvements scale smoothly and predictably according to power laws.

---

## 2. BASE MODELS VS INSTRUCTION / CHAT MODELS

| Attribute | Base Model (e.g., `Llama-3-8B`) | Instruct/Chat Model (e.g., `Llama-3-8B-Instruct`) |
| :--- | :--- | :--- |
| **Primary Goal** | Minimize perplexity over internet text | Act as a helpful, harmless, and honest assistant |
| **Behavior** | Document completer (e.g., given a question, might generate another question or unrelated web text) | Instruction follower (directly answers question, follows formatting, admits ignorance) |
| **Safety / Guardrails**| Completely unfiltered; mimics web noise | Guardrailed against generating harmful, illegal, or toxic content |
| **Training Phase** | Pre-training only | Pre-training $\to$ SFT $\to$ Alignment (RLHF / DPO) |
| **Input Format** | Raw string continuation | Formatted dialogue with system/user/assistant roles |

---

## 3. THE COMPLETE THREE-STAGE TRAINING PIPELINE

```
+-------------------------------------------------------------------------+
| STAGE 1: PRE-TRAINING (Self-Supervised Next-Token Prediction)           |
| Data: Trillions of web tokens (Common Crawl, GitHub, Wikipedia, Books)  |
| Compute: ~98% of total training FLOPs                                   |
| Output: BASE MODEL                                                      |
+-------------------------------------------------------------------------+
                                    ↓
+-------------------------------------------------------------------------+
| STAGE 2: SUPERVISED FINE-TUNING / SFT (Instruction Tuning)              |
| Data: 10K - 1M high-quality prompt-response pairs                       |
| Compute: ~1-2% of total training compute                                |
| Output: SFT MODEL (Learns conversational syntax and tone)               |
+-------------------------------------------------------------------------+
                                    ↓
+-------------------------------------------------------------------------+
| STAGE 3: PREFERENCE ALIGNMENT (RLHF or DPO)                             |
| Data: Pairwise human/AI preferences (Prompt x, Winner y_w, Loser y_l)   |
| Compute: <1% of total training compute                                  |
| Output: ALIGNED INSTRUCT MODEL (Optimized for helpfulness & safety)     |
+-------------------------------------------------------------------------+
```

### 3.1 Pre-training Techniques: Fill-in-the-Middle (FIM)
Standard CLM only predicts tokens from left to right. FIM (Bavarian et al., OpenAI 2022) enables models (especially code models) to infill missing blocks inside existing code.
- Transformation: A document $D = [Prefix, Middle, Suffix]$ is rearranged into:
$$\langle\text{PRE}\rangle Prefix \langle\text{SUF}\rangle Suffix \langle\text{MID}\rangle Middle \langle\text{EOT}\rangle$$
- Enables code autocompletion tools to condition on code written *below* the cursor.

---

## 4. PREFERENCE ALIGNMENT: RLHF VS DPO

### 4.1 Reinforcement Learning from Human Feedback (RLHF)

RLHF aligns the model with human preferences using three sub-steps:

```
[Prompt x] ---> [Policy Model π_θ] ---> [Generated Response y]
                      ↑                         ↓
                 PPO Update               [Reward Model r_ψ]
                      |                         ↓
             [KL Penalty vs π_ref] <----+  [Scalar Reward R]
```

#### Step 1: Train Reward Model ($r_\psi(x, y)$)
Using pair comparison data ($x, y_w, y_l$) where $y_w$ is preferred over $y_l$:
$$\mathcal{L}_{RM}(\psi) = -\mathbb{E}_{(x, y_w, y_l)}\left[\log \sigma\left(r_\psi(x, y_w) - r_\psi(x, y_l)\right)\right]$$
Based on the **Bradley-Terry preference model**: $P(y_w \succ y_l \mid x) = \sigma(r(x, y_w) - r(x, y_l))$.

#### Step 2: Policy Optimization with PPO & KL Penalty
Optimize the policy $\pi_\theta$ to maximize the reward while staying close to the reference SFT policy $\pi_{ref}$:
$$\max_\theta \mathbb{E}_{x \sim \mathcal{D}, y \sim \pi_\theta(x)}\left[r_\psi(x, y) - \beta \mathbb{D}_{KL}\left(\pi_\theta(y \mid x) \,\|\, \pi_{ref}(y \mid x)\right)\right]$$
Where:
- $\beta$: Hyperparameter controlling the strength of the KL penalty.
- **Why KL Penalty?** Prevents **reward hacking** (where the policy exploits vulnerabilities in the imperfect reward model, producing repetitive or adversarial outputs with high reward scores but degraded human utility).

---

### 4.2 Direct Preference Optimization (DPO - Rafailov et al., 2023)

RLHF with PPO is notoriously unstable, complex, and memory-intensive (requiring 4 concurrent models in VRAM: Actor, Critic/Value, Reference, and Reward).

**DPO Mathematical Breakthrough:**
Rafailov et al. proved analytically that the optimal policy under the KL-constrained RLHF objective satisfies:
$$r(x, y) = \beta \log \frac{\pi_\theta(y \mid x)}{\pi_{ref}(y \mid x)} + \beta \log Z(x)$$
By substituting this exact relation directly into the Bradley-Terry reward loss, the partition function $Z(x)$ cancels out, allowing optimization of the policy $\pi_\theta$ directly on preference pairs **without any explicit reward model or reinforcement learning**:

$$\mathcal{L}_{DPO}(\theta) = -\mathbb{E}_{(x, y_w, y_l)}\left[\log \sigma\left(\beta \log \frac{\pi_\theta(y_w \mid x)}{\pi_{ref}(y_w \mid x)} - \beta \log \frac{\pi_\theta(y_l \mid x)}{\pi_{ref}(y_l \mid x)}\right)\right]$$

#### Comparison: RLHF vs DPO

| Feature | RLHF (PPO) | DPO (Direct Preference Optimization) |
| :--- | :--- | :--- |
| **Models in Memory** | 4 (Policy, Reference, Reward, Critic) | 2 (Active Policy, Frozen Reference) |
| **Stability** | Sensitive to hyperparameters, PPO clipping, value loss | Highly stable (standard binary cross-entropy loss) |
| **Sampling during training**| Must sample new generations on each step | Offline training on fixed preference pairs |
| **Computational Cost** | High ($3\times$ to $5\times$ SFT cost) | Low ($\approx 1.2\times$ SFT cost) |

---

## 5. CHAT TEMPLATES & SPECIAL TOKENS

To distinguish multi-turn dialogues, models use structured **special tokens**.

### 5.1 Llama 3 Chat Format
```
<|begin_of_text|>
<|start_header_id|>system<|end_header_id|>
You are a helpful assistant.<|eot_id|>
<|start_header_id|>user<|end_header_id|>
What is the capital of France?<|eot_id|>
<|start_header_id|>assistant<|end_header_id|>
The capital of France is Paris.<|eot_id|>
```

### 5.2 Why Special Tokens Are Critical
1. **Role Separation:** Prevents user inputs from spoofing system prompt instructions (prompt injection mitigation).
2. **Stop Tokens:** The model emits `<|eot_id|>` (End of Turn) to yield control back to the application; if not properly recognized, the model will hallucinate user replies indefinitely.

---

## 6. EVALUATION BENCHMARKS FOR MODERN LLMS

| Benchmark | Target Capability | Metric Format |
| :--- | :--- | :--- |
| **MMLU** (Massive Multitask Language Understanding) | General academic knowledge (57 domains: STEM, humanities, law) | 5-shot multiple choice accuracy |
| **GSM8K** (Grade School Math 8K) | Multi-step mathematical reasoning | Few-shot exact numeric match |
| **HumanEval** | Python coding capabilities | `pass@1` / `pass@10` unit test execution |
| **MT-Bench** | Multi-turn conversational quality & instruction following | LLM-as-a-Judge score (1 to 10) |
| **LMSYS Chatbot Arena** | Real-world blind human preference comparison | Bradley-Terry Elo rating from crowdsourced votes |

---

## 7. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: Why can't we use a Base Model directly in a conversational customer service bot?**
   - **A:** A base model is strictly trained for next-token prediction over internet text. When presented with `"User: How do I reset my password?"`, it might predict `"User: I forgot my username too"`, or append a legal disclaimer, or repeat the question. It has not been trained to adopt an assistant role or terminate generation after answering.

2. **Q: What is the purpose of the KL divergence penalty in RLHF?**
   - **A:** The learned reward model is an approximation of human preferences. If unconstrained, the policy network will discover degenerate token sequences that exploit quirks in the reward model to get maximum scores (reward hacking / mode collapse). The KL penalty $-\beta D_{KL}(\pi_\theta \| \pi_{ref})$ ensures the policy does not drift too far from the sensible, coherent SFT base model.

### Intermediate
3. **Q: How does DPO eliminate the need for a separate reward model?**
   - **A:** DPO uses the mathematical identity connecting the reward function to the optimal policy ratio: $r(x,y) = \beta \log(\pi_\theta(y|x)/\pi_{ref}(y|x)) + \beta \log Z(x)$. When this expression is substituted into the Bradley-Terry preference likelihood equation, the partition function $Z(x)$ cancels out entirely. The loss function directly compares the implicit reward of preferred vs dispreferred responses using only the policy and reference probabilities.

4. **Q: What is the difference between SFT and Preference Alignment?**
   - **A:** SFT teaches the model *format, grammar, and capability* (how to speak like an assistant using supervised $(x, y)$ examples). Preference alignment teaches the model *judgment and values* (choosing between multiple valid responses, optimizing nuance, tone, safety, and helpfulness based on contrastive $(y_w, y_l)$ feedback).

### Advanced
5. **Q: What are the main limitations of DPO compared to PPO-based RLHF?**
   - **A:**
     1. **Offline Nature:** DPO trains on static, offline datasets. As the model updates, it cannot explore novel response distributions or self-correct via on-policy rollouts, potentially causing distribution shift.
     2. **Overfitting to Preference Noise:** If the dataset contains contradictory human ratings, DPO can overfit and degrade likelihood on standard validation corpora.
     3. **Lack of Dynamic Feedback:** Online RLHF allows the policy to generate new completions that the reward model continuously scores, leading to stronger reasoning improvements in models like OpenAI o1.

---

## 8. RAPID REVISION SUMMARY

- **Emergent Abilities:** Performance jumps at scale; partly real (phase shifts in multi-step reasoning), partly metric artifact (stepwise thresholding).
- **Pipeline:** Pre-training ($\approx 98\%$ compute) $\to$ SFT (style/instruction) $\to$ Alignment (RLHF / DPO).
- **Bradley-Terry Model:** $P(y_w \succ y_l) = \sigma(r(y_w) - r(y_l))$.
- **DPO Objective:** Directly increases log-prob of $y_w$ relative to $\pi_{ref}$ while decreasing log-prob of $y_l$ without reward model or PPO.
- **Chat Templates:** Jinja2 templates mapping JSON message lists (`role`, `content`) into strings delimited by unique special tokens.
- **Gold Standard Benchmarks:** MMLU (Knowledge), GSM8K (Math), HumanEval (Code), LMSYS Arena (Human Elo).

---

## 9. RECALL CHECKLIST

- [ ] Define emergent abilities and cite why non-linear metrics can exaggerate emergence.
- [ ] Diagram the three stages of LLM training and state the purpose of each.
- [ ] Write the Bradley-Terry preference loss equation used to train Reward Models.
- [ ] Explain why the KL penalty is necessary during PPO policy optimization.
- [ ] Write the DPO loss function from memory.
- [ ] Explain why DPO requires only 2 models in memory whereas PPO requires 4.
- [ ] Explain Fill-In-the-Middle (FIM) and its usage in code models.
- [ ] Structure a multi-turn chat prompt using Llama 3 special tokens.
