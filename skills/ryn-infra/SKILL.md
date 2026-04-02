---
name: ryn-infra
description: Serverless infrastructure decision guide. Cloudflare-first (Workers, Queues, Workflows, Durable Objects, KV, R2, D1), GCP when needed (Cloud Run, Document AI, Firestore). Use when choosing infrastructure, deploying services, designing system architecture, or deciding between cloud providers. Triggers on "deploy", "infrastructure", "serverless", "cloudflare", "cloud run", "where to host".
---

# Infrastructure Guide

Serverless-first infrastructure decisions. Use cloud primitives intelligently.

## Decision Framework

### Default: Cloudflare

Use Cloudflare for most projects. It provides a complete serverless ecosystem with excellent DX and pricing.

### Exception: GCP (or other cloud)

Use GCP when you need:
- Specific managed services (Document AI, BigQuery, Vertex AI)
- Long-running processes (>30s CPU time)
- Heavy compute that exceeds Workers limits
- Services not available on Cloudflare

## Cloudflare Stack

### Compute

| Service | Use When |
|---------|----------|
| **Workers** | HTTP handlers, API routes, webhooks, short tasks (<30s) |
| **Durable Objects** | Stateful coordination, WebSockets, real-time, MCP sessions |
| **Workflows** | Multi-step durable processes, retries, long-running orchestration |
| **Queues** | Async task processing, decoupling producers/consumers |
| **Cron Triggers** | Scheduled tasks (cleanup, sync, reports) |

### Storage

| Service | Use When |
|---------|----------|
| **KV** | Config, feature flags, cached data (eventually consistent) |
| **R2** | Files, blobs, documents, images (S3-compatible) |
| **D1** | Relational data, structured queries (SQLite at edge) |
| **Durable Object storage** | Per-object state, counters, session data |

### Architecture Patterns

#### Simple API
```
Client → Worker → D1/KV
```

#### API with async processing
```
Client → Worker → Queue → Consumer Worker → R2/D1
```

#### MCP Server (remote, Cloudflare)
```
LLM → Worker → Durable Object (Agent)
                    ├── External API
                    ├── D1 (state)
                    └── KV (cache)
```

#### Durable workflow
```
Client → Worker → Workflow
                    ├── Step 1: Fetch data
                    ├── Step 2: Process (with retry)
                    ├── Step 3: Store results
                    └── Step 4: Notify
```

#### Webhook receiver with notifications
```
External → Worker → Queue → Consumer → Telegram/Slack/Email
```

### Cloudflare Agents for MCP

When building remote MCP servers on Cloudflare:

- Use `@cloudflare/agents` package
- Each MCP session maps to a Durable Object
- Built-in WebSocket transport
- State persists across requests within the same session
- Fetch latest docs: `https://developers.cloudflare.com/agents/`

### Workers Configuration

```toml
# wrangler.toml
name = "my-service"
main = "src/index.ts"
compatibility_date = "2024-12-01"

[vars]
ENVIRONMENT = "production"

[[queues.producers]]
queue = "tasks"
binding = "TASK_QUEUE"

[[queues.consumers]]
queue = "tasks"
max_batch_size = 10

[[d1_databases]]
binding = "DB"
database_name = "my-db"
database_id = "..."

[durable_objects]
bindings = [
  { name = "MCP_AGENT", class_name = "McpAgent" }
]
```

## GCP Stack

### When to use GCP

- **Cloud Run**: Containerized services needing >30s execution, custom runtimes
- **Cloud Tasks**: Rate-limited async processing with OIDC auth
- **Firestore**: Document database with real-time listeners
- **Cloud Storage**: Large file storage with lifecycle policies
- **Document AI / Vertex AI**: ML services not available elsewhere
- **Cloud Scheduler**: Cron-like triggers for Cloud Run endpoints

### Cloud Run Pattern

```dockerfile
FROM node:22-slim AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:22-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./
ENV NODE_ENV=production PORT=8080
EXPOSE 8080
CMD ["node", "dist/index.js"]
```

Deploy:
```bash
gcloud run deploy SERVICE_NAME \
  --source . \
  --region us-central1 \
  --memory 2Gi \
  --timeout 600 \
  --allow-unauthenticated
```

### GCP + Async Processing Pattern
```
Client → Cloud Run → Cloud Tasks (rate-limited queue)
                          ↓
                     Cloud Run /worker (OIDC verified)
                          ├── Process data
                          ├── Store results in GCS
                          └── Update status in Firestore
```

## General Principles

1. **Serverless first** — No VMs, no Kubernetes, no self-managed infrastructure
2. **Use cloud primitives** — Queues, workflows, scheduled tasks, object storage. Don't reinvent them.
3. **Separate compute from storage** — Workers + D1/KV/R2, not monolith with local state
4. **Design for failure** — Retry with backoff, idempotent operations, dead letter queues
5. **Minimize cold starts** — Keep bundles small, lazy-load heavy dependencies
6. **Environment config** — Use platform env vars (Cloudflare vars, Cloud Run env), not `.env` files in production
7. **Cost-aware** — Cloudflare Workers are free tier friendly. Cloud Run bills per request/CPU. Choose based on usage patterns.

## Decision Checklist

When choosing infrastructure for a new project:

- [ ] Can this run on Cloudflare Workers? (default yes)
- [ ] Does it need >30s execution? → Cloud Run or Workflows
- [ ] Does it need specific cloud services? → That cloud provider
- [ ] Does it need real-time/WebSocket? → Durable Objects
- [ ] Does it need async processing? → Queues (CF) or Cloud Tasks (GCP)
- [ ] Does it need durable multi-step execution? → Workflows (CF) or Cloud Tasks chain (GCP)
- [ ] Does it need file storage? → R2 (CF) or Cloud Storage (GCP)
- [ ] Does it need relational data? → D1 (CF) or managed Postgres
