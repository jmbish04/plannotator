# Plannotator — Phase 2: Self-Hosted Agentic Architecture

> **Status:** Architecture approved, pending implementation.  
> **Branch:** `copilot/research-unified-infra-architecture`  
> **Companion files:** [`TASKS.json`](./TASKS.json) · [`PROMPT.md`](./PROMPT.md)

---

## Table of Contents

1. [Overview & Goals](#1-overview--goals)
2. [Technology Stack](#2-technology-stack)
3. [Pillar 1 — Self-Correcting Standards Engine & Learning Loop](#3-pillar-1--self-correcting-standards-engine--learning-loop)
4. [Pillar 2 — Hub-and-Spoke Multi-Agent Mesh](#4-pillar-2--hub-and-spoke-multi-agent-mesh)
5. [Pillar 3 — Real-Time Backlog Service & WebSocket Pub/Sub](#5-pillar-3--real-time-backlog-service--websocket-pubsub)
6. [Database & Schema Design](#6-database--schema-design)
7. [wrangler.jsonc Configuration](#7-wranglerjsonc-configuration)
8. [New Application Entry Point](#8-new-application-entry-point)
9. [Consistency & Reliability Guarantees](#9-consistency--reliability-guarantees)
10. [File-System Layout for New Code](#10-file-system-layout-for-new-code)

---

## 1. Overview & Goals

Phase 2 evolves Plannotator from a local Bun-based review server into a fully self-hosted, agentic review platform deployed on Cloudflare Workers. The existing annotation UI (`packages/ui`), plan parser, and hook plugin remain unchanged. Phase 2 wraps them with a cloud-native backend.

**Three core deliverables:**

| # | Pillar | Primary Cloudflare Primitives |
|---|--------|-------------------------------|
| 1 | Self-Correcting Standards Engine | Agents SDK, Vectorize, D1, Workers AI, Queues |
| 2 | Hub-and-Spoke Multi-Agent Mesh | `this.subAgent()`, `this.addMcpServer()`, Durable Objects |
| 3 | Real-Time Backlog Pub/Sub | WebSocket Hibernation API, `this.broadcast()`, `waitUntil`, D1 |

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

*End of Implementation Plan — Phase 2*
