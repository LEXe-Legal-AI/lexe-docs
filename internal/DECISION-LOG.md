---
owner: platform-team
audience: internal
source_of_truth: true
status: active
sensitivity: internal
last_verified: 2026-03-19
---

# Architecture Decision Log -- LEXE Platform

> Chronological record of architecture decisions, extracted from ADRs, sprint notes, and reference-historical documents.

---

## Decision Index

| ID | Decision | Date | Status |
|----|----------|------|--------|
| D1 | Unified Intent Classifier | 2026-02-21 | Superseded by Pipeline v2 |
| D2 | Differentiated Semaphores | 2026-02-21 | Active |
| D3 | LEXORC Roles via Database | 2026-02-21 | Active (roles reused by Pipeline v2) |
| D4 | ff_legis_agent Default True | 2026-02-21 | Active |
| D5 | 3-Level Confidence Spectrum | 2026-02-21 | Active |
| D6 | Self-Correction Max 2 Retries | 2026-02-21 | Active |
| D7 | Pipeline v2 Binary Routing | 2026-03-12 | Active |
| D8 | All Models to Gemini 3 Flash | 2026-03-12 | Active |
| D9 | Depth Budgets (MINIMAL/STANDARD/DEEP) | 2026-03-15 | Active |
| D10 | Super Tool Deprecation | 2026-03-12 | Active (code retained) |
| D11 | LEXORC Deprecation | 2026-02-21 | Active (code retained) |
| D12 | GDPR Consent System | 2026-03-19 | Active (FF gated, default false) |

---

## ADR-D1: Unified Intent Classifier

| Field | Value |
|-------|-------|
| **Date** | 2026-02-21 |
| **Status** | **Superseded** by Pipeline v2 binary routing (D7) |
| **Scope** | lexe-core / agent pipeline |
| **ADR file** | `lexe-docs/knowhow/adr/ADR-D1-unified-classifier.md` |

### Context

The original pipeline used two separate LLM calls for query classification:
1. **LEXORC Complexity Classifier** (`classifier.py`): FLASH/STANDARD/DEEP/STRATEGIC (~1.5s)
2. **LEGIS Intent Detector** (`intent_detector.py`): PipelineLevel + ScenarioType (~1.5s)

This caused ~2.5-3s latency, redundant inference, potential classifier disagreements, and doubled cost.

### Decision

Unify both classifiers into a single LLM call (`_pre_intent_route` in `customer_router.py`) that determines route (`toolloop`/`legis`/`lexorc`), complexity level, legal references, and scenario type in one JSON response. Rule-based pre-filter handles ~30% of queries (short non-legal messages) with 0ms latency.

### Outcome

Latency reduced from ~3s to ~1.5s. 50% cost reduction on classification calls. Single source of truth for routing.

### Superseded By

Pipeline v2 (D7) replaced the unified classifier with a deterministic binary decomposer: CHAT goes to "toolloop", all legal queries go to "legal". The LLM-based complexity classification was replaced by depth budget assignment.

---

## ADR-D2: Differentiated Semaphores for Tool Rate Limiting

| Field | Value |
|-------|-------|
| **Date** | 2026-02-21 |
| **Status** | **Active** |
| **Scope** | lexe-core / tools / executor |
| **ADR file** | `lexe-docs/knowhow/adr/ADR-D2-semaphores-rate-limiting.md` |

### Context

LEGIS pipeline executes multiple tools in parallel during research. Without rate limiting: upstream overload on government APIs, resource exhaustion, cascading failures, unfair scheduling between fast KB lookups and slow web searches.

### Decision

Per-request asyncio semaphores differentiated by tool category:

| Category | Limit | Tools |
|----------|-------|-------|
| **official** | 6 | `normattiva_search`, `eurlex_search` |
| **certified** | 4 | `infolex_search`, `kb_search`, `kb_normativa_search` |
| **web** | 3 | `lex_search`, `lex_search_enriched` |

