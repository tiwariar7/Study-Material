# Day 26: AI Agents — Architecture, Tool Use, Memory, Planning & Multi-Agent Systems

---

## Topics Covered
1. **The Core Anatomy of an AI Agent** (Perception, Brain, Planning, Memory, Tools)
2. **Memory Systems in Autonomous Agents**:
   - Short-term (Working memory in KV cache)
   - Long-term (Episodic & Semantic memory via Vector DBs / Key-Value stores)
3. **Planning & Self-Correction Paradigms**:
   - Sub-goal decomposition (Plan-and-Solve)
   - **Reflexion** (Shinn et al., 2023: Verbal reinforcement learning & self-evaluators)
4. **Tool Use & Native Function Calling**:
   - JSON Schema specification & model-level tool invocation protocols
   - Sandboxed execution environments
5. **Multi-Agent Orchestration Architectures**:
   - **Supervisor / Hierarchical Pattern** (Router / Manager delegating to worker agents)
   - **Collaborative Network Pattern** (CrewAI, MetaGPT role-playing software teams)
   - **Multi-Agent Debate Pattern** (Consensus generation to eliminate hallucinations)
6. **Production Failure Modes & Safeguards** (Loop guards, recursion limits, deterministic fallback)
7. **Complete Python Implementation: Multi-Agent Supervisor Pattern**
8. **Interview Questions, Cheat Sheet & Recall Checklist**

---

## 1. THE ANATOMY OF AN AUTONOMOUS AGENT

```
                            THE AGENTIC BRAIN (LLM)
                                       |
         +-----------------------------+-----------------------------+
         |                             |                             |
         ↓                             ↓                             ↓
    [ PLANNING ]                  [ MEMORY ]                    [ TOOLS ]
  • Goal Decomposition          • Short-term: Context Window  • Search APIs
  • Self-Reflection / Reflexion • Long-term: Vector DB        • Code Execution
  • Dynamic Replanning          • Working: Key-Value scratch  • SQL / Enterprise DBs
```

An **AI Agent** is an autonomous system that uses a Foundation Model as its central reasoning engine to perceive an environment, formulate multi-step plans, invoke tools to alter the external world, observe the consequences, and iterate until a defined terminal objective is satisfied.

---

## 2. AGENT MEMORY TAXONOMY

Inspired by cognitive neuroscience, agent memory is categorized into three tiers:

| Memory Tier | Storage Medium | Lifespan | Primary Purpose |
| :--- | :--- | :--- | :--- |
| **Sensory / Short-Term** | Active Context Window (KV Cache) | Single conversation or session | Current task state, recent messages, active variables |
| **Episodic Long-Term** | Vector Database (Embeddings of past events) | Persistent across sessions | *"How did I solve a similar SQL query error last week?"* |
| **Semantic Long-Term** | Knowledge Graph / Document Store | Permanent enterprise knowledge | Factual reference data, company policy manuals |

---

## 3. PLANNING & REASONING PARADIGMS

```
           PLANNING STRATEGIES COMPARISON
  
  Linear Execution (ReAct):
  Step 1 -> Action 1 -> Step 2 -> Action 2 -> Step 3 (Cannot look ahead; vulnerable to local traps)

  Plan-and-Solve (Wang et al., 2023):
  [Goal] ---> [Create 5-Step Plan] ---> [Execute Steps 1-5 sequentially] (Fails if Step 2 changes reality)

  Reflexion (Shinn et al., 2023):
  [Attempt 1 Fails] ---> [Evaluator Evaluates Error] ---> [Generate Verbal Reflection] 
          |
          +-----> [Stored in Reflection Memory] ---> [Attempt 2 conditions on past mistake!]
```

### The Reflexion Framework
Traditional reinforcement learning requires millions of numerical reward updates. Reflexion equips agents with **verbal reinforcement learning**:
1. **Actor:** Generates an action trajectory to solve a problem.
2. **Evaluator:** Inspects the final outcome and assigns a scalar/binary success score.
3. **Self-Reflection Model:** If the attempt failed, the model analyzes *why* it failed, synthesizing an explicit self-critique:
   - *"In my previous attempt, I assumed the date column was an integer, which raised a TypeError. Next time, I must parse it as datetime using `pd.to_datetime`."*
