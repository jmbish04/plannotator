# Plannotator — Phases 2 & 3: Self-Hosted Agentic Architecture

> **Status:** Architecture approved, pending implementation.  
> **Branch:** `copilot/research-unified-infra-architecture`  
> **Companion files:** [`TASKS.json`](./TASKS.json) · [`PROMPT.md`](./PROMPT.md)

---

## Table of Contents

### Phase 2 — Cloud-Native Backend

1. [Overview & Goals](#1-overview--goals)
2. [Technology Stack](#2-technology-stack)
3. [Pillar 1 — Self-Correcting Standards Engine & Learning Loop](#3-pillar-1--self-correcting-standards-engine--learning-loop)
4. [Pillar 2 — Hub-and-Spoke Multi-Agent Mesh](#4-pillar-2--hub-and-spoke-multi-agent-mesh)
5. [Pillar 3 — Real-Time Backlog Service & WebSocket Pub/Sub](#5-pillar-3--real-time-backlog-service--websocket-pubsub)
6. [Database & Schema Design (Phase 2)](#6-database--schema-design)
7. [wrangler.jsonc Configuration (Phase 2 base)](#7-wranglerjsonc-configuration)
8. [New Application Entry Point](#8-new-application-entry-point)
9. [Consistency & Reliability Guarantees](#9-consistency--reliability-guarantees)
10. [File-System Layout (Phase 2)](#10-file-system-layout-for-new-code)

### Phase 3 — Pattern Ingestion, Schema Discipline, AI Abstractions & Lifecycle Artifacts

11. [Phase 3 Overview](#11-phase-3-overview)
12. [Pillar 4 — Private Pattern Ingestion & Template Injection](#12-pillar-4--private-pattern-ingestion--template-injection)
13. [Pillar 5 — Deep Schema Modularization & Auto-Generated CRUD](#13-pillar-5--deep-schema-modularization--auto-generated-crud)
14. [Pillar 6 — Unified AI Abstractions & Agent Runtime Boundaries](#14-pillar-6--unified-ai-abstractions--agent-runtime-boundaries)
15. [Pillar 7 — Lifecycle Instruction & Stored-Plan Continuity](#15-pillar-7--lifecycle-instruction--stored-plan-continuity)
16. [wrangler.jsonc Additions (Phase 3)](#16-wranglerjsonc-additions-phase-3)
17. [File-System Layout (Phase 3 additions)](#17-file-system-layout-phase-3-additions)

---

## 1. Overview & Goals

Phase 2 evolves Plannotator from a local Bun-based review server into a fully self-hosted, agentic review platform deployed on Cloudflare Workers. The existing annotation UI (`packages/ui`), plan parser, and hook plugin remain unchanged. Phase 2 wraps them with a cloud-native backend. Phase 3 (see §11–17) layers on pattern ingestion, schema discipline, unified AI abstractions, and lifecycle artifact generation.

**Phase 2 — Three core deliverables:**

| # | Pillar | Primary Cloudflare Primitives |
|---|--------|-------------------------------|
| 1 | Self-Correcting Standards Engine | Agents SDK, Vectorize, D1, Workers AI, Queues |
| 2 | Hub-and-Spoke Multi-Agent Mesh | `this.subAgent()`, `this.addMcpServer()`, Durable Objects |
| 3 | Real-Time Backlog Pub/Sub | WebSocket Hibernation API, `this.broadcast()`, `waitUntil`, D1 |

**Phase 3 — Four additional pillars:**

| # | Pillar | Scope |
|---|--------|-------|
| 4 | Private Pattern Ingestion & Template Injection | `core-github-standardization` repo scan, baton-passing annotations, specialist tailoring |
| 5 | Deep Schema Modularization & Auto-CRUD | Per-table isolated files, `drizzle-zod` admin surface, journey-driven endpoints |
| 6 | Unified AI Abstractions & Agent Base Classes | `ModelProvider` control plane, `BackgroundOperationAgent`, `FrontendInteractiveAgent` |
| 7 | Lifecycle Instruction & Stored-Plan Continuity | Auto-generated `AGENTS.md`, `.agent/rules/`, D1-backed anti-vacuum continuity |

---

## 2. Technology Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Cloudflare Workers (ES modules) |
| Agent framework | `agents` SDK (`Agent`, `McpAgent`, `AIChatAgent`) |
| Database — durable | Cloudflare D1 (SQLite-compatible) |
| Database — hot cache | Agent-internal SQLite (`this.sql` template tag) |
| Vector search | Cloudflare Vectorize (`@cf/baai/bge-base-en-v1.5`) |
| LLM inference | Workers AI (`env.AI.run()`) via AI Gateway |
| Message queue | Cloudflare Queues |
| API router | Hono |
| Schema / migrations | Drizzle ORM + `drizzle-kit` |
| Frontend | Existing Astro 5 marketing site + React Islands |
| Type safety | TypeScript strict mode, Zod for runtime validation |

---

## 3. Pillar 1 — Self-Correcting Standards Engine & Learning Loop

### 3.1 Signal Extraction Pipeline

Every annotation submitted through the existing UI (approve/deny) that contains user-written text is treated as a potential preference signal. The extraction flow is:

```
User submits annotation via POST /api/approve or /api/deny
         │
         ├─ Standard approval flow (Phase 1 — unchanged)
         │
         └─ For each annotation with non-empty text content:
                Enqueue { annotationId, rawText, projectId }
                to "standards-learning-queue" (Cloudflare Queue)
                         │
                         ▼
                Queue consumer → StandardsAgent.processLearningSignal()
                         │
                         ├─ 1. Embed rawText:
                         │       vector = await env.AI.run(
                         │         '@cf/baai/bge-base-en-v1.5',
                         │         { text: rawText }
                         │       )
                         │
                         ├─ 2. Deduplicate via Vectorize topK=5, threshold 0.88
                         │       If ≥3 semantically similar signals exist → candidate cluster
                         │
                         ├─ 3. INSERT into learned_preferences (D1)
                         │       { rawText, extractedIntent: null, vectorId, processed: false }
                         │
                         ├─ 4. If cluster threshold met → LLM extraction pass:
                         │       Prompt: "Given these recurring signals, extract one
                         │                actionable engineering rule.
                         │                Output JSON: { name, description, category }"
                         │
                         ├─ 5. INSERT into rules (D1):
                         │       { source:'learned', approvalStatus:'pending_confirmation' }
                         │
                         └─ 6. UPDATE learned_preferences SET processed=true, promotedRuleId=...
```

### 3.2 Human-in-the-Loop Approval

Learned rules start as `pending_confirmation` and **never** enter agent system prompts or PR checks until explicitly promoted by the user.

**Rule lifecycle FSM:**

```
[extracted signal]
        │
        ▼
pending_confirmation ──→ approved_global        (hardcoded baseline, always enforced)
        │            ──→ approved_project_only  (scoped one-off, single project)
        └──────────→ rejected                   (discarded, blocks future dedup)
```

**Typed frontend contract** (`packages/shared/src/rules.ts`):

```typescript
// GET /api/rules?status=pending_confirmation
interface PendingRule {
  id: string;
  name: string;
  description: string;
  category:
    | 'package_manager'
    | 'data_layer'
    | 'code_organization'
    | 'architecture'
    | 'style'
    | 'custom';
  source: 'learned' | 'user_defined';
  approvalStatus: 'pending_confirmation';
  confidence: number;           // 0.0 – 0.99
  supportingEvidence: SupportingEvidence[];
  createdAt: number;
}

interface SupportingEvidence {
  annotationType: string;
  originalText: string;
  commentText: string | null;
  projectName: string;
  createdAt: number;
}

// PATCH /api/rules/:id
type RuleApprovalAction =
  | { action: 'approve_global' }
  | { action: 'approve_project'; projectId: string }
  | { action: 'reject' }
  | { action: 'edit'; name?: string; description?: string; category?: string };

// GET /api/rules (full list, all statuses)
interface RulesListResponse {
  hardcoded: HardcodedRule[];
  approved: ApprovedRule[];
  pending: PendingRule[];
}
```

### 3.3 Frontend Rules Config Panel

The panel lives at `/rules` in the Astro marketing site. The static shell is server-rendered; a single React island provides all interactivity:

```astro
---
// apps/marketing/src/pages/rules.astro
import RulesConfigPanel from '../components/RulesConfigPanel';
const initial = await fetch('/api/rules').then(r => r.json());
---
<Layout>
  <RulesConfigPanel client:load initialData={initial} />
</Layout>
```

**Panel sections:**

1. **Global Hardcoded Standards** — read-only lock-icon badges, no toggle
2. **Pending Confirmation** — card per rule with confidence meter, supporting-evidence accordion, and `[Approve Global] [Approve for Project] [Reject]` action buttons; badge count updated in real-time via `useAgent()` connected to `StandardsAgent`
3. **Active Learned & Custom Rules** — toggle switch per rule, inline description edit, category filter tabs, confidence bar (learned rules only), "Convert to global" button for project-scoped rules

### 3.4 StandardsAgent Responsibilities

- Instantiated per user (keyed by `userId`)
- Owns the learning queue consumer
- Holds the prompt-injection logic: on every `OrchestratorAgent` session start, fetches all `approved_global` rules from D1 and serializes them into the shared system-prompt snippet
- Exposes `@callable getPendingRules()`, `@callable promoteRule(id, action)` for frontend RPC calls

---

## 4. Pillar 2 — Hub-and-Spoke Multi-Agent Mesh

### 4.1 Topology

```
┌──────────────────────────────────────────────────────────────┐
│              OrchestratorAgent  (DO per review session)       │
│                                                               │
│  onStart():                                                   │
│    • loads approved_global rules from D1 into system prompt   │
│    • await this.addMcpServer('cf-docs', env.CF_DOCS_MCP)      │
│                                                               │
│  conductReview(plan):                                         │
│    ux   = await this.subAgent(UXSpecialistAgent,  'ux-'+id)   │
│    plat = await this.subAgent(PlatSpecialistAgent,'pl-'+id)   │
│    [uxResult, platResult] = await Promise.all([               │
│      ux.analyze(plan, streamCollector),                       │
│      plat.analyze(plan, streamCollector),                     │
│    ])                                                         │
│    → LLM synthesis pass → OrchestratorVerdict                 │
└─────────────────┬──────────────────────┬─────────────────────┘
                  │                      │
     ┌────────────▼──────────┐   ┌───────▼──────────────────────┐
     │  UXSpecialistAgent    │   │  PlatformSpecialistAgent      │
     │  (sub, isolated DB)   │   │  (sub, isolated DB)           │
     │                       │   │                               │
     │  Checks:              │   │  Checks:                      │
     │  • Astro SSR patterns │   │  • Native binding usage       │
     │  • client:load usage  │   │  • wrangler.jsonc migrations  │
     │  • Shadcn UI design   │   │  • AI Gateway fallback config │
     │                       │   │  • External REST API misuse   │
     │  Tools:               │   │  • D1/R2/KV/Queue correctness │
     │  cf-docs MCP lookup   │   │                               │
     └───────────────────────┘   │  Tools:                       │
                                 │  cf-docs MCP lookup           │
                                 └───────────────────────────────┘
```

### 4.2 Sub-Agent Instantiation

```typescript
// OrchestratorAgent.conductReview()
const ux   = await this.subAgent(UXSpecialistAgent,       `ux-${this.name}`);
const plat = await this.subAgent(PlatformSpecialistAgent, `pl-${this.name}`);
```

- `this.name` is the session ID, ensuring stable child identity within a session but fresh isolation across sessions
- Subsequent calls with the same name return the existing instance (deduplication built into the SDK)
- Each sub-agent's `onStart()` connects to the Cloudflare Docs `McpAgent` via DO binding (no HTTP round-trip)

### 4.3 Zero-Network MCP Tool Access

```typescript
// In UXSpecialistAgent.onStart() and PlatformSpecialistAgent.onStart()
await this.addMcpServer('cf-docs', this.env.CF_DOCS_MCP);
// CF_DOCS_MCP is a DO namespace binding to co-deployed CloudflareDocsMcp McpAgent
```

During `analyze()`, each specialist calls `search_cloudflare_documentation(query)` via `this.mcp` and injects the result into its LLM context window before scoring.

### 4.4 Callback Streaming

Sub-agent `analyze()` accepts an `RpcTarget` callback so the Orchestrator can stream progress to the UI without awaiting full completion:

```typescript
import { RpcTarget } from 'cloudflare:workers';

class StreamCollector extends RpcTarget {
  chunks: string[] = [];
  onChunk(text: string) { this.chunks.push(text); }
}

// In OrchestratorAgent:
const collector = new StreamCollector();
await ux.analyze(plan, collector);
// collector.chunks contains streamed output
```

### 4.5 Typed Review Contracts

```typescript
// packages/shared/src/review.ts

interface UXReviewResult {
  score: number;                   // 0–100
  astroViolations: string[];
  shadcnViolations: string[];
  suggestions: string[];
  rawAnalysis: string;
}

interface PlatformReviewResult {
  score: number;
  bindingViolations: string[];
  wranglerIssues: string[];
  aiGatewayConfigured: boolean;
  suggestions: string[];
  rawAnalysis: string;
}

interface OrchestratorVerdict {
  sessionId: string;
  planId: string;
  uxResult: UXReviewResult;
  platformResult: PlatformReviewResult;
  standardsViolations: string[];
  overallVerdict: 'pass' | 'fail' | 'needs_revision';
  synthesisNarrative: string;
}
```

### 4.6 Parent ↔ Child Identity

Sub-agents can call back to the Orchestrator using `this.parentAgent(OrchestratorAgent)`, which returns a typed RPC stub without any network overhead. This allows specialists to push streaming progress updates to the parent, which re-broadcasts them over WebSocket to connected UI clients.

---

## 5. Pillar 3 — Real-Time Backlog Service & WebSocket Pub/Sub

### 5.1 Backlog Decomposition Pipeline

Triggered asynchronously after plan approval via `waitUntil`:

```
POST /api/approve
         │
         ├─ Standard approval flow (unchanged)
         │
         └─ ctx.waitUntil(decomposeIntoBacklog(planVersionId, projectId))
                   │
                   ▼
           decomposeBacklog():
             1. Read plan markdown from plan_versions D1 table
             2. LLM structured-output call:
                  "Parse into Epics and Tasks.
                   H1/H2 sections → Epics.
                   Bullets/sub-tasks → Tasks with acceptance criteria.
                   Output strict JSON: { epics: Epic[], tasks: Task[] }"
             3. Zod schema validation
             4. Batch INSERT epics → D1 backlog.epics
             5. Batch INSERT tasks → D1 backlog.tasks
             6. Call BacklogAgent.initializeFromPlan(epicIds, taskIds) via DO RPC
                  → Loads tasks into hot SQLite mirror
                  → Broadcasts { type:'backlog_initialized', epicCount, taskCount }
```

### 5.2 External Coding Agent REST API (Hono)

Mounted at `/api/tasks/*` on the Worker. Authentication via per-project API token (Worker secret).

| Endpoint | Method | Auth | Description |
|---|---|---|---|
| `/api/tasks/{projectId}` | GET | Bearer | List tasks; filter by `?status=backlog&epicId=xxx` |
| `/api/tasks/{projectId}/{taskId}/claim` | POST | Bearer | Claim a task; body: `{ agentId }` |
| `/api/tasks/{projectId}/{taskId}/status` | PATCH | Bearer | Update status; body: `{ newStatus, note, agentId }` |
| `/api/tasks/{projectId}/{taskId}/release` | POST | Bearer | Release ownership; body: `{ agentId, reason? }` |
| `/api/backlog/{projectId}/ws` | GET | Query token | WebSocket upgrade → `BacklogAgent` connection |

All REST mutations proxy through `BacklogAgent`'s `@callable updateTaskStatus()` via DO RPC — never writing D1 directly — so every mutation follows the hot-SQLite-first → broadcast → async D1-mirror path.

### 5.3 WebSocket Connection Lifecycle

#### Step 1 — Upgrade & Attach

```
Client → HTTP GET /api/agents/backlog/{projectId}?type=ui&agentId=xxx
       → Upgrade: websocket

BacklogAgent.onConnect(connection, ctx):
  1. Parse ?type and ?agentId from URL
  2. connection.setState({ type, agentId, connectedAt: Date.now() })
     └─ Persists across hibernation via serializeAttachment
  3. Send initial snapshot:
       connection.send(JSON.stringify({
         type: 'snapshot',
         tasks: this.sql`SELECT * FROM hot_tasks WHERE project_id = ${projectId}`,
         presence: this.getPresence()
       }))
  4. Broadcast presence-joined to others:
       this.broadcast(
         JSON.stringify({ type: 'presence_joined', agentId }),
         [connection.id]   // exclude sender
       )
```

#### Step 2 — Task Mutation (Hot Path)

```
Client/Agent sends WS message:
  { type: 'task_update', taskId, newStatus, note, agentId }

BacklogAgent.onMessage(connection, message):
  1. Parse + validate via Zod
  2. Ownership check: if newStatus requires prior claim,
     verify connection.state.agentId === current task owner
  3. Write to hot SQLite (synchronous, < 1 ms):
       this.sql`UPDATE hot_tasks SET status=${newStatus},
                updated_at=${Date.now()} WHERE id=${taskId}`
  4. Append event log:
       this.sql`INSERT INTO hot_task_events VALUES (...)`
  5. Broadcast to ALL connections:
       this.broadcast(JSON.stringify({
         type: 'task_updated', taskId, newStatus, agentId, timestamp: Date.now()
       }))
  6. Async D1 mirror (non-blocking):
       import { waitUntil } from 'cloudflare:workers'
       waitUntil(this.mirrorToD1(taskId, newStatus, note, agentId))
```

#### Step 3 — Async D1 Mirror

```typescript
async mirrorToD1(taskId: string, newStatus: string, note: string, agentId: string) {
  await this.env.DB
    .prepare('UPDATE tasks SET status=?, updated_at=? WHERE id=?')
    .bind(newStatus, Date.now(), taskId).run();

  await this.env.DB
    .prepare(`INSERT INTO task_assignments
              (id,task_id,agent_id,action,to_status,note,created_at)
              VALUES (?,?,?,'status_update',?,?,?)`)
    .bind(crypto.randomUUID(), taskId, agentId, newStatus, note, Date.now())
    .run();
}
```

**Latency profile:** UI sees the state change in `< 1 ms` (SQLite write + broadcast). D1 write completes asynchronously `~10–50 ms` later. No client ever blocks on D1 I/O.

#### Step 4 — Hibernation & Wake-Up

```
DO goes idle (~30 s no messages)
        │
        ▼
Cloudflare runtime hibernates DO
  • All WebSocket connections stay OPEN on client side
  • Compute billing pauses
  • Connection metadata preserved via serializeAttachment()
        │
New WS message arrives
        │
        ▼
Runtime wakes DO via webSocketMessage handler
  • DO state reconstructed from embedded SQLite
  • connection.state restored via ws.deserializeAttachment()
  • this.broadcast() resumes via this.ctx.getWebSockets()
    (hibernation registry — not in-memory list)
```

#### Step 5 — Scheduled Reconciliation

An hourly alarm (`this.schedule('0 * * * *', 'reconcileWithD1')`) performs a full reconciliation between hot SQLite and D1 to repair any missed `waitUntil` calls from transient failures.

---

## 6. Database & Schema Design

### 6.1 Schema File Organization

```
src/db/schema/
├── index.ts                          ← re-exports all tables
├── projects/
│   └── projects.ts                   ← Phase 1, unchanged
├── plans/
│   ├── plans.ts                      ← Phase 1, unchanged
│   └── plan_versions.ts              ← Phase 1, unchanged
├── annotations/
│   └── annotations.ts                ← Phase 1, unchanged
├── rules/
│   ├── rules.ts                      ← EXTENDED with Phase 2 columns
│   ├── rule_observations.ts          ← Phase 1, unchanged
│   └── learned_preferences.ts        ← NEW
├── backlog/
│   ├── epics.ts                      ← NEW
│   ├── sprints.ts                    ← NEW
│   ├── tasks.ts                      ← NEW
│   └── task_assignments.ts           ← NEW
├── review/
│   ├── review_sessions.ts            ← NEW
│   └── specialist_reports.ts         ← NEW
└── pr_checks/
    └── pr_checks.ts                  ← Phase 1, unchanged
```

### 6.2 New / Modified Table Schemas (Drizzle)

#### `rules/learned_preferences.ts`

```typescript
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';
import { projects } from '../projects/projects';
import { rules } from './rules';

export const learnedPreferences = sqliteTable('learned_preferences', {
  id:              text('id').primaryKey(),
  projectId:       text('project_id').references(() => projects.id),
  sourceType:      text('source_type', {
                     enum: ['annotation','plan_comment','global_comment','user_note']
                   }).notNull(),
  sourceId:        text('source_id').notNull(),
  rawText:         text('raw_text').notNull(),
  extractedIntent: text('extracted_intent'),
  clusterKey:      text('cluster_key'),
  vectorId:        text('vector_id'),
  processed:       integer('processed', { mode: 'boolean' }).notNull().default(false),
  promotedRuleId:  text('promoted_rule_id').references(() => rules.id),
  createdAt:       integer('created_at', { mode: 'timestamp' }).notNull(),
});
```

#### `rules/rules.ts` — Phase 2 additions (new columns via Drizzle migration)

```typescript
// Additional columns added in Phase 2 migration
approvalStatus: text('approval_status', {
  enum: [
    'pending_confirmation',
    'approved_global',
    'approved_project_only',
    'rejected',
  ],
}).notNull().default('pending_confirmation'),
approvedBy:     text('approved_by'),
approvedAt:     integer('approved_at', { mode: 'timestamp' }),
scope:          text('scope', { enum: ['global', 'project'] })
                  .notNull().default('global'),
```

#### `backlog/epics.ts`

```typescript
import { sqliteTable, text, integer, index } from 'drizzle-orm/sqlite-core';
import { plans } from '../plans/plans';
import { projects } from '../projects/projects';

export const epics = sqliteTable('epics', {
  id:          text('id').primaryKey(),
  planId:      text('plan_id').notNull().references(() => plans.id),
  projectId:   text('project_id').notNull().references(() => projects.id),
  title:       text('title').notNull(),
  description: text('description'),
  status:      text('status', {
                 enum: ['draft','active','completed','cancelled']
               }).notNull().default('draft'),
  order:       integer('order').notNull(),
  createdAt:   integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt:   integer('updated_at', { mode: 'timestamp' }).notNull(),
});
```

#### `backlog/sprints.ts`

```typescript
export const sprints = sqliteTable('sprints', {
  id:        text('id').primaryKey(),
  projectId: text('project_id').notNull().references(() => projects.id),
  name:      text('name').notNull(),
  goal:      text('goal'),
  status:    text('status', {
               enum: ['planned','active','completed','cancelled']
             }).notNull().default('planned'),
  startDate: integer('start_date', { mode: 'timestamp' }),
  endDate:   integer('end_date', { mode: 'timestamp' }),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).notNull(),
});
```

#### `backlog/tasks.ts`

```typescript
export const tasks = sqliteTable('tasks', {
  id:                 text('id').primaryKey(),
  epicId:             text('epic_id').notNull().references(() => epics.id),
  sprintId:           text('sprint_id').references(() => sprints.id),
  projectId:          text('project_id').notNull().references(() => projects.id),
  title:              text('title').notNull(),
  description:        text('description'),
  acceptanceCriteria: text('acceptance_criteria'),
  status:             text('status', {
                        enum: ['backlog','claimed','in_progress','review','done','blocked']
                      }).notNull().default('backlog'),
  priority:           text('priority', {
                        enum: ['critical','high','medium','low']
                      }).notNull().default('medium'),
  storyPoints:        integer('story_points'),
  claimedBy:          text('claimed_by'),
  claimedAt:          integer('claimed_at', { mode: 'timestamp' }),
  completedAt:        integer('completed_at', { mode: 'timestamp' }),
  createdAt:          integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt:          integer('updated_at', { mode: 'timestamp' }).notNull(),
}, (t) => ({
  epicStatusIdx:   index('tasks_epic_status_idx').on(t.epicId, t.status),
  sprintStatusIdx: index('tasks_sprint_status_idx').on(t.sprintId, t.status),
}));
```

#### `backlog/task_assignments.ts`

```typescript
export const taskAssignments = sqliteTable('task_assignments', {
  id:         text('id').primaryKey(),
  taskId:     text('task_id').notNull().references(() => tasks.id),
  agentId:    text('agent_id').notNull(),
  action:     text('action', {
                enum: ['claimed','released','reassigned','status_update']
              }).notNull(),
  fromStatus: text('from_status'),
  toStatus:   text('to_status'),
  note:       text('note'),
  createdAt:  integer('created_at', { mode: 'timestamp' }).notNull(),
});
```

#### `review/review_sessions.ts`

```typescript
export const reviewSessions = sqliteTable('review_sessions', {
  id:                       text('id').primaryKey(),
  planVersionId:            text('plan_version_id').notNull(),
  projectId:                text('project_id').notNull().references(() => projects.id),
  orchestratorInstanceName: text('orchestrator_instance_name').notNull(),
  status:                   text('status', {
                              enum: ['pending','running','completed','error']
                            }).notNull().default('pending'),
  verdict:                  text('verdict', {
                              enum: ['pass','fail','needs_revision']
                            }),
  synthesisNarrative:       text('synthesis_narrative'),
  createdAt:                integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt:                integer('updated_at', { mode: 'timestamp' }).notNull(),
});
```

#### `review/specialist_reports.ts`

```typescript
export const specialistReports = sqliteTable('specialist_reports', {
  id:             text('id').primaryKey(),
  sessionId:      text('session_id').notNull().references(() => reviewSessions.id),
  specialistType: text('specialist_type', { enum: ['ux','platform'] }).notNull(),
  score:          real('score').notNull(),
  violations:     text('violations').notNull(),   // JSON array
  suggestions:    text('suggestions').notNull(),  // JSON array
  rawAnalysis:    text('raw_analysis'),
  createdAt:      integer('created_at', { mode: 'timestamp' }).notNull(),
});
```

---

## 7. wrangler.jsonc Configuration

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "plannotator",
  "main": "src/worker/index.ts",
  "compatibility_date": "2026-05-17",
  "compatibility_flags": ["nodejs_compat"],

  "assets": {
    "directory": "./dist/",
    "not_found_handling": "single-page-application",
    "binding": "ASSETS",
    "run_worker_first": ["/api/*", "/agents/*"]
  },

  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "plannotator-db",
      "database_id": "<REPLACE_WITH_D1_ID>"
    }
  ],

  "vectorize": [
    {
      "binding": "VECTORIZE",
      "index_name": "annotation-embeddings"
    }
  ],

  "ai": { "binding": "AI" },

  "queues": {
    "producers": [
      { "queue": "pr-review-queue",          "binding": "PR_QUEUE" },
      { "queue": "standards-learning-queue", "binding": "STANDARDS_QUEUE" }
    ],
    "consumers": [
      { "queue": "pr-review-queue",          "max_batch_size": 1 },
      {
        "queue": "standards-learning-queue",
        "max_batch_size": 10,
        "max_batch_timeout": 5
      }
    ]
  },

  "durable_objects": {
    "bindings": [
      { "name": "ORCHESTRATOR_AGENT",        "class_name": "OrchestratorAgent" },
      { "name": "UX_SPECIALIST_AGENT",       "class_name": "UXSpecialistAgent" },
      { "name": "PLATFORM_SPECIALIST_AGENT", "class_name": "PlatformSpecialistAgent" },
      { "name": "STANDARDS_AGENT",           "class_name": "StandardsAgent" },
      { "name": "BACKLOG_AGENT",             "class_name": "BacklogAgent" },
      { "name": "CF_DOCS_MCP",              "class_name": "CloudflareDocsMcp" }
    ]
  },

  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["StandardsAgent"]
    },
    {
      "tag": "v2",
      "new_sqlite_classes": [
        "OrchestratorAgent",
        "UXSpecialistAgent",
        "PlatformSpecialistAgent",
        "BacklogAgent",
        "CloudflareDocsMcp"
      ]
    }
  ]
}
```

---

## 8. New Application Entry Point

A new app `apps/worker/` is the Cloudflare Worker entry point. It is **separate** from the existing apps and does not modify any existing package.

```
apps/worker/
├── src/
│   ├── index.ts                    ← Worker fetch + queue handler exports
│   ├── router.ts                   ← Hono app — mounts all /api/* routes
│   ├── agents/
│   │   ├── OrchestratorAgent.ts
│   │   ├── UXSpecialistAgent.ts
│   │   ├── PlatformSpecialistAgent.ts
│   │   ├── StandardsAgent.ts
│   │   ├── BacklogAgent.ts
│   │   └── CloudflareDocsMcp.ts
│   ├── routes/
│   │   ├── tasks.ts                ← /api/tasks/* (Hono)
│   │   ├── rules.ts                ← /api/rules/* (Hono)
│   │   ├── backlog.ts              ← /api/backlog/* (Hono)
│   │   └── review.ts               ← /api/review/* (Hono)
│   └── lib/
│       ├── backlogDecomposer.ts    ← LLM decomposition pipeline
│       ├── embeddings.ts           ← Vectorize helpers
│       └── auth.ts                 ← Bearer token validation
├── wrangler.jsonc
└── package.json
```

---

## 9. Consistency & Reliability Guarantees

| Layer | Write Path | Consistency Level |
|-------|------------|-------------------|
| BacklogAgent hot SQLite | Synchronous (same DO activation) | Immediate, strongly consistent within DO |
| WebSocket broadcast | `this.broadcast()` after SQLite write | All active connections updated in same event-loop tick |
| D1 durable write | `waitUntil(mirrorToD1(...))` | Eventually consistent, typically < 100 ms after broadcast |
| Hibernation recovery | `ws.deserializeAttachment()` on wake | Full per-connection state restored; hot SQLite reloads lazily |
| Cross-sub-agent calls | `this.subAgent()` typed RPC | Zero-network, synchronous within parent event loop |
| D1 ↔ DO reconciliation | Scheduled hourly alarm | Full consistency repair of any missed `waitUntil` failures |

---

## 10. File-System Layout for New Code

```
plannotator/
├── apps/
│   └── worker/                          ← NEW: Cloudflare Worker entry
│       ├── src/
│       │   ├── index.ts
│       │   ├── router.ts
│       │   ├── agents/
│       │   │   ├── OrchestratorAgent.ts
│       │   │   ├── UXSpecialistAgent.ts
│       │   │   ├── PlatformSpecialistAgent.ts
│       │   │   ├── StandardsAgent.ts
│       │   │   ├── BacklogAgent.ts
│       │   │   └── CloudflareDocsMcp.ts
│       │   ├── routes/
│       │   │   ├── tasks.ts
│       │   │   ├── rules.ts
│       │   │   ├── backlog.ts
│       │   │   └── review.ts
│       │   └── lib/
│       │       ├── backlogDecomposer.ts
│       │       ├── embeddings.ts
│       │       └── auth.ts
│       └── wrangler.jsonc
│
├── packages/
│   └── shared/                          ← EXTENDED: new shared types
│       └── src/
│           ├── rules.ts                 ← NEW: PendingRule, RuleApprovalAction, etc.
│           └── review.ts                ← NEW: UXReviewResult, OrchestratorVerdict, etc.
│
├── src/
│   └── db/
│       └── schema/
│           ├── rules/
│           │   └── learned_preferences.ts   ← NEW
│           ├── backlog/
│           │   ├── epics.ts                 ← NEW
│           │   ├── sprints.ts               ← NEW
│           │   ├── tasks.ts                 ← NEW
│           │   └── task_assignments.ts      ← NEW
│           └── review/
│               ├── review_sessions.ts       ← NEW
│               └── specialist_reports.ts    ← NEW
│
└── apps/marketing/
    └── src/
        ├── pages/
        │   └── rules.astro                  ← NEW: Rules Config page
        └── components/
            └── RulesConfigPanel.tsx         ← NEW: React island
```

---

*End of Phase 2 content — Phase 3 sections follow below.*

---

## 11. Phase 3 Overview

Phase 3 does not introduce new Cloudflare infrastructure — it **disciplines** how Phase 2 infrastructure is used and adds four cross-cutting capabilities that are pre-requisites for long-running multi-agent sprints.

| Concern | Problem Solved |
|---------|----------------|
| Pattern Ingestion (§12) | Agents reinvent proven wheels every sprint; template injection + specialist tailoring eliminates redundant scratch-generation |
| Schema Modularization (§13) | Monolithic schema files → merge conflicts, opaque ownership; per-table isolation + drizzle-zod admin surface eliminates manual CRUD maintenance |
| AI Abstractions (§14) | Each agent wires inference differently → untestable, fragmented fallback logic; `ModelProvider` + base classes standardize all LLM calls |
| Lifecycle Artifacts (§15) | Agents run without persistent context across sprints → drift and regression; D1 ledger + auto-generated rule files ensure continuity |

**Guiding constraint:** Phase 3 code lives exclusively under:

- `apps/worker/src/backend/` — all new server-side modules
- `.agent/rules/` — generated IDE orchestration rules (committed output artifact)

No existing Phase 2 agent files are rewritten; they are extended via inheritance or composition.

---

## 12. Pillar 4 — Private Pattern Ingestion & Template Injection

### 12.1 Pattern Repository Contract

The `core-github-standardization` repository contains battle-tested engineering primitives organized by pattern type:

```
core-github-standardization/
├── patterns/
│   ├── full-stack-astro-worker/    ← Astro + Cloudflare Worker routing
│   ├── d1-drizzle-setup/           ← D1 + Drizzle ORM boilerplate
│   ├── realtime-agent-ws/          ← Agent WebSocket + hibernation
│   ├── ai-gateway-fallback/        ← Multi-provider AI Gateway config
│   └── hono-typed-router/          ← Hono + Zod end-to-end typed router
└── manifest.json                   ← Pattern index with tags + paths
```

The `manifest.json` schema (fetched at runtime by the agent):

```typescript
interface PatternManifest {
  version: string;
  patterns: PatternEntry[];
}

interface PatternEntry {
  id: string;                  // e.g. "d1-drizzle-setup"
  name: string;
  description: string;
  tags: string[];              // e.g. ["database", "drizzle", "d1", "migration"]
  entryPath: string;           // e.g. "patterns/d1-drizzle-setup/"
  files: string[];             // Relative file list in pattern directory
  parameterSchema: Record<string, string>; // { PROJECT_NAME, TABLE_PREFIX, ... }
  injectCommand: string;       // Parameterized shell command template
}
```

### 12.2 PatternIngestionMcp Agent

A new `McpAgent` class that wraps read access to the private `core-github-standardization` repository:

```typescript
// apps/worker/src/backend/agents/PatternIngestionMcp.ts
import { McpAgent } from 'agents/mcp';
import type { Env } from '../../index';

export class PatternIngestionMcp extends McpAgent<Env, {}> {
  async init() {
    // Register tools exposed to OrchestratorAgent
    this.server.tool('list_patterns', { description: 'List all available patterns from the standardization repo' },
      async () => {
        const manifest = await this.fetchManifest();
        return { content: [{ type: 'text', text: JSON.stringify(manifest.patterns) }] };
      }
    );

    this.server.tool('get_pattern_files', {
        description: 'Fetch the file contents of a specific pattern block',
        inputSchema: { type: 'object', properties: { patternId: { type: 'string' } }, required: ['patternId'] }
      },
      async ({ patternId }) => {
        const files = await this.fetchPatternFiles(patternId);
        return { content: [{ type: 'text', text: JSON.stringify(files) }] };
      }
    );

    this.server.tool('build_inject_command', {
        description: 'Generate the parameterized inject command for a pattern with project-specific values',
        inputSchema: {
          type: 'object',
          properties: {
            patternId:  { type: 'string' },
            parameters: { type: 'object' }
          },
          required: ['patternId', 'parameters']
        }
      },
      async ({ patternId, parameters }) => {
        const cmd = await this.buildCommand(patternId, parameters);
        return { content: [{ type: 'text', text: cmd }] };
      }
    );
  }

  private async fetchManifest(): Promise<PatternManifest> {
    const token = this.env.STANDARDIZATION_REPO_TOKEN;
    const res = await fetch(
      'https://raw.githubusercontent.com/OWNER/core-github-standardization/main/manifest.json',
      { headers: { Authorization: `token ${token}` } }
    );
    return res.json();
  }

  // fetchPatternFiles() and buildCommand() omitted for brevity
}
```

### 12.3 OrchestratorAgent — Template Annotation Pass

After its standard review pass (§4), the `OrchestratorAgent` runs an additional **template-mapping pass**:

```
conductReview(plan)
        │
        ├─ [existing] Parallel UX + Platform specialist analysis
        │
        └─ [NEW] Template Mapping Pass:
               ┌─────────────────────────────────────────────────────┐
               │  1. Call this.mcp['pattern-ingestion'].list_patterns │
               │  2. LLM prompt:                                      │
               │       "Given this plan and available patterns,       │
               │        identify sections that could be replaced by   │
               │        a standardized template injection rather      │
               │        than scratch-generation.                      │
               │        Output TemplateAnnotation[] JSON."            │
               │  3. Zod-validate TemplateAnnotation[]                │
               │  4. For each matched annotation:                     │
               │       • Call build_inject_command(patternId, params) │
               │       • Attach injectCommand to annotation           │
               │  5. Store template_annotations in D1                 │
               │  6. Broadcast { type:'template_annotations', ... }   │
               └─────────────────────────────────────────────────────┘
```

**`TemplateAnnotation` type** (`packages/shared/src/patterns.ts`):

```typescript
interface TemplateAnnotation {
  planSectionRef: string;         // Heading or block ID in plan
  patternId: string;              // Matched pattern from manifest
  patternName: string;
  matchConfidence: number;        // 0.0 – 1.0
  parameters: Record<string, string>; // Project-specific fill values
  injectCommand: string;          // Executable command to pull template
  rationale: string;              // Why this pattern maps to this section
}

type TemplateAnnotatedPlan = {
  originalPlan: string;
  annotations: TemplateAnnotation[];
  unreplacedSections: string[];   // Sections with no matching pattern → scratch-generate
}
```

### 12.4 Specialist Tailoring Pass

When a specialist agent receives a `TemplateAnnotation` alongside the plan section it covers, it **does not** validate scratch-built code. Instead it performs a surgical tailoring analysis:

```
UXSpecialistAgent.analyze(plan, streamCollector, templateAnnotations?)
        │
        ├─ [standard] Astro / Shadcn violation scan (unchanged)
        │
        └─ [NEW] For each templateAnnotation in section:
               ┌──────────────────────────────────────────────────────────┐
               │  1. Fetch pattern file contents via get_pattern_files     │
               │  2. LLM prompt:                                          │
               │       "Given this standard template and the project's    │
               │        domain spec, generate a TailoringGroup — a        │
               │        precise list of line-level edits to bind the      │
               │        shared pattern to this project's requirements."   │
               │  3. Output TailoringGroup[] (Zod-validated)             │
               │  4. Attach TailoringGroup to TemplateAnnotation          │
               └──────────────────────────────────────────────────────────┘
```

**`TailoringGroup` type** (`packages/shared/src/patterns.ts`):

```typescript
interface TailoringInstruction {
  filePath: string;              // Relative path within injected pattern
  lineNumber?: number;
  searchPattern: string;         // Exact string or regex to locate the line
  replacement: string;           // Project-specific replacement value
  reason: string;
}

interface TailoringGroup {
  patternId: string;
  projectId: string;
  instructions: TailoringInstruction[];
  appliedBy: 'ux' | 'platform';
  generatedAt: number;
}
```

### 12.5 New D1 Tables for Pattern Phase

```typescript
// src/backend/db/schemas/patterns/template_annotations.ts
export const templateAnnotations = sqliteTable('template_annotations', {
  id:               text('id').primaryKey(),
  reviewSessionId:  text('review_session_id').notNull().references(() => reviewSessions.id),
  planSectionRef:   text('plan_section_ref').notNull(),
  patternId:        text('pattern_id').notNull(),
  matchConfidence:  real('match_confidence').notNull(),
  parameters:       text('parameters').notNull(),  // JSON
  injectCommand:    text('inject_command').notNull(),
  rationale:        text('rationale'),
  tailoringGroup:   text('tailoring_group'),        // JSON TailoringGroup, set after specialist pass
  createdAt:        integer('created_at', { mode: 'timestamp' }).notNull(),
});
```

---

## 13. Pillar 5 — Deep Schema Modularization & Auto-Generated CRUD

### 13.1 Canonical Schema Directory Layout

All Drizzle table files (Phase 2 and Phase 3) are reorganized into strictly isolated per-table files:

```
apps/worker/src/backend/db/schemas/
├── index.ts                             ← re-exports all tables
├── configurations/
│   ├── projects.ts
│   └── rules.ts
├── plans/
│   ├── plans.ts
│   ├── plan_versions.ts
│   └── template_annotations.ts
├── annotations/
│   └── annotations.ts
├── backlog/
│   ├── epics.ts
│   ├── sprints.ts
│   ├── tasks.ts
│   └── task_assignments.ts
├── review/
│   ├── review_sessions.ts
│   └── specialist_reports.ts
├── signals/
│   └── learned_preferences.ts
└── lifecycle/
    ├── sprint_contexts.ts               ← NEW (Phase 3)
    └── agent_rule_snapshots.ts          ← NEW (Phase 3)
```

**Rules:**
- One Drizzle table = one `.ts` file. No exceptions.
- File name = snake_case table name.
- Imports reference sibling schemas by relative path only (no barrel re-imports inside schema files).
- `index.ts` is the only file allowed to re-export; no other file imports from `index.ts`.

### 13.2 drizzle-zod Auto-Admin CRUD Layer

Every table defined in `schemas/` automatically gets a full CRUD admin surface with zero manual route authoring:

```typescript
// apps/worker/src/backend/routes/admin.ts
import { Hono } from 'hono';
import { drizzle } from 'drizzle-orm/d1';
import { createInsertSchema, createSelectSchema } from 'drizzle-zod';
import { zValidator } from '@hono/zod-validator';
import * as schema from '../db/schemas/index';
import { requireAdminBearer } from '../lib/auth';
import type { Env } from '../../index';

const admin = new Hono<{ Bindings: Env }>();
admin.use('*', requireAdminBearer);

// Dynamically register CRUD for each exported table
for (const [tableName, table] of Object.entries(schema)) {
  if (typeof table !== 'object' || !('$inferSelect' in table)) continue;

  const insertSchema = createInsertSchema(table);
  const selectSchema = createSelectSchema(table);

  // GET /api/admin/tables/:tableName
  admin.get(`/tables/${tableName}`, async (c) => {
    const db = drizzle(c.env.DB, { schema });
    const rows = await db.select().from(table as any);
    return c.json(rows);
  });

  // GET /api/admin/tables/:tableName/:id
  admin.get(`/tables/${tableName}/:id`, async (c) => {
    const db = drizzle(c.env.DB, { schema });
    const rows = await db.select().from(table as any)
      .where((t: any) => eq(t.id, c.req.param('id')));
    return rows.length ? c.json(rows[0]) : c.json({ error: 'Not found' }, 404);
  });

  // POST /api/admin/tables/:tableName
  admin.post(`/tables/${tableName}`, zValidator('json', insertSchema), async (c) => {
    const db = drizzle(c.env.DB, { schema });
    const body = c.req.valid('json');
    const result = await db.insert(table as any).values(body).returning();
    return c.json(result[0], 201);
  });

  // PATCH /api/admin/tables/:tableName/:id
  admin.patch(`/tables/${tableName}/:id`, zValidator('json', insertSchema.partial()), async (c) => {
    const db = drizzle(c.env.DB, { schema });
    const body = c.req.valid('json');
    const result = await db.update(table as any).set(body)
      .where((t: any) => eq(t.id, c.req.param('id'))).returning();
    return result.length ? c.json(result[0]) : c.json({ error: 'Not found' }, 404);
  });

  // DELETE /api/admin/tables/:tableName/:id
  admin.delete(`/tables/${tableName}/:id`, async (c) => {
    const db = drizzle(c.env.DB, { schema });
    await db.delete(table as any).where((t: any) => eq(t.id, c.req.param('id')));
    return c.json({ deleted: true });
  });
}

export { admin };
```

**Schema-change resilience:** When a new table is added to `schemas/`, the loop automatically exposes its full CRUD surface on the next deploy — no route changes needed.

### 13.3 Journey-Driven API Boundaries

The admin layer handles **maintenance operations**. **Product journeys** live in dedicated route files with hand-authored, multi-table Drizzle queries and domain-specific Zod schemas:

| Journey | Route Prefix | What It Does |
|---------|-------------|--------------|
| Plan submission + backlog trigger | `POST /api/plans/submit` | Inserts plan, enqueues decomposition, returns backlog stub |
| Template injection execution log | `POST /api/patterns/inject` | Records injection command execution + outcome |
| Sprint context snapshot | `POST /api/sprints/snapshot` | Captures current sprint rules + completed task IDs into `sprint_contexts` |
| Agent rule compilation | `POST /api/lifecycle/compile-rules` | Triggers `LifecycleArtifactAgent.compileArtifacts()` via DO RPC |

Each journey route has its own Zod schema, does **not** reuse the drizzle-zod generated schemas, and may span multiple D1 tables in a single transaction.

### 13.4 New Phase 3 Schema Tables

#### `lifecycle/sprint_contexts.ts`

```typescript
export const sprintContexts = sqliteTable('sprint_contexts', {
  id:                  text('id').primaryKey(),
  projectId:           text('project_id').notNull().references(() => projects.id),
  sprintId:            text('sprint_id').references(() => sprints.id),
  planVersionId:       text('plan_version_id').notNull(),
  completedTaskIds:    text('completed_task_ids').notNull(),   // JSON string[]
  activeRuleSnapshot:  text('active_rule_snapshot').notNull(), // JSON ApprovedRule[]
  templateInjections:  text('template_injections').notNull(),  // JSON TemplateAnnotation[]
  agentsMdHash:        text('agents_md_hash'),                 // SHA-256 of emitted AGENTS.md
  createdAt:           integer('created_at', { mode: 'timestamp' }).notNull(),
});
```

#### `lifecycle/agent_rule_snapshots.ts`

```typescript
export const agentRuleSnapshots = sqliteTable('agent_rule_snapshots', {
  id:           text('id').primaryKey(),
  projectId:    text('project_id').notNull().references(() => projects.id),
  ruleContent:  text('rule_content').notNull(),  // Full .agent/rules/*.md content
  ruleFileName: text('rule_file_name').notNull(),
  trigger:      text('trigger', {
                  enum: ['sprint_complete','manual','plan_approved']
                }).notNull(),
  createdAt:    integer('created_at', { mode: 'timestamp' }).notNull(),
});
```

---

## 14. Pillar 6 — Unified AI Abstractions & Agent Runtime Boundaries

### 14.1 ModelProvider — Central Inference Control Plane

All LLM calls in the codebase route through a single class. No agent, route, or service calls `env.AI.run()` directly.

```typescript
// apps/worker/src/backend/ai/providers/model-provider.ts
import type { Env } from '../../index';

type Provider = 'workers-ai' | 'openai' | 'anthropic' | 'gemini';

interface GenerateTextOptions {
  prompt: string;
  systemPrompt?: string;
  provider?: Provider;
  model?: string;
  maxTokens?: number;
}

interface GenerateStructuredOptions<T> extends GenerateTextOptions {
  schema: import('zod').ZodType<T>;
  retries?: number;             // Default: 2
}

interface GenerateVisionOptions {
  prompt: string;
  imageUrl: string;
  provider?: Provider;
  model?: string;
}

export class ModelProvider {
  constructor(private env: Env) {}

  // Default: workers-ai @cf/meta/llama-3.1-8b-instruct → openai gpt-4o fallback
  async generateText(opts: GenerateTextOptions): Promise<string> {
    const provider = opts.provider ?? 'workers-ai';
    const gatewayUrl = `https://gateway.ai.cloudflare.com/v1/${this.env.CF_ACCOUNT_ID}/${this.env.AI_GATEWAY_ID}`;

    if (provider === 'workers-ai') {
      // Workers AI model names are typed as string in the SDK;
      // cast through unknown to satisfy strict model-name literal unions where needed.
      type WorkersAITextModel = Parameters<typeof this.env.AI.run>[0];
      const model = (opts.model ?? '@cf/meta/llama-3.1-8b-instruct') as WorkersAITextModel;
      const result = await this.env.AI.run(model, { messages: this.buildMessages(opts) });
      return (result as { response?: string }).response ?? '';
    }

    // External providers via AI Gateway proxy
    const res = await fetch(`${gatewayUrl}/${provider}/chat/completions`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${this.resolveToken(provider)}`,
      },
      body: JSON.stringify({
        model: opts.model ?? this.defaultModelFor(provider),
        messages: this.buildMessages(opts),
        max_tokens: opts.maxTokens ?? 2048,
      }),
    });
    const data = await res.json() as any;
    return data.choices?.[0]?.message?.content ?? '';
  }

  // Default: gemini gemini-2.0-flash — strict JSON schema adherence
  async generateStructured<T>(opts: GenerateStructuredOptions<T>): Promise<T> {
    const provider = opts.provider ?? 'gemini';
    const model    = opts.model    ?? 'gemini-2.0-flash';
    const zodToJsonSchema = (await import('zod-to-json-schema')).zodToJsonSchema;
    const jsonSchema = zodToJsonSchema(opts.schema);

    const systemPrompt = `${opts.systemPrompt ?? ''}\n\nYou MUST respond with valid JSON matching this schema:\n${JSON.stringify(jsonSchema, null, 2)}\n\nDo not include markdown fences or any other text.`;
    const raw = await this.generateText({ ...opts, provider, model, systemPrompt });

    const parsed = opts.schema.safeParse(JSON.parse(raw));
    if (parsed.success) return parsed.data;

    // Retry loop (up to opts.retries)
    const retries = opts.retries ?? 2;
    for (let i = 0; i < retries; i++) {
      const retry = await this.generateText({ ...opts, provider, model, systemPrompt: `${systemPrompt}\n\nPrevious attempt had validation errors. Try again.` });
      const p2 = opts.schema.safeParse(JSON.parse(retry));
      if (p2.success) return p2.data;
    }
    throw new Error(`generateStructured: failed after ${retries} retries`);
  }

  // Default: workers-ai @cf/llava-1.5-7b-hf
  async generateVision(opts: GenerateVisionOptions): Promise<string> {
    const provider = opts.provider ?? 'workers-ai';
    if (provider === 'workers-ai') {
      const result = await this.env.AI.run('@cf/llava-1.5-7b-hf' as any, {
        prompt: opts.prompt,
        image: opts.imageUrl,
      });
      return (result as any).description ?? '';
    }
    return this.generateText({ prompt: `[IMAGE: ${opts.imageUrl}]\n${opts.prompt}`, provider });
  }

  private buildMessages(opts: GenerateTextOptions) {
    const messages: { role: string; content: string }[] = [];
    if (opts.systemPrompt) messages.push({ role: 'system', content: opts.systemPrompt });
    messages.push({ role: 'user', content: opts.prompt });
    return messages;
  }

  private defaultModelFor(provider: Provider): string {
    return {
      'openai': 'gpt-4o',
      'anthropic': 'claude-3-5-sonnet-20241022',
      'gemini': 'gemini-2.0-flash',
      'workers-ai': '@cf/meta/llama-3.1-8b-instruct',
    }[provider];
  }

  private resolveToken(provider: Provider): string {
    return {
      'openai': this.env.OPENAI_API_KEY,
      'anthropic': this.env.ANTHROPIC_API_KEY,
      'gemini': this.env.GEMINI_API_KEY,
      'workers-ai': '',
    }[provider] ?? '';
  }
}
```

**Usage inside any agent or route:**

```typescript
const ai = new ModelProvider(this.env);
const text = await ai.generateText({ prompt: 'Hello' });
const data = await ai.generateStructured({ prompt: '...', schema: MyZodSchema });
```

### 14.2 BackgroundOperationAgent — Base Class

```typescript
// apps/worker/src/backend/ai/agents/BackgroundOperationAgent.ts
import { Agent } from 'agents';
import { getAgentByName } from 'agents';
import { ModelProvider } from '../providers/model-provider';
import type { Env } from '../../index';

export interface BackgroundState {
  jobId: string;
  status: 'idle' | 'running' | 'complete' | 'error';
  startedAt?: number;
  completedAt?: number;
  error?: string;
}

/**
 * Base class for all async, non-interactive background agents.
 * Extend this for pipeline steps, queue consumers, decomposition agents, etc.
 *
 * IMPORTANT: Always instantiate agents using getAgentByName() — never use raw
 * Durable Object primitives (idFromName + .get()).
 *
 * Example:
 *   const agent = await getAgentByName<MyAgent>(env.MY_AGENT_NAMESPACE, 'job-123');
 *   await agent.run(payload);
 */
export abstract class BackgroundOperationAgent<E extends Env = Env, S extends BackgroundState = BackgroundState>
  extends Agent<E, S> {

  protected ai!: ModelProvider;

  override initialState: S = {
    jobId: '',
    status: 'idle',
  } as S;

  override async onStart(): Promise<void> {
    this.ai = new ModelProvider(this.env);
    await this.onAgentStart();
  }

  /** Override in subclasses to add initialization logic. */
  protected async onAgentStart(): Promise<void> {}

  /** Override to implement the actual background operation. */
  abstract run(payload: unknown): Promise<void>;

  protected async setRunning(jobId: string): Promise<void> {
    await this.setState({ ...this.state, jobId, status: 'running', startedAt: Date.now() });
  }

  protected async setComplete(): Promise<void> {
    await this.setState({ ...this.state, status: 'complete', completedAt: Date.now() });
  }

  protected async setError(error: string): Promise<void> {
    await this.setState({ ...this.state, status: 'error', error, completedAt: Date.now() });
  }
}
```

### 14.3 FrontendInteractiveAgent — Base Class

```typescript
// apps/worker/src/backend/ai/agents/FrontendInteractiveAgent.ts
import { AIChatAgent } from 'agents/ai-chat-agent';
import { getAgentByName } from 'agents';
import { ModelProvider } from '../providers/model-provider';
import type { Env } from '../../index';

export interface InteractiveState {
  userId: string;
  sessionId: string;
  connectedAt: number;
  systemPrompt: string;
  messageHistory: { role: 'user' | 'assistant'; content: string }[];
}

/**
 * Base class for all real-time, user-facing agents that stream responses
 * over WebSockets and integrate with assistant-ui.
 *
 * IMPORTANT: Always instantiate using getAgentByName() — never use raw DO primitives.
 *
 * Example:
 *   const agent = await getAgentByName<ReviewChatAgent>(env.REVIEW_CHAT, sessionId);
 *   // Then route WebSocket upgrade to its DO stub
 */
export abstract class FrontendInteractiveAgent<E extends Env = Env, S extends InteractiveState = InteractiveState>
  extends AIChatAgent<E, S> {

  protected ai!: ModelProvider;

  override initialState: S = {
    userId: '',
    sessionId: '',
    connectedAt: 0,
    systemPrompt: '',
    messageHistory: [],
  } as S;

  override async onStart(): Promise<void> {
    this.ai = new ModelProvider(this.env);
    await this.onAgentStart();
  }

  /** Override in subclasses to add initialization logic (e.g., load rules). */
  protected async onAgentStart(): Promise<void> {}

  override async onConnect(connection: any, ctx: any): Promise<void> {
    const url  = new URL(ctx.request.url);
    const userId    = url.searchParams.get('userId')    ?? 'anonymous';
    const sessionId = url.searchParams.get('sessionId') ?? this.name;
    connection.setState({ userId, sessionId, connectedAt: Date.now() });
    await this.onClientConnect(connection, ctx);
  }

  /** Override to implement connection setup (e.g., send initial snapshot). */
  protected async onClientConnect(_connection: any, _ctx: any): Promise<void> {}

  /** Override to handle streaming tool calls in subclasses. */
  protected async executeTools(_toolName: string, _args: unknown): Promise<unknown> {
    throw new Error(`Tool ${_toolName} not implemented`);
  }
}
```

### 14.4 Factory Usage Pattern

**Always use `getAgentByName()` — never use raw Durable Object stubs:**

```typescript
import { getAgentByName } from 'agents';
import { OrchestratorAgent } from './agents/OrchestratorAgent';

// ✅ Correct — SDK factory pattern
const orchestrator = await getAgentByName<OrchestratorAgent>(
  env.ORCHESTRATOR_AGENT,
  `session-${planVersionId}`
);
await orchestrator.startReview(planVersionId);

// ❌ Wrong — raw DO primitives
const id   = env.ORCHESTRATOR_AGENT.idFromName(`session-${planVersionId}`);
const stub = env.ORCHESTRATOR_AGENT.get(id);
await stub.startReview(planVersionId); // No type safety, no SDK lifecycle
```

### 14.5 Refactoring Existing Agents

All agents created in Phase 2 are updated (not rewritten) to extend the appropriate base class:

| Agent | Previous Base | Phase 3 Base |
|-------|--------------|--------------|
| `StandardsAgent` | `Agent` | `BackgroundOperationAgent` |
| `OrchestratorAgent` | `Agent` | `BackgroundOperationAgent` |
| `UXSpecialistAgent` | `Agent` | `BackgroundOperationAgent` |
| `PlatformSpecialistAgent` | `Agent` | `BackgroundOperationAgent` |
| `BacklogAgent` | `Agent` | `FrontendInteractiveAgent` |
| `LifecycleArtifactAgent` | *(new)* | `BackgroundOperationAgent` |
| `PatternIngestionMcp` | `McpAgent` | `McpAgent` *(unchanged)* |

---

## 15. Pillar 7 — Lifecycle Instruction & Stored-Plan Continuity

### 15.1 LifecycleArtifactAgent

A new background agent triggered at sprint boundaries. Its sole responsibility is compiling all available context into durable, committed text files that the next coding agent session picks up automatically.

```typescript
// apps/worker/src/backend/agents/LifecycleArtifactAgent.ts
export class LifecycleArtifactAgent extends BackgroundOperationAgent {

  async run(payload: LifecycleCompilePayload): Promise<void> {
    await this.setRunning(payload.projectId);

    // 1. Pull plan ledger from D1
    const ledger = await this.loadPlanLedger(payload.projectId);

    // 2. Pull active rules snapshot
    const rules = await this.loadActiveRules(payload.projectId);

    // 3. Pull template injections from this sprint
    const injections = await this.loadTemplateAnnotations(payload.reviewSessionId);

    // 4. Generate AGENTS.md via structured LLM call
    const agentsMd = await this.generateAgentsMd({ ledger, rules, injections, payload });

    // 5. Generate per-pillar rule files for .agent/rules/
    const ruleFiles = await this.generateRuleFiles({ ledger, rules, payload });

    // 6. Write outputs to R2 (KV would work too; R2 for file-like output)
    await this.persistArtifacts(payload.projectId, agentsMd, ruleFiles);

    // 7. Store snapshot in D1 for anti-vacuum continuity
    await this.storeRuleSnapshots(payload.projectId, ruleFiles, payload);

    await this.setComplete();
  }
  // ... (generateAgentsMd, generateRuleFiles, persistArtifacts, storeRuleSnapshots)
}
```

### 15.2 AGENTS.md Generation Template

The generated `AGENTS.md` is always project-specific — it is not a static template. It is produced by an LLM call with the following context:

```
System: You are a technical writer generating an AGENTS.md for a coding agent.
        This file tells the agent exactly what infrastructure, rules, and
        conventions apply to this project. Be precise and concrete.

User:   Project: {projectName}
        Platform: Cloudflare Workers + D1 + Vectorize + Agents SDK

        === Active Engineering Rules ===
        {rules.map(r => `- ${r.name}: ${r.description}`).join('\n')}

        === Completed Sprint Tasks ===
        {ledger.completedTasks.map(t => `- ${t.title}`).join('\n')}

        === Template Injections Applied ===
        {injections.map(i => `- ${i.patternId} → ${i.planSectionRef}`).join('\n')}

        === Previous AGENTS.md (if exists) ===
        {previousAgentsMd ?? 'None — first sprint'}

        Generate a structured AGENTS.md with sections:
        1. Project Overview
        2. Infrastructure & Bindings
        3. Agent Class Conventions
        4. Database Schema Map
        5. Active Engineering Rules
        6. Completed Sprint Context
        7. Anti-Regression Checklist
```

### 15.3 .agent/rules/ Directory Structure

The agent generates one `.md` file per rule category, committed to the repository under `.agent/rules/`:

```
.agent/rules/
├── 00-platform.md        ← Cloudflare-specific binding and runtime rules
├── 01-database.md        ← Schema organization, Drizzle, D1 conventions
├── 02-ai-inference.md    ← ModelProvider usage, provider selection rules
├── 03-agent-classes.md   ← Base class inheritance, getAgentByName() mandate
├── 04-api-boundaries.md  ← Admin CRUD vs journey routes, no-direct-D1-write
├── 05-pattern-injection.md ← Template annotation process, tailoring protocol
└── 99-project-specific.md ← Rules unique to this project (auto-appended)
```

Each file is structured as:

```markdown
# Rule Category: Platform Bindings

## Rules

### RULE-CF-01: Use native bindings over external HTTP
> **Enforcement:** blocking  
> **Source:** approved_global

Never call external Cloudflare product APIs via fetch(). Use the binding declared
in wrangler.jsonc instead. This applies to AI, D1, Vectorize, Queues, R2, and KV.

**Correct:**
```typescript
const result = await env.AI.run('@cf/model', { ... });
```

**Wrong:**
```typescript
const result = await fetch('https://api.cloudflare.com/ai/...');
```
```

### 15.4 Anti-Vacuum Continuity Protocol

Before any new planning or implementation sprint begins, the orchestrator must:

```
OrchestratorAgent.onStart() — Phase 3 extended version:
        │
        ├─ [Phase 2] Load approved_global rules from StandardsAgent
        │
        └─ [NEW Phase 3] Load plan ledger from D1:
               1. SELECT * FROM sprint_contexts WHERE project_id = ?
                  ORDER BY created_at DESC LIMIT 10
               2. SELECT * FROM template_annotations WHERE ...
                  (injections applied in last 3 sprints)
               3. Read latest AGENTS.md from R2 (if exists)
               4. Inject into system prompt:
                    "EXISTING SPRINT CONTEXT:\n{ledger}\n\n
                     PREVIOUSLY APPLIED TEMPLATES:\n{injections}\n\n
                     DO NOT re-derive or overwrite these unless explicitly asked."
               5. Store cross-reference context in hot SQLite for session duration
```

This guarantees no new sprint runs "blind" — every agent session inherits the full project history.

---

## 16. wrangler.jsonc Additions (Phase 3)

Add the following to the Phase 2 `wrangler.jsonc`:

```jsonc
{
  // Add to existing durable_objects.bindings array:
  "durable_objects": {
    "bindings": [
      // ... (existing Phase 2 bindings)
      { "name": "PATTERN_INGESTION_MCP",    "class_name": "PatternIngestionMcp" },
      { "name": "LIFECYCLE_ARTIFACT_AGENT", "class_name": "LifecycleArtifactAgent" }
    ]
  },

  // Add to existing migrations array:
  "migrations": [
    // ... (existing v1, v2)
    {
      "tag": "v3",
      "new_sqlite_classes": [
        "PatternIngestionMcp",
        "LifecycleArtifactAgent"
      ]
    }
  ],

  // Add R2 bucket for artifact storage:
  "r2_buckets": [
    {
      "binding": "ARTIFACTS",
      "bucket_name": "plannotator-artifacts"
    }
  ],

  // Add secrets (set via wrangler secret put):
  // STANDARDIZATION_REPO_TOKEN  — GitHub PAT for core-github-standardization
  // OPENAI_API_KEY
  // ANTHROPIC_API_KEY
  // GEMINI_API_KEY
  // CF_ACCOUNT_ID
  // AI_GATEWAY_ID
  // ADMIN_BEARER_TOKEN
}
```

---

## 17. File-System Layout (Phase 3 additions)

The following additions extend the Phase 2 layout (§10). Existing Phase 2 paths are unchanged.

```
plannotator/
├── apps/
│   └── worker/
│       └── src/
│           └── backend/                           ← NEW root for all Phase 3 modules
│               ├── ai/
│               │   ├── providers/
│               │   │   └── model-provider.ts      ← NEW: unified LLM control plane
│               │   └── agents/
│               │       ├── BackgroundOperationAgent.ts  ← NEW: base for async agents
│               │       └── FrontendInteractiveAgent.ts  ← NEW: base for WS agents
│               ├── agents/
│               │   ├── PatternIngestionMcp.ts     ← NEW: core-github-standardization MCP
│               │   └── LifecycleArtifactAgent.ts  ← NEW: AGENTS.md + .agent/rules/ emitter
│               ├── db/
│               │   └── schemas/                   ← RELOCATED + REORGANIZED from src/db/schema/
│               │       ├── index.ts
│               │       ├── configurations/
│               │       │   ├── projects.ts
│               │       │   └── rules.ts
│               │       ├── plans/
│               │       │   ├── plans.ts
│               │       │   ├── plan_versions.ts
│               │       │   └── template_annotations.ts  ← NEW
│               │       ├── annotations/
│               │       │   └── annotations.ts
│               │       ├── backlog/
│               │       │   ├── epics.ts
│               │       │   ├── sprints.ts
│               │       │   ├── tasks.ts
│               │       │   └── task_assignments.ts
│               │       ├── review/
│               │       │   ├── review_sessions.ts
│               │       │   └── specialist_reports.ts
│               │       ├── signals/
│               │       │   └── learned_preferences.ts
│               │       └── lifecycle/
│               │           ├── sprint_contexts.ts        ← NEW
│               │           └── agent_rule_snapshots.ts   ← NEW
│               └── routes/
│                   └── admin.ts                   ← NEW: drizzle-zod auto-CRUD surface
│
├── packages/
│   └── shared/
│       └── src/
│           └── patterns.ts                        ← NEW: TemplateAnnotation, TailoringGroup types
│
└── .agent/
    └── rules/                                     ← NEW: generated IDE orchestration rules
        ├── 00-platform.md
        ├── 01-database.md
        ├── 02-ai-inference.md
        ├── 03-agent-classes.md
        ├── 04-api-boundaries.md
        ├── 05-pattern-injection.md
        └── 99-project-specific.md
```

---

*End of Implementation Plan — Phases 2 & 3*
