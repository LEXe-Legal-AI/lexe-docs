---
owner: platform-team
audience: internal
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: monthly
sensitivity: internal
last_verified: 2026-03-19
---

# LEXE Documentation Hub

> Documentazione tecnica per **LEXe Legal AI Platform**
> Riorganizzata 2026-03-19 — Branch `docs-reorg-launch`

---

## Navigation

### Internal Documentation (Confidential/Internal)

| Document | Description | Status |
|----------|-------------|--------|
| [ARCHITECTURE-V2.md](internal/ARCHITECTURE-V2.md) | Platform architecture: 13 containers, 3-layer orchestration, network topology | Source of Truth |
| [PIPELINE-REFERENCE.md](internal/PIPELINE-REFERENCE.md) | PRVS pipeline, binary routing, depth budgets, multi-agent research | Source of Truth |
| [TOOL-CATALOG.md](internal/TOOL-CATALOG.md) | 8 legal tools: parameters, purity, rate limiting, circuit breaker | Source of Truth |
| [SECURITY-POSTURE.md](internal/SECURITY-POSTURE.md) | HMAC, RLS (12+ tables), RBAC (7 roles), GDPR, known gaps | Source of Truth |
| [RUNBOOK-V2.md](internal/RUNBOOK-V2.md) | Deploy/rollback 13 containers, migrations, health checks | Source of Truth |
| [OBSERVABILITY.md](internal/OBSERVABILITY.md) | Prometheus (72 rules), Grafana, Jaeger, Loki, Langfuse, EventSink | Source of Truth |
| [TENANT-MODEL.md](internal/TENANT-MODEL.md) | Multitenancy: plans, limits, provisioning, RBAC, FF per-tenant | Source of Truth |
| [CONFIDENCE-PIPELINE.md](internal/CONFIDENCE-PIPELINE.md) | Scoring formula, 3-band thresholds, authority weights, verification | Source of Truth |
| [DECISION-LOG.md](internal/DECISION-LOG.md) | 12 architecture decisions (ADR D1-D6 + Sprint 9-10 decisions) | Source of Truth |
| [SYSTEM-BOUNDARIES.md](internal/SYSTEM-BOUNDARIES.md) | 24 systems: owned vs external, trust levels, SLA dependencies | Source of Truth |
| [DATA-FLOWS-GDPR.md](internal/DATA-FLOWS-GDPR.md) | GDPR data flows, retention, consent, Art. 15-21, pseudonymization | Source of Truth |
| [ADJUDICATION-RESULTS.md](internal/ADJUDICATION-RESULTS.md) | Doc reorg analysis: accuracy matrix, conflicts resolved, claims review | Reference |

### Public Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [FEATURES.md](public/FEATURES.md) | Platform features: multi-agent research, confidence, citations, tools | Customers |
| [PLATFORM-OVERVIEW.md](public/PLATFORM-OVERVIEW.md) | End-to-end platform narrative: AI-first UX, architecture, differentiation | Marketing / Sales |
| [BENCHMARK-SUMMARY.md](public/BENCHMARK-SUMMARY.md) | G-Eval benchmark: 11 models, 5 dimensions, marketing-safe claims | Marketing |
| [TECHNOLOGY.md](public/TECHNOLOGY.md) | Tech stack overview (no secrets) | Partners |
| [API-OVERVIEW.md](public/API-OVERVIEW.md) | Customer-facing API reference (chat, conversations, memory) | Developers |
| [TRUST-SECURITY.md](public/TRUST-SECURITY.md) | Compliance: tenant isolation, GDPR, TLS, audit | Compliance |
| [USE-CASES.md](public/USE-CASES.md) | 4 segments: studio piccolo/medio, corporate, partner | Sales |
| [WHY-LEXE.md](public/WHY-LEXE.md) | 6 differentiators vs alternatives | Marketing |

### Active Reference Documents (Pre-existing, verified)

