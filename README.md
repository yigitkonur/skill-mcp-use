# skill-mcp-use

> **other skills by [@yigitkonur](https://github.com/yigitkonur):**
> [testing mcp servers](https://github.com/yigitkonur/skill-mcp-server-tester) · [extracting design dna from dashboards](https://github.com/yigitkonur/skill-design-soul-saas) · [converting saved webpages to next.js](https://github.com/yigitkonur/skill-snapshot-to-nextjs) · [generating devin review config](https://github.com/yigitkonur/skill-devin-review-init) · [generating greptile review config](https://github.com/yigitkonur/skill-greptile-init) · [mcp server for searching skills](https://github.com/yigitkonur/mcp-skills-as-context) · [tauri observability & mcp bridge](https://github.com/yigitkonur/skill-tauri-mcp)

a claude code skill that reviews, tests, and migrates python applications built with the [`mcp-use`](https://github.com/mcp-use/mcp-use) library. catches the 6 derailment patterns ai agents fall into when working with mcp-use codebases — confusing server sdks with client sdks, botching async lifecycles, using wrong schema field names, and more.

## what it does

point this skill at any python codebase that uses `mcp-use` (`pip install mcp-use`) and it gives the reviewing agent deep knowledge of:

- **sdk disambiguation** — which import belongs where. `MCPClient` for consuming tools, `mcp.server.Server` for building servers, `MCPServer` for production servers. agents constantly mix these up.
- **6 derailment patterns** — the exact mistakes ai agents make when writing mcp-use code, each with detection signals and fixes.
- **async lifecycle** — the precise order of `MCPClient()` → `create_session()` → `session.list_tools()` → `session.call_tool()` → cleanup, and what breaks when you skip a step.
- **config validation** — stdio, http/sse, and websocket transport configs with the exact dict keys that trigger each connector.
- **langchain bridge** — how mcp-use converts mcp tools into langchain `BaseTool` instances, version requirements, correct import paths.
- **migration mapping** — side-by-side guide for moving from the official `@modelcontextprotocol/sdk` typescript patterns to mcp-use equivalents.

## the 6 derailment patterns

these are the specific mistakes ai coding agents make when working with mcp-use code:

| # | pattern | what goes wrong |
|---|---------|----------------|
| 1 | **server sdk vs. client sdk conflation** | agent imports `from mcp.server import Server` in a file that should be consuming tools via `MCPClient` |
| 2 | **@tool decorator misapplication** | agent uses claude agent sdk's `@tool` decorator thinking it connects to external servers — mcp-use *discovers* tools, it doesn't define them |
| 3 | **schema field name mismatch** | agent writes `parameters:` (openai style) or `inputSchema:` (ts sdk) instead of `input_schema` (mcp-use python) |
| 4 | **missing async lifecycle steps** | agent calls `client.get_session()` before `await client.create_session()`, or skips `await` on tool calls |
| 5 | **tool name format incompatibility** | tools with dots in names break openai's `^[a-zA-Z0-9_-]+$` regex — mcp-use doesn't namespace, names come straight from servers |
| 6 | **langchain import path mismatch** | agent uses deprecated `from langchain.llms` instead of `from langchain_openai` or `from langchain_anthropic` |

each pattern includes the exact code the agent writes, why it fails, how to detect it with grep, and the correct replacement.

## what makes this different

- **grounded in the real api** — every class name, method signature, and config key was extracted from the actual mcp-use source code and docs. no guessing.
- **migration-aware** — includes a full mapping from official typescript sdk patterns (`registerTool`, `StdioServerTransport`, manual session maps) to mcp-use equivalents (`server.tool()`, `server.listen()`, pluggable session stores).
- **detection-oriented** — every derailment pattern has a concrete grep command. the review checklist has 22 items, all verifiable by searching the codebase.
- **covers the full stack** — client consumption (python), server creation (typescript + python), langchain bridge, auth flows, session storage, deployment.

### usage

```
review my mcp-use integration and check for the 6 derailment patterns
```

```
is my async lifecycle correct? check create_session, list_tools, call_tool order
```

```
migrate this typescript mcp server code to mcp-use python
```

```
validate my mcp config dict — am i using the right transport keys?
```

### file overview

| file | lines | what it covers |
|------|-------|---------------|
| `SKILL.md` | 775 | entry point — disambiguation table, config schema, async lifecycle, 6 derailment patterns, 22-item review checklist, test patterns, error signatures, migration guide |
| `references/migrating-from-ts-sdk.md` | 494 | side-by-side migration from official `@modelcontextprotocol/sdk` to mcp-use — server creation, tools, resources, prompts, transport, sessions, auth |
| `references/advanced-features.md` | 449 | elicitation, sampling, subscriptions, notifications, logging, widget responses |
| `references/authentication-and-sessions.md` | 360 | oauth providers (auth0, supabase, workos, keycloak), session storage (memory, filesystem, redis), user context |
| `references/deployment-patterns.md` | 328 | local dev, manufact cloud, supabase edge functions, google cloud run, docker, cli commands |
| `references/tools-deep-dive.md` | 304 | tool definition, schema formats (`inputSchema` vs `input_schema`), discovery, invocation, response helpers (`text()`, `object()`, `error()`) |
| `references/resources-and-prompts.md` | 271 | mcp resources (static, dynamic, templates) and prompt definitions with argument schemas |
| `references/async-lifecycle.md` | 265 | full async lifecycle diagram, correct code template, common mistakes and their error signatures |
| `references/langchain-bridge.md` | 239 | how `LangChainAdapter` converts mcp tools to langchain `BaseTool`, version requirements, agent integration |
| `references/server-creation-patterns.md` | 221 | comparing server creation: official ts sdk vs mcp-use ts vs mcp-use python |
| `references/server-config-patterns.md` | 217 | stdio, http/sse, websocket config dicts with exact keys that trigger each connector |

**total: 3,923 lines across 11 files.**

### scope

**built for:** python codebases using the [`mcp-use`](https://github.com/mcp-use/mcp-use) library (`pip install mcp-use`). reviews, tests, and migrations of mcp client code.

**not for:** building mcp servers (use the official `@modelcontextprotocol/sdk`). not for typescript mcp clients. not for testing mcp servers (use [skill-mcp-server-tester](https://github.com/yigitkonur/skill-mcp-server-tester) instead).

### install

```bash
npx skills add yigitkonur/skill-mcp-use
```

> works with claude code, cursor, codex, copilot, windsurf, and [30+ other agents](https://skills.sh).

### license

mit