4. This verbal critique is stored in long-term episodic memory and injected into the prompt of subsequent attempts, driving rapid self-correction without weight updates!

---

## 4. TOOL CALLING MECHANICS (FUNCTION CALLING)

Tool calling does **not** mean the LLM executes code inside its own neural network. The LLM is strictly a text-in, text-out prediction engine.

```
1. Client sends Prompt + Available Tool JSON Schemas to LLM.
2. LLM decides a tool is needed.
3. LLM returns structured JSON payload: {"name": "run_sql", "arguments": {"query": "SELECT *..."}}
4. Client application executes the tool locally / in a secure sandbox.
5. Client captures Tool Output: "Rows returned: 42".
6. Client sends Tool Output back to LLM as a new message: {"role": "tool", "content": "..."}
7. LLM reads the result and continues reasoning.
```

---

## 5. MULTI-AGENT ARCHITECTURAL PATTERNS

```
                         MULTI-AGENT PATTERNS
  
  A. SUPERVISOR PATTERN (Hierarchical)
                 [SUPERVISOR AGENT]
                   /      |      \
                  /       |       \
        [Researcher]  [Coder]  [Reviewer]
  
  B. COLLABORATIVE NETWORK (Peer-to-Peer)
        [Product Mgr] <---> [Architect] <---> [Engineer] <---> [QA]
  
  C. MULTI-AGENT DEBATE (Consensus)
        [Agent A: Pro]  <=======>  [Agent B: Con]
                           |
                           ↓
                   [Judge / Aggregator]
```

1. **Supervisor / Hierarchical Pattern:** A central coordinator agent breaks down a complex user prompt, routes subtasks to specialized worker agents with restricted toolsets, aggregates their outputs, and manages execution state.
2. **Collaborative Role-Playing (MetaGPT / CrewAI):** Agents adopt specialized personas (e.g., Software Architect, Engineer, QA Tester), passing structured artifacts (PRDs, code files, test logs) along a defined assembly line.
3. **Multi-Agent Debate (Du et al., 2023):** Multiple agent instances generate candidate answers to a complex logic puzzle and critique each other's reasoning across rounds. Encourages divergence and reduces individual hallucinations.

---

## 6. PRODUCTION SAFEGUARDS & LOOP MITIGATION

Agents operating in unbounded loops can exhaust API budgets or cause cascading failures in external systems.

```
Production Guardrails Checklist:
[✓] Recursion Limit: Hard stop at N iterations (e.g., max 10 tool calls).
[✓] Budget Throttling: Cap max token spend per task (e.g., max $0.50).
[✓] Loop Detection: Hash previous (Action, Argument) tuples; abort if identical action runs 3 times.
[✓] Sandboxed Tool Execution: Run Python scripts inside isolated Docker containers with no internet access.
[✓] Human Intervention: Trigger interrupt for high-consequence tools (e.g., payment, deletion).
```

---

## 7. COMPLETE PYTHON IMPLEMENTATION: MULTI-AGENT SUPERVISOR

```python
from typing import TypedDict, Literal
from openai import OpenAI

class TaskState(TypedDict):
    user_request: str
    research_notes: str
    code_output: str
    final_answer: str

client = OpenAI()

def supervisor_router(state: TaskState) -> Literal["researcher", "coder", "synthesizer"]:
    """Decides which agent to invoke next based on current state."""
    if not state.get("research_notes"):
        return "researcher"
    elif not state.get("code_output"):
        return "coder"
    return "synthesizer"

def researcher_agent(state: TaskState) -> dict:
    prompt = f"Extract key mathematical formulas needed for: {state['user_request']}"
    res = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return {"research_notes": res.choices[0].message.content}

def coder_agent(state: TaskState) -> dict:
    prompt = f"""Write a Python function based on these notes:
    {state['research_notes']}
    Request: {state['user_request']}"""
    res = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return {"code_output": res.choices[0].message.content}

def synthesizer_agent(state: TaskState) -> dict:
    prompt = f"""Deliver the final response to the user incorporating the research and code:
    Research: {state['research_notes']}
    Code: {state['code_output']}"""
    res = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )
    return {"final_answer": res.choices[0].message.content}

# Orchestration Loop
state: TaskState = {
    "user_request": "Calculate the Fibonacci sequence up to N using dynamic programming.",
    "research_notes": "",
    "code_output": "",
    "final_answer": ""
}

max_steps = 5
for step in range(max_steps):
    next_agent = supervisor_router(state)
    print(f"Step {step + 1}: Routing to [{next_agent}]")

    if next_agent == "researcher":
        state.update(researcher_agent(state))
    elif next_agent == "coder":
        state.update(coder_agent(state))
    elif next_agent == "synthesizer":
        state.update(synthesizer_agent(state))
        break

print("\n--- FINAL ANSWER ---\n", state["final_answer"])
```

