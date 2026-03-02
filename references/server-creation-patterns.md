# Server Creation Patterns

## Overview

This reference covers how MCP servers are created across three approaches: the official TypeScript SDK, mcp-use TypeScript, and mcp-use Python.

## Official TypeScript SDK (`@modelcontextprotocol/sdk`)

### Two-Level Architecture

The official SDK has two server classes:

1. **`McpServer`** (high-level) — ergonomic API with `registerTool()`, `registerResource()`, `registerPrompt()`
2. **`Server`** (low-level) — raw JSON-RPC protocol handler with `setRequestHandler()`

```typescript
import { McpServer } from '@modelcontextprotocol/server';

// High-level
const server = new McpServer(
    { name: 'my-server', version: '1.0.0' },    // Implementation info
    {                                              // ServerOptions
        capabilities: {
            tools: {},
            resources: { listChanged: true },
            prompts: { listChanged: true },
            logging: {}
        },
        instructions: 'Instructions for the LLM client'
    }
);
```

```typescript
import { Server } from '@modelcontextprotocol/server';

// Low-level
const server = new Server(
    { name: 'my-server', version: '1.0.0' },
    { capabilities: { tools: {} } }
);
server.setRequestHandler('tools/list', async () => ({
    tools: [{ name: 'add', inputSchema: { type: 'object', properties: { a: { type: 'number' } } } }]
}));
server.setRequestHandler('tools/call', async (request) => {
    // Handle tool calls manually
});
```

### Transport Setup (Manual)

```typescript
import { StdioServerTransport } from '@modelcontextprotocol/server';

const transport = new StdioServerTransport();
await server.connect(transport);  // Blocks until stdin closes
```

For HTTP, you must wire up Express/Hono + StreamableHTTPServerTransport manually (see migrating-from-ts-sdk.md).

## mcp-use TypeScript (`mcp-use/server`)

### Single MCPServer Class

mcp-use merges the official McpServer + Hono HTTP framework into one class:

```typescript
import { MCPServer } from 'mcp-use/server'

const server = new MCPServer({
    name: 'my-server',
    version: '1.0.0',
    description: 'What this server does',
    
    // HTTP config (built-in)
    host: '0.0.0.0',              // default: 'localhost'
    baseUrl: 'https://myserver.com',
    allowedOrigins: ['*'],         // CORS
    
    // Session config
    sessionIdleTimeoutMs: 86400000,  // 1 day default
    autoCreateSessionOnInvalidId: false,
    
    // Storage backends
    sessionStore: new RedisSessionStore({ client: redis }),
    streamManager: new RedisStreamManager({ client: redis, pubSubClient: pubSub }),
    
    // Authentication
    oauth: oauthAuth0Provider({ domain: '...', audience: '...' }),
    
    // Metadata
    favicon: '/favicon.ico',
    websiteUrl: 'https://mysite.com',
})
```

### Starting the Server

```typescript
// HTTP mode (production)
await server.listen(3000)
// → Serves /mcp (Streamable HTTP), /inspector (debug UI), /openmcp.json (manifest)

// Stdio mode (CLI)
// $ npx mcp-use run server.ts

// Development mode (HMR)
// $ npx mcp-use dev server.ts
```

### ServerConfig Interface

```typescript
interface ServerConfig {
    name: string;                              // Required: server name
    version?: string;                          // Default: '1.0.0'
    title?: string;                            // Display name
    description?: string;                      // Server instructions
    host?: string;                             // Default: 'localhost'
    baseUrl?: string;                          // Override host:port
    allowedOrigins?: string[];                 // CORS origins
    sessionIdleTimeoutMs?: number;             // Default: 86400000 (1 day)
    autoCreateSessionOnInvalidId?: boolean;    // Auto-create on invalid session
    oauth?: OAuthProvider;                     // OAuth config
    favicon?: string;                          // Server favicon
    websiteUrl?: string;                       // Server website
    icons?: Array<{ src: string; mimeType?: string; sizes?: string[] }>;
    sessionStore?: SessionStore;               // Pluggable session persistence
    streamManager?: StreamManager;             // Pluggable SSE stream management
}
```

### How It Works Internally

1. Creates an official `McpServer` from `@modelcontextprotocol/sdk` internally
2. Creates a Hono HTTP application
3. Uses JavaScript `Proxy` to merge both APIs (`.tool()`, `.resource()` + `.get()`, `.post()`)
4. Mounts `/mcp` endpoint with Streamable HTTP transport
5. Sets up session management, OAuth, inspector
6. `listen(port)` starts the Hono server

## mcp-use Python (`mcp_use.server`)

### MCPServer Class (extends FastMCP)

```python
from mcp_use import MCPServer

server = MCPServer(
    name="my-server",
    # Extends FastMCP from official Python SDK
)

@server.tool(name="greet", description="Greet someone")
async def greet(name: str) -> str:
    return f"Hello, {name}!"

@server.resource(uri="info://version", name="version")
async def version() -> str:
    return "1.0.0"

@server.prompt(name="review", title="Code Review")
def review_prompt(code: str) -> str:
    return f"Please review:\n{code}"

# Run the server
server.run()  # or use with uvicorn for HTTP
```

### Python MCPRouter (Modular Organization)

```python
from mcp_use.server import MCPRouter

# math_routes.py
router = MCPRouter()

@router.tool()
def add(a: int, b: int) -> int:
    return a + b

@router.tool()
def multiply(a: int, b: int) -> int:
    return a * b

# main.py
from mcp_use.server import MCPServer
from math_routes import router as math_router

server = MCPServer(name="calculator")
server.include_router(math_router, prefix="math")
# Tools become: math_add, math_multiply
```

### Python Authentication

```python
from mcp_use.server.auth import BearerAuthProvider

server = MCPServer(
    name="protected-server",
    auth=BearerAuthProvider(token="secret-token"),
)
```

## Comparison Table

| Aspect | Official TS SDK | mcp-use TS | mcp-use Python |
|---|---|---|---|
| Class | `McpServer` + `Server` | `MCPServer` (single) | `MCPServer` (extends FastMCP) |
| Transport | Manual setup | Built-in via `listen()` | Built-in via `run()` |
| HTTP Framework | BYO (Express/Hono) | Built-in Hono | Built-in (Starlette) |
| Session Storage | Manual Map tracking | Pluggable: Memory/FS/Redis | Via FastMCP |
| OAuth | Manual implementation | 5 built-in providers | BearerAuthProvider |
| Schema | Zod v4 | Zod | Pydantic + Field |
| CLI | None | `mcp-use run/dev/inspect/init` | None |
| HMR | None | `mcp-use dev` | None |
| Inspector | Separate package | Built-in at `/inspector` | Built-in debug mode |
| Response helpers | None | `text()`, `object()`, etc. | Return strings/dicts |
| Routing | None | Hono routes | MCPRouter |
| Middleware | None | Hono + Express adapter | Custom Middleware class |
