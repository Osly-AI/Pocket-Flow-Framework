---
layout: default
title: "(Advanced) Parallel"
parent: "Core Abstraction"
nav_order: 6
---

# (Advanced) Parallel

**Parallel** Nodes and Flows let you run multiple **Async** Nodes and Flows **concurrently**—for example, summarizing multiple texts at once. This can improve performance by overlapping I/O and compute.

> In Node.js, JavaScript can only have true parallelism when using worker threads or multiple processes, but using async/await can still **overlap I/O** operations (e.g., LLM/network/database calls) effectively.
{: .warning }

## 1. AsyncParallelBatchNode

This concept is akin to an **AsyncBatchNode** but runs `execAsync()` in **parallel** for each item. Let's define a `ParallelSummaries` node that splits an array of texts, calls an LLM for each one **in parallel**, and then combines results:

```typescript
import { AsyncParallelBatchNode, Flow, DEFAULT_ACTION } from "pocketflowframework";
import { callLLM } from "../path/to/your/llm-wrapper";

export class ParallelSummaries extends AsyncParallelBatchNode<string, string> {
  // prepAsync returns the array of items to process in parallel
  public async prepAsync(sharedState: any): Promise<string[]> {
    return sharedState.texts ?? [];
  }

  // execAsync is called in parallel for each item from prepAsync()
  public async execAsync(text: string): Promise<string> {
    const prompt = `Summarize: ${text}`;
    return await callLLM(prompt); // LLM call
  }

  // postAsync collects the results into sharedState
  public async postAsync(
    sharedState: any,
    prepResult: string[],
    execResultList: string[]
  ): Promise<string> {
    sharedState.summary = execResultList.join("\n\n");
    return DEFAULT_ACTION; // continue or end flow as needed
  }
}

// Example usage:
const node = new ParallelSummaries();
const flow = new Flow(node);

const shared: any = {
  texts: [
    "Node.js is a JavaScript runtime built on Chrome's V8 engine.",
    "TypeScript is a typed superset of JavaScript providing better tooling.",
    "Parallel processing can reduce total latency for I/O-bound tasks."
  ],
};

flow.runAsync(shared).then(() => {
  console.log("All parallel summaries done.");
  console.log("Combined summary:", shared.summary);
});
```

**Key Points**:
1. **prepAsync** returns an array of texts.  
2. **execAsync** is invoked in parallel for each text.  
3. **postAsync** merges results (e.g. joins them as a single string).

## 2. AsyncParallelBatchFlow

A **parallel** version of a **BatchFlow**, where each iteration of a sub-flow runs **concurrently** using different parameters. For example, if you have a **LoadAndSummarizeFile** flow, you can run it in parallel for multiple files at once.

```typescript
import { AsyncParallelBatchFlow, Flow } from "pocketflowframework";
import { LoadAndSummarizeFile } from "./somewhere";

export class SummarizeMultipleFiles extends AsyncParallelBatchFlow {
  // We override prepAsync to produce a list of param objects
  public async prepAsync(sharedState: any): Promise<any[]> {
    // Return one param object per file
    const files: string[] = sharedState.files ?? [];
    return files.map((filename) => ({ filename }));
  }
}

// Example usage:
const subFlow = new Flow(new LoadAndSummarizeFile());
const parallelFlow = new SummarizeMultipleFiles(subFlow);

const shared: any = {
  files: ["doc1.txt", "doc2.txt", "doc3.txt"],
};

parallelFlow.runAsync(shared).then(() => {
  console.log("All files processed in parallel!");
  // shared might now contain combined summaries or saved results per file
});
```

**Notes**:
1. **prepAsync** returns an array of param objects.  
2. Each item runs the `subFlow` **at the same time** with different parameters.  
3. This is especially suitable for I/O-bound tasks like LLM calls or file operations.

## Best Practices

1. **Ensure Tasks Are Independent**  
   - Parallelizing tasks that share state or depend on each other can introduce conflicts or race conditions. Plan to keep each task's data separate or properly synchronized if needed.

2. **Watch Out for Rate Limits**  
   - Parallel calls may quickly trigger rate limits on LLM services. You may need a **throttling** mechanism or concurrency limits (e.g. a semaphore).

3. **Leverage Batch APIs If Available**  
   - Some LLM providers offer batch inference endpoints. This can be more efficient than launching many parallel requests and may help with rate-limit policies.

4. **Tune Concurrency**  
   - In Node.js, concurrency techniques rely on asynchronous I/O. For CPU-bound tasks, consider Worker Threads or other processes.

5. **Clean Error Handling**  
   - If one parallel task fails, decide whether to continue or fail the entire flow. A custom error handling strategy may be needed for large-scale parallel tasks.
