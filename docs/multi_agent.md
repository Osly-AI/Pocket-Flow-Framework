---
layout: default
title: "(Advanced) Multi-Agents"
parent: "Paradigm"
nav_order: 7
---

# (Advanced) Multi-Agents

Sometimes, you want **multiple agents** (or flows) working together, each performing different tasks or roles. They can communicate by passing messages or updating shared states. Below are some **TypeScript** examples.

---

## Example 1: Agent Communication with a Shared Queue

Here's how to implement communication using a queue-like structure in Node.js. The agent listens for messages, processes them, and loops back to await more. We will simulate an asynchronous message queue using standard JavaScript patterns (e.g., an array plus `setInterval` or an event-based approach).

```typescript
import { BaseNode, Flow, DEFAULT_ACTION } from "pocketflowframework";

// We'll define a simple queue interface
interface MessageQueue {
  messages: string[];
  // optional signals or a real concurrency approach
}

// This is our AgentNode, which reads a message from the queue each time it runs.
// For demonstration, we poll the queue at intervals to simulate asynchronous arrival of messages.
export class AgentNode extends BaseNode {
  public async prepAsync(sharedState: any): Promise<string | null> {
    const messageQueue: MessageQueue = this.params["messages"] as MessageQueue;
    if (!messageQueue || !Array.isArray(messageQueue.messages)) {
      throw new Error("Invalid message queue");
    }

    // Wait until there's at least one message or return null if no messages are pending
    if (messageQueue.messages.length === 0) {
      return null;
    }

    // Dequeue the first message
    return messageQueue.messages.shift() || null;
  }

  public async execAsync(message: string | null): Promise<string | null> {
    if (message === null) {
      // No new message
      return null;
    }

    // Process or log the message
    console.log(`Agent received: ${message}`);
    // You could also call an LLM or do more sophisticated processing here
    return message;
  }

  public async postAsync(sharedState: any, prepResult: string | null, execResult: string | null): Promise<string> {
    // We can continue if there's more work to do; otherwise, we might wait or exit.
    // For this example, we just loop forever (polling), so we return "loop"
    return "loop";
  }
}

// Example usage
(async () => {
  // Our simulated queue
  const messageQueue: MessageQueue = {
    messages: [],
  };

  // Create the agent node
  const agent = new AgentNode();
  // Connect the agent node to itself on the "loop" action
  agent.addSuccessor(agent, "loop");

  // Create a flow starting from our agent
  const flow = new Flow(agent);

  // Set the flow params so the agent node knows about the queue
  flow.setParams({ messages: messageQueue });

  // We'll also define a simple message generator that appends to the queue periodically
  setInterval(() => {
    const timestamp = Date.now();
    const sampleMessages = [
      "System status: all systems operational",
      "Memory usage: normal",
      "Network connectivity: stable",
      "Processing load: optimal",
    ];
    const msg = `${sampleMessages[timestamp % sampleMessages.length]} | timestamp_${timestamp}`;
    messageQueue.messages.push(msg);
  }, 1000);

  // Run the flow
  const shared = {};
  // Because it loops indefinitely, this flow never truly ends unless we forcibly stop it.
  // In a real app, you'd have a stopping condition or signal.
  flow.runAsync(shared).catch((err) => {
    console.error("Flow execution failed:", err);
  });
})();
```

**Explanation:**
- We store messages in `messageQueue.messages`.  
- Each agent run cycle uses `prepAsync` to dequeue one message.  
- The node returns `"loop"` in `postAsync`, telling the flow to run the same node again for the next message.

---

## Example 2: Interactive Multi-Agent Example: Taboo Game

Here's a more complex setup with **two agents** (a "Hinter" and a "Guesser") playing a simplified word-guessing game. They communicate via two queues:  
- `hinterQueue` sends guesses to the Hinter agent,  
- `guesserQueue` sends hints to the Guesser agent.

**Warning**: Below is a conceptual example. In a real Node.js environment, you might orchestrate concurrency differently (e.g., using `Promise.all`, or a dedicated event system).

```typescript
import { BaseNode, Flow } from "pocketflowframework";

// Placeholder LLM function (replace with real calls as needed)
async function callLLM(prompt: string): Promise<string> {
  // For demonstration
  return `LLM says: ${prompt.substring(0, 60)}`;
}

/** 
 * AsyncHinter:
 *  1) Waits for a guess from the guesser (via hinterQueue).
 *  2) Generates a new hint while avoiding certain forbidden words.
 *  3) Sends the hint to guesserQueue.
 */
export class AsyncHinter extends BaseNode {
  public async prepAsync(sharedState: any): Promise<{
    guess: string;
    target: string;
    forbidden: string[];
    pastGuesses: string[];
  } | null> {
    // Dequeue guess
    const hinterQueue = sharedState.hinterQueue as string[];
    if (!Array.isArray(hinterQueue)) throw new Error("hinterQueue not found");

    if (hinterQueue.length === 0) {
      return null; // no new guess in queue
    }
    const guess = hinterQueue.shift() as string;

    // If guess == "GAME_OVER", we can end the Hinter agent
    if (guess === "GAME_OVER") {
      return null;
    }

    return {
      guess,
      target: sharedState.target_word,
      forbidden: sharedState.forbidden_words,
      pastGuesses: sharedState.past_guesses ?? [],
    };
  }

  public async execAsync(inputs: {
    guess: string;
    target: string;
    forbidden: string[];
    pastGuesses: string[];
  } | null): Promise<string | null> {
    if (!inputs) {
      return null; // means we should end
    }
    const { guess, target, forbidden, pastGuesses } = inputs;

    // The prompt for generating a hint from the LLM
    let prompt = `Generate a 1-sentence hint for the word "${target}". Avoid these words: ${forbidden.join(", ")}. `;
    if (guess !== "") {
      prompt += `Previous guess was: "${guess}". `;
    }
    if (pastGuesses.length) {
      prompt += `Past wrong guesses: ${pastGuesses.join(", ")}. `;
    }
    prompt += "Hint: use at most 5 words.";

    const hint = await callLLM(prompt);
    console.log(`\nHinter: Here's your hint -> ${hint}`);
    return hint;
  }

  public async postAsync(
    sharedState: any,
    prepResult: {
      guess: string;
      target: string;
      forbidden: string[];
      pastGuesses: string[];
    } | null,
    execResult: string | null
  ): Promise<string> {
    // If no inputs or execResult, game is over or no messages left
    if (!prepResult || execResult === null) {
      return "end"; 
    }

    // Send the generated hint to guesserQueue
    const guesserQueue = sharedState.guesserQueue as string[];
    guesserQueue.push(execResult);

    return "continue"; 
  }
}

