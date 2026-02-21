# LEXE Platform — Refactor Blueprint: Routing, Models, Tools, Prompts, Quality Gates

> **Version**: 1.0 — 2026-02-20
> **Scope**: Routing automatico, scelta modelli, tool governance, prompt governance, quality gates
> **Constraint**: Compliance by design (GDPR, audit immutabile, zero cross-session in Legal)

---

## Stato Attuale — Sintesi Critica

### Cosa esiste oggi

| Layer | Stato | Debt |
|-------|-------|------|
| **Routing** | 3 pipeline (Studio toolloop, LEGIS 4-phase, LEXORC 1+5 agents) selezionate da FF + intent detector | Intent detector è LLM-based ma non ha eval harness; fallback chain è implicita |
| **Model selection** | 3-tier (tenant override → global default → hardcoded), 8 role slots, 9 modelli in LiteLLM | Nessun budget cap runtime; nessun quality gate pre-deploy su model swap; config JSONB non validata |
| **Tool governance** | 8 tools, purity 1-5, persona-based allow/block, max 20 calls/turn | Nessun tool budget per-tenant; circuit breaker solo per source, non per tool; no rate limit per-tool |
| **Prompt governance** | ~15 prompt inline in Python files, no versioning, no A/B test | Prompt hardcoded, no release process, no redaction audit, no regression test |
| **Quality gates** | Auditor L1 structural + L2 LLM coherence, verification_logs table | No eval harness automatico; no gold set; no acceptance criteria formali |
| **Budget** | Virtual key per tenant, spend dashboard, `litellm_budget_monthly` column (unused) | Budget non enforced runtime; no alert su soglia; no tier-based routing |

### Architettura Pipeline Attuale

```
Request → Auth (Logto JWT) → Tenant Resolution (cache 5min)
        → Contact Resolution (JIT create)
        → Conversation Mode Lock (studio|legal, locked after 1st msg)
        → Mode Preset (tools, memory, context window)
        → Model Role Resolution (3-tier query)
        → LiteLLM Key Resolution (virtual → master fallback)
        → Pipeline Router:
            ├─ ff_orchestrator_enabled? → Orchestrator (disabled)
            ├─ ff_legis_agent? → Intent Detector → "toolloop"|"legis"|"lexorc"
            │   ├─ "lexorc" + ff_lexorc_enabled → LexeOrchestrator
            │   ├─ "legis" → legis_agent_pipeline (P→R→V→S)
            │   └─ "toolloop" → Studio tool loop
            └─ fallback → Studio tool loop
        → Memory Retrieve (v2: L0-L3 + profile)
        → [Pipeline Execution]
        → Memory Store (delta + profile evolve)
        → Follow-up Generation
        → done_event
```

---

## A) Target Architecture — Routing Decision Tree & Model Policy

### A.1 Piano di Segmentazione

| Piano | Pipeline ammesse | Tool budget/turn | LLM budget/msg | Memory | Context window |
|-------|-----------------|------------------|-----------------|--------|----------------|
| **Studio** | Studio toolloop only | 10 calls, 60s total | $0.02 | L0-L3 | 15 msg |
| **Legal Standard** | Studio + LEGIS | 15 calls, 90s total | $0.08 | Off (zero cross-session) | 60 msg |
| **Legal Deep** | Studio + LEGIS + LEXORC | 25 calls, 180s total | $0.25 | Off | 60 msg |
| **Enterprise** | All + custom | Configurable | Configurable | Configurable | Configurable |

### A.2 Routing Decision Tree (Target)

