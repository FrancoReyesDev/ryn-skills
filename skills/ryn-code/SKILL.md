---
name: ryn-code
description: TypeScript code style and architecture guidelines. Enforces 200-line file limit, barrel exports, pure functions, Zod validation, and Effect.ts patterns. Use when writing TypeScript code, creating new files, refactoring, or reviewing code quality. Triggers on "code style", "new project", "refactor", "clean code".
---

# Code Style Guidelines

These are mandatory rules for all TypeScript projects. Follow them exactly.

## Critical Rules

1. **Max 200 lines per file** — No exceptions. Split into focused modules.
2. **Barrel exports** — Every directory has an `index.ts` that re-exports its public API.
3. **Pure functions** — Default to pure. Side effects go in dedicated infrastructure modules.
4. **TypeScript strict mode** — Always `"strict": true` in tsconfig.
5. **ESM only** — `"type": "module"` in package.json, `"module": "Node16"` in tsconfig.
6. **Tabs for indentation** — Not spaces.
7. **Zod for validation** — All external input validated with Zod schemas.

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

## File Organization

### Every file MUST:
- Have a single, clear responsibility
- Export through barrel `index.ts`
- Stay under 200 lines

### Splitting a file:
- If a file approaches 200 lines, split by sub-responsibility
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

## Validation with Zod

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
	gcsUri: z.string().startsWith("gs://").optional(),
})

export type DocumentInput = z.infer<typeof DocumentInputSchema>
```

## Effect.ts Usage

Effect.ts is used ONLY in critical/complex projects where error handling, dependency injection, and composability matter. For simple projects, use plain TypeScript.

### When to use Effect:
- Projects with complex error hierarchies
- Services requiring dependency injection
- Pipelines with multiple failure modes
- Long-running processes needing cancellation/retry

### When NOT to use Effect:
- Simple CRUD APIs
- Scripts or CLIs
- Projects where team members don't know Effect

### Key principle:
AI models often generate incorrect Effect.ts code. When working with Effect:
- Read the official docs at `https://effect.website/docs` before writing Effect code
- Prefer `pipe` + `Effect.gen` over manual chaining
- Use `Layer` for dependency injection
- Use `Schema` instead of Zod in Effect projects
- Always verify generated Effect code against docs

## Code Quality Checklist

Before finishing any file, verify:

- [ ] Under 200 lines
- [ ] Single responsibility
- [ ] Exported through barrel index.ts
- [ ] Pure functions separated from side effects
- [ ] Zod schemas for external input
- [ ] No `any` types — use `unknown` if needed
- [ ] No default exports — use named exports
- [ ] Imports use `.js` extension (ESM)
- [ ] Tabs for indentation
