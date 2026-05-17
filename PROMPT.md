# Plannotator Phases 2 & 3 — Coding Agent Implementation Prompt

> **Purpose:** Hand this file (along with `IMPLEMENTATION_PLAN.md` and `TASKS.json`) to a coding agent (Jules, Claude Code, Copilot Workspace, etc.) to drive the full Phase 2 + Phase 3 implementation.

---

## Context

You are implementing **Phases 2 and 3** of the Plannotator project — a self-hosted agentic review platform built on Cloudflare Workers. Plannotator is currently a Bun-based plan review UI that runs locally as a Claude Code plugin. Phases 2 & 3 extend it with a cloud-native backend and add four cross-cutting capabilities: pattern ingestion, schema discipline, unified AI abstractions, and lifecycle artifact generation.

**The existing code must not be broken.** All changes are additive. The existing `apps/hook`, `apps/opencode-plugin`, `apps/marketing`, `packages/ui`, `packages/editor`, `packages/review-editor`, `packages/server`, and `packages/shared` packages continue to work exactly as before.

---

## Repository

- **Repo root:** `/home/runner/work/plannotator/plannotator` (or wherever it is cloned)
- **Runtime:** Bun (for local dev), Cloudflare Workers (for deployment)
- **Package manager:** Bun workspaces
- **Build:** `bun run build` at repo root
- **Existing builds verified:** `bun run build:hook && bun run build:opencode` must still pass after your changes

---

## What to Build

All deliverables are described in `IMPLEMENTATION_PLAN.md`. Execute in the order defined by `TASKS.json#executionOrder` — complete all P2-* tasks before starting P3-* tasks.

**Phase 2 — Three cloud-native backend pillars:**

1. **Self-Correcting Standards Engine** (`StandardsAgent`, learning queue, rules API, Astro panel)
2. **Hub-and-Spoke Multi-Agent Mesh** (`OrchestratorAgent`, `UXSpecialistAgent`, `PlatformSpecialistAgent`, `CloudflareDocsMcp`)
3. **Real-Time Backlog Service** (`BacklogAgent`, WebSocket pub/sub, Hono REST API for external agents)

**Phase 3 — Four cross-cutting capabilities:**

4. **Pattern Ingestion & Template Injection** (`PatternIngestionMcp`, template mapping pass in `OrchestratorAgent`, specialist tailoring pass)
5. **Schema Modularization & Auto-CRUD** (per-table isolated files under `src/backend/db/schemas/`, `drizzle-zod` admin surface, journey routes)
6. **Unified AI Abstractions** (`ModelProvider`, `BackgroundOperationAgent`, `FrontendInteractiveAgent`, `getAgentByName()` mandate)
7. **Lifecycle Artifacts & Anti-Vacuum Continuity** (`LifecycleArtifactAgent`, AGENTS.md generation, `.agent/rules/`, D1 plan ledger)

---

## Step-by-Step Instructions

### 1. Read before writing

Before touching a single file:

1. Read `IMPLEMENTATION_PLAN.md` in full — it is the authoritative reference.
2. Read `TASKS.json` — follow the `executionOrder` array exactly. Each task lists the exact files to create/modify and its acceptance criteria.
3. Read `CLAUDE.md` — understand the existing project structure, build commands, and conventions.
4. Read `packages/shared/src/index.ts` (if it exists) and `packages/ui/types.ts` to understand existing shared types.
5. Scan `.agent/rules/` — if any rule files exist, they are binding constraints for this session.

### 2. Work task-by-task

Process tasks in the order given by `TASKS.json#executionOrder`. Do not skip tasks or reorder them — each builds on the previous.

For each task:
- Create or modify only the files listed in `task.files`
- Implement exactly what `task.description` says
- Verify all items in `task.acceptance` are satisfied before moving to the next task
- Commit after each task with message: `feat(phase2): [task.id] [task.title]` or `feat(phase3): [task.id] [task.title]`

### 3. Agent class conventions

All Cloudflare Agents SDK classes must follow these patterns:

```typescript
// Every agent file structure
import { Agent } from 'agents';
import { callable } from 'agents';
import type { Env } from '../index';

export class MyAgent extends Agent<Env, MyState> {
  initialState: MyState = { /* ... */ };

  async onStart() { /* initialization */ }

  // Methods callable from external WS clients use @callable
  @callable()
  async myPublicMethod(arg: string): Promise<string> { /* ... */ }

  // Methods called via DO RPC from other agents do NOT need @callable
  async myInternalMethod(arg: string): Promise<string> { /* ... */ }
}
```

Key rules:
- `@callable()` is for **external** (browser/non-Worker) clients only
- Internal Worker-to-DO and DO-to-DO calls use native Durable Object RPC — no `@callable` needed
- Every agent class must be **exported by name** from `apps/worker/src/index.ts`
- Sub-agent names must be namespaced by session: `'ux-' + this.name`, `'pl-' + this.name`