```
Request enters
│
├─ [Gate 1: Tenant Plan Check]
│   tenant.plan ∈ {studio, legal_standard, legal_deep, enterprise}
│   → Reject if pipeline not in plan's allowed set
│
├─ [Gate 2: Budget Check]  ← NEW
│   IF tenant.spend_month >= tenant.budget_monthly * 0.95
│   → Degrade to cheaper model tier, emit budget_warning event
│   IF tenant.spend_month >= tenant.budget_monthly
│   → Block with 429, emit budget_exceeded event
│
├─ [Gate 3: Mode Resolution]
│   conversation_mode = "studio" | "legal"
│   → Apply ModePreset (tools, memory, context)
│
├─ [Gate 4: Pipeline Selection]  ← REFACTORED
│   IF mode == "studio":
│       → Studio Toolloop (always, no classifier needed)
│   IF mode == "legal":
│       ├─ [Intent Classifier — deterministic first, LLM fallback]
│       │   Rule-based fast path:
│       │   - "art. N" + single code → DIRECT (no planner, 1 normattiva_search)
│       │   - "testo art." / "cosa dice" → SIMPLE (light planner, 3-5 tools)
│       │   - default → LLM classifier → STANDARD | COMPLEX
│       │   - explicit "analisi approfondita" / multi-domain → COMPLEX
│       │
│       ├─ DIRECT → normattiva_search + concise synth (lexe-fast)
│       ├─ SIMPLE → light planner (lexe-fast) + 3-5 tools + concise synth
│       ├─ STANDARD → full LEGIS (planner→research→verify→synth)
│       └─ COMPLEX → LEXORC if plan allows, else LEGIS with extended budget
│
├─ [Gate 5: Model Selection]  ← NEW policy engine
│   resolved_model = ModelPolicy.resolve(
│       role, tenant_plan, complexity, budget_remaining
│   )
│   → Applies tier downgrade if budget pressure
│   → Validates model is active in catalog
│   → Falls back to plan default if override invalid
│
└─ [Gate 6: Tool Policy]
    tools = ToolPolicy.resolve(
        mode_preset, persona, tenant_plan, budget_remaining
    )
    → Intersects: preset ∩ persona ∩ plan_allowed
    → Applies per-tool rate limit
    → Applies total call budget
```

### A.3 Model Selection Policy

| Role | Studio | Legal Standard | Legal Deep | Enterprise |
|------|--------|---------------|------------|------------|
| `chat_primary` | lexe-fast | lexe-primary | lexe-primary | tenant override |
| `chat_followup` | lexe-fast | lexe-fast | lexe-fast | tenant override |
| `fact_extraction` | lexe-fast | — (no memory) | — (no memory) | tenant override |
| `legis_planner` | — | lexe-primary | lexe-complex | tenant override |
| `legis_planner_light` | — | lexe-fast | lexe-fast | tenant override |
| `legis_synthesizer` | — | lexe-primary | lexe-primary | tenant override |
| `legis_verifier` | — | legal-tier2-haiku | legal-tier2-haiku | tenant override |
| `intent_detector` | — | lexe-fast | lexe-fast | tenant override |
| `lexorc_classifier` | — | — | lexe-fast | tenant override |
| `lexorc_synthesizer` | — | — | lexe-primary | tenant override |
| `lexorc_auditor` | — | — | legal-tier2-haiku | tenant override |

**Budget pressure degradation chain**:
```
Normal → tier3 (frontier)
>80% budget → tier2 (mid, e.g. haiku)
>95% budget → tier1 (flash/free)
>100% budget → 429 reject
```

---

## B) Nuova Mappa Modelli — Tiers, Quality Gates, Fallback

### B.1 Model Tier System

| Tier | Models | Cost Range | Use Cases | Quality Gate |
|------|--------|------------|-----------|--------------|
| **tier1-free** | gemini-2.5-flash | ~$0 | Classification, follow-ups, fact extraction | Latency < 2s, format compliance > 95% |
| **tier2-economy** | claude-haiku-4.5, gemini-2.5-flash | $0.001-0.005/1K | Verification, light planning, intent detection | Accuracy > 85% on eval set |
| **tier3-standard** | mistral-large-2512, claude-sonnet-4.6 | $0.005-0.02/1K | Chat, synthesis, research | Legal citation accuracy > 90% |
| **tier4-frontier** | gpt-5.2, claude-opus-4.6 | $0.02-0.10/1K | Complex planning, strategic analysis | Reasoning quality > 92% on eval set |

### B.2 Fallback Chain per Role

```yaml
legis_planner:
  primary: lexe-complex       # tier4
  fallback_1: lexe-primary    # tier3
  fallback_2: lexe-fast       # tier1
  max_retries: 1
  timeout: 60s

legis_synthesizer:
  primary: lexe-primary       # tier3
  fallback_1: legal-tier2-haiku  # tier2
  fallback_2: lexe-fast       # tier1
  max_retries: 1
  timeout: 120s

legis_verifier:
  primary: legal-tier2-haiku  # tier2
  fallback_1: lexe-fast       # tier1
  max_retries: 0              # structural L1 if LLM fails
  timeout: 30s

chat_primary:
  primary: lexe-primary       # tier3
  fallback_1: lexe-fast       # tier1
  max_retries: 1
  timeout: 30s

lexorc_synthesizer:
  primary: lexe-primary       # tier3
  fallback_1: legal-tier2-haiku  # tier2
  timeout: 120s
```

