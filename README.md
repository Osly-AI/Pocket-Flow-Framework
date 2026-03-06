[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docs](https://img.shields.io/badge/docs-latest-blue)](https://osly-ai.github.io/Pocket-Flow-Framework/)

# Pocketflow Framework

**100 lines of code. That's the entire framework.**

Most LLM frameworks ship thousands of lines of abstraction — classes, adapters, vendor wrappers, plugin systems — and yet the underlying idea is always the same: chain some LLM calls together, pass state between them, and handle failures.

Pocketflow strips all of that away. The core engine is a single TypeScript file that models LLM workflows as a **Nested Directed Graph**: nodes do work, actions route between them, flows orchestrate the graph, and shared state ties it all together.

This is the original core abstraction behind the **[Pocketflow Platform](https://pocketflow.ai)**, where non-developers create custom AI workflows in natural language.

<p align="center">
  <img src="./assets/landing.png" alt="Pocketflow Platform" width="700"/>
</p>

---

## Table of Contents

- [Why This Exists](#why-this-exists)
- [Architecture](#architecture)
- [How It Works](#how-it-works)
- [Quick Start](#quick-start)
- [Usage Guide](#usage-guide)
  - [Nodes](#nodes)
  - [Flows](#flows)
  - [Branching & Cycles](#branching--cycles)
  - [RetryNode](#retrynode)
  - [Nested Flows](#nested-flows)
  - [BatchFlow](#batchflow)
- [Design Patterns](#design-patterns)
- [CLI](#cli)
- [Project Structure](#project-structure)
- [Whitepaper](#whitepaper)
- [Contributing](#contributing)
- [License](#license)

---

## Why This Exists

Every LLM framework eventually converges on the same pattern:

1. Break a task into steps
2. Let each step read/write shared context
3. Decide what happens next based on the result
4. Handle failures gracefully

The difference is how much ceremony surrounds that pattern. Pocketflow's answer: almost none. The entire runtime is ~100 lines. There are no vendor lock-ins, no mandatory adapters, no configuration files. You bring your own LLM client, your own database, your own whatever — and the framework just orchestrates the graph.

**Key properties:**

- **Vendor-agnostic** — Call OpenAI, Anthropic, local models, or a REST API. The framework doesn't care.
- **Tiny surface area** — One file, four classes. You can read and understand the entire codebase in 10 minutes.
- **Nested composition** — Flows can contain other flows. Build complex systems from simple, tested pieces.
- **Parallel execution** — BatchFlow runs multiple flow instances concurrently with automatic fan-out.
- **Built-in retry** — RetryNode wraps any node with configurable retry + backoff.

---

## Architecture

<p align="center">
  <img src="./assets/arc.png" alt="Architecture: nodes sharing state through a directed graph" width="700"/>
</p>

The framework models every workflow as a **Nested Directed Graph**:

| Concept | What it does |
|---------|-------------|
| **Node** | An atomic unit of work. Reads from shared state, executes logic, writes back, and returns an action string. |
| **Action** | A labeled edge connecting one node to the next. The string returned by `post()` determines routing. |
| **Flow** | An orchestrator that walks the graph from a start node, following actions until no successor is found. |
| **Shared State** | A plain object passed to every node. This is how nodes communicate — no message passing, no events. |

---

## How It Works

Every node runs the same three-step lifecycle:

<p align="center">
  <img src="./assets/flow.png" alt="Node lifecycle: Prep → ExecWrapper → Exec → Post, all reading/writing shared state" width="700"/>
</p>

| Step | Method | Purpose |
|------|--------|---------|
| **1. Prep** | `prep(sharedState)` | Pull data from shared state. Validate inputs. Return a prep result for the next step. |
| **2. Exec** | `execCore(prepResult)` | Do the actual work — call an LLM, query a database, run a calculation. Pure logic, no side effects on shared state. |
| **3. Post** | `post(prep, exec, sharedState)` | Write results back to shared state. Return an action string to pick the next node. |

The `execWrapper` sits between prep and exec — it's where `RetryNode` adds its retry loop, and where you can add your own middleware (rate limiting, caching, logging, etc.).

---

## Quick Start

```bash
git clone https://github.com/Osly-AI/Pocket-Flow-Framework.git
cd Pocket-Flow-Framework
npm install
```

Run the test suite:

```bash
npm test
```

Build:

```bash
npm run build
```

---

## Usage Guide

### Nodes

Every node extends `BaseNode` and implements three methods:

```typescript
import { BaseNode, DEFAULT_ACTION } from "pocketflow";

class SummarizeNode extends BaseNode {
  async prep(sharedState: any) {
    // Read from shared state
    return { text: sharedState.document };
  }

  async execCore(prepResult: any) {
    // Call your LLM (bring your own client)
    const response = await openai.chat.completions.create({
      model: "gpt-4",
      messages: [{ role: "user", content: `Summarize: ${prepResult.text}` }]
    });
    return { summary: response.choices[0].message.content };
  }

  async post(prepResult: any, execResult: any, sharedState: any) {
    // Write back to shared state
    sharedState.summary = execResult.summary;
    return DEFAULT_ACTION;
  }

  _clone() { return new SummarizeNode(); }
}
```

### Flows

Connect nodes with `addSuccessor`, then wrap them in a `Flow`:

```typescript
import { Flow, DEFAULT_ACTION } from "pocketflow";

const summarize = new SummarizeNode();
const review = new ReviewNode();
const publish = new PublishNode();

// Linear chain: summarize → review → publish
summarize.addSuccessor(review, DEFAULT_ACTION);
review.addSuccessor(publish, DEFAULT_ACTION);

const pipeline = new Flow(summarize);
await pipeline.run({ document: "..." });
```

### Branching & Cycles

Nodes can return different action strings to route execution:

```typescript
class QualityCheckNode extends BaseNode {
  async post(prepResult: any, execResult: any, sharedState: any) {
    if (execResult.score >= 0.8) return "approved";
    if (sharedState.retryCount < 3) return "retry";
    return "reject";
  }
  // ...
}

const check = new QualityCheckNode();
const publish = new PublishNode();
const revise = new ReviseNode();
const reject = new RejectNode();

check.addSuccessor(publish, "approved");
check.addSuccessor(revise, "retry");
check.addSuccessor(reject, "reject");

// Create a cycle: revise feeds back into check
revise.addSuccessor(check, DEFAULT_ACTION);
```

### RetryNode

Wrap any node with automatic retry and configurable backoff:

```typescript
import { RetryNode } from "pocketflow";

class ResilientLLMNode extends RetryNode {
  constructor() {
    super(3, 2000); // 3 attempts, 2 seconds between retries
  }

  async prep(sharedState: any) { /* ... */ }
  async execCore(prepResult: any) {
    // If this throws, RetryNode catches it and retries
    return await callUnreliableAPI(prepResult);
  }
  async post(prep: any, exec: any, sharedState: any) { /* ... */ }
  _clone() { return new ResilientLLMNode(); }
}
```

### Nested Flows

Flows are nodes — so you can nest them arbitrarily:

```typescript
// Inner flow: extract → summarize
const extract = new ExtractNode();
const summarize = new SummarizeNode();
extract.addSuccessor(summarize, DEFAULT_ACTION);
const extractAndSummarize = new Flow(extract);

// Outer flow: extractAndSummarize → publish
const publish = new PublishNode();
extractAndSummarize.addSuccessor(publish, DEFAULT_ACTION);
const fullPipeline = new Flow(extractAndSummarize);

await fullPipeline.run(sharedState);
```

This is the key design insight: because `Flow` extends `BaseNode`, you can compose arbitrarily complex systems from small, individually testable flows.

### BatchFlow

Process multiple items concurrently. Override `prep()` to return an array — each element spawns a parallel flow execution:

```typescript
import { BatchFlow } from "pocketflow";

class ProcessDocuments extends BatchFlow {
  async prep(sharedState: any) {
    // Return one config per document — each runs the flow in parallel
    return sharedState.documents.map((doc: string) => ({ document: doc }));
  }
}

// The inner flow runs once per document, concurrently
const summarize = new SummarizeNode();
const batch = new ProcessDocuments(summarize);

await batch.run({
  documents: ["paper1.pdf", "paper2.pdf", "paper3.pdf", /* ...100 more */]
});
```

---

## Design Patterns

All common LLM patterns are just different graph topologies:

<p align="center">
  <img src="./assets/paradigm.png" alt="Design patterns: Chaining, Chat, RAG, CoT, Map-Reduce, Agent, Multi-Agent, Supervisor" width="700"/>
</p>

| Pattern | Graph Shape | Example |
|---------|------------|---------|
| **Chaining** | Linear path | Summarize → Draft Reply |
| **Chat** | Self-loop with history | Chat node cycling with stored context |
| **RAG** | Write + Read through vector store | Upload docs → Answer questions |
| **Chain-of-Thought** | Self-loop with reasoning store | Think → Evaluate → Think again |
| **Map-Reduce** | Fan-out + merge | Split text → Summarize chunks → Reduce |
| **Agent** | Loop with branching | Summarize → Review ↔ Draft Reply |
| **Multi-Agent** | Multiple loops with pub/sub | Agents communicating via shared state |
| **Supervisor** | Nested flow with approval loop | Worker flow ↔ Supervisor approval |

You don't need different APIs for these. They're all the same `Node` + `Flow` + `addSuccessor` primitives, wired differently.

---

## Core Abstractions

<p align="center">
  <img src="./assets/abstraction.png" alt="Core abstractions: Node, Flow, Comms, Batch, Async, Action" width="700"/>
</p>

| Abstraction | Description |
|-------------|------------|
| **Node** | Single-step processing unit |
| **Flow** | Multi-step orchestration (directed path through nodes) |
| **Comms** | Shared state store for inter-node communication |
| **Batch** | Repeat the same flow across multiple inputs |
| **Async** | Overlap I/O operations for parallel execution |
| **Action** | Conditional routing (branching and cycles) |

---

## CLI

Scaffold new components quickly:

```bash
npx pocket new node MyProcessor    # Creates src/nodes/MyProcessor.node.ts
npx pocket new flow MyPipeline     # Creates src/flows/MyPipeline.flow.ts
```

Generated files include typed interfaces, lifecycle stubs, and inline documentation.

---

## Project Structure

```
├── src/
│   └── pocket.ts          # The entire framework (~100 lines)
├── tests/
│   ├── pocket.test.ts     # 14 tests, ~99% coverage
│   └── testNodes.ts       # Test helper nodes
├── cli/                   # Code generation CLI
│   ├── src/               # CLI logic
│   └── templates/         # Node and Flow templates
├── examples/
│   └── mcp-addition-ts/   # MCP server example
├── docs/                  # Full documentation site (MkDocs)
├── WHITEPAPER.md           # Technical whitepaper
└── CONTRIBUTING.md         # Contribution guidelines
```

---

## Whitepaper

For a deeper technical discussion of the design philosophy, architecture decisions, and comparisons with other frameworks, see **[WHITEPAPER.md](WHITEPAPER.md)**.

---

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for setup instructions and guidelines.

---

## License

[MIT](LICENSE)
