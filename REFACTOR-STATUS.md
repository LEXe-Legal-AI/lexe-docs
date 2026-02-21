# LEXE Refactor — Stato Completo e Prossimi Passi

> **Data**: 2026-02-20
> **Blueprint di riferimento**: `lexe-docs/architecture/LEXE-REFACTOR-BLUEPRINT.md`
> **Scope**: Routing automatico, Model selection, Tool governance, Prompt governance, Quality gates

---

## 1. COSA ESISTE OGGI (Pre-Refactor)

### 1.1 Pipeline attive

| Pipeline | Trigger | Complessita | Status |
|----------|---------|-------------|--------|
| **Studio Toolloop** | `mode=studio` (default) | Bassa | Prod |
| **LEGIS 4-phase** | `mode=legal` + intent detector | Media-Alta | Prod |
| **LEXORC 1+5 agents** | `mode=legal` + intent `complex` + `ff_lexorc_enabled` | Alta | Staging |

### 1.2 Flusso request attuale

```
Request → Logto JWT → Tenant (cache 5min) → Contact (JIT)
  → Conversation Mode Lock (studio|legal, locked at 1st msg)
  → Mode Preset (tools ON/OFF, memory ON/OFF, context window)
  → Model Role Resolution (tenant override → global default → hardcoded)
  → LiteLLM Virtual Key (per-tenant isolation → master fallback)
  → Pipeline Router:
      ├─ studio → Studio Toolloop
      └─ legal → Intent Detector (LLM) → DIRECT|SIMPLE|STANDARD|COMPLEX
            ├─ DIRECT/SIMPLE → LEGIS light (fast synth)
            ├─ STANDARD → LEGIS full (Plan→Research→Verify→Synth)
            └─ COMPLEX + ff_lexorc → LEXORC (Classifier→5 parallel agents)
  → Memory Retrieve (v2: L0-L3 + profile) [solo Studio]
  → [Pipeline Execution] → SSE streaming
  → Memory Store (delta + profile evolve) [solo Studio]
  → Follow-up Generation (3 domande)
  → done_event
```

### 1.3 Database (13 migrazioni, 23+ tabelle)

**lexe-postgres** (sistema):
- `core.tenants` — multi-tenant, plan non ancora presente
- `core.conversation_messages` — con `tenant_id NOT NULL` (mig. 001)
- `core.tenant_model_roles`, `core.model_role_defaults` — 3-tier model resolution
- `core.llm_model_catalog` — 18 modelli, `config JSONB` (mig. 010)
- `core.tool_catalog`, `core.persona_catalog` — cataloghi globali (mig. 007)
- `core.tool_execution_logs` — logging tool con latency_ms (mig. 006)
- `core.evidence_packs`, `core.verification_logs` — LEGIS/LEXORC audit
- `core.audit_log`, `core.rbac_role_assignments` — RBAC (mig. 002-003)

**lexe-max** (KB legal):
- `kb.normativa` — 4,117 articoli (CC=3170, CP=947)
- `kb.massime` — 46,767 (38,718 active)
- `kb.graph_edges` — 58,737 relazioni CITES
- `kb.embeddings` — 41,437 vettori

### 1.4 Modelli LLM (LiteLLM config)

| Alias | Provider | Modello | Uso |
|-------|----------|---------|-----|
| `lexe-primary` | OpenRouter | `mistral-large-2512` | Chat, synthesis |
| `lexe-complex` | OpenRouter | `gpt-5.2` | Complex planning |
| `lexe-fast` | Google | `gemini-2.5-flash` | Classification, follow-ups |
| `legal-tier2-haiku` | Anthropic | `claude-haiku-4.5` | Verification |
| `lexe-embeddings` | OpenAI | `text-embedding-3-large` | Embeddings |
| + 4 aliases extra | vari | vari | routing specializzato |

### 1.5 Tool disponibili (8)