Each semaphore is per-request (not global), so concurrent users each get independent limits.

### Key files

- `lexe-core/src/lexe_core/tools/executor.py`
- `lexe-core/src/lexe_core/tools/policy.py`

---

## ADR-D3: LEXORC Roles Managed via Database

| Field | Value |
|-------|-------|
| **Date** | 2026-02-21 |
| **Status** | **Active** (roles reused by Pipeline v2 and model catalog) |
| **Scope** | lexe-core / agent / config |
| **ADR file** | `lexe-docs/knowhow/adr/ADR-D3-lexorc-roles-db.md` |

### Context

LEXORC agents (NormAgent, CaseLawAgent, DoctrineAgent, Auditor, Synthesizer) each needed model configuration. Initially hardcoded in Python files.

### Decision

Store agent role configurations in the database (`model_role_defaults` table for global defaults, `tenant_model_roles` for per-tenant overrides). Enables runtime model changes without redeployment, per-tenant customization, A/B testing, and audit trail.

Configuration per role includes: model alias, temperature, max_tokens, system_prompt_version, enabled flag.

### Fallback chain

1. `tenant_model_roles` (per-tenant override)
2. `model_role_defaults` (global default)
3. `settings.lexorc_*_model` (environment variables)
4. Hardcoded defaults in code

### Current State

LEXORC itself is deprecated (D11), but the `model_role_defaults` and `tenant_model_roles` tables remain active. Pipeline v2 and the model catalog system use them for configuring the 6 LLM aliases (lexe-fast, lexe-primary, lexe-complex, lexe-verifier, lexe-frontier, lexe-embedding).

### Key files

- `lexe-core/src/lexe_core/agent/config_utils.py` (`apply_config_override()`)
- DB migration: `017_rename_manus_to_lexorc.sql`

---

## ADR-D4: Feature Flag ff_legis_agent Default True

| Field | Value |
|-------|-------|
| **Date** | 2026-02-21 |
| **Status** | **Active** |
| **Scope** | lexe-core / gateway / feature flags |
| **ADR file** | `lexe-docs/knowhow/adr/ADR-D4-ff-legis-default-true.md` |

### Context

LEGIS Agent was initially gated behind `ff_legis_agent=False` for gradual rollout. After Sprint 8: routing accuracy >85% on gold set, P95 latency <15s for STANDARD, zero data loss in 2 weeks staging.

### Decision

Set `ff_legis_agent=True` as default. All new deployments have LEGIS enabled. The flag remains for emergency rollback (`LEXE_FF_LEGIS_AGENT=false` + container restart).

### Trade-offs

Higher resource usage (3-8 tool calls per query vs. 0-1 for ToolLoop), higher LLM cost, 5-15s latency vs. 2-5s for ToolLoop. Quality improvement justifies the cost.

### Rollback

Per-request check. Set env var to `false` and restart. In-flight requests complete; new requests fall back to ToolLoop. Zero downtime.

---

## ADR-D5: Three-Level Confidence Spectrum

| Field | Value |
|-------|-------|
| **Date** | 2026-02-21 |
| **Status** | **Active** |
| **Scope** | lexe-core / agent / scoring + verifier |
| **ADR file** | `lexe-docs/knowhow/adr/ADR-D5-confidence-spectrum.md` |

### Context

Numeric 0-100 confidence scores were hard to interpret, falsely precise, and provided no actionable guidance for legal practitioners.

### Decision

Three-band confidence spectrum:

| Level | Score Range | Badge | Criteria |
|-------|-------------|-------|----------|
| **VERIFIED** | >= 45 | Green | Official source confirmed (Normattiva/EUR-Lex), certified jurisprudence, cross-source coherence passed |
| **CAUTION** | 25 -- 44 | Yellow | Some sources verified but not all, minor inconsistencies, temporal scope unclear, single-source |
| **LOW_CONFIDENCE** | < 25 | Red | No official confirmation, conflicts detected, verifier flagged errors, fallback/timeout during research |

