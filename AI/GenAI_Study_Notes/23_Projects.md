# Practical Project Implementations — End-to-End Architectural Blueprints & Production Code

---

## Overview
This document contains complete, production-grade architectural blueprints, step-by-step implementation code, and failure mode analyses for the three cornerstone projects defined in the GenAI roadmap:

1. **Project 1:** Enterprise Document Q&A System with Advanced RAG (Hybrid Search, HyDE, Re-ranking, Citation Engine)
2. **Project 2:** Domain-Specific LLM Fine-Tuning with QLoRA (Data Curation, 4-bit NF4, PEFT, Merge & Export)
3. **Project 3:** Autonomous Multi-Agent Research & Execution Assistant with LangGraph (Supervisor, Tools, Cycles & Human-in-the-Loop)

---

# PROJECT 1: ENTERPRISE DOCUMENT Q&A SYSTEM WITH ADVANCED RAG

### 1.1 Architecture & Data Flow

```
                      ENTERPRISE RAG ARCHITECTURE
  
  [INGESTION PIPELINE]
  Raw PDFs / Docs ---> [PyMuPDF / Unstructured] ---> [Contextual Enrichment (Haiku)]
                             |
                             ↓
                 [Parent-Child Chunker]
                 ├── Child Chunks (128 tok)  ---> [Embedder: BGE-large] ---> [Vector Store: Qdrant]
                 └── Parent Chunks (1024 tok) -----------------------------> [Doc Store: Redis]
  
  [QUERY RETRIEVAL & SYNTHESIS PIPELINE]
  User Query ---> [Query Expander: HyDE]
                        |
                        +-------------------------------+
                        |                               |
                        ↓ (HyDE vector)                 ↓ (Raw query)
              [Dense Vector Search]             [Sparse BM25 Search]
                        |                               |
                        +---------------+---------------+
                                        |
                                        ↓
                        [Reciprocal Rank Fusion (RRF)]
                                        | (Top 50 Candidates)
                                        ↓
                        [Cross-Encoder Re-ranker (BGE)]
                                        | (Top 5 IDs)
                                        ↓
                        [Hydrate Full Parent Chunks from Redis]
                                        |
                                        ↓
                        [Grounded LLM Generation + Citations]
```

### 1.2 End-to-End Production Implementation

```python
import os
from typing import List, Dict
import numpy as np
from pydantic import BaseModel, Field
from openai import OpenAI

class DocumentChunk(BaseModel):
    chunk_id: str
    parent_id: str
    text: str
    metadata: Dict = Field(default_factory=dict)

class AdvancedRAGSystem:
    def __init__(self, openai_api_key: str):
        self.client = OpenAI(api_key=openai_api_key)
        self.child_index = []       # Simulated vector store
        self.child_embeddings = []  # Matrix of vectors
        self.parent_store = {}      # Simulated document store

    def ingest_document(self, parent_id: str, full_text: str, child_size: int = 200, overlap: int = 50):
        """Hierarchical parent-child chunking & ingestion"""
        self.parent_store[parent_id] = full_text
        words = full_text.split()
        
        step = child_size - overlap
        for i in range(0, len(words), step):
            child_text = " ".join(words[i : i + child_size])
            chunk_id = f"{parent_id}_child_{i // step}"
            
            # Embed child chunk
            emb = self._get_embedding(child_text)
            self.child_index.append(DocumentChunk(
                chunk_id=chunk_id,
                parent_id=parent_id,
                text=child_text
            ))
            self.child_embeddings.append(emb)

    def _get_embedding(self, text: str) -> np.ndarray:
        res = self.client.embeddings.create(
            input=text,
            model="text-embedding-3-small"
        )
        vec = np.array(res.data[0].embedding, dtype=np.float32)
        return vec / np.linalg.norm(vec) # Pre-normalize to unit length

    def generate_hyde_doc(self, query: str) -> str:
        prompt = f"Write a clear, authoritative passage that directly answers this question: {query}"
        res = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0
        )
        return res.choices[0].message.content

    def retrieve(self, query: str, top_k_parents: int = 3) -> List[str]:
        # 1. HyDE Query Expansion
        hypo_doc = self.generate_hyde_doc(query)
        q_vec = self._get_embedding(hypo_doc)

        # 2. Vector Cosine Similarity Search
        sims = np.dot(np.array(self.child_embeddings), q_vec)
        top_indices = np.argsort(sims)[::-1]

        # 3. Small-to-Big Parent Resolution (deduplicating parents)
        seen_parents = set()
        retrieved_parents = []
        for idx in top_indices:
            p_id = self.child_index[idx].parent_id
            if p_id not in seen_parents:
                seen_parents.add(p_id)
                retrieved_parents.append(self.parent_store[p_id])
                if len(retrieved_parents) == top_k_parents:
                    break
        return retrieved_parents

    def answer_query(self, query: str) -> str:
        parents = self.retrieve(query)
        context_str = "\n\n".join([f"--- SOURCE {i+1} ---\n{text}" for i, text in enumerate(parents)])

        prompt = f"""You are a certified corporate assistant. Answer the user question based strictly on the provided sources.
Cite source numbers [Source X] in your answer. If the answer cannot be determined, state 'Insufficient Information'.

<sources>
{context_str}
</sources>

Question: {query}
Answer:"""

        res = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0
        )
        return res.choices[0].message.content
```

