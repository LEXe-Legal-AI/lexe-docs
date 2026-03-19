# LEXE Platform - Catalogo Documentazione

> Censimento completo di ~160 documenti. Generato 2026-03-19.
> Status: **fonte_verita** | **da_aggiornare** | **da_verificare** | **obsoleto**

---

## Statistiche

| Status | Count | Azione |
|--------|-------|--------|
| fonte_verita | ~45 | Mantenere, usare come RAG source |
| da_aggiornare | ~40 | Aggiornare con stato attuale |
| da_verificare | ~35 | Review manuale necessaria |
| obsoleto | ~40 | Archiviare o eliminare |

---

## 1. lexe-docs/ (Repo Documentazione Centrale)

### Core & Status

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| STATUS.md | LEXE Platform - Status | 2026-02-01 | 2026-03-19 | 28 | operations | fonte_verita | Stato infra prod/stage, sprint 10 checklist, aggiornato attivamente |
| BACKLOG-2.0.md | LEXE Platform - Backlog 2.0 | 2026-02-21 | 2026-02-22 | 12 | sprint | da_aggiornare | Post-benchmark backlog; BK-002/003 risolti, non aggiornato da sprint 8 |
| BACKLOG.md | LEXE Platform - Backlog | 2026-02-02 | 2026-02-18 | 17 | sprint | da_aggiornare | Backlog originale P0-P3; molti item risolti, altri superati da pipeline v2 |
| LEXE-VISION.md | LEXe - Legal AI Assistant | 2026-02-21 | 2026-02-21 | 18 | vision | fonte_verita | Vision prodotto: principi UX, 4 fasi pipeline, 6 scenari, roadmap trimestrale |
| README.md | LEXE Documentation | 2026-02-01 | 2026-02-01 | 6 | architecture | obsoleto | Index doc con diagramma ORCHIDEA e modelli pre-benchmark (Mistral, Gemini 2.5) |
| DEPLOY-STAGE.md | Deploy LEXE su Staging | 2026-02-06 | 2026-02-10 | 11 | operations | fonte_verita | Workflow deploy staging con regole critiche compose override |
| mappa-lexe-docs.md | Mappa LEXE: lexe-docs | 2026-02-01 | 2026-02-01 | 5 | architecture | obsoleto | Auto-generato, lista solo 2 file su 45+ |

### Architecture

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| ARCHITECTURE-REFERENCE.md | Architecture Reference | 2026-02-21 | 2026-02-21 | 17 | architecture | da_aggiornare | Modelli ora tutti Gemini 3 Flash, manca lexe-admin prod |
| REFACTOR-STATUS.md | Refactor - Stato Completo | 2026-02-21 | 2026-02-21 | 20 | architecture | obsoleto | Piano refactor 6 fasi pre-pipeline-v2; superato da orchestration v2 |
| LEXe_CCC_Implementation_Blueprint.md | Piano Esecutivo di Implementazione | 2026-02-21 | 2026-02-21 | 57 | adr | da_aggiornare | Blueprint v3.2 multi-sprint; molti item implementati, modelli/stack outdated |
| LEXe_MultiProposal_Synthesis.md | Sintesi Multi-Proposta | 2026-02-21 | 2026-02-21 | 24 | adr | obsoleto | Confronto 5 analisi AI; decisioni prese e implementate, valore solo storico |
| architecture/LEXE-REFACTOR-BLUEPRINT.md | Refactor Blueprint | 2026-02-21 | 2026-02-21 | 31 | architecture | da_aggiornare | Pre-orchestration v2; superato da pipeline v2 e depth budgets |
| panels.md | LEXE Admin Panel - Architettura 3 Livelli | 2026-02-21 | 2026-02-21 | 35 | architecture | da_verificare | lexe-admin ora separato; endpoint da verificare |
| WEBGUI-ADMIN-DESIGN.md | WebGUI Admin - Design Document | 2026-02-02 | 2026-02-02 | 31 | architecture | da_aggiornare | Design originale, TMB implementato ma doc non riflette stato reale |
| MEMORY-SYSTEMS.md | I Due Sistemi di Memoria LEXE | 2026-02-15 | 2026-02-21 | 15 | architecture | da_aggiornare | L0-L4 conversazionale + semantica; pre-pipeline v2, ragV3 OFF |
| memory-lite.md | Memory Lite (Quick Fix v1) | 2026-02-03 | 2026-02-03 | 2 | architecture | obsoleto | Superata da MEMORY-SYSTEMS e memory v2 |

