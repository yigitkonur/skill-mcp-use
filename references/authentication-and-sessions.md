# Authentication and Session Management

## Authentication

### Overview

MCP supports OAuth 2.0/2.1 for server authentication. The approach differs significantly between the official SDK (manual) and mcp-use (built-in providers).

### Official TS SDK — Manual OAuth

The official SDK provides interfaces but requires manual implementation:

```typescript
// You must implement this interface
interface OAuthServerProvider {
    clientsStore: OAuthRegisteredClientsStore;
    authorize(client: OAuthClientInformationFull, params: AuthorizationParams): Promise<string>;
    challengeForAuthorizationCode(client: OAuthClientInformationFull, authorizationCode: string): Promise<string>;
    exchangeAuthorizationCode(client: OAuthClientInformationFull, authorizationCode: string, codeVerifier?: string): Promise<OAuthTokens>;
    exchangeRefreshToken(client: OAuthClientInformationFull, refreshToken: string, scopes?: string[]): Promise<OAuthTokens>;
    verifyAccessToken(token: string): Promise<AuthInfo>;
}

// Then wire up auth middleware manually
function requireBearerAuth(options) {
    return async (req, res, next) => {
        const token = req.headers.authorization?.slice('Bearer '.length);
        if (!token) { res.status(401).end(); return; }
        const authInfo = await provider.verifyAccessToken(token);
        next();
    };
}
```

### mcp-use TS — Built-in OAuth Providers

```typescript
import { MCPServer, oauthAuth0Provider } from 'mcp-use/server'

const server = new MCPServer({
    name: 'protected-server',
    oauth: oauthAuth0Provider({
        domain: 'your-tenant.auth0.com',
        audience: 'https://your-api.example.com',
    }),
})
```

### Available Providers

| Provider | Import | Key Config |
|---|---|---|
| Auth0 | `oauthAuth0Provider` | `{ domain, audience }` |
| Supabase | `oauthSupabaseProvider` | `{ projectUrl, serviceRoleKey }` |
| WorkOS | `oauthWorkOSProvider` | `{ clientId, apiKey }` |
| Keycloak | `oauthKeycloakProvider` | `{ realm, serverUrl, clientId }` |
| Custom | `oauthCustomProvider` | Full OAuth endpoint config |

### Auth0 Configuration

```typescript
import { oauthAuth0Provider } from 'mcp-use/server'

const oauth = oauthAuth0Provider({
    domain: 'your-tenant.auth0.com',
    audience: 'https://your-api.example.com',
    // Optional:
    scopes: ['read:data', 'write:data'],
    clientId: 'override-client-id',
})
```

### Supabase Configuration

```typescript
import { oauthSupabaseProvider } from 'mcp-use/server'

// With JWKS (ES256) — recommended
const oauth = oauthSupabaseProvider({
    projectUrl: 'https://xxxx.supabase.co',
    serviceRoleKey: process.env.SUPABASE_SERVICE_ROLE_KEY,
})

// With JWT Secret (HS256) — simpler
const oauth = oauthSupabaseProvider({
    projectUrl: 'https://xxxx.supabase.co',
    jwtSecret: process.env.SUPABASE_JWT_SECRET,
})
```

### WorkOS Configuration

```typescript
import { oauthWorkOSProvider } from 'mcp-use/server'

const oauth = oauthWorkOSProvider({
    clientId: process.env.WORKOS_CLIENT_ID,
    apiKey: process.env.WORKOS_API_KEY,
    // Supports: SSO (SAML/OIDC), Directory Sync, Organizations
})
```

### Accessing Auth in Tools

```typescript
import { getAuth, requireScope } from 'mcp-use/server'

server.tool({ name: 'protected-action' }, async (_, ctx) => {
    // Get auth info
    const auth = getAuth(ctx)
    console.log(auth.user?.name, auth.user?.email)
    console.log(auth.scopes)  // ['read:data', 'write:data']
    
    // Enforce scopes
    requireScope(ctx, 'write:data')  // Throws if scope missing
    
    // Conditional tool visibility
    if (!auth.scopes.includes('admin')) {
        return error('Admin access required')
    }
    
    return text('Action completed')
})
```

### User Context

```typescript
// UserInfo interface
interface UserInfo {
    sub: string;          // Unique user identifier
    name?: string;
    email?: string;
    picture?: string;
    [key: string]: any;   // Provider-specific fields
}

// Custom user info extraction
const server = new MCPServer({
    oauth: oauthAuth0Provider({...}),
    getUserInfo: async (token) => {
        const decoded = await verifyToken(token);
        return {
            sub: decoded.sub,
            name: decoded.name,
            email: decoded.email,
            roles: decoded['https://myapp.com/roles'],
        };
    },
})
```

### Python Authentication

```python
from mcp_use.server.auth import BearerAuthProvider

server = MCPServer(
    name="protected-server",
    auth=BearerAuthProvider(token="your-secret-token"),
)
```

### OAuth Endpoints (Auto-configured by mcp-use)

| Endpoint | Purpose |
|---|---|
| `GET /oauth/authorize` | Start OAuth flow |
| `POST /oauth/token` | Exchange code for tokens |
| `GET /oauth/callback` | OAuth callback handler |
| `GET /.well-known/oauth-authorization-server` | Server metadata |
| `GET /.well-known/oauth-protected-resource` | Protected resource metadata |

