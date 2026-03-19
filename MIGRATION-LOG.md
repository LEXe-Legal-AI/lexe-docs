# Documentation Migration Log

> Riorganizzazione documentazione LEXE — Fase Lancio
> Branch: `docs-reorg-launch`
> Data inizio: 2026-03-19

## Legend

| Action | Meaning |
|--------|---------|
| MOVE | File spostato |
| RENAME | File rinominato |
| MERGE | Contenuto fuso in altro doc |
| ARCHIVE | Spostato in archived/obsolete |
| DEPRECATE | Marcato deprecated (resta in posizione) |
| CREATE | Nuovo documento creato |
| DELETE | File eliminato |

---

## Fase -1 — Safety Net (2026-03-19)

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | CREATE | - | `lexe-docs/internal/TREE-SNAPSHOT-2026-03-19.txt` | Snapshot 1555 file paths |
| 2026-03-19 | CREATE | - | `lexe-docs/DOC-CATALOG-BACKUP-2026-03-19.md` | Backup catalogo |
| 2026-03-19 | CREATE | - | `lexe-docs/MIGRATION-LOG.md` | Questo file |

## Fase 0 — Archiviazione Fisica

### Tier 3: archived-obsolete

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | ARCHIVE | `db.md` | `archived/obsolete/` | DB doc superato |
| 2026-03-19 | ARCHIVE | `lexe-8d8138ed.md` | `archived/obsolete/` | Doc generato obsoleto |
| 2026-03-19 | ARCHIVE | `STARTUP.md` | `archived/obsolete/` | Procedura startup vecchia |
| 2026-03-19 | ARCHIVE | `POSTSTART.md` | `archived/obsolete/` | Post-startup vecchio |
| 2026-03-19 | ARCHIVE | `MEMO_STATUS_2026-02-21.md.html` | `archived/obsolete/` | Duplicato HTML |

### Tier 2: reference-historical

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | MOVE | `lexe-docs/REFACTOR-STATUS.md` | `lexe-docs/reference-historical/` | Decisioni refactor pre-v2 |
| 2026-03-19 | MOVE | `lexe-docs/LEXe_MultiProposal_Synthesis.md` | `lexe-docs/reference-historical/` | Confronto 5 analisi AI |
| 2026-03-19 | MOVE | `lexe-docs/PROMPT-OPTIMIZATION-PROPOSAL.md` | `lexe-docs/reference-historical/` | Proposta integrata in v2.1 |
| 2026-03-19 | MOVE | `lexe-docs/system-prompts/lexe-legal-assistant-PROPOSED.md` | `lexe-docs/reference-historical/` | Prompt proposto |
| 2026-03-19 | MOVE | `lexe-docs/mappa-lexe-docs.md` | `lexe-docs/reference-historical/` | Mappa vecchia |
| 2026-03-19 | MOVE | `lexe-docs/memory-lite.md` | `lexe-docs/reference-historical/` | Memory spec superata |
| 2026-03-19 | MOVE | `LEXE_STATUS_2026-02-21.md.html` | `lexe-docs/reference-historical/` | Kimera report datato |
| 2026-03-19 | MOVE | `LEXE_SPRINT-REPORT_2026-02-21.md.html` | `lexe-docs/reference-historical/` | Kimera sprint report |
| 2026-03-19 | MOVE | `LEXE_MULTI_TERMINAL_ORCHESTRATION.md` | `lexe-docs/reference-historical/` | Metodologia multi-terminale |
| 2026-03-19 | MOVE | `velvet-squishing-boole.md` | `lexe-docs/reference-historical/` | Piano auto-improve |
| 2026-03-19 | MOVE | `elegant-finding-horizon.md` | `lexe-docs/reference-historical/` | Piano LEGIS agent |
| 2026-03-19 | MOVE | `skunkworks.md` | `lexe-docs/reference-historical/` | R&D active learning |
| 2026-03-19 | MOVE | `plan spec and modality.md` | `lexe-docs/reference-historical/` | Spec hardening |

### Tier 2: deprecate-in-place

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | DEPRECATE | `lexe-docs/SUPER-TOOL-EXPERIMENTAL.md` | (in place) | Policy no-delete, status→deprecated |

## Fase 1.5 — Adjudication

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | CREATE | - | `lexe-docs/internal/ADJUDICATION-RESULTS.md` | Consolidated findings from 15 agents |

## Fase 2 — Generazione Documentazione

### Internal (10 documenti)

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | CREATE | - | `internal/ARCHITECTURE-V2.md` | 3-layer arch, 13 containers, 29 FF |
| 2026-03-19 | CREATE | - | `internal/PIPELINE-REFERENCE.md` | PRVS pipeline, depth budgets, multi-agent |
| 2026-03-19 | CREATE | - | `internal/TOOL-CATALOG.md` | 8 tools, purity, rate limiting, circuit breaker |
| 2026-03-19 | CREATE | - | `internal/SECURITY-POSTURE.md` | HMAC, RLS, RBAC, GDPR, known gaps |
| 2026-03-19 | CREATE | - | `internal/RUNBOOK-V2.md` | Deploy/rollback 13 containers + migrations |
| 2026-03-19 | CREATE | - | `internal/OBSERVABILITY.md` | Prometheus, Grafana, Jaeger, Langfuse, EventSink |
| 2026-03-19 | CREATE | - | `internal/TENANT-MODEL.md` | Multitenancy, plans, limits, provisioning |
| 2026-03-19 | CREATE | - | `internal/DECISION-LOG.md` | 12 ADR/decisions cronologiche |
| 2026-03-19 | CREATE | - | `internal/SYSTEM-BOUNDARIES.md` | 24 systems, trust levels, SLA matrix |
| 2026-03-19 | CREATE | - | `internal/DATA-FLOWS-GDPR.md` | GDPR flows, retention, consent, Art. 15-21 |
| 2026-03-19 | CREATE | - | `internal/CONFIDENCE-PIPELINE.md` | Scoring formula, 3-band, authority, verification |

### Public (7 documenti)

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | CREATE | - | `public/FEATURES.md` | 10 confirmed features |
| 2026-03-19 | CREATE | - | `public/BENCHMARK-SUMMARY.md` | G-Eval, marketing-safe claims |
| 2026-03-19 | CREATE | - | `public/TECHNOLOGY.md` | Stack senza segreti |
| 2026-03-19 | CREATE | - | `public/API-OVERVIEW.md` | Customer-facing endpoints |
| 2026-03-19 | CREATE | - | `public/TRUST-SECURITY.md` | Compliance, isolation, GDPR |
| 2026-03-19 | CREATE | - | `public/USE-CASES.md` | 4 segmenti, scenari concreti |
| 2026-03-19 | CREATE | - | `public/WHY-LEXE.md` | 6 differenziatori |

## Fase 3 — README & Index

| Timestamp | Action | From | To | Note |
|-----------|--------|------|----|------|
| 2026-03-19 | CREATE | - | `lexe-docs/README.md` | Indice navigabile documentazione |
