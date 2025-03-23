---
layout: default
title: "Flow"
parent: "Core Abstraction"
nav_order: 2
---

# Flow

A **Flow** orchestrates how Nodes connect and run, based on **Actions** returned from each Node's `postAsync()` method. You can chain Nodes in a sequence or create branching logic depending on the **Action** string.

## 1. Action-based Transitions

Each Node's `postAsync(shared, prepResult, execResult)` method returns an **Action** string. By default, if `postAsync()` doesn't explicitly return anything, we treat that as `"default"`.

You define transitions with the syntax:

1. **Basic default transition:** `nodeA >> nodeB`  
   This means if `nodeA.postAsync()` returns `"default"` (or `undefined`), proceed to `nodeB`.  
   *(Equivalent to `nodeA - "default" >> nodeB`)*

2. **Named action transition:** `nodeA - "actionName" >> nodeB`  
   This means if `nodeA.postAsync()` returns `"actionName"`, proceed to `nodeB`.

It's possible to create loops, branching, or multi-step flows.

## 2. Creating a Flow

A **Flow** begins with a **start** node (or another Flow). You create it using `new Flow(startNode)` to specify the entry point. When you call `flow.runAsync(sharedState)`, it executes the first node, looks at its `postAsync()` return Action, follows the corresponding transition, and continues until there's no next node or you explicitly stop.

### Example: Simple Sequence

Here's a minimal flow of two nodes in a chain:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Define NodeA
class NodeA extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Preparation logic for NodeA
  }

  public async execAsync(_: void): Promise<void> {
    // Execution logic for NodeA
  }

  public async postAsync(sharedState: any, _: void, __: void): Promise<string> {
    // Transition to NodeB
    return "default";
  }
}

// Define NodeB
class NodeB extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Preparation logic for NodeB
  }

  public async execAsync(_: void): Promise<void> {
    // Execution logic for NodeB
  }

  public async postAsync(sharedState: any, _: void, __: void): Promise<string> {
    // No further nodes to transition to
    return DEFAULT_ACTION;
  }
}

// Instantiate nodes
const nodeA = new NodeA();
const nodeB = new NodeB();

// Define the flow connections
nodeA.addSuccessor(nodeB, "default");

// Create the flow starting with nodeA
const flow = new Flow(nodeA);

// Initial shared state
const sharedState = {};

// Run the flow
flow.runAsync(sharedState).then(() => {
  console.log("Flow completed successfully.");
}).catch(error => {
  console.error("Flow execution failed:", error);
});
```

- When you run the flow, it executes `NodeA`.
- Suppose `NodeA.postAsync()` returns `"default"`.
- The flow then sees the `"default"` Action is linked to `NodeB` and runs `NodeB`.
- If `NodeB.postAsync()` returns `"default"` but we didn't define `NodeB >> somethingElse`, the flow ends there.

### Example: Branching & Looping

Here's a simple expense approval flow that demonstrates branching and looping. The `ReviewExpenseNode` can return three possible Actions:

- `"approved"`: Expense is approved, move to payment processing.
- `"needs_revision"`: Expense needs changes, send back for revision.
- `"rejected"`: Expense is denied, finish the process.

We can wire them like this:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Define ReviewExpenseNode
class ReviewExpenseNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Prepare expense data
  }

  public async execAsync(_: void): Promise<void> {
    // Execute review logic
  }

  public async postAsync(sharedState: any, prepResult: void, execResult: void): Promise<string> {
    // Example decision logic
    const decision = "approved"; // Replace with actual decision-making
    return decision;
  }
}

// Define PaymentNode
class PaymentNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Prepare payment data
  }

  public async execAsync(_: void): Promise<void> {
    // Execute payment processing
  }

  public async postAsync(sharedState: any, prepResult: void, execResult: void): Promise<string> {
    return DEFAULT_ACTION;
  }
}

// Define ReviseExpenseNode
class ReviseExpenseNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Prepare revision data
  }

  public async execAsync(_: void): Promise<void> {
    // Execute revision logic
  }

  public async postAsync(sharedState: any, prepResult: void, execResult: void): Promise<string> {
    return "needs_revision";
  }
}

// Define FinishNode
class FinishNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Prepare finish data
  }

  public async execAsync(_: void): Promise<void> {
    // Execute finish logic
  }

  public async postAsync(sharedState: any, prepResult: void, execResult: void): Promise<string> {
    return DEFAULT_ACTION;
  }
}

// Instantiate nodes
const reviewExpense = new ReviewExpenseNode();
const payment = new PaymentNode();
const reviseExpense = new ReviseExpenseNode();
const finish = new FinishNode();

// Define the flow connections
reviewExpense.addSuccessor(payment, "approved");
reviewExpense.addSuccessor(reviseExpense, "needs_revision");
reviewExpense.addSuccessor(finish, "rejected");

reviseExpense.addSuccessor(reviewExpense, "needs_revision"); // Loop back for revision
payment.addSuccessor(finish, "default"); // Proceed to finish after payment

// Create the flow starting with reviewExpense
const expenseFlow = new Flow(reviewExpense);

// Initial shared state
const sharedState = {};

// Run the flow
expenseFlow.runAsync(sharedState).then(() => {
  console.log("Expense flow completed successfully.");
}).catch(error => {
  console.error("Expense flow execution failed:", error);
});
```

### Flow Diagram

```mermaid
flowchart TD
    reviewExpense[Review Expense] -->|approved| payment[Process Payment]
    reviewExpense -->|needs_revision| reviseExpense[Revise Report]
    reviewExpense -->|rejected| finish[Finish Process]

    reviseExpense --> reviewExpense
    payment --> finish
```

