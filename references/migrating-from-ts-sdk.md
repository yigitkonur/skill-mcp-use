# Migrating from Official MCP TypeScript SDK to mcp-use

## Overview

The official `@modelcontextprotocol/sdk` provides low-level building blocks. mcp-use wraps it and adds production concerns. This guide maps every official SDK pattern to its mcp-use equivalent.

## Package Changes

```
# Official SDK
npm install @modelcontextprotocol/server @modelcontextprotocol/client zod

# mcp-use
npm install mcp-use zod
```

mcp-use re-exports everything needed. No separate server/client packages.

## Import Mapping

| Official SDK | mcp-use |
|---|---|
| `import { McpServer } from '@modelcontextprotocol/server'` | `import { MCPServer } from 'mcp-use/server'` |
| `import { StdioServerTransport } from '@modelcontextprotocol/server'` | Not needed — transport is automatic |
| `import { StreamableHTTPServerTransport } from '@modelcontextprotocol/server'` | Not needed — `server.listen()` handles it |
| `import { ResourceTemplate } from '@modelcontextprotocol/server'` | `server.resourceTemplate()` method instead |
| `import { Client } from '@modelcontextprotocol/client'` | `import { MCPClient } from 'mcp-use'` (Python) |
| `import { StdioClientTransport } from '@modelcontextprotocol/client'` | Configured via `mcpServers` config dict |

## Server Creation

### Official SDK

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';

const server = new McpServer(
    { name: 'my-server', version: '1.0.0' },
    {
        capabilities: {
            tools: {},
            resources: { listChanged: true },
            prompts: { listChanged: true },
            logging: {}
        },
        instructions: 'Server instructions for the client'
    }
);

// Manual transport setup
const transport = new StdioServerTransport();
await server.connect(transport);
```

### mcp-use

```typescript
import { MCPServer } from 'mcp-use/server'

const server = new MCPServer({
    name: 'my-server',
    version: '1.0.0',
    description: 'Server instructions for the client',
    // Production config built-in:
    host: '0.0.0.0',
    allowedOrigins: ['https://myapp.com'],
    sessionIdleTimeoutMs: 86400000,
})

// One-liner start — handles transport, sessions, CORS, /mcp endpoint
await server.listen(3000)
```

### What changes:
1. **Constructor**: Two-arg `(info, options)` → single config object
2. **Capabilities**: Auto-detected from registrations (no manual declaration)
3. **Transport**: Automatic — `listen()` for HTTP, `npx mcp-use run` for stdio
4. **Instructions**: `options.instructions` → `config.description`

## Tool Registration

### Official SDK

```typescript
server.registerTool('search', {
    title: 'Search',
    description: 'Search the database',
    inputSchema: z.object({
        query: z.string().describe('Search query'),
        limit: z.number().default(10).describe('Max results')
    }),
    outputSchema: z.object({
        results: z.array(z.object({
            id: z.string(),
            title: z.string(),
            score: z.number()
        }))
    }),
    annotations: {
        readOnlyHint: true,
        openWorldHint: false
    }
}, async ({ query, limit }, ctx) => {
    const results = await db.search(query, limit);
    await ctx.mcpReq.log('info', `Found ${results.length} results`);
    return {
        content: [{ type: 'text', text: JSON.stringify(results) }],
        structuredContent: { results }
    };
});
```

### mcp-use

```typescript
import { text, object } from 'mcp-use/server'

server.tool({
    name: 'search',
    description: 'Search the database',
    schema: z.object({
        query: z.string().describe('Search query'),
        limit: z.number().default(10).describe('Max results')
    }),
    annotations: { readOnlyHint: true },
}, async ({ query, limit }, ctx) => {
    const results = await db.search(query, limit);
    ctx.log.info(`Found ${results.length} results`);
    return object({ results });
})
```

### What changes:
1. **Method**: `registerTool(name, config, cb)` → `tool(config, cb)` (name moves into config)
2. **Schema key**: `inputSchema` → `schema`
3. **Response**: Manual `{ content: [{ type: 'text', text }] }` → `text()`, `object()`, etc.
4. **Logging**: `ctx.mcpReq.log('info', msg)` → `ctx.log.info(msg)`
5. **Output schema**: Not yet supported in mcp-use tool API (use `object()` for structured data)

## Resource Registration

### Official SDK — Static Resource

```typescript
server.registerResource('readme', 'docs://readme', {
    title: 'README',
    mimeType: 'text/markdown'
}, async (uri) => ({
    contents: [{ uri: uri.href, text: '# My App\nDocumentation here.' }]
}));
```

### mcp-use — Static Resource

```typescript
import { markdown } from 'mcp-use/server'

