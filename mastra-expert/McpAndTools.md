# Mastra MCP & Tools

## MCPClient — Connecting to MCP Servers

```typescript
import { MCPClient } from "@mastra/mcp";

const mcp = new MCPClient({
  id: "my-mcp",
  servers: {
    // Stdio transport
    wikipedia: {
      command: "npx",
      args: ["-y", "wikipedia-mcp"],
    },
    // HTTP/SSE transport
    weather: {
      url: new URL("https://server.smithery.ai/@weather/mcp"),
    },
  },
});
```

### Static Tools (Top-Level Await)

```typescript
const agent = new Agent({
  id: "mcp-agent",
  model: "openai/gpt-5.1",
  tools: await mcp.listTools(),
});
```

### Dynamic Tools (Per-Request)

```typescript
async function handleRequest(prompt: string, apiKey: string) {
  const userMcp = new MCPClient({
    servers: {
      api: {
        url: new URL("http://localhost:8080/mcp"),
        requestInit: { headers: { Authorization: `Bearer ${apiKey}` } },
      },
    },
  });

  const response = await agent.generate(prompt, {
    toolsets: await userMcp.listToolsets(),
  });

  await userMcp.disconnect();
  return response;
}
```

### Resources

```typescript
const resources = await mcp.resources.list();
const content = await mcp.resources.read("weatherServer", "weather://current");

await mcp.resources.subscribe("weatherServer", "weather://current");
mcp.resources.onUpdated("weatherServer", (params) => {
  console.log(`Updated: ${params.uri}`);
});
```

### Prompts

```typescript
const prompts = await mcp.prompts.list();
const { prompt, messages } = await mcp.prompts.get({
  serverName: "myServer",
  name: "analyze-code",
  args: { code: "function hello() {}" },
});
```

### Elicitation (Interactive Input)

```typescript
await mcp.elicitation.onRequest("server", async (request) => {
  const schema = request.requestedSchema;
  // Collect user input based on schema...
  return { action: "accept", content: { name: "John", email: "john@example.com" } };
  // Or: { action: "decline" } or { action: "cancel" }
});
```

### OAuth

```typescript
import { MCPClient, MCPOAuthClientProvider } from "@mastra/mcp";

const oauthProvider = new MCPOAuthClientProvider({
  redirectUrl: "http://localhost:3000/oauth/callback",
  clientMetadata: {
    redirect_uris: ["http://localhost:3000/oauth/callback"],
    client_name: "My MCP Client",
    grant_types: ["authorization_code", "refresh_token"],
  },
  onRedirectToAuthorization: (url) => console.log(`Visit: ${url}`),
});

const client = new MCPClient({
  servers: {
    protected: {
      url: new URL("https://mcp.example.com/mcp"),
      authProvider: oauthProvider,
    },
  },
});
```

## MCPServer — Exposing as MCP

```typescript
import { MCPServer } from "@mastra/mcp";

const server = new MCPServer({
  id: "my-server",
  name: "My Server",
  version: "1.0.0",
  tools: { weatherTool },
  agents: { myAgent },    // Becomes tool "ask_myAgent"
  resources: myResourceHandlers,
  prompts: myPromptHandlers,
});

// Transport options
await server.startStdio();                                    // Stdio
server.startSSE({ url, ssePath: "/sse", messagePath: "/message", req, res }); // SSE
server.startHTTP({ url, httpPath: "/mcp", req, res });        // HTTP
server.startHTTP({ url, httpPath: "/mcp", req, res, options: { serverless: true } }); // Serverless
```

### Server Resources

```typescript
const resources = {
  listResources: async () => [
    { uri: "file://data/123.txt", name: "Data File", mimeType: "text/plain" },
  ],
  getResourceContent: async ({ uri }) => ({ text: "File content here" }),
  resourceTemplates: async () => [
    { uriTemplate: "file://data/{id}", name: "Data File" },
  ],
};
```

### Server OAuth Protection

```typescript
import { createOAuthMiddleware, createStaticTokenValidator } from "@mastra/mcp";

const oauth = createOAuthMiddleware({
  oauth: {
    resource: "https://mcp.example.com/mcp",
    validateToken: createStaticTokenValidator(["allowed-token"]),
  },
  mcpPath: "/mcp",
});
```

## createTool()

```typescript
import { createTool } from "@mastra/core/tools";
import { z } from "zod";

const tool = createTool({
  id: "reverse-string",
  description: "Reverses the input string",
  inputSchema: z.object({ input: z.string() }),
  outputSchema: z.object({ output: z.string() }),
  execute: async (inputData) => ({
    output: inputData.input.split("").reverse().join(""),
  }),
});
```

### Tool with MCP Annotations

```typescript
const tool = createTool({
  id: "weather",
  description: "Get weather",
  inputSchema: z.object({ location: z.string() }),
  mcp: {
    annotations: {
      title: "Weather Lookup",
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true,
    },
  },
  execute: async (input) => fetchWeather(input.location),
});
```

### Tool Lifecycle Hooks

```typescript
const tool = createTool({
  id: "example",
  onInputStart: ({ toolCallId }) => console.log("Started"),
  onInputDelta: ({ delta }) => console.log("Chunk:", delta),
  onInputAvailable: ({ input }) => console.log("Input:", input),
  execute: async (input) => ({ result: "done" }),
  onOutput: ({ output }) => console.log("Output:", output),
});
```

### Tool Approval

```typescript
const tool = createTool({
  id: "sensitive-op",
  requireApproval: true,
  inputSchema: z.object({ action: z.string() }),
  execute: async (input) => ({ success: true }),
});
```

### Tool Suspension

```typescript
const tool = createTool({
  id: "weather",
  inputSchema: z.object({ location: z.string() }),
  resumeSchema: z.object({ approved: z.boolean() }),
  suspendSchema: z.object({ reason: z.string() }),
  execute: async (input, context) => {
    const { resumeData, suspend } = context?.agent ?? {};
    if (!resumeData?.approved) {
      return suspend?.({ reason: "Approval required" });
    }
    return { weather: await fetchWeather(input.location) };
  },
});
```

## Vector Query Tool

```typescript
import { createVectorQueryTool } from "@mastra/rag";
import { ModelRouterEmbeddingModel } from "@mastra/core/llm";

const queryTool = createVectorQueryTool({
  vectorStoreName: "pgVector",
  indexName: "embeddings",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
});

// With reranking
const queryTool = createVectorQueryTool({
  vectorStoreName: "pgVector",
  indexName: "embeddings",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
  reranker: {
    model: "openai/gpt-5.1",
    options: { weights: { semantic: 0.5, vector: 0.3, position: 0.2 }, topK: 5 },
  },
});
```

## GraphRAG Tool

```typescript
import { createGraphRAGTool } from "@mastra/rag";

const graphTool = createGraphRAGTool({
  vectorStoreName: "pgVector",
  indexName: "embeddings",
  model: new ModelRouterEmbeddingModel("openai/text-embedding-3-small"),
  graphOptions: { dimension: 1536, threshold: 0.7 },
});
```
