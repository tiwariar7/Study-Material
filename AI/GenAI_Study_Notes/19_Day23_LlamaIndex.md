# Day 23: LlamaIndex — Data Connectors, Indexing Architectures & Advanced Query Engines

---

## Topics Covered
1. **LlamaIndex Philosophy & Core Abstractions**:
   - Documents vs Nodes (Metadata, Text, Relationships)
   - Data Connectors & LlamaHub
2. **Comprehensive Index Taxonomies**:
   - `VectorStoreIndex` (Semantic vector retrieval)
   - `SummaryIndex` (List-based complete summarization)
   - `TreeIndex` (Hierarchical summary trees)
   - `KeywordTableIndex` & `KnowledgeGraphIndex`
3. **Advanced Retrieval Paradigms in LlamaIndex**:
   - **Sentence-Window Retrieval** (`MetadataReplacementPostProcessor`)
   - **Auto-Merging Retriever** (Hierarchical leaf-to-parent merging)
   - **Router Query Engine** (Semantic selector between Vector DB, SQL, and Summary engines)
4. **Response Synthesizer Modes**:
   - `compact`, `refine`, `tree_summarize`, `accumulate`
5. **Architectural Comparison: LlamaIndex vs LangChain**
6. **Complete Python Implementation: Sentence-Window Retrieval Pipeline**
7. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. CORE PHILOSOPHY & ABSTRACTIONS

While LangChain is an agent orchestration and general chaining framework, **LlamaIndex** is an specialized **data framework** engineered specifically for ingestion, structuring, indexing, and querying private enterprise data.

```
                         LLAMAINDEX DATA LIFECYCLE
  [Raw Data: PDFs, SQL, APIs]
               |
               ↓
     [Data Connectors / Readers]  ---> Ingests into `Document` objects
               |
               ↓
     [Node Parsers / Splitters]   ---> Splits documents into `Node` objects
               |                        (Preserves metadata & sibling links)
               ↓
         [Index Building]         ---> Creates VectorStoreIndex, SummaryIndex, etc.
               |
               ↓
    [Query Engine / Retriever]    ---> Node Postprocessing -> Response Synthesis
```

### 1.1 Document vs Node
- **`Document`:** A container around any data source (e.g., an entire 50-page PDF, an entire Notion page, or a complete database record).
- **`Node`:** The atomic unit of data in LlamaIndex (analogous to a text chunk). Crucially, a Node contains:
  - Text payload.
  - User-defined metadata (e.g., page number, author, creation timestamp).
  - **Explicit Graph Relationships:** `NodeRelationship.PREVIOUS`, `NodeRelationship.NEXT`, `NodeRelationship.PARENT`, `NodeRelationship.CHILD`.

---

## 2. INDEX TAXONOMIES & DATA STRUCTURES

```
                            LLAMAINDEX INDEX TYPES
  +-------------------------+-------------------------+-------------------------+
  | VectorStoreIndex        | SummaryIndex (List)     | TreeIndex               |
  | Embeds nodes into       | Retains flat sequence.  | Builds bottom-up tree   |
  | vector space. Best for  | Traverses ALL nodes to  | of summaries. Ideal for |
  | specific Q&A retrieval. | generate full summaries.| hierarchical docs.      |
  +-------------------------+-------------------------+-------------------------+
```

| Index Type | Internal Data Structure | Best Use Case | Query Mechanism |
| :--- | :--- | :--- | :--- |
| **`VectorStoreIndex`** | Vector embeddings in ANN index | Specific question answering | Cosine / Dot product top-$k$ |
| **`SummaryIndex`** | Linear sequential list of nodes | "Summarize this entire report" | Iterates through all nodes, accumulating answers |
| **`TreeIndex`** | Directed tree where parent nodes summarize child nodes | High-level conceptual questions across massive corpora | Traverses root down to leaf, or bottom-up summarization |
| **`KeywordTableIndex`** | Inverted index mapping keywords to node IDs | Queries containing specific domain acronyms/names | Exact keyword lookup |

---

## 3. ADVANCED RETRIEVAL PATTERNS

### 3.1 Sentence-Window Retrieval
One of LlamaIndex's most powerful native patterns for solving the context-vs-retrieval trade-off:

