---
owner: platform-team
audience: internal
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: on-change
sensitivity: internal
last_verified: 2026-03-19
---

# Adjudication Results — Documentation Reorg 2026-03-19

> Consolidated findings from 15 analysis agents (4 verification + 11 mapping).
> Rule: where doc and code diverge, **code wins**.

---

## 1. Document Accuracy Matrix

| Document | Agent | Accuracy | Action |
|----------|-------|----------|--------|
| ARCHITECTURE-REFERENCE.md | A1 | 95% | patch (missing migrations, routers) |
| orchestration-v2-architecture.md | A1 | 70% | **rewrite** (outdated: no LegalStrategy, no DepthLevel) |
| agentic-workflow.md | A1 | 85% | patch (missing depth budget, gap reporter) |
| panels.md | A1 | 95% | patch (6 new routers) |
| WEBGUI-ADMIN-DESIGN.md | A1 | 60% | **deprecate** (design doc, impl diverges) |
| RUNBOOK-deploy.md | A2 | 65% | **rewrite** (missing containers, ports, migrations) |
| RUNBOOK-rollback.md | A2 | 60% | **rewrite** (missing admin/temporal/tools rollback) |
| LOGTO-MULTITENANCY-RUNBOOK.md | A2 | 85% | patch (URL fix: auth-admin → logto-admin) |
| DEPLOY-STAGE.md | A2 | 90% | patch (minor health check gaps) |
| RLS-MULTITENANCY.md | A3 | 60% | **rewrite** (naming mismatch, table coverage) |
| HARDENING-CONTRACT.md | A3 | 85% | patch (degradation codes) |
| gdpr-consent-system.md | A3 | 95% | keep (nearly complete) |
| ADR D1-D6 | A3 | 90% | patch (D5 thresholds, D1 superato) |
| KB docs (all) | A4 | 73% | **rewrite** (stats +111% articles, missing migrations) |

## 2. Source-of-Truth Assignments

One authoritative document per area. Everything else → reference-historical or deprecated.

| Area | Source of Truth | Location |
|------|----------------|----------|
| Architecture (overall) | `internal/ARCHITECTURE-V2.md` | NEW — replaces ARCHITECTURE-REFERENCE |
| Pipeline & Orchestration | `internal/PIPELINE-REFERENCE.md` | NEW — replaces orchestration-v2-arch + agentic-workflow |
| Tool Catalog | `internal/TOOL-CATALOG.md` | NEW — from B2 findings |
| Auth & Identity | `internal/TENANT-MODEL.md` | NEW — from B3 findings |
| Security & GDPR | `internal/SECURITY-POSTURE.md` | NEW — replaces RLS-MULTI + HARDENING |
| Deploy & Ops | `internal/RUNBOOK-V2.md` | NEW — replaces both RUNBOOK-deploy + rollback |
| KB & Legal Data | `internal/KB-REFERENCE.md` | NEW — replaces all KB docs |
| Memory System | `internal/MEMORY-REFERENCE.md` | NEW — from B8 findings |
| Observability | `internal/OBSERVABILITY.md` | NEW — from B10 findings |
| Quality & Scoring | `internal/CONFIDENCE-PIPELINE.md` | NEW — from B4 findings |
| Admin Panel | CLAUDE.md (lexe-admin) | EXISTS — accurate per B6 |
| Webchat Frontend | CLAUDE.md (lexe-webchat) | EXISTS — needs update per B5 |
| Benchmark | tests/benchmarks/README.md | EXISTS — accurate per B11 |
| LLM Selection | lexe-docs/llm-selection.md | EXISTS — accurate |
| Consent System | lexe-docs/gdpr-consent-system.md | EXISTS — accurate per A3 |

## 3. Resolved Conflicts (Code Wins)

| Conflict | Doc Says | Code Says | Resolution |
|----------|----------|-----------|------------|
| Core port | 8000 | 8100 | **8100** |
| RLS function | `core.current_tenant_id()` | `core.fn_get_tenant_id()` | **fn_ prefix** |
| RLS config var | `app.current_tenant_id` | `app.tenant_id` | **app.tenant_id** |
| Protected tables | 6 | 12+ (migration 039) | **12+** |
| Strategies | 4 (legis/lexorc/super_tool/toolloop) | 2 active (legal/toolloop) + 3 deprecated | **binary routing** |
| Pipeline levels | 5 (PipelineLevel) | 3 (DepthLevel: MINIMAL/STANDARD/DEEP) | **DepthLevel** |
| Event types | 17 | 13 core + PipelineDataEvent | **13 core** |
| Admin routers | 11 | 18 | **18** |
| Normativa articles | 6,335 | 13,397 | **13,397** |
| Embeddings | 3,300 | 44,253 | **44,253** |
| Massime | 41,437 | 46,767 | **46,767** |
| Works | 69 | 95 | **95** |
| Migrations (core) | 13 | 035+ | **035+** |
| Migrations (KB) | ~20 | 26 files (001-080) | **26** |
| Logto admin URL (stage) | auth-admin.stage | logto-admin.stage | **logto-admin** |

## 4. Commercial Claims Review

Based on B11 (Benchmark) findings:

### Verified (safe for public docs)
- G-Eval automated benchmark suite with independent judge (GPT-5.2)
- 11 models tested, 6 runs, peer-review validation
- 5-dimension scoring (accuracy, completeness, citations, Italian, structure)
- Current production: Gemini 3 Flash suite (all 6 aliases)
- Multi-agent research with 3-wave parallel execution
- 3-band confidence system (VERIFIED/CAUTION/LOW_CONFIDENCE)
- GDPR consent system with audit trail
- Hybrid search (dense + sparse + RRF fusion)
- 38K+ Cassazione massime, 13K+ legislative articles

### NOT Safe (avoid in public docs)
- Specific model comparison rankings (internal benchmark, small n)
- Quantitative accuracy percentages without context
- Claims about reasoning tokens/inference cost (evolving)
- "Outperforms industry standard" (no external validation)
- Any feature gated by FF not enabled in production

## 5. Documents to Generate (Fase 2)

### Internal (10 docs)
1. ARCHITECTURE-V2.md
2. PIPELINE-REFERENCE.md
3. TOOL-CATALOG.md
4. SECURITY-POSTURE.md
5. RUNBOOK-V2.md
6. DECISION-LOG.md
7. SYSTEM-BOUNDARIES.md
8. TENANT-MODEL.md
9. OBSERVABILITY.md
10. DATA-FLOWS-GDPR.md

### Public (7 docs)
1. FEATURES.md
2. BENCHMARK-SUMMARY.md
3. TECHNOLOGY.md
4. API-OVERVIEW.md
5. TRUST-SECURITY.md
6. USE-CASES.md
7. WHY-LEXE.md
