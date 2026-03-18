# LEXE Platform - Status

> Stato attuale della piattaforma LEXE Legal AI

---

## Infrastructure Status (2026-03-15)

### Production (49.12.85.92)

| Container         | Status  | Note                                           |
| ----------------- | ------- | ---------------------------------------------- |
| lexe-core         | Healthy | Gateway + Auth OIDC + Admin API                |
| lexe-orchestrator | Healthy | ORCHIDEA Pipeline (FF disabled)                |
| lexe-memory       | Healthy | L1-L3 Memory (porta 8103)                      |
| lexe-webchat      | Healthy | Frontend chat                                  |
| lexe-admin        | Healthy | Admin panel SPA                                |
| lexe-logto        | Healthy | CIAM Auth                                      |
| lexe-litellm      | Healthy | LLM Gateway (20 models synced)                 |
| lexe-postgres     | Healthy | DB + pgvector (porta 5435)                     |
| lexe-valkey       | Healthy | Cache L1                                       |
| lexe-max          | Healthy | KB Legal (normativa, massime, embeddings)      |
| lexe-tools        | Healthy | Legal Tools API (Normattiva, EUR-Lex, InfoLex) |
| lexe-temporal     | Healthy | Workflow orchestration                         |
| lexe-temporal-ui  | Healthy | Temporal dashboard                             |

**Total:** 13 containers

### Staging (91.99.229.111)

| Container     | Status  | Note                           |
| ------------- | ------- | ------------------------------ |
| lexe-core     | Healthy | stage branch                   |
| lexe-webchat  | Healthy | Built from source via override |
| lexe-admin    | Healthy | Admin panel SPA                |
| lexe-logto    | Healthy | ENDPOINT=auth.stage.lexe.pro   |
| lexe-litellm  | Healthy |                                |
| lexe-tools    | Healthy |                                |
| lexe-postgres | Healthy |                                |
| lexe-valkey   | Healthy |                                |
| lexe-max      | Healthy |                                |
| lexe-temporal | Healthy |                                |

---

## Current Sprint: Sprint 10 — Pipeline v2 + TMB + Limits (2026-03-15/18)

### Pipeline v2 — Unified Legal Pipeline
- [x] Binary routing: chat->toolloop, legal->LegalStrategy (was 4 strategies)
- [x] DepthLevel (MINIMAL/STANDARD/DEEP) replaces 5-level PipelineLevel
- [x] 3 new scenarios: NDA_TRIAGE, COMPLIANCE_CHECK, RISK_ASSESSMENT
- [x] unified_pipeline.py: single entry point, depth-based budget
- [x] LegalStrategy: unified strategy replacing legis+lexorc+super_tool
- [x] gap_reporter.py: research gap analysis for synthesizer
- [x] depth_budget.py: budget config per DepthLevel
- [x] Super Tool disconnected from routing (files retained)
- [x] legis/lexorc strategies deprecated (files retained)
- [x] **73/73 orchestration tests, 544/560 total**

### Prompt Enrichment (Interventions A-G)
- [x] A: Contract Review Playbook (13 clauses, redline format)
- [x] B: NDA Triage (10-point checklist, GREEN/YELLOW/RED)
- [x] C: Escalation Triggers (8 universal + scenario-specific)
- [x] D: Authority Scoring (classify_authority, 5 weights)
- [x] E: DPA Checklist (18-item Art. 28 GDPR)
- [x] F: Gap Reporting ({research_gaps} placeholder)
- [x] G: Risk Matrix (5x5, unified Italian definitions)

### Tool Improvements
- [x] P2: kb_search — anno_min, anno_max, sezione filters
- [x] P4: normattiva_search vs kb_normativa_search descriptions clarified

