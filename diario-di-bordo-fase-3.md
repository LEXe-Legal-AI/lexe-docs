# Diario di Bordo — Progetto LEXe Fase 3

> Legal Intelligence Platform
> *"Scrivi il caso. LEXe lavora."*

---

## Panoramica

| Campo | Valore |
|-------|--------|
| **Inizio progetto** | 25 Gennaio 2026 |
| **Stato attuale** | 24 Marzo 2026 |
| **Durata** | 57 giorni |
| **Repository** | 11 |
| **Commit totali** | 1.135 |
| **Righe nette** | ~616.000 |
| **Ambienti** | Staging + Production (Hetzner, 12 container ciascuno) |
| **Utenti** | Operativi su invito |

---

## Numeri del codice

### Volumi complessivi

| Metrica | Valore |
|---------|--------|
| Commit totali | 1.135 |
| Righe aggiunte | ~770.000 |
| Righe rimosse | ~154.000 |
| Righe nette | ~616.000 |
| Media commit/giorno | ~20 |

### Per repository

| Repo | Commit | +Righe | Descrizione |
|------|--------|--------|-------------|
| lexe-core | 453 | 185K | API Gateway, pipeline PRVS, agent, scoring, auth |
| lexe-webchat | 253 | 103K | Frontend chat, consent, profilo, evidence pack |
| lexe-infra | 101 | 5K | Docker Compose, deploy, Traefik, monitoring |
| lexe-admin | 87 | 44K | Pannello admin, tenant management, benchmark |
| lexe-tools-it | 72 | 29K | Tool legali Italia (KB, Normattiva, graph) |
| lexe-max | 50 | 274K | Knowledge Base (normativa, massime, embeddings) |
| lexe-docs | 49 | 38K | Documentazione, piani, architettura |
| lexe-memory | 43 | 26K | Sistema memoria L0-L4 |
| lexe-orchestrator | 22 | 37K | Pipeline ORCHIDEA (deprecata, sostituita da PRVS) |
| lexe-privacy | 4 | 28K | Pseudonimizzazione GDPR |
| lexe-tools-br | 1 | 0.2K | Stub per il Brasile (futuro) |

### Linguaggi

| Linguaggio | Righe | % | Dove |
|------------|------:|--:|------|
| Python | 273.191 | 51.5% | lexe-core, lexe-max, lexe-orchestrator, lexe-memory, lexe-tools-it |
| SQL | 100.434 | 18.9% | Migrations, seed data, KB (massime, normativa, graph, embeddings) |
| TypeScript/TSX | 77.739 | 14.7% | lexe-webchat, lexe-admin |
| HTML | 34.292 | 6.5% | Email template branded, report renderer |
| Markdown | 33.564 | 6.3% | Documentazione, piani, specifiche |
| Shell | 4.214 | 0.8% | Script deploy, ingest, backup |
| CSS | 1.126 | 0.2% | Styling custom (Tailwind copre il resto) |
| YAML | 914 | 0.2% | Docker Compose, configurazione |
| Dockerfile | 698 | 0.1% | 11 definizioni container |
| **Totale** | **~530.000** | **100%** | |

---

## Timeline per sprint

### Sprint 1-4 (Gennaio — inizio Febbraio)
- Setup infrastruttura: Docker Compose, Traefik, PostgreSQL, Valkey
- Fork webchat da LEO platform
- Integrazione Logto CIAM (autenticazione enterprise)
- Primi tool legali (Normattiva, fetch_article_text, verify_vigenza)
- lexe-tools-it: struttura iniziale

### Sprint 5-8 (Febbraio)
- **208 commit solo su lexe-core**
- Pipeline v1 con strategia legale
- LLM Benchmark: 6 run, 11 modelli, 2 judge → Gemini 3 Flash vince tutto
- Auto-improve: EventSink (21 eventi), export conversazioni, trace hash
- Prometheus + Grafana: dashboard operazionale, 5 alert rules
- Knowledge Base: 10.154 articoli, 41.256 chunks+embeddings, 38.718 massime
- Memory system L0-L4

### Sprint 9-11 (prima meta' Marzo)
- **Pipeline v2 PRVS**: Plan → Research → Verify → Synthesize
- Depth routing: MINIMAL / STANDARD / DEEP
- Multi-agent research: 3 wave (Norm+Juris+Doctrine → Vigenza+Gap → Remediation)
- Tenant Management Board: provisioning, limiti, Valkey enforcement
- Consent system GDPR: overlay, audit trail immutabile, art. 1341 c.c.
- Legacy vigenza verifier con whitelist costituzionale
- E2E test suite: 57 test, 43 passed, 11 skipped
- Normattiva bulk fetch: 18 codici, 1.360 articoli, 2.742 chunks su prod

### Sprint 12-14 (seconda meta' Marzo)
- **NOVA RATIO — Zero Norm Engine**: dual-layer evidence classifier, Principle Mode, Solidarity Score
- **Hybrid Score**: SQ 40% + RQ 35% + CQ 25% (sostituisce confidence inflazionato)
- 7 feature flag per rollout graduale
- 3 nuovi tool: doctrine_search, juris_verifier, principle_mapper
- Migration 082: principle_anchors (24 principi fondamentali seed)
- Email onboarding redesign: layout professionale, password temporanea
- **Forced password change**: overlay multi-step (consent → set password)
- **Voice Policy centralizzata**: base + 11 override scenario, identity rule riscritta
- **Logto**: forgot password abilitato, terms of use configurati
- **Legal disclaimer** sotto la textbox di chat

