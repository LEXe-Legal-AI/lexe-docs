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

# System Boundaries -- What LEXE Owns vs External Dependencies

> Single reference for every system that participates in the LEXE platform,
> who owns it, what trust and SLA dependency LEXE places on it, and the
> blast radius when it fails.

---

## Scope

This document covers:

- Every container, service, or external system that LEXE depends on at runtime.
- Data flow direction between systems (who calls whom, who stores what).
- Trust classification: OWNED, MANAGED-OSS (self-hosted open-source), EXTERNAL-API, EXTERNAL-OSS-PROXY.
- SLA dependency: whether LEXE can continue serving users if the dependency is down.
- Failure impact: what breaks and what degrades gracefully.

## Non-Scope

- Internal code architecture (see `ARCHITECTURE-V2.md`, `PIPELINE-REFERENCE.md`).
- GDPR data flows (see `DATA-FLOWS-GDPR.md`).
- Deployment procedures (see `RUNBOOK-V2.md`).
- Confidence scoring internals (see `CONFIDENCE-PIPELINE.md`).
- Individual API endpoint contracts.

---

## Trust Levels

| Level | Definition |
|-------|-----------|
| **OWNED** | Source code in lexe-genesis monorepo. Full control over code, config, data. |
| **MANAGED-OSS** | Open-source software we self-host on our infrastructure. We control config and data, but depend on upstream for patches. |
| **EXTERNAL-API** | Third-party API we call over the network. No control over availability or data retention on their side. |
| **EXTERNAL-OSS-PROXY** | OSS tool that proxies requests to external services (e.g., LiteLLM proxying to Google/OpenAI). |

---

## System Inventory

### OWNED Systems

| System | Container | Port | What It Does for LEXE | Data Flow | SLA Dependency | Failure Impact |
|--------|-----------|------|----------------------|-----------|----------------|----------------|
| **lexe-core** | `lexe-core` | 8100 | API Gateway, orchestration, auth middleware, pipeline, admin API, SSE streaming, limits enforcement | Receives all client requests; calls lexe-memory, lexe-tools, LiteLLM, Logto, Valkey | **CRITICAL** -- platform down | All API, chat, admin unavailable |
| **lexe-webchat** | `lexe-webchat` | 3013 | Customer-facing chat SPA (React). ConsentOverlay, CitationBadges, PrivacyTab | Browser -> Traefik -> lexe-core (API), Logto (auth) | **HIGH** -- chat unavailable | Users cannot access chat; admin panel unaffected |
| **lexe-admin** | `lexe-admin` | 3014 | Admin panel SPA (React). Tenant management, model config, personas, audit | Browser -> Traefik -> lexe-core (admin API) | **MEDIUM** -- admin ops blocked | Operational management blocked; user-facing chat unaffected |
| **lexe-memory** | `lexe-memory` | 8103 | Memory L0-L4, RAG context, brainprint profiles, semantic search, delta tracking | lexe-core -> lexe-memory (HTTP); lexe-memory -> lexe-postgres (asyncpg) | **MEDIUM** -- degraded personalization | Chat works without memory context; responses less personalized |
| **lexe-tools-it** | `lexe-tools` | 8020 | Legal tools: Normattiva scraper, EUR-Lex client, InfoLex, kb_search, vigenza checker, one_search, lex_enricher | lexe-core -> lexe-tools (HTTP tool calls); lexe-tools -> Normattiva, EUR-Lex, SearXNG, lexe-max | **HIGH** -- legal research degraded | Legal queries return incomplete evidence; chat CHAT intents unaffected |
| **lexe-max** | `lexe-max` | 5436 | Legal KB: PostgreSQL + pgvector + Apache AGE. Normativa articles, chunks, embeddings, massime, graph edges, concept hierarchy | lexe-tools -> lexe-max (SQL); lexe-core -> lexe-max (for KB admin) | **HIGH** -- KB lookup fails | kb_search and kb_normativa_search return empty; Normattiva API still works |
| **lexe-privacy** | library (no container) | -- | PII detection via spaCy + custom engines. Used for pseudonymization pipeline | Imported by lexe-core at runtime | **LOW** -- pseudonymization skipped | Pseudonymized training data not available; no user-facing impact |
| **lexe-postgres** | `lexe-postgres` | 5435 | System database: Logto tables, LiteLLM tables, core schema (tenants, conversations, messages, RBAC, consent, events), memory schema | lexe-core, lexe-memory, lexe-logto, lexe-litellm -> lexe-postgres | **CRITICAL** -- platform down | Total outage: auth, chat, admin, memory all fail |
| **lexe-valkey** | `lexe-valkey` | 6379 | Cache and session store: tenant limits cache (TTL 5min), daily counters (conv/tokens), L0/L1 memory cache | lexe-core -> lexe-valkey; lexe-memory -> lexe-valkey | **MEDIUM** -- performance degraded | Limits enforcement falls back to DB; L0 cache misses; slight latency increase |

