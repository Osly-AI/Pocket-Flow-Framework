---
layout: default
title: "Batch"
parent: "Core Abstraction"
nav_order: 4
---

# Batch

**Batch** makes it easier to handle large inputs in one Node or **rerun** a Flow multiple times. Handy for:
- **Chunk-based** processing (e.g., splitting large texts)
- **Multi-file** processing
- **Iterating** over lists of params (e.g., user queries, documents, URLs)

## 1. BatchNode

A **BatchNode** extends `BaseNode` but changes how we handle execution:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

class MapSummaries extends BaseNode {
    // The 'prep' method returns chunks to process
    public async prep(sharedState: any): Promise<string[]> {
        const content = sharedState.data["large_text.txt"] || "";
        const chunkSize = 10000;
        const chunks = [];
        for (let i = 0; i < content.length; i += chunkSize) {
            chunks.push(content.substring(i, i + chunkSize));
        }
        return chunks;
    }

    // The 'execCore' method processes each chunk
    public async execCore(chunk: string): Promise<string> {
        const prompt = `Summarize this chunk in 10 words: ${chunk}`;
        return await callLLM(prompt);
    }

    // The 'post' method combines all summaries
    public async post(prepResult: string[], execResult: string, sharedState: any): Promise<string> {
        sharedState.summary["large_text.txt"] = execResult;
        return DEFAULT_ACTION;
    }
}

// Create and run the flow
const mapSummaries = new MapSummaries();
const flow = new Flow(mapSummaries);

await flow.run({
    data: {
        "large_text.txt": "Your very large text content goes here..."
    },
    summary: {}
});
```

## 2. BatchFlow

A **BatchFlow** runs a **Flow** multiple times with different parameters. Think of it as a loop that replays the Flow for each parameter set.

```typescript
import { BaseNode, BatchFlow, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Define nodes for processing a single file
class LoadFile extends BaseNode {
    public async prep(sharedState: any): Promise<string> {
        return sharedState.filename;
    }

    public async execCore(filename: string): Promise<string> {
        return await readFile(filename);
    }

    public async post(prepResult: string, execResult: string, sharedState: any): Promise<string> {
        sharedState.currentContent = execResult;
        return "summarize";
    }
}

class SummarizeFile extends BaseNode {
    public async prep(sharedState: any): Promise<string> {
        return sharedState.currentContent;
    }

    public async execCore(content: string): Promise<string> {
        return await callLLM(`Summarize: ${content}`);
    }

    public async post(prepResult: string, execResult: string, sharedState: any): Promise<string> {
        sharedState.summaries[sharedState.filename] = execResult;
        return DEFAULT_ACTION;
    }
}

// Build the per-file flow
const loadFileNode = new LoadFile();
const summarizeFileNode = new SummarizeFile();

loadFileNode.addSuccessor(summarizeFileNode, "summarize");
const fileFlow = new Flow(loadFileNode);

// Define the BatchFlow that iterates over files
class SummarizeAllFiles extends BatchFlow {
    public async prep(sharedState: any): Promise<{ filename: string }[]> {
        const filenames = Object.keys(sharedState.data);
        return filenames.map(fn => ({ filename: fn }));
    }

    public async post(prepResults: any[], results: any[], sharedState: any): Promise<string> {
        console.log("All files summarized");
        return DEFAULT_ACTION;
    }
}

// Run the BatchFlow
const summarizeAllFiles = new SummarizeAllFiles();
await summarizeAllFiles.run({
    data: {
        "file1.txt": "Content of file 1...",
        "file2.txt": "Content of file 2..."
    },
    summaries: {}
});
```

## 3. Example: Building an Agent with Pocket Flow

Here's how to use ChatGPT/Claude to help build an agent using Pocket Flow. Copy these prompts:

```
I want to build an agent that can:
1. Take a complex task
2. Break it into subtasks
3. Execute each subtask
4. Combine the results

Using this TypeScript framework:

[paste the pocket.ts code here]

Can you help me create:
1. A TaskPlannerNode that uses an LLM to break down tasks
2. A TaskExecutorNode that handles individual subtasks
3. A ResultCombinerNode that synthesizes results
4. A Flow that connects these together
```

Or for a more specific agent:

```
I want to build a research agent that can:
1. Take a research question
2. Search for relevant information
3. Analyze and synthesize findings
4. Generate a summary report

Using this TypeScript framework:

[paste the pocket.ts code here]

Can you help me create:
1. A SearchNode that finds relevant sources
2. An AnalysisNode that extracts key information
3. A SynthesisNode that combines findings
4. A Flow that orchestrates the research process
```

These prompts help you leverage AI to build sophisticated agents while maintaining the structure and type safety of the Pocket Flow framework.

## Summary

- **BatchNode**: Process large inputs in chunks
- **BatchFlow**: Run flows multiple times with different parameters
- Use AI assistants to help build complex agents with the framework

Remember to handle errors appropriately and test your implementations thoroughly.