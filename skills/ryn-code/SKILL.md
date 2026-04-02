---
name: ryn-code
description: TypeScript code style and architecture guidelines. Enforces 200-line file limit, barrel exports, pure functions, Zod validation, and Effect.ts patterns. Use when writing TypeScript code, creating new files, refactoring, or reviewing code quality. Triggers on "code style", "new project", "refactor", "clean code".
---

# Code Style Guidelines

These are mandatory rules for all TypeScript projects. Follow them exactly.

## Critical Rules

1. **Max 200 lines per file** — Ideal target. Each file = one thing. Each function = one thing. Atomize and modularize.
2. **Barrel exports** — Every directory has an `index.ts` that re-exports its public API.
3. **Pure functions** — Default to pure. Side effects go in dedicated infrastructure modules. One function, one responsibility.
4. **TypeScript strict mode** — Always `"strict": true` in tsconfig.
5. **ESM only** — `"type": "module"` in package.json, `"module": "Node16"` in tsconfig.
6. **Tabs for indentation** — Not spaces.
7. **Validation** — Zod for standard projects. `@effect/schema` (Effect Schema) for Effect.ts projects. Never mix both.

## Project Structure

Separate concerns into three layers:

```
src/
├── index.ts              # Entry point only — wiring, no logic
├── types/                # Shared TypeScript types
│   └── index.ts
├── [domain]/             # Pure business logic (formatters, validators, schemas)
│   ├── index.ts
│   └── [feature].ts
├── [infrastructure]/     # Side-effectful code (DB, HTTP, queues, storage)
│   ├── index.ts
│   ├── client.ts         # Singleton clients
│   └── operations.ts     # CRUD / IO operations
└── [transport]/          # Entry layer (HTTP handlers, CLI, MCP tools)
    └── index.ts
```

### Layer Rules

- **Pure logic**: No imports from infrastructure. No side effects. Easy to test.
- **Infrastructure**: Wraps external services. Singleton clients. Operations as functions.
- **Transport**: Wires pure logic + infrastructure. Handles request/response.

## Atomization Philosophy

The 200-line limit is not an arbitrary cap — it enforces **atomic design**:

- **Each file = one concept** (one schema, one client, one set of related functions)
- **Each function = one thing** (transform, validate, fetch, format — never all at once)
- **Each module = one domain** (auth, products, storage — never mixed)

When something grows, split it. A 150-line file with two responsibilities is worse than two 80-line files with one each.

### Every file MUST:
- Have a single, clear responsibility
- Export through barrel `index.ts`
- Stay under 200 lines (ideal target)

### Splitting a file:
- Split by sub-responsibility, not by line count
- Create a directory with `index.ts` barrel
- Each sub-file handles one focused concern

### Barrel export pattern:
```typescript
// src/auth/index.ts
export { generateApiKey, hashApiKey } from "./api-key.js"
export { apiKeyAuth } from "./middleware.js"
export { adminAuth } from "./admin-auth.js"
```

## TypeScript Configuration

```json
{
	"compilerOptions": {
		"target": "ES2022",
		"module": "Node16",
		"moduleResolution": "Node16",
		"strict": true,
		"outDir": "dist",
		"rootDir": "src",
		"declaration": true,
		"esModuleInterop": true,
		"skipLibCheck": true
	},
	"include": ["src"]
}
```

## Validation

### Standard projects: Zod

- Define schemas close to where they are used
- Infer types from schemas: `type MyInput = z.infer<typeof MyInputSchema>`
- Validate at system boundaries (API input, config, external data)
- Do NOT validate internal function calls

```typescript
import { z } from "zod"

export const DocumentInputSchema = z.object({
	content: z.string().optional(),
	mimeType: z.string().optional(),
	url: z.string().url().optional(),
})

export type DocumentInput = z.infer<typeof DocumentInputSchema>
```

### Effect.ts projects: Effect Schema

When using Effect.ts, use `@effect/schema` — NOT Zod. Effect Schema integrates natively with the Effect ecosystem (encoding/decoding, error channel, services).

```typescript
import { Schema } from "@effect/schema"

const DocumentInput = Schema.Struct({
	content: Schema.optional(Schema.String),
	mimeType: Schema.optional(Schema.String),
	url: Schema.optional(Schema.String.pipe(Schema.filter((s) => URL.canParse(s)))),
})

type DocumentInput = Schema.Schema.Type<typeof DocumentInput>
```

## Effect.ts Usage

Effect.ts is used ONLY in critical/complex projects where error handling, dependency injection, and composability matter. For simple projects, use plain TypeScript with Zod.

### When to use Effect:
- Projects with complex error hierarchies
- Services requiring dependency injection
- Pipelines with multiple failure modes
- Long-running processes needing cancellation/retry

### When NOT to use Effect:
- Simple CRUD APIs
- Scripts or CLIs
- Projects where team members don't know Effect

### CRITICAL: AI models generate incorrect Effect.ts code.

Before writing ANY Effect code:
1. **Use the Effect MCP** — Install `@niklaserik/effect-mcp` to get latest docs in context
2. **Read official docs** — `https://effect.website/docs`
3. **Verify every API call** — Do not trust memorized APIs, they are likely outdated

### Effect patterns:
- Use `Effect.gen` for sequential effectful code (generator syntax)
- Use `pipe` for composition
- Use `Layer` for dependency injection
- Use `Schema` (NOT Zod) for validation/encoding/decoding
- Use `Effect.Service` for defining service interfaces
- Use `Effect.runPromise` or `Effect.runFork` at the edge

## Code Quality Checklist

Before finishing any file, verify:

- [ ] Under 200 lines
- [ ] Single responsibility
- [ ] Exported through barrel index.ts
- [ ] Pure functions separated from side effects
- [ ] Validation at boundaries (Zod for standard, Effect Schema for Effect projects)
- [ ] No `any` types — use `unknown` if needed
- [ ] No default exports — use named exports
- [ ] Imports use `.js` extension (ESM)
- [ ] Tabs for indentation
