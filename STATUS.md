# LEXE Platform - Status

> Stato attuale della piattaforma LEXE Legal AI

---

## Infrastructure Status (2026-02-13)

### Production (49.12.85.92)

| Container         | Status  | Note                                           |
| ----------------- | ------- | ---------------------------------------------- |
| lexe-core         | Healthy | Gateway + Auth OIDC + Admin API                |
| lexe-orchestrator | Healthy | ORCHIDEA Pipeline (FF disabled)                |
| lexe-memory       | Healthy | L1-L3 Memory                                   |
| lexe-webchat      | Healthy | Frontend chat                                  |
| lexe-logto        | Healthy | CIAM Auth                                      |
| lexe-litellm      | Healthy | LLM Gateway                                    |
| lexe-postgres     | Healthy | DB + pgvector                                  |
| lexe-valkey       | Healthy | Cache L1                                       |
| lexe-max          | Healthy | KB Legal (normativa, massime, embeddings)      |
| lexe-tools        | Healthy | Legal Tools API (Normattiva, EUR-Lex, InfoLex) |
| lexe-temporal     | Healthy | Workflow orchestration                         |
| lexe-temporal-ui  | Healthy | Temporal dashboard                             |

**Total:** 12 containers

### Staging (91.99.229.111)

| Container     | Status  | Note                           |
| ------------- | ------- | ------------------------------ |
| lexe-core     | Healthy | stage branch                   |
| lexe-webchat  | Healthy | Built from source via override |
| lexe-logto    | Healthy | ENDPOINT=auth.stage.lexe.pro   |
| lexe-litellm  | Healthy |                                |
| lexe-tools    | Healthy |                                |
| lexe-postgres | Healthy |                                |
| lexe-valkey   | Healthy |                                |
| lexe-max      | Healthy |                                |
| lexe-temporal | Healthy |                                |

---

## Services URLs

### Production

| Service     | URL                   | Status |
| ----------- | --------------------- | ------ |
| Webchat     | https://ai.lexe.pro   | OK     |
| API         | https://api.lexe.pro  | OK     |
| Auth OIDC   | https://auth.lexe.pro | OK     |
| LLM Gateway | https://llm.lexe.pro  | OK     |

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

> **CRITICO:** L'override file è OBBLIGATORIO. Senza override, Traefik labels mancano e Logto/lexe-core puntano all'ambiente sbagliato.

---

## Database Schema

### lexe-postgres (core)

| Schema   | Tabelle                                                                                                                                                                                                                                                                                                                                                                                                                                            | Note                 |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| `core`   | tenants, contacts, conversations, conversation_messages, responder_personas, users, roles, permissions, role_permissions, tools, tenant_feature_flags, contact_groups, contact_group_members, contact_merges, channel_accounts, channel_policies, channel_routings, group_routings, service_conversations, verified_identifiers, pending_verifications, webhook_endpoints, audit_log, tenant_llm_models, email_verification_tokens, token_sessions | Multi-tenant con RLS |
| `memory` | memories, semantic_vectors, episodic_vectors, working_memories, audit_log, profiles, profile_facts, fact_history, delta_tracking | pgvector 1536d, RLS 8 tabelle |

### lexe-max (KB Legal)

| Schema | Tabelle                                                                                 | Note             |
| ------ | --------------------------------------------------------------------------------------- | ---------------- |
| `kb`   | normativa (6,335), normativa_chunk (10,246), annotation (13,281), work (69), embeddings | Asset principale |

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
- [x] Indagine Normattiva permalink: confermato che deep-link a singolo articolo NON è supportato

### Admin Panel Backend (2026-02-13)

- [x] 33 endpoints REST montati su /api/v1/admin
- [x] RBAC: 6 ruoli (superadmin, admin, agent, operator, user, viewer)
- [x] Audit log con tenant isolation
- [x] FF_STRICT_TENANT_NO_FALLBACK=true

---

## Architettura Auth

### Logto OIDC Flow