### Pipeline & LLM

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| orchestration-v2-architecture.md | Orchestration v2 -- Architettura 3-Layer | 2026-03-12 | 2026-03-19 | 47 | architecture | da_aggiornare | v1.0, Phase 2d+ completate ma header dice pending |
| orchestration-v2-test-report.md | Orchestration V2 - Test Report | 2026-03-12 | 2026-03-19 | 6 | testing | fonte_verita | 7+ test staging, FF per-tenant verificati |
| agentic-workflow.md | LEXE Agentic Workflow | 2026-03-13 | 2026-03-19 | 29 | architecture | da_verificare | Reference completa pipeline LEGIS; verificare allineamento pipeline v2 |
| phase5-report.md | Phase 5 - Experiment Lifecycle | 2026-02-23 | 2026-03-19 | 9 | sprint | fonte_verita | Completato E2E, 1764 LOC, 63 test 100% pass |
| llm-selection.md | LLM Selection & Model Configuration | 2026-03-12 | 2026-03-19 | 6 | llm | fonte_verita | Benchmark 6 run/11 modelli, tutti alias -> Gemini 3 Flash |
| system-prompts/lexe-legal-assistant-CURRENT.md | System Prompt v2.1 (Merged) | 2026-03-02 | 2026-03-19 | 16 | pipeline | fonte_verita | Prompt attivo in DB |
| system-prompts/lexe-legal-assistant-PROPOSED.md | System Prompt PROPOSTO v2.0 | 2026-02-21 | 2026-02-21 | 13 | pipeline | obsoleto | Superato da CURRENT v2.1 |
| PROMPT-OPTIMIZATION-PROPOSAL.md | Proposte Ottimizzazione | 2026-02-06 | 2026-02-21 | 7 | pipeline | obsoleto | Integrato nel system prompt CURRENT v2.1 |
| SUPER-TOOL-EXPERIMENTAL.md | SUPER_TOOL Pipeline Sperimentale | 2026-02-23 | 2026-03-19 | 14 | architecture | obsoleto | Disabilitata dal routing, file mantenuti per policy no-delete |
| TOOL_STABILIZATION_MVP.md | Tool Stabilization + One Search MVP | 2026-02-04 | 2026-02-21 | 11 | api | da_aggiornare | Sprint 1 completato; superato da orchestration v2 SSE adapter |

### Security & Operations

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| gdpr-consent-system.md | GDPR Consent System | 2026-03-19 | 2026-03-19 | 17 | security | fonte_verita | 4 fasi GDPR tutte DONE; schema DB + API completi |
| tenant-cleanup.md | Tenant Cleanup - Guida Operativa | 2026-03-18 | 2026-03-19 | 6 | operations | fonte_verita | Procedura step-by-step eliminazione tenant |
| RLS-MULTITENANCY.md | Row Level Security | 2026-02-02 | 2026-02-02 | 20 | security | da_aggiornare | RLS attivo su 6 tabelle; app usa superuser |
| RLS-ACTIVATION-ROADMAP.md | RLS Activation Roadmap | 2026-02-02 | 2026-02-02 | 9 | adr | fonte_verita | ADR: Opzione A scelta |
| LOGTO-MULTITENANCY-RUNBOOK.md | Logto Multitenancy Runbook | 2026-02-02 | 2026-02-02 | 12 | operations | da_verificare | Eseguito su staging; prod non aggiornata |
| observability-audit.md | Observability Audit Report | 2026-03-12 | 2026-03-19 | 13 | observability | fonte_verita | Langfuse/Prometheus OK, Jaeger/OTEL inattivo |
| AUDIT-TECNICO-LEXE-2026-02-20.md | Audit Tecnico | 2026-02-21 | 2026-02-21 | 39 | security | da_aggiornare | Modelli e alias pre-benchmark |
| PROPOSTA-NOMI-UI.md | Proposta Nomi UI | 2026-02-21 | 2026-03-19 | 8 | vision | da_verificare | Mapping nomi interni -> label utente |

