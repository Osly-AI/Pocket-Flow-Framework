---
layout: default
title: "Node"
parent: "Core Abstraction"
nav_order: 1
---

# Node

A **Node** is the smallest building block. Each Node (in TypeScript, commonly extending `BaseNode`) has three main **method overrides** (all optional):

1. `prepAsync(sharedState)`  
   - A reliable step for **preprocessing** data from the `sharedState`.  
   - Examples: *query a DB, read files, or serialize data into a string*.  
   - Returns `prepResult`, which is passed to `execAsync()` and `postAsync()`.

2. `execAsync(prepResult)`  
   - The **main execution** step, with optional retries and error handling configured in the node's options.  
   - Examples: *primarily for LLM calls, but can also be used for remote APIs*.  
   - **Note**: If retries are enabled, ensure an **idempotent** implementation.  
   - **Important**: This method should **not** write to `sharedState`. If you need data from `sharedState`, gather it in `prepAsync()` and pass it along as `prepResult`.  
   - Returns `execResult`, which is passed to `postAsync()`.

3. `postAsync(sharedState, prepResult, execResult)`  
   - A reliable postprocessing step to **write results** back to the `sharedState` and decide the next Action.  
   - Examples: *update DB, change states, log results, decide next Action*.  
   - Returns a **string** specifying the next Action (e.g. `"default"` if none is specified).

> Not all Nodes need all three steps. You could implement only `prepAsync()` if you just need to prepare data without calling an LLM.  
{: .note }

---

## Fault Tolerance & Retries

Nodes can **retry** `execAsync()` if it throws an exception. You configure this by passing options (e.g., `maxRetries`, `wait`) to the constructor (or use a property). For instance:

```typescript
const myNode = new SummarizeFile({ maxRetries: 3, wait: 10 });
```

This means:
- `maxRetries = 3`: Up to 3 attempts total.  
- `wait = 10`: Wait 10 seconds before each retry.  

When an exception occurs in `execAsync()`, the Node automatically retries until:
- It either succeeds, or  
- The Node has retried `maxRetries - 1` times already and fails on the last attempt.

---

## Graceful Fallback

If you want to handle errors gracefully instead of throwing, override:

```typescript
public async execFallbackAsync(
  sharedState: any,
  prepResult: any,
  error: Error
): Promise<any> {
  throw error; // Default just rethrows
}
```

By default, it rethrows the exception. But you can return a fallback result instead, which becomes the `execResult` passed to `postAsync()`.

---

## Example: Summarize a File

Below is a Node that reads file content from `sharedState`, calls an LLM to summarize it, and saves the result back:

```typescript
import { BaseNode, DEFAULT_ACTION } from "pocketflowframework";
import { callLLM } from "../path/to/your/llm-wrapper";

export class SummarizeFile extends BaseNode {
  // constructor to accept node config (like maxRetries)
  constructor(private config?: { maxRetries?: number; wait?: number }) {
    super(config);
  }

  // prepAsync: read data from sharedState
  public async prepAsync(sharedState: any): Promise<string> {
    const filename = this.params["filename"] as string;
    const fileContent = sharedState.data?.[filename];
    return fileContent ?? "";
  }

  // execAsync: make the LLM call with the prepared file content
  public async execAsync(fileContent: string): Promise<string> {
    if (!fileContent) {
      throw new Error("Empty file content!");
    }
    const prompt = `Summarize this text in ~10 words:\n${fileContent}`;
    const summary = await callLLM(prompt); // might fail
    return summary;
  }

  // execFallbackAsync: provide a default summary if there's an error
  public async execFallbackAsync(
    sharedState: any,
    prepResult: string,
    err: Error
  ): Promise<string> {
    console.error("LLM call failed:", err);
    return "There was an error processing your request.";
  }

  // postAsync: store the result in sharedState and return "default"
  public async postAsync(sharedState: any, prepResult: string, execResult: string): Promise<string> {
    const filename = this.params["filename"] as string;
    if (!sharedState.summary) {
      sharedState.summary = {};
    }
    sharedState.summary[filename] = execResult;
    return DEFAULT_ACTION; // or just "default"
  }
}

// Example usage:
const summarizeNode = new SummarizeFile({ maxRetries: 3, wait: 10 });
summarizeNode.setParams({ filename: "test_file.txt" });

// Prepare a shared state with data
const shared: any = {
  data: {
    "test_file.txt": "Once upon a time in a faraway land...",
  },
};

// node.runAsync(...) calls prepAsync->execAsync->postAsync
// If execAsync() fails repeatedly, it calls execFallbackAsync() before postAsync().
summarizeNode.runAsync(shared).then((actionReturned) => {
  console.log("Action returned:", actionReturned);  // Usually "default"
  console.log("Summary stored:", shared.summary?.["test_file.txt"]);
}).catch((error) => {
  console.error("Node execution error:", error);
});
```

### Explanation
1. **prepAsync(sharedState)** grabs the file content from `sharedState.data[filename]`.  
2. **execAsync(prepResult)** calls an LLM to summarize the text.  
3. Any error triggers **retry** logic. If all retries fail, **execFallbackAsync()** is called, returning a fallback value.  
4. Finally, **postAsync(sharedState, prepResult, execResult)** writes the summary to `sharedState.summary` and returns `"default"` so the flow can continue.

---  

## Summary

In your `pocket.ts` framework, each Node inherits from `BaseNode` and may implement:

- **prepAsync()**: Gather/parse data from `sharedState`.  
- **execAsync()** (with optional retry logic): Execute the main logic, frequently an LLM call.  
- **execFallbackAsync()**: Provide a fallback result if `execAsync()` fails too many times.  
- **postAsync()**: Write results back to `sharedState`, decide next `Action`.

This approach keeps your code organized while ensuring reliability, retryability, and a clear separation of concerns.  