server.resource({
    uri: 'docs://readme',
    name: 'README',
    title: 'README',
    mimeType: 'text/markdown',
}, async () => markdown('# My App\nDocumentation here.'))
```

### Official SDK — Dynamic Resource (Template)

```typescript
import { ResourceTemplate } from '@modelcontextprotocol/server';

server.registerResource(
    'user-profile',
    new ResourceTemplate('users://{userId}/profile', {
        list: async () => ({
            resources: users.map(u => ({
                uri: `users://${u.id}/profile`,
                name: u.name
            }))
        })
    }),
    { title: 'User Profile', mimeType: 'application/json' },
    async (uri, { userId }) => ({
        contents: [{ uri: uri.href, text: JSON.stringify(getUser(userId)) }]
    })
);
```

### mcp-use — Dynamic Resource (Template)

```typescript
import { object } from 'mcp-use/server'

server.resourceTemplate({
    uriTemplate: 'users://{userId}/profile',
    name: 'User Profile',
    mimeType: 'application/json',
}, async ({ userId }) => object(getUser(userId)))
```

### What changes:
1. **Static**: `registerResource(name, uri, meta, cb)` → `resource(config, cb)`
2. **Template**: `new ResourceTemplate(...)` + `registerResource(...)` → `resourceTemplate(config, cb)`
3. **Response**: Manual `contents: [{ uri, text }]` → response helpers (`text()`, `object()`, etc.)
4. **Listing**: Separate `list` callback in ResourceTemplate → handled automatically or via list callback in config

## Prompt Registration

### Official SDK

```typescript
server.registerPrompt('summarize', {
    title: 'Summarize Text',
    description: 'Create a summary of provided text',
    argsSchema: z.object({
        text: z.string(),
        style: z.enum(['brief', 'detailed']).default('brief')
    })
}, ({ text, style }) => ({
    messages: [{
        role: 'user',
        content: { type: 'text', text: `Summarize (${style}):\n\n${text}` }
    }]
}));
```

### mcp-use

```typescript
import { text } from 'mcp-use/server'

server.prompt({
    name: 'summarize',
    description: 'Create a summary of provided text',
    schema: z.object({
        text: z.string(),
        style: z.enum(['brief', 'detailed']).default('brief')
    }),
}, async ({ text: input, style }) => text(`Summarize (${style}):\n\n${input}`))
```

### What changes:
1. **Method**: `registerPrompt(name, config, cb)` → `prompt(config, cb)`
2. **Schema key**: `argsSchema` → `schema`
3. **Response**: Manual `{ messages: [{ role, content }] }` → response helpers auto-convert

## Transport Layer

### Official SDK — stdio

```typescript
import { StdioServerTransport } from '@modelcontextprotocol/server';
const transport = new StdioServerTransport();
await server.connect(transport);
```

### mcp-use — stdio

```bash
npx mcp-use run server.ts
# Or in development:
npx mcp-use dev server.ts  # With hot reload
```

No code changes needed — the CLI handles stdio transport automatically.

### Official SDK — HTTP (Streamable HTTP)

```typescript
import express from 'express';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/server';
import { randomUUID } from 'crypto';

const app = express();
const transports = new Map<string, StreamableHTTPServerTransport>();

app.post('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    let transport = transports.get(sessionId);
    
    if (!transport) {
        transport = new StreamableHTTPServerTransport({
            sessionIdGenerator: () => randomUUID(),
        });
        transport.onsessioninitialized = (id) => transports.set(id, transport);
        await server.connect(transport);
    }
    
    await transport.handleRequest(req, res, req.body);
});

app.get('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    const transport = transports.get(sessionId);
    if (!transport) { res.status(400).end(); return; }
    await transport.handleRequest(req, res);
});

app.delete('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    const transport = transports.get(sessionId);
    if (transport) { await transport.close(); transports.delete(sessionId); }
    res.status(200).end();
});

app.listen(3000);
```

### mcp-use — HTTP

```typescript
await server.listen(3000)
```

That's it. All of the above (POST/GET/DELETE handlers, session routing, transport management) is handled internally by mcp-use.

## Session Management

### Official SDK

Manual — track in a Map:
```typescript
const sessions = new Map<string, Transport>();
// Route requests by mcp-session-id header
// Clean up on disconnect
// No persistence across restarts
```

### mcp-use

Pluggable storage backends:
```typescript
import { 
    MCPServer,
    InMemorySessionStore,    // Development (default)
    FileSystemSessionStore,  // Single-instance production
    RedisSessionStore,       // Distributed production
    RedisStreamManager,      // Distributed SSE streams
} from 'mcp-use/server'

// Development
const server = new MCPServer({ name: 'dev' })

// Single-instance production
const server = new MCPServer({
    name: 'prod',
    sessionStore: new FileSystemSessionStore({ path: './sessions' }),
})

