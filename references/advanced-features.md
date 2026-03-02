# Advanced Features

## Elicitation

Elicitation allows tools to request additional input from the user during execution.

### Two Modes

| Mode | Use Case | How |
|---|---|---|
| **Form** | Collect structured data (non-sensitive) | `ctx.elicit(message, zodSchema)` |
| **URL** | Sensitive data (credentials, OAuth) | `ctx.elicit(message, { url: '...' })` |

### Form Mode

```typescript
server.tool({
    name: 'create-account',
    description: 'Create a new user account',
}, async (_, ctx) => {
    const result = await ctx.elicit(
        'Please provide your account details',
        z.object({
            name: z.string().describe('Full name'),
            email: z.string().email().describe('Email address'),
            role: z.enum(['admin', 'user', 'viewer']).default('user').describe('Account role'),
            notifications: z.boolean().default(true).describe('Email notifications'),
        })
    )

    if (result.action === 'accept') {
        // result.data is typed: { name: string, email: string, role: 'admin'|'user'|'viewer', ... }
        await createUser(result.data)
        return text(`Account created for ${result.data.name}`)
    } else if (result.action === 'decline') {
        return text('Account creation cancelled by user')
    } else {
        return text('Operation cancelled')
    }
})
```

### URL Mode

```typescript
server.tool({ name: 'connect-github' }, async (_, ctx) => {
    const result = await ctx.elicit(
        'Please authorize with GitHub',
        { url: 'https://github.com/login/oauth/authorize?client_id=xxx&scope=repo' }
    )
    // User is redirected to URL, then back
})
```

### ElicitResult Interface

```typescript
interface ElicitResult<T> {
    action: 'accept' | 'decline' | 'cancel';
    data: T;           // Typed from Zod schema (only when action === 'accept')
    validationError?: string;  // Server-side validation error
}
```

### Server-Side Validation Flow

1. Zod schema → JSON Schema (sent to client)
2. Client collects user input
3. Client validates (optional)
4. Data sent back to server
5. **Server validates against original Zod schema** (mcp-use feature)
6. Typed data returned to tool handler

### Capability Check

```typescript
server.tool({ name: 'interactive-tool' }, async (_, ctx) => {
    if (!ctx.client?.can('elicitation')) {
        return error('This tool requires a client that supports elicitation')
    }
    const result = await ctx.elicit('...', z.object({...}))
})
```

---

## Sampling

Sampling allows tools to request LLM completions from the client.

### Basic Usage

```typescript
server.tool({
    name: 'smart-search',
    description: 'Search and summarize results',
    schema: z.object({ query: z.string() }),
}, async ({ query }, ctx) => {
    // Search for data
    const results = await searchDatabase(query)
    
    // Ask the client's LLM to summarize
    const summary = await ctx.sample(`Summarize these search results:\n${JSON.stringify(results)}`)
    
    return text(summary)
})
```

### Extended Usage

```typescript
const response = await ctx.sample({
    messages: [
        { role: 'user', content: { type: 'text', text: 'Analyze this data...' } },
    ],
    modelPreferences: {
        hints: [{ name: 'claude-sonnet-4-5' }],
        intelligencePriority: 0.8,
        speedPriority: 0.2,
    },
    systemPrompt: 'You are a data analyst...',
    maxTokens: 1000,
    temperature: 0.5,
    includeContext: 'thisServer',  // Include current server context
})

// response is CreateMessageResult
console.log(response.content.text)
console.log(response.model)       // Which model was used
console.log(response.stopReason)   // 'endTurn' | 'stopSequence' | 'maxTokens'
```

### Progress Reporting During Sampling

```typescript
const response = await ctx.sample({
    messages: [{ role: 'user', content: { type: 'text', text: '...' } }],
    onProgress: (progress) => {
        ctx.reportProgress(progress.current, progress.total)
    },
})
```

### Capability Check

```typescript
if (ctx.client?.can('sampling')) {
    const result = await ctx.sample('...')
} else {
    // Fallback: use a direct API call
    const result = await callOpenAI('...')
}
```

---

## Subscriptions

Resource subscriptions allow clients to receive notifications when resource content changes.

### Server-Side

```typescript
import { sendResourcesListChanged } from 'mcp-use/server'

// Resources automatically support subscriptions when the server
// has resources capability with listChanged: true

// Notify specific resource updated
await server.notifyResourceUpdated('users://123/profile')

// Notify all clients that the resource list changed
await sendResourcesListChanged(sessions)
```

### Subscription Lifecycle

```
Client                                 Server
──────                                 ──────
resources/subscribe(uri)         →     Track subscription for session
                                       (auto-cleanup on disconnect)

[resource changes]               ←     notifications/resources/updated
                                       { uri: 'users://123/profile' }

resources/read(uri)              →     Return fresh content

resources/unsubscribe(uri)       →     Remove subscription tracking
```

### Per-Session Tracking

Subscriptions are tracked per session and automatically cleaned up when sessions disconnect:

```typescript
// Internal tracking (handled by mcp-use)
// Map<sessionId, Set<resourceUri>>
```

---

## Notifications

MCP supports bidirectional notifications (fire-and-forget, no response expected).

### Server → Client Notifications

