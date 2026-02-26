# Workflow: Configure Memory

Guide the user through adding memory and RAG capabilities to their Mastra agents.

## Steps

### 1. Determine Memory Needs

Ask the user:
- Do you need conversation history? (message history)
- Do you need persistent state across messages? (working memory)
- Do you need to recall relevant past messages? (semantic recall)
- Do you need to process documents for context? (RAG)

### 2. Install Dependencies

```bash
# Core memory
npm install @mastra/memory

# Storage (pick one)
npm install @mastra/libsql    # Lightweight, file-based
npm install @mastra/pg        # PostgreSQL

# Vector store (for semantic recall or RAG)
npm install @mastra/pinecone  # Managed
npm install @mastra/pg        # pgvector (self-hosted)
npm install @mastra/qdrant    # High-performance

# RAG pipeline
npm install @mastra/rag
```

### 3. Configure Memory

```typescript
import { Agent } from "@mastra/core/agent";
import { Memory } from "@mastra/memory";
import { LibSQLStore } from "@mastra/libsql";
import { PineconeVector } from "@mastra/pinecone";
import { ModelRouterEmbeddingModel } from "@mastra/core/llm";

export const agent = new Agent({
  id: "memory-agent",
  model: "openai/gpt-5.1",
  instructions: "You are a helpful assistant with memory.",
  memory: new Memory({
    // Storage for messages and state
    storage: new LibSQLStore({ id: "store", url: "file:memory.db" }),

    // Message history
    lastMessages: 20,

    // Working memory (structured notepad)
    workingMemory: {
      enabled: true,
      template: `
## User Profile
- Name:
- Preferences:

## Current Task
- Goal:
- Status:
      `,
    },

    // Semantic recall (vector search)
    vector: new PineconeVector({ id: "vec", apiKey: process.env.PINECONE_API_KEY }),
    embedder: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
    semanticRecall: {
      enabled: true,
      topK: 3,
      messageRange: { before: 2, after: 1 },
    },
  }),
});

// Usage with threads
const response = await agent.generate("My name is Alex", {
  memory: { thread: "user-123", resource: "my-app" },
});
```

### 4. Set Up RAG Pipeline (Optional)

```typescript
import { MDocument } from "@mastra/rag";
import { embedMany } from "ai";
import { PgVector } from "@mastra/pg";
import { ModelRouterEmbeddingModel } from "@mastra/core/llm";

// 1. Initialize vector store
const pgVector = new PgVector({
  id: "pg-vec",
  connectionString: process.env.POSTGRES_CONNECTION_STRING,
});
await pgVector.createIndex({ indexName: "docs", dimension: 1536 });

// 2. Ingest documents
const doc = MDocument.fromText("Your document text...");
const chunks = await doc.chunk({ strategy: "recursive", size: 512, overlap: 50 });

const { embeddings } = await embedMany({
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
  values: chunks.map((c) => c.text),
});

await pgVector.upsert({
  indexName: "docs",
  vectors: embeddings,
  metadata: chunks.map((c) => ({ text: c.text })),
});

// 3. Create query tool
import { createVectorQueryTool } from "@mastra/rag";

const queryTool = createVectorQueryTool({
  vectorStoreName: "pgVector",
  indexName: "docs",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
});

// 4. Give to agent
const ragAgent = new Agent({
  id: "rag-agent",
  model: "openai/gpt-5.1",
  instructions: "Answer questions using the query tool to search documents.",
  tools: { queryTool },
});
```

### 5. Register Vector Stores

```typescript
import { Mastra } from "@mastra/core";

export const mastra = new Mastra({
  agents: { agent, ragAgent },
  vectors: { pgVector },
  storage: new LibSQLStore({ id: "store", url: "file:mastra.db" }),
});
```

## Advanced Patterns

### GraphRAG (Relationship Discovery)
```typescript
import { createGraphRAGTool } from "@mastra/rag";

const graphTool = createGraphRAGTool({
  vectorStoreName: "pgVector",
  indexName: "docs",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
  graphOptions: { dimension: 1536, threshold: 0.7 },
});
```

### Re-ranking Results
```typescript
import { rerankWithScorer, MastraAgentRelevanceScorer } from "@mastra/rag";

const reranked = await rerankWithScorer({
  results: initialResults,
  query: "search query",
  scorer: new MastraAgentRelevanceScorer("scorer", "openai/gpt-5.1"),
  options: { weights: { semantic: 0.5, vector: 0.3, position: 0.2 } },
});
```

### Observational Memory (Long Conversations)
```typescript
const memory = new Memory({
  options: {
    observationalMemory: {
      model: "google/gemini-2.5-flash",
      scope: "thread",
    },
  },
});
```

### Chunking Strategies
```typescript
// Recursive (default, smart splitting)
await doc.chunk({ strategy: "recursive", size: 512, overlap: 50 });

// Sentence-aware
await doc.chunk({ strategy: "sentence", maxSize: 450 });

// Semantic markdown (preserves relationships)
await doc.chunk({ strategy: "semantic-markdown", joinThreshold: 500 });

// Other: character, token, markdown, html, json, latex
```
