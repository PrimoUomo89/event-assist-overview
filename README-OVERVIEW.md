# EventAssist (ea20) — Architecture Overview

EventAssist is a multi-tenant B2B SaaS platform that embeds AI-powered chat widgets
into event and conference websites. Each widget is backed by a structured knowledge base
that event platforms populate and maintain. Attendees interact through a lightweight 
JavaScript widget; the platform handles semantic search, LLM response generation, and 
result rendering — all scoped to the tenant's data.

The project is a solo architectural effort: one engineer owns the full stack from domain
modeling and database schema design through to the embeddable front-end widget and
customer-facing API. Every design decision documented here was made and implemented by
a single developer, with an explicit focus on maintainability, operational simplicity,
and long-term extensibility over short-term velocity.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Browser                                                                    │
│  ┌──────────────────────────┐                                               │
│  │  chat-embed (UMD widget) │  ← hosted on customer's event site           │
│  │  @assistant-ui/react     │                                               │
│  └──────────┬───────────────┘                                               │
└─────────────│───────────────────────────────────────────────────────────────┘
              │ HTTPS + JWT (HS256, per-deployment key)
              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  agent-backend  (Fastify)                                                   │
│  ├── /token    — issues short-lived JWTs from publicKey                     │
│  ├── /invoke   — rate-limited entry point, calls KbChatAgentV2              │
│  ├── /thread   — conversation thread management                             │
│  ├── /documents — document view rendering                                   │
│  └── /search-results — result validation state                             │
│                                                                             │
│  OpenTelemetry instrumented · pg-boss job queue (opt-in)                   │
└──────────────┬──────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  KbChatAgentV2  (packages/agents)                                           │
│  ├── Step 1: Response-type analysis LLM call (Anthropic)                   │
│  │           → no_context | specific | browse                              │
│  ├── Step 2: KbSearchAgentV2                                               │
│  │    ├── Generate search query (LLM)                                      │
│  │    ├── Embed query → OpenAI text-embedding-3-small                      │
│  │    ├── PgSearchIndex  ──► pgvector HNSW  (PostgreSQL)                  │
│  │    └── LLM validation of top results                                    │
│  └── Step 3: Final LLM response (Anthropic, minifiedDocsByType context)    │
└──────────────┬──────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  PostgreSQL                                                                 │
│  ├── documents, document_indexes (scalar columns + vector(1536))           │
│  ├── knowledge_bases, tenants, agent_deployments                           │
│  ├── agent_threads, agent_invocations, agent_logs                          │
│  ├── signing_keys, rate_limit_buckets, api_keys                            │
│  └── search_results (persisted for validation + audit)                     │
└─────────────────────────────────────────────────────────────────────────────┘

Out-of-band management:
  admin-backend (Express, no auth — internal only)
    └── admin (React/Vite dashboard)  ← tenant, KB, document, index mgmt

  customer-backend (Express, API-key auth, OpenAPI/Swagger docs)
    └── tenant self-service: manage own KBs and documents via REST API
```

---

## Monorepo Structure

```
ea20/
├── domains/                   Pure domain logic — no infrastructure imports
│   ├── knowledge-base/        Tenants, KnowledgeBases, Documents, Search, ClientApps
│   ├── agent-applications/    AgentDeployments, Threads, Invocations, Logs, ApiKeys
│   └── chat-auth/             JWT signing keys, rate limit rules and buckets
│
├── packages/                  Infrastructure implementations and shared utilities
│   ├── db/                    Pg*Repo implementations, PgSearchIndex, migrations
│   ├── agents/                KbChatAgentV2, KbSearchAgentV2, Vercel AI SDK wiring
│   ├── agent-engine/          Low-level invocation types, invoke() runner
│   ├── jobs/                  pg-boss workers (async search result validation)
│   ├── document-views-react/  DocumentSummaryCard React component
│   └── demo-data/             Seed JSON for conferences (sessions, speakers, etc.)
│
└── apps/                      Deployable services
    ├── agent-backend/         Fastify API — JWT auth, rate limiting, OTel
    ├── admin-backend/         Express API — admin CRUD (no auth, internal)
    ├── admin/                 React/Vite admin dashboard
    ├── customer-backend/      Express API — tenant self-service, OpenAPI docs
    ├── chat-embed/            React/Vite embeddable UMD/ES widget (ea-chat)
    ├── embed-host-sample/     Minimal Express host showing widget integration
    └── cmd-chat/              CLI chat tool for local dev and testing
