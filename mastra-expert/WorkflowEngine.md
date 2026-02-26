# Mastra Workflow Engine

## Creating Workflows

```typescript
import { createWorkflow, createStep } from "@mastra/core/workflows";
import { z } from "zod";

const step1 = createStep({
  id: "format",
  inputSchema: z.object({ message: z.string() }),
  outputSchema: z.object({ formatted: z.string() }),
  execute: async ({ inputData }) => ({ formatted: inputData.message.toUpperCase() }),
});

const workflow = createWorkflow({
  id: "my-workflow",
  inputSchema: z.object({ message: z.string() }),
  outputSchema: z.object({ formatted: z.string() }),
})
  .then(step1)
  .commit();
```

## Control Flow

### Sequential: `.then()`
```typescript
workflow.then(step1).then(step2).then(step3).commit();
```

### Parallel: `.parallel()`
```typescript
workflow.parallel([step1, step2]).then(mergeStep).commit();
// Output keyed by step ID: { "step-1": {...}, "step-2": {...} }
```

### Branching: `.branch()`
```typescript
workflow
  .then(step1)
  .branch([
    [async ({ inputData }) => inputData.value > 10, highValueStep],
    [async ({ inputData }) => inputData.value <= 10, lowValueStep],
  ])
  .then(finalStep)
  .commit();
```

### Data Mapping: `.map()`
```typescript
workflow
  .then(step1)
  .map(async ({ inputData, getStepResult, getInitData }) => ({
    transformed: inputData.formatted,
  }))
  .then(step2)
  .commit();
```

### Looping: `.dountil()` / `.dowhile()`
```typescript
// Repeat UNTIL condition is true
workflow.then(step1).dountil(incrementStep, async ({ inputData }) => inputData.number > 10).commit();

// Repeat WHILE condition is true
workflow.then(step1).dowhile(incrementStep, async ({ inputData }) => inputData.number < 10).commit();
```

### Array Iteration: `.foreach()`
```typescript
workflow
  .foreach(processStep)                    // Sequential (default)
  .foreach(processStep, { concurrency: 4 }) // Parallel (up to 4)
  .then(aggregateStep)
  .commit();
```

### Nested Workflows
```typescript
const childWorkflow = createWorkflow({ id: "child", ... })
  .then(step1).then(step2).commit();

const parentWorkflow = createWorkflow({ id: "parent", ... })
  .then(childWorkflow).commit();

// Better than chained foreach for multi-step per item:
const batchWorkflow = createWorkflow({ id: "batch", ... })
  .foreach(childWorkflow, { concurrency: 3 }).commit();
```

## Running Workflows

```typescript
const workflow = mastra.getWorkflow("myWorkflow");
const run = await workflow.createRun();

// Start
const result = await run.start({ inputData: { message: "Hello" } });
if (result.status === "success") console.log(result.result);

// Stream
const stream = await run.stream({ inputData: { message: "Hello" } });
for await (const chunk of stream) console.log(chunk);

// Cancel
await run.cancel();

// Restart
const restarted = await run.restart();

// Time travel to specific step
const timeTraveled = await run.timeTravel({ step: "step-2" });
```

## Suspend & Resume (Human-in-the-Loop)

```typescript
const approvalStep = createStep({
  id: "approval",
  inputSchema: z.object({ email: z.string() }),
  outputSchema: z.object({ output: z.string() }),
  resumeSchema: z.object({ approved: z.boolean() }),
  suspendSchema: z.object({ reason: z.string() }),
  execute: async ({ inputData, resumeData, suspend }) => {
    if (!resumeData?.approved) {
      return await suspend({ reason: "Human approval required." });
    }
    return { output: `Email sent to ${inputData.email}` };
  },
});

// Check for suspension
const result = await run.start({ inputData: { email: "user@example.com" } });
if (result.status === "suspended") {
  const payload = result.steps[result.suspended[0][0]].suspendPayload;
  console.log(payload); // { reason: "Human approval required." }
}

// Resume
const resumed = await run.resume({
  step: "approval",
  resumeData: { approved: true },
});
```

## Workflow State

```typescript
const step = createStep({
  id: "counter",
  inputSchema: z.object({ message: z.string() }),
  outputSchema: z.object({ count: z.number() }),
  stateSchema: z.object({ counter: z.number() }),
  execute: async ({ inputData, state, setState }) => {
    setState({ ...state, counter: state.counter + 1 });
    return { count: state.counter + 1 };
  },
});

// Initialize state
const result = await run.start({
  inputData: { message: "Hello" },
  initialState: { counter: 0 },
});
```

## Error Handling & Retries

```typescript
// Workflow-level retries
const workflow = createWorkflow({
  retryConfig: { attempts: 5, delay: 2000 },
}).then(step1).commit();

// Step-level retries
const step = createStep({
  execute: async () => { /* ... */ },
  retries: 3,
});

// Early exit with bail()
const step = createStep({
  execute: async ({ bail }) => {
    return bail({ result: "bailed early" });
  },
});
```

## Agents & Tools in Workflows

```typescript
// Agent as step
const agentStep = createStep(testAgent);
// Default: inputSchema: { prompt: string }, outputSchema: { text: string }

// Agent with structured output
const agentStep = createStep(testAgent, {
  structuredOutput: { schema: z.object({ title: z.string(), tags: z.array(z.string()) }) },
});

// Tool as step
const toolStep = createStep(testTool);

// Call agent inside step
const step = createStep({
  execute: async ({ inputData, mastra }) => {
    const agent = mastra.getAgent("testAgent");
    const response = await agent.generate(`Process: ${inputData.message}`);
    return { result: response.text };
  },
});
```

## Streaming

```typescript
// Step writer
const step = createStep({
  execute: async ({ inputData, writer }) => {
    await writer?.write({ type: "progress", status: "pending" });
    const result = await processData(inputData);
    await writer?.write({ type: "progress", status: "done" });
    return { value: result };
  },
});

// Pipe agent stream to workflow writer
const step = createStep({
  execute: async ({ inputData, mastra, writer }) => {
    const agent = mastra?.getAgent("testAgent");
    const stream = await agent?.stream(`Process: ${inputData.value}`);
    await stream!.textStream.pipeTo(writer!);
    return { value: await stream!.text };
  },
});
```

## RequestContext in Workflows

```typescript
const step = createStep({
  execute: async ({ requestContext }) => {
    const tier = requestContext.get("user-tier");
    const maxResults = tier === "enterprise" ? 1000 : 50;
    return { maxResults };
  },
});
```

## Workflow Result Structure

```typescript
{
  status: "success" | "failed" | "suspended" | "tripwire" | "paused",
  input: { message: "Hello" },
  steps: {
    "step-1": { status: "success", payload: {...}, output: {...} },
    "step-2": { status: "success", payload: {...}, output: {...} },
  },
  result: { formatted: "HELLO" },
  state: { counter: 1 },
}
```
