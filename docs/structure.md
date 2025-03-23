---
layout: default
title: "Structured Output"
parent: "Paradigm"
nav_order: 1
---

# Structured Output

In many use cases, you may want the LLM to output a specific structure, such as a list or a dictionary with predefined keys.

Common approaches include:

- **Prompting** the LLM to strictly return a defined structure (often sufficient for modern LLMs).  
- Using LLMs or libraries that provide **schema enforcement** (e.g., OpenAI function calling, JSON schema validators).  
- **Post-processing** the LLM's response to extract structured content.

## Example Use Cases

- **Extracting Key Information**

```yaml
product:
  name: Widget Pro
  price: 199.99
  description: |
    A high-quality widget designed for professionals.
    Suitable for advanced users.
```

- **Summarizing Documents into Bullet Points**

```yaml
summary:
  - This product is easy to use.
  - It is cost-effective.
  - Suitable for novices and experts alike.
```

- **Generating Configuration Files**

```yaml
server:
  host: 127.0.0.1
  port: 8080
  ssl: true
```

## Prompt Engineering

When prompting an LLM for **structured** output:

1. **Wrap** the structure in code fences (e.g., \`\`\`yaml).  
2. **Validate** that all required fields exist, and if absent or ill-formed, handle retries or cleanup in your Node logic.

### Example: Summarizing Text in *YAML*

Below is a **TypeScript** Node (`BaseNode`) demonstrating how to prompt an LLM for a YAML-based summary. It prompts for exactly 3 bullet points and then parses the LLM's response as YAML.

```typescript
import { BaseNode, DEFAULT_ACTION } from "pocketflowframework";
import { callLLM } from "../path/to/your/llm-wrapper";

/**
 * SummarizeNode:
 * 1) Prepares a prompt with instructions for YAML output.
 * 2) Calls the LLM to generate the structured YAML.
 * 3) Parses the YAML and validates the result.
 */
export class SummarizeNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<string> {
    // Grab the text to summarize
    const textToSummarize: string = sharedState.text ?? "No text provided.";
    return textToSummarize;
  }

  public async execAsync(text: string): Promise<string> {
    // Construct a prompt that instructs the LLM to return exactly 3 bullet points in YAML
    const prompt = `
Please summarize the following text in YAML with exactly 3 bullet points.

Text:
${text}

The YAML should look like this:

\`\`\`yaml
summary:
  - bullet 1
  - bullet 2
  - bullet 3
\`\`\`

Only return the YAML (including the fences).
`;
    // Call the LLM with your custom logic
    const response = await callLLM(prompt);
    return response;
  }

  public async postAsync(
    sharedState: any,
    prepResult: string,
    llmResponse: string
  ): Promise<string> {
    // Extract the YAML content between the fences
    let yamlStr = "";
    try {
      const startTag = "```yaml";
      const endTag = "```";

      const startIndex = llmResponse.indexOf(startTag);
      const endIndex = llmResponse.indexOf(endTag, startIndex + startTag.length);

      if (startIndex !== -1 && endIndex !== -1) {
        yamlStr = llmResponse.substring(startIndex + startTag.length, endIndex).trim();
      } else {
        throw new Error("LLM response did not contain valid ```yaml``` fences.");
      }

      // Parse the YAML
      // (You might need "js-yaml" or similar library for safe YAML parsing.)
      const yaml = await import("js-yaml");
      const structResult = yaml.load(yamlStr);

      // Validate the structure
      if (!structResult || typeof structResult !== "object") {
        throw new Error("Parsed result is not a valid YAML object.");
      }
      if (!("summary" in structResult) || !Array.isArray((structResult as any).summary)) {
        throw new Error("Expected a 'summary' array in the YAML output.");
      }

      // Save the structured output
      sharedState.summary = structResult;
    } catch (err) {
      // Optionally retry or provide fallback. For now, just log and store an error.
      console.error("Error parsing YAML from LLM:", err);
      sharedState.summary = { error: "Invalid YAML output from LLM." };
    }

    return DEFAULT_ACTION; // Continue the flow
  }
}
```

**How It Works**:  
1. **prepAsync** fetches the text from `sharedState`.  
2. **execAsync** sends a strict YAML-form request to the LLM.  
3. **postAsync** parses and validates the result. If invalid, you can handle it (e.g., retry or fallback).

### Why YAML instead of JSON?

YAML tends to be more **LLM-friendly** for multiline strings. In JSON, you often need to carefully escape quotes or newline characters. For example, a multi-line string in JSON has to store newlines as `\n` and use escaped quotes like `\"`. With YAML, you can use the `|` block literal style to naturally preserve newlines and quotes.

**In JSON**  
```json
{
  "dialogue": "Alice said: \"Hello Bob.\\nHow are you?\\nI am good.\""
}
```
**In YAML**  
```yaml
dialogue: |
  Alice said: "Hello Bob.
  How are you?
  I am good."
```
No special escaping needed.

## Summary

- **Structured Outputs** can make LLM calls more controllable and predictable.  
- **Prompt carefully**: specify the desired structure and syntax (e.g., YAML code fence).  
- **Parse and validate** the returned structure. If the LLM fails to comply, you can retry or use an alternative fallback.  

With `pocket.ts`, you can integrate these steps into a Node-based flow, ensuring that each node is responsible for prompting, parsing, and validating data.
