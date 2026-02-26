# Workflow: Deploy Project

Guide the user through building and deploying their Mastra app to production.

## Steps

### 1. Build for Production

```bash
mastra build
```

Output:
```
.mastra/output/
├── index.mjs         # Server entry point
├── mastra.mjs        # Bundled config
├── tools.mjs         # Tools
├── package.json      # Production deps
├── node_modules/     # Installed deps
└── playground/       # Studio UI (if --studio)
```

### 2. Test Locally

```bash
mastra start
# or
node .mastra/output/index.mjs
```

Server starts at `http://localhost:4111` by default.

### 3. Choose Deployment Target

## Mastra Cloud (Managed)

1. Go to [cloud.mastra.ai](https://cloud.mastra.ai)
2. Import your GitHub repository
3. Click **Deployment** -> **Enable Deployments**
4. Configure environment variables
5. Push to main — auto-deploys

```typescript
// Use LibSQLStore for Mastra Cloud storage
import { LibSQLStore } from "@mastra/libsql";

export const mastra = new Mastra({
  storage: new LibSQLStore({ id: "store", url: "file:./mastra.db" }),
  agents: { myAgent },
});
```

## Vercel

### With Next.js
```typescript
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  serverExternalPackages: ["@mastra/*"],
};

export default nextConfig;
```

Deploy:
```bash
vercel --prod
```

### Standalone
```bash
mastra build
# Deploy .mastra/output/ to Vercel
```

## Netlify

```bash
mastra build
# Deploy .mastra/output/ as Netlify Function
```

## Cloudflare Workers

```bash
mastra build
# Deploy with wrangler
```

## AWS Lambda

```bash
mastra build
# Package .mastra/output/ for Lambda deployment
```

## Docker / Self-Hosted

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY .mastra/output/ .
EXPOSE 4111
CMD ["node", "index.mjs"]
```

```bash
docker build -t my-mastra-app .
docker run -p 4111:4111 --env-file .env my-mastra-app
```

## Railway / Digital Ocean / Azure

```bash
mastra build
# Push .mastra/output/ to your platform
```

### 4. Configure Environment Variables

Set these in your deployment platform:

```bash
# Required: At least one model provider key
OPENAI_API_KEY=sk-...
# ANTHROPIC_API_KEY=...
# GOOGLE_GENERATIVE_AI_API_KEY=...

# Optional: Server config
PORT=4111

# Optional: Database
DATABASE_URL=postgres://...

# Optional: Auth
MASTRA_JWT_SECRET=your-secret

# Optional: Observability
LANGFUSE_PUBLIC_KEY=...
LANGFUSE_SECRET_KEY=...
```

### 5. Verify Deployment

```bash
# Health check
curl https://your-app.com/health

# Test agent
curl -X POST https://your-app.com/api/agents/myAgent/generate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-token" \
  -d '{"messages": "Hello!"}'
```

### 6. Deploy Studio (Optional)

```bash
# Build with Studio
mastra build --studio

# Set studio path
MASTRA_STUDIO_PATH=.mastra/output/studio node .mastra/output/index.mjs
```

Or deploy as standalone SPA to any CDN.

## Production Checklist

- [ ] Environment variables configured
- [ ] Authentication enabled (JWT, Supabase, etc.)
- [ ] Storage configured (not in-memory)
- [ ] Observability/tracing enabled
- [ ] Health check endpoint responding
- [ ] CORS configured for frontend origins
- [ ] Rate limiting in place
- [ ] Error monitoring (Sentry, etc.)