### B.3 Quality Gates per Model Swap

**Prima di promuovere un modello in produzione**:

1. **Format Compliance Test** (automated):
   - JSON output parse rate > 98% (planner, classifier, verifier)
   - Markdown structure compliance > 95% (synthesizer)
   - Run on 50 golden prompts

2. **Legal Citation Accuracy** (automated):
   - Correct article numbers > 95%
   - Correct code identification > 98%
   - No fabricated citations (0 tolerance)
   - Run on 20 citation test cases

3. **Vigenza Accuracy** (automated):
   - Correct vigenza status > 98%
   - Run on 10 known abrogated + 10 vigente articles

4. **Latency SLA**:
   - P50 < role_timeout * 0.5
   - P95 < role_timeout * 0.8
   - P99 < role_timeout

5. **Cost Gate**:
   - New model cost/1K < current * 1.5 (unless justified)
   - Approval required if cost increase > 50%

### B.4 Migration: DB Changes

```sql
-- New: Plan tier on tenants
ALTER TABLE core.tenants
    ADD COLUMN plan VARCHAR(30) DEFAULT 'studio'
        CHECK (plan IN ('studio', 'legal_standard', 'legal_deep', 'enterprise')),
    ADD COLUMN budget_alert_threshold NUMERIC(5,2) DEFAULT 0.80;

-- New: Model fallback chain
CREATE TABLE core.model_fallback_chain (
    role VARCHAR(50) NOT NULL,
    priority INTEGER NOT NULL,        -- 0=primary, 1=fallback_1, etc.
    model_name VARCHAR(200) NOT NULL,
    max_retries INTEGER DEFAULT 1,
    timeout_seconds NUMERIC(6,1),
    PRIMARY KEY (role, priority),
    FOREIGN KEY (model_name) REFERENCES core.llm_model_catalog(model_name)
);

-- New: Budget enforcement log
CREATE TABLE core.budget_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES core.tenants(id),
    event_type VARCHAR(30) NOT NULL
        CHECK (event_type IN ('warning_80', 'warning_95', 'exceeded', 'degraded', 'reset')),
    spend_at_event NUMERIC(10,4),
    budget_limit NUMERIC(10,4),
    created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_budget_events_tenant ON core.budget_events(tenant_id, created_at DESC);
```

---

## C) Tool Governance

### C.1 Tool Access Matrix per Piano

| Tool | Studio | Legal Std | Legal Deep | Enterprise |
|------|--------|-----------|------------|------------|
| normattiva_search | Y | Y | Y | configurable |
| eurlex_search | Y | Y | Y | configurable |
| infolex_search | Y | Y | Y | configurable |
| kb_search | Y | Y | Y | configurable |
| kb_normativa_search | Y | Y | Y | configurable |
| lex_search | Y | Y | Y | configurable |
| lex_search_enriched | N | Y | Y | configurable |
| web_search | opt-in | N | opt-in | configurable |

### C.2 Tool Budget per Turn

```python
@dataclass
class ToolBudget:
    max_calls_total: int          # Max tool invocations per turn
    max_calls_per_tool: int       # Max invocations of same tool
    max_wall_time_seconds: float  # Total wall time for all tools
    max_parallel: int             # Max concurrent tool calls

TOOL_BUDGETS = {
    "studio":         ToolBudget(10, 3, 60.0, 3),
    "legal_standard": ToolBudget(15, 5, 90.0, 5),
    "legal_deep":     ToolBudget(25, 8, 180.0, 8),
    "enterprise":     ToolBudget(50, 10, 300.0, 10),
}
```

### C.3 Per-Tool Rate Limits (NEW)

```python
TOOL_RATE_LIMITS = {
    "normattiva_search":    {"per_turn": 8, "timeout": 30.0, "retry": 1},
    "eurlex_search":        {"per_turn": 6, "timeout": 30.0, "retry": 1},
    "infolex_search":       {"per_turn": 6, "timeout": 30.0, "retry": 1},
    "kb_search":            {"per_turn": 8, "timeout": 20.0, "retry": 0},
    "kb_normativa_search":  {"per_turn": 6, "timeout": 20.0, "retry": 0},
    "lex_search":           {"per_turn": 6, "timeout": 15.0, "retry": 1},
    "lex_search_enriched":  {"per_turn": 4, "timeout": 30.0, "retry": 0},
    "web_search":           {"per_turn": 3, "timeout": 15.0, "retry": 1},
}
```