```

DI wiring: each backend app owns a single `initializeDependencies()` function
(see `apps/agent-backend/src/services/dependencies.ts`) that instantiates all
`Pg*Repo` classes, services, and the agent chain, then passes them via a
`ServiceDependencies` interface to route handlers.

---

## Hexagonal Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Domain Layer  (domains/)                                        │
│  ─────────────────────────────────────────────────────────────  │
│  Entities: KnowledgeBase, Document, AgentDeployment,            │
│            SigningKey, ApiKey, AgentThread, ...                  │
│  Port interfaces: KnowledgeBaseRepo, DocumentRepo,              │
│                   SearchIndexPort, EmbeddingPort, ...            │
│  Domain services: KnowledgeBaseService, SearchService,          │
│                   AgentApplicationService                        │
│                                                                  │
│  Pure TypeScript — zero infrastructure imports                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │ implements
┌──────────────────────────▼──────────────────────────────────────┐
│  Infrastructure Layer  (packages/db, packages/agents)            │
│  ─────────────────────────────────────────────────────────────  │
│  PgKnowledgeBaseRepo, PgDocumentRepo, PgTenantRepo, ...         │
│  PgSearchIndex   — pgvector HNSW queries                        │
│  OpenAIEmbeddingPort — text-embedding-3-small                   │
│  KbChatAgentV2, KbSearchAgentV2 — Vercel AI SDK + LLMs         │
└──────────────────────────┬──────────────────────────────────────┘
                           │ wired by
┌──────────────────────────▼──────────────────────────────────────┐
│  Application Layer  (apps/)                                      │
│  ─────────────────────────────────────────────────────────────  │
│  initializeDependencies() assembles the full object graph        │
│  Route handlers receive ServiceDependencies (interface only)     │
│  cmd-chat CLI and Fastify API share the same domain interfaces   │
└─────────────────────────────────────────────────────────────────┘
```

The CLI tool (`cmd-chat`) and the production Fastify API both call the same
`KbChatAgentV2.invoke()` method — the domain interface is identical regardless
of invocation surface.

---

## LLM Agent Pipeline

```
User message
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│  Step 1 — Response-type analysis  (Anthropic LLM call)         │
│                                                                │
│  Input:  system prompt + last 3 conversation messages          │
│  Output: responseType: "no_context" | "specific" | "browse"   │
│          query: string (extracted search intent)               │
│          category: string | null (document category hint)      │
│          reasoning: string (internal, logged, not shown)       │
└──────────┬─────────────────────────────────────────────────────┘
           │
           ├─── no_context ──► Step 3 directly (no search)
           │
           ▼ specific | browse
┌────────────────────────────────────────────────────────────────┐
│  Step 2 — KbSearchAgentV2                                      │
│                                                                │
│  1. Embed query → OpenAI text-embedding-3-small (1536-dim)     │
│  2. PgSearchIndex HNSW query (pgvector, m=16, ef_construction=64)│
│     ├── vector only (no filter)                                │
│     ├── filter only (structured filter tree)                   │
│     └── vector + filter (CTE: preselect → filter join)        │
│  3. LLM validates top-5 results (valid / invalid / error)      │
│  4. Persist search_result row + validation state               │
│  5. Optional: enqueue async re-validation job (pg-boss)        │
│                                                                │
│  Returns: minifiedValidResults, hasValidResults, searchResultId│
└──────────┬─────────────────────────────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────────────────────────────┐
│  Step 3 — Final LLM response  (Anthropic)                      │
│                                                                │
│  Context: minifiedDocsByType templates (compact JSON per doc)  │
│  Output:  text, type (GENERAL_INFORMATION/SPECIFIC/BROWSE),    │
│           documentIds (SPECIFIC), searchResultId (BROWSE)      │
└──────────┬─────────────────────────────────────────────────────┘
           │
           ▼
     API response → chat-embed renders by type:
       GENERAL_INFORMATION → plain text answer
       SPECIFIC            → text + DocumentSummaryCard(s)
       BROWSE              → text + paginated result list
```