### Scoring Rules

- VERIFIED requires at least one official source AND no verifier errors
- Jurisprudence absence is NOT a penalty (informational, not verification gate)
- Hard cap at 60 if primary citation failed
- Floor at 40% minimum score
- Weighted factors: source (0.30), vigenza (0.25), link (0.20), cross-validation (0.15), text quality (0.10)

### Key files

- `lexe-core/src/lexe_core/agent/scoring.py` (`classify_authority()`, `_AUTHORITY_WEIGHTS`)
- `lexe-webchat/src/components/CitationBadge.tsx`

---

## ADR-D6: Self-Correction Max 2 Retries with 15s Timeout

| Field | Value |
|-------|-------|
| **Date** | 2026-02-21 |
| **Status** | **Active** |
| **Scope** | lexe-core / agent / pipeline |
| **ADR file** | `lexe-docs/knowhow/adr/ADR-D6-self-correction-max-2.md` |

### Context

The LEGIS verifier can trigger re-research cycles when it detects broken citations, vigenza mismatches, or coherence issues. Without limits: infinite loops, escalating latency, token waste.

### Decision

Maximum 2 self-correction retries, 15-second wall-clock timeout per retry.

```
Attempt 1 (initial): Planner -> Researcher -> Verifier
  If issues:
Attempt 2 (retry 1): Re-research specific items -> Verifier
  If still issues:
Attempt 3 (retry 2): Re-research specific items -> Verifier
  Regardless: Proceed to Synthesizer with best available evidence
```

Total maximum self-correction time: 30 seconds. Total maximum pipeline time (with retries): ~45 seconds for COMPLEX queries.

### Observed Metrics (Staging)

- 70% queries: 0 retries (verifier passes first attempt)
- 25% queries: 1 retry
- 4% queries: 2 retries
- 1% would benefit from 3+ retries (accepted as LOW_CONFIDENCE)

After max retries, unresolved issues are surfaced via confidence badge (CAUTION or LOW_CONFIDENCE) and a transparency note.

---

## D7: Pipeline v2 -- Binary Routing

| Field | Value |
|-------|-------|
| **Date** | 2026-03-12 |
| **Status** | **Active** |
| **Scope** | lexe-core / orchestration / strategies |

### Context

The original routing used a 4-strategy model (ToolLoop, LEGIS, LEXORC, Super Tool) with LLM-based intent classification selecting the strategy. This was complex, expensive (LLM call per classification), and the 4 strategies had significant overlap.

### Decision

Replace with deterministic binary routing via `decomposer.py`:
- **CHAT** (greetings, generic questions, non-legal) -> `"toolloop"` strategy
- **All legal queries** (normativa, giurisprudenza, parere, contratto, etc.) -> `"legal"` strategy

The new `LegalStrategy` in `strategies/legal.py` wraps `unified_pipeline.py`, which itself wraps the existing legis_agent_pipeline with depth-based configuration.

### Supersedes

- D1 (Unified Intent Classifier) -- no longer needed, routing is deterministic
- 4-strategy model (toolloop/legis/lexorc/super_tool) collapsed to 2 (toolloop/legal)

### Key files

- `lexe-core/src/lexe_core/agent/decomposer.py`
- `lexe-core/src/lexe_core/orchestration/strategies/legal.py` (`LegalStrategy`)
- `lexe-core/src/lexe_core/agent/unified_pipeline.py`

---

## D8: All Models to Gemini 3 Flash Preview

| Field | Value |
|-------|-------|
| **Date** | 2026-03-12 (Sprint 9) |
| **Status** | **Active** |
| **Scope** | lexe-infra / litellm / config |

### Context

LLM benchmark Phase 1 (2026-02-23): 6 runs, 11 models, 2 judges (Sonnet 4 + GPT-5-mini), peer-review methodology. Evaluated across all LEXE pipeline roles.

### Decision

