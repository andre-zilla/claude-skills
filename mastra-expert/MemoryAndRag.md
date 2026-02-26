# Mastra Memory & RAG

## Memory System Overview

Four memory types work together:

| Type | Purpose | Config |
|------|---------|--------|
| **Message History** | Recent conversation messages | `lastMessages: N` |
| **Working Memory** | Persistent structured state (like a notepad) | `workingMemory: { enabled: true }` |
| **Semantic Recall** | Vector search for relevant past messages | `semanticRecall: { enabled: true }` |
| **Observational** | Long-horizon compression (5-40x) | `observationalMemory: { model: "..." }` |

## Basic Memory Setup

```typescript
import { Agent } from "@mastra/core/agent";
import { Memory } from "@mastra/memory";
import { LibSQLStore } from "@mastra/libsql";

const agent = new Agent({
  id: "my-agent",
  model: "openai/gpt-5.1",
  memory: new Memory({
    storage: new LibSQLStore({ id: "store", url: "file:memory.db" }),
    lastMessages: 20,
  }),
});

// Use with threads
const response = await agent.generate("Remember my name is Alex", {
  memory: { thread: "user-123", resource: "my-app" },
});
```

## Working Memory

```typescript
const memory = new Memory({
  storage: new LibSQLStore({ id: "store", url: "file:memory.db" }),
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
});
```

## Semantic Recall

```typescript
import { PineconeVector } from "@mastra/pinecone";
import { ModelRouterEmbeddingModel } from "@mastra/core/llm";

const memory = new Memory({
  storage: new LibSQLStore({ id: "store", url: "file:memory.db" }),
  vector: new PineconeVector({ id: "vec", apiKey: process.env.PINECONE_API_KEY }),
  embedder: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
  semanticRecall: {
    enabled: true,
    topK: 3,
    messageRange: { before: 2, after: 1 },
  },
});
```

## Observational Memory

```typescript
const memory = new Memory({
  options: {
    observationalMemory: {
      model: "google/gemini-2.5-flash", // Best overall (1M context)
      scope: "thread",
      observation: { messageTokens: 30_000 },
      reflection: { observationTokens: 40_000 },
    },
  },
});
```

## Storage Adapters

```typescript
// PostgreSQL
import { PostgresStore } from "@mastra/pg";
const storage = new PostgresStore({ id: "pg", connectionString: process.env.DATABASE_URL });

// libSQL
import { LibSQLStore } from "@mastra/libsql";
const storage = new LibSQLStore({ id: "sql", url: "file:memory.db" });

// MongoDB
import { MongoDBStore } from "@mastra/mongodb";
const storage = new MongoDBStore({ id: "mongo", uri: process.env.MONGODB_URI });

// Instance-level (shared by all agents)
const mastra = new Mastra({
  storage: new PostgresStore({ id: "shared", connectionString: process.env.DATABASE_URL }),
});
```

## RAG Pipeline

### 1. Create Document & Chunk

```typescript
import { MDocument } from "@mastra/rag";

const doc = MDocument.fromText("Your document text...");
// Also: .fromHTML(), .fromMarkdown(), .fromJSON()

const chunks = await doc.chunk({
  strategy: "recursive",  // or "sentence", "semantic-markdown", "character", "token"
  size: 512,
  overlap: 50,
});
```

### 2. Generate Embeddings

```typescript
import { embedMany } from "ai";
import { ModelRouterEmbeddingModel } from "@mastra/core/llm";

const { embeddings } = await embedMany({
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
  values: chunks.map((c) => c.text),
});
```

### 3. Store in Vector Database

```typescript
import { PgVector } from "@mastra/pg";

const pgVector = new PgVector({
  id: "pg-vec",
  connectionString: process.env.POSTGRES_CONNECTION_STRING,
});

await pgVector.createIndex({ indexName: "embeddings", dimension: 1536 });

await pgVector.upsert({
  indexName: "embeddings",
  vectors: embeddings,
  metadata: chunks.map((c) => ({ text: c.text, source: "doc.txt" })),
});
```

### 4. Query

```typescript
import { embed } from "ai";

const { embedding } = await embed({
  value: "search query",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
});

const results = await pgVector.query({
  indexName: "embeddings",
  queryVector: embedding,
  topK: 10,
  filter: { source: "doc.txt" },
});
```

### 5. Re-rank (Optional)

```typescript
import { rerankWithScorer, MastraAgentRelevanceScorer } from "@mastra/rag";

const reranked = await rerankWithScorer({
  results: initialResults,
  query: "search query",
  scorer: new MastraAgentRelevanceScorer("scorer", "openai/gpt-5.1"),
  options: { weights: { semantic: 0.5, vector: 0.3, position: 0.2 }, topK: 5 },
});
```

## Vector Query Tool

```typescript
import { createVectorQueryTool } from "@mastra/rag";

const queryTool = createVectorQueryTool({
  vectorStoreName: "pgVector",
  indexName: "embeddings",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
});

const agent = new Agent({
  id: "rag-agent",
  tools: { queryTool },
  model: "openai/gpt-5.1",
});
```

## GraphRAG

```typescript
import { createGraphRAGTool } from "@mastra/rag";

const graphTool = createGraphRAGTool({
  vectorStoreName: "pgVector",
  indexName: "embeddings",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
  graphOptions: { dimension: 1536, threshold: 0.7 },
});

const agent = new Agent({
  id: "graph-agent",
  tools: { queryTool, graphTool }, // Combine both for best results
  model: "openai/gpt-5.1",
});
```

## Supported Vector Databases

| Provider | Package | Features |
|----------|---------|----------|
| PostgreSQL pgvector | `@mastra/pg` | Full-featured, self-hosted |
| Pinecone | `@mastra/pinecone` | Managed, namespaces |
| Qdrant | `@mastra/qdrant` | High-performance |
| Chroma | `@mastra/chroma` | Local or cloud |
| MongoDB Atlas | `@mastra/mongodb` | Integrated with MongoDB |
| Astra DB | `@mastra/astra` | Cassandra-based |
| libSQL | `@mastra/core/vector/libsql` | Embedded, lightweight |
| Upstash | `@mastra/upstash` | Serverless Redis-based |
| Cloudflare Vectorize | `@mastra/vectorize` | Edge-native |
| OpenSearch | `@mastra/opensearch` | Search-integrated |
| Elasticsearch | `@mastra/elasticsearch` | Enterprise search |
| Couchbase | `@mastra/couchbase` | Multi-model database |
| LanceDB | `@mastra/lance` | Embedded, columnar |
| S3 Vectors | `@mastra/s3vectors` | S3-native storage |

## Metadata Filtering

```typescript
// Simple equality
filter: { source: "article1.txt" }

// Numeric comparison
filter: { price: { $gt: 100 } }

// Array operations
filter: { tags: { $in: ["sale", "new"] } }

// Logical operators
filter: {
  $or: [{ category: "electronics" }, { category: "accessories" }],
  $and: [{ price: { $gt: 50 } }, { price: { $lt: 200 } }],
}
```
