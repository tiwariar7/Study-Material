# Day 24–25: Knowledge Graphs & Graph RAG — Neo4j, Cypher & Microsoft GraphRAG

---

## Topics Covered
1. **The Fundamental Failure Modes of Pure Vector RAG**:
   - The Multi-Hop Reasoning Blindspot
   - The Global Aggregation / Dataset Summarization Failure
2. **Knowledge Graph (KG) Foundations**:
   - Triplet Representation: $(Subject, Predicate, Object)$
   - Property Graphs (Labeled Nodes, Directed Relationships, Key-Value Attributes)
   - Neo4j Graph Database & **Cypher Query Language** syntax
3. **Automated KG Construction with LLMs**:
   - Named Entity Recognition (NER) & Relation Extraction (RE)
   - Entity Resolution & Deduplication (e.g., "Apple Inc." vs "AAPL")
4. **Graph RAG Architectures**:
   - **Microsoft GraphRAG** (Leiden Community Detection, Hierarchical Summaries, Global vs Local Search)
   - Hybrid Vector + Graph RAG (Text-to-Cypher + Vector Similarity)
5. **Python Implementation: Text-to-Cypher RAG Pipeline with Neo4j**
6. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. WHY PURE VECTOR RAG FAILS AT COMPLEX REASONING

```
           QUERY: "Which vendors supply parts to both Apple and Boeing?"
  
  Vector RAG Approach:
  • Embeds query into a single vector.
  • Retrieves chunks discussing Apple vendors OR Boeing vendors.
  • Chunks rarely mention BOTH companies in the same 300-token window!
  • LLM misses the connection entirely (Multi-Hop Blindspot).

  Knowledge Graph Approach:
  • Finds (Apple) and (Boeing) nodes.
  • Traverses 2-hop edges: MATCH (v:Vendor)-[:SUPPLIES]->(c:Company)
  • Instantly returns the exact intersection of connected vendor nodes!
```

| Challenge | Pure Vector RAG | Graph RAG |
| :--- | :--- | :--- |
| **Multi-Hop Reasoning** | Fails (requires disparate chunks to magically co-locate in top-$k$) | Native (traverses explicit multi-step edges between entities) |
| **Global Sense-Making** | Cannot answer "What are the primary themes across all 5,000 files?" | Hierarchical graph communities summarize global themes |
| **Relational Accuracy** | Prone to confusing which subject performed which action | Explicit directed edges guarantee relationship veracity |
| **Deterministic Updates** | Updating an embedding requires re-indexing chunks | Simply add/modify a node or edge in Cypher |

---

## 2. KNOWLEDGE GRAPH FUNDAMENTALS & PROPERTY GRAPHS

### 2.1 The Labeled Property Graph (LPG) Model
A Property Graph consists of:
- **Nodes (Vertices):** Entities (e.g., `Person`, `Company`, `Drug`).
- **Labels:** Categories identifying node types (e.g., `:Person`).
- **Relationships (Edges):** Directed connections between nodes (e.g., `[:WORKS_AT]`, `[:TREATS]`).
- **Properties:** Arbitrary key-value pairs stored on *both* nodes and relationships (e.g., `since: 2021`, `dosage: "50mg"`).

```
   (p:Person {name: "Ada", role: "CEO"})
                     |
                     | [:FOUNDED {year: 2022}]
                     ↓
  (c:Company {name: "NeuroTech", valuation: "$50M"})
```

### 2.2 Cypher Query Language (Neo4j)
Cypher uses ASCII-art syntax to express graph traversal patterns:

```cypher
// 1. Find all people who work at companies in Berlin
MATCH (p:Person)-[:WORKS_AT]->(c:Company)-[:LOCATED_IN]->(city:City {name: "Berlin"})
RETURN p.name, c.name;

// 2. Multi-hop supplier query: vendors shared by Apple and Boeing
MATCH (v:Vendor)-[:SUPPLIES]->(c1:Company {name: "Apple"}),
      (v)-[:SUPPLIES]->(c2:Company {name: "Boeing"})
RETURN v.name;

// 3. Create a new relationship
MERGE (a:Person {name: "Alan Turing"})
MERGE (b:Field {name: "Computer Science"})
MERGE (a)-[:PIONEERED]->(b);
```

