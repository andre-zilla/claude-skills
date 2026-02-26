# Mastra Agents

## Agent Class

```typescript
import { Agent } from "@mastra/core/agent";

const agent = new Agent({
  id: "my-agent",
  name: "My Agent",
  instructions: "You are a helpful assistant.",
  model: "openai/gpt-5.1",
  tools: { weatherTool, searchTool },
  memory: new Memory({ storage: new LibSQLStore({ id: "store", url: "file:memory.db" }) }),
  voice: new OpenAIVoice(),
  inputProcessors: [new ModerationProcessor()],
  outputProcessors: [new PIIDetector()],
});
```

### Key Methods

```typescript
// Generate text
const response = await agent.generate("Hello");
console.log(response.text);

// Stream text
const stream = await agent.stream("Tell me a story");
for await (const chunk of stream.fullStream) {
  if (chunk.type === "text-delta") console.log(chunk.payload.delta);
}

// With memory (thread-based)
const response = await agent.generate("Remember my name is Alex", {
  memory: { thread: "user-123", resource: "app" },
});

// Structured output
const response = await agent.generate("Plan my day", {
  structuredOutput: {
    schema: z.array(z.object({ name: z.string(), activities: z.array(z.string()) })),
  },
});
console.log(response.object);

// With max steps
const response = await agent.generate("Research this topic", { maxSteps: 10 });
```

### Dynamic Instructions

```typescript
const agent = new Agent({
  id: "dynamic-agent",
  instructions: async ({ requestContext }) => {
    const tier = requestContext?.get("user-tier");
    return tier === "enterprise"
      ? "You are a premium support agent. Provide detailed responses."
      : "You are a helpful assistant. Be concise.";
  },
  model: "openai/gpt-5.1",
});
```

## Agent Networks

Route tasks across multiple agents, tools, and workflows:

```typescript
import { Agent } from "@mastra/core/agent";

const routingAgent = new Agent({
  id: "routing-agent",
  name: "Routing Agent",
  instructions: "You are a network of writers and researchers.",
  model: "openai/gpt-5.1",
  agents: { researchAgent, writingAgent },
  workflows: { cityWorkflow },
  tools: { weatherTool },
  memory: new Memory({ storage: new LibSQLStore({ id: "store", url: "file:mastra.db" }) }),
});

// Call network (streaming)
const result = await routingAgent.network("Tell me about dolphins");
for await (const chunk of result) {
  if (chunk.type === "network-execution-event-step-finish") {
    console.log(chunk.payload.result);
  }
}
```

## Processors (Guardrails)

### Input Processors (before LLM)

```typescript
import {
  UnicodeNormalizer,
  ModerationProcessor,
  PromptInjectionDetector,
  PIIDetector,
  LanguageDetector,
} from "@mastra/core/processors";

const agent = new Agent({
  id: "secure-agent",
  inputProcessors: [
    new UnicodeNormalizer({ stripControlChars: true }),
    new PromptInjectionDetector({ threshold: 0.8, strategy: "rewrite" }),
    new PIIDetector({ strategy: "redact", detectionTypes: ["email", "phone"] }),
    new ModerationProcessor({ threshold: 0.7, strategy: "block" }),
  ],
});
```

### Output Processors (after LLM)

```typescript
import {
  BatchPartsProcessor,
  TokenLimiterProcessor,
  SystemPromptScrubber,
} from "@mastra/core/processors";

const agent = new Agent({
  id: "filtered-agent",
  outputProcessors: [
    new TokenLimiterProcessor({ limit: 1000, strategy: "truncate" }),
    new SystemPromptScrubber({ strategy: "redact" }),
  ],
});
```

### Custom Processor

```typescript
import type { Processor, MastraDBMessage } from "@mastra/core";

class CustomProcessor implements Processor {
  id = "custom";
  async processInput({ messages }) {
    return messages.map((msg) => ({
      ...msg,
      content: { ...msg.content, content: msg.content.content.toLowerCase() },
    }));
  }
}
```

### Per-Step Processing

```typescript
class DynamicModelProcessor implements Processor {
  id = "dynamic-model";
  async processInputStep({ stepNumber, model }) {
    if (stepNumber === 0) return { model: "openai/gpt-4o-mini" };
    if (stepNumber > 5) return { toolChoice: "none" };
    return {};
  }
}
```

### Tripwire Handling

```typescript
const result = await agent.generate("Is this credit card valid?: 4543...");
if (result.tripwire) {
  console.error("Blocked:", result.tripwire.reason);
  console.error("Processor:", result.tripwire.processorId);
}
```

## Tool Approval & Suspension

### Agent-Level Approval

```typescript
const stream = await agent.stream("What's the weather?", { requireToolApproval: true });

for await (const chunk of stream.fullStream) {
  if (chunk.type === "tool-call-approval") console.log("Approval required.");
}

// Approve or decline
const approved = await agent.approveToolCall({ runId: stream.runId });
const declined = await agent.declineToolCall({ runId: stream.runId });
```

### Tool Suspension

```typescript
const tool = createTool({
  id: "weather",
  inputSchema: z.object({ location: z.string() }),
  outputSchema: z.object({ weather: z.string() }),
  resumeSchema: z.object({ approved: z.boolean() }),
  suspendSchema: z.object({ reason: z.string() }),
  execute: async (input, context) => {
    const { resumeData, suspend } = context?.agent ?? {};
    if (!resumeData?.approved) {
      return suspend?.({ reason: "Approval required." });
    }
    return { weather: await fetchWeather(input.location) };
  },
});

// Resume suspended tool
const resumed = await agent.resumeStream({ approved: true }, { runId: stream.runId });
```

## Structured Output

```typescript
// Zod schema
const response = await agent.generate("Plan my day", {
  structuredOutput: {
    schema: z.object({ tasks: z.array(z.string()), priority: z.number() }),
  },
});
console.log(response.object);

// Streaming structured output
const stream = await agent.stream("Plan my day", {
  structuredOutput: {
    schema: z.object({ tasks: z.array(z.string()) }),
  },
});
console.log(await stream.object);

// Separate structuring model
const response = await agent.generate("Analyze TypeScript", {
  structuredOutput: {
    schema: z.object({ strengths: z.array(z.string()) }),
    model: "openai/gpt-4o",
  },
});
```

## Image Analysis

```typescript
const response = await agent.generate([{
  role: "user",
  content: [
    { type: "image", image: "https://example.com/photo.jpg", mimeType: "image/jpeg" },
    { type: "text", text: "Describe this image." },
  ],
}]);
```

## Registration

```typescript
import { Mastra } from "@mastra/core";

export const mastra = new Mastra({
  agents: { myAgent, routingAgent },
});

const agent = mastra.getAgent("myAgent");
```
