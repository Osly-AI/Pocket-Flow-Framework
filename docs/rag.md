---
layout: default
title: "RAG"
parent: "Paradigm"
nav_order: 4
---

# RAG (Retrieval Augmented Generation)

When building LLM applications that **answer questions** from a corpus of documents, you often need to:
1. **Embed** your documents and create a **search index**.  
2. **Retrieve** relevant sections at query time.  
3. **Augment** the LLM prompt with the retrieved context.

Below is a two-Node flow in **TypeScript** that **builds an embedding index** and **answers questions** via a retrieval step.

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

/** 
 * Placeholder for your embedding + index building code.
 * In a real scenario, you'd call a vector DB or embedding service.
 */
async function getEmbedding(text: string): Promise<Float32Array> {
  // Return a 768-D or 1536-D embedding from a service or local model
  // Here we just fake it by returning a vector of length = text.length
  return new Float32Array(text.length).fill(0.5);
}
function createIndex(embeddings: Float32Array[]): any {
  // Build or store the index in memory or an external DB
  // Return an index object for subsequent searches
  return { someIndexObject: true };
}
function searchIndex(index: any, queryEmbedding: Float32Array, topK: number): [Array<[number, number]>, any] {
  // This function should return indices of the most relevant documents
  // and potentially their scores. For demonstration, we'll assume it always returns
  // an index of 0 with a dummy similarity score.
  return [[[0, 0.99]], null];
}

/**
 * Placeholder for an LLM call. Replace with your actual logic, e.g. OpenAI chat API.
 */
async function callLLM(prompt: string): Promise<string> {
  return `Answer to: ${prompt.substring(0, 60)}...`;
}

/**
 * Step 1: PrepareEmbeddingsNode
 *  - Gathers the corpus from sharedState["texts"]
 *  - Creates embeddings
 *  - Builds a search index and stores it in sharedState["search_index"]
 */
export class PrepareEmbeddingsNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<string[]> {
    if (!Array.isArray(sharedState.texts)) {
      throw new Error("sharedState.texts must be an array of strings");
    }
    return sharedState.texts;
  }

  public async execAsync(texts: string[]): Promise<any> {
    // Compute embeddings for each text
    const embeddings: Float32Array[] = [];
    for (let text of texts) {
      const emb = await getEmbedding(text);
      embeddings.push(emb);
    }
    // Create an index from these embeddings
    const index = createIndex(embeddings);
    return index;
  }

  public async postAsync(sharedState: any, prepResult: string[], execResult: any): Promise<string> {
    // Store the search index
    sharedState.search_index = execResult;
    return DEFAULT_ACTION;
  }
}

/**
 * Step 2: AnswerQuestionNode
 *  - Reads a question from the user or a passed-in param
 *  - Searches the index for the most relevant text
 *  - Calls the LLM with the question + relevant text to get an answer
 *  - Outputs the answer
 */
export class AnswerQuestionNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<{ question: string; relevantText: string }> {
    // For a real UI, you might read from the console or an API param
    const question = this.params["question"] ?? "How does Node.js handle concurrency?";

    // Get embedding for this question
    const questionEmbedding = await getEmbedding(question);

    // Search the existing index
    if (!sharedState.search_index) {
      throw new Error("No search index found in sharedState");
    }
    const [results] = searchIndex(sharedState.search_index, questionEmbedding, 1);
    // results might look like: [[docIndex, similarityScore], ...], pick the top one
    const docIndex = results[0][0]; // The top doc index
    const relevantText = sharedState.texts?.[docIndex] ?? "";

    return { question, relevantText };
  }

  public async execAsync({ question, relevantText }: { question: string; relevantText: string }): Promise<string> {
    const prompt = `Question: ${question}\nContext: ${relevantText}\nAnswer: `;
    return await callLLM(prompt);
  }

  public async postAsync(sharedState: any, prepResult: any, execResult: string): Promise<string> {
    console.log(`Answer: ${execResult}`);
    // In a real scenario, you might store the answer in sharedState or return a next Action
    return DEFAULT_ACTION;  // Flow ends or continues
  }
}

/**
 * Example usage:
 */
(async () => {
  const shared: any = {
    texts: [
      "Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient.",
      "TypeScript extends JavaScript by adding types, improving developer productivity.",
      "Retrieval Augmented Generation (RAG) helps LLMs ground responses in real sources."
    ]
  };

  // Create the nodes
  const prepNode = new PrepareEmbeddingsNode();
  const answerNode = new AnswerQuestionNode();
  // Connect them
  prepNode.addSuccessor(answerNode, "default");

  // Build the Flow
  const flow = new Flow(prepNode);

  // Optionally set the question as a param for answerNode
  answerNode.setParams({ question: "What is Node.js concurrency model?" });

  // Run the flow
  await flow.runAsync(shared);
})();
```

**Explanation**:

1. **PrepareEmbeddingsNode**:  
   - Reads an array of texts from `sharedState.texts`.  
   - Computes embeddings and stores them in a new `sharedState.search_index`.  

2. **AnswerQuestionNode**:  
   - Retrieves a question from `params["question"]`.  
   - Embeds the question, searches the index, fetches the top relevant text.  
   - Calls the LLM with a prompt containing both the question and the relevant text.  
   - Logs (or returns) the LLM's answer.

This **RAG** approach ensures answers are *grounded* in a known corpus of documents, improving relevance and trustworthiness.