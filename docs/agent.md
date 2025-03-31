---
layout: default
title: "Agent"
parent: "Paradigm"
nav_order: 6
---

# Agent

For many tasks, we need agents that take dynamic and recursive actions based on the inputs they receive.
You can create these agents as **Nodes** connected by *Actions* in a directed graph using [Flow](./flow.md).

### Example: Search Agent

This agent:
1. Decides whether to search or answer.
2. If it decides to search, loops back to decide if more searches are needed.
3. Finally answers once enough context has been gathered.

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Placeholder for an LLM call
async function callLLM(prompt: string): Promise<string> {
  // Example function to call a Large Language Model
  // Return a YAML-like string (or any structured string) in practice
  return `
\`\`\`yaml
action: "search"
reason: "Need more results"
search_term: "Nobel Prize 2024"
\`\`\`
`;
}

// Placeholder for a web search
async function searchWeb(searchTerm: string): Promise<string> {
  // Example function that interacts with an external API
  return `Search Results for: ${searchTerm}`;
}

export class DecideAction extends BaseNode {
  // The 'prep' method extracts data from sharedState to pass into execCore
  public async prep(sharedState: any): Promise<[string, string]> {
    const context = sharedState.context ?? "No previous search";
    const query = sharedState.query;
    return [query, context];
  }

  // The main logic calls callLLM to decide whether to search again or to answer
  public async execCore(inputs: [string, string]): Promise<any> {
    const [query, context] = inputs;
    const prompt = `
Given input: ${query}
Previous search results: ${context}
Should I: 1) Search web for more info 2) Answer with current knowledge
Output in YAML:
\`\`\`yaml
action: search/answer
reason: Explanation
search_term: search phrase if action is search
\`\`\`
    `;

    const resp = await callLLM(prompt);

    // Parse YAML from resp (this is example logic; you'd use a real YAML parser)
    const yamlStr = resp.split("```yaml")[1]?.split("```")[0]?.trim() || "";
    // Assume the structure is { action, reason, search_term? }
    const parsed = { action: "search", reason: "Need more results", search_term: "Nobel Prize 2024" };
    // In a real scenario, you'd do something like:
    // const parsed = yaml.load(yamlStr); // using js-yaml or similar

    if (parsed.action === "search" && !parsed.search_term) {
      throw new Error("Missing search_term for 'search' action!");
    }

    return parsed;
  }

  public async post(prepResult: [string, string], execResult: any, sharedState: any): Promise<string> {
    if (execResult.action === "search") {
      sharedState.search_term = execResult.search_term;
    }
    return execResult.action;
  }
}

export class SearchWebNode extends BaseNode {
  public async prep(sharedState: any): Promise<string> {
    return sharedState.search_term;
  }

  public async execCore(searchTerm: string): Promise<string> {
    return await searchWeb(searchTerm);
  }

  public async post(prepResult: string, execResult: string, sharedState: any): Promise<string> {
    const previous = sharedState.context || [];
    sharedState.context = [...previous, { term: prepResult, result: execResult }];

    // Loop back to the DecideAction node
    return "decide";
  }
}

export class DirectAnswer extends BaseNode {
  public async prep(sharedState: any): Promise<[string, any]> {
    return [sharedState.query, sharedState.context ?? ""];
  }

  public async execCore(inputs: [string, any]): Promise<string> {
    const [query, context] = inputs;
    const prompt = `Context: ${JSON.stringify(context)}\nAnswer this query: ${query}`;
    return await callLLM(prompt);
  }

  public async post(
    prepResult: [string, any],
    execResult: string,
    sharedState: any
  ): Promise<string> {
    console.log("Answer:", execResult);
    sharedState.answer = execResult;
    return DEFAULT_ACTION; // or any string that indicates the flow is done
  }
}

// Connect nodes
const decide = new DecideAction();
const search = new SearchWebNode();
const answer = new DirectAnswer();

decide.addSuccessor(search, "search");
decide.addSuccessor(answer, "answer");
search.addSuccessor(decide, "decide");  // loop back

const flow = new Flow(decide);
flow.run({ query: "Who won the Nobel Prize in Physics 2024?" });
```

In this TypeScript example:

- **DecideAction** checks the shared state, queries an LLM, and sets the next action to either “search” or “answer.”
- **SearchWebNode** performs a web search and appends the result to `sharedState.context`, then loops back to the decision node.
- **DirectAnswer** produces a final answer when enough context has been gathered.

**Notes:**

- **YAML Parsing:** In the `execCore` method of `DecideAction`, replace the placeholder parsing logic with a proper YAML parser like [`js-yaml`](https://github.com/nodeca/js-yaml) to handle the YAML response from `callLLM`.

  ```typescript
  import yaml from 'js-yaml';

  // Inside execCore
  const parsed = yaml.load(yamlStr) as { action: string; reason: string; search_term?: string };
  ```

- **LLM and Search Implementations:** Ensure that the `callLLM` and `searchWeb` functions are properly implemented to interact with your chosen LLM service and web search API, respectively.

- **Error Handling:** Enhance error handling as needed, especially around the parsing and execution steps to make your agents more robust.

Feel free to adjust the code to fit your actual implementations and expand upon the agent’s capabilities as required!