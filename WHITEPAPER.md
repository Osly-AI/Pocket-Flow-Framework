# Pocketflow: A Minimal Abstraction for LLM Workflow Orchestration

**Helena Zhang, Osly AI**  
**February 2026**

---

## Abstract

Large Language Model (LLM) application development has produced a growing ecosystem of orchestration frameworks — LangChain, LlamaIndex, CrewAI, AutoGen — each adding layers of vendor-specific adapters, plugin systems, and configuration complexity. We present Pocketflow, a framework that takes the opposite approach: a ~100-line TypeScript core that models all LLM workflows as Nested Directed Graphs. We show that four primitives — **Node**, **Flow**, **Action**, and **Shared State** — are sufficient to express every major LLM design pattern, from simple chaining to multi-agent supervisor architectures, without any vendor coupling or configuration overhead.

---

## 1. Introduction

### 1.1 The Problem

The current generation of LLM frameworks shares a common failure mode: over-abstraction. To support every possible use case, these frameworks introduce deep class hierarchies, mandatory adapters for each LLM provider, and complex configuration systems. The result is that developers spend more time learning the framework than building their application.

Consider a typical LLM workflow: retrieve some documents, generate a response, check the quality, and either publish or retry. In most frameworks, this requires understanding chains, agents, tools, retrievers, memory modules, and output parsers — each with their own API surface. But the underlying computation is always the same: a directed graph where nodes do work and edges determine what happens next.

### 1.2 Our Approach

Pocketflow starts from a different premise: **what is the smallest possible abstraction that can express all common LLM patterns?**

Our answer is four concepts:

1. **Node** — An atomic unit of work with a three-phase lifecycle (prep → exec → post)
2. **Flow** — A graph walker that follows nodes along labeled edges
3. **Action** — A string returned by each node that determines the next edge to follow
4. **Shared State** — A plain object passed to every node for inter-node communication

These four primitives, implemented in approximately 100 lines of TypeScript, can express chaining, branching, cycles, retry logic, batch processing, nested composition, and concurrent execution.

---

## 2. Architecture

### 2.1 The Node Lifecycle

Every node in Pocketflow runs the same three-phase lifecycle:

```
prep(sharedState) → execWrapper(prepResult) → execCore(prepResult) → post(prep, exec, sharedState)
```

| Phase | Responsibility | Touches Shared State? |
|-------|---------------|----------------------|
| **Prep** | Read from shared state, validate inputs, prepare data for execution | Read only |
| **Exec** | Perform the core computation (LLM call, API request, calculation) | No |
| **Post** | Write results back to shared state, determine next action | Read + Write |

This separation is intentional. By isolating the execution phase from state access, we achieve:

- **Testability**: `execCore` can be unit-tested with mock inputs, independent of shared state
- **Composability**: The `execWrapper` layer allows cross-cutting concerns (retry, caching, rate limiting) without modifying node logic
- **Safety**: Parallel execution is safe because only `post` writes to shared state, and `post` runs sequentially within a flow

### 2.2 The Graph Model

A Pocketflow workflow is a directed graph where:

- **Vertices** are `BaseNode` instances
- **Edges** are labeled with action strings
- **Traversal** is deterministic: at each node, the `post()` return value selects exactly one outgoing edge

The `Flow` class walks this graph:

```typescript
async orchestrate(sharedState, flowParams?) {
    let currentNode = await this.getStartNode();
    while (currentNode) {
        currentNode.setParams(flowParams ?? this.flow_params);
        const action = await currentNode.run(sharedState);
        currentNode = currentNode.getSuccessor(action);
    }
}
```

When `getSuccessor` returns `undefined` (no edge for the returned action), the flow terminates. This is the entire orchestration engine.

### 2.3 Nested Composition

The critical design decision in Pocketflow is that `Flow` extends `BaseNode`. This means flows are nodes — they can be added as successors to other nodes, wrapped in other flows, and composed arbitrarily.

```
Flow A
├── Node 1
├── Flow B (treated as a node)
│   ├── Node 2
│   └── Node 3
└── Node 4
```

This enables hierarchical decomposition: complex workflows are built by composing well-tested sub-flows, not by writing monolithic graph definitions.

### 2.4 Concurrency Model

`BatchFlow` extends `Flow` to support fan-out parallelism. Its `prep()` returns an array, and each element triggers an independent, concurrent execution of the underlying flow:

```typescript
async run(sharedState) {
    const prepResultList = await this.prep(sharedState);
    const resultPromises = prepResultList.map(
        prepResult => this.orchestrate(sharedState, prepResult)
    );
    await Promise.all(resultPromises);
    return this.post(prepResultList, resultList, sharedState);
}
```

Because each flow execution receives its own `flowParams` (from the prep result array) and nodes are cloned before execution (`getSuccessor` calls `clone()`), there are no race conditions between concurrent executions.

### 2.5 Retry Semantics

`RetryNode` overrides the `execWrapper` method to add retry logic around `execCore`:

```typescript
async execWrapper(prepResult) {
    for (let i = 0; i < this.maxRetries; i++) {
        try {
            return await this.execCore(prepResult);
        } catch (error) {
            await sleep(this.intervalMs);
        }
    }
    throw new Error("Max retries reached");
}
```

This design demonstrates the extensibility of the `execWrapper` pattern: any cross-cutting concern (circuit breaking, rate limiting, caching, telemetry) can be implemented the same way.

---

## 3. Pattern Expressiveness

We demonstrate that all commonly cited LLM workflow patterns are expressible as Pocketflow graphs.

### 3.1 Linear Chaining