---

## 3. MICROSOFT GraphRAG (Edge et al., 2024)

Traditional RAG fails at answering broad global questions like: *"What are the top 5 geopolitical risks mentioned in this 10,000-page archive?"*
Microsoft's GraphRAG pipeline solves this using **Graph Community Detection**:

```
                       MICROSOFT GraphRAG PIPELINE
  [Raw Documents]
         |
         ↓
  [LLM Extraction: Entities & Directed Relationships]
         |
         ↓
  [Graph Clustering: Leiden Algorithm detects communities at multiple hierarchies]
         |
         ↓
  [Hierarchical Summarization: LLM generates comprehensive summaries for each community]
         |
         ↓
  [Global Query Answering: Maps query across community summaries in parallel -> Reducer]
```

### 3.1 The Leiden Algorithm & Community Hierarchies
- The graph of extracted entities is partitioned into hierarchical **communities** using the Leiden community detection algorithm (measuring modularity).
- Level 0: Small tightly connected clusters (e.g., a specific project team).
- Level 1: Medium functional divisions (e.g., an entire engineering department).
- Level 2: High-level organizational clusters (e.g., an entire corporation).
- An LLM pre-generates a rich **Community Summary Report** for each cluster at every level during ingestion.

### 3.2 Global Search vs Local Search in GraphRAG
- **Local Search:** Traverses immediate neighboring entities and raw text units around a specific query entity (best for: *"What did person X say about topic Y?"*).
- **Global Search:** Evaluates the question against all high-level Community Summaries in parallel, scoring and synthesizing them into a comprehensive global overview without needing individual chunks!

---

## 4. HYBRID VECTOR + GRAPH RAG (Text-to-Cypher)

```
[User Query: "Who manages the engineers working on Project Titan?"]
                              |
              +---------------+---------------+
              |                               |
              ↓ (LLM Text-to-Cypher)          ↓ (Dense Vector Search)
     Generate Cypher Query:              Retrieve Unstructured
     MATCH (p:Person)-[:MANAGES]->(e)    Project Titan Chunks
     WHERE e.project = 'Titan' ...                    |
              |                                       |
     Execute against Neo4j DB                         |
              |                                       |
              +---------------+---------------+-------+
                              |
                              ↓
              Synthesize Unified Response with LLM
```

---

## 5. COMPLETE PYTHON IMPLEMENTATION: TEXT-TO-CYPHER RAG

```python
import os
from neo4j import GraphDatabase
from openai import OpenAI

class Neo4jGraphRAG:
    def __init__(self, uri, user, password, openai_api_key):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))
        self.client = OpenAI(api_key=openai_api_key)
        self.schema = self._get_schema()

    def _get_schema(self) -> str:
        query = """
        CALL apoc.meta.schema() YIELD value
        RETURN value
        """
        # Alternatively, provide explicit string schema for prompt conditioning
        return "Node labels: (:Company {name}), (:Person {name, role}), (:Product {name})\n" \
               "Relationships: (:Person)-[:WORKS_AT]->(:Company), (:Company)-[:PRODUCES]->(:Product)"

    def text_to_cypher(self, user_question: str) -> str:
        prompt = f"""You are an expert Neo4j developer. Given the graph schema below, convert the user's natural language question into a valid Cypher query.
Return ONLY the raw Cypher query without backticks, markdown, or explanation.

Schema:
{self.schema}

Question: {user_question}
Cypher Query:"""

        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.0
        )
        return response.choices[0].message.content.strip()

    def query_graph(self, cypher_query: str):
        with self.driver.session() as session:
            result = session.run(cypher_query)
            return [record.data() for record in result]

    def answer_question(self, user_question: str) -> str:
        # 1. Translate question to Cypher
        cypher = self.text_to_cypher(user_question)
        print(f"Generated Cypher: {cypher}")

        # 2. Execute on Neo4j
        graph_data = self.query_graph(cypher)

        # 3. Synthesize natural language answer
        synth_prompt = f"""Answer the user's question based strictly on the graph database results.
Question: {user_question}
Graph Results: {graph_data}
Answer:"""

        res = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": synth_prompt}],
            temperature=0.0
        )
        return res.choices[0].message.content
```