```
  Document Text: "... Sentence A. [ Target Sentence B ] Sentence C. ..."
                                        |
  1. Embed only [ Target Sentence B ]  ---> Sharp, highly focused embedding vector!
  2. Index in Vector Store with Metadata: {"window": "Sentence A + B + C"}
  3. Query matches Sentence B vector.
  4. MetadataReplacementPostProcessor replaces Sentence B with the 
     full 3-sentence window before sending to LLM!
```
- **Result:** Maximum vector retrieval precision with zero context starvation during answer synthesis.

### 3.2 Auto-Merging Retriever (Hierarchical Merging)
- Ingests text into a hierarchy: Big Parent Nodes (e.g., 2048 tokens) $\to$ Middle Child Nodes (512 tokens) $\to$ Small Leaf Nodes (128 tokens).
- Searches only the small Leaf Nodes.
- If more than a threshold percentage (e.g., 60%) of child leaf nodes belonging to the same parent are retrieved, the retriever **automatically merges them and passes the clean Parent Node instead**!
- Prevents fragmented context from entering the prompt.

---

## 4. RESPONSE SYNTHESIS MODES

When multiple context nodes are retrieved, how should the LLM formulate the final answer? LlamaIndex provides 4 primary strategies:

```
       SYNTHESIS MODES
  1. COMPACT (Default)           2. REFINE                      3. TREE_SUMMARIZE
  +----------------------+      +----------------------+       +----------------------+
  | Stuff max chunks into|      | Node 1 -> Initial Ans|       |  [Final Summary]     |
  | prompt; if overflow, |      |          ↓           |       |      ^        ^      |
  | fallback to refine.  |      | Node 2 -> Updated Ans|       | [Sub-A]    [Sub-B]   |
  | (Fastest & cheapest) |      | (Iterative refinement|       |   ^  ^       ^  ^    |
  +----------------------+      |  over every chunk)   |       | Node 1,2   Node 3,4  |
                                +----------------------+       +----------------------+
```

- **`compact` (Default):** Maximally packs chunks into the context window up to the token limit, minimizing overall LLM API calls.
- **`refine`:** Answers based on Node 1; feeds answer + Node 2 into LLM to refine answer; continues sequentially. Highly thorough, but high latency ($O(K)$ sequential calls).
- **`tree_summarize`:** Recursively pairs chunks and summarizes them in a bottom-up hierarchical tree until reaching a final unified answer.
- **`accumulate`:** Queries each chunk independently in parallel, returning a list of responses for each chunk.

---

## 5. COMPARISON: LLAMAINdex VS LANGCHAIN

| Aspect | LlamaIndex | LangChain |
| :--- | :--- | :--- |
| **Primary Domain** | **Data retrieval, indexing, & RAG** | **Agent orchestration, tool use, & workflows** |
| **Data Ingestion** | Advanced readers (LlamaHub), document graph structures | Basic loaders |
| **Retrieval Tooling**| Hierarchical, Auto-merging, Sentence-window, Hybrid | Basic vector store retrieval |
| **Agent Paradigm** | Data Agents (Agents over query engines) | LangGraph (Cyclic multi-agent state machines) |
| **Best Production Architecture** | **Use LlamaIndex for RAG & Retrieval; use LangGraph for Agent Orchestration!** |

---

## 6. COMPLETE CODE: SENTENCE-WINDOW RETRIEVAL PIPELINE

```python
from llama_index.core import Document, VectorStoreIndex, ServiceContext
from llama_index.core.node_parser import SentenceWindowNodeParser
from llama_index.core.indices.postprocessor import MetadataReplacementPostProcessor
from llama_index.core.indices.postprocessor import SentenceTransformerRerank

# 1. Sample Document
doc_text = """
The Apollo program was the third United States human spaceflight program carried out by NASA.
It succeeded in landing the first humans on the Moon in 1969.
Neil Armstrong was the first person to step onto the lunar surface on July 21, 1969.
Buzz Aldrin joined him 19 minutes later.
They spent about two and a quarter hours together outside the spacecraft.
"""
documents = [Document(text=doc_text)]

# 2. Configure Sentence-Window Node Parser
# Embeds individual sentences, storing window_size surrounding sentences in metadata
node_parser = SentenceWindowNodeParser.from_defaults(
    window_size=2,
    window_metadata_key="window",
    original_text_metadata_key="original_sentence"
)
nodes = node_parser.get_nodes_from_documents(documents)

# 3. Build Vector Store Index over parsed Sentence Nodes
index = VectorStoreIndex(nodes)

# 4. Configure Query Engine with Window Replacement & Reranking
query_engine = index.as_query_engine(
    similarity_top_k=3,
    node_postprocessors=[
        # Replaces the matched sentence with its full 2-sentence surrounding window!
        MetadataReplacementPostProcessor(target_metadata_key="window"),
        # Reranks the expanded context
        SentenceTransformerRerank(
            top_n=2, 
            model="BAAI/bge-reranker-base"
        )
    ]
)

# 5. Query
response = query_engine.query("When did Neil Armstrong step on the moon?")
print(f"Answer: {response.response}")
```