### ADR & Runbooks

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| knowhow/adr/ADR-D1-unified-classifier.md | Unified Intent Classifier | 2026-02-21 | 2026-02-21 | 3 | adr | da_aggiornare | Superato da Pipeline v2 binary routing |
| knowhow/adr/ADR-D2-semaphores-rate-limiting.md | Semaphores for Tool Rate Limiting | 2026-02-21 | 2026-02-21 | 3 | adr | fonte_verita | Ancora attivi in executor.py |
| knowhow/adr/ADR-D3-lexorc-roles-db.md | LEXORC Roles via Database | 2026-02-21 | 2026-02-21 | 3 | adr | da_aggiornare | LEXORC disabilitato; ruoli usati da Pipeline v2 |
| knowhow/adr/ADR-D4-ff-legis-default-true.md | FF ff_legis_agent Default True | 2026-02-21 | 2026-02-21 | 2 | adr | da_aggiornare | LEGIS avvolto da LegalStrategy |
| knowhow/adr/ADR-D5-confidence-spectrum.md | Three-Level Confidence Spectrum | 2026-02-21 | 2026-02-21 | 3 | adr | da_aggiornare | Soglie 3-band non riflesse |
| knowhow/adr/ADR-D6-self-correction-max-2.md | Self-Correction Max 2 Retries | 2026-02-21 | 2026-02-21 | 3 | adr | da_verificare | Verificare coerenza con DepthLevel |
| knowhow/runbooks/RUNBOOK-deploy.md | Deploy Staging -> Production | 2026-02-21 | 2026-02-21 | 5 | runbook | da_aggiornare | Manca lexe-admin nei repo |
| knowhow/runbooks/RUNBOOK-rollback.md | Rollback Procedures | 2026-02-21 | 2026-02-21 | 7 | runbook | da_aggiornare | Manca rollback lexe-admin e migrations 018-031 |

### Sprint Reports

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| sprints/SPRINT-LEXORC-F1-F4.md | Sprint LEXORC F1-F4 | 2026-02-21 | 2026-02-21 | 7 | sprint | fonte_verita | 2496 LOC backend + 1195 LOC frontend |
| HANDOFF-TOOLS-9.md | 9 Legal Tools Webchat | 2026-02-10 | 2026-02-21 | 21 | api | da_aggiornare | Verificare tool aggiunti/rimossi dopo Sprint 9 |

---

## 2. lexe-core/

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| README.md | LEXE Core | 2026-02-01 | 2026-02-01 | 1 | api | da_aggiornare | Manca orchestration v2, pipeline v2, memory, admin refs |
| api-lexe-core.md | API Reference | 2026-02-01 | 2026-02-01 | 9 | api | da_aggiornare | Mancano endpoint Sprint 9-10 |
| mappa-lexe-core.md | Mappa lexe-core | 2026-02-01 | 2026-02-01 | 12 | architecture | da_aggiornare | Mancano orchestration/, agent/, strategies/ |
| tests/benchmarks/README.md | LLM Benchmark Suite v2.0 | 2026-02-23 | 2026-02-23 | 46 | benchmark | fonte_verita | G-Eval pipeline 3 stadi, 9 test case |
| tests/e2e/README.md | E2E Test Suite | 2026-03-13 | 2026-03-13 | 12 | testing | fonte_verita | 7 smoke + 11 contract + quality P2 |
| LEXE-ADMIN_TECH-SPEC_2026-02-22.md.html | Admin Panel Tech Spec | 2026-02-22 | 2026-02-22 | 43 | architecture | da_verificare | Spec pre-implementazione; verificare delta vs Sprint 10 |
| LEXE_BENCHMARK_REPORT_2026-02-23.md.html | Full Benchmark Report | 2026-02-23 | 2026-02-23 | 92 | benchmark | fonte_verita | 11 modelli 6 run, guidato selezione Gemini 3 Flash |

