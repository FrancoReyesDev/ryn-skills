# ryn-skills

Personal Claude Code skills plugin for consistent engineering workflows.

## Skills

| Skill | Description |
|-------|-------------|
| `ryn-code` | TypeScript code style guidelines — 200-line files, barrel exports, pure functions, Zod, Effect.ts |
| `ryn-mcp` | Build MCP servers from REST APIs — transport selection, tool design, auth, rate limiting |
| `ryn-infra` | Serverless infrastructure patterns — Cloudflare-first, GCP when needed, queues, workflows |

## Installation

```bash
/plugin install https://github.com/FrancoReyesDev/ryn-skills
```

## Usage

```
/ryn-skills:ryn-code
/ryn-skills:ryn-mcp [API documentation URL or description]
/ryn-skills:ryn-infra [project requirements]
```