| Tool | Purity | Fonte | Uso |
|------|--------|-------|-----|
| `normattiva_search` | 1 (pura) | Normattiva.it | Testo vigente articoli IT |
| `eurlex_search` | 1 | EUR-Lex | Normativa UE |
| `infolex_search` | 2 | KB LEXe | Dottrina + giurisprudenza |
| `kb_search` | 2 | Massimario | 38K massime, hybrid search |
| `kb_normativa_search` | 2 | KB normativa | 5K articoli, semantic search |
| `lex_search` | 3 | SearXNG (13 siti) | Ricerca web giuridica |
| `lex_search_enriched` | 4 | SearXNG + LLM | Ricerca analizzata |
| `web_search` | 5 | SearXNG (general) | Ricerca web generica |

### 1.6 Debito tecnico principale

| Area | Problema | Impatto |
|------|----------|---------|
| **Budget** | `litellm_budget_monthly` esiste ma NON enforced runtime | Zero cost control |
| **Prompt** | ~20 prompt inline in 6 file Python, no versioning | No A/B test, no audit |
| **Model swap** | Nessun quality gate pre-deploy | Risk di regressione |
| **Tool limits** | Max 20 calls/turn globale, no per-tool rate limit | Tool abuse |
| **Eval** | No gold set, no automated regression | No CI quality gate |
| **Fallback** | Implicito (hardcoded), no retry chain | Single point of failure |

---

## 2. REFACTOR BLUEPRINT — Le 6 Fasi

### Fase F.1: Prompt Extraction (IN CORSO)

**Obiettivo**: Estrarre tutti i prompt inline in `prompts/v1/`, zero behavior change.

**Stato**: 11 file creati, mancano registry + rewiring imports.

| File creato | Prompt estratti | Fonte originale |
|-------------|----------------|-----------------|
| `prompts/v1/studio_system.py` | `STUDIO_SYSTEM_PROMPT_V1` | `customer_router.py:183-205` |
| `prompts/v1/intent_detector.py` | `INTENT_DETECTOR_SYSTEM_PROMPT_V1` + `_USER_TEMPLATE_V1` | `agent/prompts.py:14-77` |
| `prompts/v1/legis_planner.py` | `PLANNER_SYSTEM_PROMPT_V1` + `_USER_TEMPLATE_V1` | `agent/prompts.py:334-460` |
| `prompts/v1/legis_researcher.py` | `RESEARCHER_SYSTEM_PROMPT_V1` | `agent/prompts.py:467-482` |
| `prompts/v1/legis_verifier.py` | `VERIFIER_SYSTEM_PROMPT_V1` | `agent/prompts.py:489-527` |
| `prompts/v1/legis_synthesizer.py` | `SYNTHESIZER_SYSTEM_PROMPT_V1` + 7 scenario + evidence template | `agent/prompts.py:84-607` |
| `prompts/v1/lexorc_classifier.py` | `CLASSIFIER_SYSTEM_PROMPT_V1` | `agent/classifier.py:26-44` |
| `prompts/v1/lexorc_synthesizer.py` | `LEXORC_SYNTHESIZER_SYSTEM_PROMPT_V1` | `agent/agents/lexorc_synthesizer.py:39-73` |
| `prompts/v1/lexorc_auditor.py` | `AUDITOR_SYSTEM_PROMPT_V1` + `AUDITOR_COHERENCE_PROMPT_V1` | `agent/agents/auditor.py:392-420` |
| `prompts/v1/fact_extractor.py` | `FACT_EXTRACTOR_SYSTEM_PROMPT_V1` + `_USER_TEMPLATE_V1` | `gateway/fact_extractor_llm.py:35-70` |
| `prompts/v1/follow_up.py` | `FOLLOW_UP_PROMPT_V1` | `customer_router.py:2293-2308` |

**Da completare**:
1. `prompts/registry.py` — PromptRegistry singleton (mappa id → prompt, supporta version swap)
2. Rewiring imports: `agent/prompts.py` importa da `prompts/v1/` e ri-esporta (backward compat)
3. Rewiring imports: `classifier.py`, `lexorc_synthesizer.py`, `auditor.py`, `fact_extractor_llm.py`, `customer_router.py`
4. py_compile check
5. Deploy staging + smoke test

---

### Fase F.2: Budget Enforcement (NEXT)

**Obiettivo**: Aggiungere colonna `plan` su tenants, budget gates, degradazione modelli.