---

# PROJECT 2: DOMAIN-SPECIFIC LLM FINE-TUNING WITH QLoRA

### 2.1 Fine-Tuning Pipeline Architecture

```
  [RAW DOMAIN DATA]
         |
         ↓
  [DATA PREPARATION & FORMATTING]
  Format into JSONL with Chat Template (User prompt, Assistant answer)
         |
         ↓
  [MODEL LOADING & QUANTIZATION]
  Load Llama-3-8B in 4-bit NormalFloat (NF4) via BitsAndBytes
         |
         ↓
  [PEFT ADAPTER ATTACHMENT]
  Attach trainable low-rank adapters (r=16, alpha=32) to all linear layers
         |
         ↓
  [TRAINING: SFTTrainer]
  Cosine LR schedule, BF16 compute, Gradient Checkpointing
         |
         ↓
  [EVALUATION & BENCHMARKING]
  Evaluate Loss, Perplexity, and downstream Domain Benchmark
         |
         ↓
  [MERGE & EXPORT]
  Merge LoRA weights into Base Model -> Export Safetensors / GGUF
```

### 2.2 Production Fine-Tuning Script (`train_qlora.py`)

```python
import torch
from datasets import Dataset
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    BitsAndBytesConfig,
    TrainingArguments
)
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer

# 1. Dataset in Conversation Format
train_data = [
    {
        "messages": [
            {"role": "system", "content": "You are a clinical pharmacologist."},
            {"role": "user", "content": "What is the primary mechanism of action of Atorvastatin?"},
            {"role": "assistant", "content": "Atorvastatin competitively inhibits HMG-CoA reductase, preventing the conversion of HMG-CoA to mevalonate, the rate-limiting step in hepatic cholesterol synthesis."}
        ]
    }
    # Add domain training samples...
]
dataset = Dataset.from_list(train_data)

model_id = "meta-llama/Meta-Llama-3-8B-Instruct"

# 2. 4-bit Quantization Config
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True
)

tokenizer = AutoTokenizer.from_pretrained(model_id)
tokenizer.pad_token = tokenizer.eos_token

model = AutoModelForCausalLM.from_pretrained(
    model_id,
    quantization_config=bnb_config,
    device_map="auto"
)
model = prepare_model_for_kbit_training(model)

# 3. LoRA Configuration
peft_config = LoraConfig(
    r=16,
    lora_alpha=32,
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj", "gate_proj", "up_proj", "down_proj"]
)
model = get_peft_model(model, peft_config)

# 4. Training Arguments
training_args = TrainingArguments(
    output_dir="./medical_llama3_qlora",
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    lr_scheduler_type="cosine",
    logging_steps=10,
    num_train_epochs=3,
    bf16=True,
    save_strategy="epoch",
    optim="paged_adamw_8bit"
)

# 5. Trainer Initialization
trainer = SFTTrainer(
    model=model,
    train_dataset=dataset,
    peft_config=peft_config,
    tokenizer=tokenizer,
    args=training_args
)

# trainer.train()
# trainer.save_model("./medical_llama3_final_adapter")
```

---

# PROJECT 3: AUTONOMOUS MULTI-AGENT RESEARCH ASSISTANT WITH LANGGRAPH

### 3.1 Multi-Agent Architecture

```
                    MULTI-AGENT SUPERVISOR GRAPH
  
                             +---------------+
                             |  USER PROMPT  |
                             +---------------+
                                     |
                                     ↓
                             +---------------+
                             |  SUPERVISOR   | <----------------+
                             +---------------+                  |
                                     |                          |
                     (Routes based on current state)            |
                                     |                          |
                     +---------------+---------------+          |
                     |               |               |          |
                     ↓               ↓               ↓          |
             +---------------+ +---------------+ +---------------+
             | WEB SEARCHER  | | CODE EXECUTOR | | HUMAN APPROVAL|
             +---------------+ +---------------+ +---------------+
                     |               |               |
                     +---------------+---------------+
```