### 4. Database conventions

- All D1 writes for backlog mutations **must go through `BacklogAgent`** — never write D1 directly from Hono route handlers for mutation endpoints
- Hot SQLite (`this.sql`) writes happen **before** `this.broadcast()` and **before** `waitUntil(mirrorToD1())`
- Use `import { waitUntil } from 'cloudflare:workers'` for non-blocking D1 mirrors
- **Phase 3:** All Drizzle schema files live under `apps/worker/src/backend/db/schemas/{domain}/{table_name}.ts` — one file per table, no exceptions
- Run `bunx drizzle-kit generate` after adding schema files, then `wrangler d1 migrations apply --local`

### 5. Type safety requirements

- TypeScript strict mode must pass: `tsc --noEmit` with zero errors
- Use Zod for all external input validation (HTTP request bodies, WS message payloads, LLM JSON outputs)
- The shared types in `packages/shared/src/rules.ts`, `packages/shared/src/review.ts`, and `packages/shared/src/patterns.ts` are the **single source of truth** — import them in both agent and frontend code

### 6. Hono router conventions

```typescript
// apps/worker/src/routes/tasks.ts
import { Hono } from 'hono';
import { zValidator } from '@hono/zod-validator';
import { validateBearer } from '../lib/auth';
import type { Env } from '../index';

const tasks = new Hono<{ Bindings: Env }>();

tasks.use('*', validateBearer);

tasks.get('/:projectId', async (c) => { /* ... */ });
// etc.

export { tasks };
```

Mount all sub-routers in `apps/worker/src/router.ts`.

### 7. MCP tool access

Sub-agents connect to `CloudflareDocsMcp` via the DO binding, **not** via an HTTP URL:

```typescript
// In UXSpecialistAgent.onStart() and PlatformSpecialistAgent.onStart()
await this.addMcpServer('cf-docs', this.env.CF_DOCS_MCP);
// CF_DOCS_MCP must be declared in Env and bound in wrangler.jsonc
```

### 8. WebSocket broadcast pattern

The exact ordering for every task mutation is mandatory — do not change the sequence:

```typescript
// 1. Validate
// 2. Write hot SQLite (synchronous)
this.sql`UPDATE hot_tasks SET status = ${s}, updated_at = ${now} WHERE id = ${id}`;
// 3. Broadcast (before D1, so UI feels instant)
this.broadcast(JSON.stringify({ type: 'task_updated', taskId, newStatus: s }));
// 4. Mirror to D1 (async, non-blocking)
import { waitUntil } from 'cloudflare:workers';
waitUntil(this.mirrorToD1(taskId, s, note, agentId));
```

### 9. Per-connection state (hibernation-safe)

```typescript
// In onConnect():
connection.setState({ userId, role, connectedAt: Date.now() });
// This is persisted via serializeAttachment under the hood.

// In onMessage():
const { userId } = connection.state; // Works even after hibernation
```

### 10. wrangler.jsonc completeness check

Before considering Phase 2 done, verify `apps/worker/wrangler.jsonc` contains:
- All 6 Phase 2 DO bindings with correct class names matching exports
- Both queue consumer and producer bindings
- Vectorize binding
- AI binding
- D1 binding
- Migration tags v1 and v2 with the correct `new_sqlite_classes` arrays

Before considering Phase 3 done, also verify:
- PATTERN_INGESTION_MCP and LIFECYCLE_ARTIFACT_AGENT DO bindings
- ARTIFACTS R2 bucket binding
- Migration tag v3 with new_sqlite_classes
- All secret vars documented (STANDARDIZATION_REPO_TOKEN, OPENAI_API_KEY, ANTHROPIC_API_KEY, GEMINI_API_KEY, CF_ACCOUNT_ID, AI_GATEWAY_ID, ADMIN_BEARER_TOKEN)

### 11. ModelProvider mandate (Phase 3)

All LLM inference calls must use `ModelProvider`. This is a hard rule:

```typescript
// ✅ Correct — always use ModelProvider
const ai = new ModelProvider(this.env);
const text = await ai.generateText({ prompt: '...' });
const data = await ai.generateStructured({ prompt: '...', schema: MySchema });

// ❌ Wrong — never call env.AI.run() directly outside ModelProvider
const result = await this.env.AI.run('@cf/model', { ... });
```

### 12. Agent factory mandate (Phase 3)

All agent instantiation must use `getAgentByName()`. This is a hard rule:

```typescript
import { getAgentByName } from 'agents';

// ✅ Correct — SDK factory
const agent = await getAgentByName<OrchestratorAgent>(
  env.ORCHESTRATOR_AGENT,
  `session-${planVersionId}`
);

// ❌ Wrong — raw Durable Object primitives
const id   = env.ORCHESTRATOR_AGENT.idFromName(`session-${planVersionId}`);
const stub = env.ORCHESTRATOR_AGENT.get(id);
```

### 13. Base class inheritance (Phase 3)

