# LEXE Auto-Improve System — Piano di Implementazione Parallela

> 5 Fasi, Team Paralleli con Reviewer, Test Cooperativi tra Fasi
> Data: 2026-02-23

---

## Contesto & Motivazione

Il deploy di LEGIS (memory fix + confidence calibration) ha evidenziato che:

1. Gli SSE events (21+ tipi) sono **effimeri** — persi dopo la risposta
2. Non esiste export conversazioni, ne valutazione utente
3. Il monitoring e frammentario (SLA in-memory, Prometheus parziale, no Grafana/Loki deployati)
4. Non c'e un loop di auto-improvement controllato

Questo piano costruisce l'infrastruttura completa: dalla persistenza dati al loop di improvement automatico, in 5 fasi con risultati fin dalla prima.

---

## Stato Attuale (dall'esplorazione)

### Backend (lexe-core)

| Componente          | File                                   | Stato                                                                        |
| ------------------- | -------------------------------------- | ---------------------------------------------------------------------------- |
| Streaming generator | `gateway/customer_router.py:1102-2288` | stream_to_orchestrator() — 21+ event types yielded                           |
| save_message()      | `gateway/customer_router.py:771-818`   | Salva solo role+content+tenant_id, **metadata JSONB inutilizzato**           |
| ID generation       | `gateway/sse_contracts.py:53-55`       | generate_ids() → (correlation_id, trace_id) via uuid4                        |
| SSE contracts       | `gateway/sse_contracts.py` (586 righe) | 21 event functions, ben strutturate                                          |
| Prometheus          | `gateway/metrics.py` (358 righe)       | 6 metriche: latency, requests, errors, sse_connections, rate_limit, messages |
| SLA dashboard       | `gateway/sla_router.py` (168 righe)    | In-memory, 9 counters, si resetta al restart                                 |
| Tool logging        | `tools/logging.py` (67 righe)          | log_tool_execution() async, non-blocking                                     |
| Config/flags        | `config.py` (330 righe)                | 10+ feature flags, pattern is_*_enabled_for_tenant()                         |
| Admin routers       | `admin/router.py` (72 righe)           | 17 sub-routers montati sotto /api/v1/admin                                   |
| Main.py             | `main.py` (156 righe)                  | 4 routers: customer, tools, admin, sla                                       |
| Migrations          | `migrations/001-023`                   | 20 files SQL, ultimo: 023_super_tool_tracking.sql                            |
| Tests               | `tests/`                               | 14 test files, pytest + pytest-asyncio                                       |
| Conversation model  | `conversation/models.py`               | metadata_ JSONB field esiste ma **inutilizzato**                             |

**Tabelle NON esistenti:** conversation_events, message_evaluations, experiments
**Servizi NON deployati:** Prometheus, Grafana, Loki, Jaeger (non nel docker-compose.yml)

### Frontend (lexe-webchat)

| Componente        | File                                         | Stato                                                                         |
| ----------------- | -------------------------------------------- | ----------------------------------------------------------------------------- |
| Message rendering | `chat/ChatMessage.tsx:80-502`                | Props: debugInfo, confidenceLevel/Score, action buttons (copy, export, retry) |
| Chat input        | `App.tsx:919-933`                            | textarea con `disabled={isStreaming}`                                         |
| Stream store      | `stores/streamStore.ts`                      | Handles done event, extracts confidence. **No trace hash**                    |
| Chat store        | `stores/chatStore.ts`                        | ChatMessage interface con debugInfo, confidence. **No evaluation**            |
| API client        | `services/api/client.ts`                     | Auth headers automatici, get/post/patch/delete                                |
| DropdownMenu      | `components/ui/DropdownMenu.tsx` (281 righe) | Completo con trigger, content, items, animations                              |
| Modal pattern     | `lexorc/StrategicPause.tsx` (237 righe)      | AnimatePresence + motion.div, selection cards                                 |
| Footer toolbar    | `App.tsx:972-1032`                           | Language, theme, help, model badge                                            |
| Export existing   | `ChatMessage.tsx:287-303`                    | Single message .txt export pattern                                            |
| Icons             | lucide-react v0.468.0                        | Star, ShieldCheck, Copy, FileDown, ecc.                                       |

### Integrazione critica: save_message() (customer_router.py:771-818)

```python
# OGGI — salva solo basico, metadata ignorato
async def save_message(conversation_id, role, content, tenant_id=None):
    INSERT INTO core.conversation_messages (conversation_id, role, content, tenant_id)
    # metadata_ column esiste ma MAI usato!
```

### Streaming path — dove wrappare per persistenza

```
customer_stream() [line 2590]
  └── stream_to_orchestrator() [line 1102]
        ├── generate_ids() → correlation_id, trace_id [line 1132]
        ├── meta_event() [line 1255]
        ├── preprocessing_event() [line 1266+]
        ├── ROUTING DECISION [line 1282+]
        │   ├── ORCHESTRATOR path [line 1282] → stream_via_orchestrator()
        │   ├── SUPER_TOOL path [line 1404] → SuperToolAgent.run()
        │   ├── LEXORC path [line 1456] → LexeOrchestrator.run()
        │   ├── LEGIS path [line 1506] → legis_agent_pipeline()
        │   └── QUICK_FIX path [line 1563] → direct LiteLLM
        ├── save_message() assistant [lines 1446/1496/1552/2281]
        └── done_event() [final]
```

---

## Strategia Repo

**Scelta: Option 1 — Repo dedicato `lexe-improve-lab`**

Motivazione:

- Separazione netta tra codice production (lexe-core) e codice esperimenti
- Il lab legge dal DB (read-only), non serve coupling diretto
- Riduce rischio di regressioni in production
- CLI tools dedicati per dataset building e evaluation
- Feature flags in lexe-core per le integrazioni leggere

```
lexe-improve-lab/          # Nuovo repo
├── pyproject.toml
├── src/
│   └── lexe_improve/
│       ├── collector/     # Data collection da PostgreSQL
│       ├── datasets/      # Dataset builder (JSONL)
│       ├── detectors/     # Issue detection
│       ├── proposers/     # Fix proposal generation
│       ├── evaluators/    # Offline evaluation suite
│       ├── replay/        # Replay runner
│       └── cli.py         # CLI entrypoint
├── tests/
├── experiments/           # Experiment configs
└── data/                  # Local dataset cache
```

---

## Feature Flags & Safe Defaults

| Flag                             | Default | Dove                | Scopo                                 |
| -------------------------------- | ------- | ------------------- | ------------------------------------- |
| `ff_event_persistence`           | `true`  | config.py           | Abilita persistenza SSE events        |
| `ff_trace_hash`                  | `true`  | config.py           | Genera trace hash nel metadata        |
| `ff_evaluation_enabled`          | `true`  | config.py           | Abilita endpoint valutazione          |
| `ff_evaluation_mandatory_global` | `false` | config.py           | Valutazione obbligatoria (globale)    |
| `evaluation_mandatory`           | `false` | core.tenants column | Valutazione obbligatoria (per tenant) |
| `ff_extended_metrics`            | `true`  | config.py           | Metriche Prometheus estese            |
| `ff_assessment_enabled`          | `true`  | config.py           | Assessment report endpoint            |
| `ff_self_refine`                 | `false` | config.py           | Self-refine loop (Phase 5)            |
| `ff_structured_reflection`       | `false` | config.py           | Episodic reflection (Phase 5)         |
| `ff_tool_budget`                 | `false` | config.py           | Tool budget enforcement (Phase 5)     |
| `ff_memory_hygiene`              | `false` | config.py           | Memory gating/dedup (Phase 5)         |

---

## Nuove Tabelle & Migrations

| Migration                     | Tabella                       | Schema                                                                                                               | Fase |
| ----------------------------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------- | ---- |
| `024_conversation_events.sql` | `core.conversation_events`    | id, tenant_id, conversation_id, message_id, correlation_id, trace_id, event_type, event_data JSONB, created_at       | F1   |
| `025_message_evaluations.sql` | `core.message_evaluations`    | id, tenant_id, conversation_id, message_id, user_id, rating 1-5, comment, created_at                                 | F1   |
| `025b_tenant_eval_flag.sql`   | ALTER core.tenants            | ADD evaluation_mandatory BOOLEAN DEFAULT FALSE                                                                       | F1   |
| `026_experiments.sql`         | `core.experiments`            | id, name, status, hypothesis, baseline_config JSONB, candidate_config JSONB, metrics JSONB, created_at, completed_at | F4   |
| `027_reflections.sql`         | `core.structured_reflections` | id, tenant_id, conversation_id, message_id, trigger_type, cause, evidence, patch_idea, test_idea, created_at         | F5   |

---

## Nuove Metriche Prometheus

| Metrica                                 | Tipo      | Labels                                       | Fase |
| --------------------------------------- | --------- | -------------------------------------------- | ---- |
| `lexe_pipeline_routed_total`            | Counter   | pipeline=legis\|toolloop\|lexorc\|quickfix   | F1   |
| `lexe_confidence_level_total`           | Counter   | level=VERIFIED\|CAUTION\|LOW_CONFIDENCE      | F1   |
| `lexe_memory_retrieval_total`           | Counter   | status=retrieved\|empty\|error\|circuit_open | F1   |
| `lexe_memory_retrieval_latency_seconds` | Histogram | -                                            | F1   |
| `lexe_tokens_used`                      | Counter   | pipeline, type=prompt\|completion            | F3   |
| `lexe_model_calls_total`                | Counter   | model                                        | F3   |
| `lexe_evaluation_rating`                | Histogram | tenant_id                                    | F1   |
| `lexe_event_sink_batch_size`            | Histogram | -                                            | F1   |
| `lexe_event_sink_flush_latency_seconds` | Histogram | -                                            | F1   |
| `lexe_event_sink_errors_total`          | Counter   | -                                            | F1   |

---

## CLI & Endpoint per Evaluation/Replay

### Endpoints (lexe-core)