### Running Individual Nodes vs. Running a Flow

- **`node.runAsync(sharedState)`**:  
  Just runs that node alone (calls `prepAsync()`, `execAsync()`, `postAsync()`), and returns an Action.  
  *Use this for debugging or testing a single node.*

- **`flow.runAsync(sharedState)`**:  
  Executes from the start node, follows Actions to the next node, and so on until the flow can't continue (no next node or no next Action).  
  *Use this in production to ensure the full pipeline runs correctly.*

> **Warning:**  
> `node.runAsync(sharedState)` **does not** proceed automatically to the successor and may use incorrect parameters.  
> Always use `flow.runAsync(sharedState)` in production.

## 3. Nested Flows

A **Flow** can act like a Node, enabling powerful composition patterns:

```typescript
// Define sub-flow nodes
const nodeA = new NodeA();
const nodeB = new NodeB();
nodeA.addSuccessor(nodeB);

// Create sub-flow
const subFlow = new Flow(nodeA);

// Connect sub-flow to another node
const nodeC = new NodeC();
subFlow.addSuccessor(nodeC);

// Create parent flow
const parentFlow = new Flow(subFlow);
```

When `parentFlow.run(sharedState)` executes:
1. It starts the `subFlow`
2. `subFlow` runs through its nodes (`NodeA` then `NodeB`)
3. After `subFlow` completes, execution continues to `NodeC`

## Summary

- **Flow:** Orchestrates Node execution based on Actions
- **Action-based Transitions:** Define Node connections using `addSuccessor()`
- **Nested Flows:** Flows can be used as Nodes within other Flows

Remember to handle errors appropriately and test your implementations thoroughly.

## 4. Specialized Flows

In addition to the basic `Flow`, your framework supports specialized flows like `AsyncFlow`, `BatchFlow`, and more, allowing for complex orchestration patterns. These specialized flows can handle asynchronous operations, batch processing, and other advanced scenarios seamlessly within your application.

### Example: Nested Flows in Order Processing

Continuing from the previous order processing example, suppose you want to add another layer of processing, such as logging and notification. You can create additional sub-flows or combine existing ones.

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Define LoggingNode
class LoggingNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Prepare logging data
  }

  public async execAsync(_: void): Promise<void> {
    // Execute logging logic
  }

  public async postAsync(sharedState: any, _, __: void): Promise<string> {
    console.log(`Order ${sharedState.orderId} processed for ${sharedState.customer}`);
    return DEFAULT_ACTION;
  }
}

// Define NotificationNode
class NotificationNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<void> {
    // Prepare notification data
  }

  public async execAsync(_: void): Promise<void> {
    // Execute notification logic
  }

  public async postAsync(sharedState: any, _, __: void): Promise<string> {
    // Send notification
    return DEFAULT_ACTION;
  }
}

// Instantiate logging and notification nodes
const loggingNode = new LoggingNode();
const notificationNode = new NotificationNode();

// Connect shippingFlow to logging and notification
shippingFlow.addSuccessor(loggingNode, "default");
loggingNode.addSuccessor(notificationNode, "default");

// Update the master flow to include logging and notification
orderProcessingFlow.addSuccessor(loggingNode, "default");
loggingNode.addSuccessor(notificationNode, "default");

// Run the updated flow
orderProcessingFlow.runAsync(sharedState).then(() => {
  console.log("Order processing with logging and notification completed successfully.");
}).catch(error => {
  console.error("Order processing failed:", error);
});
```

### Flow Diagram with Nested Flows

```mermaid
flowchart LR
    subgraph orderProcessingFlow["Order Processing Flow"]
        subgraph paymentFlow["Payment Flow"]
            A[Validate Payment] -->|process_payment| B[Process Payment]
            B -->|payment_confirmation| C[Payment Confirmation]
            A -->|reject_payment| D[Handle Rejection]
        end

        subgraph inventoryFlow["Inventory Flow"]
            E[Check Stock] -->|reserve_items| F[Reserve Items]
            F -->|update_inventory| G[Update Inventory]
            E -->|out_of_stock| H[Handle Out of Stock]
        end

        subgraph shippingFlow["Shipping Flow"]
            I[Create Label] -->|assign_carrier| J[Assign Carrier]
            J -->|schedule_pickup| K[Schedule Pickup]
        end

        subgraph loggingFlow["Logging & Notification Flow"]
            L[Logging] -->|default| M[Notification]
        end

        paymentFlow --> inventoryFlow
        inventoryFlow --> shippingFlow
        shippingFlow --> loggingFlow
    end
```

# Summary

By converting your Python-based Flow examples to TypeScript, you can leverage TypeScript's strong typing and modern asynchronous features. The provided examples demonstrate how to implement **Flows**, handle **Action-based Transitions**, and manage **Nested Flows** within your `pocket.ts` framework, ensuring efficient and organized workflow orchestration.

**Key Points:**

- **Flow:**  
  Orchestrates the execution of Nodes based on Actions returned by each node's `postAsync()` method.

- **Action-based Transitions:**  
  Define how Nodes transition to one another based on specific Action strings.

- **Nested Flows:**  
  Allows flows to act as Nodes within other flows, enabling complex and reusable workflow patterns.

- **Running Flows:**  
  Use `flow.runAsync(sharedState)` to execute the entire pipeline, ensuring all transitions and nodes are processed correctly.

**Next Steps:**

- **Implement Actual Logic:**  
  Replace placeholder functions like `callLLM`