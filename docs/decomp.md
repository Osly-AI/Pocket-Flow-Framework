---
layout: default
title: "Task Decomposition"
parent: "Paradigm"
nav_order: 2
---

# Task Decomposition

Many real-world tasks are too complex for a single LLM call. The solution is to decompose them into multiple calls as a [Flow](./flow.md) of Nodes.

### Example: Article Writing

This example demonstrates how to break down the task of writing an article into smaller, manageable steps using TypeScript and the `pocket.ts` framework.

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// Placeholder for an asynchronous LLM call
async function callLLM(prompt: string): Promise<string> {
  // Replace with actual implementation, e.g., API call to an LLM service
  return `Generated response based on prompt: ${prompt}`;
}

export class GenerateOutlineNode extends BaseNode {
  // The 'prepAsync' method prepares the topic for generating an outline
  public async prepAsync(sharedState: any): Promise<string> {
    return sharedState.topic;
  }

  // The 'execAsync' method generates a detailed outline for the article
  public async execAsync(topic: string): Promise<string> {
    const prompt = `Create a detailed outline for an article about ${topic}`;
    const outline = await callLLM(prompt);
    return outline;
  }

  // The 'postAsync' method stores the generated outline in the shared store
  public async postAsync(sharedState: any, _: string, execResult: string): Promise<string> {
    sharedState.outline = execResult;
    return DEFAULT_ACTION; // Proceed to the next node
  }
}

export class WriteSectionNode extends BaseNode {
  // The 'prepAsync' method retrieves the outline from the shared store
  public async prepAsync(sharedState: any): Promise<string> {
    return sharedState.outline;
  }

  // The 'execAsync' method writes content based on the outline
  public async execAsync(outline: string): Promise<string> {
    const prompt = `Write content based on this outline: ${outline}`;
    const draft = await callLLM(prompt);
    return draft;
  }

  // The 'postAsync' method stores the draft in the shared store
  public async postAsync(sharedState: any, _: string, execResult: string): Promise<string> {
    sharedState.draft = execResult;
    return DEFAULT_ACTION; // Proceed to the next node
  }
}

export class ReviewAndRefineNode extends BaseNode {
  // The 'prepAsync' method retrieves the draft from the shared store
  public async prepAsync(sharedState: any): Promise<string> {
    return sharedState.draft;
  }

  // The 'execAsync' method reviews and improves the draft
  public async execAsync(draft: string): Promise<string> {
    const prompt = `Review and improve this draft: ${draft}`;
    const finalArticle = await callLLM(prompt);
    return finalArticle;
  }

  // The 'postAsync' method stores the final article in the shared store
  public async postAsync(sharedState: any, _: string, execResult: string): Promise<string> {
    sharedState.final_article = execResult;
    return DEFAULT_ACTION; // Flow is complete
  }
}

// Instantiate nodes
const generateOutline = new GenerateOutlineNode();
const writeSection = new WriteSectionNode();
const reviewAndRefine = new ReviewAndRefineNode();

// Connect nodes to form the flow: GenerateOutline -> WriteSection -> ReviewAndRefine
generateOutline.addSuccessor(writeSection, DEFAULT_ACTION);
writeSection.addSuccessor(reviewAndRefine, DEFAULT_ACTION);

// Create the flow starting with the GenerateOutline node
const writingFlow = new Flow(generateOutline);

// Define the shared state with the article topic
const shared = { topic: "AI Safety" };

// Run the flow
writingFlow.runAsync(shared).then(() => {
  console.log("Final Article:", shared.final_article);
}).catch(error => {
  console.error("Flow execution failed:", error);
});
```

---

### Explanation:

1. **GenerateOutlineNode**:
   - **prepAsync(sharedState)**:
     - Retrieves the `topic` from the `sharedState` to use in generating the outline.
   - **execAsync(topic)**:
     - Calls the `callLLM` function with a prompt to create a detailed outline for the given topic.
   - **postAsync(sharedState, _, execResult)**:
     - Stores the generated outline in `sharedState.outline`.
     - Returns `DEFAULT_ACTION` to proceed to the next node in the flow.

2. **WriteSectionNode**:
   - **prepAsync(sharedState)**:
     - Retrieves the `outline` from the `sharedState`.
   - **execAsync(outline)**:
     - Calls the `callLLM` function with a prompt to write content based on the provided outline.
   - **postAsync(sharedState, _, execResult)**:
     - Stores the draft content in `sharedState.draft`.
     - Returns `DEFAULT_ACTION` to proceed to the next node.

3. **ReviewAndRefineNode**:
   - **prepAsync(sharedState)**:
     - Retrieves the `draft` from the `sharedState`.
   - **execAsync(draft)**:
     - Calls the `callLLM` function with a prompt to review and improve the draft.
   - **postAsync(sharedState, _, execResult)**:
     - Stores the final article in `sharedState.final_article`.
     - Returns `DEFAULT_ACTION`, indicating that the flow is complete.

4. **Flow Setup**:
   - **Node Instantiation**:
     - Creates instances of each node.
   - **Connecting Nodes**:
     - Sets up the flow by connecting the nodes in the order: `GenerateOutlineNode` → `WriteSectionNode` → `ReviewAndRefineNode`.
   - **Running the Flow**:
     - Initializes the `shared` state with the article topic.
     - Runs the flow asynchronously and logs the final article upon completion.

---

### **Notes:**

- **Asynchronous Operations**:
  - All node methods (`prepAsync`, `execAsync`, `postAsync`) are asynchronous to handle operations like API calls to LLMs efficiently.
  
- **Shared Store Usage**:
  - The `sharedState` object serves as a global store, allowing nodes to share data seamlessly.
  
- **Flow Execution**:
  - The flow begins with generating an outline, proceeds to writing sections based on the outline, and culminates in reviewing and refining the draft to produce the final article.

- **Error Handling**:
  - Ensure that the `callLLM` function and other asynchronous operations include proper error handling to manage potential failures gracefully.

Feel free to adjust the code to fit your actual implementations and expand upon the task decomposition capabilities as required!