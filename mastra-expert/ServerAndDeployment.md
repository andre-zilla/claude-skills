# Mastra Server & Deployment

## Server Configuration

```typescript
import { Mastra } from "@mastra/core";

export const mastra = new Mastra({
  agents: { weatherAgent },
  workflows: { processWorkflow },
  server: {
    port: 3000,
    host: "0.0.0.0",
    auth: new MastraJwtAuth({ secret: process.env.MASTRA_JWT_SECRET }),
    middleware: [
      {
        path: "/api/*",
        handler: async (c, next) => {
          console.log(`${c.req.method} ${c.req.url}`);
          await next();
        },
      },
    ],
    apiRoutes: [/* custom routes */],
    build: { swaggerUI: true, openAPIDocs: true },
  },
});
```

## Authentication Providers

### JWT
```typescript
import { MastraJwtAuth } from "@mastra/auth";

auth: new MastraJwtAuth({ secret: process.env.MASTRA_JWT_SECRET })
```

### Supabase
```typescript
import { MastraAuthSupabase } from "@mastra/auth-supabase";

auth: new MastraAuthSupabase({
  url: process.env.SUPABASE_URL,
  anonKey: process.env.SUPABASE_ANON_KEY,
})
```

### Other Providers
- **Auth0**: `@mastra/auth-auth0`
- **Clerk**: `@mastra/auth-clerk`
- **Firebase**: `@mastra/auth-firebase`
- **WorkOS**: `@mastra/auth-workos`
- **Better Auth**: `@mastra/auth-better-auth`
- **Simple Auth**: Token-to-user mapping for API keys
- **Composite Auth**: Combine multiple providers

## MastraClient

```typescript
import { MastraClient } from "@mastra/client-js";

const client = new MastraClient({
  baseUrl: "http://localhost:4111",
  headers: { Authorization: `Bearer ${token}` },
});

// Agent operations
const agent = client.getAgent("weatherAgent");
const response = await agent.generate("What's the weather?");
const stream = await agent.stream("Hello");

// Workflow operations
const workflow = client.getWorkflow("processWorkflow");
```

### Client Tools (Browser-side)

```typescript
import { createTool } from "@mastra/client-js";

const colorTool = createTool({
  id: "color-change",
  description: "Changes background color",
  inputSchema: z.object({ color: z.string() }),
  outputSchema: z.object({ success: z.boolean() }),
  execute: async ({ color }) => {
    document.body.style.backgroundColor = color;
    return { success: true };
  },
});

const response = await agent.generate("Change background to blue", {
  clientTools: { colorTool },
});
```

## Custom API Routes

```typescript
import { registerApiRoute } from "@mastra/core/server";

apiRoutes: [
  registerApiRoute("/my-route", {
    method: "GET",
    handler: async (c) => {
      const mastra = c.get("mastra");
      return c.json({ message: "Hello" });
    },
  }),

  // Public route (no auth)
  registerApiRoute("/webhooks/github", {
    method: "POST",
    requiresAuth: false,
    handler: async (c) => c.json({ received: true }),
  }),

  // With OpenAPI docs
  registerApiRoute("/items/:id", {
    method: "GET",
    openapi: {
      summary: "Get item",
      tags: ["Items"],
      parameters: [{ name: "id", in: "path", required: true }],
    },
    handler: async (c) => c.json({ id: c.req.param("id") }),
  }),
]
```

## RequestContext

```typescript
// Set in middleware
middleware: [
  async (c, next) => {
    const ctx = c.get("requestContext");
    ctx.set("user-tier", "enterprise");
    await next();
  },
]

// Access in agent
const agent = new Agent({
  model: ({ requestContext }) => {
    return requestContext.get("user-tier") === "enterprise"
      ? "openai/gpt-5" : "openai/gpt-4o-mini";
  },
});

// Access in tools
execute: async (input, context) => {
  const tier = context?.requestContext?.get("user-tier");
}
```

### User Isolation

```typescript
import { MASTRA_RESOURCE_ID_KEY } from "@mastra/core/request-context";

middleware: [{
  path: "/api/*",
  handler: async (c, next) => {
    const ctx = c.get("requestContext");
    const user = ctx.get("user");
    if (!user) return c.json({ error: "Unauthorized" }, 401);
    ctx.set(MASTRA_RESOURCE_ID_KEY, user.id); // Force user isolation
    return next();
  },
}]
```

## Build & Deploy

### Build
```bash
mastra build          # Build for production
mastra start          # Start production server
node .mastra/output/index.mjs  # Direct start
```

### Build Output
```
.mastra/output/
├── index.mjs         # Server entry point
├── mastra.mjs        # Bundled config
├── tools.mjs         # Tools
├── package.json      # Production deps
└── playground/       # Studio UI
```

### Environment Variables
| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | 4111 | Server port |
| `MASTRA_STUDIO_PATH` | `./playground` | Studio build dir |
| `MASTRA_SKIP_DOTENV` | false | Skip .env loading |

### Deploy Targets

**Vercel** (Next.js):
```typescript
// next.config.ts
const config: NextConfig = {
  serverExternalPackages: ["@mastra/*"],
};
```

**Mastra Cloud** (Managed):
1. Import project at cloud.mastra.ai
2. Click Deployment -> Enable Deployments
3. Pushes to main auto-deploy

**Other Platforms**: AWS Lambda, Azure, Cloudflare, Netlify, Digital Ocean — each with dedicated guides.

### Studio

```bash
mastra dev       # Dev server + Studio
mastra studio    # Standalone Studio at localhost:3000

# Production Studio
mastra build --studio
MASTRA_STUDIO_PATH=.mastra/output/studio node .mastra/output/index.mjs
```

## Server Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Health check (200 OK) |
| `GET /openapi.json` | OpenAPI spec |
| `GET /swagger-ui` | Interactive API docs |
| `POST /api/agents/:agentId/generate` | Generate text |
| `POST /api/agents/:agentId/stream` | Stream text |
| `POST /api/workflows/:workflowId/start` | Start workflow |