---

## 3. lexe-max/ (Knowledge Base)

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| CLAUDE.md | Claude Context | 2026-02-07 | 2026-02-21 | 21 | architecture | da_aggiornare | DB user errato (`lexe_max` vs `lexe_kb`), stats obsolete |
| KB_SETUP.md | Setup Guide | 2026-02-06 | 2026-02-21 | 4 | operations | da_verificare | Ref `lexe_max` user potrebbe essere stale |
| docs/KB_V3_UNIFIED_SCHEMA.md | KB V3 Unified Schema | 2026-02-06 | 2026-02-21 | 26 | architecture | fonte_verita | Schema unificato v3.0.0, migration 050 |
| docs/SCHEMA_KB_OVERVIEW.md | Schema PostgreSQL Overview | 2026-02-06 | 2026-02-21 | 17 | architecture | da_aggiornare | Pre-V3, superato da KB_V3_UNIFIED_SCHEMA |
| docs/KB-MASSIMARI-ARCHITECTURE.md | Massimari Architettura | 2026-02-01 | 2026-02-21 | 20 | architecture | da_aggiornare | MVP iniziale, ref a `lexe-postgres` errato |
| docs/KB-MASSIMARI-COMPLETE.md | Massimari Documento Definitivo | 2026-02-06 | 2026-02-21 | 16 | kb | da_verificare | Stats 772 massime (pre-scale, ora 46k+) |
| docs/KB-MASSIMARI-STAGING-DEPLOY.md | Massimari Staging Deploy | 2026-02-01 | 2026-02-02 | 16 | operations | obsoleto | Deploy iniziale 3,127 massime, superato |
| docs/KB-MASSIMARI-BENCHMARK-REPORT.md | Massimari Benchmark | 2026-02-01 | 2026-02-02 | 19 | testing | da_verificare | Hybrid 87.4% accuracy su 772 massime |
| docs/KB-NORMATIVA-INGESTION.md | Normativa Ingestion | 2026-02-06 | 2026-02-21 | 12 | kb | da_verificare | Pipeline forense CC/CP |
| docs/KB-NORMATIVA-ALTALEX.md | Altalex Pipeline | 2026-02-04 | 2026-02-21 | 11 | kb | da_verificare | 68 PDF Altalex, chunking semantico |
| docs/KB-HANDOFF.md | Massimari Handoff | 2026-02-01 | 2026-02-02 | 11 | operations | da_aggiornare | text-embedding-3-small 1536dim |
| docs/GRAPH-IMPLEMENTATION-PLAN.md | Grafi KB Piano | 2026-02-06 | 2026-02-21 | 13 | architecture | da_aggiornare | AGE extension, citation+thematic graph |
| docs/KB-NORM-GRAPH-HANDOFF.md | Norm Graph Handoff | 2026-02-01 | 2026-02-01 | 8 | kb | fonte_verita | 4,128 norme, 42,338 edges |
| docs/KB-CATEGORY-GRAPH-HANDOFF.md | Category Graph Handoff | 2026-02-01 | 2026-02-01 | 9 | kb | fonte_verita | 8 L1 + 43 L2 categorie, 97.2% coverage |
| docs/KB-EMBEDDING-BENCHMARK-DECISION-LOG.md | Embedding Benchmark Decision | 2026-02-01 | 2026-02-01 | 5 | testing | obsoleto | Pre-decisione finale |
| docs/ARTICLE_EXTRACTION_STRATEGIES.md | Estrazione Articoli PDF | 2026-02-05 | 2026-02-21 | 16 | kb | fonte_verita | Algoritmo 4-fasi production ready |
| docs/QA_REPORT_2026-01-30.md | QA Protocol v3.2 | 2026-02-01 | 2026-02-01 | 15 | testing | da_verificare | Coverage 66.7% |
| docs/QC_PLAYBOOK_KB_NORMATIVA.md | QC Playbook v1.0 | 2026-02-08 | 2026-02-21 | 4 | testing | fonte_verita | KPI 10,556 articoli |
| docs/quasigrafi.md | Category Graph v2.4 | 2026-02-01 | 2026-02-01 | 12 | kb | da_verificare | 80% L1 accuracy |
| docs/evaluation/ARCHITECTURE.md | KB System Architecture | 2026-02-21 | 2026-02-21 | 15 | architecture | fonte_verita | Dense+Sparse+RRF |
| docs/evaluation/HYBRID-SEARCH-EVALUATION.md | Hybrid Search Evaluation | 2026-02-21 | 2026-02-21 | 11 | testing | fonte_verita | Dense+Sparse+RRF evaluation |
| mappa-lexe-max.md | Mappa lexe-max | 2026-02-01 | 2026-02-02 | 16 | architecture | da_aggiornare | Pre-refactor |

