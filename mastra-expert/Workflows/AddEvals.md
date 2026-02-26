# Workflow: Add Evals

Guide the user through adding evaluation scorers and observability to their Mastra app.

## Steps

### 1. Install Dependencies

```bash
# Evals
npm install @mastra/evals

# Observability (pick exporters)
npm install @mastra/langfuse    # Langfuse
npm install @mastra/langsmith   # LangSmith
npm install @mastra/datadog     # Datadog
npm install @mastra/sentry      # Sentry
npm install @mastra/posthog     # PostHog
```

### 2. Create Custom Scorer

```typescript
// src/mastra/scorers/quality-scorer.ts
import { createScorer } from "@mastra/evals";
import { z } from "zod";

export const qualityScorer = createScorer({
  id: "quality-scorer",
  description: "Evaluates response quality",
  model: "openai/gpt-5.1",
})
  .preprocess(({ run }) => ({
    text: run.output.text.trim(),
    length: run.output.text.length,
  }))
  .analyze({
    description: "Analyze response quality",
    outputSchema: z.object({
      isHelpful: z.boolean(),
      isAccurate: z.boolean(),
      reasoning: z.string(),
    }),
    createPrompt: ({ results }) =>
      `Evaluate this response for helpfulness and accuracy:\n\n${results.preprocessStepResult.text}`,
  })
  .generateScore(({ results }) => {
    const { isHelpful, isAccurate } = results.analyzeStepResult;
    if (isHelpful && isAccurate) return 1;
    if (isHelpful || isAccurate) return 0.5;
    return 0;
  });
```

### 3. Use Built-in Scorers

```typescript
import {
  HallucinationScorer,
  ToxicityScorer,
  FaithfulnessScorer,
  AnswerRelevancyScorer,
} from "@mastra/evals";

const hallucination = new HallucinationScorer({ model: "openai/gpt-5.1" });
const toxicity = new ToxicityScorer({ model: "openai/gpt-5.1" });

// Score a response
const result = await hallucination.score({
  input: "What is the capital of France?",
  output: "The capital of France is Paris.",
  context: "France is a European country. Its capital is Paris.",
});

console.log(result.score); // 0-1
```

### 4. Register Scorers

```typescript
import { Mastra } from "@mastra/core";

export const mastra = new Mastra({
  agents: { myAgent },
  scorers: {
    qualityScorer,
    hallucinationScorer: new HallucinationScorer({ model: "openai/gpt-5.1" }),
    toxicityScorer: new ToxicityScorer({ model: "openai/gpt-5.1" }),
  },
});
```

### 5. Run Evals in CI

```bash
npx mastra eval --agent myAgent
```

### 6. Set Up Observability

```typescript
import { Observability } from "@mastra/core/observability";
import { DefaultExporter } from "@mastra/core/observability/exporters";
import { LangfuseExporter } from "@mastra/langfuse";

export const mastra = new Mastra({
  agents: { myAgent },
  observability: new Observability({
    configs: {
      default: {
        serviceName: "my-app",
        sampling: { type: "ratio", probability: 1.0 },
        exporters: [
          new DefaultExporter(),  // For Mastra Studio
          new LangfuseExporter({
            publicKey: process.env.LANGFUSE_PUBLIC_KEY,
            secretKey: process.env.LANGFUSE_SECRET_KEY,
            baseUrl: "https://cloud.langfuse.com",
          }),
        ],
      },
    },
  }),
  logger: new PinoLogger({ name: "MyApp", level: "info" }),
});
```

### 7. Access Traces

```typescript
// From agent responses
const result = await agent.generate("Hello");
console.log("Trace ID:", result.traceId);

// From streams
const stream = await agent.stream("Hello");
console.log("Trace ID:", stream.traceId);

// Flush in serverless
await mastra.getObservability().flush();
```

## Built-in Scorer Reference

| Scorer | Purpose | Input |
|--------|---------|-------|
| `HallucinationScorer` | Detect fabricated info | input, output, context |
| `ToxicityScorer` | Detect harmful content | output |
| `FaithfulnessScorer` | Check source accuracy | input, output, context |
| `AnswerRelevancyScorer` | Measure relevance | input, output |
| `ContextPrecisionScorer` | RAG precision | input, output, context |
| `ContextRecallScorer` | RAG recall | input, output, context |
| `CompletenessScorer` | Response completeness | input, output |
| `ContentSimilarityScorer` | Text similarity | output, expected |
| `CoverageScorer` | Topic coverage | input, output |
| `PromptAlignmentScorer` | Instruction adherence | input, output, instructions |
| `SummarizationScorer` | Summary quality | input, output |
| `BiasScorer` | Bias detection | output |
| `ToneConsistencyScorer` | Tone matching | output, expected_tone |

## Custom Child Spans

```typescript
const tool = createTool({
  execute: async (input, context) => {
    const span = context?.tracingContext.currentSpan?.createChildSpan({
      type: "generic",
      name: "db-query",
      input: { query: input.query },
    });
    try {
      const results = await db.query(input.query);
      span?.end({ output: results });
      return results;
    } catch (error) {
      span?.error({ error });
      throw error;
    }
  },
});
```
