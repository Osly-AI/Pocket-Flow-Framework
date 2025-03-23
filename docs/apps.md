---
layout: default
title: "Sample Applications"
nav_order: 5
---

# Sample Applications

Here are some example applications built with the Pocket Flow Framework to demonstrate its capabilities.

## 1. Document Processing Pipeline

A flow that processes documents through multiple stages:

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

class DocumentLoaderNode extends BaseNode {
    async prep(sharedState: any) {
        return sharedState.documentPath;
    }

    async execCore(path: string) {
        // Simulate loading document
        return { content: "Sample document content" };
    }

    async post(prepResult: any, execResult: any, sharedState: any) {
        sharedState.document = execResult;
        return DEFAULT_ACTION;
    }
}

class TextExtractorNode extends BaseNode {
    async prep(sharedState: any) {
        return sharedState.document;
    }

    async execCore(document: any) {
        return document.content.toLowerCase();
    }

    async post(prepResult: any, execResult: any, sharedState: any) {
        sharedState.extractedText = execResult;
        return DEFAULT_ACTION;
    }
}

// Create and connect nodes
const loader = new DocumentLoaderNode();
const extractor = new TextExtractorNode();
loader.addSuccessor(extractor);

// Create flow
const docFlow = new Flow(loader);

// Run the flow
await docFlow.run({
    documentPath: "path/to/document.pdf"
});
```

## 2. Data Processing with Retry Logic

An example showing retry capabilities for API calls:

```typescript
class ApiNode extends RetryNode {
    constructor() {
        super(3, 1000); // 3 retries, 1 second interval
    }

    async prep(sharedState: any) {
        return sharedState.apiEndpoint;
    }

    async execCore(endpoint: string) {
        // Simulate API call that might fail
        const response = await fetch(endpoint);
        if (!response.ok) throw new Error("API call failed");
        return response.json();
    }

    async post(prepResult: any, execResult: any, sharedState: any) {
        sharedState.apiResponse = execResult;
        return DEFAULT_ACTION;
    }
}
```

## 3. Parallel Processing with BatchFlow

Example of processing multiple items in parallel:

```typescript
class ImageProcessingFlow extends BatchFlow {
    async prep(sharedState: any) {
        // Return array of image paths to process
        return sharedState.imagePaths;
    }

    async post(prepResults: string[], results: any[], sharedState: any) {
        sharedState.processedImages = results;
        return DEFAULT_ACTION;
    }
}

// Usage
const batchFlow = new ImageProcessingFlow(processingNode);
await batchFlow.run({
    imagePaths: [
        "image1.jpg",
        "image2.jpg",
        "image3.jpg"
    ]
});
```

## 4. Conditional Branching Flow

Example of a flow with different paths based on conditions:

```typescript
class ValidationNode extends BaseNode {
    async prep(sharedState: any) {
        return sharedState.data;
    }

    async execCore(data: any) {
        return data.isValid;
    }

    async post(prepResult: any, execResult: any, sharedState: any) {
        return execResult ? "valid" : "invalid";
    }
}

// Create nodes
const validator = new ValidationNode();
const successHandler = new SuccessNode();
const errorHandler = new ErrorNode();

// Set up branching
validator.addSuccessor(successHandler, "valid");
validator.addSuccessor(errorHandler, "invalid");

// Create and run flow
const flow = new Flow(validator);
await flow.run({
    data: { isValid: true }
});
```

## 5. Nested Flows Example

Demonstrating how to compose complex workflows:

```typescript
// Sub-flow for data preprocessing
const preprocessFlow = new Flow(preprocessNode);
preprocessFlow.addSuccessor(validationNode);

// Sub-flow for model inference
const inferenceFlow = new Flow(modelNode);
inferenceFlow.addSuccessor(postprocessNode);

// Main flow combining sub-flows
preprocessFlow.addSuccessor(inferenceFlow);
const mainFlow = new Flow(preprocessFlow);

// Run the composed flow
await mainFlow.run({
    input: "Raw data"
});
```

These examples demonstrate key features of the framework:
- Basic node implementation
- Retry logic for robust operations
- Parallel processing with BatchFlow
- Conditional branching
- Flow composition

Each example can be extended and customized based on specific requirements.