All 6 primary model aliases mapped to `google/gemini-3-flash-preview`, differentiated by `reasoning_effort` in the LiteLLM catalog config JSONB:

| Alias | reasoning_effort | max_tokens | Use Case |
|-------|-----------------|------------|----------|
| `lexe-fast` | medium | 4,096 | Intent detection, follow-up, classification |
| `lexe-primary` | medium | 16,384 | Chat, synthesis, research |
| `lexe-complex` | medium | 16,384 | Reasoning, planning |
| `lexe-verifier` | low | 8,192 | Fast evidence verification |
| `lexe-frontier` | high | 16,384 | Maximum writing quality, complex synthesis |
| `lexe-embedding` | -- | -- | text-embedding-3-small (unchanged) |

### reasoning_effort Flow

`config_utils.py:apply_config_override()` reads `reasoning_effort` from the catalog config JSONB and maps it into the LiteLLM request payload.

### Support Models

Additional models available for specific use cases: `gpt-5-mini` (reasoning=low, CRITICAL), `qwen3.5-plus` (reasoning=medium), `deepseek-v3.2`, `gem-flash-reason-med`.

### Warning: GPT-5-mini

GPT-5-mini MUST have `reasoning_effort=low`. Without it, costs spike from 1.00 to 4.75. NEVER set `max_tokens` on GPT-5-mini.

---

## D9: Depth Budgets (MINIMAL/STANDARD/DEEP)

| Field | Value |
|-------|-------|
| **Date** | 2026-03-15 |
| **Status** | **Active** |
| **Scope** | lexe-core / agent / depth_budget |

### Context

The previous PipelineLevel enum (DIRECT/SIMPLE/STANDARD/COMPLEX) was tightly coupled to the LEGIS intent detector. Pipeline v2 needed a simpler, more explicit resource allocation mechanism.

### Decision

Three depth levels with explicit budgets:

| Depth | Max Tool Calls | Max LLM Calls | Use Case |
|-------|---------------|---------------|----------|
| **MINIMAL** | 3 | 2 | Quick lookup, single-article reference |
| **STANDARD** | 8 | 5 | Typical legal research question |
| **DEEP** | 15 | 8 | Complex multi-source analysis, pareri |

Depth is assigned by the decomposer based on query complexity heuristics (keyword analysis, reference count, question structure). The `DepthBudget` class in `agent/depth_budget.py` tracks consumption and enforces limits.

### Multi-Agent Research Budgets

When `ff_multi_agent_research` is enabled, budgets are expressed in total agent steps:
- SIMPLE: 12 steps
- STANDARD: 22 steps
- COMPLEX: 35 steps

### Key files

- `lexe-core/src/lexe_core/agent/depth_budget.py` (`DepthBudget`)
- `lexe-core/src/lexe_core/agent/unified_pipeline.py`
- `lexe-core/src/lexe_core/agent/decomposer.py`

---

## D10: Super Tool Deprecation

| Field | Value |
|-------|-------|
| **Date** | 2026-03-12 |
| **Status** | **Active** (disabled from routing, code retained) |
| **Scope** | lexe-core / orchestration / strategies |

### Decision

The Super Tool strategy (`super_tool_agent`) is disconnected from the routing layer. The `ff_super_tool` feature flag defaults to `false`. No queries are routed to the Super Tool strategy.

### Code Retention Policy

Per explicit feedback (`feedback_no_delete_supertool.md`): Super Tool files are NEVER deleted. They are retained in the codebase for potential future reuse. The strategy class, agent code, and related tests remain in their original locations but are unreachable from production routing.

### Affected Files (retained, not deleted)

- `lexe-core/src/lexe_core/orchestration/strategies/super_tool.py`
- `lexe-core/src/lexe_core/agent/super_tool_agent.py`
- Related test files

---

## D11: LEXORC Deprecation

| Field | Value |
|-------|-------|
| **Date** | 2026-02-21 |
| **Status** | **Active** (disabled, code retained) |
| **Scope** | lexe-core / agent / LEXORC |