---

## 7. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: What is the architectural difference between a Document and a Node in LlamaIndex?**
   - **A:** A Document represents an entire ingested data entity (e.g., a complete file, web page, or database table). A Node is an atomic chunk of that Document containing specific text segments, but unlike raw chunks, Nodes preserve rich relational metadata—including parent-child relationships, sequential previous/next links, and source document origins.

2. **Q: How does the `compact` response synthesizer mode optimize cost and latency?**
   - **A:** The `compact` mode takes all retrieved context nodes and packs as many of them as possible into a single prompt up to the model's maximum context limit. This ensures the maximum amount of evidence is processed in a single forward pass, reducing sequential API calls compared to iterative modes like `refine`.

### Intermediate
3. **Q: Explain how Sentence-Window Retrieval works and why it outperforms standard fixed-size chunking.**
   - **A:** In standard chunking, chunks of 300–500 tokens contain multiple mixed ideas, which blurs the vector embedding and hurts search precision. Sentence-Window Retrieval isolates individual sentences for embedding, creating sharp semantic vectors that match queries accurately. However, sending a single isolated sentence to an LLM leaves it without context. The postprocessor dynamically replaces the matched sentence with its surrounding $k$ sentences stored in metadata, delivering complete local context to the LLM without sacrificing embedding accuracy.

4. **Q: How does the Auto-Merging Retriever prevent context fragmentation?**
   - **A:** It establishes a hierarchical node tree (Parent $\to$ Children $\to$ Leaves). Vector search occurs on the small leaf nodes. If the query retrieves a significant majority of sibling leaf nodes belonging to the same parent, the retriever automatically discards the scattered leaf fragments and replaces them with the unified parent node, providing cohesive context without redundant boundaries.

### Advanced
5. **Q: How would you design a Router Query Engine in LlamaIndex to query structured SQL databases and unstructured technical manuals simultaneously?**
   - **A:** Create two distinct query engines:
     1. A `NLSQLTableQueryEngine` connected to the SQL database using text-to-SQL translation.
     2. A `VectorStoreIndex` query engine indexed over unstructured PDF technical manuals.
     Wrap both engines in a `RouterQueryEngine` using an LLM-based selector (`LLMSingleSelector`). When a query arrives, the selector examines the query intent: quantitative/aggregation queries (e.g., *"What was total revenue in March?"*) are routed to SQL, while conceptual/troubleshooting queries (e.g., *"How do I replace the hydraulic seal?"*) are routed to the Vector Store engine.

---

## 8. RAPID REVISION SUMMARY

- **Core Role:** LlamaIndex = Data framework for LLMs (Ingestion, Indexing, Retrieval).
- **Node:** Atomic unit of data with rich parent, child, and sibling graph links.
- **Sentence-Window Retrieval:** Embed individual sentences $\to$ replace with surrounding window before LLM synthesis.
- **Auto-Merging:** Leaf nodes search $\to$ merge siblings into parent node if threshold exceeded.
- **Response Synthesizer Modes:** `compact` (stuff and refine), `refine` (iterative sequential), `tree_summarize` (bottom-up tree).
- **Best Practice:** Combine LlamaIndex (data indexing & retrieval) with LangGraph (cyclic agent control).

---

## 9. RECALL CHECKLIST

- [ ] Define the relationship between `Document`, `Node`, and `Index` in LlamaIndex.
- [ ] Diagram the Sentence-Window Retrieval workflow.
- [ ] Explain how Auto-Merging Retriever decides when to promote children to a parent node.
- [ ] Contrast `compact` and `refine` response synthesizer modes.
- [ ] Describe the purpose of `MetadataReplacementPostProcessor`.
- [ ] Implement a basic LlamaIndex query engine in Python.