| Endpoint                                                   | Metodo | Fase | Scopo                 |
| ---------------------------------------------------------- | ------ | ---- | --------------------- |
| `POST /api/v1/conversations/{id}/messages/{mid}/evaluate`  | POST   | F1   | Submit valutazione    |
| `GET /api/v1/conversations/{id}/messages/{mid}/evaluation` | GET    | F1   | Get valutazione       |
| `GET /api/v1/config/evaluation`                            | GET    | F1   | Config mandatory flag |
| `GET /api/v1/admin/evaluations`                            | GET    | F2   | Stats aggregate       |
| `GET /api/v1/admin/trace/{hash}`                           | GET    | F2   | Lookup trace hash     |
| `GET /api/v1/conversations/{id}/export`                    | GET    | F2   | Export role-based     |
| `GET /api/v1/admin/assessment`                             | GET    | F3   | Assessment report     |

### CLI (lexe-improve-lab)

| Comando                                                            | Fase | Scopo                  |
| ------------------------------------------------------------------ | ---- | ---------------------- |
| `lexe-improve collect --period 7d --output data/`                  | F4   | Raccoglie traces da DB |
| `lexe-improve dataset build --input data/ --output datasets/`      | F4   | Build JSONL dataset    |
| `lexe-improve detect --dataset datasets/latest.jsonl`              | F4   | Rileva issues          |
| `lexe-improve propose --issues issues.json`                        | F4   | Genera fix proposals   |
| `lexe-improve evaluate --baseline base.json --candidate cand.json` | F4   | Offline evaluation     |
| `lexe-improve replay --dataset datasets/latest.jsonl --sandbox`    | F4   | Replay in sandbox      |
| `lexe-improve report --experiment exp-001`                         | F4   | Report esperimento     |

---

# ============================================================================

# FASE 1: DATA FOUNDATION

# ============================================================================

## Obiettivo

Persistenza completa di tutti gli SSE events + valutazione utente + trace hash.
**Risultato immediato:** ogni conversazione lascia una traccia completa nel DB, interrogabile.

## Team Structure

```
TEAM ALPHA — "Persistence Engine" (Backend Audit Trail)
├── Agent A1-CODER-1: Migration + Event Sink
├── Agent A1-CODER-2: customer_router integration + metadata enrichment
├── Agent A1-CODER-3: Prometheus metrics extensions
└── Agent A1-REVIEWER: Test + review + backlog

TEAM BETA — "Evaluation & Trace" (Backend Evaluation + Trace Hash)
├── Agent B1-CODER-1: Migration evaluations + tenant flag + evaluation_router
├── Agent B1-CODER-2: Trace hash generation + config endpoint + done_event extension
├── Agent B1-CODER-3: Feature flags + main.py router registration
└── Agent B1-REVIEWER: Test + review + backlog
```

---

### TEAM ALPHA — Agent A1-CODER-1: Migration + Event Sink

**Task 1A: Migration 024_conversation_events.sql**

File: `lexe-core/migrations/024_conversation_events.sql` (NUOVO)

```sql
-- Migration 024: Conversation Events (Audit Trail)
-- Persists all SSE events for complete pipeline reconstruction

CREATE TABLE IF NOT EXISTS core.conversation_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    conversation_id UUID NOT NULL REFERENCES core.conversations(id) ON DELETE CASCADE,
    message_id UUID,
    correlation_id VARCHAR(64) NOT NULL,
    trace_id VARCHAR(64),
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_conv_events_conversation
    ON core.conversation_events(conversation_id, created_at);
CREATE INDEX IF NOT EXISTS idx_conv_events_correlation
    ON core.conversation_events(correlation_id);
CREATE INDEX IF NOT EXISTS idx_conv_events_type
    ON core.conversation_events(event_type, created_at);
CREATE INDEX IF NOT EXISTS idx_conv_events_tenant_time
    ON core.conversation_events(tenant_id, created_at);

-- Partition-ready: conversation_events will grow fast
-- Future: PARTITION BY RANGE (created_at) per month
COMMENT ON TABLE core.conversation_events IS
    'Audit trail of all SSE events per conversation. Non-blocking persistence.';
```

**Task 1B: Event Sink**

File: `lexe-core/src/lexe_core/gateway/event_sink.py` (NUOVO)

Requisiti:

- Class `EventSink` con `asyncio.Queue` interna
- Buffer fino a 50 eventi o timeout 2s, poi batch INSERT
- `persist(tenant_id, conversation_id, correlation_id, trace_id, raw_event_str)` — parse SSE string, extract type + data
- MAI bloccare il response path — fire-and-forget con backpressure
- Se queue piena (max 500): drop evento + log warning + increment `lexe_event_sink_errors_total`
- Background task `_flush_loop()` avviato con `asyncio.create_task()` al primo persist
- Graceful shutdown: flush residui al shutdown FastAPI
- Metriche: `event_sink_batch_size`, `event_sink_flush_latency_seconds`, `event_sink_errors_total`
- Pattern da seguire: simile a `tools/logging.py:18-67` (async, non-blocking, direct asyncpg)

```python
# Struttura
class EventSink:
    def __init__(self, dsn: str, max_queue: int = 500, batch_size: int = 50, flush_interval: float = 2.0):
        self._queue: asyncio.Queue[EventRecord] = asyncio.Queue(maxsize=max_queue)
        self._task: asyncio.Task | None = None
        ...

    async def persist(self, tenant_id, conversation_id, correlation_id, trace_id, raw_sse: str) -> None:
        record = self._parse_sse(raw_sse)
        try:
            self._queue.put_nowait(record)
        except asyncio.QueueFull:
            logger.warning("Event sink queue full, dropping event")
            EVENT_SINK_ERRORS.inc()

    async def _flush_loop(self):
        while True:
            batch = []
            try:
                # Collect up to batch_size or timeout
                ...
                await self._batch_insert(batch)
            except Exception:
                logger.warning("Event sink flush failed", exc_info=True)
                EVENT_SINK_ERRORS.inc()

    async def _batch_insert(self, records: list[EventRecord]):
        # asyncpg.connect() + executemany
        # INSERT INTO core.conversation_events (tenant_id, conversation_id, correlation_id, trace_id, event_type, event_data)
        ...

    def _parse_sse(self, raw: str) -> EventRecord:
        # Parse "data: {...}" SSE format → extract event_type + event_data
        ...
```

---

### TEAM ALPHA — Agent A1-CODER-2: customer_router Integration + Metadata Enrichment

**Task 2A: Wrap streaming con event sink**

File: `lexe-core/src/lexe_core/gateway/customer_router.py`

In `stream_to_orchestrator()` (line 1102), aggiungere:

1. Importare `EventSink` dal modulo event_sink
2. Inizializzare sink globale (singleton o FastAPI dependency)
3. Prima di ogni `yield event_str`, chiamare `sink.persist(...)` — NON await bloccante

Punti specifici dove inserire la persistenza:

```python
# Line ~1255: meta_event
meta = meta_event(correlation_id, trace_id, ...)
await event_sink.persist(tenant_id, conversation_uuid, correlation_id, trace_id, meta)
yield meta

# Line ~1266: preprocessing_event
pp = preprocessing_event(correlation_id, trace_id, step)
await event_sink.persist(tenant_id, conversation_uuid, correlation_id, trace_id, pp)
yield pp

# Pattern: wrap ALL yield points
```

**Approccio migliore:** creare helper wrapper:

```python
async def _emit(event_str: str, *, sink: EventSink, tenant_id, conversation_id, correlation_id, trace_id):
    """Persist + yield in one call."""
    await sink.persist(tenant_id, conversation_id, correlation_id, trace_id, event_str)
    return event_str
```

Poi sostituire tutti i `yield meta_event(...)` con `yield await _emit(meta_event(...), sink=sink, ...)`.

**Task 2B: Metadata enrichment in save_message()**

File: `lexe-core/src/lexe_core/gateway/customer_router.py:771-818`

Modificare `save_message()` per accettare metadata:

```python
async def save_message(
    conversation_id: UUID,
    role: str,
    content: str,
    tenant_id: UUID | None = None,
    metadata: dict | None = None,      # NUOVO
) -> UUID:                               # NUOVO: ritorna message_id
```

Modificare INSERT per includere metadata:

```sql
INSERT INTO core.conversation_messages
    (id, conversation_id, role, content, tenant_id, metadata_)
VALUES ($1, $2, $3, $4, $5, $6::jsonb)
RETURNING id
```

Aggiungere metadata collector nelle 4 call sites (SUPER_TOOL/LEXORC/LEGIS/QUICK_FIX):

```python
# Prima di save_message(), raccogliere metadata
msg_metadata = {
    "pipeline": pipeline_name,
    "model": model_name,
    "correlation_id": correlation_id,
    "trace_id": trace_id,
    "trace_hash": compute_trace_hash(correlation_id),
    "latency_ms": int((time.perf_counter() - run_start_time) * 1000),
    "tools_used": list(tools_used_set),
    "tool_count": len(tools_used_set),
    "memory": {
        "status": memory_status or "none",
        "source": memory_source,
        "chars": len(memory_section) if memory_section else 0,
    },
    "confidence": {
        "level": confidence_level,
        "score": confidence_score,
        "reason": confidence_reason,
    },
}

msg_id = await save_message(
    conversation_uuid, "assistant", full_response,
    tenant_id=tenant_id, metadata=msg_metadata
)
```

Punti save_message() da modificare:

- Line 1446 (SUPER_TOOL path)
- Line 1496 (LEXORC path)
- Line 1552 (LEGIS path)
- Line 2281 (QUICK_FIX path)

---

### TEAM ALPHA — Agent A1-CODER-3: Prometheus Metrics Extensions

**Task 3A: Nuove metriche**

File: `lexe-core/src/lexe_core/gateway/metrics.py` (estendere)

Aggiungere dopo le metriche esistenti (line ~50):