### C.4 Circuit Breaker Enhancement

```
Current: Per-source CB (3 failures / 120s window / 30s recovery)
Target:  Per-tool CB   (3 failures / 120s window / 60s recovery)
         + Global CB   (10 failures across all tools / 300s window / 120s recovery)
```

**New**: Tool health status exposed via SSE `tool_health_event` on degradation.

### C.5 Privacy & Zero Cross-Session (Legal Mode)

| Aspect | Studio | Legal |
|--------|--------|-------|
| Memory retrieve | Yes (L0-L3 + profile) | **No** — zero personal context |
| Memory store | Yes (delta + evolve) | **No** — no facts extracted |
| Tool results cached | Per-session only | Per-session only |
| Conversation context | 15 msg window | 60 msg window (same conv only) |
| Cross-conversation | Via memory | **Impossible by design** |
| Audit trail | evidence_packs + verification_logs | Same + immutable |
| Retention | 90d (evidence_packs) | 90d (GDPR Art. 5.1.e) |

---

## D) Prompt Governance — Versioning, Redaction, Release Process

### D.1 Prompt Registry (NEW)

```
lexe-core/src/lexe_core/prompts/
├── registry.py              # PromptRegistry singleton
├── v1/
│   ├── studio_system.py     # STUDIO_SYSTEM_PROMPT_V1
│   ├── legis_planner.py     # LEGIS_PLANNER_V1
│   ├── legis_verifier.py    # LEGIS_VERIFIER_V1
│   ├── legis_synthesizer.py # LEGIS_SYNTHESIZER_V1 + scenarios
│   ├── lexorc_classifier.py  # LEXORC_CLASSIFIER_V1
│   ├── lexorc_synthesizer.py # LEXORC_SYNTHESIZER_V1
│   ├── lexorc_auditor.py     # LEXORC_AUDITOR_V1 (L2)
│   ├── intent_detector.py   # INTENT_DETECTOR_V1
│   ├── fact_extractor.py    # FACT_EXTRACTOR_V1
│   ├── follow_up.py         # FOLLOW_UP_V1
│   └── memory_summary.py    # MEMORY_SUMMARY_V1
├── v2/                      # Next version (when ready)
│   └── ...
└── __init__.py
```

### D.2 Prompt Schema

```python
@dataclass(frozen=True)
class PromptVersion:
    id: str                    # "legis_planner_v1"
    version: int               # 1
    template: str              # The actual prompt text
    variables: list[str]       # ["user_message", "conversation_context"]
    model_compatibility: list[str]  # ["lexe-complex", "lexe-primary"]
    eval_set_id: str | None    # "legis_planner_eval_v1"
    created_at: str            # ISO date
    author: str                # "claude" or "human"
    changelog: str             # What changed
```

### D.3 Release Process

```
1. DRAFT   → Prompt written, not tested
2. EVAL    → Running against eval set (automated)
3. STAGING → Deployed to staging, shadow mode (logs but doesn't serve)
4. CANARY  → 10% traffic on staging
5. GA      → Full deployment to production
```

**Rollback**: Instant via `PromptRegistry.set_active_version(prompt_id, version)`.

### D.4 Redaction Rules

| Data Type | Redaction | Applies to |
|-----------|-----------|------------|
| User PII (name, email) | Hash before logging | All prompts in audit_log |
| Legal case details | No redaction (needed for function) | Evidence packs |
| Tool results | Truncate to 500 chars in logs | tool_execution_logs |
| Full LLM response | Store in conversation_messages (RLS) | Per-tenant isolation |
| Prompt templates | Never log variables, only template ID | Audit trail |

### D.5 Migration Path

**Phase 1** (non-breaking): Extract all inline prompts into `prompts/v1/` files. Import from there. Zero behavior change.

**Phase 2**: Add `prompt_version` column to `evidence_packs` and `verification_logs`. Track which prompt version produced each result.

**Phase 3**: Build A/B testing via `tenant_feature_flags` — route 10% of tenants to `v2/` prompts, compare metrics.

---

## E) Eval Harness — Test Suite, Gold Set, Metrics, Acceptance Criteria

### E.1 Eval Structure