### Massimario Links — Multi-Format Search + No Fallback (2026-03-15)
- [x] Removed Google/SearXNG fallback URL generation — return "" when no authoritative URL found
- [x] Multi-format search `_search_massima_multi()`: 3 strategies in cascade per RV, 2 per sentenza
  - Strategy 1: Exact "Rv. N" with site restriction (dejure, brocardi, altalex, cortedicassazione, studiocataldi)
  - Strategy 2: Compact format ("rv645132", "Rv.645132") + cassazione context
  - Strategy 3: Bare number + cassazione massima sentenza
- [x] Negative caching (24h TTL, `__NONE__` sentinel) — prevents repeated SearXNG queries for unfindable massime
- [x] Removed dead code: `_build_search_fallback()`, `_build_search_query()`
- [x] Fixed bug: `researcher.py` imported deleted `_build_search_fallback` (silent ImportError)
- [x] Frontend: "Fonte interna" shown when no URL (already handled by CitationBadge)
- [x] Confidence: url_tier=0 naturally reduces link_verification score (20% weight)
- [x] **18/18 tests passing**

### Tenant Management Board (2026-03-18)
- [x] PLAN_PRESETS: free_demo, starter, professional (limits in tenant.settings JSONB)
- [x] `POST /admin/tenants/provision` — saga: DB → Logto → Welcome email
- [x] Slug policy: slugify + stopwords + auto-dedup (-2, -3), max 63 chars
- [x] Welcome email: navy #0A1628 + cyan #06B6D4, temp password 72h warning
- [x] Enhanced `GET /admin/tenants` — user_count, conv_today, plan, usage bars, is_over_limit
- [x] Frontend: TenantProvisionDialog + ProvisionSuccessDialog + plan badges + usage bars
- [x] i18n: 36 keys IT+EN for provision dialog, plans, usage

### Limits Enforcement (2026-03-18)
- [x] `limits_enforcement.py` — LimitsEnforcer (pre-check + post-accounting)
- [x] max_users check (403) in `invite_user()`
- [x] conv/day (429) in `create_customer_conversation()` + post-accounting
- [x] tokens/day PRE (429, 5% margin) in `stream_to_orchestrator()`
- [x] tokens/day POST — INCRBY actual tokens from done event (debt tolerated, warned)
- [x] messages/conversation (429) in `stream_to_orchestrator()`
- [x] Valkey cache: `lexe:limits:{tid}` TTL 5min, invalidated on PUT /admin/limits
- [x] Prometheus: `lexe_limits_block_total{type}`, `lexe_tenant_provision_total{status}`
- [x] HTTP semantics: 403 structural (users/personas), 429 temporal (conv/tokens/messages)

### Infra Ops (2026-03-18)
- [x] `docker-compose.override.prod.yml` — HMAC key env for lexe-core
- [ ] A1: Generate HMAC key on prod server, restart lexe-core + lexe-memory
- [ ] A2: Langfuse keys on prod (localhost:3005)
- [ ] A3: Remove stale alias `qwen3-5-plus-02-15` from LiteLLM

### Document Attachments (2026-03-15)
- [x] Upload pipeline: DOCX/PDF extraction, context injection into conversation
- [x] Migration 035: `core.attachments` table
- [x] Frontend: document upload flow with char count display

### KB Concept Hierarchy (lexe-max Sprint 10 F2)
- [x] Migration 080: concept hierarchy for normativa retrieval
- [x] `populate_concept_hierarchy.py` script

---

## Previous Sprint: Sprint 9 (2026-03-12/13)

### Orchestration v2 -- Phase 3d Complete

- [x] 3-layer architecture: orchestration > gateway > agent
- [x] Native strategies: toolloop, legis, super_tool, lexorc
- [x] Per-capability degradation policies (fallback, retries, timeout, circuit breaker, skip_on_failure)
- [x] DomainEvent hierarchy (17 event types)
- [x] TurnContext dataclass with ToolKit, LLMClient, PipelineRunner protocols
- [x] Architecture boundary enforcement tests
- [x] `RawSSEEvent` deprecated (backward compat only)
- [x] **71 tests passing**

### Multi-Agent Research "La Bomba"