```typescript
import { 
    sendNotification, 
    sendNotificationToSession,
    sendToolsListChanged,
    sendResourcesListChanged,
    sendPromptsListChanged 
} from 'mcp-use/server'

// Built-in list-change notifications
await sendToolsListChanged(sessions)       // After adding/removing tools
await sendResourcesListChanged(sessions)   // After adding/removing resources
await sendPromptsListChanged(sessions)     // After adding/removing prompts

// Custom notification to all sessions
await sendNotification(sessions, 'custom/event', { key: 'value' })

// Custom notification to specific session
await sendNotificationToSession(session, 'custom/event', { key: 'value' })

// From tool context
server.tool({ name: 'my-tool' }, async (_, ctx) => {
    await ctx.sendNotification('processing/started', {})
    // ... do work ...
    await ctx.sendNotification('processing/complete', { items: 42 })
    return text('Done')
})
```

### Progress Reporting

```typescript
server.tool({ name: 'long-task' }, async (_, ctx) => {
    for (let i = 0; i < 100; i++) {
        await doWork(i)
        ctx.reportProgress(i + 1, 100)  // 1/100, 2/100, ... 100/100
    }
    return text('Complete')
})
```

### Client → Server Notifications

```typescript
// Roots changed notification
server.onRootsChanged(async (roots) => {
    console.log('Client roots changed:', roots)
    // Re-index or update server state based on new roots
})
```

---

## Logging

### Server-Side Logging (mcp-use Logger)

```typescript
import { Logger } from 'mcp-use/server'

// Configure globally
Logger.configure({
    level: 'info',                          // 'error' | 'warn' | 'info' | 'debug' | 'trace'
    format: 'pretty',                       // 'pretty' | 'json' | 'minimal'
    timestamps: true,
})

// Or via environment variable
// DEBUG=1  → info level
// DEBUG=2  → debug level
```

### Tool-to-Client Logging

Tools can send log messages to the connected client:

```typescript
server.tool({ name: 'my-tool' }, async (_, ctx) => {
    ctx.log.info('Starting processing')
    ctx.log.debug('Detailed debug info', { key: 'value' })
    ctx.log.warn('Something unexpected')
    ctx.log.error('Something failed')
    
    // MCP protocol log levels:
    // 'emergency' | 'alert' | 'critical' | 'error' | 'warning' | 'notice' | 'info' | 'debug'
    
    return text('Done')
})
```

### Log Levels

| Level | Numeric | Use Case |
|---|---|---|
| emergency | 0 | System unusable |
| alert | 1 | Immediate action needed |
| critical | 2 | Critical conditions |
| error | 3 | Error conditions |
| warning | 4 | Warning conditions |
| notice | 5 | Normal but significant |
| info | 6 | Informational |
| debug | 7 | Debug-level |

---

## Middleware

### Hono Middleware (mcp-use TS)

```typescript
import { MCPServer } from 'mcp-use/server'
import { cors } from 'hono/cors'
import { logger } from 'hono/logger'

const server = new MCPServer({ name: 'my-server' })

// Standard Hono middleware
server.use('*', cors())
server.use('*', logger())

// Custom middleware
server.use('*', async (c, next) => {
    const start = Date.now()
    await next()
    console.log(`${c.req.method} ${c.req.url} - ${Date.now() - start}ms`)
})
```

### Express Middleware Adapter

```typescript
import { adaptMiddleware, adaptConnectMiddleware } from 'mcp-use/server'
import helmet from 'helmet'
import morgan from 'morgan'

server.use(adaptMiddleware(helmet()))
server.use(adaptMiddleware(morgan('combined')))
```

### Custom HTTP Routes

```typescript
// Health check
server.get('/health', (c) => c.json({ status: 'ok' }))

// Custom API endpoint
server.post('/api/webhook', async (c) => {
    const body = await c.req.json()
    // Process webhook...
    return c.json({ received: true })
})
```

### Python Middleware

```python
from mcp_use.server.middleware import Middleware

class LoggingMiddleware(Middleware):
    async def process_request(self, request, context):
        print(f"Request: {request.method}")
        return request
    
    async def process_response(self, response, context):
        return response

server = MCPServer(
    name="my-server",
    middleware=[LoggingMiddleware()],
)
```

---

## UI Widgets (mcp-use Exclusive)

Widgets enable rich interactive UI in supported MCP clients.

### Widget Types

| Type | Description |
|---|---|
| `mcp-apps` | MCP-Apps protocol components |
| `external-url` | Embedded external URL |
| `raw-html` | Raw HTML content |
| `remote-dom` | Remote DOM rendering |

### Basic Widget

```typescript
server.widget({
    name: 'dashboard',
    title: 'Dashboard Widget',
    type: 'mcp-apps',
    component: DashboardComponent,
})

// Or as a UI resource
server.uiResource({
    name: 'chart',
    title: 'Data Chart',
    type: 'external-url',
    url: 'https://charts.example.com/embed/123',
})
```

### Widget in Tool Response

```typescript
import { widget } from 'mcp-use/server'

server.tool({ name: 'show-chart' }, async (_, ctx) => {
    return widget({
        props: { data: chartData },
        output: { summary: 'Chart showing...' },    // For LLM context
        metadata: { title: 'Sales Chart' },
    })
})
```

---

## Code Execution Mode (Python)

```python
from mcp_use import MCPClient

client = MCPClient(config=config, code_mode=True)
await client.create_all_sessions()

result = await client.execute_code('''
    tools = await search_tools("github")
    pr = await github.get_pull_request(owner="facebook", repo="react", number=12345)
    return {"title": pr["title"]}
''')
```

- Requires `code_mode=True` in MCPClient constructor
- Code has access to all discovered MCP tools as async functions
- Results returned as Python objects
- Raises `RuntimeError` if `code_mode=False`