```python
# Pipeline routing
PIPELINE_ROUTED = Counter(
    "lexe_pipeline_routed_total",
    "Queries routed per pipeline",
    ["pipeline"],  # legis, toolloop, lexorc, quickfix, orchestrator
)

# Confidence distribution
CONFIDENCE_LEVEL = Counter(
    "lexe_confidence_level_total",
    "Confidence levels emitted",
    ["level"],  # VERIFIED, CAUTION, LOW_CONFIDENCE
)

# Memory retrieval
MEMORY_RETRIEVAL = Counter(
    "lexe_memory_retrieval_total",
    "Memory retrieval outcomes",
    ["status"],  # retrieved, empty, error, circuit_open
)
MEMORY_RETRIEVAL_LATENCY = Histogram(
    "lexe_memory_retrieval_latency_seconds",
    "Memory retrieval latency",
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.0, 5.0],
)

# Evaluation
EVALUATION_RATING = Histogram(
    "lexe_evaluation_rating",
    "User evaluation ratings",
    ["tenant_id"],
    buckets=[1, 2, 3, 4, 5],
)

# Event sink
EVENT_SINK_BATCH_SIZE = Histogram(
    "lexe_event_sink_batch_size",
    "Events per sink batch",
    buckets=[1, 5, 10, 20, 50],
)
EVENT_SINK_FLUSH_LATENCY = Histogram(
    "lexe_event_sink_flush_latency_seconds",
    "Event sink flush latency",
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0],
)
EVENT_SINK_ERRORS = Counter(
    "lexe_event_sink_errors_total",
    "Event sink errors (drops, flush failures)",
)
```

**Task 3B: Instrumentare customer_router**

Aggiungere incrementi nei punti di routing:

```python
# Line ~1404 (SUPER_TOOL)
PIPELINE_ROUTED.labels(pipeline="super_tool").inc()

# Line ~1456 (LEXORC)
PIPELINE_ROUTED.labels(pipeline="lexorc").inc()

# Line ~1506 (LEGIS)
PIPELINE_ROUTED.labels(pipeline="legis").inc()

# Line ~1563 (QUICK_FIX)
PIPELINE_ROUTED.labels(pipeline="quickfix").inc()
```

Aggiungere confidence tracking nel done_event handler:

```python
# Dove confidence viene calcolata (pipeline.py o customer_router.py)
if confidence_level:
    CONFIDENCE_LEVEL.labels(level=confidence_level).inc()
```

Aggiungere memory tracking in memory_client.py:

```python
# In retrieve_memory_context_v2()
import time
start = time.perf_counter()
result = await _retrieve(...)
MEMORY_RETRIEVAL_LATENCY.observe(time.perf_counter() - start)
MEMORY_RETRIEVAL.labels(status="retrieved" if result else "empty").inc()
```

---

### TEAM ALPHA — Agent A1-REVIEWER: Test + Review + Backlog

**Responsabilita:**

1. Appena A1-CODER-1 consegna migration + event_sink → verifica:
   
   - SQL syntax valido
   - Indexes appropriati
   - Event sink: test unitario con mock asyncpg
   - Backpressure funziona (queue full → drop + log)
   - Graceful shutdown

2. Appena A1-CODER-2 consegna customer_router changes → verifica:
   
   - save_message() signature backward-compatible (metadata=None default)
   - Tutti i 4 call sites aggiornati
   - _emit() helper non blocca mai
   - trace_hash generato correttamente
   - Metadata JSON completo

3. Appena A1-CODER-3 consegna metrics → verifica:
   
   - Import condizionale prometheus_client (pattern esistente in metrics.py)
   - Labels corretti
   - Nessun crash se prometheus non installato

**Test da scrivere:**

File: `lexe-core/tests/test_event_sink.py` (NUOVO)

```python
# Test cases:
# 1. test_parse_sse_meta_event — parse SSE string → EventRecord
# 2. test_parse_sse_tool_call_event — parse tool_call SSE
# 3. test_parse_sse_done_event — parse done SSE
# 4. test_batch_insert_mock — mock asyncpg, verify batch INSERT
# 5. test_queue_full_drops — fill queue, verify drop + warning log
# 6. test_flush_on_timeout — verify 2s flush
# 7. test_flush_on_batch_size — verify 50-event flush
```

File: `lexe-core/tests/test_trace_hash.py` (NUOVO)

```python
# Test cases:
# 1. test_trace_hash_deterministic — same correlation_id → same hash
# 2. test_trace_hash_length — always 8 chars
# 3. test_trace_hash_charset — only uppercase + digits
# 4. test_trace_hash_unique — different correlation_ids → different hashes (statistical)
```

File: `lexe-core/tests/test_save_message_metadata.py` (NUOVO)

```python
# Test cases:
# 1. test_save_message_backward_compat — metadata=None still works
# 2. test_save_message_with_metadata — metadata dict persisted
# 3. test_metadata_contains_trace_hash — verify trace_hash in metadata
# 4. test_metadata_contains_pipeline — verify pipeline name
# 5. test_metadata_contains_confidence — verify confidence object
```

**Backlog format:** Markdown file `lexe-core/BACKLOG_PHASE1.md` con:

- [x] Task completato
- [ ] Bug trovato: descrizione + file:line
- [ ] Improvement suggerito: descrizione

---

### TEAM BETA — Agent B1-CODER-1: Migration + Evaluation Router

**Task 1A: Migration 025_message_evaluations.sql**

File: `lexe-core/migrations/025_message_evaluations.sql` (NUOVO)

```sql
-- Migration 025: Message Evaluations (User Feedback)

CREATE TABLE IF NOT EXISTS core.message_evaluations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    conversation_id UUID NOT NULL REFERENCES core.conversations(id) ON DELETE CASCADE,
    message_id UUID NOT NULL,
    user_id UUID NOT NULL,
    rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(message_id, user_id)
);

CREATE INDEX IF NOT EXISTS idx_eval_conversation
    ON core.message_evaluations(conversation_id);
CREATE INDEX IF NOT EXISTS idx_eval_tenant_time
    ON core.message_evaluations(tenant_id, created_at);
CREATE INDEX IF NOT EXISTS idx_eval_rating
    ON core.message_evaluations(tenant_id, rating);
CREATE INDEX IF NOT EXISTS idx_eval_message
    ON core.message_evaluations(message_id);

-- Tenant-level mandatory evaluation flag
ALTER TABLE core.tenants ADD COLUMN IF NOT EXISTS evaluation_mandatory BOOLEAN DEFAULT FALSE;

COMMENT ON TABLE core.message_evaluations IS
    'Per-message user evaluation (1-5 stars + optional comment)';
```

**Task 1B: Evaluation Router**

File: `lexe-core/src/lexe_core/conversation/evaluation_router.py` (NUOVO)

```python
# Endpoints:
# POST /api/v1/conversations/{conv_id}/messages/{msg_id}/evaluate
#   Body: { "rating": 4, "comment": "Ottima risposta" }
#   Auth: customer (JWT) — solo proprie conversazioni
#   Response: 201 Created
#
# GET /api/v1/conversations/{conv_id}/messages/{msg_id}/evaluation
#   Auth: customer
#   Response: { "rating": 4, "comment": "...", "created_at": "..." }
#
# GET /api/v1/admin/evaluations?period=7d&min_rating=1&max_rating=5
#   Auth: admin
#   Response: { "evaluations": [...], "stats": { "avg", "count", "distribution" } }
#
# GET /api/v1/config/evaluation
#   Auth: customer
#   Response: { "mandatory": true/false }

# Pattern: segui admin/routers/conversations.py per auth + DB access
# DB: direct asyncpg (pattern da tools/logging.py)
# Metrics: EVALUATION_RATING.labels(tenant_id=str(tenant_id)).observe(rating)
```

---

### TEAM BETA — Agent B1-CODER-2: Trace Hash + done_event Extension

**Task 2A: Trace hash function**

File: `lexe-core/src/lexe_core/gateway/sse_contracts.py` (estendere)

Aggiungere dopo `generate_ids()` (line 55):

```python
def compute_trace_hash(correlation_id: str) -> str:
    """Generate 8-char human-readable hash from correlation_id."""
    import hashlib
    digest = hashlib.sha256(correlation_id.encode()).digest()
    chars = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"  # No I/O/0/1 per leggibilita
    return "".join(chars[b % len(chars)] for b in digest[:8])
```

**Task 2B: Estendere done_event()**

File: `lexe-core/src/lexe_core/gateway/sse_contracts.py:267-306`

Aggiungere parametri a `done_event()`:

```python
def done_event(
    correlation_id: str,
    trace_id: str,
    conversation_id: str,
    tokens_used: int = 0,
    follow_ups: list[str] | None = None,
    memory_status: str | None = None,
    memory_intent: str | None = None,
    memory_source: str | None = None,
    memories_stored: int | None = None,
    memory_latency_ms: int | None = None,
    conversation_mode: str | None = None,
    conversation_mode_locked: bool = False,
    # NUOVI PARAMETRI:
    trace_hash: str | None = None,
    confidence_level: str | None = None,
    confidence_score: int | None = None,
    confidence_reason: str | None = None,
    pipeline: str | None = None,
    model: str | None = None,
    latency_ms: int | None = None,
) -> str:
```

Aggiungere i nuovi campi al payload `data`:

```python
data = {
    "type": "done",
    ...existing...,
    # Nuovi
    "trace_hash": trace_hash,
    "confidence_level": confidence_level,
    "confidence_score": confidence_score,
    "confidence_reason": confidence_reason,
    "pipeline": pipeline,
    "model": model,
    "latency_ms": latency_ms,
}
```

**Task 2C: Aggiornare tutte le call sites di done_event()**

In customer_router.py, aggiungere i nuovi parametri dove done_event() viene chiamato:

- LEGIS path (line ~1550): gia ha confidence dal pipeline, aggiungere trace_hash + pipeline + model + latency
- SUPER_TOOL path: aggiungere trace_hash + pipeline="super_tool" + model + latency
- LEXORC path: aggiungere trace_hash + pipeline="lexorc" + model + latency
- QUICK_FIX path: aggiungere trace_hash + pipeline="quickfix" + model + latency

