# LEXE Platform - Architecture Reference (Source of Truth)

> **Auto-generated from codebase exploration: 2026-02-20**
> **Purpose:** Single Source of Truth per audit di tutti i documenti della piattaforma

---

## 1. CONTAINER STACK (13 servizi)

| Servizio | Container | Porta | Tecnologia | Ruolo |
|----------|-----------|-------|------------|-------|
| lexe-postgres | lexe-postgres | 5435 | PostgreSQL 16 + pgvector | DB SISTEMA (core, memory, Logto, LiteLLM) |
| lexe-max | lexe-max | 5436 | PostgreSQL 17 + pgvector 0.7.4 + Apache AGE + ParadeDB pg_search + pg_trgm | DB KB LEGAL |
| lexe-valkey | lexe-valkey | 6381 | Valkey 8 | Cache, session, L0/L1 memory |
| lexe-logto | lexe-logto | 3304-3305 | Logto | Auth CIAM |
| lexe-litellm | lexe-litellm | 4001 | LiteLLM | LLM Gateway (OpenRouter) |
| lexe-core | lexe-core | 8100 | FastAPI Python 3.11+ | API Gateway, Auth, Identity, Agent |
| lexe-orchestrator | lexe-orchestrator | 8102 | FastAPI Python | ORCHIDEA Pipeline (**DISABLED default**) |
| lexe-memory | lexe-memory | 8103 | FastAPI Python | Memory L0-L4 |
| lexe-tools | lexe-tools | 8021 | FastAPI Python | Legal tools IT |
| lexe-webchat | lexe-webchat | 3013 | React 18 + Vite + TypeScript | Frontend chat utente |
| lexe-admin | lexe-admin | 3014 | React + Vite | Admin panel frontend |
| lexe-temporal | lexe-temporal | 7234 | Temporal 1.24 | Workflow orchestration |
| lexe-temporal-ui | lexe-temporal-ui | 8180 | Temporal UI | Temporal dashboard |

### NON ESISTE nella piattaforma

- **Qdrant** — embeddings in PostgreSQL pgvector
- **Neo4j** — graph in Apache AGE (estensione PostgreSQL)
- **Redis** — sostituito da Valkey 8
- **Open WebUI** — frontend custom React (lexe-webchat)

---

## 2. NETWORK

| Network | Tipo | Scopo |
|---------|------|-------|
| lexe_internal | Internal | Backend-only communication |
| shared_public (leo_public) | External | Traefik ingress (condiviso con LEO) |

---

## 3. DATABASE

### lexe-postgres (porta 5435) — SISTEMA

- **User:** `lexe/lexe` (SUPERUSER)
- **Schema `core`:**
  - `tenants`, `contacts`, `conversations`, `conversation_messages`
  - `responder_personas` (system prompts per tenant)
  - `evidence_packs` (LEGIS audit trail, mig. 008)
  - `verification_logs` (anti-hallucination, mig. 008)
  - `model_role_defaults` (role→model globale, mig. 009)
  - `tenant_model_roles` (override per tenant, mig. 009)
  - `llm_model_catalog`, `tool_catalog`, `persona_catalog` (cataloghi globali, mig. 007)
  - `tool_execution_logs` (tool logging, mig. 006)
  - `lexorc_sessions` (LEXORC blackboard, mig. 012)
  - `legis_usage_patterns` (intent detection learning, mig. 013)
- **Schema `memory`:**
  - `working_memories` (L1 persistence)
  - `episodic_vectors` (L2, pgvector 1536d)
  - `semantic_vectors` (L3, pgvector 1536d)
  - `delta_tracking` (idempotency)
  - `audit_log` (GDPR)

### lexe-max (porta 5436) — KB LEGAL

- **User staging:** `lexe_kb/lexe_kb` | **User prod:** `lexe_max/lexe_max`
- **Estensioni:** pgvector 0.7.4, Apache AGE, ParadeDB pg_search, pg_trgm, uuid-ossp, unaccent
- **Schema `kb`:**
  - `work` (69 codici/leggi)
  - `normativa` (articoli estratti)
  - `normativa_chunk` (chunks per retrieval)
  - `normativa_chunk_embeddings` (vector 1536d)
  - `annotation` (note Brocardi)
  - `annotation_embeddings` (vector)
  - `massime` (46,767 totali, 38,718 active)
  - `norms` (4,128)
  - `massima_norms` (42,338 link)
  - `graph_edges` (58,737 CITES)
  - `embeddings` (41,437)
  - `sections`, `categories`, `category_predictions_v2`
- **Graph:** Apache AGE graph `lexe_knowledge`
- **27 migration files** (001-070)

---

## 4. LLM MODEL ALIASES (litellm/config.yaml)

