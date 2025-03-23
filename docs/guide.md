---
layout: default
title: "Guide"
nav_order: 2
---

# Pocket Flow Framework Guide

{: .important }
> The Pocket Flow Framework helps you build modular, flow-based applications by composing nodes into directed graphs.

## Core Concepts

### BaseNode
Every node in a flow extends the `BaseNode` class and implements three key lifecycle methods:

```typescript
abstract class BaseNode {
  // Prepare data or transform shared state
  abstract prep(sharedState: any): Promise<any>;
  
  // Core execution logic
  abstract execCore(prepResult: any): Promise<any>;
  
  // Process results and determine next action
  abstract post(prepResult: any, execResult: any, sharedState: any): Promise<string>;
}
```

### Flow Construction
Nodes are connected using `addSuccessor(node, action)` with "default" as the default action. Here's a complete example:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

class MyNode extends BaseNode {
  async prep(sharedState: any): Promise<any> {
    return { /* prep data */ };
  }
  
  async execCore(prepResult: any): Promise<any> {
    return { /* execution result */ };
  }
  
  async post(prepResult: any, execResult: any, sharedState: any): Promise<string> {
    return DEFAULT_ACTION;
  }

  _clone(): BaseNode {
    return new MyNode();
  }
}

// Connect nodes
const nodeA = new MyNode();
const nodeB = new MyNode();
nodeA.addSuccessor(nodeB, DEFAULT_ACTION);

// Create and run flow
const flow = new Flow(nodeA);
await flow.run({ /* initial shared state */ });
```

## Advanced Features

### RetryNode
Built-in retry mechanism for handling failures:

```typescript
class MyRetryNode extends RetryNode {
  constructor() {
    super(3, 1000); // 3 retries, 1 second interval
  }
  // Implement abstract methods...
}
```

### BatchFlow
Process multiple items in parallel by overriding `prep()` to return an array of items:

```typescript
class MyBatchFlow extends BatchFlow {
  async prep(sharedState: any): Promise<any[]> {
    return [/* array of items to process */];
  }
}
```

### Shared State
- Passed through entire flow
- Modified by nodes as needed
- Persists between node executions

## Best Practices

### Node Design
- Keep nodes focused on single responsibility
- Use `prep()` for data preparation
- Put core logic in `execCore()`
- Use `post()` for state updates and flow control

### Flow Management
- Implement `_clone()` for each node
- Use meaningful action names
- Handle errors appropriately

### State Management
- Keep shared state minimal
- Use TypeScript interfaces for type safety
- Document state structure

### Testing
- Test individual nodes using `node.run()`
- Test flows using `flow.run()`
- Verify state transformations