---

### TEAM BETA — Agent B1-CODER-3: Feature Flags + Router Registration

**Task 3A: Feature flags**

File: `lexe-core/src/lexe_core/config.py` (estendere)

Aggiungere dopo le flags esistenti (line ~104):

```python
# Event persistence
ff_event_persistence: bool = True  # LEXE_FF_EVENT_PERSISTENCE

# Evaluation system
ff_evaluation_enabled: bool = True  # LEXE_FF_EVALUATION_ENABLED
ff_evaluation_mandatory_global: bool = False  # LEXE_FF_EVALUATION_MANDATORY_GLOBAL

# Extended metrics
ff_extended_metrics: bool = True  # LEXE_FF_EXTENDED_METRICS

# Assessment
ff_assessment_enabled: bool = True  # LEXE_FF_ASSESSMENT_ENABLED
```

**Task 3B: Router registration**

File: `lexe-core/src/lexe_core/main.py` (estendere)

Aggiungere dopo i router esistenti (line ~144):

```python
# Evaluation router
from lexe_core.conversation.evaluation_router import router as evaluation_router
app.include_router(evaluation_router, prefix="/api/v1", tags=["evaluation"])
```

**Task 3C: Startup event per EventSink**

File: `lexe-core/src/lexe_core/main.py` (estendere)

```python
from lexe_core.gateway.event_sink import EventSink, get_event_sink

@app.on_event("startup")
async def startup_event_sink():
    sink = get_event_sink()
    await sink.start()

@app.on_event("shutdown")
async def shutdown_event_sink():
    sink = get_event_sink()
    await sink.stop()  # Flush remaining events
```

---

### TEAM BETA — Agent B1-REVIEWER: Test + Review + Backlog

**Test da scrivere:**

File: `lexe-core/tests/test_evaluation.py` (NUOVO)

```python
# Test cases:
# 1. test_submit_evaluation_valid — rating 1-5 → 201
# 2. test_submit_evaluation_invalid_rating — rating 0 or 6 → 422
# 3. test_submit_evaluation_duplicate — same message+user → update, not duplicate
# 4. test_get_evaluation — retrieve after submit
# 5. test_evaluation_stats — aggregate stats correct
# 6. test_config_evaluation_default — mandatory=false by default
# 7. test_config_evaluation_tenant — tenant flag overrides
# 8. test_config_evaluation_global — global flag overrides tenant
```

File: `lexe-core/tests/test_done_event_extended.py` (NUOVO)

```python
# Test cases:
# 1. test_done_event_trace_hash — trace_hash in payload
# 2. test_done_event_confidence — confidence fields in payload
# 3. test_done_event_backward_compat — old callers still work (None defaults)
# 4. test_done_event_pipeline — pipeline name in payload
```

**Backlog:** `lexe-core/BACKLOG_PHASE1_BETA.md`

---

## Cooperative Test Checkpoint 1

**Quando:** Entrambi i team hanno completato Fase 1
**Chi:** Io (utente) + Claude
**Come:**

1. Applicare migration 024 + 025 su staging DB:
   
   ```bash
   ssh -i ~/.ssh/id_stage_new root@91.99.229.111
   docker exec lexe-postgres psql -U lexe -d lexe -f /migrations/024_conversation_events.sql
   docker exec lexe-postgres psql -U lexe -d lexe -f /migrations/025_message_evaluations.sql
   ```

2. Build + deploy lexe-core su staging

3. Inviare una query LEGIS dal webchat

4. Verificare:
   
   - `SELECT count(*) FROM core.conversation_events WHERE correlation_id = 'xxx';` → >10 eventi
   - `SELECT metadata_ FROM core.conversation_messages WHERE role='assistant' ORDER BY created_at DESC LIMIT 1;` → metadata con trace_hash, pipeline, confidence
   - `POST /conversations/{id}/messages/{mid}/evaluate` con `{rating: 5}` → 201
   - `GET /config/evaluation` → `{"mandatory": false}`
   - `curl https://api.stage.lexe.pro/metrics | grep lexe_pipeline_routed` → counter incrementato

5. Documentare risultati e bug in backlog

---

# ============================================================================

# FASE 2: EXPORT & FRONTEND INTEGRATION

# ============================================================================

## Obiettivo

Export conversazioni role-based + frontend trace hash + evaluation UI.
**Risultato:** L'utente vede il trace hash, puo esportare la chat, puo valutare le risposte.

## Team Structure

```
TEAM ALPHA — "Export Engine" (Backend Export + Trace Lookup)
├── Agent A2-CODER-1: Export router (role-based endpoints)
├── Agent A2-CODER-2: HTML renderer (brand LEXE, chat-like)
├── Agent A2-CODER-3: Admin trace lookup + admin evaluations stats
└── Agent A2-REVIEWER: Test + review + backlog

TEAM BETA — "Frontend UX" (Trace Hash + Export + Evaluation UI)
├── Agent B2-CODER-1: ChatMessage trace hash badge + streamStore extraction
├── Agent B2-CODER-2: Export dropdown in header/footer
├── Agent B2-CODER-3: EvaluationStars + EvaluationModal + evaluationStore
└── Agent B2-REVIEWER: Test + review + backlog
```

---

### TEAM ALPHA — Agent A2-CODER-1: Export Router

File: `lexe-core/src/lexe_core/conversation/export_router.py` (NUOVO)

```python
router = APIRouter()

@router.get("/conversations/{conversation_id}/export")
async def export_conversation(
    conversation_id: UUID,
    format: str = Query("json", regex="^(json|md|html)$"),
    role: str = Query("user", regex="^(user|admin|global_admin)$"),
    include_events: bool = Query(False),
    customer: CurrentCustomer = Depends(get_current_customer),
):
    # 1. Auth check: user can only export own conversations
    # 2. Fetch messages + metadata from DB
    # 3. If role=admin: fetch tool_execution_logs + evaluations
    # 4. If role=global_admin + include_events: fetch conversation_events
    # 5. Format output based on format parameter
    # 6. Return appropriate content type
```

**Livello 1 (user):** Solo messaggi con trace_hash. No metadata tecnici.
**Livello 2 (admin):** Messaggi + metadata completi + tool logs + evaluations.
**Livello 3 (global_admin):** Tutto + raw SSE events.

Auth enforcement:

- User: verifica conversation.contact_id == customer.contact_id
- Admin: verifica conversation.tenant_id == customer.tenant_id (richiede ruolo admin)
- Global Admin: nessun filtro (richiede ruolo global_admin)

---

### TEAM ALPHA — Agent A2-CODER-2: HTML Renderer

File: `lexe-core/src/lexe_core/conversation/html_renderer.py` (NUOVO)

Funzione: `render_conversation_html(messages, metadata, role, evaluations=None) -> str`

Template HTML standalone con:

- CSS inline completo (no CDN)
- Brand LEXE: `--navy: #1a365d`, `--gold: #c9a227`, `--green: #2d8659`
- Layout chat-like:
  - User bubbles: allineate a destra, background navy scuro
  - Assistant bubbles: allineate a sinistra, background grigio chiaro
- Markdown rendering: `<strong>`, `<em>`, `<code>`, `<a href>`, `<ul>/<ol>/<li>`
  - Links cliccabili (target="_blank")
  - Code blocks con syntax highlighting base (monospace + background)
- Trace hash badge: piccolo tag sotto ogni risposta (es. `ref: K7X2M9PL`)
- Se role=admin:
  - Sezione collapsibile `<details>` per ogni tool execution
  - Badge confidence: VERIFIED (green), CAUTION (amber), LOW (red)
  - Footer metadata: modello, tokens, latency, pipeline
  - Evaluations rating (stelle)
- Print-ready: `@media print { ... }` con margini, page-break
- Responsive: mobile-friendly

Pattern da seguire: simile a Kimera skill template ma piu semplice.

---

### TEAM ALPHA — Agent A2-CODER-3: Admin Trace Lookup + Evaluation Stats

**Task 3A: Trace Lookup**

File: `lexe-core/src/lexe_core/admin/routers/trace_router.py` (NUOVO)

```python
@router.get("/trace/{hash}")
async def lookup_trace(hash: str, admin = Depends(require_admin)):
    # Query: SELECT conversation_id, message_id, correlation_id, created_at
    #   FROM core.conversation_messages
    #   WHERE metadata_->>'trace_hash' = $1
    # Also: SELECT count(*) FROM core.conversation_events WHERE correlation_id = ...
    # Return: { conversation_id, message_id, correlation_id, events_count, created_at }
```

Registrare in `admin/router.py`:

```python
from lexe_core.admin.routers.trace_router import router as trace_router
admin_router.include_router(trace_router, prefix="/trace", tags=["trace"])
```

**Task 3B: Evaluation Stats Endpoint**

Estendere `evaluation_router.py`:

```python
@router.get("/admin/evaluations")
async def get_evaluation_stats(
    period: str = Query("7d"),
    admin = Depends(require_admin),
):
    # Query aggregate stats from message_evaluations
    # Return: {
    #   "period": "...",
    #   "total_evaluations": 150,
    #   "avg_rating": 4.2,
    #   "distribution": {1: 5, 2: 10, 3: 25, 4: 60, 5: 50},
    #   "lowest_rated": [{message_id, conversation_id, rating, comment}],
    #   "highest_rated": [{message_id, conversation_id, rating}],
    # }
```

---

### TEAM ALPHA — Agent A2-REVIEWER

**Test da scrivere:**

File: `lexe-core/tests/test_export.py` (NUOVO)