| Alias | Modello Sottostante | Uso |
|-------|-------------------|-----|
| lexe-fast | openrouter/google/gemini-2.5-flash | Intent detection, follow-up, classificazione |
| lexe-primary | openrouter/mistralai/mistral-large-2512 | Chat, synthesis, ricerca |
| lexe-complex | openrouter/openai/gpt-5.2 | Reasoning, planning (LEGIS planner) |
| legal-tier1-gemini | openrouter/google/gemini-2.5-flash | Tier 1 free |
| legal-tier2-haiku | openrouter/anthropic/claude-haiku-4.5 | Auditor, verifier |
| legal-tier2-gemini | openrouter/google/gemini-2.5-flash | Tier 2 |
| legal-tier3-sonnet | openrouter/anthropic/claude-sonnet-4.6 | High verification |
| legal-tier3-gpt5 | openrouter/openai/gpt-5.2 | Tier 3 frontier |
| lexe-embedding | openrouter/openai/text-embedding-3-small | Embeddings (1536d) |

Tutti via OpenRouter API (`https://openrouter.ai/api/v1`).

---

## 5. PIPELINE ARCHITECTURE

### Flusso di routing (customer_router.py)

```
User message → POST /api/v1/gateway/customer/stream
  │
  ├─ [ff_orchestrator_enabled?] → stream_via_orchestrator() → lexe-orchestrator:8102
  │                                 (fallback on error)
  │
  ├─ [ff_legis_agent?] → _pre_intent_route()
  │     │
  │     ├─ Short non-legal (<30 chars) → "toolloop" (0ms, heuristic)
  │     ├─ Legal hints → classify_complexity() via LLM
  │     │     ├─ FLASH + no domains → "toolloop"
  │     │     ├─ FLASH/STANDARD + domains → "legis"
  │     │     └─ DEEP/STRATEGIC → "lexorc" (if enabled) or "legis"
  │     │
  │     ├─ route="lexorc" & ff_lexorc_enabled → LexeOrchestrator.run()
  │     ├─ route="legis" → legis_agent_pipeline()
  │     └─ route="toolloop" → direct LiteLLM streaming
  │
  └─ [default / fallback] → direct LiteLLM streaming with tool calls
```

### Pipeline default: Direct LiteLLM Streaming (toolloop)
- `ff_legis_agent = False` by default
- Singola chiamata LiteLLM con tool calling
- Fallback universale

### LEGIS Agent Pipeline (ff_legis_agent=True)
Fasi: Intent Detection → Planner → Researcher (tools paralleli) → Verifier → Synthesizer

| Fase | File | Modello |
|------|------|---------|
| Intent Detector | `agent/intent_detector.py` | lexe-fast (5s timeout) |
| Planner | `agent/planner.py` | lexe-complex / GPT-5.2 (60s) |
| Researcher | `agent/researcher.py` | lexe-primary / Mistral Large (120s) |
| Verifier | `agent/verifier.py` | legal-tier2-haiku / Claude Haiku 4.5 (30s) |
| Synthesizer | `agent/synthesizer.py` | lexe-primary / Mistral Large (120s) |

Prompt templates: `lexe-core/src/lexe_core/prompts/v1/`

### LEXORC Multi-Agent (ff_lexorc_enabled=True)
5 agenti specializzati + orchestratore

| Agente | File | Modello | Timeout |
|--------|------|---------|---------|
| Classifier | `agent/classifier.py` | lexe-fast | 5s |
| Norm Agent | `agent/agents/norm_agent.py` | lexe-primary | 20s |
| Caselaw Agent | `agent/agents/caselaw_agent.py` | lexe-primary | 20s |
| Doctrine Agent | `agent/agents/doctrine_agent.py` | lexe-primary | 20s |
| Auditor | `agent/agents/auditor.py` | legal-tier2-haiku | 10s |
| Synthesizer | `agent/agents/lexorc_synthesizer.py` | lexe-primary | 120s |

Orchestratore: `agent/advanced_orchestrator.py`, Parallel engine: `agent/parallel_engine.py`
Blackboard: `agent/blackboard.py` (Valkey-backed, DB 3, TTL 24h)

### ORCHIDEA Pipeline (ff_orchestrator_enabled=True)
8 fasi in `lexe-orchestrator/src/lexe_orchestrator/phases/`:
phase0_analysis → phase1_capture → phase2_semantic → phase3_drafting → phase4_review → phase5_delivery → phase6_verification → phase7_feedback
**DISABLED globalmente.**

---

## 6. FEATURE FLAGS (config.py)