---

## 6. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: What is a knowledge graph triplet and how does it relate to Property Graphs?**
   - **A:** A triplet is the foundational semantic statement composed of $(Subject, Predicate, Object)$ representing a directed relationship between two entities (e.g., `(Paris, is_capital_of, France)`). A Labeled Property Graph (LPG) extends this model by allowing nodes and directed relationships to hold labels and arbitrary key-value attribute maps (properties), providing high expressive capability for enterprise data.

2. **Q: Write a basic Cypher query to retrieve all employees in an "Engineering" department.**
   - **A:**
     ```cypher
     MATCH (e:Employee)-[:MEMBER_OF]->(d:Department {name: "Engineering"})
     RETURN e.name, e.title;
     ```

### Intermediate
3. **Q: Why does traditional Vector RAG struggle with multi-hop questions like "Who was the doctoral advisor of the inventor of the Transformer?"**
   - **A:** Vector similarity searches for text chunks that semantically match the entire query string. Information about the inventor of the Transformer is likely in one paper/article, while information about that inventor's university doctoral advisor is in an entirely separate document. Vector search rarely retrieves both unrelated documents in the same top-$k$ window. A Knowledge Graph connects `(Transformer)-[:INVENTED_BY]->(Person)-[:ADVISED_BY]->(Advisor)` through deterministic edge traversals across hops.

4. **Q: How does Microsoft GraphRAG perform Global Dataset Summarization?**
   - **A:** During ingestion, GraphRAG extracts entities and relations, applies the Leiden clustering algorithm to group nodes into hierarchical modular communities, and uses an LLM to generate summary reports for each community. When answering global queries, GraphRAG evaluates the question against these community summaries in parallel, aggregates the intermediate ratings, and synthesizes a comprehensive holistic answer without searching raw text chunks.

### Advanced
5. **Q: What is the primary operational failure point of Text-to-Cypher in production, and how do you safeguard against it?**
   - **A:**
     1. **Failure Modes:** LLMs hallucinating non-existent node labels, relationship types, or properties; generating computationally catastrophic queries (unbounded Cartesian products `MATCH (a), (b)` that freeze the database); or executing malicious write/delete mutations (`DELETE`, `DROP`).
     2. **Production Safeguards:**
        - **Read-Only Database Roles:** Execute LLM queries strictly under read-only database credentials.
        - **Cypher AST Validation:** Parse the generated query with an AST validator (like `cypher-parser`) before execution to reject write operations and enforce timeouts/query limits (`LIMIT 50`).
        - **Few-Shot Schema Grounding:** Provide explicit graph schema constraints and few-shot Cypher templates in the system prompt.

---

## 7. RAPID REVISION SUMMARY

- **Core Limitation of Vector RAG:** Fails at multi-hop graph traversals and dataset-wide thematic summarization.
- **Property Graph:** Nodes + Directed Relationships + Labels + Key-Value Properties.
- **Cypher:** Declarative pattern-matching language for Neo4j (`MATCH (a)-[:REL]->(b)`).
- **Microsoft GraphRAG:** Ingest text $\to$ extract graph $\to$ Leiden community clustering $\to$ hierarchical community summaries $\to$ parallel global search.
- **Hybrid RAG:** Combine Text-to-Cypher for structured relational queries with Vector Search for unstructured text retrieval.

---

## 8. RECALL CHECKLIST

- [ ] Write a Cypher query using `MATCH`, `WHERE`, and `RETURN`.
- [ ] Diagram a multi-hop query failure in Vector RAG vs its solution in Graph RAG.
- [ ] Define the $(Subject, Predicate, Object)$ triplet representation.
- [ ] Explain how Microsoft GraphRAG uses the Leiden algorithm for community detection.
- [ ] Contrast GraphRAG Global Search vs Local Search.
- [ ] List three security safeguards required for production Text-to-Cypher pipelines.