**Cosa cambia**:
- `core.tenants` + colonna `plan VARCHAR(30)` con 4 valori: `studio`, `legal_standard`, `legal_deep`, `enterprise`
- `core.tenants` + colonna `budget_alert_threshold NUMERIC(5,2)` default 0.80
- Nuova tabella `core.budget_events` (log di warning/exceeded/degraded)
- `gateway/budget_gate.py` (NEW) — Gate 2 nel routing decision tree
- FF: `ff_budget_enforcement` (default false)

**Pipeline routing per plan**:

| Piano | Pipeline ammesse | Tool budget/turn | LLM budget/msg |
|-------|-----------------|------------------|----------------|
| Studio | Solo toolloop | 10 calls, 60s | $0.02 |
| Legal Standard | Toolloop + LEGIS | 15 calls, 90s | $0.08 |
| Legal Deep | Toolloop + LEGIS + LEXORC | 25 calls, 180s | $0.25 |
| Enterprise | All + custom | Configurable | Configurable |

**Budget pressure degradation**:
```
Normal       → tier3 (frontier: mistral-large, claude-sonnet)
>80% budget  → tier2 (mid: haiku, gemini-flash)
>95% budget  → tier1 (flash/free: gemini-flash)
>100% budget → 429 reject
```

---

### Fase F.3: Tool Governance (DOPO F.2)

**Obiettivo**: Rate limit per-tool, budget per turn, enhanced circuit breaker.

**Cosa cambia**:
- `tools/budget.py` (NEW) — ToolBudget dataclass per piano
- `tools/rate_limiter.py` (NEW) — per-tool invocation cap
- Circuit breaker: da per-source a per-tool + global CB
- FF: `ff_tool_budgets` (default false)

**Tool access matrix per piano**:

| Tool | Studio | Legal Std | Legal Deep |
|------|--------|-----------|------------|
| `normattiva_search` | Y | Y | Y |
| `eurlex_search` | Y | Y | Y |
| `lex_search_enriched` | **N** | Y | Y |
| `web_search` | opt-in | **N** | opt-in |

**Per-tool rate limits**:

| Tool | Per turn | Timeout | Retry |
|------|----------|---------|-------|
| `normattiva_search` | 8 | 30s | 1 |
| `kb_search` | 8 | 20s | 0 |
| `lex_search_enriched` | 4 | 30s | 0 |
| `web_search` | 3 | 15s | 1 |

---

### Fase F.4: Model Fallback Chain (DOPO F.3)

**Obiettivo**: Fallback DB-backed per ogni role, con retry e timeout per tier.

**Cosa cambia**:
- Nuova tabella `core.model_fallback_chain` (role, priority, model_name, max_retries, timeout)
- `agent/model_resolver.py` (NEW) — ModelPolicy.resolve()
- customer_router.py usa il resolver invece del 3-tier query attuale
- FF: `ff_model_fallback_chain` (default false)

**Fallback chain esempio (legis_planner)**:
```
priority 0: lexe-complex    (tier4, timeout 60s, 1 retry)
priority 1: lexe-primary    (tier3, timeout 45s, 1 retry)
priority 2: lexe-fast       (tier1, timeout 30s, 0 retry)
```

**Quality gates per model swap** (prima di promuovere un modello):
1. Format compliance > 98% su 50 golden prompts
2. Legal citation accuracy > 95% su 20 test
3. Zero fabricated citations
4. Latency P95 < role_timeout * 0.8
5. Cost gate: new model cost < current * 1.5

---

### Fase F.5: Eval Harness (PARALLELA a F.2-F.4)

**Obiettivo**: Gold set, automated CI gate, regression testing.

**Cosa cambia**:
- `tests/eval/gold_sets/` — 100+ test cases in JSONL
- `tests/eval/runners/` — format compliance, citation accuracy, vigenza, hallucination, latency
- CI gate: PR blocking se metric sotto soglia

**Gold set**:
- 30 Studio (domande dirette, lookup, definizioni)
- 30 LEGIS (ricerche cross-normativa, pareri, contratti)
- 10 LEXORC (analisi complesse multi-agente)
- 20 Citazioni (art. specifici con vigenza nota)
- 10 Vigenza (mix vigente/abrogato per test negativo)