---

## Architettura attuale

### Stack container (12 servizi)

```
lexe-postgres     — DB sistema (Logto, LiteLLM, core) — porta 5435
lexe-max          — DB KB legal (normativa, massime) — porta 5436
lexe-valkey       — Cache/Session
lexe-logto        — Auth CIAM
lexe-litellm      — LLM Gateway
lexe-core         — API Gateway + Pipeline agentica
lexe-memory       — Memory system L0-L4
lexe-tools-it     — Tool legali Italia
lexe-webchat      — Frontend chat
lexe-admin        — Pannello amministrazione
shared-prometheus — Metriche
shared-grafana    — Dashboard operazionale
```

### Pipeline PRVS

```
Query utente
  → Intent Detection (CHAT / DIRECT / SIMPLE / STANDARD / COMPLEX)
  → Scenario Classification (parere, contenzioso, contratto, ...)
  → Depth Routing (MINIMAL / STANDARD / DEEP)
  → P: Planner — genera piano di ricerca
  → R: Researcher — esegue ricerca multi-tool (max 3 wave)
  → V: Verifier — vigenza + confidence scoring
  → S: Synthesizer — risposta strutturata per scenario
  → Follow-up suggestions
```

### Knowledge Base

| Risorsa | Quantita' |
|---------|-----------|
| Codici normativi | 69 (49 con chunks) |
| Articoli | 10.154 |
| Chunks + embeddings | 41.256 |
| Massime Cassazione | 38.718 |
| Graph edges | 58.737 |
| Principle anchors | 24 |
| Annotation dottrinali | 13.281 |

### LLM

Tutti i ruoli principali su **Gemini 3 Flash Preview**:
- lexe-fast (reasoning=medium, 4K tokens)
- lexe-primary (reasoning=medium, 16K tokens)
- lexe-complex (reasoning=medium, 16K tokens)
- lexe-verifier (reasoning=low, 8K tokens)
- lexe-frontier (reasoning=high, 16K tokens)
- lexe-embedding (text-embedding-3-small, 1536d)

### Scenari legali supportati

| Scenario | Template | Sezioni |
|----------|----------|---------|
| Parere pro veritate | IRAC italiano | Quesito, Normativa, Giurisprudenza, Dottrina, Analisi, Conclusioni, Fonti |
| Contenzioso | Strategico | Fatti, Quadro normativo, Precedenti, Rischio, Strategia, Schema atto |
| Recupero crediti | Procedurale | Credito, Prescrizione, Procedura, Competenza, Costi, Bozza atto |
| Contratto redazione | Bozza | 12 sezioni con citazioni normative |
| Contratto revisione | Report | Rating GREEN/YELLOW/RED, T1/T2/T3, escalation |
| NDA triage | Checklist | 10 punti, GREEN/YELLOW/RED, classificazione rischio |
| Compliance | Gap analysis | Checklist, gap, piano adeguamento |
| Risk assessment | Matrice | Rischi, matrice, risk register, raccomandazioni |
| Principle mode | 8 sezioni | Principi cost., clausole generali, dottrina, analogia, avvertenze |

---

## Decisioni architetturali chiave

1. **Pipeline PRVS > ORCHIDEA**: pipeline ORCHIDEA deprecata a favore di PRVS (piu' semplice, depth routing, fallback sicuro)
2. **Gemini 3 Flash per tutto**: un solo modello con reasoning_effort variabile, semplifica la gestione
3. **Feature flag ovunque**: 7 FF per NOVA RATIO, tutti off = Sprint 13 esatto (zero regressioni possibili)
4. **Hybrid Score > Confidence Score**: formula deterministica, soft penalties, bande 85/77/60 (niente rosso ingiusto)
5. **Voice Policy centralizzata**: modulo dedicato (`voice_policy.py`), base + override per scenario, single source of truth
6. **Consent GDPR a doppia tabella**: lookup rapido (`user_consents`) + audit immutabile (`consent_acceptance_log`)
7. **Password change forzato**: flag DB `password_must_change`, endpoint senza current password, overlay multi-step
8. **KB come database relazionale**: PostgreSQL + pgvector, non vector DB standalone. Graph edges, embeddings, FTS nello stesso schema

---

## Cosa c'e' oggi in produzione

- Pipeline agentica PRVS con 10+ scenari
- Knowledge Base da 69 codici + 38K massime + graph
- Auth enterprise Logto con consent GDPR
- Tenant management con limiti e provisioning
- Email onboarding branded
- Forced password change al primo accesso
- Voice policy per output professionali
- Legal disclaimer
- Observability: Prometheus, Grafana, Langfuse, EventSink
- Benchmark suite: Kryptonite A/B, Zero-norm (25 query), Quickfire (17 items)

---

## Prossimi passi

- Calibrare Hybrid Score su traffico reale
- Fix bench runner (routing `unknown` → `legal`)
- LEXe v4: pipeline agentica orchestrata (3 varianti studiate, "Corte d'Appello" raccomandata)
- Nightly re-ingest Normattiva (~2000 art/notte)
- Password-less invite (link "imposta password" via Logto, eliminare password temporanea)
- Canary rollout NOVA RATIO (5% → 25% → 50% → 100%)

---

*LEXe — Legal Intelligence Platform*
*Diario di Bordo Fase 3 — 24 Marzo 2026*
*IT Consulting SRL*
