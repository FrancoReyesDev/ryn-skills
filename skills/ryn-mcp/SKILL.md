---
name: ryn-mcp
description: Build MCP servers from REST APIs or services. Analyzes API nature, selects transport (stdio/StreamableHTTP/Cloudflare Agents), designs tools by domain, implements auth, rate limiting, and retry. Use when building MCP servers, wrapping APIs for AI agents, or designing tool interfaces. Triggers on "MCP server", "build MCP", "wrap API", "create tools for".
---

# MCP Server Builder

Build production-grade MCP servers following consistent architecture patterns.

## Process

### Phase 1: Analyze the API

Before writing code, understand the API:

1. **Fetch and read API documentation** — Use WebFetch or read local docs
2. **Identify domains** — Group endpoints by business domain (e.g., products, orders, users)
3. **Identify auth mechanism** — API key, OAuth, OIDC, session tokens
4. **Check rate limits** — Per-endpoint or global limits
5. **Check response patterns** — Pagination, error formats, envelope structures
6. **Identify long-running operations** — Anything that takes >30 seconds needs async handling

### Phase 2: Choose Transport

Select transport based on deployment context:

| Scenario | Transport | When |
|----------|-----------|------|
| **Local CLI tool** | stdio | Single user, runs on their machine |
| **Remote service (simple)** | StreamableHTTP | Cloud Run, serverless, multi-user |
| **Remote service (Cloudflare)** | Cloudflare Agents | Durable Objects for state, Workers for compute |

#### stdio (local)
- Long-lived process per Claude instance
- In-memory state (token cache, rate limiter)
- Config file at `~/.config/{server-name}/config.json`
- Entry: `node dist/index.js` via Claude Desktop config

#### StreamableHTTP (remote)
- Express/Hono app with `POST/GET/DELETE /mcp` endpoints
- Session management via `mcp-session-id` header
- API key auth middleware
- Deploy to Cloud Run or similar

#### Cloudflare Agents (remote)
- Uses `@cloudflare/agents` with Durable Objects
- Each session = Durable Object instance
- Built-in WebSocket support
- Deploy to Cloudflare Workers

### Phase 3: Design Tools

#### Naming convention
- `{domain}_{action}` format: `products_search`, `orders_create`
- Action verbs: `get`, `search`, `create`, `update`, `delete`, `list`
- Be descriptive — LLMs use tool names to decide which tool to call

#### Tool design rules
- **One tool per operation** — Don't combine create + update
- **Zod schemas for all inputs** — With descriptions on each field
- **Return structured text** — Markdown or plain text, not raw JSON
- **Include pagination** — For any list endpoint
- **Descriptive tool descriptions** — What it does + when to use it + what it returns

#### Grouping by domain
```
src/tools/
├── index.ts              # Barrel: registers all tools on server
├── schema.ts             # Shared Zod schemas
├── config.ts             # Credential management tools
├── products.ts           # Product domain tools
├── orders.ts             # Order domain tools
├── inventory.ts          # Inventory domain tools
└── helpers.ts            # Response formatting utilities
```

### Phase 4: Implement

#### Project structure
```
src/
├── index.ts              # Entry point — create server, connect transport
├── server.ts             # McpServer factory — registers all tools
├── types/
│   └── index.ts          # Shared types
├── client/               # HTTP + Auth layer
│   ├── index.ts
│   ├── http.ts           # HTTP client with auth headers
│   ├── auth.ts           # Token acquisition/renewal
│   ├── rate-limiter.ts   # Token bucket or sliding window
│   └── retry.ts          # Exponential backoff for transient errors
├── tools/                # MCP tools grouped by domain
│   ├── index.ts
│   └── [domain].ts
└── [worker.ts]           # Only if async processing needed
```

#### Auth implementation
```typescript
// client/auth.ts — Token cache with auto-renewal
let cachedToken: { token: string; expiresAt: number } | null = null

export const getToken = async (config: Config): Promise<string> => {
	if (cachedToken && Date.now() < cachedToken.expiresAt - 60_000) {
		return cachedToken.token
	}
	const response = await fetch(`${config.baseUrl}/token`, {
		method: "POST",
		body: new URLSearchParams({
			grant_type: "client_credentials",
			client_id: config.clientId,
			client_secret: config.clientSecret,
		}),
	})
	const data = await response.json()
	cachedToken = {
		token: data.access_token,
		expiresAt: Date.now() + data.expires_in * 1000,
	}
	return cachedToken.token
}
```

#### Rate limiter (token bucket)
```typescript
// client/rate-limiter.ts
export const createRateLimiter = (tokensPerWindow: number, windowMs: number) => {
	let tokens = tokensPerWindow
	let lastRefill = Date.now()

	return async (): Promise<void> => {
		const now = Date.now()
		const elapsed = now - lastRefill
		tokens = Math.min(tokensPerWindow, tokens + (elapsed / windowMs) * tokensPerWindow)
		lastRefill = now

		if (tokens < 1) {
			const waitMs = ((1 - tokens) / tokensPerWindow) * windowMs
			await new Promise((resolve) => setTimeout(resolve, waitMs))
			tokens = 0
		} else {
			tokens -= 1
		}
	}
}
```

#### Retry strategy
```typescript
// client/retry.ts
const RETRYABLE_CODES = [429, 500, 502, 503, 504]

export const withRetry = async <T>(
	fn: () => Promise<T>,
	maxAttempts = 3,
): Promise<T> => {
	for (let attempt = 1; attempt <= maxAttempts; attempt++) {
		try {
			return await fn()
		} catch (error) {
			const status = (error as any)?.status
			if (attempt === maxAttempts || !RETRYABLE_CODES.includes(status)) throw error
			await new Promise((r) => setTimeout(r, 2 ** attempt * 1000))
		}
	}
	throw new Error("Unreachable")
}
```

### Phase 5: Async Processing (if needed)

For APIs with long-running operations, use the queue pattern:

1. Tool enqueues a task and returns immediately with a task ID
2. Worker processes the task asynchronously
3. `get_result` tool retrieves the result by task ID

This requires infrastructure: queue service + storage for results.

## Code Standards

All code MUST follow `/ryn-skills:ryn-code` guidelines:
- Max 200 lines per file
- Barrel exports
- Pure functions separated from infrastructure
- Zod for input validation
- Tabs, ESM, strict TypeScript

## SDK References

Before writing MCP code, fetch the latest SDK docs:
- **TypeScript SDK**: `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md`
- **MCP Protocol**: `https://modelcontextprotocol.io/specification/draft.md`
- **Cloudflare Agents**: `https://developers.cloudflare.com/agents/`

## Quality Checklist

- [ ] Every tool has Zod schema with field descriptions
- [ ] Tool descriptions explain WHAT, WHEN, and WHAT IT RETURNS
- [ ] Auth tokens are cached and auto-renewed
- [ ] Rate limiting implemented per API requirements
- [ ] Retry with exponential backoff on transient errors
- [ ] Pagination supported on list endpoints
- [ ] Responses formatted as readable text (not raw JSON)
- [ ] Error messages are actionable (tell the LLM what to do next)
- [ ] All files under 200 lines with barrel exports