// Distributed production
const server = new MCPServer({
    name: 'prod',
    sessionStore: new RedisSessionStore({ client: redis }),
    streamManager: new RedisStreamManager({ 
        client: redis, 
        pubSubClient: pubSubRedis 
    }),
})
```

## Authentication

### Official SDK

Manual middleware — implement interfaces yourself:
```typescript
// You must implement:
interface OAuthServerProvider {
    authorize(client, params): Promise<...>;
    challengeForAuthorizationCode(client, authCode): Promise<...>;
    exchangeAuthorizationCode(client, authCode, ...): Promise<...>;
    exchangeRefreshToken(client, refreshToken, ...): Promise<...>;
    verifyAccessToken(token): Promise<...>;
}
// Then wire up /oauth/* endpoints manually
```

### mcp-use

Built-in providers:
```typescript
import { MCPServer, oauthAuth0Provider, oauthSupabaseProvider } from 'mcp-use/server'

const server = new MCPServer({
    name: 'protected-server',
    oauth: oauthAuth0Provider({
        domain: 'your-tenant.auth0.com',
        audience: 'https://your-api.example.com',
    }),
    // OR:
    oauth: oauthSupabaseProvider({
        projectUrl: 'https://xxx.supabase.co',
        serviceRoleKey: 'xxx',
    }),
})

// Access auth in tools:
server.tool({ name: 'profile' }, async (_, ctx) => {
    const auth = getAuth(ctx)
    requireScope(ctx, 'read:profile')
    return text(`Hello, ${auth.user?.name}`)
})
```

Available providers: `oauthAuth0Provider`, `oauthSupabaseProvider`, `oauthWorkOSProvider`, `oauthKeycloakProvider`, `oauthCustomProvider`.

## Advanced Features (mcp-use only)

These features have no equivalent in the official SDK:

| Feature | mcp-use API | Description |
|---|---|---|
| Response helpers | `text()`, `object()`, `markdown()`, `image()`, `audio()`, `binary()`, `file()`, `error()`, `mix()` | Type-safe response builders |
| UI Widgets | `server.widget()`, `server.uiResource()` | Interactive UI in MCP clients |
| Inspector | Built-in at `/inspector` | Debug tools, resources, prompts visually |
| HMR | `npx mcp-use dev` | Hot reload during development |
| Tunneling | `npx @mcp-use/tunnel 3000` | Public URL for local server |
| CLI | `npx mcp-use init/run/dev/inspect` | Full development toolchain |
| Middleware | Hono middleware + Express adapter | `server.use()`, `adaptMiddleware()` |
| Custom HTTP routes | `server.get()`, `server.post()` | REST endpoints alongside MCP |
| Router pattern | Python `MCPRouter` (like FastAPI) | Modular tool organization |
| Elicitation | `ctx.elicit(message, zodSchema)` | Request user input from tools |
| Sampling | `ctx.sample(prompt)` | Request LLM completion from tools |
| Notifications | `sendToolsListChanged()`, `sendNotification()` | Push updates to clients |

## Complete Migration Example

### Before (Official SDK, ~80 lines)

```typescript
import { McpServer, ResourceTemplate } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';
import * as z from 'zod/v4';

const server = new McpServer(
    { name: 'weather-server', version: '1.0.0' },
    { capabilities: { tools: {}, resources: { listChanged: true } } }
);

server.registerTool('get-weather', {
    description: 'Get weather for a city',
    inputSchema: z.object({ city: z.string() })
}, async ({ city }) => ({
    content: [{ type: 'text', text: `Weather in ${city}: Sunny, 72°F` }]
}));

server.registerResource('cities', 'weather://cities', {
    title: 'Available Cities'
}, async (uri) => ({
    contents: [{ uri: uri.href, text: JSON.stringify(['NYC', 'LA', 'London']) }]
}));

server.registerPrompt('forecast', {
    argsSchema: z.object({ city: z.string(), days: z.number() })
}, ({ city, days }) => ({
    messages: [{
        role: 'user',
        content: { type: 'text', text: `Give me a ${days}-day forecast for ${city}` }
    }]
}));

const transport = new StdioServerTransport();
await server.connect(transport);
```

### After (mcp-use, ~40 lines)

```typescript
import { MCPServer, text, object } from 'mcp-use/server'
import { z } from 'zod'

const server = new MCPServer({
    name: 'weather-server',
    version: '1.0.0',
})

server.tool({
    name: 'get-weather',
    description: 'Get weather for a city',
    schema: z.object({ city: z.string() }),
}, async ({ city }) => text(`Weather in ${city}: Sunny, 72°F`))

server.resource({
    uri: 'weather://cities',
    name: 'Available Cities',
}, async () => object(['NYC', 'LA', 'London']))

server.prompt({
    name: 'forecast',
    schema: z.object({ city: z.string(), days: z.number() }),
}, async ({ city, days }) => text(`Give me a ${days}-day forecast for ${city}`))

await server.listen(3000)
```
