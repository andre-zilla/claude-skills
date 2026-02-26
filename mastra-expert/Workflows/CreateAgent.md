# Workflow: Create Agent

Guide the user through creating a Mastra AI agent with tools, memory, and voice.

## Steps

### 1. Define the Agent's Purpose

Ask the user:
- What should the agent do?
- What tools does it need?
- Should it have memory (conversation history)?
- Does it need voice capabilities?

### 2. Create the Tool(s)

```typescript
// src/mastra/tools/my-tool.ts
import { createTool } from "@mastra/core/tools";
import { z } from "zod";

export const myTool = createTool({
  id: "my-tool",
  description: "Describe what this tool does clearly — the LLM reads this",
  inputSchema: z.object({
    query: z.string().describe("The search query"),
  }),
  outputSchema: z.object({
    result: z.string(),
  }),
  execute: async (inputData) => {
    // Tool implementation
    return { result: `Processed: ${inputData.query}` };
  },
});
```

### 3. Create the Agent

```typescript
// src/mastra/agents/my-agent.ts
import { Agent } from "@mastra/core/agent";
import { myTool } from "../tools/my-tool";

export const myAgent = new Agent({
  id: "my-agent",
  name: "My Agent",
  instructions: `
    You are a helpful assistant that [specific purpose].

    When responding:
    - [Guideline 1]
    - [Guideline 2]

    Use the myTool to [when to use].
  `,
  model: "openai/gpt-5.1",
  tools: { myTool },
});
```

### 4. Add Memory (Optional)

```typescript
import { Memory } from "@mastra/memory";
import { LibSQLStore } from "@mastra/libsql";

export const myAgent = new Agent({
  id: "my-agent",
  name: "My Agent",
  instructions: "...",
  model: "openai/gpt-5.1",
  tools: { myTool },
  memory: new Memory({
    storage: new LibSQLStore({ id: "store", url: "file:memory.db" }),
    lastMessages: 20,
    workingMemory: { enabled: true },
  }),
});
```

### 5. Add Voice (Optional)

```typescript
import { OpenAIVoice } from "@mastra/voice-openai";

export const myAgent = new Agent({
  id: "my-agent",
  // ... other config
  voice: new OpenAIVoice(),
});

// Usage
const audio = await myAgent.voice.speak("Hello!");
```

### 6. Register with Mastra

```typescript
// src/mastra/index.ts
import { Mastra } from "@mastra/core";
import { myAgent } from "./agents/my-agent";

export const mastra = new Mastra({
  agents: { myAgent },
});
```

### 7. Test

```bash
mastra dev  # Opens Studio for interactive testing
```

Or programmatically:
```typescript
const agent = mastra.getAgent("myAgent");
const response = await agent.generate("Hello!");
console.log(response.text);

// With memory
const response = await agent.generate("Remember my name is Alex", {
  memory: { thread: "user-123", resource: "my-app" },
});
```

## Common Patterns

### Agent with Structured Output
```typescript
const response = await agent.generate("Plan my day", {
  structuredOutput: {
    schema: z.object({ tasks: z.array(z.string()) }),
  },
});
console.log(response.object);
```

### Agent Network (Multi-Agent)
```typescript
const routerAgent = new Agent({
  id: "router",
  agents: { researchAgent, writingAgent },
  tools: { searchTool },
});

const result = await routerAgent.network("Research and write about AI");
```

### Agent with Processors (Guardrails)
```typescript
import { ModerationProcessor, PIIDetector } from "@mastra/core/processors";

const agent = new Agent({
  id: "safe-agent",
  inputProcessors: [new ModerationProcessor({ threshold: 0.7 })],
  outputProcessors: [new PIIDetector({ strategy: "redact" })],
});
```