```python
# 1. test_export_json_user — solo messaggi, no metadata tecnici
# 2. test_export_json_admin — messaggi + metadata + tools
# 3. test_export_json_global_admin_events — include events
# 4. test_export_html_user — HTML valido, trace hash presente, no metadata
# 5. test_export_html_admin — HTML con tool details, confidence badge
# 6. test_export_md_user — markdown formattato
# 7. test_export_auth_user_own_only — user non puo esportare altrui
# 8. test_export_auth_admin_same_tenant — admin limitato al tenant
# 9. test_trace_lookup_found — hash esistente → 200
# 10. test_trace_lookup_not_found — hash inesistente → 404
# 11. test_evaluation_stats_period — stats corretti per periodo
```

---

### TEAM BETA — Agent B2-CODER-1: Trace Hash Badge + Stream Extraction

**Task 1A: streamStore trace hash extraction**

File: `lexe-webchat/src/stores/streamStore.ts`

Nel handler del done event (line ~382-403), aggiungere:

```typescript
case "done": {
    const traceHash = data.trace_hash as string || null;
    // ... existing confidence extraction ...
    set({
        ...existing...,
        traceHash,
    });
    break;
}
```

Aggiungere allo state interface:

```typescript
traceHash: string | null;  // in defaultState: null
```

Aggiungere selector:

```typescript
export const selectTraceHash = (state: StreamState) => state.traceHash;
```

**Task 1B: chatStore — persist trace hash**

File: `lexe-webchat/src/stores/chatStore.ts`

Nell'interface `ChatMessage` (line ~26-45), aggiungere:

```typescript
traceHash?: string;
```

Quando il messaggio assistant viene finalizzato (streaming complete), salvare il traceHash dal streamStore nel ChatMessage.

**Task 1C: ChatMessage trace hash badge**

File: `lexe-webchat/src/components/chat/ChatMessage.tsx`

Dopo il confidence badge (line ~464), aggiungere per assistant messages:

```tsx
{/* Trace Hash Badge */}
{!isUser && traceHash && (
    <button
        onClick={() => {
            navigator.clipboard.writeText(traceHash);
            // toast or visual feedback
        }}
        className="inline-flex items-center gap-1 text-[10px] text-muted-foreground/50 hover:text-muted-foreground/80 transition-colors cursor-pointer"
        title="Copia codice di riferimento"
    >
        <Hash className="w-3 h-3" />
        <span className="font-mono">{traceHash}</span>
    </button>
)}
```

---

### TEAM BETA — Agent B2-CODER-2: Export Dropdown

**Task 2A: Export dropdown nel footer toolbar**

File: `lexe-webchat/src/App.tsx`

Nella footer toolbar area (line ~972-1032), aggiungere un bottone export con DropdownMenu:

```tsx
import { DropdownMenu, DropdownMenuTrigger, DropdownMenuContent, DropdownMenuItem } from "@/components/ui/DropdownMenu";
import { FileDown } from "lucide-react";

// Nel footer toolbar, dopo help icon:
<DropdownMenu>
    <DropdownMenuTrigger asChild>
        <button className="..." title="Esporta conversazione">
            <FileDown className="w-4 h-4" />
        </button>
    </DropdownMenuTrigger>
    <DropdownMenuContent align="end" side="top">
        <DropdownMenuItem onClick={() => handleExport("html")}>
            Scarica HTML
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => handleExport("json")}>
            Scarica JSON
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => handleExport("md")}>
            Scarica Markdown
        </DropdownMenuItem>
    </DropdownMenuContent>
</DropdownMenu>
```

**Task 2B: Export handler**

```typescript
const handleExport = useCallback(async (format: "html" | "json" | "md") => {
    const conversationId = useChatStore.getState().activeConversationId;
    if (!conversationId) return;

    const client = getApiClient();
    const response = await client.get(`/conversations/${conversationId}/export?format=${format}&role=user`);

    const blob = new Blob(
        [typeof response === "string" ? response : JSON.stringify(response, null, 2)],
        { type: format === "html" ? "text/html" : format === "json" ? "application/json" : "text/markdown" }
    );
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `lexe-chat-${conversationId.slice(0, 8)}.${format === "md" ? "md" : format}`;
    a.click();
    URL.revokeObjectURL(url);
}, []);
```

---

### TEAM BETA — Agent B2-CODER-3: Evaluation UI

**Task 3A: EvaluationStars component**

File: `lexe-webchat/src/components/chat/EvaluationStars.tsx` (NUOVO)

```tsx
// Props: messageId, conversationId, existingRating?, onEvaluated
// - 5 stelle cliccabili (Star icon, filled/outline)
// - Hover: stelle si illuminano progressivamente
// - Click: POST /conversations/{id}/messages/{mid}/evaluate
// - Dopo submit: stelle gold fisse, breve feedback "Grazie!"
// - Sotto le stelle: textarea opzionale per commento (toggle con link "Aggiungi commento")
// - Posizionamento: sotto ogni risposta assistant, inline, discreto
// - Colori: stelle gold (#c9a227) per filled, muted per outline
```

**Task 3B: EvaluationModal component**

File: `lexe-webchat/src/components/chat/EvaluationModal.tsx` (NUOVO)

```tsx
// Modale che appare quando evaluation_mandatory=true e ultima risposta non valutata
// - Overlay semi-trasparente z-[200]
// - Card centrata con: "Valuta la risposta precedente per continuare"
// - 5 stelle grandi (cliccabili)
// - Textarea "Commento (opzionale)"
// - Bottone "Invia valutazione" (disabled finche rating=0)
// - NO bottone chiudi (obbligatorio)
// - Dopo submit: modale si chiude, input si riabilita
// Pattern: seguire StrategicPause.tsx per AnimatePresence + motion.div
```

**Task 3C: evaluationStore**

File: `lexe-webchat/src/stores/evaluationStore.ts` (NUOVO)

```typescript
interface EvaluationState {
    pendingEvaluation: boolean;
    lastAssistantMessageId: string | null;
    mandatoryEvaluation: boolean;  // from config endpoint
    evaluations: Record<string, { rating: number; comment?: string }>;  // messageId -> eval
}

// Actions:
// - setMandatory(val) — set from config endpoint at boot
// - markPending(messageId) — called when assistant response received
// - submitEvaluation(messageId, rating, comment) — POST + clear pending
// - clearPending() — after submit
// - isEvaluated(messageId) — check if already rated
```

**Task 3D: Integrare nel chat input**

File: `lexe-webchat/src/App.tsx:919-933`

```tsx
const evaluationPending = useEvaluationStore((s) => s.pendingEvaluation && s.mandatoryEvaluation);

<textarea
    ...
    disabled={isStreaming || evaluationPending}
    ...
/>

{/* Evaluation Modal */}
<AnimatePresence>
    {evaluationPending && (
        <EvaluationModal
            messageId={useEvaluationStore.getState().lastAssistantMessageId!}
            conversationId={activeConversationId}
            onEvaluated={() => useEvaluationStore.getState().clearPending()}
        />
    )}
</AnimatePresence>
```

**Task 3E: Config fetch al boot**

All'avvio dell'app, fetchare `GET /config/evaluation` e settare `mandatoryEvaluation` nello store.

---

### TEAM BETA — Agent B2-REVIEWER

**Test da scrivere:**

File: `lexe-webchat/src/components/chat/__tests__/EvaluationStars.test.tsx` (NUOVO)

```typescript
// 1. test_renders_5_stars — 5 star icons rendered
// 2. test_click_star_submits — click star 4 → POST with rating:4
// 3. test_hover_illuminates — hover star 3 → first 3 highlighted
// 4. test_existing_rating_displayed — existingRating=4 → 4 filled stars
// 5. test_comment_toggle — click "Aggiungi commento" shows textarea
```

File: `lexe-webchat/src/components/chat/__tests__/EvaluationModal.test.tsx` (NUOVO)

```typescript
// 1. test_modal_renders_when_pending — visible when pendingEvaluation=true
// 2. test_modal_blocks_input — textarea disabled when modal shown
// 3. test_modal_no_close_button — no X or escape
// 4. test_submit_closes_modal — rating + submit → modal disappears
// 5. test_submit_button_disabled_without_rating — can't submit with 0 stars
```

---

## Cooperative Test Checkpoint 2

**Quando:** Entrambi i team hanno completato Fase 2
**Come:**

1. Deploy backend + frontend su staging
2. Test flow completo:
   - Inviare query LEGIS
   - Verificare trace hash visibile sotto la risposta
   - Cliccare trace hash → copiato negli appunti
   - Esportare conversazione HTML → file scaricato, apertura in browser, layout corretto
   - Esportare JSON → file con messaggi + trace hash
   - Valutare risposta con 4 stelle → stelle gold fisse
   - Attivare evaluation_mandatory → modale compare, input bloccato
   - Valutare → modale si chiude, input riabilitato
3. Test admin:
   - `GET /admin/trace/{hash}` → conversation trovata
   - `GET /admin/evaluations?period=7d` → stats corretti
   - Export con role=admin → metadata tecnici visibili

---

# ============================================================================

# FASE 3: METRICS, MONITORING & ASSESSMENT

# ============================================================================

## Obiettivo

Dashboard operativa + alerting + assessment report.
**Risultato:** Visibilita completa su performance, qualita, anomalie.

## Team Structure

```
TEAM ALPHA — "Metrics & Assessment" (Backend)
├── Agent A3-CODER-1: Assessment report endpoint
├── Agent A3-CODER-2: SLA persistence (migrate from in-memory)
└── Agent A3-REVIEWER: Test + review + backlog

TEAM BETA — "Monitoring Stack" (Infra + Dashboards)
├── Agent B3-CODER-1: docker-compose monitoring services
├── Agent B3-CODER-2: Grafana dashboards + alerting rules
└── Agent B3-REVIEWER: Validate dashboards + test alerts
```

---

### TEAM ALPHA — Agent A3-CODER-1: Assessment Report

File: `lexe-core/src/lexe_core/admin/routers/assessment_router.py` (NUOVO)