---

## 4. Webchat, Admin, Tools, Memory, Infra

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| lexe-webchat/CLAUDE.md | Webchat Claude Instructions | 2026-02-01 | 2026-02-01 | 4 | frontend | da_aggiornare | Auth descrive Magic Link, ora usa Logto |
| lexe-webchat/README.md | LEO Webchat | 2026-02-01 | 2026-02-01 | 6 | frontend | obsoleto | Titolo "LEO", clone URL errato |
| lexe-webchat/api-lexe-webchat.md | Webchat API Flows | 2026-02-01 | 2026-02-01 | 11 | api | da_verificare | Flow Logto OIDC |
| lexe-webchat/mappa-lexe-webchat.md | Mappa webchat | 2026-02-01 | 2026-02-01 | 11 | architecture | da_aggiornare | 43 ref LEO da rinominare |
| lexe-webchat/.claude/docs/STYLE_GUIDE.md | Style Guide | 2026-02-01 | 2026-02-01 | 14 | frontend | da_aggiornare | Titolo "LEO"; colori divergono |
| lexe-admin/CLAUDE.md | Admin Panel | 2026-02-20 | 2026-02-20 | 3 | frontend | fonte_verita | Conciso, aggiornato |
| lexe-admin/PROMPT-LEXE-CORE-BACKEND-FIXES.md | Backend Fixes | 2026-03-18 | 2026-03-18 | 9 | api | da_verificare | Fix RBAC superadmin |
| lexe-tools-it/README.md | LEXe Tools Italia | 2026-02-02 | 2026-02-02 | 5 | api | da_aggiornare | Mappa dice "libreria, NO FastAPI" |
| lexe-tools-it/mappa-lexe-tools-it.md | Mappa tools-it | 2026-02-01 | 2026-02-01 | 12 | architecture | fonte_verita | 0 ref LEO |
| lexe-tools-it/docs/NORMATTIVA-API-HANDOFF.md | Normattiva API Handoff | 2026-02-21 | 2026-02-21 | 8 | api | fonte_verita | Guida agenti AI per Normattiva |
| lexe-memory/README.md | LEXE Memory | 2026-02-01 | 2026-02-01 | 1 | architecture | da_aggiornare | Stub minimo |
| lexe-memory/API-CONTRACTS.md | Memory API Contracts | 2026-02-21 | 2026-02-21 | 20 | api | fonte_verita | 49 endpoint attivi |
| lexe-memory/mappa-lexe-memory.md | Mappa memory | 2026-02-01 | 2026-02-01 | 17 | architecture | da_verificare | Verificare nuovi file |
| lexe-infra/README.md | LEXE Infrastructure | 2026-02-01 | 2026-02-01 | 1 | operations | da_aggiornare | Porte non allineate |
| lexe-infra/mappa-lexe-infra.md | Mappa infra | 2026-02-01 | 2026-02-01 | 15 | operations | da_verificare | Verificare nuovi container |