**Acceptance criteria (blocking)**:

| Metrica | Minimo | Target |
|---------|--------|--------|
| Format compliance | > 95% | > 98% |
| Citation accuracy | > 90% | > 95% |
| Vigenza accuracy | > 95% | > 98% |
| Fabricated citations | **0** | **0** |
| Latency P95 Studio | < 8s | < 5s |
| Latency P95 LEGIS | < 30s | < 20s |
| Tool success rate | > 90% | > 95% |

---

### Fase F.6: Prompt Versioning + A/B (ULTIMA)

**Obiettivo**: Routing A/B tra versioni prompt, metriche comparative.

**Cosa cambia**:
- `prompts/v2/` con nuove versioni (quando pronte)
- `evidence_packs.prompt_version` + `verification_logs.prompt_version` (tracking)
- A/B via `tenant_feature_flags` → 10% traffic su v2
- FF: `ff_prompt_v2` (default false)

**Release process per prompt**:
```
DRAFT → EVAL → STAGING → CANARY (10%) → GA (100%)
```

---

## 3. MIGRAZIONE SQL PROGETTATA (Migration 014)

```sql
BEGIN;

-- F.2: Plan tier su tenants
ALTER TABLE core.tenants
    ADD COLUMN IF NOT EXISTS plan VARCHAR(30) DEFAULT 'studio'
        CHECK (plan IN ('studio', 'legal_standard', 'legal_deep', 'enterprise')),
    ADD COLUMN IF NOT EXISTS budget_alert_threshold NUMERIC(5,2) DEFAULT 0.80;

-- F.2: Budget events
CREATE TABLE IF NOT EXISTS core.budget_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES core.tenants(id) ON DELETE CASCADE,
    event_type VARCHAR(30) NOT NULL
        CHECK (event_type IN ('warning_80','warning_95','exceeded','degraded','reset')),
    spend_at_event NUMERIC(10,4),
    budget_limit NUMERIC(10,4),
    model_degraded_from VARCHAR(200),
    model_degraded_to VARCHAR(200),
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_budget_events_tenant
    ON core.budget_events(tenant_id, created_at DESC);
ALTER TABLE core.budget_events ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON core.budget_events
    USING (tenant_id::text = current_setting('app.tenant_id', true)
           OR core.fn_is_superadmin());

-- F.4: Model fallback chain
CREATE TABLE IF NOT EXISTS core.model_fallback_chain (
    role VARCHAR(50) NOT NULL,
    priority INTEGER NOT NULL DEFAULT 0,
    model_name VARCHAR(200) NOT NULL,
    max_retries INTEGER DEFAULT 1,
    timeout_seconds NUMERIC(6,1),
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (role, priority)
);

-- F.4: Seed fallback chain con valori attuali
INSERT INTO core.model_fallback_chain
    (role, priority, model_name, max_retries, timeout_seconds)
VALUES
    ('chat_primary',       0, 'lexe-primary',     1, 30.0),
    ('chat_primary',       1, 'lexe-fast',        1, 15.0),
    ('legis_planner',      0, 'lexe-complex',     1, 60.0),
    ('legis_planner',      1, 'lexe-primary',     1, 45.0),
    ('legis_planner',      2, 'lexe-fast',        0, 30.0),
    ('legis_synthesizer',  0, 'lexe-primary',     1, 120.0),
    ('legis_synthesizer',  1, 'legal-tier2-haiku', 1, 90.0),
    ('legis_verifier',     0, 'legal-tier2-haiku', 0, 30.0),
    ('legis_verifier',     1, 'lexe-fast',        0, 15.0),
    ('lexorc_synthesizer',  0, 'lexe-primary',     1, 120.0),
    ('lexorc_synthesizer',  1, 'legal-tier2-haiku', 1, 90.0),
    ('lexorc_auditor',      0, 'legal-tier2-haiku', 0, 30.0),
    ('lexorc_auditor',      1, 'lexe-fast',        0, 15.0),
    ('intent_detector',    0, 'lexe-fast',        0, 5.0),
    ('lexorc_classifier',   0, 'lexe-fast',        0, 5.0),
    ('fact_extraction',    0, 'lexe-fast',        0, 5.0),
    ('chat_followup',      0, 'lexe-fast',        0, 8.0)
ON CONFLICT DO NOTHING;

-- F.6: Prompt version tracking
ALTER TABLE core.evidence_packs
    ADD COLUMN IF NOT EXISTS prompt_version VARCHAR(50);
ALTER TABLE core.verification_logs
    ADD COLUMN IF NOT EXISTS prompt_version VARCHAR(50);

-- Backfill
UPDATE core.tenants SET plan = 'studio' WHERE plan IS NULL;

COMMIT;
```

