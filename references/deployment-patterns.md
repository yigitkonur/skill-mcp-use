# Deployment Patterns

## Overview

mcp-use servers can be deployed to various platforms. This reference covers deployment configurations for each supported target.

## Local Development

### CLI Commands

```bash
# Initialize a new project
npx mcp-use init

# Run in stdio mode (for MCP client integration)
npx mcp-use run server.ts

# Development mode with hot reload
npx mcp-use dev server.ts

# Inspect server (opens debug UI)
npx mcp-use inspect server.ts
```

### Development Server (HTTP)

```typescript
import { MCPServer } from 'mcp-use/server'

const server = new MCPServer({
    name: 'dev-server',
    version: '1.0.0',
})

// Register tools, resources, prompts...

await server.listen(3000)
// → http://localhost:3000/mcp      (Streamable HTTP endpoint)
// → http://localhost:3000/inspector (Debug UI)
// → http://localhost:3000/openmcp.json (Manifest)
```

### Tunneling (Public URL)

```bash
npx @mcp-use/tunnel 3000
# → https://your-server.local.mcp-use.run
```

## Manufact Cloud (mcp-use Cloud)

### Deploy via CLI

```bash
# Login
npx mcp-use login

# Deploy
npx mcp-use deploy

# With environment variables
npx mcp-use deploy --env API_KEY=sk-xxx --env DB_URL=postgres://...

# Specify region
npx mcp-use deploy --region us-east-1
```

### Environment Variables

```bash
# Set env vars
npx mcp-use env set API_KEY sk-xxx
npx mcp-use env set DATABASE_URL postgres://...

# List env vars
npx mcp-use env list

# Remove env var
npx mcp-use env unset API_KEY
```

### Configuration (mcp-use.json)

```json
{
    "name": "my-server",
    "version": "1.0.0",
    "entry": "server.ts",
    "region": "us-east-1",
    "env": {
        "NODE_ENV": "production"
    }
}
```

## Supabase Edge Functions

### Setup

```bash
# Initialize Supabase project
supabase init

# Create the edge function
supabase functions new mcp-server
```

### Edge Function Code

```typescript
// supabase/functions/mcp-server/index.ts
import { MCPServer, text, oauthSupabaseProvider } from 'mcp-use/server'
import { z } from 'zod'

const server = new MCPServer({
    name: 'supabase-mcp-server',
    version: '1.0.0',
    oauth: oauthSupabaseProvider({
        projectUrl: Deno.env.get('SUPABASE_URL')!,
        serviceRoleKey: Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!,
    }),
})

server.tool({
    name: 'query-database',
    description: 'Query the Supabase database',
    schema: z.object({ table: z.string(), limit: z.number().default(10) }),
}, async ({ table, limit }, ctx) => {
    const { data } = await supabase.from(table).select('*').limit(limit)
    return text(JSON.stringify(data))
})

export default server.fetch  // Supabase Edge Functions use fetch handler
```

### Deploy

```bash
# Set secrets
supabase secrets set API_KEY=sk-xxx

# Deploy
supabase functions deploy mcp-server

# Test
curl -X POST https://xxx.supabase.co/functions/v1/mcp-server/mcp \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"initialize","params":{...}}'
```

### Limitations

- 150-second execution limit per request
- Deno runtime (not Node.js)
- Cold starts on first invocation

## Google Cloud Run

### Dockerfile

```dockerfile
FROM node:20-slim

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

EXPOSE 3000
CMD ["npx", "mcp-use", "run", "server.ts"]
```

### Server Code

```typescript
import { MCPServer, text } from 'mcp-use/server'
import { z } from 'zod'

const server = new MCPServer({
    name: 'cloud-run-mcp',
    version: '1.0.0',
    host: '0.0.0.0',  // Required for Cloud Run
})

server.tool({
    name: 'hello',
    schema: z.object({ name: z.string() }),
}, async ({ name }) => text(`Hello, ${name}!`))

await server.listen(parseInt(process.env.PORT || '3000'))
```

### Deploy

```bash
# Build and push
gcloud builds submit --tag gcr.io/PROJECT_ID/mcp-server

# Deploy
gcloud run deploy mcp-server \
    --image gcr.io/PROJECT_ID/mcp-server \
    --platform managed \
    --region us-central1 \
    --allow-unauthenticated \
    --set-env-vars "API_KEY=sk-xxx"

# With IAM authentication
gcloud run deploy mcp-server \
    --image gcr.io/PROJECT_ID/mcp-server \
    --platform managed \
    --no-allow-unauthenticated
```

### Gemini CLI Integration

```bash
# Connect Gemini CLI to your Cloud Run MCP server
gemini configure mcp add my-server \
    --transport sse \
    --url https://mcp-server-xxx-uc.a.run.app/mcp
```

## Docker (Self-Hosted)

### Dockerfile

```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Docker Compose with Redis

```yaml
version: '3.8'
services:
    mcp-server:
        build: .
        ports:
            - "3000:3000"
        environment:
            - REDIS_URL=redis://redis:6379
            - NODE_ENV=production
        depends_on:
            - redis

    redis:
        image: redis:7-alpine
        ports:
            - "6379:6379"
        volumes:
            - redis-data:/data

volumes:
    redis-data:
```

## Environment Variables

mcp-use supports these environment variables:

| Variable | Purpose | Default |
|---|---|---|
| `MCP_TRANSPORT` | Override transport type | Auto-detected |
| `MCP_PORT` | Override listen port | 3000 |
| `MCP_HOST` | Override host binding | localhost |
| `MCP_BASE_URL` | Override base URL | `http://{host}:{port}` |
| `DEBUG` | Enable debug logging (0/1/2) | 0 |
| `NODE_ENV` | Environment mode | development |

## Connecting Clients to Deployed Servers

### mcp-use Python Client

```python
config = {
    "mcpServers": {
        "remote": {
            "url": "https://your-server.example.com/mcp",
            "headers": {"Authorization": "Bearer your-token"}
        }
    }
}

client = MCPClient(config=config)
agent = MCPAgent(llm=llm, client=client)
result = await agent.run("Query the remote server")
```

### Claude Desktop / Cursor

```json
{
    "mcpServers": {
        "remote-server": {
            "url": "https://your-server.example.com/mcp",
            "headers": {
                "Authorization": "Bearer your-token"
            }
        }
    }
}
```

### Stdio Connection (Local Servers)

```json
{
    "mcpServers": {
        "local-server": {
            "command": "npx",
            "args": ["mcp-use", "run", "server.ts"]
        }
    }
}
```
