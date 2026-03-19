---
owner: platform-team
audience: internal
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: quarterly
sensitivity: internal
last_verified: 2026-03-19
---

# LEXE Platform Architecture v2

> Definitive architecture reference for the LEXE Legal AI platform.
> All container ports, layer boundaries, database schemas, and pipeline flows
> documented here supersede any other source.

---

## Table of Contents

1. [Platform Overview](#1-platform-overview)
2. [Scope and Non-Scope](#2-scope-and-non-scope)
3. [Container Stack](#3-container-stack)
4. [3-Layer Architecture](#4-3-layer-architecture)
5. [Database Schema](#5-database-schema)
6. [Pipeline Flow](#6-pipeline-flow)
7. [Depth Budgets](#7-depth-budgets)
8. [Multi-Agent Research](#8-multi-agent-research)
9. [Tool Ecosystem](#9-tool-ecosystem)
10. [Auth and Identity](#10-auth-and-identity)
11. [Memory System](#11-memory-system)
12. [LLM Gateway](#12-llm-gateway)
13. [Monitoring](#13-monitoring)
14. [Network Topology](#14-network-topology)
15. [Feature Flags](#15-feature-flags)
16. [Key File Reference](#16-key-file-reference)

---

## 1. Platform Overview

LEXE is a multi-tenant Legal AI platform for the Italian legal domain. It combines
structured legal data (Normattiva, EUR-Lex, Cassazione massime) with LLM-powered
analysis pipelines to deliver grounded, citation-backed legal responses.

**Core facts:**

- 13 containers in production (as of 2026-03-19)
- 3-layer orchestration architecture (Orchestration, Gateway, Agent)
- Binary routing: CHAT and DIRECT intents use toolloop; all legal intents use the
  unified LEGIS pipeline via LegalStrategy
- Multi-tenant with Row-Level Security (12+ RLS-protected tables)
- GDPR-compliant consent system (4 phases complete)
- Dual database: system DB (lexe-postgres) + legal KB (lexe-max)
- All 6 primary LLM aliases route to Gemini 3 Flash Preview with per-role
  `reasoning_effort` configuration

**Repositories:**

| Repository       | Language    | Purpose                                |
|------------------|-------------|----------------------------------------|
| lexe-core        | Python      | API Gateway, Auth, Orchestration, Agent|
| lexe-webchat     | TypeScript  | User chat interface (React)            |
| lexe-admin       | TypeScript  | Admin panel SPA (React)                |
| lexe-memory      | Python      | Memory system L0-L4                    |
| lexe-tools-it    | Python      | Legal tools API (Normattiva, EUR-Lex)  |
| lexe-max         | SQL/Python  | Legal KB (normativa, massime)          |
| lexe-infra       | YAML        | Docker Compose, Traefik, ops           |
| lexe-docs        | Markdown    | Documentation                          |
| lexe-privacy     | Python      | Privacy and pseudonymization           |

---

## 2. Scope and Non-Scope

### In Scope

- Runtime architecture of all 13 production containers
- Data flow from user message to synthesized legal response
- Database schemas (core, memory, kb) and their RLS policies
- Auth flow (Logto OIDC, RBAC, tenant isolation)
- LLM model configuration and routing
- Monitoring and observability stack
- Network topology and container connectivity

### Non-Scope

- Frontend component architecture (see lexe-webchat and lexe-admin READMEs)
- Individual API endpoint documentation (see `api-lexe-core.md`)
- Deployment procedures (see `RUNBOOK-deploy.md`)
- Rollback procedures (see `RUNBOOK-rollback.md`)
- Sprint history (see `STATUS.md`)
- Business requirements and product roadmap (see `LEXE-VISION.md`)
- GDPR consent implementation details (see `gdpr-consent-system.md`)

---

## 3. Container Stack

All containers run via Docker Compose with environment-specific override files.

```
Production:  docker-compose.yml + docker-compose.override.prod.yml
Staging:     docker-compose.yml + docker-compose.override.stage.yml
```

### Container Inventory

| Container          | Image/Build   | Internal Port | External Port | Network(s)                    | Purpose                              |
|--------------------|---------------|---------------|---------------|-------------------------------|--------------------------------------|
| lexe-core          | build         | **8100**      | via Traefik   | shared_public, lexe_internal  | API Gateway, Auth, Orchestration     |
| lexe-webchat       | build         | 3013          | via Traefik   | shared_public                 | User chat frontend (React)           |
| lexe-admin         | build         | 3014          | via Traefik   | shared_public                 | Admin panel SPA (React)              |
| lexe-orchestrator  | build         | 8102          | --            | lexe_internal                 | ORCHIDEA Pipeline (FF disabled)      |
| lexe-memory        | build         | **8103**      | --            | lexe_internal                 | Memory L0-L4 microservice            |
| lexe-tools         | build         | 8021          | --            | lexe_internal                 | Legal tools API                      |
| lexe-postgres      | postgres:16   | 5432          | **5435**      | lexe_internal                 | System DB (core + memory schemas)    |
| lexe-max           | postgres:16   | 5432          | **5436**      | lexe_internal                 | Legal KB DB (kb schema)              |
| lexe-valkey        | valkey:7      | 6379          | **6381**      | lexe_internal                 | Cache, sessions, rate limits         |
| lexe-litellm       | litellm       | 4000          | **4001**      | lexe_internal, shared_public  | LLM Gateway proxy                    |
| lexe-logto         | logto         | 3001/3002     | **3304/3305** | shared_public, lexe_internal  | CIAM (OIDC provider)                 |
| lexe-temporal      | temporal      | 7233          | **7234**      | lexe_internal                 | Workflow orchestration               |
| lexe-temporal-ui   | temporal-ui   | 8080          | **8180**      | shared_public                 | Temporal dashboard                   |

**WARNING:** lexe-core listens on port **8100** inside the container, not 8000.
Prometheus scrapes `lexe-core:8100`. The CLAUDE.md repository map still says 8000
-- this document is authoritative.

### Container Dependency Graph

```
                    +------------------+
                    |   shared-traefik |  (external, shared with LEO)
                    +--------+---------+
                             |
              +--------------+----------------+
              |              |                |
        +-----v----+  +-----v------+  +------v-----+
        | webchat   |  |   admin    |  |   logto    |
        | :3013     |  |   :3014    |  |  :3304/05  |
        +-----------+  +------------+  +------+-----+
                                              |
                    +-------------------------+
                    |
              +-----v----+
              | lexe-core |------+-------+--------+--------+
              |   :8100   |      |       |        |        |
              +-----+-----+      |       |        |        |
                    |             |       |        |        |
              +-----v-----+ +----v--+ +--v----+ +-v------+ |
              |  litellm   | |valkey | |postgres| | logto | |
              |   :4001    | | :6381 | | :5435  | | mgmt  | |
              +------------+ +-------+ +--------+ +-------+ |
                    |                                        |
              +-----v------+  +----------+  +-------+       |
              |   memory   |  |  tools   |  |  max  |       |
              |   :8103    |  |  :8021   |  | :5436 |       |
              +-----+------+  +----------+  +-------+       |
                    |                                        |
              +-----v------+                                 |
              |  postgres  |<--------------------------------+
              |   :5435    |
              +------------+
```

---

## 4. 3-Layer Architecture

Orchestration v2 decomposes the original monolithic `customer_router.py` (3357 LOC)
into three layers with strict import boundaries enforced by `test_architecture.py`.

```
+================================================================+
|                    ORCHESTRATION LAYER                           |
|                                                                  |
|  contracts.py    -- TurnContext, protocols (ToolKit, LLMClient) |
|  events.py       -- DomainEvent hierarchy (17 event types)      |
|  strategy.py     -- Base Strategy protocol, DegradationPolicy   |
|  orchestrator.py -- Main run loop, phase sequencing              |
|  decomposer.py   -- Intent-to-strategy routing                  |
|                                                                  |
|  RULE: No imports from gateway/ or agent/                       |
+================================+===============================+
                                 |
                          DomainEvents (typed)
                                 |
+================================v===============================+
|                       GATEWAY LAYER                              |
|                                                                  |
|  adapters.py      -- ToolKitAdapter, HttpLLMClient,             |
|                       PipelineRunnerAdapter                      |
|  sse_adapter.py   -- DomainEvent -> SSE text/event-stream       |
|  sse_contracts.py -- SSE event schema definitions                |
|  tenant_resolver  -- JWT org_id -> tenant_id (sub-fallback)     |
|  customer_router  -- HTTP endpoints, conversation CRUD           |
|  event_sink.py    -- Persist events to core.conversation_events |
|                                                                  |
|  RULE: Implements protocols from orchestration/contracts.py     |
+================================+===============================+
                                 |
                     Tool/LLM/Pipeline calls
                                 |
+================================v===============================+
|                        AGENT LAYER                               |
|                                                                  |
|  intent_detector.py  -- 5-level intent classification            |
|  planner.py          -- ResearchPlan generation                  |
|  researcher.py       -- Parallel tool dispatch                   |
|  verifier.py         -- 3-level verification (det + LLM)        |
|  synthesizer.py      -- 7-section legal response                 |
|  scoring.py          -- Authority scoring, confidence bands      |
|  fusion.py           -- Evidence dedup, multi-source boost       |
|  unified_pipeline.py -- Single entry point for LEGIS             |
|  depth_budget.py     -- Budget config per DepthLevel             |
|  gap_reporter.py     -- Research gap analysis                    |
|  research/           -- 5 multi-agent research agents            |
|                                                                  |
|  RULE: No imports from orchestration/                           |
+================================================================+
```

### DomainEvent Hierarchy (17 types)

| Event Type           | Description                                |
|----------------------|--------------------------------------------|
| MetaEvent            | Conversation/turn metadata                 |
| PhaseEvent           | Pipeline phase transition                  |
| PreprocessingEvent   | Intent detection results                   |
| TokenEvent           | Streaming LLM tokens                       |
| ToolCallEvent        | Tool invocation started                    |
| ToolResultEvent      | Tool returned results with metadata        |
| ToolEndEvent         | Tool execution completed                   |
| ToolRunStartEvent    | Multi-tool batch started                   |
| ToolRunEndEvent      | Multi-tool batch completed                 |
| ErrorEvent           | Error with recovery info                   |
| DoneEvent            | Turn completed (trace hash, tokens, ms)    |
| PipelineDataEvent    | Generic pipeline-specific data carrier     |
| IntentEvent          | Intent detection result                    |
| PlanEvent            | Research plan generated                    |
| ResearchEvent        | Research results collected                 |
| VerifyEvent          | Verification results                       |
| SourceProgressEvent  | Real-time source query progress            |

### Strategy Pattern

Each strategy implements the `Strategy` protocol:

```python
class Strategy(Protocol):
    async def execute(self, ctx: TurnContext) -> AsyncGenerator[DomainEvent, None]: ...
```

**Active strategies (v2 binary routing):**

| Strategy        | File                       | Intents          | Description                           |
|-----------------|----------------------------|------------------|---------------------------------------|
| ToolloopStrategy| strategies/toolloop.py     | CHAT, DIRECT     | Direct LLM with optional tool calls   |
| LegalStrategy   | strategies/legal.py        | SIMPLE, STANDARD, COMPLEX | Unified LEGIS pipeline (depth-based) |

**Deprecated strategies (files retained, disconnected from routing):**

| Strategy         | File                       | Note                                  |
|------------------|----------------------------|---------------------------------------|
| LegisStrategy    | strategies/legis.py        | Replaced by LegalStrategy             |
| LexorcStrategy   | strategies/lexorc.py       | Replaced by LegalStrategy             |
| SuperToolStrategy| strategies/super_tool.py   | Experimental, never promoted          |

### Per-Capability Degradation

Each strategy defines degradation policies via `CapabilityDegradation`:

```python
@dataclass
class CapabilityDegradation:
    fallback: str | None        # Fallback strategy name
    max_retries: int            # Retry attempts
    timeout_ms: int             # Timeout in milliseconds
    circuit_breaker: int | None # Failures before circuit opens
    skip_on_failure: bool       # Continue pipeline on failure
```

**LEGIS degradation policy:**

| Capability  | Fallback      | Retries | Timeout | Circuit Breaker | Skip on Failure |
|-------------|---------------|---------|---------|-----------------|-----------------|
| planner     | direct_plan   | 1       | 15s     | --              | no              |
| researcher  | --            | 2       | 30s     | 5               | no              |
| verifier    | --            | 0       | 10s     | --              | **yes**         |
| synthesizer | --            | 1       | 60s     | --              | no              |

The verifier's `skip_on_failure=true` is a deliberate design choice: if verification
fails or times out, synthesis proceeds with unverified evidence rather than failing
the entire request.

---

## 5. Database Schema

### 5.1 lexe-postgres (System DB) -- Port 5435

Credentials: `lexe` / `lexe` (superuser; RLS provides defense-in-depth).

**Schema: `core`** (39 migrations as of 2026-03-19, 001-039)

| Table                    | RLS | Purpose                                    |
|--------------------------|-----|--------------------------------------------|
| tenants                  | yes | Multi-tenant root entity                   |
| contacts                 | yes | End-user contact records                   |
| conversations            | yes | Chat conversations                         |
| conversation_messages    | yes | Individual messages (tenant_id NOT NULL)    |
| conversation_events      | yes | EventSink persistence (21 SSE event types) |
| message_evaluations      | yes | User star ratings with trace hash          |
| responder_personas       | yes | AI persona configurations                  |
| users                    | yes | Admin/operator users                       |
| roles                    | --  | RBAC role definitions                      |
| permissions              | --  | RBAC permission definitions                |
| role_permissions         | --  | Role-to-permission mapping                 |
| tools                    | yes | Tool configurations per tenant             |
| tenant_feature_flags     | yes | Per-tenant feature flag overrides          |
| tenant_llm_models        | yes | Per-tenant model overrides                 |
| contact_groups           | yes | Contact grouping                           |
| contact_group_members    | yes | Group membership                           |
| contact_merges           | yes | Contact deduplication records              |
| channel_accounts         | yes | Channel (WhatsApp, email) accounts         |
| channel_policies         | --  | Channel routing policies                   |
| channel_routings         | --  | Channel routing rules                      |
| group_routings           | --  | Group-based routing rules                  |
| service_conversations    | --  | Service-level conversation tracking        |
| verified_identifiers     | --  | Email/phone verification records           |
| pending_verifications    | --  | Pending verification tokens                |
| webhook_endpoints        | yes | Webhook delivery endpoints                 |
| audit_log                | yes | Admin audit trail                          |
| model_groups             | --  | LLM model group definitions               |
| model_role_defaults      | --  | Default model-to-role mappings             |
| email_verification_tokens| --  | Email verification tokens                  |
| token_sessions           | --  | Auth session tracking                      |
| lexorc_sessions          | --  | LEXORC orchestration sessions              |
| super_tool_runs          | --  | Super Tool execution tracking              |
| attachments              | yes | Document upload metadata                   |

RLS function: `fn_get_tenant_id()` reads `current_setting('app.tenant_id')` set via
`SET LOCAL` at the beginning of every request. `fn_is_superadmin()` bypasses for
admin operations. 12+ tables have active RLS policies.

**Schema: `memory`** (migrations 004-007 in lexe-memory)

| Table              | RLS | Purpose                                   |
|--------------------|-----|-------------------------------------------|
| memories           | yes | Core memory records (L1-L3)               |
| semantic_vectors   | yes | L3 pgvector embeddings (1536-dim, HNSW)   |
| semantic_vectors_v2| yes | V2 embeddings with improved indexing      |
| episodic_vectors   | yes | L2 episodic memory vectors                |
| working_memories   | yes | L1 working memory entries                 |
| audit_log          | yes | Memory operation audit trail              |
| profiles           | yes | User brainprint profiles                  |
| profile_facts      | yes | Extracted facts per profile               |
| fact_history       | --  | Fact evolution tracking                   |
| delta_tracking     | --  | L0/L1 delta writeback tracking            |

### 5.2 lexe-max (Legal KB) -- Port 5436

Credentials: `lexe_kb` / `lexe_kb` (password: `lexe_kb_secret`).

**Schema: `kb`** (80+ migrations)

| Table             | Rows     | Purpose                                    |
|-------------------|----------|--------------------------------------------|
| normativa         | 13,397   | Italian legislation articles               |
| normativa_chunk   | 17,343   | Chunked text with embeddings and FTS       |
| massime           | 46,767   | Cassazione case law summaries              |
| work              | 95       | Legal codes (Codice Civile, Penale, etc.)  |
| embeddings        | --       | Vector embeddings for KB search            |
| annotation        | --       | Legal annotations and commentary           |
| norms             | 4,128    | Normalized legislation metadata (shared)   |
| graph_edges       | 58,737   | Citation graph (CITES relationships)       |
| concept_hierarchy | --       | Concept hierarchy for normativa retrieval  |

---

## 6. Pipeline Flow

### Binary Routing

The decomposer classifies every user message into one of 5 intent levels, then routes
to one of 2 active strategies:

```
User Message
    |
    v
Intent Detection (lexe-fast, ~200ms)
    |
    +---> CHAT / DIRECT ----> ToolloopStrategy
    |                              |
    |                         LLM + optional tools
    |                              |
    |                              v
    |                         SSE Response
    |
    +---> SIMPLE / STANDARD / COMPLEX ----> LegalStrategy
                                                 |
                                          unified_pipeline.py
                                                 |
                                          LEGIS Pipeline (PRVS)
                                                 |
                                                 v
                                           SSE Response
```

### LEGIS Pipeline Phases (PRVS)

```
+----------+     +-----------+     +----------+     +------------+
| PLANNING |---->| RESEARCH  |---->| VERIFY   |---->| SYNTHESIS  |
| (P)      |     | (R)       |     | (V)      |     | (S)        |
+----------+     +-----------+     +----------+     +------------+
  lexe-complex    parallel tools    lexe-verifier    lexe-primary
  15s timeout     30s timeout       10s timeout      60s timeout
                  circuit breaker   skip_on_failure
                  (5 failures)      = true
```

**Phase details:**

| Phase     | Model         | reasoning_effort | max_tokens | Output                        |
|-----------|---------------|------------------|------------|-------------------------------|
| Planning  | lexe-complex  | medium           | 16,384     | ResearchPlan (tools, norms)   |
| Research  | --            | --               | --         | Evidence[] (parallel tools)   |
| Verify    | lexe-verifier | low              | 8,192      | PASS/WARN/FAIL per evidence   |
| Synthesis | lexe-primary  | medium           | 16,384     | 7-section legal response      |

**Synthesis output format (7 sections):**

1. Riepilogo -- executive summary
2. Quadro normativo -- legislation with Normattiva deep-links
3. Giurisprudenza -- case law (Cassazione, Corte Costituzionale, CGUE)
4. Contraddizioni e criticita -- conflicts, open interpretive questions
5. Cronologia -- temporal evolution of relevant legislation
6. Livello di confidenza -- overall confidence with explanation
7. Approfondimenti suggeriti -- follow-up questions (user perspective)

---

## 7. Depth Budgets

The `unified_pipeline.py` entry point selects a DepthLevel based on intent complexity.
Each level defines strict resource budgets.

| Depth    | Tool Calls | Waves | Timeout | Planner       | Verifier | Self-Correct |
|----------|------------|-------|---------|---------------|----------|--------------|
| MINIMAL  | 8          | 1     | 20s     | deterministic | no       | no           |
| STANDARD | 20 (25*)   | 2     | 35s (40s*) | lexe-fast  | yes      | no           |
| DEEP     | 35 (40*)   | 3     | 50s (55s*) | lexe-frontier| yes     | yes (max 2)  |

*Values in parentheses from STATUS.md; the code in `depth_budget.py` is authoritative.

**Intent-to-depth mapping:**

| Intent   | Depth    | Rationale                                    |
|----------|----------|----------------------------------------------|
| SIMPLE   | MINIMAL  | Single-norm lookup, no planning needed       |
| STANDARD | STANDARD | Multi-source analysis, LLM planning          |
| COMPLEX  | DEEP     | Cross-domain, multi-norm deep analysis       |

**Budget enforcement:** `depth_budget.py` provides a `DepthBudget` object that tracks
remaining tool calls and wall-clock time. The researcher checks
`budget.can_call_tool()` before each dispatch and aborts gracefully when exhausted.

Self-correction (DEEP only) allows the verifier to request up to 2 additional
research cycles when FAIL verdicts are detected. This is gated by
ADR-D6 (`ADR-D6-self-correction-max-2.md`).

---

## 8. Multi-Agent Research

Feature flag: `ff_multi_agent_research` (default: false).

When enabled, the LEGIS researcher replaces single-pass tool dispatch with a
3-wave parallel research engine using 5 specialized agents.

### Wave Architecture

```
Wave 1 (parallel):
    +-------------------+
    |   norm_agent_v2   |  -- Normattiva + KB normativa
    |   juris_agent     |  -- Massime + sentenze
    |   doctrine_agent  |  -- Dottrina + commentary
    +-------------------+
            |
            v
Wave 2 (parallel, uses Wave 1 results):
    +-------------------+
    |  vigenza_agent    |  -- Vigenza cross-check
    |  gap_analyst      |  -- Research gap detection
    +-------------------+
            |
            v
Wave 3 (sequential):
    +-------------------+
    |  remediation      |  -- Fill gaps identified by gap_analyst
    +-------------------+
            |
            v
    Evidence Fusion (dedup + confidence boost)
```

### Budget by Complexity

| Complexity | Total Tool Calls | Description                        |
|------------|------------------|------------------------------------|
| SIMPLE     | 12               | Single-source, 1 wave              |
| STANDARD   | 22               | Multi-source, 2 waves              |
| COMPLEX    | 35               | Full 3-wave with remediation       |

### Intent Decomposer

`intent_decomposer.py` generates a `WorkPlan` with `SubTask` objects. Each sub-task
specifies the agent, priority, and dependencies. The `research_engine.py` dispatches
sub-tasks respecting dependency order and parallelism within each wave.

### Evidence Fusion

`fusion.py` merges results from all agents:

- URN deduplication (identical norms from different sources merged)
- RV (Riferimento Vigenza) dedup for case law
- Multi-source confidence boost: +5% when evidence found by 2+ agents
- Cross-reference boost: +5% when agents cite each other's findings
- Floor: 40% minimum confidence
- Hard cap: 60 if primary citation failed

### Confidence Scoring

`scoring.py` computes a weighted confidence score:

| Factor             | Weight |
|--------------------|--------|
| source_quality     | 0.30   |
| vigenza_check      | 0.25   |
| link_verification  | 0.20   |
| cross_reference    | 0.15   |
| text_coherence     | 0.10   |

**3-band thresholds:**

| Band             | Score  | UI Treatment                          |
|------------------|--------|---------------------------------------|
| VERIFIED         | >= 45  | Green badge, high confidence          |
| CAUTION          | 25-44  | Yellow badge, caveats noted           |
| LOW_CONFIDENCE   | < 25   | Red badge, explicit disclaimer        |

**Design decision:** Jurisprudence absence is NOT a penalty. Confidence measures
normative verification only; case law is informational enrichment (see ADR-D5).

---

## 9. Tool Ecosystem

8 primary tools available to the LEGIS researcher, dispatched in parallel via the
`CapabilityDispatcher` in lexe-tools-it.

| Tool                  | Source       | Description                                  | FF / Gate    |
|-----------------------|--------------|----------------------------------------------|--------------|
| normattiva_search     | Normattiva   | Italian legislation full-text search         | always       |
| normattiva_article    | Normattiva   | Fetch specific article by URN                | always       |
| eurlex_search         | EUR-Lex      | EU legislation and case law                  | always       |
| kb_search             | Internal KB  | Semantic search over legal KB (pgvector)     | always       |
| kb_normativa_search   | Internal KB  | Structured normativa query (anno, sezione)   | always       |
| infolex_search        | InfoLex      | Legal encyclopedia and commentary            | always       |
| lex_search            | Lex.do       | Legal database search                        | always       |
| vigenza_fast          | Normattiva   | Quick check if a norm is still in force      | always       |

Additional tools available but not part of the primary research set:

- `lex_enriched` -- enriched legal database search with metadata
- `web_search` -- SearXNG web search (used for massima URL resolution)

**Tool diversity enforcement:** If the planner's `sources_to_query` is too narrow,
the researcher automatically injects mandatory sources (normattiva, kb_search)
to prevent single-source bias.

**Tool definitions:** `lexe-tools-it/` contains 14 capabilities organized by the
dispatcher pattern with singleton initialization.

---

## 10. Auth and Identity

### Logto OIDC Flow

```
Browser --> Webchat/Admin SPA
    |
    v
Logto (auth.lexe.pro / auth.stage.lexe.pro)
    |
    v
JWT with claims: sub, organization_id, roles
    |
    v
lexe-core validates:
    1. JWT signature (issuer + audience)
    2. organization_id -> core.tenants.logto_organization_id
    3. If org_id missing: sub-based fallback + auto-heal
    4. SET LOCAL app.tenant_id = '{uuid}' for RLS
```

### Dual Auth Support

lexe-core supports two auth modes simultaneously:

| Mode       | Token Source  | Use Case                             |
|------------|---------------|--------------------------------------|
| Logto OIDC | JWT Bearer    | Webchat, Admin (production auth)     |
| API Key    | X-API-Key     | Machine-to-machine, testing          |

### RBAC Model

7 roles with 22 permissions:

| Role        | Scope          | Key Permissions                                 |
|-------------|----------------|-------------------------------------------------|
| superadmin  | global         | All permissions, cross-tenant access            |
| admin       | tenant         | Manage users, personas, settings, models        |
| agent       | tenant         | Handle conversations, use tools                 |
| operator    | tenant         | View conversations, limited actions             |
| user        | tenant         | Chat access, own conversations only             |
| viewer      | tenant         | Read-only access                                |
| tool_*      | global (6 sub) | Tool-specific roles (globalOnly, tools group)   |

Permission format: `CanManageTenants`, `CanViewConversations`, etc.
RBAC enforcement: `_perm: CanXxx` parameter on route handlers, NOT
`dependencies=[Depends()]` (see gotcha in MEMORY.md).

### Tenant Isolation

- Every DB query sets `SET LOCAL app.tenant_id` via RLS context
- `FF_STRICT_TENANT_NO_FALLBACK=true` in production
- Tenant resolver: `gateway/tenant_resolver.py` with sub-based fallback,
  auto-heal Logto org membership, and TTL cache (5 min)
- Valkey keys namespaced: `lexe:limits:{tid}`, `lexe:daily_convs:{tid}:{date}`

### Admin API

18 admin routers mounted under `/api/v1/admin`:

| Router               | Endpoints | Purpose                                    |
|----------------------|-----------|--------------------------------------------|
| active_prompt        | 2         | System prompt management                   |
| assessment_router    | 1         | Quality assessment (period-based)          |
| audit                | 2         | Audit log queries                          |
| catalog_router       | 4         | LLM model catalog CRUD                     |
| consent_audit        | 2         | GDPR consent audit + CSV export            |
| contacts             | 4         | Contact management                         |
| conversations        | 3         | Conversation queries                       |
| dashboard            | 1         | Dashboard statistics                       |
| experiments_router   | 3         | A/B experiment management                  |
| limits               | 2         | Tenant limits configuration                |
| me                   | 1         | Current user info                          |
| models               | 4         | Model alias CRUD                           |
| settings             | 2         | Tenant settings                            |
| tenant_editor_router | 2         | Tenant editing                             |
| tenants              | 3         | Tenant provisioning + listing              |
| tools                | 3         | Tool configuration                         |
| trace_router         | 1         | Trace lookup by 8-char hash                |
| export_router        | 1         | Conversation export (json/md/html)         |

---

## 11. Memory System

lexe-memory (port 8103) implements a 5-layer memory pyramid.

### Layer Architecture

| Layer | Name     | Storage              | TTL        | Purpose                              |
|-------|----------|----------------------|------------|--------------------------------------|
| L0    | Session  | Valkey               | 24h        | Current session context              |
| L1    | Working  | Valkey + PostgreSQL   | 7 days     | Working memory, exponential decay    |
| L2    | Episodic | PostgreSQL            | 90 days    | Consolidated facts and decisions     |
| L3    | Semantic | PostgreSQL + pgvector | Permanent  | Semantic vectors (1536-dim HNSW)     |
| L4    | Graph    | Apache AGE (PG ext.)  | Permanent  | Knowledge graph (Cypher queries)     |

### Memory Flow

```
User message (use_memory_retrieval=true)
    |
    v
RETRIEVE: POST /memory/search
    query = user message
    similarity threshold = 0.3, limit = 5
    layer = L3 (pgvector HNSW, ~5ms)
    |
    v
Inject into system prompt:
    "## Memoria dell'utente
     Queste sono informazioni che conosci sull'utente: ..."
    |
    v
LLM generates response
    |
    v
STORE: store_memory_facts()
    HeuristicExtractor (12 regex IT/EN/PT)
    POST /memory/store -> L3
    Delta writeback -> L0/L1
```

### Key Characteristics

- **pgvector index:** HNSW cosine similarity, 1536 dimensions (text-embedding-3-small)
- **Deduplication:** 95% similarity threshold prevents duplicate facts in L3
- **Identity leak prevention:** Blocks identity facts without `contact_id`
- **Delta writeback:** L0/L1 via `apply_delta()` (Sprint 2)
- **GDPR:** Export (Art. 20), soft/hard delete (Art. 17), audit log
- **LLM extraction:** Implemented but OFF by default (`ff_fact_extractor_llm=False`)

### RAG v3 (Implemented, Not Active)

A conversational RAG system with hybrid RRF search (dense + BM25) is fully
implemented in `lexe-memory/src/lexe_memory/rag/` but not yet integrated into the
main pipeline. 10 modules, ~4,200 LOC. Activation blocked on router registration
and performance validation.

### API Surface

lexe-memory exposes 49 endpoints across memory CRUD, search, profile management,
brainprint operations, and health checks. All scoped by `tenant_id` + `contact_id`.

---

## 12. LLM Gateway

LiteLLM (port 4001) proxies all LLM calls with model aliasing, rate limiting,
and cost tracking.

### Primary Aliases (All Gemini 3 Flash Preview)

| Alias           | Role               | reasoning_effort | max_tokens | Pipeline Phase     |
|-----------------|--------------------|------------------|------------|--------------------|
| lexe-fast       | Intent detection   | medium           | 4,096      | Intent detector    |
| lexe-primary    | Synthesis          | medium           | 16,384     | Synthesizer        |
| lexe-complex    | Planning           | medium           | 16,384     | Planner            |
| lexe-verifier   | Verification       | low              | 8,192      | Verifier           |
| lexe-frontier   | Max quality        | high             | 16,384     | DEEP planner, premium |
| lexe-embedding  | Vector embeddings  | --               | --         | text-embedding-3-small |

### Support Models (20 total synced)

| Model               | reasoning_effort | Notes                                      |
|----------------------|------------------|--------------------------------------------|
| gpt-5-mini           | **low** (CRITICAL) | Score 1.00->4.75 with low. NEVER set max_tokens |
| qwen3.5-plus         | medium           | Judge, calibrated (self-bias -0.06)        |
| deepseek-v3.2        | medium           | Fallback, peer score 5.00 FAST             |
| claude-sonnet-4-6    | default          | Reasoning native, no override              |
| gemini-3.1-pro       | --               | Available but not primary                  |
| grok-4-1-fast        | --               | Available                                  |
| mimo-v2-flash        | --               | Available                                  |

### reasoning_effort Configuration

The `reasoning_effort` field is stored in the JSONB `config` column of
`core.llm_model_catalog`. At runtime, `config_utils.py:apply_config_override()`
reads this field and injects it into the LiteLLM API payload. This allows
per-model reasoning effort without code changes.

---

## 13. Monitoring

### Prometheus

- Scrapes `lexe-core:8100/metrics` (FastAPI + custom metrics)
- 72 recording/alerting rules in `lexe_alerts` group
- 5 critical alerts:
  1. `LexeCoreDown` -- container health
  2. `LexeErrorRateHigh` -- HTTP 5xx rate
  3. `LexeLatencyHigh` -- P95 response time
  4. `LexeSSEDropped` -- SSE connection drops
  5. `LexeEventSinkFailed` -- Event persistence failures

### Grafana

- Dashboard UID: `lexe-operations` (8 panels)
- Datasource UID: `prometheus`
- Panels: request rate, error rate, latency P50/P95, SSE active connections,
  pipeline duration histogram, tool call distribution, confidence histogram,
  evidence count histogram
- Staging auth: admin / `LeoGrafanaStage2026!`

### Quality Metrics (Prometheus Histograms)

| Metric                          | Type      | Description                        |
|---------------------------------|-----------|------------------------------------|
| lexe_pipeline_duration_seconds  | histogram | End-to-end pipeline duration       |
| lexe_evidence_count             | histogram | Evidence items per response        |
| lexe_confidence_score           | histogram | Confidence scores distribution     |
| lexe_tool_calls_total           | counter   | Tool call count by tool name       |
| lexe_limits_block_total         | counter   | Rate limit blocks by type          |
| lexe_tenant_provision_total     | counter   | Tenant provisioning by status      |

### Other Observability

| System    | Purpose                         | Status              |
|-----------|---------------------------------|---------------------|
| Jaeger    | Distributed tracing             | Staging: inactive   |
| Loki      | Log aggregation                 | Available           |
| Langfuse  | LLM observability               | Optional, graceful  |

### Assessment Endpoint

`GET /admin/assessment?period=1d|7d|30d` returns:

- Pipeline distribution (intent/depth/strategy counts)
- Evaluation statistics (star ratings, distribution)
- Anomaly detection (latency spikes, error clusters)
- SLA compliance (hybrid in-memory + DB, cached 60s)

---

## 14. Network Topology

4 Docker networks connect the container stack:

```
+--------------------------------------------------------------+
|  shared_public (external: true)                               |
|  - shared-traefik (reverse proxy, TLS termination)           |
|  - lexe-core, lexe-webchat, lexe-admin, lexe-logto           |
|  - lexe-litellm, lexe-temporal-ui                            |
+--------------------------------------------------------------+

+--------------------------------------------------------------+
|  lexe_internal (internal)                                     |
|  - lexe-core, lexe-memory, lexe-tools, lexe-orchestrator     |
|  - lexe-postgres, lexe-max, lexe-valkey, lexe-litellm        |
|  - lexe-logto, lexe-temporal                                 |
+--------------------------------------------------------------+

+--------------------------------------------------------------+
|  leo_internal (external: true, shared with LEO platform)      |
|  - shared-traefik                                            |
|  - LEO containers (separate platform, same host)             |
+--------------------------------------------------------------+

+--------------------------------------------------------------+
|  leo_data (external: true, shared with LEO platform)          |
|  - shared-prometheus, shared-grafana, shared-jaeger           |
+--------------------------------------------------------------+
```

**Key networking facts:**

- Traefik is shared with the LEO platform on the same host via `shared_public`
- `shared_public` is the only network with external access (Traefik TLS termination)
- `lexe_internal` is bridge-mode, no external exposure
- lexe-core connects to both `shared_public` (Traefik) and `lexe_internal` (backends)
- `leo_data` provides Prometheus/Grafana access to lexe-core metrics

### Traefik Routes (Production)

| Host               | Service      | Port |
|--------------------|-------------|------|
| ai.lexe.pro       | lexe-webchat | 3013 |
| api.lexe.pro      | lexe-core    | 8100 |
| auth.lexe.pro     | lexe-logto   | 3304 |
| admin.lexe.pro    | lexe-admin   | 3014 |
| llm.lexe.pro      | lexe-litellm | 4001 |

---

## 15. Feature Flags

Feature flags are stored in `core.tenant_feature_flags` with per-tenant overrides.
Global defaults are in `config.py`.

### Pipeline and Agent Flags

| Flag                           | Default | Description                                   |
|--------------------------------|---------|-----------------------------------------------|
| ff_legis_agent                 | true    | Enable LEGIS pipeline for legal intents       |
| ff_orchestration_v2            | true    | Use 3-layer orchestration architecture        |
| ff_lexorc_enabled              | false   | Enable LEXORC multi-agent for COMPLEX         |
| ff_multi_agent_research        | false   | Enable 3-wave parallel research               |
| ff_orchestrator_enabled        | false   | Enable legacy ORCHIDEA pipeline container      |
| ff_super_tool_enabled          | false   | Enable Super Tool strategy                    |
| ff_self_correction             | true    | Allow verifier to request re-research (DEEP)  |

### Memory and Retrieval Flags

| Flag                           | Default | Description                                   |
|--------------------------------|---------|-----------------------------------------------|
| ff_memory_retrieval            | true    | Enable L3 memory retrieval in chat            |
| ff_memory_store                | true    | Enable memory fact extraction and storage     |
| ff_fact_extractor_llm          | false   | Use LLM for fact extraction (vs heuristic)    |
| ff_rag_v3                      | false   | Enable RAG v3 conversational retrieval        |

### Auth and Consent Flags

| Flag                           | Default | Description                                   |
|--------------------------------|---------|-----------------------------------------------|
| ff_consent_required            | true    | Require GDPR consent before chat              |
| ff_logto_auth                  | true    | Use Logto OIDC authentication                 |
| ff_strict_tenant_no_fallback   | true    | No fallback tenant resolution (strict mode)   |

### Quality and Observability Flags

| Flag                           | Default | Description                                   |
|--------------------------------|---------|-----------------------------------------------|
| ff_quality_metrics             | true    | Emit Prometheus quality histograms            |
| ff_event_sink                  | true    | Persist SSE events to conversation_events     |
| ff_langfuse                    | false   | Enable Langfuse LLM observability             |
| ff_confidence_badges           | true    | Show confidence badges in frontend            |
| ff_citation_badges             | true    | Show interactive citation [N] badges          |
| ff_follow_up_suggestions       | true    | Generate follow-up question suggestions       |

### Admin and Operations Flags

| Flag                           | Default | Description                                   |
|--------------------------------|---------|-----------------------------------------------|
| ff_admin_panel                 | true    | Enable admin panel access                     |
| ff_experiments                 | false   | Enable A/B experiment framework               |
| ff_limits_enforcement          | true    | Enable tenant rate limiting (429/403)         |
| ff_welcome_email               | true    | Send welcome email on tenant provisioning     |

### Tool Flags

| Flag                           | Default | Description                                   |
|--------------------------------|---------|-----------------------------------------------|
| ff_normattiva_deep_link        | true    | Use Normattiva article deep-links             |
| ff_vigenza_check               | true    | Enable vigenza (in-force) checking            |
| ff_massima_url_resolution      | true    | Resolve massima URLs via SearXNG              |
| ff_negative_cache              | true    | Cache failed URL resolutions (24h TTL)        |
| ff_tool_diversity              | true    | Auto-inject mandatory tool sources            |

---

## 16. Key File Reference

Top 20 files by architectural significance.

| # | Path                                                          | Purpose                              |
|---|---------------------------------------------------------------|--------------------------------------|
| 1 | `lexe-core/src/lexe_core/gateway/customer_router.py`         | Main chat endpoint, SSE streaming    |
| 2 | `lexe-core/src/lexe_core/orchestration/orchestrator.py`      | Orchestration run loop               |
| 3 | `lexe-core/src/lexe_core/orchestration/contracts.py`         | TurnContext, protocols               |
| 4 | `lexe-core/src/lexe_core/orchestration/events.py`            | DomainEvent hierarchy (17 types)     |
| 5 | `lexe-core/src/lexe_core/orchestration/decomposer.py`        | Intent-to-strategy routing           |
| 6 | `lexe-core/src/lexe_core/orchestration/strategies/legal.py`  | LegalStrategy (unified pipeline)     |
| 7 | `lexe-core/src/lexe_core/orchestration/strategies/toolloop.py`| ToolloopStrategy (chat/direct)      |
| 8 | `lexe-core/src/lexe_core/agent/unified_pipeline.py`          | LEGIS single entry point             |
| 9 | `lexe-core/src/lexe_core/agent/depth_budget.py`              | DepthBudget per MINIMAL/STD/DEEP     |
| 10| `lexe-core/src/lexe_core/agent/intent_detector.py`           | 5-level intent classification        |
| 11| `lexe-core/src/lexe_core/agent/researcher.py`                | Parallel tool dispatch               |
| 12| `lexe-core/src/lexe_core/agent/synthesizer.py`               | 7-section legal response             |
| 13| `lexe-core/src/lexe_core/agent/scoring.py`                   | Authority + confidence scoring       |
| 14| `lexe-core/src/lexe_core/agent/fusion.py`                    | Evidence dedup + confidence boost    |
| 15| `lexe-core/src/lexe_core/gateway/adapters.py`                | ToolKit, LLMClient, PipelineRunner   |
| 16| `lexe-core/src/lexe_core/gateway/sse_adapter.py`             | DomainEvent -> SSE translation       |
| 17| `lexe-core/src/lexe_core/gateway/tenant_resolver.py`         | JWT -> tenant_id resolution          |
| 18| `lexe-core/src/lexe_core/config.py`                          | Settings, FF defaults, env vars      |
| 19| `lexe-core/src/lexe_core/agent/config_utils.py`              | reasoning_effort runtime override    |
| 20| `lexe-core/src/lexe_core/admin/services/limits_enforcement.py`| Rate limiting (403/429)             |

---

## Appendix A: Servers

| Environment | IP             | SSH Key                  | Infra Path                         |
|-------------|----------------|--------------------------|------------------------------------|
| Production  | 49.12.85.92    | `~/.ssh/hetzner_leo_key` | `/opt/lexe-platform/lexe-infra`    |
| Staging     | 91.99.229.111  | `~/.ssh/id_stage_new`    | `/opt/lexe-platform/lexe-infra`    |

**WARNING:** Production = 49.12.85.92, Staging = 91.99.229.111. Do not confuse.

## Appendix B: URLs

| Environment | Webchat                      | API                        | Auth                        | Admin                       | LLM                  |
|-------------|------------------------------|----------------------------|-----------------------------|-----------------------------|-----------------------|
| Production  | https://ai.lexe.pro         | https://api.lexe.pro       | https://auth.lexe.pro       | https://admin.lexe.pro      | https://llm.lexe.pro |
| Staging     | https://stage-chat.lexe.pro | https://api.stage.lexe.pro | https://auth.stage.lexe.pro | https://admin.stage.lexe.pro| https://llm.stage.lexe.pro |

## Appendix C: Related Documents

| Document                              | Purpose                                    |
|---------------------------------------|--------------------------------------------|
| `STATUS.md`                           | Current sprint status and history          |
| `agentic-workflow.md`                 | Full agentic pipeline documentation        |
| `orchestration-v2-architecture.md`    | Orchestration v2 design (historical)       |
| `MEMORY-SYSTEMS.md`                   | Memory L0-L4 and RAG v3 comparison         |
| `llm-selection.md`                    | LLM benchmark results and model choices    |
| `gdpr-consent-system.md`             | GDPR consent system technical docs         |
| `RLS-MULTITENANCY.md`               | RLS policies and multi-tenancy design      |
| `knowhow/runbooks/RUNBOOK-deploy.md` | Deployment procedures                      |
| `knowhow/adr/ADR-D*.md`             | Architecture Decision Records (6 ADRs)     |

---

*Last verified: 2026-03-19*
*Review cadence: quarterly*
*Owner: platform-team*
