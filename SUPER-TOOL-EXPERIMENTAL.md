# SUPER_TOOL — Pipeline Sperimentale

> Single-call Legal Agent via LiteLLM con modelli Anthropic Claude Sonnet/Opus 4.6
> Stato: **SPERIMENTALE** | Branch: `stage` | Data: 2026-02-23

---

## Panoramica

SUPER_TOOL e' una pipeline alternativa a LEGIS (5 LLM call, mix Haiku/Flash/Sonnet) che usa **una singola sessione LLM** con un modello potente (Claude Sonnet 4.6 o Opus 4.6) attraverso il gateway LiteLLM interno. Il modello gestisce autonomamente la strategia di ricerca legale con tool calling budget-aware.

| Pipeline | Modello | Call LLM | Costo/query | Latenza attesa |
|----------|---------|----------|-------------|----------------|
| **LEGIS** | mix Haiku/Flash/Sonnet (5 call) | 5 | ~$0.005-0.012 | 8-15s |
| **SUPER_TOOL Sonnet** | anthropic/claude-sonnet-4-6 (1-5 round) | 1-5 | ~$0.037 | 4-8s |
| **SUPER_TOOL Opus** | anthropic/claude-opus-4-6 (1-5 round) | 1-5 | ~$0.063 | 6-12s |

---

## Architettura

```
User → POST /customer/stream
  ↓
  [Selezione pipeline: manuale | prompt prefix | auto-route]
  ↓
SuperToolAgent.run():
  1. System prompt unificato (intent + piano + budget rules)
  2. LiteLLM /v1/chat/completions (OpenAI-compatible)
  3. Tool loop: tool_calls → bridge validate → HTTP lexe-tools:8021
  4. Envelope ToolBridgeResult(ok, tool, latency_ms, payload, error)
  5. Tool results appended to messages (context LLM)
  6. Final text → SSE streaming token events
  7. Metrics → core.super_tool_runs (fire & forget)
  ↓
  [Fallback LEGIS su qualsiasi errore]
```

### Flow SSE Events

```
super_tool_phase: analyzing
super_tool_budget: {rounds: 1, calls: 0, remaining: 12}
super_tool_phase: tooling
super_tool_tool_call: {tool: "normattiva_search", ok: true, latency_ms: 450}
super_tool_budget: {rounds: 2, calls: 1, remaining: 11}
super_tool_phase: tooling
super_tool_tool_call: {tool: "kb_search", ok: true, latency_ms: 380}
super_tool_phase: synthesizing
token: "## Inquadramento normativo..."
token: "..."
super_tool_phase: finalizing
super_tool_usage: {model, tokens, cost, duration, stop_reason}
tool_run_end: {status: COMPLETED}
```

---

## 3 Modi di Attivazione

### 1. Prompt Prefix (nessun flag necessario)

L'utente scrive nella chat:
```
/supertool Cos'e' l'art. 2043 del codice civile?
/super_tool Responsabilita' extracontrattuale in ambito condominiale
/st Prescrizione crediti lavoro subordinato
```

Prefissi riconosciuti: `/supertool`, `/super_tool`, `/st` (case-insensitive).
Il prefix viene strippato prima del processing — il modello riceve solo la domanda.

### 2. Body Field nella Request

```json
POST /customer/stream
{
  "message": "Cos'e' l'art. 2043 del codice civile?",
  "pipeline": "super_tool"
}
```

Valori validi per `pipeline`: `"super_tool"`, `"legis"`, `"lexorc"`, `"toolloop"`.
Overrides completamente l'auto-routing.

### 3. Auto-Route (per tenant abilitati)

Quando `ff_super_tool_enabled=True` (o tenant in `ff_super_tool_tenants`), le query con `PipelineLevel >= super_tool_min_level` (default STANDARD) vengono automaticamente instradate a SUPER_TOOL.

```
DIRECT  (1) ─┐
SIMPLE  (2) ─┤─→ LEGIS (sotto min_level)
STANDARD(3) ─┤─→ SUPER_TOOL (>= min_level, se abilitato)
COMPLEX (4) ─┘─→ LEXORC (se abilitato) o SUPER_TOOL
```