```
lexe-core/tests/eval/
├── gold_sets/
│   ├── studio_30.jsonl          # 30 Studio prompts + expected
│   ├── legis_30.jsonl           # 30 LEGIS prompts + expected
│   ├── lexorc_10.jsonl           # 10 LEXORC prompts + expected
│   ├── citations_20.jsonl       # 20 citation verification cases
│   └── vigenza_10.jsonl         # 10 vigenza verification cases
├── runners/
│   ├── format_compliance.py     # JSON/Markdown parse tests
│   ├── citation_accuracy.py     # Legal reference verification
│   ├── vigenza_checker.py       # Is-vigente correctness
│   ├── hallucination_detector.py # Fabricated citation detection
│   └── latency_profiler.py      # P50/P95/P99 measurement
├── harness.py                   # Main eval orchestrator
└── report.py                    # HTML/JSON report generator
```

### E.2 Gold Set — Studio (30 esempi)

| # | Query | Expected Pipeline | Expected Tools | Quality Check |
|---|-------|-------------------|----------------|---------------|
| 1 | "Cos'è il codice civile?" | toolloop | none | Risposta conversazionale, no tool |
| 2 | "Che differenza c'è tra dolo e colpa?" | toolloop | none | Definizioni corrette |
| 3 | "Spiegami l'art. 2043 cc" | toolloop | normattiva_search | Cita art. 2043, testo vigente |
| 4 | "Quali sono i requisiti del contratto?" | toolloop | kb_normativa_search | Cita art. 1325 cc |
| 5 | "Cosa dice il GDPR sui cookie?" | toolloop | eurlex_search | Cita Reg. 2016/679 |
| 6 | "Mi serve un modello di NDA" | toolloop | lex_search | Template contract output |
| 7 | "Termine prescrizione per danni" | toolloop | normattiva+kb | art. 2947 cc |
| 8 | "Come funziona il decreto ingiuntivo?" | toolloop | normattiva+infolex | art. 633+ cpc |
| 9 | "Differenza tra recesso e risoluzione" | toolloop | kb_normativa | art. 1453, 1373 cc |
| 10 | "Cos'è la responsabilità precontrattuale?" | toolloop | kb_search | art. 1337-1338 cc |
| 11-30 | ... (simili pattern: domande dirette, lookup singoli, definizioni) | ... | ... | ... |

### E.3 Gold Set — LEGIS (30 esempi)

| # | Query | Level | Key Norms | Quality Check |
|---|-------|-------|-----------|---------------|
| 1 | "Analisi art. 2043 cc giurisprudenza recente" | STANDARD | art. 2043 cc | Planner → 3+ tools, synth con [N] |
| 2 | "Responsabilità solidale appaltatore/committente" | STANDARD | art. 29 D.Lgs. 81/2008 | Cross-ref normattiva + kb |
| 3 | "Clausola penale vs caparra confirmatoria" | STANDARD | art. 1382-1386, 1385 cc | Distinzione corretta |
| 4 | "Danno non patrimoniale: evoluzione giurisprudenziale" | COMPLEX | art. 2059 cc | Timeline massime, S.U. 2008 |
| 5 | "Requisiti per il monitorio europeo" | STANDARD | Reg. 1896/2006 | EUR-Lex + normattiva |
| 6 | "GDPR art. 6 basi giuridiche + casistica italiana" | COMPLEX | Reg. 679/2016, D.Lgs. 196/03 | Dual source IT+EU |
| 7 | "Tutela del consumatore: garanzia legale" | STANDARD | D.Lgs. 206/2005 art. 128-135 | Codice consumo vigente |
| 8 | "Licenziamento per giusta causa: onere prova" | STANDARD | art. 2119 cc, L. 300/70 art. 18 | KB massime su onere |
| 9 | "Sequestro conservativo: presupposti" | SIMPLE | art. 671 cpc | Normattiva + dottrina |
| 10 | "Art. 32 Costituzione e TSO" | SIMPLE | Cost. art. 32 | URN senza annex |
| 11-30 | ... (ricerche complesse, pareri, contratti) | ... | ... | ... |

### E.4 Gold Set — LEXORC (10 esempi)

| # | Query | Complexity | Agents Expected | Quality Check |
|---|-------|------------|-----------------|---------------|
| 1 | "Analisi completa responsabilità medica" | DEEP | Norm+CaseLaw+Doctrine | 3 agent parallel, auditor pass |
| 2 | "Evoluzione giurisprudenza danno da ritardo PA" | DEEP | Norm+CaseLaw | Timeline cronologica |
| 3 | "Confronto tutele: concorrenza sleale vs antitrust" | STRATEGIC | All agents | Dual domain, plan A/B |
| 4-10 | ... | ... | ... | ... |