---

## 5. Root lexe-genesis/

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| CLAUDE.md | LEXE Platform - Legal AI Workspace | 2026-02-23 | 2026-02-23 | 3 | architecture | fonte_verita | Repo map, URLs, servers. Porta core dice 8000 (reale: 8100) |
| best-practices.md | Best Practices & Anti-Pattern | 2026-02-11 | 2026-02-11 | 16 | operations | fonte_verita | SSH heredoc, Traefik, PostgreSQL patterns |
| LEXE-CREDENTIALS.md | Credentials | 2026-03-18 | 2026-03-18 | 9 | credentials | fonte_verita | NON committare |
| STARTUP.md | Startup Instructions | 2026-02-01 | 2026-02-01 | 4 | operations | obsoleto | Manca lexe-admin, repos/ path non esiste |
| POSTSTART.md | Post-Start Plan | 2026-02-01 | 2026-02-01 | 9 | operations | obsoleto | Piano MVP iniziale completato |
| BACKLOG.md | Backlog | 2026-02-18 | 2026-02-18 | 14 | operations | da_aggiornare | Datato feb-18 |
| db.md | (dump DB animali) | 2026-02-19 | 2026-02-19 | 19 | legacy | obsoleto | NON pertinente a LEXE - residuo altro progetto, ELIMINARE |
| HARDENING-CONTRACT.md | Shared Contracts | 2026-03-12 | 2026-03-12 | 2 | architecture | da_verificare | Verificare vs orchestration v2 |
| LOGTO_CONFIGURATION.md | Configurazione Logto | 2026-02-14 | 2026-02-14 | 40 | reference | da_aggiornare | Manca lexe-admin SPA |
| LEXE-API-MASTER.md | Master API Reference | 2026-02-01 | 2026-02-01 | 11 | reference | da_aggiornare | Manca admin, tools-it, consent |
| KB_STATUS_2026-02-08.md | KB Status Staging | 2026-02-08 | 2026-02-08 | 6 | reference | da_aggiornare | Stats superati da MEMORY.md |

---

## 6. LEO Source (Legacy)

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| leo-core-source/CLAUDE.md | leo-core Context | 2026-02-01 | 2026-02-01 | 24 | architecture | obsoleto | Superato da lexe-core |
| leo-core-source/README.md | LEO Core | 2026-02-01 | 2026-02-01 | 13 | reference | obsoleto | Authentik, ORCHIDEA |
| leo-core-source/docs/LEO_SYSTEM_PROMPT_RECOMMENDATIONS.md | LEO System Prompt WhatsApp | 2026-01-04 | 2026-02-01 | 18 | reference | obsoleto | Specifico LEO, non LEXE |
| leo-memory-source/CLAUDE.md | leo-memory Context | 2026-02-01 | 2026-02-01 | 19 | architecture | da_verificare | Memory L0-L4 API; porta 8003 vs LEXE 8103 |
| leo-memory-source/README.md | LEO Memory | 2026-02-01 | 2026-02-01 | 19 | reference | obsoleto | Pre-LEXE |
| leo-memory-source/docs/MEMORY-V2.1-CONTRACTS.md | Memory v2.1 Contracts | 2026-01-24 | 2026-02-01 | 11 | api | da_verificare | Contratti v2.1 GDPR delete |
| leo-orchestrator-source/CLAUDE.md | leo-orchestrator Context | 2026-02-01 | 2026-02-01 | 8 | architecture | obsoleto | ORCHIDEA, Temporal |
| leo-orchestrator-source/README.md | LEO Orchestrator | 2026-02-01 | 2026-02-01 | 15 | reference | obsoleto | ORCHIDEA v1.4 + TRIDENT |
| LEO-REFERENCES-T2.md | Rebranding T2 | 2026-02-01 | 2026-02-01 | 14 | legacy | fonte_verita | Completato |
| LEO-REFERENCES-T4.md | Rebranding T4 | 2026-02-01 | 2026-02-01 | 5 | legacy | fonte_verita | Completato |
| LEO-REFERENCES-T5.md | Rebranding T5 | 2026-02-01 | 2026-02-01 | 9 | legacy | fonte_verita | Completato |