The simplest pattern: a directed path through multiple nodes.

```
[Summarize Email] → [Draft Reply]
```

Implementation: `summarize.addSuccessor(draft, DEFAULT_ACTION)`

### 3.2 Branching (Agent Pattern)

Nodes return different action strings based on execution results, routing to different successors.

```
[Summarize Email] ─── "needs_review" ──→ [Review]
                  ─── "approved" ──→ [Draft Reply]
```

### 3.3 Cycles (Chat, CoT)

A node can be its own (indirect) successor. The flow walks the cycle until a different action breaks out.

```
[Chat] ←── "continue" ──┐
  │                      │
  └──────────────────────┘
  │
  └── "done" ──→ (end)
```

### 3.4 Map-Reduce (BatchFlow)

`BatchFlow.prep()` returns N items → N concurrent flow executions → `BatchFlow.post()` aggregates.

```
[Split Text] → [Summarize Chunk] ×N → [Reduce Summaries]
```

### 3.5 RAG (Retrieval-Augmented Generation)

Two flows sharing state through a vector database written to shared state:

```
Flow 1: [Upload Documents] → writes to sharedState.vectorDB
Flow 2: [Answer Questions] → reads from sharedState.vectorDB
```

### 3.6 Multi-Agent

Multiple flows communicating via shared state, potentially running concurrently via `BatchFlow`:

```
Agent 1: [Researcher] ←→ [Writer]     ─┐
Agent 2: [Fact Checker] ←→ [Editor]    ├── via sharedState
Agent 3: [Reviewer]                    ─┘
```

### 3.7 Supervisor

Nested flows with approval cycles. A worker flow runs inside a supervisor node that loops until approval:

```
[Supervisor] → [Worker Flow (nested)] → [Quality Check]
     ↑                                       │
     └───────── "reject" ────────────────────┘
                "approve" → (end)
```

---

## 4. Comparison with Existing Frameworks

| Property | LangChain | LlamaIndex | CrewAI | Pocketflow |
|----------|-----------|------------|--------|------------|
| Core size (lines) | ~50,000+ | ~30,000+ | ~10,000+ | **~100** |
| Vendor adapters required | Yes | Yes | Yes | **No** |
| Built-in LLM client | Yes | Yes | Yes | **No** (BYOC) |
| Nested flow composition | Limited | Limited | No | **Yes** |
| Parallel batch execution | Via callbacks | Via callbacks | Via agents | **Built-in** |
| Learning curve | High | Medium | Medium | **Low** |
| Configuration files | Multiple | Multiple | YAML | **None** |

The trade-off is explicit: Pocketflow provides no built-in LLM client, no retriever implementations, no prompt templates. You bring all of that yourself. What you get in return is a runtime so simple that you can read the entire source code in 10 minutes and extend it however you need.

---

## 5. Implementation Details

### 5.1 Clone Safety

Parallel execution requires that nodes don't share mutable state. Pocketflow handles this with `clone()`:

- `getSuccessor()` returns a **cloned** node, not the original
- `getStartNode()` in `Flow` returns a **cloned** start node
- `clone()` uses `lodash.cloneDeep` for `flow_params` and a shallow copy of the successor map

This means the graph definition (the wiring between nodes) is shared, but the runtime state of each node instance is isolated.

### 5.2 Action Uniqueness

Each node enforces that action strings map to exactly one successor. Attempting to add a second successor for the same action string throws an error. This prevents ambiguous routing.

### 5.3 TypeScript Design

The framework uses abstract classes rather than interfaces because the base implementations of `run()`, `execWrapper()`, and `clone()` provide meaningful default behavior. Subclasses only need to implement the three lifecycle methods (`prep`, `execCore`, `post`) and `_clone()`.

---

## 6. Use Cases

Pocketflow was designed to power the [Pocketflow Platform](https://pocketflow.ai), where workflows are assembled from natural language descriptions. The framework's small API surface makes it a compelling target for LLM-based code generation: an AI system only needs to understand four classes to generate any workflow.

Additional validated use cases:

- **Document processing pipelines** — Batch-process PDFs through extraction → classification → summarization
- **Conversational agents** — Chat loops with tool use, memory, and conditional branching
- **Data enrichment** — Fan-out API calls across multiple data sources, aggregate results
- **Content moderation** — Multi-stage review with human-in-the-loop approval cycles
- **Research automation** — RAG pipelines with iterative refinement loops

---

## 7. Conclusion

Pocketflow demonstrates that LLM workflow orchestration does not require complex frameworks. A Nested Directed Graph with four primitives — Node, Flow, Action, and Shared State — is sufficient to express every common pattern. By stripping away vendor coupling and configuration overhead, we reduce the barrier to building, testing, and understanding LLM applications.

The framework is open-sourced under the MIT license at [github.com/Osly-AI/Pocket-Flow-Framework](https://github.com/Osly-AI/Pocket-Flow-Framework).

---

## References

1. Chase, H. (2022). LangChain. [github.com/langchain-ai/langchain](https://github.com/langchain-ai/langchain)
2. Liu, J. (2022). LlamaIndex. [github.com/run-llama/llama_index](https://github.com/run-llama/llama_index)
3. Moura, J. (2024). CrewAI. [github.com/joaomdmoura/crewAI](https://github.com/joaomdmoura/crewAI)
4. Wu, Q., et al. (2023). AutoGen: Enabling Next-Gen LLM Applications via Multi-Agent Conversation. *arXiv:2308.08155*.
5. Yao, S., et al. (2023). ReAct: Synergizing Reasoning and Acting in Language Models. *ICLR 2023*.