### E.5 Gold Set — Citazioni (20 casi)

| # | Citazione | Tipo | Verifica |
|---|-----------|------|----------|
| 1 | art. 2043 c.c. | CC | Vigente, URN `:2~art2043` |
| 2 | art. 640 c.p. | CP | Vigente, URN `:2~art640` |
| 3 | art. 2 Cost. | COST | Vigente, no annex |
| 4 | art. 700 c.p.c. | CPC | Vigente, URN `:2~art700` |
| 5 | art. 6 Reg. 2016/679 | EU | Vigente, CELEX 32016R0679 |
| 6 | art. 18 L. 300/1970 | Legge | Vigente, no annex |
| 7 | art. 2087 c.c. | CC | Vigente |
| 8 | art. 74 D.Lgs. 152/2006 | D.Lgs. | Vigente, no annex |
| 9 | art. 1 L. 241/1990 | Legge | Vigente |
| 10 | art. 844 c.c. | CC | Vigente (immissioni) |
| 11 | art. 2947 c.c. co. 3 | CC | Vigente (prescrizione danni) |
| 12 | art. 33 Reg. 2016/679 | EU | Vigente (GDPR data breach) |
| 13 | art. 1218 c.c. | CC | Vigente (inadempimento) |
| 14 | art. 633 c.p.c. | CPC | Vigente (decreto ingiuntivo) |
| 15 | art. 19 D.Lgs. 231/2001 | D.Lgs. | Vigente (resp. enti) |
| 16 | art. 146 c.p. (abrogato) | CP | **Abrogato** — test negativo |
| 17 | art. 5 D.Lgs. 196/2003 | D.Lgs. | **Abrogato** da D.Lgs. 101/2018 — test negativo |
| 18 | art. 2058 c.c. | CC | Vigente (reintegrazione in forma specifica) |
| 19 | art. 111 Cost. | COST | Vigente (giusto processo) |
| 20 | art. 2729 c.c. | CC | Vigente (presunzioni semplici) |

### E.6 Known Failure Modes

| Tipo | Descrizione | Causa Root | Mitigazione |
|------|-------------|-----------|-------------|
| **Hallucinated citation** | LLM cita "art. 2043-bis c.c." (non esiste) | Modello inventa articoli plausibili | Verifier L1 structural check |
| **Wrong code** | Chiede art. 640 CP, restituisce art. 640 CPC | Score cross-code penalty insufficiente | normattiva_search scoring fix |
| **Vigenza stale** | Cita art. come vigente ma abrogato nel 2023 | Cache Normattiva non aggiornata | vigenza_fast con TTL 24h |
| **Missing annex :2** | URL punta a decreto approvazione invece di codice | URN senza `:2` per codici | CODICI_PREDEFINITI con `urn_annex` |
| **Tool timeout cascade** | 3+ tools timeout → circuit breaker apre tutto | Normattiva.it down | Per-tool CB, graceful degradation |
| **LLM model unavailable** | OpenRouter 502 per modello specifico | Provider outage | Fallback chain per role |
| **Budget overflow** | Tenant supera budget senza enforcement | `litellm_budget_monthly` non enforced | Gate 2 budget check |
| **Memory leak to Legal** | Personal facts from Studio leak to Legal conv | Mode switch after memory store | Zero cross-session by mode lock |

### E.7 Acceptance Criteria

| Metric | Minimum | Target | Blocking |
|--------|---------|--------|----------|
| Format compliance (JSON/MD) | > 95% | > 98% | Yes |
| Legal citation accuracy | > 90% | > 95% | Yes |
| Vigenza accuracy | > 95% | > 98% | Yes |
| Zero fabricated citations | 0 | 0 | **Absolute** |
| Latency P95 (Studio) | < 8s | < 5s | Yes |
| Latency P95 (LEGIS) | < 30s | < 20s | Yes |
| Latency P95 (LEXORC) | < 120s | < 60s | No |
| Tool success rate | > 90% | > 95% | Yes |
| Budget adherence | 100% | 100% | **Absolute** |
| Cross-session leak (Legal) | 0 | 0 | **Absolute** |

### E.8 Automated Eval Pipeline

```
ci/eval.yml:
  trigger: on PR to main/stage with changes in prompts/ or agent/
  steps:
    1. Start ephemeral LiteLLM (mock or staging)
    2. Run format_compliance.py against gold_sets/*.jsonl
    3. Run citation_accuracy.py against citations_20.jsonl
    4. Run vigenza_checker.py against vigenza_10.jsonl
    5. Run hallucination_detector.py (regex + structural)
    6. Run latency_profiler.py (10 representative queries)
    7. Generate report.html
    8. Gate: FAIL PR if any blocking metric below minimum
```