```python
@router.get("/assessment")
async def get_assessment(
    period: str = Query("7d"),  # 1d, 7d, 30d
    admin = Depends(require_admin),
):
    # Query aggregates from:
    # 1. conversation_events → query counts per pipeline, error rates
    # 2. conversation_messages metadata → latency, tokens, confidence, memory
    # 3. tool_execution_logs → tool health
    # 4. message_evaluations → user satisfaction
    #
    # Compute:
    # - queries: {total, legis, toolloop, lexorc, quickfix}
    # - quality: {confidence_distribution, avg_score, tool_success_rate, memory_hit_rate}
    # - performance: {latency_p50/p95/p99, error_rate, avg_tokens}
    # - satisfaction: {avg_rating, rating_distribution, evaluation_rate}
    # - anomalies: [{type, timestamp, value, details}]
    # - excellence: [{type, count, details}]
    # - recommendations: [string]
```

Anomaly detection:

- Latency spike: p95 > 2x media periodo precedente
- Tool failure cluster: >3 failures dello stesso tool in 1h
- Confidence drop: >30% LOW_CONFIDENCE in 24h
- Memory circuit open: qualsiasi trip

Excellence detection:

- High confidence streak: 25+ risposte VERIFIED consecutive
- Fast response: risposte sotto 2s con VERIFIED confidence
- High satisfaction: rating medio >4.5 in 24h

---

### TEAM ALPHA — Agent A3-CODER-2: SLA Persistence

File: `lexe-core/src/lexe_core/gateway/sla_router.py` (modificare)

Migrare da in-memory counters a query su `conversation_events` + metadata:

```python
# PRIMA (in-memory, si perde al restart):
_metrics = {"queries_total": 0, ...}

# DOPO (query da DB):
async def get_sla_metrics():
    # Query conversation_events per conteggi pipeline
    # Query conversation_messages metadata per latency, confidence
    # Calcolo p50/p95 da metadata_.latency_ms
    # Cache in-memory per 60s per non sovraccaricare DB
```

---

### TEAM BETA — Agent B3-CODER-1: Docker Compose Monitoring

File: `lexe-infra/docker-compose.monitoring.yml` (NUOVO, override file)

```yaml
services:
  lexe-prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - lexe_internal

  lexe-grafana:
    image: grafana/grafana:latest
    volumes:
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: "${GRAFANA_ADMIN_PASSWORD}"
    networks:
      - lexe_internal

  lexe-loki:
    image: grafana/loki:latest
    volumes:
      - loki-data:/loki
    ports:
      - "3100:3100"
    networks:
      - lexe_internal
```

File: `lexe-infra/monitoring/prometheus.yml` (NUOVO)

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'lexe-core'
    static_configs:
      - targets: ['lexe-core:8000']
    metrics_path: /metrics
```

---

### TEAM BETA — Agent B3-CODER-2: Grafana Dashboards + Alerts

File: `lexe-infra/monitoring/grafana/dashboards/lexe-operations.json` (NUOVO)

10 pannelli:

1. Query Volume (rate by pipeline)
2. Latency P50/P95/P99 (histogram)
3. Error Rate (%)
4. Tool Health (success/error/timeout per tool)
5. Confidence Distribution (pie chart)
6. Memory Hit Rate (gauge)
7. Token Usage (stacked bar)
8. Active SSE Connections (gauge)
9. Model Usage (bar chart)
10. User Satisfaction (avg rating + distribution)

File: `lexe-infra/monitoring/grafana/provisioning/alerting/lexe-alerts.yml` (NUOVO)

6 regole alert per Grafana Alerting.

---

## Cooperative Test Checkpoint 3

1. SSH ai server, verificare docker-compose monitoring.yml
2. Avviare stack monitoring: `docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d`
3. Verificare:
   - Prometheus scrapes lexe-core/metrics
   - Grafana dashboard mostra dati
   - Assessment endpoint ritorna dati corretti
   - SLA non piu in-memory (restart-safe)
4. Test alert: generare errori artificiali, verificare alert trigger

---

# ============================================================================

# FASE 4: AUTO-IMPROVE FOUNDATION

# ============================================================================

## Obiettivo

Creare `lexe-improve-lab` repo con data collector, dataset builder, e issue detector.
**Risultato:** Primo dataset JSONL dalle conversazioni reali, primi issue detectati.

## Team Structure

```
TEAM ALPHA — "Data Pipeline" (Collection + Dataset)
├── Agent A4-CODER-1: Repo setup + data collector
├── Agent A4-CODER-2: Dataset builder (JSONL)
├── Agent A4-CODER-3: Replay runner (sandbox)
└── Agent A4-REVIEWER: Test pipeline end-to-end

TEAM BETA — "Intelligence" (Detection + Proposal)
├── Agent B4-CODER-1: Issue detector + classifier
├── Agent B4-CODER-2: Fix proposer + experiment registry
├── Agent B4-CODER-3: Offline evaluation suite
└── Agent B4-REVIEWER: Test detection + evaluation accuracy
```

---

### TEAM ALPHA — Agent A4-CODER-1: Repo Setup + Collector

**Task 1A: Repo structure**

Creare `C:\PROJECTS\lexe-genesis\lexe-improve-lab\` con:

```
lexe-improve-lab/
├── pyproject.toml          # Python 3.11+, asyncpg, click, pydantic
├── README.md               # Overview, setup, CLI usage
├── src/
│   └── lexe_improve/
│       ├── __init__.py
│       ├── cli.py           # Click CLI entrypoint
│       ├── config.py        # DB connection, settings
│       ├── collector/
│       │   ├── __init__.py
│       │   ├── conversations.py   # Fetch conversations + messages + metadata
│       │   ├── events.py          # Fetch conversation_events
│       │   ├── tools.py           # Fetch tool_execution_logs
│       │   └── evaluations.py     # Fetch message_evaluations
│       ├── datasets/
│       │   ├── __init__.py
│       │   └── builder.py         # Build JSONL datasets
│       ├── detectors/
│       │   ├── __init__.py
│       │   └── issues.py          # Issue detection
│       ├── proposers/
│       │   ├── __init__.py
│       │   └── fixes.py           # Fix proposals
│       ├── evaluators/
│       │   ├── __init__.py
│       │   └── offline.py         # Offline evaluation
│       ├── replay/
│       │   ├── __init__.py
│       │   └── runner.py          # Replay runner
│       └── models.py              # Pydantic models
├── tests/
│   ├── __init__.py
│   ├── test_collector.py
│   ├── test_datasets.py
│   └── test_detectors.py
├── experiments/
│   └── README.md
└── data/
    └── .gitkeep
```

**Task 1B: Data Collector**

```python
# collector/conversations.py
class ConversationCollector:
    async def collect(
        self,
        period: str = "7d",
        tenant_id: UUID | None = None,
        min_messages: int = 2,
    ) -> list[ConversationRecord]:
        """
        Fetch conversations with:
        - All messages (user + assistant) with metadata
        - All conversation_events
        - All tool_execution_logs
        - All message_evaluations
        Returns: list of ConversationRecord (pydantic model)
        """
```

```python
# models.py
class MessageRecord(BaseModel):
    id: UUID
    role: str
    content: str
    metadata: dict  # pipeline, model, tokens, confidence, trace_hash, ...
    evaluation: EvaluationRecord | None

class EventRecord(BaseModel):
    event_type: str
    event_data: dict
    created_at: datetime

class ConversationRecord(BaseModel):
    id: UUID
    tenant_id: UUID
    messages: list[MessageRecord]
    events: list[EventRecord]
    tool_logs: list[ToolLogRecord]
    total_latency_ms: int | None
    pipeline: str | None
    confidence_level: str | None
    user_rating: int | None
```

---

### TEAM ALPHA — Agent A4-CODER-2: Dataset Builder

```python
# datasets/builder.py
class DatasetBuilder:
    def build(
        self,
        conversations: list[ConversationRecord],
        output_path: Path,
        format: str = "jsonl",
        include_events: bool = True,
    ) -> DatasetMetadata:
        """
        Build JSONL dataset for offline evaluation.
        Each line = one conversation with all context.

        Columns:
        - conversation_id
        - user_query (last user message)
        - assistant_response (last assistant message)
        - pipeline
        - model
        - tokens_used
        - latency_ms
        - tools_used: list[str]
        - tool_results: list[dict]
        - confidence_level
        - confidence_score
        - user_rating (1-5 or null)
        - memory_status
        - events: list[dict] (if include_events)

        Returns: DatasetMetadata with counts, date range, stats
        """
```

CLI:

```bash
lexe-improve dataset build --period 7d --output data/dataset_2026-02-23.jsonl
# Output: Built dataset with 450 conversations, 3200 messages, avg 4.2 rating
```

---

### TEAM ALPHA — Agent A4-CODER-3: Replay Runner

```python
# replay/runner.py
class ReplayRunner:
    async def replay(
        self,
        dataset_path: Path,
        sandbox: bool = True,
        max_conversations: int = 50,
        compare_mode: str = "semantic",  # semantic, exact, tool_order
    ) -> ReplayReport:
        """
        Re-execute conversations from dataset in sandbox.
        For each conversation:
        1. Feed user query to API (or simulate)
        2. Capture response + events
        3. Compare with original:
           - Semantic similarity of response
           - Tool call order match
           - Confidence level match
           - Latency comparison
        Returns: ReplayReport with diffs, regressions, improvements
        """
```

Sandbox mode:

- Tool calls: simulated using original tool_results from dataset
- LLM calls: real (to test prompt/config changes)
- DB: read-only (no side effects)

---

### TEAM BETA — Agent B4-CODER-1: Issue Detector

```python
# detectors/issues.py
class IssueDetector:
    def detect(self, dataset_path: Path) -> list[Issue]:
        """
        Scan dataset for issues and classify them.

        Issue types:
        1. LOW_CONFIDENCE — confidence < 50 or level=LOW_CONFIDENCE
        2. TOOL_FAILURE — tool with status=failed/timeout
        3. LOW_RATING — user rating <= 2
        4. LATENCY_SPIKE — latency > 3x median
        5. MEMORY_MISS — memory_status=empty for returning user
        6. EMPTY_RESPONSE — assistant response < 50 chars
        7. HALLUCINATION_RISK — high confidence but no tool evidence
        8. TOOL_WASTE — tools called but results not used in response

        Returns: list of Issue with:
        - issue_id, type, severity (critical/warning/info)
        - conversation_id, message_id, trace_hash
        - evidence: dict with relevant data
        - suggested_fix_type: str
        """
