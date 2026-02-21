# Sprint LEXORC — F1-F4 Implementation

**Data:** 2026-02-19
**Branch:** `stage`
**Repos:** `lexe-core`, `lexe-webchat`

---

## Obiettivo

Implementazione completa del piano LEXORC (LEXe Orchestrator) — pipeline agentica 1+5 per ricerca legale avanzata con verifica in-line e sintesi strutturata.

## Architettura

```
User Query
    |
    v
LexeOrchestrator (6 fasi)
    |
    +- 1. INTAKE --- classify_complexity() -> FLASH/STANDARD/DEEP/STRATEGIC
    +- 2. PLANNING -- generate_work_plan() -> task prioritizzati
    +- 3. RESEARCHING - ParallelEngine (asyncio.gather)
    |      +- A1 NormAgent (Dogmatico) ---- Normattiva + EUR-Lex
    |      +- A2 CaseLawAgent (Pretorio) -- KB Search + InfoLex
    |      +- A3 DoctrineAgent (Esploratore) - Brocardi + Web (opt-in)
    +- 4. VERIFYING -- LexorcAuditor (Censore) -- 2-layer verification
    +- 5. SYNTHESIZING - LexorcSynthesizer (Redattore) -- verified streaming
    +- 6. COMPLETE --- NBA generation + evidence persistence
```

## File Creati/Modificati

### Backend — lexe-core (2,496 LOC new)

| File | LOC | Tipo | Descrizione |
|------|-----|------|-------------|
| `agent/classifier.py` | 115 | NEW | Classificatore complessita LLM-based |
| `agent/work_plan.py` | 185 | NEW | Generatore work plan con Plan A/B |
| `agent/advanced_orchestrator.py` | 529 | REWRITE | Pipeline 6 fasi con fallback LEGIS |
| `agent/parallel_engine.py` | 373 | NEW | asyncio.gather + CircuitBreaker |
| `agent/agents/__init__.py` | 1 | NEW | Package init |
| `agent/agents/norm_agent.py` | 167 | NEW | A1 — Normattiva + EUR-Lex |
| `agent/agents/caselaw_agent.py` | 137 | NEW | A2 — KB hybrid search + InfoLex |
| `agent/agents/doctrine_agent.py` | 151 | NEW | A3 — Brocardi + Web (purity filter) |
| `agent/agents/auditor.py` | 448 | NEW | A4 — 2-layer verification |
| `agent/agents/lexorc_synthesizer.py` | 390 | NEW | A5 — Verified streaming + NBA |

**File modificati (Phase 0 pre-esistenti):**
- `agent/events.py` (+76 lines) — LEXORC SSE event helpers
- `config.py` (+54 lines) — LEXORC settings (models, timeouts, limits)
- `gateway/customer_router.py` (+165 lines) — LEXORC endpoint integration
- `gateway/sse_contracts.py` (+144 lines) — 7 LEXORC SSE event types
- `tools/definitions.py` (+7 lines) — LEXORC tool definitions

### Frontend — lexe-webchat (1,195 LOC new)

| File | LOC | Tipo | Descrizione |
|------|-----|------|-------------|
| `components/lexorc/index.ts` | 14 | NEW | Barrel export |
| `components/lexorc/OrchidPulse.tsx` | 114 | NEW | Phase indicator con animate-ping |
| `components/lexorc/PurityBadge.tsx` | 106 | NEW | Badge purezza fonte (5 livelli) |
| `components/lexorc/ConfidenceMeter.tsx` | 86 | NEW | Barra confidenza animata |
| `components/lexorc/TaskCard.tsx` | 231 | NEW | Card agente con status/evidence |
| `components/lexorc/WorkPlanBoard.tsx` | 173 | NEW | Board task con progress bar |
| `components/lexorc/StrategicPause.tsx` | 242 | NEW | Modal Piano A/B con countdown |
| `components/lexorc/DeepeningChips.tsx` | 87 | NEW | Chips NBA cliccabili |
| `components/lexorc/LexorcPanel.tsx` | 142 | NEW | Container assemblaggio componenti |

**File modificati:**
- `components/chat/ChatMessage.tsx` (+9 lines) — LexorcPanel integration
- `hooks/useStreaming.ts` (+63 lines) — LEXORC SSE event handlers
- `services/streaming/SSEClient.ts` (+16 lines) — LEXORC event types

**File pre-esistenti (Phase 0, non modificati):**
- `stores/workplanStore.ts` — Zustand store per LEXORC state

## Totale