Every invocation is persisted with `parentInvocationId` + `rootInvocationId`
linking the three LLM calls into a single debuggable lineage chain.

---

## Schema-Driven Flexible Indexing

Each `KnowledgeBase` row carries a `schema` field (JSONB) that fully describes
how that knowledge base's documents are indexed, searched, and rendered. This
makes the platform content-agnostic: the same engine handles conference sessions,
speaker profiles, exhibitor booths, or FAQs without any code changes.

### KnowledgeBaseSchema fields

| Field | Purpose |
|---|---|
| `indexesByCategory` | Per-category `IndexDefinition[]`. Each definition extracts a scalar or embedding value from a document's JSON metadata using a json_path, concatenate_paths, or Handlebars value_template. |
| `textTemplatesByCategory` | Handlebars templates that produce the full-text string sent to the embedding model for semantic indexing. |
| `summaryCardSchemas` | Handlebars templates for `DocumentSummaryCard` (header, avatar, items with icons/hrefs). |
| `documentViewTemplateBlocks` | Handlebars templates for expanded document detail views. |
| `minifiedDocsByType` | Compact JSON templates (per category) passed to LLMs as records context — keeps token counts small. |

### IndexDefinition types

```typescript
type IndexDefinition =
  | { json_path: string;         type: IndexType; optional?: boolean }
  | { concatenate_paths: string[]; type: IndexType; ... }
  | { value_template: string;    type: IndexType; for_each_path?: string; ... }

type IndexType = "enum" | "int" | "numeric" | "date" | "ts" | "embedding"
```

### document_indexes table layout

Each document produces N rows in `document_indexes`, one per defined index:

```
document_indexes
  id            uuid PK
  document_id   uuid FK → documents
  kb_id         uuid FK → knowledge_bases
  ref           text        (field path / template name)
  category      text        (mirrors parent document category)
  enum_value    text
  int_value     bigint
  numeric_value numeric
  date_value    date
  ts_value      timestamptz
  embedding     vector(1536)   ← pgvector column, HNSW indexed
```

The HNSW index is created with `m=16, ef_construction=64` using `vector_l2_ops`.

### PgSearchIndex query modes

`PgSearchIndex` implements `SearchIndexPort` with three runtime paths:

1. **Vector only** — embeds the query, scans `document_indexes.embedding` via HNSW,
   joins back to `documents` for soft-delete filtering.
2. **Filter only** — walks a `FilterNode` composition tree to SQL predicates
   (`$and`/`$or` combiners, `$eq`/`$ne`/`$lt`/`$lte`/`$gt`/`$gte`/`$in`/`$between`
   operators), returns structured matches with score=1.0.
3. **Vector + filter** — uses a CTE to preselect the top-N vector candidates
   (10× or 50× overselect), then joins against a `filtered_docs` CTE that applies
   the structured filter tree. Avoids post-filtering the entire index.

All three modes support optional deduplication by document ID or by
`(category, external_id)` using window functions.

---

## Auth and Security

The platform uses three distinct auth mechanisms, each scoped to its surface.

### 1. Chat embed JWT (per-deployment HS256)

Each `AgentDeployment` has one or more `SigningKey` rows with lifecycle states:
`active → verify_only → retired`. The embed receives a short-lived JWT from
`/token` using the deployment's `publicKey` (safe to embed in client-side config).
Signing is done server-side with the secret key, which never leaves the backend.
JWT claims carry `threadId` and `deploymentId`; rate limiting is enforced per
thread via `RateLimitBucket`. Key rotation is zero-downtime: new keys become
`active` while old keys stay in `verify_only` until their `verifyUntil` window
passes.

### 2. Customer API keys (bcrypt-hashed)

`ApiKey` rows store only a bcrypt hash (`keyHash`) — the plaintext is shown once
at creation and never stored. Keys carry a `mode` enum (`full | runtime | admin`)
controlling capability scope. Deletion is soft (`deletedAt`), preserving audit
history.

### 3. Admin backend (intentionally unauthenticated)

`admin-backend` exposes an internal CRUD API intended for operation behind a
private network boundary. No JWT or API key is required — the security boundary
is the network, not the application layer.

### CORS