---

## 7. lexe-privacy/ & lexe-orchestrator/

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| lexe-privacy/README.md | LEXE Privacy Engine | 2026-02-01 | 2026-02-01 | 2 | privacy | fonte_verita | PII detection, dual engine |
| lexe-privacy/api/ENDPOINTS.md | Privacy API Endpoints | 2026-02-01 | 2026-02-01 | 7 | api | da_verificare | Porta 5000 non allineata |
| lexe-privacy/api/EXAMPLES.md | Privacy API Examples | 2026-02-01 | 2026-02-01 | 8 | api | da_verificare | Curl examples |
| lexe-privacy/api/IMPLEMENTATION_SUMMARY.md | Implementation Summary | 2026-02-01 | 2026-02-01 | 9 | api | da_aggiornare | Path vecchio pre-genesis |
| lexe-privacy/mappa-lexe-privacy.md | Mappa privacy | 2026-02-01 | 2026-02-01 | 10 | architecture | fonte_verita | Stack completa |
| lexe-privacy/engines/spacy/README.md | spaCy NER Engine | 2026-02-01 | 2026-02-01 | 11 | privacy | da_verificare | Import path `llsearch` vecchio |
| lexe-privacy/utils/CONFIDENCE_SCORING_GUIDE.md | PII Confidence Scoring | 2026-02-01 | 2026-02-01 | 16 | privacy | da_verificare | Import `llsearch` vecchio |
| lexe-orchestrator/README.md | LEXE Orchestrator | 2026-02-01 | 2026-02-01 | 1 | architecture | obsoleto | Disabilitato in produzione |
| lexe-orchestrator/api-lexe-orchestrator.md | Orchestrator API | 2026-02-01 | 2026-02-01 | 7 | api | obsoleto | 56 endpoint, ff_orchestrator_enabled=False |
| lexe-orchestrator/mappa-lexe-orchestrator.md | Mappa orchestrator | 2026-02-01 | 2026-02-01 | 17 | architecture | obsoleto | ORCHIDEA disabled |

---

## 8. Archived, Misc, Kimera, Brand

