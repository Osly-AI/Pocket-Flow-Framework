---
layout: default
title: "Core Abstraction"
nav_order: 2
has_children: true
---

# Core Abstractions

The Pocket Flow Framework is built around three core abstractions: **Nodes**, **Flows**, and **BatchFlows**. These abstractions allow you to define modular, reusable, and scalable workflows.

## 1. Nodes

A **Node** is the fundamental building block of a workflow. It represents a single unit of work and is implemented by extending the [BaseNode](cci:2://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:4:0-78:1) class. Nodes have three primary methods:

- **[prep(sharedState)](cci:1://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:155:4-158:5)**: Prepares the node for execution by initializing any necessary state or resources.
- **[execCore(prepResult)](cci:1://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:122:4-124:5)**: Executes the core logic of the node.
- **[post(prepResult, execResult, sharedState)](cci:1://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:65:4-65:86)**: Determines the next action to take based on the execution result.

Nodes can be chained together using **successors**, which define transitions based on the [post](cci:1://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:65:4-65:86) method's return value.

## 2. Flows

A **Flow** orchestrates the execution of a sequence of nodes. It starts with a single entry point (the `start` node) and follows transitions defined by the [post](cci:1://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:65:4-65:86) method of each node. The [Flow](cci:2://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:103:0-152:1) class extends [BaseNode](cci:2://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:4:0-78:1) and provides methods for orchestrating the execution of nodes.

## 3. BatchFlows

A **BatchFlow** extends the [Flow](cci:2://file:///Users/helenazhang/Pocket-Flow-Framework/src/pocket.ts:103:0-152:1) class to handle workflows that process multiple items in parallel. It prepares a list of items, orchestrates their execution, and aggregates the results.

## Example: Simple Flow

Here’s an example of creating a simple flow with two nodes:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

class NodeA extends BaseNode {
  async prep(sharedState: any): Promise<void> {}
  async execCore(prepResult: any): Promise<void> {}
  async post(prepResult: any, execResult: any, sharedState: any): Promise<string> {
    return "default";
  }
}

class NodeB extends BaseNode {
  async prep(sharedState: any): Promise<void> {}
  async execCore(prepResult: any): Promise<void> {}
  async post(prepResult: any, execResult: any, sharedState: any): Promise<string> {
    return DEFAULT_ACTION;
  }
}

const nodeA = new NodeA();
const nodeB = new NodeB();
nodeA.addSuccessor(nodeB, "default");

const flow = new Flow(nodeA);
flow.run({});
```

This flow starts with `NodeA` and transitions to `NodeB` when `NodeA.post()` returns `"default"`.