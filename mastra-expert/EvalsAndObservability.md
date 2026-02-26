# Mastra Evals & Observability

## Evals: createScorer()

```typescript
import { createScorer } from "@mastra/evals";
import { z } from "zod";

const myScorer = createScorer({
  id: "quality-scorer",
  description: "Evaluates response quality",
  model: "openai/gpt-5.1",
})
  .preprocess(({ run }) => ({ text: run.output.text.trim() }))
  .analyze({
    description: "Analyze quality",
    outputSchema: z.object({ quality: z.enum(["good", "bad"]) }),
    createPrompt: ({ results }) => `Analyze: ${results.preprocessStepResult.text}`,
  })
  .generateScore(({ results }) => (
    results.analyzeStepResult.quality === "good" ? 1 : 0
  ));
```

### Built-in Scorers

| Scorer | Package | Purpose |
|--------|---------|---------|
| `HallucinationScorer` | `@mastra/evals` | Detect fabricated info |
| `ToxicityScorer` | `@mastra/evals` | Detect harmful content |
| `FaithfulnessScorer` | `@mastra/evals` | Check source accuracy |
| `AnswerRelevancyScorer` | `@mastra/evals` | Measure relevance |
| `ContextPrecisionScorer` | `@mastra/evals` | RAG precision |
| `ContextRecallScorer` | `@mastra/evals` | RAG recall |
| `CompletenessScorer` | `@mastra/evals` | Response completeness |
| `ContentSimilarityScorer` | `@mastra/evals` | Text similarity |
| `CoverageScorer` | `@mastra/evals` | Topic coverage |
| `KeywordCoverageScorer` | `@mastra/evals` | Keyword presence |
| `PromptAlignmentScorer` | `@mastra/evals` | Instruction adherence |
| `SummarizationScorer` | `@mastra/evals` | Summary quality |
| `TextualDifferenceScorer` | `@mastra/evals` | Text diff scoring |
| `ToneConsistencyScorer` | `@mastra/evals` | Tone matching |
| `BiasScorer` | `@mastra/evals` | Bias detection |

### Using Built-in Scorers

```typescript
import { HallucinationScorer, ToxicityScorer } from "@mastra/evals";

const hallucination = new HallucinationScorer({
  model: "openai/gpt-5.1",
});

const result = await hallucination.score({
  input: "What is the capital of France?",
  output: "The capital of France is Paris.",
  context: "France is a country in Europe. Its capital is Paris.",
});

console.log(result.score); // 0-1 (lower = less hallucination)
```

### CI Integration

```typescript
// mastra.config.ts
import { createScorer } from "@mastra/evals";

export default {
  scorers: {
    qualityScorer: myScorer,
    hallucinationScorer: new HallucinationScorer({ model: "openai/gpt-5.1" }),
  },
};

// Run in CI
// npx mastra eval --agent weatherAgent
```

### Custom Scorer with Functions

```typescript
const lengthScorer = createScorer({
  id: "length-scorer",
  description: "Scores based on response length",
}).generateScore(({ run }) => {
  const length = run.output.text.length;
  return length > 100 ? 1 : length / 100;
});
```

## Observability: Logging

```typescript
import { Mastra } from "@mastra/core";
import { PinoLogger } from "@mastra/loggers";

const mastra = new Mastra({
  logger: new PinoLogger({ name: "MyApp", level: "info" }),
});

// Available levels: debug, info, warn, error
```

## Observability: Tracing

```typescript
import { Mastra } from "@mastra/core";
import { Observability } from "@mastra/core/observability";
import { DefaultExporter } from "@mastra/core/observability/exporters";
import { LangfuseExporter } from "@mastra/langfuse";

const mastra = new Mastra({
  observability: new Observability({
    configs: {
      default: {
        serviceName: "my-app",
        sampling: { type: "ratio", probability: 1.0 },
        exporters: [
          new DefaultExporter(),
          new LangfuseExporter({
            publicKey: process.env.LANGFUSE_PUBLIC_KEY,
            secretKey: process.env.LANGFUSE_SECRET_KEY,
            baseUrl: "https://cloud.langfuse.com",
          }),
        ],
      },
    },
  }),
});
```

### Tracing Exporters

| Exporter | Package | Platform |
|----------|---------|----------|
| `DefaultExporter` | `@mastra/core` | Mastra Studio |
| `CloudExporter` | `@mastra/core` | Mastra Cloud |
| `LangfuseExporter` | `@mastra/langfuse` | Langfuse |
| `LangSmithExporter` | `@mastra/langsmith` | LangSmith |
| `DatadogExporter` | `@mastra/datadog` | Datadog APM |
| `SentryExporter` | `@mastra/sentry` | Sentry |
| `PostHogExporter` | `@mastra/posthog` | PostHog |
| `ArizeExporter` | `@mastra/arize` | Arize Phoenix |
| `BraintrustExporter` | `@mastra/braintrust` | Braintrust |
| `LaminarExporter` | `@mastra/laminar` | Laminar |
| `OTELExporter` | `@mastra/core` | Any OTEL platform |

### Child Spans

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
      span?.end({ output: results, metadata: { rows: results.length } });
      return results;
    } catch (error) {
      span?.error({ error });
      throw error;
    }
  },
});
```

### Retrieving Trace IDs

```typescript
// From agent
const result = await agent.generate("Hello");
console.log(result.traceId);

// From stream
const stream = await agent.stream("Hello");
console.log(stream.traceId);

// From workflow
const result = await run.start({ inputData: { value: "test" } });
console.log(result.traceId);
```

### Flushing (Serverless)

```typescript
export async function POST(req: Request) {
  const result = await agent.generate([{ role: "user", content: await req.text() }]);
  await mastra.getObservability().flush(); // Ensure spans export before function ends
  return Response.json(result);
}
```

### Custom Span Formatters

```typescript
import { SpanType } from "@mastra/core/observability";

const formatter = (span) => {
  if (span.type === SpanType.AGENT_RUN && Array.isArray(span.input)) {
    const userMsg = span.input.find((m) => m.role === "user");
    return { ...span, input: userMsg?.content ?? span.input };
  }
  return span;
};

new LangfuseExporter({ customSpanFormatter: formatter });
```

### Serialization Options

```typescript
observability: new Observability({
  configs: {
    default: {
      serializationOptions: {
        maxStringLength: 2048,
        maxDepth: 10,
        maxArrayLength: 100,
        maxObjectKeys: 75,
      },
    },
  },
})
```