---

## 8. TECHNICAL INTERVIEW QUESTIONS & ANSWERS

### Fundamental
1. **Q: What is the fundamental difference between a Chain and an Agent?**
   - **A:** A Chain executes a fixed, hardcoded sequence of steps predetermined by developer code ($A \to B \to C$). An Agent uses the LLM's own internal reasoning at runtime to decide which actions to take, in what order, which tools to invoke, and when the task is sufficiently completed.

2. **Q: Does an LLM execute tool code directly inside its neural weights?**
   - **A:** No. An LLM is purely a next-token prediction engine. When given a tool definition, the LLM emits a structured text string (usually JSON) containing the tool name and input arguments. The host runtime (Python client) parses this output, executes the actual function on the host machine or sandbox, and returns the result back to the LLM context.

### Intermediate
3. **Q: How does the Reflexion architecture improve agent performance on coding benchmarks?**
   - **A:** When an agent's code fails execution or unit tests, standard agents either fail or loop blindly. In Reflexion, an evaluator agent captures the stack trace and unit test errors, and prompts a reflection model to generate an explicit verbal diagnosis of what went wrong. This diagnosis is prepended to the agent's memory buffer, preventing the agent from repeating the identical bug on subsequent iterations.

4. **Q: What is the Supervisor Pattern in multi-agent systems and what problem does it solve?**
   - **A:** In large systems with many tools (e.g., 30+ tools), giving all tools to a single monolithic agent degrades accuracy due to prompt clutter, tool confusion, and context exhaustion. The Supervisor pattern employs a top-level orchestrator agent that maintains a high-level view of the task and routes sub-goals to specialized sub-agents, each equipped only with the 2–3 tools required for their specific domain.

### Advanced
5. **Q: How do you design production safeguards to prevent an autonomous agent from entering an infinite loop of failing tool calls?**
   - **A:**
     1. **Hard Iteration / Step Limits:** Terminate execution if steps exceed a defined threshold (e.g., `max_iterations = 10`).
     2. **State Action Fingerprinting:** Compute a cryptographic hash of the tool name and arguments $(Tool, Args)$. If the identical tuple is called consecutively $N$ times with the same error, trigger a circuit breaker.
     3. **Token / Financial Budget Guards:** Maintain a running tally of prompt/completion tokens and kill the process if the task exceeds a monetary cost ceiling.
     4. **Deterministic Fallback:** Escalate gracefully to human-in-the-loop or return a structured failure message to the client.

---

## 6. RAPID REVISION SUMMARY

- **Agent Components:** Brain (LLM) + Memory (Short/Long) + Planning (Decomposition/Reflexion) + Tools (APIs/Sandboxes).
- **Reflexion:** Verbal self-correction stored in episodic memory to guide subsequent attempts.
- **Function Calling:** LLM emits JSON specifying function arguments $\to$ client executes $\to$ result returned to LLM.
- **Multi-Agent Paradigms:** Supervisor (hierarchical delegation), Collaboration (peer-to-peer pipelines), Debate (consensus).
- **Production Guardrails:** Max iteration limits, action hashing for loop detection, financial budget caps.

---

## 7. RECALL CHECKLIST

- [ ] Diagram the core anatomy of an AI Agent (Brain, Planning, Memory, Tools).
- [ ] Contrast Short-Term, Episodic Long-Term, and Semantic Long-Term memory.
- [ ] Explain the Reflexion feedback loop (Actor, Evaluator, Reflection Buffer).
- [ ] Trace the step-by-step lifecycle of an OpenAI Function Call.
- [ ] Diagram the Multi-Agent Supervisor Pattern.
- [ ] List four production safeguards against infinite agent execution loops.