| Metrica | Valore |
|---------|--------|
| **Codice nuovo** | 3,691 LOC |
| **File nuovi** | 19 |
| **File modificati** | 8 |
| **Backend Python** | 10 file (2,496 LOC) |
| **Frontend React/TS** | 9 file (1,195 LOC) |

## Dettagli Tecnici

### Classificazione Complessita

| Livello | Agenti | Esempio |
|---------|--------|---------|
| FLASH | Solo Synthesizer | "Che cos'e il codice civile?" |
| STANDARD | NormAgent + Auditor + Synth | "Obblighi del datore di lavoro" |
| DEEP | Tutti (3 research + Auditor + Synth) | "Responsabilita solidale art. 29 D.Lgs. 81/2008" |
| STRATEGIC | Tutti + Plan A/B modal | "Ho un problema con un cliente che non paga" |

### Parallel Engine — Circuit Breaker

- **Window:** 60 secondi rolling
- **Threshold:** 3 fallimenti -> agente disabilitato
- **Cooldown:** 5 minuti
- **Timeout per agente:** 20s (configurabile)

### Auditor — 2 Layer

| Layer | Tipo | Checks |
|-------|------|--------|
| L1 | Strutturale (no LLM) | Formato articolo, URL/URN, vigenza, testo, score, RV/anno |
| L2 | LLM Coherence | Rilevanza, contraddizioni, citazioni sospette |

**Verdetti:** `pass` (green, conf 0.8-0.9) / `warn` (yellow, conf 0.6) / `fail` (red, conf 0.3)

### Synthesizer — Sezioni Output

| Sezione | Contenuto |
|---------|-----------|
| A | Inquadramento normativo |
| B | Giurisprudenza di legittimita |
| C | Giurisprudenza di merito |
| D | Dottrina |
| E | Considerazioni operative |
| F | Conclusioni e raccomandazioni |

### SSE Events (7 tipi LEXORC)

| Evento | Quando |
|--------|--------|
| `LEXORC_PHASE` | Cambio fase pipeline |
| `LEXORC_CHESSBOARD_INIT` | Work plan creato |
| `LEXORC_PLAN_NODE` | Status task cambia |
| `LEXORC_EVIDENCE_CHUNK` | Evidenza trovata |
| `LEXORC_AUDIT_RESULT` | Verdetto auditor |
| `LEXORC_NB_ACTION` | Next Best Action |
| `LEXORC_STRATEGIC_PAUSE` | Query STRATEGIC -> scelta A/B |

### Frontend — Pattern Orchidea

- **OrchidPulse:** Indicatore fase con colori (purple/blue/green/amber/emerald)
- **TaskCard:** Card per agente con color dot e status icon
- **WorkPlanBoard:** Board con complexity badge e progress bar
- **PurityBadge:** 5 livelli (LEXe Graph -> Web)
- **ConfidenceMeter:** Barra gradiente (red->yellow->green) con framer-motion
- **StrategicPause:** Modal A/B con countdown auto-select
- **DeepeningChips:** Chips NBA con Sparkles header

## Graceful Fallback Chain

```
LEXORC Pipeline
  | fail?
  v
LEGIS PRVS Pipeline (pre-esistente)
  | fail?
  v
Error response con messaggio italiano
```

Ogni componente ha fallback indipendente:
- `ParallelEngine` fail -> fallback a LEGIS pipeline
- `LexorcAuditor` fail -> verdetto "warn" automatico
- `LexorcSynthesizer` fail -> LEGIS synthesizer
- `generate_nba()` fail -> silenzioso skip

## Feature Flag

```env
LEXE_FF_LEXORC_ENABLED=true          # Abilita globalmente
LEXE_FF_LEXORC_TENANTS=              # Vuoto = tutti i tenant
```

## Test Results

| Check | Risultato |
|-------|-----------|
| Python py_compile (10/10) | PASS |
| TypeScript tsc --noEmit | PASS |
| Vite production build | PASS |
| Import dependency analysis | PASS (0 circular) |
| Backend dependency existence | PASS (10/10 modules) |
| Frontend file existence | PASS (12/12 files) |
| Code quality (TODO/FIXME/print) | PASS (0 issues) |

## Prossimi Passi

1. Abilitare `LEXE_FF_LEXORC_ENABLED=true` in staging override
2. Test end-to-end con query di ogni livello di complessita
3. Monitoring latency ParallelEngine in produzione
4. Tuning timeout e circuit breaker thresholds
5. Aggiungere unit test per classifier e auditor