- Non-interactive background agents extend `BackgroundOperationAgent`
- Real-time user-facing agents extend `FrontendInteractiveAgent`
- MCP agents extend `McpAgent` (unchanged)
- **Never** extend the raw `Agent` class directly in new code

### 14. Schema file discipline (Phase 3)

- One Drizzle table = one `.ts` file under `apps/worker/src/backend/db/schemas/{domain}/{table_name}.ts`
- `schemas/index.ts` is the **only** re-export file — no other file imports from `index.ts` within the schemas directory
- Journey route files have their own independent Zod schemas — do not reuse drizzle-zod generated schemas in journey routes

---

## What Not to Do

- **Do not modify** any file in `apps/hook/`, `apps/opencode-plugin/`, `packages/ui/`, `packages/editor/`, `packages/review-editor/`, or `packages/server/`
- **Do not add** new root-level config files unless a task explicitly requires it
- **Do not use** `fetch()` for internal agent-to-agent communication — always use DO RPC or sub-agent stubs
- **Do not write** D1 directly from Hono mutation handlers — all mutations route through agent `@callable` methods
- **Do not** put `@callable` on methods only called internally between agents
- **Do not** create new test files unless a task explicitly requires it — the existing test suite must still pass
- **Do not** call `env.AI.run()` outside of `ModelProvider` (Phase 3)
- **Do not** use `idFromName()` + `.get()` to instantiate agents — always use `getAgentByName()` (Phase 3)
- **Do not** put multiple Drizzle table definitions in a single schema file (Phase 3)
- **Do not** reuse drizzle-zod generated schemas in journey route files (Phase 3)

---

## Verification Checklist

Run these checks after completing all Phase 2 tasks, and again after all Phase 3 tasks:

```bash
# 1. Existing builds still pass
bun run build:hook
bun run build:opencode

# 2. New worker builds
cd apps/worker && bunx wrangler dev --dry-run

# 3. TypeScript clean
cd apps/worker && tsc --noEmit

# 4. D1 migrations apply cleanly (all 3 migration tags)
cd apps/worker && bunx wrangler d1 migrations apply plannotator-db --local

# 5. Vectorize index exists
bunx wrangler vectorize list

# 6. Root workspace resolves
cd /repo/root && bun install

# Phase 3 additional checks:

# 7. No direct env.AI.run() calls outside ModelProvider
grep -r "env\.AI\.run" apps/worker/src --include="*.ts" \
  | grep -v "model-provider.ts" | wc -l
# Expected: 0

# 8. No raw DO instantiation patterns
grep -r "idFromName\|\.get(id)" apps/worker/src --include="*.ts" | wc -l
# Expected: 0

# 9. Schema isolation check
find apps/worker/src/backend/db/schemas -name "*.ts" ! -name "index.ts" \
  | xargs grep -l "sqliteTable" | while read f; do
    count=$(grep -c "sqliteTable" "$f")
    [ "$count" -gt 1 ] && echo "VIOLATION: $f has $count tables"
  done
# Expected: no output

# 10. Admin CRUD surface responds
curl -s -H "Authorization: Bearer test" \
  http://localhost:8787/api/admin/tables/tasks | jq '.[] | .id' | head -3
```

All ten checks must pass with zero errors before the implementation is considered complete.

---

## Questions / Ambiguities

If you encounter an ambiguity not covered by this prompt or `IMPLEMENTATION_PLAN.md`, resolve it using these tiebreakers in order:

1. Prefer the Cloudflare Agents SDK official pattern over any generic Workers pattern
2. Prefer `getAgentByName()` over raw Durable Object instantiation — always
3. Prefer `ModelProvider` methods over direct inference calls — always
4. Prefer Durable Object RPC over any form of HTTP for internal communication
5. Prefer strong types (TypeScript + Zod) over `any` / unknown
6. Prefer immutable audit trails (append-only assignment log) over destructive updates
7. Prefer per-table isolated schema files over any aggregated schema approach
8. When in doubt about a DB write path, route it through the relevant Agent's `@callable` method

---

## Deliverables Expected

When complete, open a pull request with:

**Phase 2:**
- All new files under `apps/worker/src/` (agents, routes, lib)
- New shared types under `packages/shared/src/` (rules.ts, review.ts)
- New Astro page and React island under `apps/marketing/`
- Updated root `package.json` with `dev:worker` and `build:worker` scripts

**Phase 3:**
- All new files under `apps/worker/src/backend/` (ai/providers, ai/agents, agents, db/schemas, routes)
- New shared type `packages/shared/src/patterns.ts`
- `.agent/rules/.gitkeep` (directory scaffold for generated rule files)
- Updated `CLAUDE.md` and `AGENTS.md`

The PR title should be: `feat: Phases 2 & 3 — Self-Hosted Agentic Plannotator`

---

*Last updated: 2026-05-17*  
*Reference: IMPLEMENTATION_PLAN.md · TASKS.json*
