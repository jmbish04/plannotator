# Plannotator Phase 2 — Coding Agent Implementation Prompt

> **Purpose:** Hand this file (along with `IMPLEMENTATION_PLAN.md` and `TASKS.json`) to a coding agent (Jules, Claude Code, Copilot Workspace, etc.) to drive the full Phase 2 implementation.

---

## Context

You are implementing **Phase 2** of the Plannotator project — a self-hosted agentic review platform built on Cloudflare Workers. Plannotator is currently a Bun-based plan review UI that runs locally as a Claude Code plugin. Phase 2 extends it with a cloud-native backend.

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

Three deliverables described in `IMPLEMENTATION_PLAN.md`:

1. **Self-Correcting Standards Engine** (`StandardsAgent`, learning queue, rules API, Astro panel)
2. **Hub-and-Spoke Multi-Agent Mesh** (`OrchestratorAgent`, `UXSpecialistAgent`, `PlatformSpecialistAgent`, `CloudflareDocsMcp`)
3. **Real-Time Backlog Service** (`BacklogAgent`, WebSocket pub/sub, Hono REST API for external agents)

---

## Step-by-Step Instructions

### 1. Read before writing

Before touching a single file:

1. Read `IMPLEMENTATION_PLAN.md` in full — it is the authoritative reference.
2. Read `TASKS.json` — follow the `executionOrder` array exactly. Each task lists the exact files to create/modify and its acceptance criteria.
3. Read `CLAUDE.md` — understand the existing project structure, build commands, and conventions.
4. Read `packages/shared/src/index.ts` (if it exists) and `packages/ui/types.ts` to understand existing shared types.

### 2. Work task-by-task

Process tasks in the order given by `TASKS.json#executionOrder`. Do not skip tasks or reorder them — each builds on the previous.

For each task:
- Create or modify only the files listed in `task.files`
- Implement exactly what `task.description` says
- Verify all items in `task.acceptance` are satisfied before moving to the next task
- Commit after each task with message: `feat(phase2): [task.id] [task.title]`

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
- All Drizzle schema files live under `src/db/schema/{domain}/`
- Run `bunx drizzle-kit generate` after adding schema files, then `wrangler d1 migrations apply --local`

### 5. Type safety requirements

- TypeScript strict mode must pass: `tsc --noEmit` with zero errors
- Use Zod for all external input validation (HTTP request bodies, WS message payloads, LLM JSON outputs)
- The shared types in `packages/shared/src/rules.ts` and `packages/shared/src/review.ts` are the **single source of truth** — import them in both agent and frontend code

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

Before considering the implementation done, verify `apps/worker/wrangler.jsonc` contains:
- All 6 DO bindings with correct class names matching exports
- Both queue consumer and producer bindings
- Vectorize binding
- AI binding
- D1 binding
- Migration tags v1 and v2 with the correct `new_sqlite_classes` arrays

---

## What Not to Do

- **Do not modify** any file in `apps/hook/`, `apps/opencode-plugin/`, `packages/ui/`, `packages/editor/`, `packages/review-editor/`, or `packages/server/`
- **Do not add** new root-level config files unless a task explicitly requires it
- **Do not use** `fetch()` for internal agent-to-agent communication — always use DO RPC or sub-agent stubs
- **Do not write** D1 directly from Hono mutation handlers — all mutations route through agent `@callable` methods
- **Do not** put `@callable` on methods only called internally between agents
- **Do not** create new test files unless a task explicitly requires it — the existing test suite must still pass

---

## Verification Checklist

Run these checks after completing all tasks:

```bash
# 1. Existing builds still pass
bun run build:hook
bun run build:opencode

# 2. New worker builds
cd apps/worker && bunx wrangler dev --dry-run

# 3. TypeScript clean
cd apps/worker && tsc --noEmit

# 4. D1 migration applies cleanly
cd apps/worker && bunx wrangler d1 migrations apply plannotator-db --local

# 5. Vectorize index exists
bunx wrangler vectorize list

# 6. Root workspace resolves
cd /repo/root && bun install
```

All six checks must pass with zero errors before the implementation is considered complete.

---

## Questions / Ambiguities

If you encounter an ambiguity not covered by this prompt or `IMPLEMENTATION_PLAN.md`, resolve it using these tiebreakers in order:

1. Prefer the Cloudflare Agents SDK official pattern over any generic Workers pattern
2. Prefer Durable Object RPC over any form of HTTP for internal communication
3. Prefer strong types (TypeScript + Zod) over `any` / unknown
4. Prefer immutable audit trails (append-only assignment log) over destructive updates
5. When in doubt about a DB write path, route it through the relevant Agent's `@callable` method

---

## Deliverables Expected

When complete, open a pull request with:

- All new files under `apps/worker/`
- All new schema files under `src/db/schema/`
- New shared types under `packages/shared/src/`
- New Astro page and React island under `apps/marketing/`
- Updated `CLAUDE.md` and `AGENTS.md`
- Updated root `package.json` with `dev:worker` and `build:worker` scripts

The PR title should be: `feat: Phase 2 — Self-Hosted Agentic Plannotator`

---

*Last updated: 2026-05-17*  
*Reference: IMPLEMENTATION_PLAN.md · TASKS.json*