/** 
 * AsyncGuesser:
 *  1) Waits for a hint from guesserQueue.
 *  2) Generates a guess.
 *  3) Checks correctness. If correct, game ends; else adds guess to pastGuesses and re-queues for Hinter.
 */
export class AsyncGuesser extends BaseNode {
  public async prepAsync(sharedState: any): Promise<{
    hint: string;
    pastGuesses: string[];
    target: string;
  } | null> {
    const guesserQueue = sharedState.guesserQueue as string[];
    if (!Array.isArray(guesserQueue)) throw new Error("guesserQueue not found");

    if (guesserQueue.length === 0) {
      return null;
    }
    const hint = guesserQueue.shift() as string;
    return {
      hint,
      pastGuesses: sharedState.past_guesses ?? [],
      target: sharedState.target_word,
    };
  }

  public async execAsync(inputs: {
    hint: string;
    pastGuesses: string[];
    target: string;
  } | null): Promise<string | null> {
    if (!inputs) {
      return null;
    }

    const { hint, pastGuesses, target } = inputs;
    let prompt = `We have hint: "${hint}". Past wrong guesses: ${pastGuesses.join(", ")}. Make a new single-word guess:`;
    // In reality, you'd refine this logic or call an actual LLM
    const guess = await callLLM(prompt);
    console.log(`Guesser: I guess it's -> ${guess}`);
    return guess;
  }

  public async postAsync(
    sharedState: any,
    prepResult: { hint: string; pastGuesses: string[]; target: string } | null,
    execResult: string | null
  ): Promise<string> {
    if (!prepResult || execResult === null) {
      return "end";
    }

    // Check correctness
    const guessLower = execResult.trim().toLowerCase();
    const targetLower = prepResult.target.trim().toLowerCase();
    if (guessLower === targetLower) {
      console.log("Game Over -> Correct guess!");
      // Signal the hinter to stop
      const hinterQueue = sharedState.hinterQueue as string[];
      hinterQueue.push("GAME_OVER");
      return "end";
    }

    // If guess is wrong, add to pastGuesses
    if (!sharedState.past_guesses) {
      sharedState.past_guesses = [];
    }
    sharedState.past_guesses.push(execResult);

    // Send guess to the Hinter for feedback
    const hinterQueue = sharedState.hinterQueue as string[];
    hinterQueue.push(execResult);

    return "continue";
  }
}

// Example usage
(async () => {
  const shared = {
    target_word: "nostalgia",
    forbidden_words: ["memory", "past", "remember", "feeling", "longing"],
    hinterQueue: [] as string[],
    guesserQueue: [] as string[],
    past_guesses: [] as string[],
  };

  console.log("Game starting!");
  console.log(`Target word: ${shared.target_word}`);
  console.log(`Forbidden words: ${shared.forbidden_words}`);

  // Initialize by sending an empty guess to Hinter
  shared.hinterQueue.push("");

  const hinter = new AsyncHinter();
  const guesser = new AsyncGuesser();

  // In pocket.ts, you might have AsyncFlow (if your BaseNode variants are async).
  // For demonstration, assume Flow can handle async as well. 
  const hinterFlow = new Flow(hinter);
  const guesserFlow = new Flow(guesser);

  // Connect each node to itself to allow multiple turns
  hinter.addSuccessor(hinter, "continue");
  guesser.addSuccessor(guesser, "continue");

  // Start both flows concurrently
  // Typically you'd want a coordination mechanism like Promise.all or a dedicated runner
  hinterFlow.runAsync(shared).catch((err) => console.error("Hinter flow failed:", err));
  guesserFlow.runAsync(shared).catch((err) => console.error("Guesser flow failed:", err));
})();
```

**Explanation:**
1. **Queues**:  
   - `hinterQueue` carries guesses from the Guesser to the Hinter.  
   - `guesserQueue` carries hints from the Hinter to the Guesser.
2. **AsyncHinter**:
   - Awaits a guess. If `"GAME_OVER"`, the agent ends. Otherwise, it generates a new hint and puts it into `guesserQueue`.
3. **AsyncGuesser**:
   - Pulls a hint from `guesserQueue`, generates a guess, and checks if correct.  
   - If correct, ends the game; otherwise, pushes the guess back to `hinterQueue`.
4. **Loops**:  
   - Each agent calls `addSuccessor(node, "continue")` to keep running in a loop until the game finishes or no more messages are available.

---

### Building Multi-Agent Systems

- **Shared State**: Store data structures like queues, agent statuses, or global game state in `sharedState`.
- **Flow** or **AsyncFlow**: Each agent can be a node or sub-flow that loops for repeated interactions.
- **Communication**: Use shared structures (queues or specialized abstractions) to pass messages or signals between agents.

This design enables flexible *multi-agent* architectures, letting you break down complex tasks among multiple specialized agents.