| Flag | Default | Env Var | Scopo |
|------|---------|---------|-------|
| ff_orchestrator_enabled | `False` | LEXE_FF_ORCHESTRATOR_ENABLED | ORCHIDEA pipeline |
| ff_orchestrator_tenants | `""` | LEXE_FF_ORCHESTRATOR_TENANTS | Per-tenant ORCHIDEA |
| ff_legis_agent | `False` | LEXE_FF_LEGIS_AGENT | LEGIS pipeline |
| ff_legis_allow_bypass | `False` | LEXE_FF_LEGIS_ALLOW_BYPASS | User can disable LEGIS |
| ff_lexorc_enabled | `False` | LEXE_FF_LEXORC_ENABLED | LEXORC multi-agent |
| ff_lexorc_tenants | `""` | LEXE_FF_LEXORC_TENANTS | Per-tenant LEXORC |
| ff_memory_v2 | `False` | LEXE_FF_MEMORY_V2 | Memory v2 |
| ff_memory_profile_evolve | `False` | LEXE_FF_MEMORY_PROFILE_EVOLVE | Profile evolution |
| ff_facts_llm | `False` | LEXE_FF_FACTS_LLM | LLM fact extraction |
| ff_logto_customers | `True` | LEXE_FF_LOGTO_CUSTOMERS | Logto auth |
| ff_strict_tenant_no_fallback | `False` | LEXE_FF_STRICT_TENANT_NO_FALLBACK | Strict tenant |
| ff_rbac_backend_truth | `False` | LEXE_FF_RBAC_BACKEND_TRUTH | Backend RBAC |

---

## 7. MEMORY SYSTEM (lexe-memory, porta 8103)

| Layer | Storage | TTL | File |
|-------|---------|-----|------|
| L0 Session | Valkey | 24h | `layers/l0_session.py` |
| L1 Working | Valkey + PostgreSQL (write-through) | 7d | `layers/l1_working.py` |
| L2 Episodic | PostgreSQL pgvector (1536d) | Permanent | `layers/l2_episodic.py` |
| L3 Semantic | PostgreSQL pgvector (1536d) | Permanent | `layers/l3_semantic.py`, `l3_semantic_v2.py` |
| L4 Graph | PostgreSQL Apache AGE | Permanent | `layers/l4_graph.py` |

Moduli aggiuntivi: RAG system (`rag/`), Brainprint profile (`profile/`), Delta tracking, Circuit breaker per Valkey

---

## 8. LEGAL TOOLS (lexe-tools-it, porta 8021)

| Tool | File | Endpoint |
|------|------|----------|
| normattiva | `tools/normattiva.py` | `/api/v1/tools/normattiva/search` |
| eurlex | `tools/eurlex.py` | `/api/v1/tools/eurlex/search` |
| infolex | `tools/infolex.py` | `/api/v1/tools/infolex/search` |
| lex_search | `tools/lex_search.py` | `/api/v1/tools/lex-search` |
| lex_enricher | `tools/lex_enricher.py` | - |
| kb_search | `tools/kb_search.py` | `/api/v1/tools/kb-search` |
| one_search | `tools/one_search.py` | `/api/v1/tools/one-search` |
| normativa_kb_search | `tools/normativa_kb_search.py` | - |
| vigenza_fast | `tools/vigenza_fast.py` | `/api/v1/tools/vigenza-fast` |
| graph_service | `tools/graph_service.py` | - |
| health_monitor | `tools/health_monitor.py` | - |

**NON ESISTONO:** `brocardi_search`, `dejure_search`, `gazzetta_ufficiale`

---

## 9. ADMIN SYSTEM

### Backend (lexe-core/src/lexe_core/admin/)
- **61+ endpoint** sotto `/api/v1/admin`
- **11 router:** me, tenants, settings, models, tools, contacts, conversations, dashboard, audit, limits, catalog, tenant_editor, active_prompt
- **9 service:** model_service, tool_service, catalog_service, litellm_keys, litellm_db, dashboard_service, settings_service, limits_service, audit
- **RBAC:** 6 ruoli (superadmin → viewer)

### Frontend
- `lexe-admin` (porta 3014) — app React separata
- Admin sections integrate in `lexe-webchat`

---

## 10. MIGRATIONS (lexe-core, 13 files)

| # | File | Contenuto |
|---|------|-----------|
| 001 | admin_panel.sql | Admin panel, audit_log |
| 002 | rbac_tables.sql | RBAC tables |
| 003 | personas_rls.sql | Persona RLS |
| 004 | persona_indexes.sql | Performance indexes |
| 005 | personas_permissions.sql | Permission refinements |
| 006 | tool_logging_litellm_isolation.sql | tool_execution_logs + LiteLLM columns |
| 007 | global_catalogs.sql | llm_model_catalog, tool_catalog, persona_catalog |
| 008 | legis_agent.sql | evidence_packs, verification_logs |
| 009 | model_roles.sql | model_role_defaults, tenant_model_roles |
| 010 | model_config.sql | config JSONB on catalog |
| 011 | conversation_mode.sql | conversation_mode (legal/general/custom) |
| 012 | lexorc_foundation.sql | lexorc_sessions, evidence_packs extensions |
| 013 | intent_detector.sql | legis_usage_patterns, new role defaults |