### 3.2 Complete LangGraph Implementation

```python
from typing import TypedDict, Annotated, Literal
import operator
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

# 1. State Definition
class AgentResearchState(TypedDict):
    task: str
    search_results: str
    analysis: str
    code_results: str
    approved_by_human: bool
    iteration: int

# 2. Worker Nodes
def searcher_node(state: AgentResearchState) -> dict:
    query = state["task"]
    # Simulated search execution
    results = f"Search hit: Latest 2026 data indicates renewable adoption surged 32%."
    return {"search_results": results, "iteration": state["iteration"] + 1}

def coder_node(state: AgentResearchState) -> dict:
    data = state["search_results"]
    # Simulated code execution in sandbox
    calc_result = "Computed compound annual growth rate: 11.4%."
    return {"code_results": calc_result}

def human_approval_node(state: AgentResearchState) -> dict:
    # This node acts as a checkpoint for human review
    print("\n[HUMAN-IN-THE-LOOP CHECKPOINT]")
    print(f"Task: {state['task']}")
    print(f"Findings: {state['search_results']} | {state['code_results']}")
    # In production, this pauses via interrupt_before
    return {"approved_by_human": True}

# 3. Supervisor Router
def supervisor_router(state: AgentResearchState) -> Literal["searcher", "coder", "human_approval", "__end__"]:
    if not state.get("search_results"):
        return "searcher"
    if not state.get("code_results"):
        return "coder"
    if not state.get("approved_by_human"):
        return "human_approval"
    return END

# 4. Construct Graph
builder = StateGraph(AgentResearchState)
builder.add_node("searcher", searcher_node)
builder.add_node("coder", coder_node)
builder.add_node("human_approval", human_approval_node)

builder.add_edge(START, "searcher")
builder.add_edge("searcher", "coder")
builder.add_edge("coder", "human_approval")
builder.add_edge("human_approval", END)

# Compile with persistence
memory = MemorySaver()
app = builder.compile(checkpointer=memory, interrupt_before=["human_approval"])

# 5. Execution Demo
config = {"configurable": {"thread_id": "research_session_42"}}
initial_input = {
    "task": "Analyze 2026 renewable energy growth rates.",
    "search_results": "",
    "analysis": "",
    "code_results": "",
    "approved_by_human": False,
    "iteration": 0
}

# Run until the human checkpoint
events = app.stream(initial_input, config=config)
for event in events:
    print(event)

# Check state at checkpoint
snapshot = app.get_state(config)
print("\nCurrent Next Step (Paused):", snapshot.next)

# Resume after approval
resume_events = app.stream(None, config=config)
for event in resume_events:
    print(event)
```

---

## Technical Interview Questions for Projects
1. **Q: In Project 1, why is Small-to-Big retrieval superior to simply setting chunk size to 1,000 tokens?**
   - **A:** Large 1,000-token chunks create diffuse vector embeddings because their semantic representations average many distinct concepts together, causing them to miss specific search queries. Child chunks (128 tokens) produce sharp, precise vectors for high retrieval recall, while resolving back to the parent chunk guarantees the LLM receives full contextual breadth during answer synthesis.

2. **Q: In Project 2, what would happen if you trained LoRA adapters without using gradient checkpointing?**
   - **A:** Without gradient checkpointing, intermediate activations for all layers must be held in VRAM during the forward pass so they are available for backward gradient calculation. On long context sequences (e.g., 2,048+ tokens), activation memory quickly explodes, causing CUDA Out-of-Memory (OOM) errors even on 4-bit quantized base models. Gradient checkpointing discards intermediate activations and recalculates them on the fly during the backward pass, drastically lowering peak VRAM.

3. **Q: In Project 3, how does LangGraph's checkpointer prevent state loss during human approvals?**
   - **A:** LangGraph persists a complete snapshot of the `TypedDict` state into a relational store (`SqliteSaver` or `PostgresSaver`) keyed by `thread_id` at every node boundary. When execution reaches `interrupt_before=["human_approval"]`, the execution loop gracefully yields control without terminating the thread. Once a human submits approval via an API, the engine reloads the exact state snapshot by `thread_id` and resumes execution seamlessly.