### Priorita' Routing

```
Manual override (body.pipeline) > Prompt prefix (/supertool) > Auto-intent
```

---

## Configurazione

### Environment Variables

| Variabile | Default | Descrizione |
|-----------|---------|-------------|
| `LEXE_FF_SUPER_TOOL_ENABLED` | `False` | Abilita globalmente per tutti i tenant |
| `LEXE_FF_SUPER_TOOL_TENANTS` | `""` | UUID tenant comma-separated per abilitazione selettiva |
| `LEXE_SUPER_TOOL_MODEL` | `anthropic/claude-sonnet-4-6` | Modello LiteLLM (deve supportare tool calling) |
| `LEXE_SUPER_TOOL_MIN_LEVEL` | `STANDARD` | Livello minimo PipelineLevel per auto-route |
| `LEXE_SUPER_TOOL_MAX_WALL_CLOCK` | `60.0` | Timeout totale in secondi |
| `LEXE_SUPER_TOOL_MAX_ROUNDS` | `5` | Max round-trip API |
| `LEXE_SUPER_TOOL_MAX_TOOL_CALLS` | `12` | Max tool call totali |
| `LEXE_SUPER_TOOL_MAX_STDOUT_BYTES` | `204800` | Max output (200KB) |
| `LEXE_FF_SUPER_TOOL_COMPARISON` | `False` | Abilita shadow comparison con LEGIS |
| `LEXE_SUPER_TOOL_SHADOW_RATE` | `0.2` | Percentuale query per shadow (1 su 5) |

### Hard Limits (non bypassabili)

| Limite | Valore | Stop Reason |
|--------|--------|-------------|
| Wall clock | 60s | `timeout` |
| API rounds | 5 | `max_rounds` |
| Tool calls totali | 12 | `max_tools` |
| Output size | 200KB | `max_stdout` |

Quando un limit viene raggiunto, l'agent genera un **output degradato** con:
- Testo gia' prodotto (se presente)
- Elenco fonti consultate con stato
- Nota esplicativa del motivo

---

## Deploy su Staging

### 1. Assicurarsi che il modello sia configurato in LiteLLM

Il modello `anthropic/claude-sonnet-4-6` deve essere disponibile nel gateway LiteLLM.
Verificare con:

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111
curl -s http://localhost:4000/v1/models \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" | jq '.data[].id' | grep anthropic
```

### 2. Applicare la migration

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111
docker exec lexe-postgres psql -U lexe -d lexe -f /dev/stdin <<'SQL'
-- Migration 023: Super Tool Agent Tracking
CREATE TABLE IF NOT EXISTS core.super_tool_runs (
    id              BIGSERIAL PRIMARY KEY,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tenant_id       UUID NOT NULL,
    contact_id      UUID,
    conversation_id UUID,
    correlation_id  TEXT NOT NULL,
    trace_id        TEXT,
    model           TEXT NOT NULL,
    pricing_version TEXT NOT NULL DEFAULT 'v1',
    input_tokens    INTEGER NOT NULL DEFAULT 0,
    output_tokens   INTEGER NOT NULL DEFAULT 0,
    cost_usd_est    NUMERIC(10,6) NOT NULL DEFAULT 0.0,
    api_rounds          INTEGER NOT NULL DEFAULT 0,
    tool_calls_count    INTEGER NOT NULL DEFAULT 0,
    tool_calls_failed   INTEGER NOT NULL DEFAULT 0,
    total_duration_ms   INTEGER NOT NULL DEFAULT 0,
    stop_reason     TEXT NOT NULL DEFAULT 'end_turn',
    fallback_reason TEXT,
    tool_call_log   JSONB
);
CREATE INDEX IF NOT EXISTS idx_super_tool_runs_tenant ON core.super_tool_runs (tenant_id, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_super_tool_runs_correlation ON core.super_tool_runs (correlation_id);
CREATE INDEX IF NOT EXISTS idx_super_tool_runs_model_date ON core.super_tool_runs (model, created_at DESC);
SQL
```