```

Classification rules:

- Critical: LOW_RATING + LOW_CONFIDENCE (confirms user saw bad output)
- Warning: TOOL_FAILURE clusters (same tool >3 times in 24h)
- Info: LATENCY_SPIKE (may be external service)

---

### TEAM BETA — Agent B4-CODER-2: Fix Proposer + Experiment Registry

```python
# proposers/fixes.py
class FixProposer:
    def propose(self, issues: list[Issue]) -> list[FixProposal]:
        """
        Generate structured fix proposals for detected issues.

        FixProposal schema:
        - proposal_id: UUID
        - created_at: datetime
        - author: "lexe-improve-agent"
        - change_type: prompt_change | pipeline_config | tool_config | retrieval_config |
                       memory_policy | timeout_change | verifier_rubric | ui_change
        - scope: {pipeline?, tool?, model?, route?}
        - hypothesis: str — what problem it solves
        - evidence: {trace_hashes: list, conversation_ids: list, metrics_deltas: dict}
        - candidate_patch: str — git diff or config JSON
        - evaluation_plan: {holdout_size, metrics: list, acceptance_criteria: dict}
        - promotion_decision: pending | approved | rejected | needs_review
        """
```

Experiment registry (DB-backed):

```python
# In lexe-core migration 026_experiments.sql
CREATE TABLE IF NOT EXISTS core.experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, running, completed, failed, promoted
    hypothesis TEXT,
    baseline_config JSONB,
    candidate_config JSONB,
    dataset_path TEXT,
    metrics_baseline JSONB,
    metrics_candidate JSONB,
    metrics_delta JSONB,
    holdout_results JSONB,
    promotion_decision VARCHAR(20),  -- approved, rejected, needs_review
    created_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
```

---

### TEAM BETA — Agent B4-CODER-3: Offline Evaluation Suite

```python
# evaluators/offline.py
class OfflineEvaluator:
    def evaluate(
        self,
        baseline_results: Path,
        candidate_results: Path,
        holdout_results: Path | None = None,
        metrics: list[str] = ["confidence", "latency", "tool_success", "semantic_similarity"],
    ) -> EvaluationReport:
        """
        Compare baseline vs candidate across multiple metrics.

        Metrics:
        - confidence_avg: average confidence score
        - confidence_verified_pct: % VERIFIED
        - latency_p50/p95: response time
        - tool_success_rate: % successful tool calls
        - tool_waste_rate: % tools called but unused
        - semantic_similarity: cosine sim of responses
        - response_length_avg: average response length

        Anti-overfitting protections:
        - Holdout set: 20% of data never seen during optimization
        - Cross-validation: 5-fold on training set
        - Multi-objective: must improve on primary metric WITHOUT degrading others
        - Manual approval gate: no auto-promotion

        Returns: EvaluationReport with:
        - metrics_baseline, metrics_candidate, metrics_delta
        - statistical_significance: p-value per metric
        - holdout_metrics: metrics on holdout set
        - recommendation: "promote" | "reject" | "needs_review"
        - warnings: [str] — overfitting indicators
        """
```

---

## Cooperative Test Checkpoint 4

1. Setup lexe-improve-lab, `pip install -e .`
2. Run first collection:
   
   ```bash
   lexe-improve collect --period 7d --output data/
   ```
   
   Verify: JSONL file created with real conversation data
3. Run issue detection:
   
   ```bash
   lexe-improve detect --dataset data/latest.jsonl
   ```
   
   Verify: issues identified and classified
4. Run first replay (sandbox):
   
   ```bash
   lexe-improve replay --dataset data/latest.jsonl --max 10 --sandbox
   ```
   
   Verify: replay report with diffs
5. Review: Are issues meaningful? Are proposals actionable?

---

# ============================================================================

# FASE 5: AUTO-IMPROVE FAST WINS (B2)

# ============================================================================

## Obiettivo

Implementare i 6 fast wins del Workstream B2.
**Risultato:** Improvement loop operativo con primi miglioramenti misurabili.

## Team Structure

```
TEAM ALPHA — "Response Quality" (Reflection + Self-Refine + Tool Policy)
├── Agent A5-CODER-1: Structured reflection (episodic memory)
├── Agent A5-CODER-2: Self-refine loop with external judge
├── Agent A5-CODER-3: ReAct tool policy hardening
└── Agent A5-REVIEWER: Test all 3 improvements

TEAM BETA — "System Hygiene" (Memory + Spec Lint + Safety)
├── Agent B5-CODER-1: Memory hygiene (gating, dedup, compression)
├── Agent B5-CODER-2: Specification linting + self-correction
├── Agent B5-CODER-3: Reward hacking defenses
└── Agent B5-REVIEWER: Test all 3 improvements
```

---

### TEAM ALPHA — Agent A5-CODER-1: Structured Reflection

File: `lexe-core/src/lexe_core/agent/reflection.py` (NUOVO)

```python
class StructuredReflection:
    """
    After errors, low confidence, or low user rating:
    store a structured reflection for future reference.
    """

    async def reflect(
        self,
        conversation_id: UUID,
        message_id: UUID,
        trigger: str,  # "error", "low_confidence", "low_rating"
        context: dict,  # user query, response, tools used, confidence
    ) -> Reflection:
        """
        Generate reflection using LLM with strict template:
        {
            "cause": "Tool normattiva_search timed out, fallback response lacked citations",
            "evidence": ["trace_hash: K7X2M9PL", "tool_status: timeout"],
            "patch_idea": "Increase normattiva timeout to 15s, add cached fallback",
            "test_idea": "Replay conversation with higher timeout, verify citations present"
        }

        Storage: core.structured_reflections table
        Bounded: max 100 reflections per tenant, LRU eviction
        Dedup: semantic similarity check before insert (>0.9 sim → skip)
        """
```

Migration: `027_reflections.sql`

Feature flag: `ff_structured_reflection` (default: false)

Trigger points in `customer_router.py`:

- After error_event() → reflect with trigger="error"
- After done_event() with confidence_level="LOW_CONFIDENCE" → reflect
- After evaluation with rating <= 2 → reflect (via webhook or async)

---

### TEAM ALPHA — Agent A5-CODER-2: Self-Refine Loop

File: `lexe-core/src/lexe_core/agent/self_refine.py` (NUOVO)

```python
class SelfRefineLoop:
    """
    For risky outputs: generate → critique → refine cycle.
    Uses verifier model or rule-based judge.
    """

    async def refine(
        self,
        user_query: str,
        initial_response: str,
        context: dict,  # tools results, memory, confidence
        max_iterations: int = 2,
    ) -> RefinedResponse:
        """
        1. Generate initial response (already done by synthesizer)
        2. Critique: run verifier model to identify issues
           - Missing citations for legal claims
           - Contradictions with tool evidence
           - Incomplete answers to multi-part questions
        3. Refine: re-run synthesizer with critique feedback
        4. Stop conditions:
           - Critique finds no issues → done
           - max_iterations reached → return best
           - Refinement didn't improve confidence → return original
        5. Log all iterations to conversation_events
        """
```

Feature flag: `ff_self_refine` (default: false)

Integration: in `pipeline.py` after Phase S, before Phase V2:

```python
if config.ff_self_refine and confidence_score < 60:
    refined = await self_refine.refine(user_query, response, context, max_iterations=2)
    if refined.confidence_score > confidence_score:
        response = refined.response
        confidence_score = refined.confidence_score
```

---

### TEAM ALPHA — Agent A5-CODER-3: ReAct Tool Policy

File: `lexe-core/src/lexe_core/agent/tool_policy.py` (NUOVO)

```python
class ToolPolicy:
    """
    Tool budget, timeout policy, and argument/result verifier.
    """

    def __init__(self, max_tools: int = 10, max_total_timeout_ms: int = 60000):
        self.max_tools = max_tools
        self.max_total_timeout_ms = max_total_timeout_ms
        self.calls = 0
        self.total_time_ms = 0

    def check_budget(self, tool_name: str) -> bool:
        """Check if another tool call is within budget."""
        if self.calls >= self.max_tools:
            logger.warning(f"Tool budget exceeded: {self.calls}/{self.max_tools}")
            return False
        return True

    def verify_arguments(self, tool_name: str, args: dict) -> tuple[bool, str]:
        """
        Verify tool arguments are reasonable.
        - String args: not empty, not too long (< 2000 chars)
        - URN/ID args: valid format
        - Numeric args: within range
        Returns: (valid, reason)
        """

    def verify_result(self, tool_name: str, result: dict) -> tuple[bool, str]:
        """
        Verify tool result is usable.
        - Not empty
        - Not error masquerading as success
        - Size reasonable (< 100KB)
        Returns: (valid, reason)
        """

    def record_misuse(self, tool_name: str, issue: str):
        """Record tool misuse pattern for analysis."""
```

Feature flag: `ff_tool_budget` (default: false)

---

### TEAM BETA — Agent B5-CODER-1: Memory Hygiene

File: `lexe-core/src/lexe_core/gateway/memory_hygiene.py` (NUOVO)

```python
class MemoryHygiene:
    """
    Memory gating, compression, and deduplication.
    """

    async def gate(self, facts: list[dict], user_query: str) -> list[dict]:
        """
        Filter memory facts by relevance to current query.
        Remove facts that are:
        - Too old (> 90 days without access)
        - Too low confidence (< 0.3)
        - Contradicted by newer facts
        """

    async def deduplicate(self, facts: list[dict]) -> list[dict]:
        """
        Remove semantically similar facts (cosine sim > 0.9).
        Keep the most recent version.
        """

    async def compress(self, facts: list[dict], max_chars: int = 4000) -> str:
        """
        Compress facts into a concise summary if total chars > max_chars.
        Uses LLM summarization with strict template.
        """

    async def detect_poisoning(self, facts: list[dict]) -> list[dict]:
        """
        Detect potential memory poisoning:
        - Facts contradicting established legal knowledge
        - Facts with suspiciously high confidence but no source
        - Facts that changed rapidly in short time
        Returns: list of suspicious facts for review
        """