---

## Session Management

### Overview

MCP uses sessions to track client connections. The official SDK requires manual session management; mcp-use provides pluggable storage backends.

### Session Lifecycle

```
Client                          Server
──────                          ──────
POST /mcp (initialize)    →    Create session + sessionId
                           ←    Mcp-Session-Id: abc123
POST /mcp (tools/list)    →    Route by Mcp-Session-Id header
  Mcp-Session-Id: abc123
GET /mcp (SSE stream)     →    Open SSE connection for session
  Mcp-Session-Id: abc123
...requests...
DELETE /mcp               →    Terminate session
  Mcp-Session-Id: abc123
```

### Official SDK — Manual Session Management

```typescript
const transports = new Map<string, StreamableHTTPServerTransport>();

// Create transport per session
app.post('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    let transport = transports.get(sessionId);
    
    if (!transport) {
        transport = new StreamableHTTPServerTransport({
            sessionIdGenerator: () => randomUUID(),
        });
        transport.onsessioninitialized = (id) => {
            transports.set(id, transport!);
        };
        const serverInstance = new McpServer({...});
        await serverInstance.connect(transport);
    }
    
    await transport.handleRequest(req, res, req.body);
});

// Handle SSE
app.get('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    const transport = transports.get(sessionId);
    if (!transport) { res.status(400).end(); return; }
    await transport.handleRequest(req, res);
});

// Handle termination
app.delete('/mcp', async (req, res) => {
    const sessionId = req.headers['mcp-session-id'] as string;
    const transport = transports.get(sessionId);
    if (transport) {
        await transport.close();
        transports.delete(sessionId);
    }
    res.status(200).end();
});
```

### mcp-use — Pluggable Session Storage

All of the above is handled automatically. You just choose a storage backend:

#### InMemorySessionStore (Development Default)

```typescript
const server = new MCPServer({
    name: 'dev-server',
    // InMemorySessionStore is used by default
})
```

- Fast, zero dependencies
- Lost on server restart
- Not suitable for distributed deployments

#### FileSystemSessionStore (Single-Instance Production)

```typescript
import { FileSystemSessionStore } from 'mcp-use/server'

const server = new MCPServer({
    name: 'prod-server',
    sessionStore: new FileSystemSessionStore({
        path: './sessions',        // Directory for session files
        debounceMs: 1000,          // Write debounce (default: 1000ms)
        maxAgeMs: 86400000,        // Session TTL (default: 1 day)
    }),
})
```

- Persists across restarts
- JSON files, one per session
- Auto-selected in dev mode when filesystem is available
- Atomic writes to prevent corruption

#### RedisSessionStore (Distributed Production)

```typescript
import { RedisSessionStore, RedisStreamManager } from 'mcp-use/server'
import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)
const pubSubRedis = new Redis(process.env.REDIS_URL)  // Separate for Pub/Sub

const server = new MCPServer({
    name: 'prod-server',
    sessionStore: new RedisSessionStore({ client: redis }),
    streamManager: new RedisStreamManager({
        client: redis,
        pubSubClient: pubSubRedis,
    }),
})
```

- Distributed session storage
- Redis Pub/Sub for cross-server SSE notifications
- Two components: SessionStore (metadata) + StreamManager (SSE connections)
- Supports TTL-based session expiry

### Split Architecture: SessionStore vs StreamManager

```
SessionStore                          StreamManager
────────────                          ─────────────
Stores: session metadata              Manages: active SSE connections
(serializable data)                   (in-process connections)

InMemorySessionStore ←→ InMemoryStreamManager     (single server)
RedisSessionStore    ←→ RedisStreamManager         (distributed)
FileSystemSessionStore ←→ InMemoryStreamManager    (single, persistent)
```

**Why split?** In distributed deployments, the server handling an HTTP request may not be the server holding the client's SSE connection. The StreamManager (via Redis Pub/Sub) routes notifications to the correct server.

### Session Configuration

```typescript
const server = new MCPServer({
    sessionIdleTimeoutMs: 86400000,          // 1 day (default)
    autoCreateSessionOnInvalidId: false,     // Reject invalid session IDs
    maxSessions: 1000,                       // Optional limit
})
```

### SessionStore Interface

```typescript
interface SessionStore {
    get(sessionId: string): Promise<SessionMetadata | null>;
    set(sessionId: string, data: SessionMetadata): Promise<void>;
    delete(sessionId: string): Promise<void>;
    has(sessionId: string): Promise<boolean>;
    keys(): Promise<string[]>;
    setWithTTL?(sessionId: string, data: SessionMetadata, ttlMs: number): Promise<void>;
}
```

### StreamManager Interface

```typescript
interface StreamManager {
    create(sessionId: string, controller: ReadableStreamDefaultController): Promise<void>;
    send(sessionIds: string[] | undefined, data: string): Promise<void>;
    delete(sessionId: string): Promise<void>;
    has(sessionId: string): Promise<boolean>;
    close?(): Promise<void>;
}
```

## Deployment Patterns

| Pattern | SessionStore | StreamManager | Use Case |
|---|---|---|---|
| Development | InMemory | InMemory | Local development |
| Single instance | FileSystem | InMemory | Small production |
| Distributed | Redis | Redis (Pub/Sub) | Scaled production |
| Stateless | None | None | Serverless/edge |