---

## F) Piano Rollout — Migrazioni, Osservabilità, Alerting, Retrocompatibilità

### F.1 Rollout Phases

| Phase | Scope | Duration | Risk |
|-------|-------|----------|------|
| **F.1** Prompt extraction | Extract prompts to `prompts/v1/`, zero behavior change | 1 sprint | None |
| **F.2** Budget enforcement | Add plan column, budget gates, degrade logic | 1 sprint | Low (additive) |
| **F.3** Tool governance | Per-tool rate limits, enhanced CB, tool budgets | 1 sprint | Low |
| **F.4** Model fallback chain | DB-backed fallback, tier degradation | 1 sprint | Medium |
| **F.5** Eval harness | Gold sets, automated CI gate | 1 sprint | None (test only) |
| **F.6** Prompt versioning | A/B routing, v2 prompts, metrics comparison | 1 sprint | Medium |

### F.2 Migration Plan

**Migration 014: Plan & Budget**
```sql
BEGIN;

-- Plan column
ALTER TABLE core.tenants
    ADD COLUMN IF NOT EXISTS plan VARCHAR(30) DEFAULT 'studio'
        CHECK (plan IN ('studio', 'legal_standard', 'legal_deep', 'enterprise')),
    ADD COLUMN IF NOT EXISTS budget_alert_threshold NUMERIC(5,2) DEFAULT 0.80;

-- Budget events table
CREATE TABLE IF NOT EXISTS core.budget_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES core.tenants(id) ON DELETE CASCADE,
    event_type VARCHAR(30) NOT NULL
        CHECK (event_type IN ('warning_80', 'warning_95', 'exceeded', 'degraded', 'reset')),
    spend_at_event NUMERIC(10,4),
    budget_limit NUMERIC(10,4),
    model_degraded_from VARCHAR(200),
    model_degraded_to VARCHAR(200),
    created_at TIMESTAMPTZ DEFAULT now() NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_budget_events_tenant
    ON core.budget_events(tenant_id, created_at DESC);

-- RLS
ALTER TABLE core.budget_events ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON core.budget_events
    USING (tenant_id::text = current_setting('app.tenant_id', true) OR core.fn_is_superadmin());

-- Model fallback chain
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

-- Seed fallback chains
INSERT INTO core.model_fallback_chain (role, priority, model_name, max_retries, timeout_seconds)
VALUES
    ('chat_primary', 0, 'lexe-primary', 1, 30.0),
    ('chat_primary', 1, 'lexe-fast', 1, 15.0),
    ('legis_planner', 0, 'lexe-complex', 1, 60.0),
    ('legis_planner', 1, 'lexe-primary', 1, 45.0),
    ('legis_planner', 2, 'lexe-fast', 0, 30.0),
    ('legis_synthesizer', 0, 'lexe-primary', 1, 120.0),
    ('legis_synthesizer', 1, 'legal-tier2-haiku', 1, 90.0),
    ('legis_verifier', 0, 'legal-tier2-haiku', 0, 30.0),
    ('legis_verifier', 1, 'lexe-fast', 0, 15.0),
    ('lexorc_synthesizer', 0, 'lexe-primary', 1, 120.0),
    ('lexorc_synthesizer', 1, 'legal-tier2-haiku', 1, 90.0),
    ('lexorc_auditor', 0, 'legal-tier2-haiku', 0, 30.0),
    ('lexorc_auditor', 1, 'lexe-fast', 0, 15.0),
    ('intent_detector', 0, 'lexe-fast', 0, 5.0),
    ('lexorc_classifier', 0, 'lexe-fast', 0, 5.0),
    ('fact_extraction', 0, 'lexe-fast', 0, 5.0),
    ('chat_followup', 0, 'lexe-fast', 0, 8.0)
ON CONFLICT DO NOTHING;

-- Prompt version tracking
ALTER TABLE core.evidence_packs
    ADD COLUMN IF NOT EXISTS prompt_version VARCHAR(50);
ALTER TABLE core.verification_logs
    ADD COLUMN IF NOT EXISTS prompt_version VARCHAR(50);

-- Backfill existing tenants to 'studio' plan
UPDATE core.tenants SET plan = 'studio' WHERE plan IS NULL;

COMMIT;
```

