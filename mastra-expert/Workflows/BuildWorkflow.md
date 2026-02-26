# Workflow: Build Workflow

Guide the user through creating a Mastra workflow with steps, control flow, and human-in-the-loop.

## Steps

### 1. Define the Workflow Purpose

Ask the user:
- What multi-step process should this automate?
- Are steps sequential, parallel, or conditional?
- Does any step need human approval (suspend/resume)?
- Should it call agents or tools?

### 2. Create Steps

```typescript
// src/mastra/workflows/steps.ts
import { createStep } from "@mastra/core/workflows";
import { z } from "zod";

export const fetchStep = createStep({
  id: "fetch-data",
  inputSchema: z.object({ query: z.string() }),
  outputSchema: z.object({ data: z.string() }),
  execute: async ({ inputData }) => {
    const response = await fetch(`https://api.example.com?q=${inputData.query}`);
    return { data: await response.text() };
  },
});

export const processStep = createStep({
  id: "process-data",
  inputSchema: z.object({ data: z.string() }),
  outputSchema: z.object({ result: z.string() }),
  execute: async ({ inputData }) => {
    return { result: inputData.data.toUpperCase() };
  },
});
```

### 3. Compose the Workflow

```typescript
// src/mastra/workflows/my-workflow.ts
import { createWorkflow } from "@mastra/core/workflows";
import { z } from "zod";
import { fetchStep, processStep } from "./steps";

export const myWorkflow = createWorkflow({
  id: "my-workflow",
  inputSchema: z.object({ query: z.string() }),
  outputSchema: z.object({ result: z.string() }),
})
  .then(fetchStep)
  .then(processStep)
  .commit();
```

### 4. Register and Run

```typescript
// src/mastra/index.ts
import { Mastra } from "@mastra/core";
import { myWorkflow } from "./workflows/my-workflow";

export const mastra = new Mastra({
  workflows: { myWorkflow },
});

// Run
const workflow = mastra.getWorkflow("myWorkflow");
const run = await workflow.createRun();
const result = await run.start({ inputData: { query: "hello" } });

if (result.status === "success") {
  console.log(result.result);
}
```

## Control Flow Patterns

### Parallel Steps
```typescript
const workflow = createWorkflow({ ... })
  .parallel([enrichStep, validateStep])
  .then(mergeStep)  // Receives { "enrich": {...}, "validate": {...} }
  .commit();
```

### Conditional Branching
```typescript
const workflow = createWorkflow({ ... })
  .then(classifyStep)
  .branch([
    [async ({ inputData }) => inputData.type === "urgent", urgentStep],
    [async ({ inputData }) => inputData.type === "normal", normalStep],
  ])
  .then(finalStep)
  .commit();
```

### Loop Until Done
```typescript
const workflow = createWorkflow({ ... })
  .dountil(retryStep, async ({ inputData }) => inputData.success === true)
  .commit();
```

### Array Processing
```typescript
const workflow = createWorkflow({ ... })
  .foreach(processItemStep, { concurrency: 4 })
  .then(aggregateStep)
  .commit();
```

### Human-in-the-Loop (Suspend/Resume)
```typescript
const approvalStep = createStep({
  id: "approval",
  inputSchema: z.object({ action: z.string() }),
  outputSchema: z.object({ result: z.string() }),
  resumeSchema: z.object({ approved: z.boolean() }),
  suspendSchema: z.object({ reason: z.string() }),
  execute: async ({ inputData, resumeData, suspend }) => {
    if (!resumeData?.approved) {
      return await suspend({ reason: `Approve: ${inputData.action}?` });
    }
    return { result: `Executed: ${inputData.action}` };
  },
});

// Resume later
const resumed = await run.resume({
  step: "approval",
  resumeData: { approved: true },
});
```

### Using Agents in Workflows
```typescript
// Agent as a step
const agentStep = createStep(myAgent);

// Or call inside a step
const step = createStep({
  execute: async ({ inputData, mastra }) => {
    const agent = mastra.getAgent("myAgent");
    const response = await agent.generate(`Analyze: ${inputData.text}`);
    return { analysis: response.text };
  },
});
```

### Streaming
```typescript
const step = createStep({
  execute: async ({ inputData, writer }) => {
    await writer?.write({ type: "progress", percent: 50 });
    // ... process ...
    await writer?.write({ type: "progress", percent: 100 });
    return { done: true };
  },
});

const stream = await run.stream({ inputData: { query: "test" } });
for await (const chunk of stream) {
  console.log(chunk);
}
```