```
Browser → stage-chat.lexe.pro → Logto (auth.stage.lexe.pro)
  → JWT con organization_id claim
  → lexe-core valida: issuer + audience (api.stage.lexe.pro)
  → risolve tenant via core.tenants.logto_organization_id
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

### lexe-memory Sprint 1: Router Mounting + Security (2026-02-14)

- [x] Tutti i router registrati in main.py (delta, profile, context, rag, routes, gateway)
- [x] Rate limiting su tutti gli endpoint (100/min default, 20/min su store/search)
- [x] X-Tenant-ID header validation
- [x] try/except per import opzionali (profile, rag) — graceful degradation
- [x] `GET /openapi.json` espone tutti i 40+ endpoint
- [x] Health checks L0-L4 in /health response

### lexe-memory Sprint 2: Delta Writeback L0/L1 (2026-02-15)

- [x] `apply_delta()`: writeback reale su L0 (session) e L1 (persistent)
  - `sync_l0_only` + conversation_id → facts in L0
  - `sync_l0_only` senza conversation → facts in L1
  - `async_full` → facts in L0 **e** L1
  - Preferences → sempre L1
- [x] Anti-feedback loop: blocca contenuto negativo (20 pattern IT/EN/PT)
- [x] Idempotency tracking via request_id (OrderedDict bounded 10k)
- [x] `get_by_request_id()`: delta status da idempotency cache
- [x] `get_summary()`: template-based da L1+L2 (no LLM, Sprint 4)
- [x] `get_resolution_log()`: log ADD/DELETE con cursor pagination
- [x] Intelligence metrics: facts_analyzed, facts_blocked, deltas_tracked
- [x] 25 unit tests (mock MemoryService, no DB)

### lexe-memory Sprint 3: Brainprint Profile (2026-02-14)

- [x] Migration 004: `memory.profiles`, `memory.profile_facts`, `memory.fact_history`
  - Unique partial index `(profile_id, key) WHERE is_valid = true` per dedup
  - Trigger `set_updated_at()` automatico
  - `tenant_id` su tutte e 3 le tabelle (RLS-ready)
- [x] `profile/schemas.py`: 12 Pydantic v2 models (ProfileResponse, FactCandidate, ExtractionInput, ...)
- [x] `profile/service.py`: ProfileService con dedup key-based
  - get_profile (NO auto-create), get_or_create_profile, add_fact (confirm/supersede/keep)
  - update_fact, invalidate_fact, get_profile_summary (template), get_profile_history
- [x] `profile/extractor.py`: FactExtractionProvider protocol
  - HeuristicExtractor: 12 regex IT/EN/PT, confidence scoring, uncertainty penalty
  - LLMExtractor: via LiteLLM `lexe-fast`, JSON strict + whitelist, OFF by default
  - ALLOWED_KEYS whitelist per 5 categorie (identity, preference, relation, context, behavior)
- [x] `profile/evolver.py`: orchestra extraction → persistence via add_fact()
- [x] Config: 8 nuovi settings per fact extractor
- [x] 23 unit tests (HeuristicExtractor, schemas, factory)
- [x] E2E su staging: 404 (no auto-create) → 201 (add_fact) → confirm → supersede → history

### lexe-memory Sprint 4: DB Idempotency + LLM Summaries + RLS (2026-02-16)

- [x] Migration 005: `memory.delta_tracking` table
  - `UNIQUE (tenant_id, request_id)` per-tenant idempotency
  - Status enum: `in_progress` → `completed` | `failed`
  - `cleanup_old_deltas(retention_days)` function
  - 3 indexes (composite request, processed_at DESC, partial status)
- [x] Migration 006: RLS policies su 8 tabelle memory
  - `memory.fn_get_tenant_id()` + `memory.fn_is_superadmin()` helpers
  - `tenant_isolation` policy on: episodic_vectors, semantic_vectors, working_memories, audit_log, profiles, profile_facts, fact_history, delta_tracking
  - Defense-in-depth (lexe user is SUPERUSER → RLS bypassed)
- [x] DB-backed idempotency in DeltaService
  - `init_db()` → engine + session factory (lazy, first delta call)
  - `_warm_cache()`: 24h window + max 50K entries (most restrictive wins)
  - Flow: cache check → DB insert `in_progress` → process → `completed`/`failed`
  - `get_by_request_id(request_id, tenant_id)`: cache → DB fallback
  - Survives container restarts (warm cache from DB)
- [x] LLM Summary module (`llm_summary.py`)
  - `_call_litellm()`: POST to LiteLLM gateway via httpx
  - `generate_memory_summary_llm()` + `generate_profile_summary_llm()`
  - Guard rails: 4000 char input max, prose-only output (no bullets)
  - Feature flag: `FF_LLM_SUMMARY` (default false)
  - Latency logging: `method=llm/template latency_ms=X`
- [x] LLM summary integration in DeltaService.get_summary() and ProfileService.get_profile_summary() with template fallback
- [x] Reason-code logging: `L0 skipped: no conversation_id`, `L1 skipped: sync_l0_only mode`
- [x] Config: 5 new settings (ff_llm_summary, llm_summary_model/max_tokens/timeout, litellm_api_key)
- [x] Summary endpoint fix: `tenant_id` query param for scoped L1/L2 retrieval
- [x] 26 new tests (82 total, all passing)
- [x] E2E verified on staging: idempotency, warm cache after restart, reason-code logs, summary

---

## In Progress

### Admin Panel Frontend (Task #6)

- [x] Scaffolding React (in lexe-webchat, non repo separato)
- [x] Dashboard con 10 sezioni, 27 componenti
- [x] CRUD Personas
- [x] Gestione utenti/ruoli
- [x] Audit log viewer
- [ ] Configurazione modelli LLM per tenant

---

## Known Issues

| Issue                                   | Status | Note                                                                                                                          |
| --------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------- |
| Normattiva deep-link articolo           | Fixed  | Serviva suffisso annex `:2` per codici (CC, CP, CPC, CPP, C.Nav). Senza `:2`, `~artN` puntava all'art del decreto contenitore |
| Jaeger traces export                    | Minor  | `shared-jaeger` non attivo su staging, warning nei log ma nessun impatto funzionale                                           |
| Let's Encrypt rate limit                | Minor  | Alcuni domini legacy (play.lexe.pro, leo.itconsultingsrl.com) in rate limit                                                   |
| 404 su /api/v1/identity/access-requests | Open   | Endpoint LEO-specific, rimuovere da webchat                                                                                   |

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

*Ultimo aggiornamento: 2026-02-16*