---

## 4. OSSERVABILITA E ALERTING (Progettati, non implementati)

### Metriche Prometheus/OTel

| Metrica | Tipo | Labels |
|---------|------|--------|
| `lexe_request_duration_seconds` | histogram | pipeline, complexity, tenant |
| `lexe_tool_calls_total` | counter | tool_name, status, tenant |
| `lexe_llm_calls_total` | counter | model, role, status, tenant |
| `lexe_budget_usage_ratio` | gauge | tenant |
| `lexe_circuit_breaker_state` | gauge | source, state |
| `lexe_pipeline_route_total` | counter | pipeline, complexity |

### Alerting Rules (progettate)

| Alert | Condizione | Severita |
|-------|-----------|----------|
| BudgetExceeded | usage_ratio > 0.95 per 5m | critical |
| ToolErrorRate | error_rate > 20% per 10m | warning |
| LLMErrorRate | error_rate > 10% per 5m | critical |
| CitationAccuracyDrop | accuracy < 0.90 per 1h | critical |
| CircuitBreakerOpen | state == OPEN per 1m | warning |

---

## 5. FEATURE FLAGS (Retrocompatibilita)

Ogni fase e gated da un FF per rollback istantaneo:

| Fase | Feature Flag | Default | Effetto se false |
|------|-------------|---------|------------------|
| F.2 | `ff_budget_enforcement` | false | Budget illimitato (comportamento attuale) |
| F.3 | `ff_tool_budgets` | false | No rate limit per-tool (attuale: solo max 20/turn) |
| F.4 | `ff_model_fallback_chain` | false | 3-tier query attuale |
| F.6 | `ff_prompt_v2` | false | v1 prompts only |

**Nessun cambiamento breaking** — tutti i default preservano il comportamento pre-refactor.

---

## 6. FILE MAP COMPLETA

### File GIA CREATI (F.1 in corso)

```
lexe-core/src/lexe_core/prompts/
├── __init__.py                           ✅ creato
├── registry.py                           ❌ da creare
└── v1/
    ├── __init__.py                       ✅ creato
    ├── studio_system.py                  ✅ STUDIO_SYSTEM_PROMPT_V1
    ├── intent_detector.py                ✅ INTENT_DETECTOR_*_V1
    ├── legis_planner.py                  ✅ PLANNER_*_V1
    ├── legis_researcher.py               ✅ RESEARCHER_SYSTEM_PROMPT_V1
    ├── legis_verifier.py                 ✅ VERIFIER_SYSTEM_PROMPT_V1
    ├── legis_synthesizer.py              ✅ SYNTHESIZER_*_V1 + 7 scenario
    ├── lexorc_classifier.py               ✅ CLASSIFIER_SYSTEM_PROMPT_V1
    ├── lexorc_synthesizer.py              ✅ LEXORC_SYNTHESIZER_SYSTEM_PROMPT_V1
    ├── lexorc_auditor.py                  ✅ AUDITOR_*_V1
    ├── fact_extractor.py                 ✅ FACT_EXTRACTOR_*_V1
    └── follow_up.py                      ✅ FOLLOW_UP_PROMPT_V1
```

### File DA CREARE (F.2-F.6)