### 3. Abilitare per tenant specifico (staging)

Aggiungere all'override staging (`lexe-infra/docker-compose.override.stage.yml`):

```yaml
services:
  lexe-core:
    environment:
      - LEXE_FF_SUPER_TOOL_TENANTS=<tenant-uuid-staging>
      - LEXE_SUPER_TOOL_MODEL=anthropic/claude-sonnet-4-6
```

### 4. Build e deploy

```bash
cd /opt/lexe-platform/lexe-infra
git checkout stage && git pull origin stage

cd /opt/lexe-platform/lexe-core
git checkout stage && git pull origin stage

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-core --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-core --force-recreate

# Verifica
docker logs lexe-core --tail 50 | grep SUPER_TOOL
```

### 5. Test manuale

```bash
# Via prompt prefix (non serve FF)
curl -s https://api.stage.lexe.pro/api/v1/customer/stream \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message": "/supertool Cos'\''e'\'' l'\''art. 2043 del codice civile?"}' \
  --no-buffer

# Via pipeline field
curl -s https://api.stage.lexe.pro/api/v1/customer/stream \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message": "Art. 2043 CC", "pipeline": "super_tool"}' \
  --no-buffer
```

---

## Monitoraggio

### Query DB per analisi costi

```sql
-- Ultimi 20 run
SELECT id, created_at, model,
       input_tokens, output_tokens, cost_usd_est,
       api_rounds, tool_calls_count, tool_calls_failed,
       total_duration_ms, stop_reason, fallback_reason
FROM core.super_tool_runs
ORDER BY created_at DESC LIMIT 20;

-- Costo medio per modello
SELECT model,
       COUNT(*) as runs,
       AVG(cost_usd_est) as avg_cost,
       AVG(total_duration_ms) as avg_duration_ms,
       AVG(tool_calls_count) as avg_tools,
       SUM(CASE WHEN fallback_reason IS NOT NULL THEN 1 ELSE 0 END) as fallbacks
FROM core.super_tool_runs
GROUP BY model;

-- Tool call log per un run specifico
SELECT tool_call_log
FROM core.super_tool_runs
WHERE correlation_id = '<correlation_id>';
```

### Log Keywords

```bash
# Routing
docker logs lexe-core 2>&1 | grep -E "SUPER_TOOL_ROUTE|PIPELINE"

# Agent execution
docker logs lexe-core 2>&1 | grep "SUPER_TOOL"

# Fallback to LEGIS
docker logs lexe-core 2>&1 | grep "falling back to LEGIS"
```

---

## Riferimenti Codice

### File Nuovi

| File | Righe | Scopo |
|------|-------|-------|
| `lexe-core/src/lexe_core/agent/super_tool.py` | ~490 | `SuperToolAgent` — loop LiteLLM, tool bridge, hard stops, degraded output |
| `lexe-core/src/lexe_core/agent/super_tool_definitions.py` | ~256 | 8 tool defs Anthropic format + `VALID_TOOL_NAMES` |
| `lexe-core/src/lexe_core/agent/super_tool_events.py` | ~117 | SSE events: phase, tool_call, budget, usage |
| `lexe-core/src/lexe_core/agent/super_tool_prompts.py` | ~76 | System prompt budget-aware con citation format |
| `lexe-core/migrations/023_super_tool_tracking.sql` | ~59 | Tabella `core.super_tool_runs` + 3 indici |
| `lexe-core/tests/test_super_tool.py` | ~511 | 38 unit test + golden harness |
| `lexe-core/tests/fixtures/super_tool_golden/` | 3 file | Golden SSE sequence + tool response fixtures |

### File Modificati