### MANAGED-OSS Systems

| System | Owner (OSS) | Container | Port | What It Does for LEXE | Data Flow | SLA Dependency | Failure Impact |
|--------|------------|-----------|------|----------------------|-----------|----------------|----------------|
| **Logto** | Logto (OSS) | `lexe-logto` | 3001/3002 | OAuth2/OIDC identity provider. User registration, login, JWT issuance, organization management, RBAC | Browser -> Logto (auth flow); lexe-core -> Logto Management API (user provisioning, org auto-heal) | **CRITICAL** -- no auth | Users cannot login; new user provisioning fails; existing sessions with valid JWT continue until expiry |
| **LiteLLM** | BerriAI (OSS) | `lexe-litellm` | 4000 | LLM proxy: model abstraction, spend tracking, rate limiting, key management. Routes to Google Gemini, OpenAI, Anthropic, etc. | lexe-core -> LiteLLM (HTTP); LiteLLM -> external LLM providers (HTTPS) | **CRITICAL** -- no LLM inference | All LLM-dependent features fail: synthesis, verification, planning, intent detection. Chat returns error. |
| **Traefik** | Traefik Labs (OSS) | `shared-traefik` | 80/443 | Reverse proxy, TLS termination (Let's Encrypt), routing. Shared with LEO platform on `leo_public` network | All external HTTPS -> Traefik -> backend containers | **CRITICAL** -- platform unreachable | No external access to any LEXE service. Internal container-to-container unaffected. |
| **Prometheus** | CNCF (OSS) | `shared-prometheus` | 9090 | Metrics collection. Scrapes `lexe-core:8100/metrics`. Alert rules: down, error rate, latency, SSE, event sink | Prometheus -> lexe-core (scrape); Grafana -> Prometheus (query) | **LOW** -- monitoring blind | No metrics, no alerts. Platform continues operating normally. |
| **Grafana** | Grafana Labs (OSS) | `shared-grafana` | 3000 | Dashboards and alerting UI. Dashboard uid `lexe-operations` (8 panels) | Grafana -> Prometheus (datasource) | **LOW** -- dashboards unavailable | No visual monitoring. Prometheus continues collecting. |
| **Temporal** | Temporal (OSS) | `lexe-temporal` | 7233 | Workflow orchestration engine. Currently **disabled** (FF `ff_orchestrator_enabled=False`) | Not active. Future: lexe-core -> Temporal (workflow dispatch) | **NONE** (disabled) | No impact. Containers healthy but unused. |
| **SearXNG** | SearXNG (OSS) | `shared-searxng` | 8888 | Privacy-preserving metasearch. Used by `lex_search` and massimario URL resolution | lexe-tools -> SearXNG (HTTP); SearXNG -> external search engines | **LOW** -- URL resolution degrades | Massime URLs return empty (negative cached 24h). Web search tool unavailable. KB search unaffected. |
| **Langfuse** | Langfuse (OSS) | optional | -- | LLM observability: trace logging, evaluation scores. Optional; graceful degradation if unconfigured | lexe-core -> Langfuse (HTTP async) | **NONE** (optional) | No trace logging. Platform unaffected. |

### EXTERNAL-API Systems

| System | Owner | What It Does for LEXE | Data Flow Direction | Data Sent to External | SLA Dependency | Failure Impact |
|--------|-------|----------------------|--------------------|-----------------------|----------------|----------------|
| **Normattiva** | Italian Government | Official Italian legislation API. Article text, vigenza status, URN resolution | lexe-tools -> normattiva.it (HTTPS) | Search queries (legal references only, no user data) | **MEDIUM** -- normativa tool fails | `normattiva_search` returns error; KB data still available via `kb_normativa_search`; vigenza checks fail |
| **EUR-Lex** | European Union | EU legislation: directives, regulations, CELEX lookup | lexe-tools -> eur-lex.europa.eu (HTTPS) | Search queries (legal references only, no user data) | **LOW** -- EU law lookup fails | `eurlex_search` returns empty; Italian law queries unaffected |
| **Brocardi.it** | Third-party | Italian legal commentary and article cross-references | lexe-tools -> brocardi.it (HTTPS scraping) | URL requests (article identifiers only) | **LOW** -- commentary unavailable | InfoLex enrichment reduced; core legal research unaffected |
| **Google Gemini API** | Alphabet/Google | Primary LLM provider (Gemini 3 Flash Preview for all 6 aliases) | LiteLLM -> generativelanguage.googleapis.com (HTTPS) | Conversation context, system prompts, tool results (via streaming, no persistence on their side per DPA) | **CRITICAL** -- no LLM inference | All synthesis, verification, planning fails. Fallback: switch to alternative model in LiteLLM config |
| **OpenAI API** | OpenAI | Secondary LLM provider (gpt-5-mini) + embeddings (text-embedding-3-small) | LiteLLM -> api.openai.com (HTTPS) | Embedding input text; conversation context for gpt-5-mini | **HIGH** for embeddings, **LOW** for gpt-5-mini | Embedding generation fails (KB ingestion blocked); gpt-5-mini is secondary model only |
| **Hetzner** | Hetzner Online GmbH | Infrastructure: VPS hosting (prod 49.12.85.92, staging 91.99.229.111). EU datacenter (Falkenstein/Nuremberg) | All containers run on Hetzner VPS | All platform data resides on Hetzner servers | **CRITICAL** -- total outage | Complete platform unavailability. DR: snapshots + DB backups. |

---

## Network Topology

```
Internet (HTTPS 443)
    |
    v
shared-traefik (leo_public network)
    |
    +---> lexe-webchat (3013)       [leo_public]
    +---> lexe-admin (3014)         [leo_public]
    +---> lexe-core (8100)          [leo_public + lexe_internal]
    +---> lexe-logto (3001/3002)    [leo_public + lexe_internal]
    +---> lexe-litellm (4000)       [leo_public + lexe_internal]
    |
    v
lexe_internal network (backend only, no external access)
    |
    +---> lexe-postgres (5435)
    +---> lexe-max (5436)
    +---> lexe-valkey (6379)
    +---> lexe-memory (8103)
    +---> lexe-tools (8020)
    +---> lexe-temporal (7233)
```

**Key network rules:**
- Databases (`lexe-postgres`, `lexe-max`, `lexe-valkey`) are on `lexe_internal` only -- never exposed to `leo_public`.
- `shared-traefik` is shared with the LEO platform on the same Hetzner VPS.
- Inter-container communication uses Docker DNS (container names as hostnames).

---

## Data Flow Summary

```
User Browser
    |
    | HTTPS (JWT Bearer)
    v
lexe-core -----> lexe-memory -----> lexe-postgres (memory schema)
    |                                     ^
    |                                     |
    +----------> lexe-tools ----------> lexe-max (kb schema)
    |                |
    |                +---> Normattiva (external, no user data)
    |                +---> EUR-Lex (external, no user data)
    |                +---> SearXNG (external, no user data)
    |                +---> Brocardi (external, no user data)
    |
    +----------> LiteLLM -----------> Google Gemini (streaming, no persistence)
    |                +---------------> OpenAI (embeddings + gpt-5-mini)
    |
    +----------> Logto (auth verification, user provisioning)
    |
    +----------> lexe-valkey (cache, counters)
    |
    +----------> lexe-postgres (core schema: tenants, conversations, messages, consent, events)
    |
    +----------> Prometheus (metrics scrape, outbound)
```

**Data sensitivity by flow:**

| Flow | Data Classification | Notes |
|------|-------------------|-------|
| User -> lexe-core | PII (email, name), legal queries | Encrypted in transit (TLS), tenant-isolated (RLS) |
| lexe-core -> LiteLLM -> LLM provider | Conversation content, system prompts | Streaming only, no persistence per DPA. No user identity sent. |
| lexe-core -> lexe-memory | User facts, preferences, conversation summaries | Stored in `memory` schema with RLS. L0 in Valkey (24h TTL). |
| lexe-tools -> external APIs | Legal references, search terms | No user PII. Only legal identifiers (URN, article numbers). |
| lexe-core -> Logto | User identity operations | Logto stores email, name, password hash. Self-hosted, EU. |
| lexe-core -> Prometheus | Anonymized metrics counters | No PII. Histograms: latency, confidence, evidence counts. |

---

## SLA Dependency Matrix

| Dependency | Availability Target | Degradation Mode | Recovery Action |
|-----------|-------------------|-----------------|-----------------|
| lexe-postgres | 99.9% | None (hard dependency) | Restart container; worst case: restore from backup |
| Logto | 99.9% | Existing sessions survive JWT TTL (~1h) | Restart `lexe-logto`; verify Logto DB in lexe-postgres |
| LiteLLM + LLM Provider | 99.5% | Circuit breaker per model; switch alias in admin | Change model alias in LiteLLM admin; restart if needed |
| lexe-tools + Normattiva | 99.0% | `kb_normativa_search` fallback (KB data) | Wait for Normattiva recovery; KB data serves cached content |
| lexe-valkey | 99.5% | Falls back to DB queries (slower) | Restart container; data is ephemeral (rebuilt from DB) |
| lexe-memory | 99.5% | Chat works without personalization | Restart; memory persists in lexe-postgres |
| Traefik | 99.9% | None (hard dependency for external access) | `docker restart shared-traefik`; check cert renewal |
| SearXNG | 95.0% | Massime URLs empty; negative cache 24h | Restart; URLs will resolve on next cache expiry |
| Prometheus/Grafana | 95.0% | No monitoring; platform runs fine | Restart; no data loss (TSDB on disk) |

---

## Failure Scenarios and Blast Radius

### Scenario 1: lexe-postgres down
- **Impact**: TOTAL OUTAGE. Auth, chat, admin, memory, LiteLLM (spend tracking) all fail.
- **Detection**: Prometheus alert `LexeDown` within 30s; health check fails.
- **Mitigation**: Container auto-restart (Docker restart policy). Manual: `docker restart lexe-postgres`.
- **Data risk**: None if container restarts cleanly. Volume-mounted data survives. NEVER `docker compose down -v`.

### Scenario 2: Google Gemini API outage
- **Impact**: All LLM inference fails. Users see SSE error events.
- **Detection**: LiteLLM logs 5xx from upstream; Prometheus `lexe_error_rate` spikes.
- **Mitigation**: Switch all model aliases to backup provider (gpt-5-mini, deepseek-v3.2) via LiteLLM admin.
- **Data risk**: None. Conversations saved; LLM responses not generated.

### Scenario 3: Normattiva API unreachable
- **Impact**: `normattiva_search` tool returns error. Vigenza checks fail.
- **Detection**: Tool call errors in pipeline; confidence scores drop (vigenza_skipped cap 55).
- **Mitigation**: `kb_normativa_search` provides cached legislative data from lexe-max. Results are less fresh.
- **Data risk**: None. External API, no LEXE data exposed.

### Scenario 4: Traefik down
- **Impact**: No external access. Internal container communication unaffected.
- **Detection**: External health checks fail; users report "site unreachable".
- **Mitigation**: `docker restart shared-traefik`. Check `leo_public` network attachment.
- **Data risk**: None. Traffic simply not routed.

### Scenario 5: lexe-valkey down
- **Impact**: Limits enforcement falls back to DB. L0 memory cache misses. Slight latency increase.
- **Detection**: Connection refused errors in logs. Limits checks slower.
- **Mitigation**: `docker restart lexe-valkey`. All data is ephemeral and rebuilt.
- **Data risk**: None. Counters rebuilt from DB on next check.

---

## Shared Infrastructure with LEO

LEXE shares the following infrastructure with the LEO platform on the same Hetzner VPS:

| Shared Component | Network | Risk |
|-----------------|---------|------|
| `shared-traefik` | `leo_public` | Traefik config error can affect both platforms |
| `shared-prometheus` | `shared_public` | Prometheus overload can affect scraping of both |
| `shared-grafana` | `shared_public` | Minimal risk (dashboards only) |
| `shared-searxng` | `shared_public` | Rate limiting shared across both platforms |

**Isolation measures:**
- Separate Docker Compose projects (`lexe-infra` vs `leo-infra`).
- Separate databases (lexe-postgres vs leo-postgres).
- Separate Logto instances (auth.lexe.pro vs auth.leo.pro).
- Separate LiteLLM instances.
- Only Traefik, Prometheus, Grafana, and SearXNG are shared.

---

*Ultimo aggiornamento: 2026-03-19*