### Decision

LEXORC (LEXe ORChestrator) advanced pipeline is disabled via `ff_lexorc_enabled=false`. The feature flag has never been set to `true` in production.

### Code Retention

LEXORC code remains in the codebase:
- `LexeOrchestrator`, `LexorcAuditor`, `LexorcSynthesizer` classes
- SSE event type `lexorc_phase`
- Endpoint `/customer/lexorc/stream`
- DB table `core.lexorc_sessions`
- Valkey namespace `lexorc:bb:`

The database role configuration tables (`model_role_defaults`, `tenant_model_roles`) created for LEXORC (ADR-D3) remain active and are now used by the model catalog system.

### Migration path

`017_rename_manus_to_lexorc.sql` renamed all MANUS references to LEXORC. The original MANUS name is no longer referenced anywhere in the codebase.

---

## D12: GDPR Consent System

| Field | Value |
|-------|-------|
| **Date** | 2026-03-19 |
| **Status** | **Active** (FF gated, default false) |
| **Scope** | lexe-core + lexe-webchat / consent |

### Context

LEXE processes personal data (conversations, legal queries) and must comply with GDPR (Reg. UE 2016/679) and Italian Privacy Code (D.Lgs. 196/2003 mod. D.Lgs. 101/2018).

### Decision

4-phase GDPR consent system behind feature flag `ff_consent_required` (default `false`):

| Phase | Description | Status |
|-------|-------------|--------|
| 1. Informativa | Versioned Terms of Service + Privacy Policy | Done |
| 2. Prova Consenso | IP/User-Agent capture, audit trail, CSV export for DPO | Done |
| 3. Disclaimer | AI disclaimer (art. 1229 c.c.), no-training, profiling controls | Done |
| 4. Diritti | Data export (art. 15/20), deletion request (art. 17), info panel | Done |

### Database

- `core.consent_terms`: versioned terms (free/paid/demo plan types)
- `core.user_consents`: per-user consent records with IP, User-Agent, timestamps
- RLS policies for tenant isolation
- Unique constraint: `(user_sub, terms_id)`

### Consent Toggles

| Toggle | Required | Default | Description |
|--------|----------|---------|-------------|
| Terms of Service | Yes (blocks submit) | -- | Must accept to use platform |
| Metrics | No | On | Anonymous aggregate technical metrics |
| Training | No | Off | AI training (not exposed in UI, always false) |
| Pseudonymized analysis | No | On | KPI with irreversible hash |

### Endpoints

**Customer** (under `/api/v1/consent/`):
- `GET /status` -- check if user has accepted current terms
- `GET /terms` -- retrieve HTML terms body + existing preferences
- `POST /accept` -- accept terms, capture IP/UA/timestamp
- `PATCH /preferences` -- update optional toggles without re-accepting

**Admin** (under `/api/v1/admin/consent-audit/`):
- `GET /` -- JSON audit records with filters (user_sub, date_from, date_to)
- `GET /export` -- CSV download for DPO audit trail

### Frontend

- `ConsentOverlay.tsx`: full-screen modal on first access or terms update
- `PrivacyTab.tsx`: profile tab with GDPR accordions, data export, deletion request

### Key files

- `lexe-core/migrations/036_consent_terms.sql`
- `lexe-core/src/lexe_core/gateway/consent_router.py`
- `lexe-core/src/lexe_core/admin/routers/consent_audit.py`
- `lexe-webchat/src/components/auth/ConsentOverlay.tsx`
- `lexe-webchat/src/components/profile/PrivacyTab.tsx`
- `lexe-docs/gdpr-consent-system.md` (full technical documentation)

### Legal References

- Reg. UE 2016/679 (GDPR): Art. 5, 6, 7, 12-22
- D.Lgs. 196/2003 mod. D.Lgs. 101/2018
- Art. 1229 c.c. (limitation of liability clauses)

---

*Last updated: 2026-03-19*