```

Feature flag: `ff_memory_hygiene` (default: false)

Integration: in `memory_client.py` after `retrieve_memory_context_v2()`:

```python
if config.ff_memory_hygiene:
    hygiene = MemoryHygiene()
    facts = await hygiene.gate(raw_facts, user_query)
    facts = await hygiene.deduplicate(facts)
    memory_section = await hygiene.compress(facts)
```

---

### TEAM BETA — Agent B5-CODER-2: Specification Linting

File: `lexe-improve-lab/src/lexe_improve/spec_linter.py` (NUOVO)

```python
class SpecLinter:
    """
    Lint prompts and task specifications for quality issues.
    """

    def lint(self, spec: str, spec_type: str = "system_prompt") -> LintReport:
        """
        Detect issues in specifications:
        - Ambiguity: vague instructions ("do your best", "be helpful")
        - Underspecification: missing error handling, edge cases
        - Contradiction: conflicting instructions
        - Hackability: instructions that can be circumvented
        - Length: too long (> 5000 tokens) without structure

        Returns: LintReport with issues, severity, suggestions
        """

    def correct(self, spec: str, issues: list[LintIssue]) -> CorrectedSpec:
        """
        Apply corrections to specification.
        Returns: original, corrected, rationale for each change
        """
```

---

### TEAM BETA — Agent B5-CODER-3: Reward Hacking Defenses

File: `lexe-improve-lab/src/lexe_improve/safety.py` (NUOVO)

```python
class RewardHackingDefenses:
    """
    Prevent the improvement loop from gaming metrics.
    """

    def validate_evaluation(self, metrics: dict) -> ValidationResult:
        """
        Multi-objective validation:
        - Primary metric improved? (confidence, user rating)
        - Secondary metrics NOT degraded? (latency, tool_success, response_quality)
        - Trajectory metrics reasonable? (tool_call_count, token_usage)
        - Compliance checks passed? (citations present, legal disclaimers)

        Anti-gaming:
        - Cannot inspect holdout test data
        - Cannot modify evaluation prompts
        - Multi-objective gate: ALL metrics must be within bounds
        - Trajectory analysis: suspicious patterns flagged
        """

    def create_holdout(self, dataset_path: Path, holdout_pct: float = 0.2) -> tuple[Path, Path]:
        """Split dataset into train and holdout. Holdout is SEALED."""

    def cross_validate(self, dataset_path: Path, folds: int = 5) -> list[FoldResult]:
        """5-fold cross-validation for robustness."""
```

---

## Cooperative Test Checkpoint 5

1. Test structured reflection:
   - Force a low-confidence response, verify reflection stored
   - Check dedup: similar error → no duplicate reflection
2. Test self-refine:
   - Enable ff_self_refine, send ambiguous legal query
   - Verify: refinement loop logged, confidence improved
3. Test tool policy:
   - Enable ff_tool_budget, send query requiring many tools
   - Verify: budget enforced, warning logged at limit
4. Test memory hygiene:
   - Enable ff_memory_hygiene, verify old/duplicate facts filtered
5. Run full evaluation:
   
   ```bash
   lexe-improve evaluate --baseline baseline.jsonl --candidate candidate.jsonl
   ```
   
   Verify: report with multi-objective metrics, holdout results
6. Integration test: all features together, no regressions

---

# ============================================================================

# APPENDICE: Riepilogo Completo

# ============================================================================

## File da Creare/Modificare per Fase

### FASE 1 (18 file)

| File                                              | Azione             | Team   |
| ------------------------------------------------- | ------------------ | ------ |
| `migrations/024_conversation_events.sql`          | NUOVO              | A1     |
| `migrations/025_message_evaluations.sql`          | NUOVO              | B1     |
| `lexe-core/.../gateway/event_sink.py`             | NUOVO              | A1     |
| `lexe-core/.../gateway/customer_router.py`        | MODIFICA           | A1     |
| `lexe-core/.../gateway/sse_contracts.py`          | MODIFICA           | B1     |
| `lexe-core/.../gateway/metrics.py`                | MODIFICA           | A1     |
| `lexe-core/.../gateway/memory_client.py`          | MODIFICA (metrics) | A1     |
| `lexe-core/.../conversation/evaluation_router.py` | NUOVO              | B1     |
| `lexe-core/.../config.py`                         | MODIFICA           | B1     |
| `lexe-core/.../main.py`                           | MODIFICA           | B1     |
| `lexe-core/tests/test_event_sink.py`              | NUOVO              | A1-REV |
| `lexe-core/tests/test_trace_hash.py`              | NUOVO              | A1-REV |
| `lexe-core/tests/test_save_message_metadata.py`   | NUOVO              | A1-REV |
| `lexe-core/tests/test_evaluation.py`              | NUOVO              | B1-REV |
| `lexe-core/tests/test_done_event_extended.py`     | NUOVO              | B1-REV |
| `lexe-core/BACKLOG_PHASE1.md`                     | NUOVO              | A1-REV |
| `lexe-core/BACKLOG_PHASE1_BETA.md`                | NUOVO              | B1-REV |
| `lexe-docs/FEATURES_OBSERVABILITY.md`             | NUOVO              | -      |

### FASE 2 (14 file)

| File                                              | Azione   | Team   |
| ------------------------------------------------- | -------- | ------ |
| `lexe-core/.../conversation/export_router.py`     | NUOVO    | A2     |
| `lexe-core/.../conversation/html_renderer.py`     | NUOVO    | A2     |
| `lexe-core/.../admin/routers/trace_router.py`     | NUOVO    | A2     |
| `lexe-core/.../admin/router.py`                   | MODIFICA | A2     |
| `lexe-core/.../main.py`                           | MODIFICA | A2     |
| `lexe-core/tests/test_export.py`                  | NUOVO    | A2-REV |
| `lexe-webchat/.../stores/streamStore.ts`          | MODIFICA | B2     |
| `lexe-webchat/.../stores/chatStore.ts`            | MODIFICA | B2     |
| `lexe-webchat/.../stores/evaluationStore.ts`      | NUOVO    | B2     |
| `lexe-webchat/.../chat/ChatMessage.tsx`           | MODIFICA | B2     |
| `lexe-webchat/.../chat/EvaluationStars.tsx`       | NUOVO    | B2     |
| `lexe-webchat/.../chat/EvaluationModal.tsx`       | NUOVO    | B2     |
| `lexe-webchat/src/App.tsx`                        | MODIFICA | B2     |
| `lexe-webchat/.../__tests__/Evaluation*.test.tsx` | NUOVO    | B2-REV |

### FASE 3 (8 file)

| File                                                        | Azione   | Team   |
| ----------------------------------------------------------- | -------- | ------ |
| `lexe-core/.../admin/routers/assessment_router.py`          | NUOVO    | A3     |
| `lexe-core/.../gateway/sla_router.py`                       | MODIFICA | A3     |
| `lexe-core/.../admin/router.py`                             | MODIFICA | A3     |
| `lexe-core/tests/test_assessment.py`                        | NUOVO    | A3-REV |
| `lexe-infra/docker-compose.monitoring.yml`                  | NUOVO    | B3     |
| `lexe-infra/monitoring/prometheus.yml`                      | NUOVO    | B3     |
| `lexe-infra/monitoring/grafana/dashboards/*.json`           | NUOVO    | B3     |
| `lexe-infra/monitoring/grafana/provisioning/alerting/*.yml` | NUOVO    | B3     |

### FASE 4 (20+ file)

| File                              | Azione | Team  |
| --------------------------------- | ------ | ----- |
| `lexe-improve-lab/` (intero repo) | NUOVO  | A4+B4 |
| `migrations/026_experiments.sql`  | NUOVO  | B4    |

### FASE 5 (10 file)

| File                                      | Azione   | Team   |
| ----------------------------------------- | -------- | ------ |
| `migrations/027_reflections.sql`          | NUOVO    | A5     |
| `lexe-core/.../agent/reflection.py`       | NUOVO    | A5     |
| `lexe-core/.../agent/self_refine.py`      | NUOVO    | A5     |
| `lexe-core/.../agent/tool_policy.py`      | NUOVO    | A5     |
| `lexe-core/.../gateway/memory_hygiene.py` | NUOVO    | B5     |
| `lexe-core/.../config.py`                 | MODIFICA | A5+B5  |
| `lexe-improve-lab/.../spec_linter.py`     | NUOVO    | B5     |
| `lexe-improve-lab/.../safety.py`          | NUOVO    | B5     |
| `lexe-core/tests/test_reflection.py`      | NUOVO    | A5-REV |
| `lexe-core/tests/test_self_refine.py`     | NUOVO    | A5-REV |

---

## Timeline Esecuzione

```
FASE 1: Data Foundation          ████████░░░░░░░░░░░░░░░░░░░░░░
   [Cooperative Test 1]                    ██

FASE 2: Export & Frontend              ████████░░░░░░░░░░░░░░░░
   [Cooperative Test 2]                          ██

FASE 3: Metrics & Monitoring                  ████████░░░░░░░░░░
   [Cooperative Test 3]                                ██

FASE 4: Auto-Improve Foundation                      ████████░░░░
   [Cooperative Test 4]                                        ██

FASE 5: Auto-Improve Fast Wins                              ████████
   [Cooperative Test 5]                                              ██
```

Ogni fase produce risultati testabili e deployabili indipendentemente.
Nessuna fase blocca il production path — tutto e async, fire-and-forget, behind feature flags.