- [x] Intent decomposer: query → WorkPlan with SubTasks (FF `ff_multi_agent_research`)
- [x] Research engine: 3-wave parallel (Norm+Juris+Doctrine → Vigenza+Gap → Remediation)
- [x] 5 research agents: norm_v2, juris, doctrine_v2, vigenza, gap_analyst
- [x] Evidence fusion: dedup by URN/RV, confidence boost (multi-source +5%, cross-ref +5%)
- [x] Budget by complexity: SIMPLE=12, STANDARD=22, COMPLEX=35

### Confidence Scoring

- [x] BK-006 Fix: removed penalty for absent jurisprudence (not a verification gate)
- [x] 3-band thresholds: VERIFIED ≥45, CAUTION 25-44, LOW_CONFIDENCE <25
- [x] Follow-ups: always emit 3 suggestions, regex fallback for JSON parse failures
- [x] Quality metrics: Prometheus histograms for evidence, confidence, tools, pipeline duration

### Citation Badges (Frontend)

- [x] Interactive `[N]` badges with tooltip popovers (portal-based, 3 types)
- [x] Click → highlight evidence item in panel, auto-scroll, 2.5s ring effect
- [x] BUG FIX: `selectCitationMap` infinite re-render (React #185) — use inline selector

### Admin Panel Sprint 5

- [x] Unified "Modelli & Alias" tab (Gateway + Catalogo merged)
- [x] RuoliTab: compact accordion layout, test button, read-only chips
- [x] Tools AI group with `globalOnly` badge
- [x] i18n: eliminated hardcoded Italian strings
- [x] Brand Kit v2.2: real LEXe bicolor logos, fixed dark/light variant detection

### Tenant Resolution

- [x] Sub-based fallback when JWT org_id missing
- [x] Auto-heal: re-add user to Logto org
- [x] TTL cache (5 min) for org→tenant lookups

### Tools AI Model Roles

- [x] Migration 030: 6 `tool_*` roles (role_group='tools', globalOnly)
- [x] Migration 031: `lexe-embedding` alias fix (was wrongly set to gemini)

### LLM Model Consolidation

All 6 primary model aliases now point to **Gemini 3 Flash Preview**:

| Alias           | Reasoning Effort | Max Tokens | Role                |
|-----------------|------------------|------------|---------------------|
| `lexe-fast`     | medium           | 4,096      | Intent detection    |
| `lexe-primary`  | medium           | 16,384     | Synthesis           |
| `lexe-complex`  | medium           | 16,384     | Planning            |
| `lexe-verifier` | low              | 8,192      | Verification        |
| `lexe-frontier` | high             | 16,384     | Max quality writing |
| `lexe-embedding`| --               | --         | text-embedding-3-small |

**LiteLLM:** 20 models synced between staging and production (including gpt-5-mini, qwen3.5-plus, deepseek-v3.2, sonnet-4-6, gemini-3.1-pro, grok-4-1-fast, mimo-v2-flash).

### Production Deploy (2026-02-23)

- [x] All 7 repos merged stage to main and deployed
- [x] 13 containers healthy on prod
- [x] Migrations 018-025 applied on prod (LEXORC roles, model groups, super_tool_runs, conversation_events, message_evaluations)
- [x] Logto OIDC working on auth.lexe.pro, new app `lexe-admin-spa`
- [x] DNS: admin.lexe.pro configured

### KB Stats (Production)

| Metric                 | Count   |
|------------------------|---------|
| Articles (normativa)   | 13,397  |
| Chunks + embeddings    | 17,343  |
| Massime                | 46,767  |
| Codici                 | 95      |
| Norms (shared)         | 4,128   |
| Graph edges (CITES)    | 58,737  |

---

## Services URLs

### Production

| Service     | URL                    | Status |
| ----------- | ---------------------- | ------ |
| Webchat     | https://ai.lexe.pro    | OK     |
| API         | https://api.lexe.pro   | OK     |
| Auth OIDC   | https://auth.lexe.pro  | OK     |
| Admin Panel | https://admin.lexe.pro | OK     |
| LLM Gateway | https://llm.lexe.pro   | OK     |

### Staging

| Service     | URL                          | Status |
| ----------- | ---------------------------- | ------ |
| Webchat     | https://stage-chat.lexe.pro  | OK     |
| API         | https://api.stage.lexe.pro   | OK     |
| Auth OIDC   | https://auth.stage.lexe.pro  | OK     |
| Auth Admin  | https://admin.stage.lexe.pro | OK     |
| LLM Gateway | https://llm.stage.lexe.pro   | OK     |

---

## Deploy Commands

```bash
# STAGING (91.99.229.111)
ssh -i ~/.ssh/id_stage_new root@91.99.229.111
cd /opt/lexe-platform/lexe-infra
git checkout stage && git pull origin stage
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d

# PRODUCTION (49.12.85.92)
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml up -d
```

> **CRITICO:** L'override file e' OBBLIGATORIO. Senza override, Traefik labels mancano e Logto/lexe-core puntano all'ambiente sbagliato.

---

## Architecture: Orchestration v2

3-layer architecture with clean separation of concerns:

```
Orchestration Layer    contracts, events, strategy, orchestrator, decomposer
        |
   DomainEvents
        |
Gateway Layer          adapters (ToolKit, LLMClient, PipelineRunner), SSE translation
        |
   Tool/LLM calls
        |
Agent Layer            intent_detector, planner, research_engine (5 agents), verifier, synthesizer, scoring, fusion
```

**Strategies (v2 — binary routing):**

| Strategy | Intents              | Description                                  |
|----------|----------------------|----------------------------------------------|
| toolloop | CHAT (non-legal)     | Direct LLM with optional tools               |
| legal    | All legal scenarios  | Unified legal pipeline (depth-based budget)   |

**DepthLevel Budget:**

| Depth    | Tool Calls | Waves | Timeout | Planner       | Verifier | Self-Correct |
|----------|------------|-------|---------|---------------|----------|--------------|
| MINIMAL  | 8          | 1     | 20s     | deterministic | no       | no           |
| STANDARD | 20         | 2     | 35s     | lexe-fast     | yes      | no           |
| DEEP     | 35         | 3     | 50s     | lexe-frontier | yes      | yes          |

**Deprecated (files retained, disconnected from routing):**
- `strategies/legis.py`, `strategies/lexorc.py`, `strategies/super_tool.py`
- `agent/super_tool.py` + related files

See [agentic-workflow.md](agentic-workflow.md) for the full agentic architecture documentation.

---

## Database Schema

### lexe-postgres (core)

| Schema   | Tabelle                                                                                                                                                                                                                                                                                                                                                                                                                                            | Note                 |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `core`   | tenants, contacts, conversations, conversation_messages, conversation_events, message_evaluations, responder_personas, users, roles, permissions, role_permissions, tools, tenant_feature_flags, contact_groups, contact_group_members, contact_merges, channel_accounts, channel_policies, channel_routings, group_routings, service_conversations, verified_identifiers, pending_verifications, webhook_endpoints, audit_log, tenant_llm_models, email_verification_tokens, token_sessions, lexorc_sessions, super_tool_runs, model_groups, model_role_defaults, attachments | Multi-tenant con RLS, migrations 001-035 |
| `memory` | memories, semantic_vectors, episodic_vectors, working_memories, audit_log, profiles, profile_facts, fact_history, delta_tracking, semantic_vectors_v2 | pgvector 1536d, RLS 8 tabelle |

### lexe-max (KB Legal)

| Schema | Tabelle                                                                                 | Note             |
| ------ | --------------------------------------------------------------------------------------- | ---------------- |
| `kb`   | normativa (13,397), normativa_chunk (17,343), annotation, work (95), embeddings, massime (46,767) | Asset principale |

---

## Completed Work

### Phase 0-4: Separazione da LEO

- [x] Backup completo
- [x] DNS configurato (prod + staging)
- [x] LEXE Logto su auth.lexe.pro (prod) e auth.stage.lexe.pro (staging)
- [x] LEXE LiteLLM su llm.lexe.pro
- [x] Containers LEXE separati e healthy
- [x] LEO references cleanup (whatsapp.py, aria_router.py, etc.)

### WS5: Legal Tools Integration (2026-02-12)

- [x] lexe-tools-it: Normattiva, EUR-Lex, InfoLex API tools
- [x] Tool executor con URL nei risultati LLM
- [x] System prompt persona con "Riferimenti Normativi" obbligatori in calce
- [x] E2E test: tool calls + LLM con persona prompt
- [x] EUR-Lex: URL incluso nel response dict

### WS6: Database Migrations (2026-02-13)

- [x] **Migration 001**: RLS policies (fn_get_tenant_id, fn_is_superadmin), audit_log, tenant_llm_models, conversation_messages.tenant_id
- [x] **Migration 002**: 13 tabelle mancanti create (RBAC, channel, verification, etc.)
- [x] **Migration 003**: responder_personas migrato da organization_id (VARCHAR) a tenant_id (UUID FK)
- [x] customer_router.py aggiornato: get_persona_prompt() usa JOIN con tenants
- [x] save_message() aggiornato: passa tenant_id per conversation_messages NOT NULL

### WS7: Staging Infrastructure Fix (2026-02-13)

- [x] docker-compose.override.stage.yml: extra_hosts per auth.stage.lexe.pro
- [x] Logto admin router corretto: admin.stage.lexe.pro (match DNS)
- [x] .env.stage allineato con override (auth.stage.lexe.pro)
- [x] Deploy con override: tutti i servizi healthy
- [x] Traefik routers: webchat, API, auth, admin, litellm tutti enabled
- [x] Auth flow E2E funzionante su staging

### WS8: UX Fixes (2026-02-13)

- [x] Follow-up suggestions: prompt corretto per generare domande dal POV utente
- [x] Indagine Normattiva permalink: confermato che deep-link a singolo articolo NON e' supportato

### Admin Panel (2026-02-13 -- 2026-02-23)

- [x] 33 endpoints REST montati su /api/v1/admin
- [x] RBAC: 6 ruoli (superadmin, admin, agent, operator, user, viewer)
- [x] Audit log con tenant isolation
- [x] FF_STRICT_TENANT_NO_FALLBACK=true
- [x] Frontend React SPA (lexe-admin)
- [x] Dashboard con 10 sezioni, 27 componenti
- [x] CRUD Personas, gestione utenti/ruoli
- [x] Deployed to prod as lexe-admin container

### LLM Benchmark Phase 1 (2026-02-23)

- [x] Benchmark: 6 run (FAST x 3, COMPLEX x 3), 11 modelli, 2 judge, peer-review
- [x] Winner: `gemini-3-flash-preview` per fast/primary/complex roles
- [x] `reasoning_effort` config applicato nel catalogo
- [x] Runtime verificato: `config_utils.py:apply_config_override()` mappa reasoning_effort nel payload LiteLLM

### Auto-Improve Phase 2: Export + Eval (2026-02-23)

- [x] EventSink `_persist_event()` wired per 21 SSE event points
- [x] Export router: GET `/conversations/{id}/export` (json, md, html)
- [x] HTML renderer: standalone branded HTML
- [x] Trace router: GET `/admin/trace/{hash}` with timeline
- [x] Frontend: EvaluationStars, trace badge, evaluation API

### Auto-Improve Phase 3: Dashboards (2026-02-23)

- [x] Prometheus scraping `lexe-core:8100`, 5 alert rules
- [x] Grafana dashboard uid `lexe-operations`, 8 panels
- [x] Assessment endpoint: GET `/admin/assessment?period=1d|7d|30d`
- [x] SLA persistence: hybrid in-memory + DB (cached 60s)

### Orchestration v2 (2026-03-12)

- [x] **Phase 0**: 14 modules, 6 endpoints in lexe-tools-it/capabilities/
- [x] **Phase 1**: 7 modules extracted from customer_router.py (3357 to 2739 LOC)
- [x] **Phase 2a-2c**: orchestration/ package (contracts, events, strategy, decomposer, orchestrator, 4 bridge strategies)
- [x] **Phase 2d**: gateway/adapters.py, wire TurnContext, ff_orchestration_v2
- [x] **Phase 3a**: Native ToolloopStrategy (ToolKit/LLMClient protocols, circuit breaker)
- [x] **Phase 3b+3c**: Native Legis/SuperTool/Lexorc (PipelineRunner protocol, SSE parser)
- [x] **Phase 3d**: Per-capability degradation + RawSSEEvent deprecated, 71/71 tests passing

### Sprint 9: Multi-Agent + Citations + Admin UI (2026-03-12/13)

- [x] Orchestration v2 Phase 3a-3d: Native strategies, per-capability degradation, 71 tests
- [x] Multi-Agent Research "La Bomba": 5 agents, 3-wave parallel, evidence fusion
- [x] Confidence scoring: 3-band (45/25), removed jurisprudence penalty, Prometheus metrics
- [x] Citation badges: interactive [N] with tooltip popovers, click→highlight, BUG FIX #185
- [x] Tenant resolver: sub-based fallback, auto-heal org, TTL cache
- [x] Migrations 030-031: Tools AI roles + embedding alias fix
- [x] Admin Sprint 5: unified "Modelli & Alias", RuoliTab accordion, Tools AI globalOnly, Brand Kit v2.2, i18n
- [x] Quality metrics: Prometheus histograms (evidence, confidence, tools, pipeline duration)

### lexe-memory Sprints 1-4 (2026-02-14 -- 2026-02-16)

- [x] Router mounting + security + rate limiting
- [x] Delta writeback L0/L1 with anti-feedback loop
- [x] Brainprint profile (extraction, dedup, evolution)
- [x] DB idempotency + LLM summaries + RLS (migrations 004-007)
- [x] 82 total tests passing

---

## In Progress

### Pipeline v2 Staging Validation
- [x] Deploy pipeline v2 changes to staging (ff_orchestration_v2 enabled globally)
- [ ] E2E test: MINIMAL/STANDARD/DEEP depth routing
- [ ] E2E test: NDA_TRIAGE, COMPLIANCE_CHECK, RISK_ASSESSMENT scenarios
- [x] Validate authority scoring with real massime (massimario multi-format search deployed)

### Evidence URL Persistence (Planned)
- [ ] Migration: add `source_url TEXT` to `kb.massime` and `kb.sentenza`
- [ ] LLM-validated URL write-back to KB (resolve → HTTP HEAD → LLM verify → persist)
- [ ] Read-first: skip SearXNG if KB already has source_url
- [ ] Same flow for giurisprudenza (case_law)

### Pending Items
- [ ] Chat persistence in Logto mode
- [ ] Admin panel: degradation policy config UI
- [ ] Admin panel: ApiHealthPage, AuditPage, Command Palette

---

## Known Issues

| Issue                                   | Status | Note                                                                                                                          |
| --------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------- |
| Normattiva deep-link articolo           | Fixed  | Serviva suffisso annex `:2` per codici (CC, CP, CPC, CPP, C.Nav). Senza `:2`, `~artN` puntava all'art del decreto contenitore |
| Jaeger traces export                    | Minor  | `shared-jaeger` non attivo su staging, warning nei log ma nessun impatto funzionale                                           |
| Let's Encrypt rate limit                | Minor  | Alcuni domini legacy (play.lexe.pro, leo.itconsultingsrl.com) in rate limit                                                   |
| 404 su /api/v1/identity/access-requests | Open   | Endpoint LEO-specific, rimuovere da webchat                                                                                   |
| 500 su /v2/memory/context               | Fixed  | set_rls_context_v2() e semantic_vectors_v2 mancanti. Migration 007 crea entrambi                                              |
| CitationBadge infinite re-render #185   | Fixed  | `selectCitationMap` crea nuovo Map ad ogni render → loop infinito. Fix: inline selector `s.evidenceItems[index-1]`            |
| JWT missing org_id → 401                | Fixed  | Sub-based fallback lookup in tenant_resolver.py + auto-heal org membership                                                    |
| Low confidence when jurisprudence absent| Fixed  | Removed penalty — confidence = norm verification only, jurisprudence is informational enrichment                                |
| Chat lost after crash in Logto mode     | Open   | Frontend Logto mode skips loading messages from server. Need GET /conversations/{id}/messages endpoint                         |

---

## Normattiva Permalink - Deep Link Articoli

### Regola fondamentale: Codici vs Atti normali

I **codici** (CC, CP, CPC, CPP, C.Nav.) sono allegati a decreti. L'URN deve includere il suffisso annex `:2` per raggiungere gli articoli del codice:

```
# CODICI: servono :2 prima di ~art (articoli nell'allegato del decreto)
# Senza :2, ~artN punta all'articolo del decreto contenitore (sempre art. 1 di approvazione)

# Codice Civile, Art. 2043 (vigente)
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:regio.decreto:1942-03-16;262:2~art2043!vig=

# Codice Penale, Art. 575 (vigente)
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:regio.decreto:1930-10-19;1398:2~art575!vig=

# ATTI NORMALI: ~art funziona direttamente (no annex)
# D.Lgs. 152/2006, Art. 74
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:decreto.legislativo:2006-04-03;152~art74!vig=

# Vigente a data specifica
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:decreto.legge:2008-11-10;180~art2!vig=2009-11-10
```

### Suffissi annex per codice

| Codice              | Decreto         | Annex                   |
| ------------------- | --------------- | ----------------------- |
| Codice Civile       | R.D. 262/1942   | `:2`                    |
| Codice Penale       | R.D. 1398/1930  | `:2`                    |
| Codice Proc. Civile | R.D. 1443/1940  | `:2`                    |
| Codice Proc. Penale | D.P.R. 447/1988 | `:2`                    |
| Codice Navigazione  | R.D. 327/1942   | `:2`                    |
| Costituzione        | -               | nessuno (atto autonomo) |

Il tool (`normattiva.py`) usa `CODICI_PREDEFINITI[code]["urn_annex"]` per inserire automaticamente il suffisso corretto.

---

## Architettura Auth

### Logto OIDC Flow

```
Browser -> stage-chat.lexe.pro -> Logto (auth.stage.lexe.pro)
  -> JWT con organization_id claim
  -> lexe-core valida: issuer + audience (api.stage.lexe.pro)
  -> risolve tenant via core.tenants.logto_organization_id
```

### Staging Logto Config

| Parametro    | Valore                               |
| ------------ | ------------------------------------ |
| App ID       | dbj6ca04zxzwz7m61aww3                |
| App Name     | LEXE Webchat STAGE                   |
| Redirect URI | https://stage-chat.lexe.pro/callback |
| API Resource | https://api.stage.lexe.pro           |
| Organization | lexe-default-stage                   |
| Tenant UUID  | 67a08dc0-4677-423d-b962-f890c6d6b5a9 |

---

## Staging Deploy History

| Date       | Repos                                        | Migrations Applied    | Notes                                           |
|------------|----------------------------------------------|-----------------------|-------------------------------------------------|
| 2026-03-15 | lexe-core, lexe-docs                         | 032-035 (core)        | Massimario multi-search, attachments, docs      |
| 2026-03-13 | lexe-core, webchat, admin, infra, tools      | 030-031 (core)        | Sprint 9: Orch v2, multi-agent, citations       |
| 2026-02-23 | All 7 repos                                  | 018-025 (core)        | First full prod deploy, admin panel, LLM bench  |

---

*Ultimo aggiornamento: 2026-03-15*