`AgentDeployment` carries an `allowed_origins[]` array. The agent-backend enforces
origin matching per request, preventing the widget from being embedded on
unauthorized domains.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | TypeScript (strict, throughout) |
| Monorepo | pnpm workspaces + Turborepo |
| API (chat) | Fastify |
| API (admin, customer) | Express |
| Frontend | React + Vite |
| UI primitives | Radix UI + Tailwind CSS |
| State management | Redux Toolkit |
| Tables | TanStack Table |
| Chat UI | @assistant-ui/react |
| Database | PostgreSQL + pgvector extension |
| Vector index | HNSW (m=16, ef_construction=64) |
| Migrations | node-pg-migrate |
| Job queue | pg-boss (PostgreSQL-backed) |
| LLM orchestration | Vercel AI SDK (`ai` package) |
| LLM providers | Anthropic (Claude), OpenAI (GPT) |
| Embeddings | OpenAI text-embedding-3-small (1536-dim) |
| JWT | jose |
| Password/key hashing | bcrypt |
| Observability | OpenTelemetry (agent-backend) |

---

## Architecture Notes

The decisions below are framed as trade-offs, not defaults. Each one was chosen
after considering the alternatives.

### 1. Domains import nothing from infrastructure

`domains/` contains zero `import` statements referencing `packages/` or `apps/`.
Domain services depend only on port interfaces (TypeScript abstractions), which
infrastructure implementations satisfy. The benefit is testability and portability:
swapping Postgres for another store requires only a new `Pg*Repo`-shaped class.
The cost is more indirection and more boilerplate at the DI boundary.

### 2. Schema-as-data (KnowledgeBaseSchema JSONB) vs. schema-as-code

Index definitions, rendering templates, and LLM context shapes are stored as JSONB
on the `knowledge_bases` row rather than encoded in application code. This lets a
single running deployment serve conferences with entirely different content
structures without a redeploy. The trade-off is that schema validation shifts to
runtime rather than compile time, and template errors surface during indexing
rather than during the build.

### 3. Three LLM calls vs. a single agentic loop

`KbChatAgentV2` makes exactly three sequential LLM calls (response-type analysis
→ search query generation + validation → final response) rather than giving a
single agent tool-use capabilities and letting it loop. The three-call structure
produces deterministic execution paths, predictable latency budgets, and a clear
audit trail (every step is a separate `agent_invocation` row with `parentInvocationId`
linkage). The trade-off is less flexibility: complex multi-hop reasoning is not
supported by this design.

### 4. pgvector + HNSW vs. a dedicated vector database

Using pgvector keeps the vector index co-located with the relational data.
The filter+vector query mode (Case 3 in `PgSearchIndex`) expresses the combined
search as a single SQL CTE — the planner joins the HNSW preselect against a
structured filter on the same connection, with no inter-service round-trip. The
trade-off is that pgvector's recall/throughput ceiling is lower than purpose-built
ANN databases at very large scale. For the target workload (per-tenant knowledge
bases of thousands to low-millions of documents), the operational simplicity
outweighs the ceiling.

### 5. Soft deletes (`deleted_at`) vs. hard deletes

All document and key deletion is soft: rows are marked `deleted_at` rather than
removed. This preserves audit history and — critically — keeps `search_result`
rows referencing valid document IDs even after the document is logically deleted.
The trade-off is that queries must always filter `WHERE deleted_at IS NULL`, and
storage grows monotonically unless a separate archive job runs.

### 6. Per-deployment signing keys vs. a shared JWT secret

Each `AgentDeployment` has its own `SigningKey` set. Compromising one deployment's
key does not affect any other deployment. Key rotation is zero-downtime: old keys
remain in `verify_only` state while new keys are issued. The trade-off is a more
complex key lifecycle (active/verify_only/retired states) compared to rotating a
single application secret.

### 7. pg-boss vs. a separate queue service

Background jobs (async search result document validation) are dispatched via
pg-boss, which stores jobs in a dedicated Postgres schema on the same database
instance. This keeps the operational posture to a single Postgres dependency —
no Redis, no SQS, no separate queue service to provision, monitor, or back up.
The trade-off is that the job queue shares the database's I/O budget and cannot
be scaled independently from the primary store.