**Prossima migrazione:** 014

---

## 11. KEY FILE PATHS (verificati)

```
lexe-core/src/lexe_core/
  config.py                              # Settings + feature flags
  main.py                                # FastAPI app + CORS
  database.py                            # DB connection
  gateway/
    customer_router.py                   # Main SSE streaming + routing
    streaming/stream_state.py            # Stream state/timeouts
    streaming/wire_protocol.py           # SSE protocol
    services/llm_config.py              # LLM config resolution
    services/response_router.py         # Response routing
    guardrails.py                        # Content guardrails
  agent/
    classifier.py                        # classify_complexity() → FLASH/STANDARD/DEEP/STRATEGIC
    intent_detector.py                   # run_intent_detector() → PipelineLevel + ScenarioType
    pipeline.py                          # LEGIS pipeline orchestration
    planner.py                           # Research plan generation
    researcher.py                        # Parallel tool execution
    verifier.py                          # Anti-hallucination verification
    synthesizer.py                       # Response synthesis with evidence
    advanced_orchestrator.py             # LEXORC orchestrator
    blackboard.py                        # Shared state (Valkey-backed)
    parallel_engine.py                   # Parallel agent execution
    scoring.py                           # Confidence scoring
    models.py                            # PipelineLevel, ScenarioType, IntentResult
    events.py                            # SSE event generators
    scenarios.py                         # Scenario definitions
    prompts.py                           # Prompt templates
    agents/norm_agent.py                 # LEXORC normativa agent
    agents/caselaw_agent.py              # LEXORC case law agent
    agents/doctrine_agent.py             # LEXORC doctrine agent
    agents/auditor.py                    # LEXORC auditor
    agents/lexorc_synthesizer.py         # LEXORC synthesizer
  prompts/v1/
    intent_detector.py                   # Intent detector prompt
    legis_planner.py                     # Planner prompt
    legis_researcher.py                  # Researcher prompt
    legis_synthesizer.py                 # Synthesizer prompt
    legis_verifier.py                    # Verifier prompt
  tools/
    executor.py                          # Tool execution
    policy.py                            # Tool policy resolution
    logging.py                           # Tool execution logging
    definitions.py                       # Tool definitions
    health.py                            # Tool health probes
    purity.py                            # Side effect tracking
  admin/
    routers/                             # 11 admin routers
    services/                            # 9 admin services
    schemas/                             # Admin schemas
  identity/
    schemas/__init__.py                  # TenantResponse, TenantCreate etc.
    service.py                           # Tenant management
  migrations/                            # 13 SQL files (001-013)

lexe-orchestrator/src/lexe_orchestrator/
  phases/                                # 8+ ORCHIDEA phases (DISABLED)
  routing/                               # Feature flags, pipeline mode
  tools/                                 # Tool catalog, execution
  clients/                               # LiteLLM, memory clients

lexe-memory/src/lexe_memory/
  layers/                                # L0-L4 implementations
  api/                                   # REST endpoints
  rag/                                   # RAG system
  retrieval/                             # Brain, fusion, rerank
  profile/                               # Brainprint

lexe-tools-it/src/lexe_tools_it/
  tools/                                 # 11 legal tools
  api/                                   # REST endpoints
  scrapers/                              # HTTP scraping

lexe-infra/
  docker-compose.yml                     # 13 services
  docker-compose.override.stage.yml      # Staging
  docker-compose.override.prod.yml       # Production
  litellm/config.yaml                    # LLM model catalog
```

---

## 12. FRONTEND STACK (lexe-webchat)

- React 18.3.1 + TypeScript 5.7.2
- Build: Vite 6.0.5
- State: Zustand 5.0.2
- Query: TanStack React Query 5.62.8
- Router: React Router DOM 6.30.3
- Auth: Logto React 3.0.16
- UI: Radix UI + Tailwind CSS 3.4.17
- Charts: Recharts 3.7.0
- i18n: i18next 24.2.0
- Icons: Lucide React
- Animation: Framer Motion

---

## 13. URLs

| Ambiente | API | Chat | Auth | LLM | Admin |
|----------|-----|------|------|-----|-------|
| **Production** | api.lexe.pro | ai.lexe.pro | auth.lexe.pro | llm.lexe.pro | admin.lexe.pro |
| **Staging** | api.stage.lexe.pro | stage-chat.lexe.pro | auth.stage.lexe.pro | - | admin.stage.lexe.pro |

---

## 14. DEPLOY

```bash
# STAGING (branch: stage)
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d

# PRODUCTION (branch: main)
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml up -d
```

Override OBBLIGATORIO: senza override mancano Traefik labels e env vars puntano ad ambiente sbagliato.

---

*Generato da esplorazione codebase — 2026-02-20*