| File | Tipo | Fase |
|------|------|------|
| `prompts/registry.py` | NEW | F.1 |
| `gateway/budget_gate.py` | NEW | F.2 |
| `tools/budget.py` | NEW | F.3 |
| `tools/rate_limiter.py` | NEW | F.3 |
| `agent/model_resolver.py` | NEW | F.4 |
| `migrations/014_plan_budget.sql` | NEW | F.2-F.4 |
| `tests/eval/` (intera dir) | NEW | F.5 |

### File DA MODIFICARE

| File | Modifica | Fase |
|------|----------|------|
| `agent/prompts.py` | Import da `prompts/v1/`, re-export | F.1 |
| `agent/classifier.py` | Import `CLASSIFIER_SYSTEM_PROMPT` da `prompts/v1/` | F.1 |
| `agent/agents/lexorc_synthesizer.py` | Import `SYNTHESIZER_SYSTEM_PROMPT` da `prompts/v1/` | F.1 |
| `agent/agents/auditor.py` | Import auditor prompts da `prompts/v1/` | F.1 |
| `gateway/fact_extractor_llm.py` | Import da `prompts/v1/` | F.1 |
| `gateway/customer_router.py` | Import studio + follow-up da `prompts/v1/` | F.1 |
| `gateway/customer_router.py` | + budget gate, model resolver | F.2-F.4 |
| `config.py` | + FF + plan settings | F.2 |
| `identity/schemas/__init__.py` | + plan field su TenantResponse/Create/Update | F.2 |
| `admin/services/catalog_service.py` | + plan validation | F.2 |
| `admin/routers/catalog_router.py` | + fallback chain CRUD | F.4 |

---

## 7. PRIVACY E COMPLIANCE BY DESIGN

| Aspetto | Studio | Legal |
|---------|--------|-------|
| Memory retrieve | Si (L0-L3 + profile) | **No** — zero personal context |
| Memory store | Si (delta + evolve) | **No** — no facts extracted |
| Cross-conversation | Via memory | **Impossibile by design** |
| Audit trail | evidence_packs + verification_logs | Stesso + immutabile |
| Retention | 90d (evidence_packs) | 90d (GDPR Art. 5.1.e) |
| Tool results | Cache per-session only | Cache per-session only |

**Zero cross-session in Legal** e implementato via mode lock: una volta che la conversazione e in modalita legal, memory retrieve/store sono disabilitati a livello di codice (non solo di config).

---

## 8. PROSSIMI PASSI IMMEDIATI

### Completare F.1 (oggi)
1. Creare `prompts/registry.py`
2. Rewire imports in 6 file sorgente
3. py_compile check
4. Commit + push `stage`
5. Deploy staging + smoke test

### Avviare F.2 (prossimo sprint)
1. Migration 014 (plan + budget_events)
2. `budget_gate.py`
3. `config.py` aggiornato con FF
4. `identity/schemas/__init__.py` + plan field
5. Admin panel: piano tenant nel Tenant Editor
6. Test con `ff_budget_enforcement=false` (shadow mode)

### Parallelo: F.5 seed
1. Creare `tests/eval/gold_sets/` con prime 20 test cases
2. Runner base per format compliance
3. CI step (non blocking inizialmente)

---

## 9. KNOWN FAILURE MODES

| Tipo | Causa Root | Mitigazione pianificata |
|------|-----------|------------------------|
| Hallucinated citation | LLM inventa articoli plausibili | Verifier L1 structural + eval harness (F.5) |
| Wrong code | Cross-code score penalty insufficiente | normattiva_search scoring fix |
| Vigenza stale | Cache Normattiva non aggiornata | vigenza_fast con TTL 24h |
| Missing annex `:2` | URN senza `:2` per codici | `CODICI_PREDEFINITI` gia presente |
| Tool timeout cascade | Normattiva.it down | Per-tool CB (F.3) |
| LLM model unavailable | Provider outage | Fallback chain (F.4) |
| Budget overflow | `litellm_budget_monthly` non enforced | Gate 2 budget check (F.2) |
| Memory leak to Legal | Facts leakano da Studio a Legal | Mode lock gia implementato |

---

*Generato il 2026-02-20. Riferimento: `lexe-docs/architecture/LEXE-REFACTOR-BLUEPRINT.md`*