| file | title | created | modified | size_kb | category | status | notes |
|------|-------|---------|----------|---------|----------|--------|-------|
| archived/normattiva api .md | Normattiva API Spec Ufficiale | 2026-02-10 | 2026-02-10 | 99 | legacy | fonte_verita | Specifica ufficiale 09/01/2025 |
| archived/LEO-REFS-REMAINING.md | LEO Refs Rimasti | 2026-02-01 | 2026-02-01 | 15 | archive | obsoleto | Refactoring completato |
| archived/mappa-lexe-core.md | Mappa vecchia | 2026-02-01 | 2026-02-01 | 17 | archive | da_verificare | Potrebbe essere disallineata |
| velvet-squishing-boole.md | Auto-Improve Plan | 2026-02-23 | 2026-02-23 | 80 | legacy | obsoleto | Fasi 1-3 implementate |
| elegant-finding-horizon.md | LEGIS Agent Plan | 2026-02-18 | 2026-02-17 | 44 | legacy | obsoleto | Superato da Pipeline v2 |
| skunkworks.md | Active Learning Loop | 2026-02-04 | 2026-02-04 | 32 | legacy | obsoleto | Mai implementato |
| chimera-plugin/README.md | Kimera README | 2026-02-21 | 2026-02-21 | 11 | kimera | fonte_verita | Pubblico, CC BY-NC 4.0 |
| chimera-plugin/MARKETING-PROMPT.md | Kimera Marketing | 2026-02-21 | 2026-02-21 | 5 | kimera | fonte_verita | Growth prompt |
| chimera-plugin/kimera-universal-prompt.md | Kimera Universal Prompt | 2026-02-21 | 2026-02-21 | 6 | kimera | fonte_verita | System prompt universale |
| LEXe_2.0_Brand_Kit_v2/.../LEXe_2.0_Brand_Book.md | Brand Book | 2026-02-28 | 2026-03-19 | 16 | brand | fonte_verita | Navy+Cyan v2.0 |
| BOTTEGA-ALU_TOKEN-OPTIMIZATION_2026-02-23.md.html | Token Optimization | 2026-02-23 | 2026-02-23 | 30 | kimera | fonte_verita | Best practice Claude Code |
| LEXE_CCC-BLUEPRINT_ISSUES.md.html | CCC Blueprint Issues | 2026-02-21 | 2026-02-21 | 9 | report | fonte_verita | 12 issue tutte FIXED |
| lexe-improve-lab/docs/collect-detect.md | Improve Lab Collect & Detect | 2026-03-13 | 2026-03-13 | 20 | report | fonte_verita | Closed-loop quality system |
| web2.0/lexepro-landing/DEPLOYMENT.md | Landing Deploy | 2026-03-19 | 2026-03-19 | 2 | misc | fonte_verita | SiteGround deploy |
| web2.0/lexepro-landing/todo.md | Landing TODO | 2026-03-19 | 2026-03-19 | 6 | misc | da_aggiornare | Voci aperte |
| LEXE_MULTI_TERMINAL_ORCHESTRATION.md | Multi-Terminal Orchestration | 2026-02-21 | 2026-02-21 | 39 | sprint | obsoleto | Superato da orchestration v2 |
| LEXE_PIPELINE-BUG-ANALYSIS_2026-02-21.md | Pipeline Bug Analysis | 2026-02-22 | 2026-02-22 | 17 | report | da_verificare | 6 bug (2 critical) |
| LEXE_STATUS_2026-02-21.md.html | Status Memo | 2026-02-21 | 2026-02-21 | 26 | kimera | obsoleto | Snapshot pre-Sprint 9 |
| LEXE_SPRINT-REPORT_2026-02-21.md.html | Sprint Report | 2026-02-21 | 2026-02-21 | 25 | kimera | obsoleto | Report multi-terminale |
| LEXE_BENCHMARK-REPORT_2026-02-21.md.html | Benchmark Report | 2026-02-21 | 2026-02-21 | 34 | benchmark | da_verificare | Modelli cambiati |
| MEMO_STATUS_2026-02-21.md.html | Status Memo (dup) | 2026-02-21 | 2026-02-21 | 19 | kimera | obsoleto | Duplicato LEXE_STATUS |
| LEXe 2.0_ Proposte.md | Proposte Manus AI | 2026-02-21 | 2026-02-21 | 21 | legacy | da_verificare | Alcune idee implementate |
| plan spec and modality.md | Hardening Spec | 2026-03-12 | 2026-03-12 | 11 | legacy | obsoleto | Superato da Orchestration v2 |
| marker-bug-report.md | Marker PDF Bug | 2026-02-05 | 2026-02-05 | 3 | misc | da_verificare | Perde righe "In vigore dal" |
| lexe-8d8138ed.md | Chat Export Test | 2026-03-16 | 2026-03-16 | 5 | misc | obsoleto | Sample conversazione test |

---

## Prossimi Passi

1. **Eliminare**: `db.md` (animali), `lexe-8d8138ed.md` (chat export), duplicati `MEMO_STATUS`
2. **Archiviare**: tutti i `obsoleto` in `archived/`
3. **Aggiornare P1**: RUNBOOK-deploy, ARCHITECTURE-REFERENCE, orchestration-v2-architecture
4. **RAG Ingest**: tutti i `fonte_verita` (~45 doc) come knowledge base prioritaria