| File | Sezione | Modifica |
|------|---------|----------|
| `lexe-core/src/lexe_core/config.py` | linee 101-114 | 12 settings `ff_super_tool_*` + `super_tool_*` |
| `lexe-core/src/lexe_core/config.py` | metodo | `is_super_tool_enabled_for_tenant()` + `ff_super_tool_tenant_set` |
| `lexe-core/src/lexe_core/gateway/customer_router.py` | `CustomerStreamRequest` | Campo `pipeline: str | None` |
| `lexe-core/src/lexe_core/gateway/customer_router.py` | routing (linee ~1347-1454) | Prompt prefix detection, manual override, SUPER_TOOL PATH |
| `lexe-core/src/lexe_core/gateway/customer_router.py` | `_pre_intent_route()` | Route `"super_tool"` per level >= min_level |
| `lexe-core/src/lexe_core/gateway/sse_contracts.py` | fine file | 4 helper SSE `super_tool_*_event()` |

### Classi e Dataclass Principali

| Classe | File | Ruolo |
|--------|------|-------|
| `SuperToolAgent` | `super_tool.py` | Agent principale con `run()` async generator |
| `SuperToolFallbackError` | `super_tool.py` | Exception per fallback a LEGIS |
| `ToolBridgeResult` | `super_tool.py` | Envelope: `ok`, `tool`, `latency_ms`, `payload`, `error` |
| `ToolError` | `super_tool.py` | Errore strutturato: `code` + `message` |
| `SuperToolRunMetrics` | `super_tool.py` | Metriche accumulate: tokens, rounds, tool calls, cost |

### Codici Errore Bridge

| Codice | Quando |
|--------|--------|
| `TOOL_TIMEOUT` | Tool HTTP call supera il timeout |
| `TOOL_4XX` | Tool restituisce `success: false` |
| `TOOL_5XX` | Connection error o exception |
| `TOOL_SCHEMA` | Tool name sconosciuto, input > 4KB, URL in input non-web |
| `TOOL_CIRCUIT_OPEN` | Riservato per circuit breaker futuro |

### Stop Reasons

| Valore | Significato |
|--------|-------------|
| `end_turn` | Completamento normale |
| `stop` | LLM finish_reason = stop |
| `max_rounds` | Raggiunto limite round API |
| `max_tools` | Raggiunto limite tool call |
| `timeout` | Raggiunto wall clock limit |
| `max_stdout` | Output supera 200KB |
| `error` | Errore non recuperabile |

---

## Test

```bash
# Tutti i test (38 test, nessuno richiede API key)
cd lexe-core && python -m pytest tests/test_super_tool.py -v

# Solo golden harness
python -m pytest tests/test_super_tool.py::TestGoldenSSESequence -v

# Solo config e manual selection
python -m pytest tests/test_super_tool.py::TestConfig tests/test_super_tool.py::TestManualSelection -v
```

### Copertura Test

| Area | Test | Stato |
|------|------|-------|
| Tool definitions format | 6 | OK |
| SSE events format | 5 | OK |
| Bridge envelope | 3 | OK |
| Cost calculation | 4 | OK |
| Config & tenant gating | 5 | OK |
| Manual selection & prefix | 4 | OK |
| Prompts | 3 | OK |
| Golden SSE sequence | 3 | OK |
| Block normalization | 3 | OK |
| LiteLLM pattern | 2 | OK |

---

## Pricing v1

| Modello LiteLLM | Input ($/1M) | Output ($/1M) | Costo stimato 10K in + 2K out |
|------------------|-------------|---------------|-------------------------------|
| `anthropic/claude-sonnet-4-6` | $3.00 | $15.00 | $0.06 |
| `anthropic/claude-opus-4-6` | $5.00 | $25.00 | $0.10 |

Formula: `cost = (input_tokens / 1M) * input_price + (output_tokens / 1M) * output_price`

---

## Rollback

Se SUPER_TOOL causa problemi:

```bash
# 1. Disabilitare il feature flag
# In docker-compose.override.stage.yml, rimuovere LEXE_FF_SUPER_TOOL_TENANTS

# 2. Restart
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-core --force-recreate

# 3. Verificare che il routing torna a LEGIS
docker logs lexe-core --tail 20 | grep "AUTO_ROUTE"
```

Il fallback a LEGIS e' automatico su qualsiasi errore del SuperToolAgent.
Il prompt prefix `/supertool` continua a funzionare anche con FF disabilitato (bypass manuale).

---

*Ultimo aggiornamento: 2026-02-23*
