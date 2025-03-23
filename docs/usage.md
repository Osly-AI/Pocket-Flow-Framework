---
layout: default
title: "Usage"
nav_order: 2
---

# Usage

This document demonstrates how to use **pocket.ts** in a typical TypeScript project. You can create **Nodes**, connect them in a **Flow**, and run that Flow with some shared state.

---

## 1. Install & Import

```bash
npm install pocketflowframework
```

In your TypeScript code:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework"; 
// Adjust import if your local path is different or you have a monorepo structure
```

---

## 2. Create Nodes

A **Node** extends `BaseNode` (or another node class) and implements any of the following async methods:

- `prepAsync(sharedState: any)`: Prepare and return data for execution.
- `execAsync(prepResult: any)`: Main logic, possibly calling an LLM or API.
- `postAsync(sharedState: any, prepResult: any, execResult: any)`: Post-processing (e.g., saving results, deciding next action).

```typescript
class GreetNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<string> {
    // Suppose we want to greet a user
    const userName = sharedState.userName ?? "Guest";
    return userName;
  }

  public async execAsync(userName: string): Promise<string> {
    return `Hello, ${userName}!`;
  }

  public async postAsync(
    sharedState: any,
    prepResult: string,
    execResult: string
  ): Promise<string> {
    sharedState.greeting = execResult;
    console.log("GreetNode result:", execResult);
    return DEFAULT_ACTION; // proceed to next node or end if none
  }
}
```

---

## 3. Connect Nodes into a Flow

You can chain nodes by specifying how one node transitions to the other. In `pocket.ts`, you do this via `addSuccessor(nextNode, actionString)`:

```typescript
class AskFavoriteColorNode extends BaseNode {
  // ...
}

class RespondColorNode extends BaseNode {
  // ...
}

// Create instances
const greetNode = new GreetNode();
const askColorNode = new AskFavoriteColorNode();
const respondColorNode = new RespondColorNode();

// Connect
greetNode.addSuccessor(askColorNode, DEFAULT_ACTION);
askColorNode.addSuccessor(respondColorNode, DEFAULT_ACTION);

// Build a Flow that starts with greetNode
const flow = new Flow(greetNode);
```

---

## 4. Run the Flow

Create a `sharedState` object and pass it to the Flow's `runAsync` method:

```typescript
(async () => {
  const shared: any = {
    userName: "Alice",
  };

  await flow.runAsync(shared);

  // After the flow completes, inspect sharedState
  console.log("All done! Final shared state:", shared);
})();
```

---

## 5. Handling Multiple Actions

Sometimes, you'll want to branch based on different outcomes from a node. Each node's `postAsync` can return a **string** (often an "Action") which is used to choose the next node.

```typescript
class SurveyNode extends BaseNode {
  public async execAsync(_: unknown): Promise<string> {
    const userInput = await this.askQuestion("Do you like TypeScript? (yes/no)");
    return userInput.trim().toLowerCase();
  }

  public async postAsync(
    sharedState: any,
    prepResult: unknown,
    userResponse: string
  ): Promise<string> {
    if (userResponse === "yes") {
      return "yes_action";
    } else {
      return "no_action";
    }
  }

  private async askQuestion(question: string): Promise<string> {
    // minimal prompt for demonstration
    // replace with a real I/O method, or an LLM prompt
    return Promise.resolve("yes"); 
  }
}

// Usage:
//     surveyNode.addSuccessor(yesNode, "yes_action");
//     surveyNode.addSuccessor(noNode, "no_action");
```

---

## 6. Retrying & Error Handling

`BaseNode` and its subclasses can optionally accept retry parameters (e.g., `maxRetries` and `wait`). If `execAsync` fails, the node will automatically retry.

```typescript
class UnreliableNode extends BaseNode {
  constructor() {
    // e.g., ask for 3 attempts with a 2-second wait
    super({ maxRetries: 3, wait: 2 });
  }

  public async execAsync(_: unknown): Promise<string> {
    if (Math.random() < 0.5) {
      throw new Error("Random failure!");
    }
    return "Success this time!";
  }
}

// If all retries fail, execFallbackAsync is called, if defined.
// If fallback is not defined or rethrows, the flow can fail unless you handle it.
```

---

## 7. Async vs Sync

Most LLM-based tasks or external API calls require **async**. With `pocket.ts`, `prepAsync`, `execAsync`, `postAsync`, and `execFallbackAsync` can all be asynchronous. You can handle synchronous tasks simply by returning a resolved Promise.

---

## 8. Example: Putting It All Together

Below is a small end-to-end flow:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Node #1: Greet user
class GreetNode extends BaseNode {
  public async execAsync(_: unknown): Promise<string> {
    return "Hello! What's your name?";
  }
  public async postAsync(sharedState: any, _: any, greeting: string): Promise<string> {
    console.log(greeting);
    return DEFAULT_ACTION;
  }
}

// Node #2: Get user's name (mock input)
class GetNameNode extends BaseNode {
  public async execAsync(_: unknown): Promise<string> {
    return "Alice"; // In real life, you'd ask or read from user input
  }
  public async postAsync(sharedState: any, _: any, name: string): Promise<string> {
    sharedState.userName = name;
    return DEFAULT_ACTION;
  }
}

// Node #3: Personalize farewell
class FarewellNode extends BaseNode {
  public async execAsync(_: unknown): Promise<string> {
    return "Nice to meet you!";
  }
  public async postAsync(sharedState: any, _: any, farewell: string): Promise<string> {
    console.log(`${farewell} Goodbye, ${sharedState.userName}!`);
    return DEFAULT_ACTION;
  }
}

// Build flow
const greetNode = new GreetNode();
const nameNode = new GetNameNode();
const farewellNode = new FarewellNode();

greetNode.addSuccessor(nameNode, DEFAULT_ACTION);
nameNode.addSuccessor(farewellNode, DEFAULT_ACTION);

const flow = new Flow(greetNode);

// Run flow
flow.runAsync({}).then(() => {
  console.log("Flow complete!");
});
```

Console output might look like:
```
Hello! What's your name?
Nice to meet you! Goodbye, Alice!
Flow complete!
```

---

## Summary

- **Import** `BaseNode` and `Flow` from `pocket.ts`.  
- **Create** Nodes with `prepAsync`, `execAsync`, `postAsync` as needed.  
- **Chain** nodes in a Flow and specify actions.  
- **Run** the flow using `runAsync(sharedState)`.  
- Optionally use features like **retry**, **fallback**, or **action-based branching**.  

This is the basic usage pattern for building Node-based flows in `pocket.ts`. Feel free to explore more advanced features like parallel flows, multi-agent architectures, or specialized node types (batch nodes, etc.). 