| Document | Description | Status |
|----------|-------------|--------|
| [STATUS.md](STATUS.md) | Platform status, sprint checklist | Active |
| [llm-selection.md](llm-selection.md) | LLM benchmark results, model selection | Source of Truth |
| [gdpr-consent-system.md](gdpr-consent-system.md) | GDPR consent system (4 phases) | Source of Truth |
| [orchestration-v2-test-report.md](orchestration-v2-test-report.md) | V2 test report, FF per-tenant | Source of Truth |
| [DEPLOY-STAGE.md](DEPLOY-STAGE.md) | Staging deploy workflow | Active |
| [tenant-cleanup.md](tenant-cleanup.md) | Tenant deletion procedure | Active |
| [system-prompts/lexe-legal-assistant-CURRENT.md](system-prompts/lexe-legal-assistant-CURRENT.md) | Active system prompt v2.1 | Source of Truth |

### ADR & Knowhow

| Document | Description |
|----------|-------------|
| [knowhow/adr/ADR-D2-semaphores-rate-limiting.md](knowhow/adr/ADR-D2-semaphores-rate-limiting.md) | Tool rate limiting (active) |
| [knowhow/adr/ADR-D5-confidence-spectrum.md](knowhow/adr/ADR-D5-confidence-spectrum.md) | 3-band confidence (active) |
| [knowhow/adr/ADR-D6-self-correction-max-2.md](knowhow/adr/ADR-D6-self-correction-max-2.md) | Self-correction retries (active) |
| [knowhow/runbooks/](knowhow/runbooks/) | Legacy runbooks (superseded by RUNBOOK-V2) |

### Historical & Archived

| Location | Content |
|----------|---------|
| [reference-historical/](reference-historical/) | 13 docs with historical/decision value (plans, proposals, reports) |
| [DOC-CATALOG.md](DOC-CATALOG.md) | Full document census (2026-03-19, 160+ docs) |
| [MIGRATION-LOG.md](MIGRATION-LOG.md) | Log of all moves, creates, archives in this reorg |

---

## Repository Map

| Repository | Description | Port | Key Doc |
|------------|-------------|------|---------|
| **lexe-core** | API Gateway, Auth, Orchestration, Pipeline | 8100 | [ARCHITECTURE-V2](internal/ARCHITECTURE-V2.md) |
| **lexe-webchat** | Customer Chat Interface (React) | 3013 | CLAUDE.md |
| **lexe-admin** | Admin Panel (React) | 3014 | CLAUDE.md |
| **lexe-memory** | Memory L0-L4, RAG, Graph | 8103 | [ARCHITECTURE-V2](internal/ARCHITECTURE-V2.md) |
| **lexe-max** | Legal KB (PostgreSQL + pgvector + AGE) | 5436 | [TOOL-CATALOG](internal/TOOL-CATALOG.md) |
| **lexe-tools-it** | Legal Tools API (Normattiva, EUR-Lex) | 8021 | [TOOL-CATALOG](internal/TOOL-CATALOG.md) |
| **lexe-infra** | Docker Compose, Traefik, Monitoring | - | [RUNBOOK-V2](internal/RUNBOOK-V2.md) |
| **lexe-privacy** | PII Detection (spaCy) | - | [DATA-FLOWS-GDPR](internal/DATA-FLOWS-GDPR.md) |
| **lexe-orchestrator** | ORCHIDEA Pipeline (disabled) | 8102 | Deprecated |
| **lexe-docs** | This documentation repo | - | This file |

---

## Production URLs

| Environment | Webchat | API | Auth | Admin | LLM |
|-------------|---------|-----|------|-------|-----|
| **Production** | https://ai.lexe.pro | https://api.lexe.pro | https://auth.lexe.pro | https://admin.lexe.pro | https://llm.lexe.pro |
| **Staging** | https://stage-chat.lexe.pro | https://api.stage.lexe.pro | https://auth.stage.lexe.pro | https://admin.stage.lexe.pro | https://llm.stage.lexe.pro |

---

## Quick Deploy Reference

```bash
# Staging
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml up -d

# NEVER without override (missing Traefik labels + wrong env vars)
```

See [RUNBOOK-V2.md](internal/RUNBOOK-V2.md) for complete procedures.

---

*Last updated: 2026-03-19 — Documentation Reorg (Fase Lancio)*