### F.3 Osservabilità

**Metriche da esporre (Prometheus/OpenTelemetry)**:

| Metric | Type | Labels | Alert |
|--------|------|--------|-------|
| `lexe_request_duration_seconds` | histogram | pipeline, complexity, tenant | P95 > SLA |
| `lexe_tool_calls_total` | counter | tool_name, status, tenant | error_rate > 20% |
| `lexe_tool_duration_seconds` | histogram | tool_name, tenant | P95 > timeout |
| `lexe_llm_calls_total` | counter | model, role, status, tenant | error_rate > 10% |
| `lexe_llm_tokens_total` | counter | model, direction(in/out), tenant | — |
| `lexe_budget_usage_ratio` | gauge | tenant | > 0.80 warn, > 0.95 crit |
| `lexe_budget_events_total` | counter | event_type, tenant | exceeded > 0 |
| `lexe_circuit_breaker_state` | gauge | source, state | open > 0 |
| `lexe_pipeline_route_total` | counter | pipeline, complexity | — |
| `lexe_citation_accuracy` | gauge | eval_run | < 0.90 crit |
| `lexe_memory_operations_total` | counter | operation, status | error_rate > 30% |

### F.4 Alerting Rules

```yaml
groups:
  - name: lexe-core
    rules:
      - alert: BudgetExceeded
        expr: lexe_budget_usage_ratio > 0.95
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "Tenant {{ $labels.tenant }} at {{ $value }}% budget"

      - alert: ToolErrorRate
        expr: rate(lexe_tool_calls_total{status="error"}[5m]) /
              rate(lexe_tool_calls_total[5m]) > 0.20
        for: 10m
        labels: { severity: warning }

      - alert: LLMErrorRate
        expr: rate(lexe_llm_calls_total{status!="200"}[5m]) /
              rate(lexe_llm_calls_total[5m]) > 0.10
        for: 5m
        labels: { severity: critical }

      - alert: CitationAccuracyDrop
        expr: lexe_citation_accuracy < 0.90
        for: 1h
        labels: { severity: critical }

      - alert: CircuitBreakerOpen
        expr: lexe_circuit_breaker_state == 2  # OPEN
        for: 1m
        labels: { severity: warning }

      - alert: HighLatency
        expr: histogram_quantile(0.95, lexe_request_duration_seconds) > 30
        for: 10m
        labels: { severity: warning }
```

### F.5 Retrocompatibilità

| Change | Breaking? | Migration |
|--------|-----------|-----------|
| `plan` column on tenants | No (default 'studio') | Backfill all existing |
| Budget enforcement | No (default unlimited for 'studio') | No change for existing |
| Tool rate limits | No (limits higher than current usage) | Shadow mode first |
| Prompt extraction | No (same text, different import) | Mechanical refactor |
| Model fallback chain | No (additive, existing behavior preserved) | Seed with current defaults |
| Eval harness | No (test-only, no production impact) | CI config only |
| Prompt versioning | No (v1 = current behavior) | Column addition only |

### F.6 Feature Flag Gating

Ogni fase è gated da un FF per rollback istantaneo:

| Phase | Feature Flag | Default |
|-------|-------------|---------|
| F.2 Budget | `ff_budget_enforcement` | false |
| F.3 Tool governance | `ff_tool_budgets` | false |
| F.4 Model fallback | `ff_model_fallback_chain` | false |
| F.6 Prompt A/B | `ff_prompt_v2` | false |

---

## Appendice: File Map delle Modifiche

| File | Tipo | Phase |
|------|------|-------|
| `prompts/v1/*.py` (11 files) | NEW | F.1 |
| `prompts/registry.py` | NEW | F.1 |
| `gateway/budget_gate.py` | NEW | F.2 |
| `gateway/customer_router.py` | EDIT (add gates) | F.2-F.4 |
| `tools/budget.py` | NEW | F.3 |
| `tools/rate_limiter.py` | NEW | F.3 |
| `agent/model_resolver.py` | NEW | F.4 |
| `migrations/014_plan_budget.sql` | NEW | F.2-F.4 |
| `tests/eval/` (entire dir) | NEW | F.5 |
| `config.py` | EDIT (add FF + plan settings) | F.2 |
| `admin/services/catalog_service.py` | EDIT (plan validation) | F.2 |
| `admin/routers/catalog_router.py` | EDIT (fallback chain CRUD) | F.4 |
| `identity/schemas/__init__.py` | EDIT (plan field) | F.2 |
