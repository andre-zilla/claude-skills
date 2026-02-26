# Workflow: Setup Project

Guide the user through setting up a new Mastra project from scratch.

## Option A: Quick Start (Recommended)

```bash
npx create-mastra@latest
```

This scaffolds a complete project with:
- TypeScript configuration
- Agent + tool templates
- Mastra entry point
- Dev scripts

## Option B: Manual Installation

### 1. Initialize Project

```bash
mkdir my-mastra-app && cd my-mastra-app
npm init -y
```

### 2. Install Dependencies

```bash
# npm
npm install @mastra/core@latest zod@^4
npm install -D typescript @types/node mastra@latest

# pnpm
pnpm add @mastra/core@latest zod@^4
pnpm add -D typescript @types/node mastra@latest

# bun
bun add @mastra/core@latest zod@^4
bun add --dev typescript @types/node mastra@latest
```

### 3. Configure TypeScript

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true,
    "outDir": "dist"
  },
  "include": ["src/**/*"]
}
```

**Important:** Mastra requires `module: "ES2022"` and `moduleResolution: "bundler"`. Using `CommonJS` or `node` will cause errors.

### 4. Add Scripts

```json
// package.json
{
  "scripts": {
    "dev": "mastra dev",
    "build": "mastra build",
    "start": "mastra start"
  }
}
```

### 5. Create .env

```bash
# .env
OPENAI_API_KEY=sk-...
# Or any supported provider key
```

### 6. Create Project Structure

```
src/mastra/
├── index.ts          # Mastra entry point
├── agents/
│   └── my-agent.ts   # Agent definition
└── tools/
    └── my-tool.ts    # Tool definition
```

**Tool:**
```typescript
// src/mastra/tools/my-tool.ts
import { createTool } from "@mastra/core/tools";
import { z } from "zod";

export const myTool = createTool({
  id: "my-tool",
  description: "Does something useful",
  inputSchema: z.object({ input: z.string() }),
  outputSchema: z.object({ output: z.string() }),
  execute: async (inputData) => ({ output: `Result: ${inputData.input}` }),
});
```

**Agent:**
```typescript
// src/mastra/agents/my-agent.ts
import { Agent } from "@mastra/core/agent";
import { myTool } from "../tools/my-tool";

export const myAgent = new Agent({
  id: "my-agent",
  name: "My Agent",
  instructions: "You are a helpful assistant. Use myTool when needed.",
  model: "openai/gpt-5.1",
  tools: { myTool },
});
```

**Entry Point:**
```typescript
// src/mastra/index.ts
import { Mastra } from "@mastra/core";
import { myAgent } from "./agents/my-agent";

export const mastra = new Mastra({
  agents: { myAgent },
});
```

### 7. Run

```bash
npm run dev  # Starts dev server + Studio
```

## Adding Common Features

### Add Storage
```bash
npm install @mastra/libsql
```

```typescript
import { LibSQLStore } from "@mastra/libsql";

export const mastra = new Mastra({
  agents: { myAgent },
  storage: new LibSQLStore({ id: "store", url: "file:mastra.db" }),
});
```

### Add Memory
```bash
npm install @mastra/memory
```

```typescript
import { Memory } from "@mastra/memory";

const agent = new Agent({
  id: "my-agent",
  memory: new Memory({ lastMessages: 20 }),
});
```

### Add Vector Store (for RAG)
```bash
npm install @mastra/pg  # or @mastra/pinecone, @mastra/qdrant, etc.
```

### Add Evals
```bash
npm install @mastra/evals
```

### Add Voice
```bash
npm install @mastra/voice-openai @mastra/node-audio
```

## Framework Integration

### Next.js
```typescript
// next.config.ts
const config: NextConfig = {
  serverExternalPackages: ["@mastra/*"],
};
```

### Astro
```typescript
// astro.config.mjs
import vercel from "@astrojs/vercel";
export default defineConfig({ adapter: vercel(), output: "server" });
```
