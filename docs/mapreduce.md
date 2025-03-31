---
layout: default
title: "Map Reduce"
parent: "Paradigm"
nav_order: 3
---

# Map Reduce

Process large inputs by splitting them into chunks (using something like a [BatchNode](./batch.md)-style approach), then combining the result in a **reduce** step.

### Example: Document Summarization

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Placeholder LLM call that takes a prompt and returns a string
async function callLLM(prompt: string): Promise<string> {
  // Replace with your actual LLM logic (OpenAI, local model, etc.)
  return `Summary: ${prompt.substring(0, 60)}...`;
}

/**
 * MapSummaries Node:
 *  - Splits a large text into chunks (prepAsync)
 *  - Summarizes each chunk (execAsync)
 *  - Collects the chunk summaries into an array (postAsync)
 */
export class MapSummaries extends BaseNode {
  // The 'prepAsync' method chunks the text
  public async prepAsync(sharedState: any): Promise<string[]> {
    const text = sharedState.text ?? "";
    const chunkSize = 10000;
    const chunks: string[] = [];
    for (let i = 0; i < text.length; i += chunkSize) {
      chunks.push(text.slice(i, i + chunkSize));
    }
    return chunks;
  }

  // The 'execAsync' method calls the LLM for each chunk
  public async execAsync(chunk: string): Promise<string> {
    return await callLLM(`Summarize this chunk: ${chunk}`);
  }

  // The 'postAsync' method saves each summarized chunk into sharedState
  public async postAsync(sharedState: any, prepResult: string[], execResultList: string[]): Promise<string> {
    sharedState.summaries = execResultList;
    return DEFAULT_ACTION; // Transition to the reduce node
  }
}

/**
 * ReduceSummaries Node:
 *  - Takes the array of chunk summaries
 *  - Merges them into one final summary
 */
export class ReduceSummaries extends BaseNode {
  // The 'prepAsync' method retrieves the chunk summaries
  public async prepAsync(sharedState: any): Promise<string[]> {
    return sharedState.summaries ?? [];
  }

  // The 'execAsync' method calls the LLM to combine the chunk summaries
  public async execAsync(summaries: string[]): Promise<string> {
    const prompt = `Combine these summaries:\n${summaries.join("\n")}`;
    return await callLLM(prompt);
  }

  // The 'postAsync' method saves the final summary
  public async postAsync(sharedState: any, prepResult: string[], execResult: string): Promise<string> {
    sharedState.final_summary = execResult;
    return DEFAULT_ACTION; // Flow ends here by default
  }
}

// Instantiate the Map (split+summaries) and Reduce nodes
const mapNode = new MapSummaries();
const reduceNode = new ReduceSummaries();

// Connect mapNode to reduceNode
mapNode.addSuccessor(reduceNode, DEFAULT_ACTION);

// Create the Flow
const summarizeFlow = new Flow(mapNode);

// Example usage
(async () => {
  const shared: any = {
    text: "Very large text content goes here...",
  };
  await summarizeFlow.runAsync(shared);
  console.log("Final summary:", shared.final_summary);
})();
```

In this **Map-Reduce** approach:

- **MapSummaries**:
  - **prepAsync**: Splits the input text (`sharedState.text`) into chunks.  
  - **execAsync**: Summarizes each chunk via `callLLM(...)`.  
  - **postAsync**: Saves the individual chunk summaries into `sharedState.summaries`.

- **ReduceSummaries**:
  - **prepAsync**: Retrieves the list of chunk summaries.  
  - **execAsync**: Combines them into one final summary with another LLM call.  
  - **postAsync**: Saves the final summary in `sharedState.final_summary`.

By chaining these Nodes, you achieve a straightforward **Map-Reduce** flow for processing large text inputs.
