# LEXE Platform - Observability Audit Report

**Data**: 2026-03-12
**Ambiente verificato**: Staging (91.99.229.111)
**Tipo**: Audit di sola lettura - nessuna modifica applicata

---

## 1. Infrastruttura di Observability su Staging

### 1.1 Servizi Attivi

| Servizio | Container | Stato | Porta | Note |
|----------|-----------|-------|-------|------|
| **Langfuse** | `shared-langfuse` | UP (health: starting) | 3005 | Con ClickHouse backend (`shared-langfuse-clickhouse`) |
| **Jaeger** | `shared-jaeger` | UP (healthy) | 16686 (UI), 4317-4318 (OTLP) | Nessun servizio sta inviando trace (0 services) |
| **Prometheus** | `shared-prometheus` | UP (healthy) | 9090 | Solo `lexe-core` come target attivo |
| **Grafana** | `shared-grafana` | UP (healthy) | 3001 | Dashboard `lexe-operations` configurata |
| **Loki** | `shared-loki` | UP (healthy) | 3100 | Solo label `docker` come source |
| **Fluent Bit** | `shared-fluent-bit` | UP (healthy) | 24224, 2020 | Log forwarding attivo |
| **Alertmanager** | `shared-alertmanager` | UP (healthy) | 9093 | Regole alert LEXE configurate |

### 1.2 Stato Connettivita

| Connessione | Stato | Dettaglio |
|-------------|-------|-----------|
| LiteLLM -> Langfuse | **Configurato** | `LITELLM_CALLBACKS=langfuse`, chiavi presenti |
| lexe-core -> Langfuse | **Configurato** | `LEXE_LANGFUSE_HOST`, public/secret key presenti, `ff_langfuse_spans=True` |
| lexe-core -> Jaeger (OTEL) | **Configurato ma INATTIVO** | ENV vars presenti (`OTEL_EXPORTER_OTLP_ENDPOINT=http://shared-jaeger:4317`) ma Jaeger riporta 0 servizi |
| lexe-memory -> Jaeger (OTEL) | **Configurato ma INATTIVO** | ENV vars presenti, default `OTEL_ENABLED=false` nel codice |
| lexe-tools-it -> Jaeger (OTEL) | **NON CONFIGURATO** | Nessuna ENV OTEL nel container runtime |
| lexe-tools-it -> Langfuse | **NON CONFIGURATO** | Nessuna ENV Langfuse |
| Prometheus -> lexe-core | **ATTIVO** | Scraping OK, metriche visibili |
| Prometheus -> lexe-tools-it | **NON CONFIGURATO** | Nessun scrape target per lexe-tools |
| Prometheus -> lexe-memory | **NON CONFIGURATO** | Nessun scrape target; endpoint `/metrics` protetto da HMAC |

---

## 2. Matrice di Tracing per Servizio

### Legenda
- **OK**: Configurato e funzionante in staging
- **CODE**: Codice presente ma non attivo in staging (ENV mancanti o disabilitato)
- **NONE**: Nessuna implementazione nel codice
- **PARTIAL**: Parzialmente implementato

| Servizio | Langfuse LLM Traces | OTEL Distributed Tracing | Prometheus Metrics | Loki Structured Logs |
|----------|---------------------|--------------------------|--------------------|--------------------|
| **lexe-core** | OK | CODE | OK | OK (via Fluent Bit) |
| **lexe-litellm** | OK (callback) | NONE | NONE (target down) | OK (via Fluent Bit) |
| **lexe-tools-it** | NONE | CODE | CODE (non scrappato) | OK (via Fluent Bit) |
| **lexe-memory** | NONE | CODE | CODE (HMAC-locked) | OK (via Fluent Bit) |
| **lexe-orchestrator** | NONE (disabled) | NONE | NONE | OK (via Fluent Bit) |

---

## 3. Analisi Dettagliata per Servizio

### 3.1 lexe-core

**Langfuse Integration** (OK)
- `langfuse_integration.py`: `LangfuseSpanManager` singleton con batch ingestion API
- Trace create per ogni conversation turn (`customer_router.py:1289-1291`)
- Custom span per pipeline phases
- Score per confidence e user evaluation (`evaluation_router.py:148-149`, `turn_envelope.py:198-217`)
- LiteLLM callback automatico per tutte le chiamate LLM

**OpenTelemetry** (CODE - non attivo)
- `telemetry.py`: Setup completo con FastAPI + HTTPX instrumentor
- Decoratore `@traced()` disponibile per funzioni async/sync
- ENV vars configurate in Docker (`OTEL_EXPORTER_OTLP_ENDPOINT=http://shared-jaeger:4317`)
- Jaeger non riceve dati (0 services) — probabilmente pacchetti OTEL non installati nel container o import fallisce silenziosamente

**Prometheus** (OK)
- `metrics.py`: 15+ metriche (latency, requests, errors, SSE, pipeline routing, confidence, memory retrieval, evaluations, event sink, turn envelope)
- Endpoint `/metrics` funzionante e scrappato ogni 15s
- Dati visibili (`lexe_pipeline_routed_total`, `lexe_memory_retrieval_total`, etc.)

### 3.2 lexe-tools-it

**Langfuse** (NONE)
- Nessun import, nessuna configurazione, nessuna ENV var
- Le chiamate LLM in capabilities passano tutte via LiteLLM HTTP, quindi sono tracciate indirettamente tramite il callback LiteLLM -> Langfuse, ma senza contesto di pipeline (nessun trace_id, nessun parent span)

**OpenTelemetry** (CODE - non deployato)
- `telemetry.py`: Setup identico a lexe-core (FastAPI + HTTPX instrumentor, decoratore `@traced`)
- Config in codice: `otel_enabled=True`, `otel_service_name=lexe-tools-it`
- Docker compose ha le ENV vars (`OTEL_EXPORTER_OTLP_ENDPOINT=http://shared-jaeger:4317`)
- Ma nel container runtime: **nessuna ENV OTEL presente** — probabilmente non passate nell'override stage

**Prometheus** (CODE - non scrappato)
- `metrics.py`: 4 metriche (tool latency, invocations, cache ops, errors)
- Endpoint `/metrics` risponde con intestazioni metriche (verificato con curl)
- **Non presente** nella configurazione Prometheus (`prometheus.yml` ha solo `lexe-core`)

### 3.3 lexe-memory

**Langfuse** (NONE)
- Nessuna integrazione Langfuse

**OpenTelemetry** (CODE - disabilitato per default)
- `telemetry/tracing.py`: `create_span()` context manager con NoOpSpan fallback
- `config.py`: `otel_enabled` default `false` (esplicito nel codice)
- ENV in staging: `OTEL_ENABLED` non impostata (default false), `OTEL_EXPORTER_OTLP_ENDPOINT=http://shared-jaeger:4317` presente
- `main.py`: Bootstrap OTEL condizionale su `settings.otel_enabled`

**Prometheus** (CODE - non raggiungibile)
- `telemetry/metrics.py`: Counter + Histogram per memory operations
- Endpoint `/metrics` protetto da HMAC (ritorna 401 `Missing HMAC headers`)
- Non presente nella configurazione Prometheus

---

## 4. Audit Capabilities lexe-tools-it

### 4.1 Matrice Tracing per Capability

| Capability | Langfuse Spans | OTEL Spans | Prometheus Metrics | Structured Logging |
|-----------|---------------|------------|-------------------|--------------------|
| `intent_classifier.py` | NONE | NONE | NONE | OK (`logger.info` con timing) |
| `research_planner.py` | NONE | NONE | NONE | OK (`logger.info` con timing) |
| `research_executor.py` | NONE | NONE | NONE | OK (`logger.info` con timing) |
| `legal_verifier.py` | NONE | NONE | NONE | OK (`logger.info` con timing) |
| `legal_synthesizer.py` | NONE | NONE | NONE | OK (`logger.info` con timing) |
| `super_tool_agent.py` | NONE | NONE | NONE | OK (custom `SuperToolRunMetrics`) |
| `dispatcher.py` | NONE | NONE | NONE | OK (`logger.debug/info`) |

### 4.2 Dettaglio per Capability

**intent_classifier.py**
- Chiamata LLM: `httpx.AsyncClient` -> LiteLLM `/v1/chat/completions` (modello `lexe-fast`)
- Tracing: solo `logger.info("[INTENT_CLASSIFIER] Completed in %dms, model=%s")`
- GAP: nessun span Langfuse, nessun OTEL span, nessuna metrica Prometheus
- LLM call tracciata solo indirettamente via LiteLLM callback (senza context/parent)

**research_planner.py**
- Chiamata LLM: `httpx.AsyncClient` -> LiteLLM `/v1/chat/completions` (modello `lexe-primary`)
- Tracing: solo `logger.info("[RESEARCH_PLANNER] Completed in %dms, model=%s")`
- GAP: identico a intent_classifier

**research_executor.py**
- Tool calls: via `execute_tool_fn` (dispatcher o HTTP fallback)
- Tracing: `logger.info("[RESEARCH_EXECUTOR] Tool %s: %d items in %dms")`
- GAP: nessuna metrica per-tool-call, nessun span per l'intero batch P1/P2
- La `observe_tool_latency` di `metrics.py` non viene usata qui

**legal_verifier.py**
- Chiamata LLM opzionale: `httpx.AsyncClient` -> LiteLLM (coherence check)
- Tracing: `logger.info("[LEGAL_VERIFIER] Complete: status=%s confidence=%.2f...")`
- GAP: nessun Langfuse span per la verifica, nessun OTEL span

**legal_synthesizer.py**
- Streaming LLM: `httpx.AsyncClient` con `stream=True` -> LiteLLM
- Tracing: solo logging di base
- GAP: streaming non tracciato, nessun token count, nessun Langfuse generation span

**super_tool_agent.py**
- Loop agentico: multiple LLM calls + tool calls in loop
- Metriche interne: `SuperToolRunMetrics` (tokens, rounds, tool calls, duration)
- Persistence: `_save_run_metrics()` e' un no-op stub ("no DB in lexe-tools-it")
- GAP: metriche calcolate ma non inviate a Prometheus/Langfuse, solo logged

**dispatcher.py**
- Dispatch locale o HTTP fallback
- Tracing: solo `logger.debug/info`
- GAP: nessuna metrica su dispatch locale vs HTTP fallback

---

## 5. Gap Identificati e Raccomandazioni

### P0 - Critici (Impatto immediato sulla visibilita)

| # | Gap | Servizio | Raccomandazione |
|---|-----|----------|----------------|
| G1 | **Jaeger non riceve trace da nessun servizio** | Tutti | Verificare che i pacchetti `opentelemetry-exporter-otlp-proto-grpc` siano installati nei container. Controllare i log di startup per errori di import OTEL. |
| G2 | **lexe-tools-it non ha ENV OTEL nel container** | lexe-tools-it | Aggiungere `OTEL_EXPORTER_OTLP_ENDPOINT` e `OTEL_SERVICE_NAME` nel docker-compose.override.stage.yml. Le ENV sono nel base yml ma non passate. |
| G3 | **lexe-tools-it non scrappato da Prometheus** | lexe-tools-it | Aggiungere job `lexe-tools` in `prometheus.yml` con target `lexe-tools:8021` e `metrics_path: /metrics`. |
| G4 | **lexe-memory non scrappato da Prometheus** | lexe-memory | O esporre `/metrics` senza HMAC (su porta interna) oppure aggiungere un sidecar exporter. Aggiungere job in `prometheus.yml`. |

### P1 - Importanti (Visibilita sulle nuove capabilities)

| # | Gap | Servizio | Raccomandazione |
|---|-----|----------|----------------|
| G5 | **Capabilities non hanno Langfuse spans** | lexe-tools-it | Propagare `trace_id` da lexe-core -> lexe-tools-it nelle chiamate HTTP. Aggiungere Langfuse span per ogni fase (classify, plan, execute, verify, synthesize). |
| G6 | **research_executor non usa metrics.py** | lexe-tools-it | Wrappare ogni tool call con `observe_tool_latency()` gia presente in `metrics.py`. |
| G7 | **super_tool_agent metriche non persistite** | lexe-tools-it | Inviare `SuperToolRunMetrics` a Prometheus (Counter per rounds, Histogram per duration) e/o a Langfuse come span metadata. |
| G8 | **OTEL_ENABLED=false per lexe-memory** | lexe-memory | Impostare `OTEL_ENABLED=true` nel docker-compose e verificare che i pacchetti OTEL siano installati. |
| G9 | **LLM calls nelle capabilities non hanno parent trace** | lexe-tools-it | Passare `x-trace-id` header nelle chiamate LiteLLM per correlare con il trace Langfuse avviato da lexe-core. |

### P2 - Miglioramenti (Best practice)

| # | Gap | Servizio | Raccomandazione |
|---|-----|----------|----------------|
| G10 | **Nessuna metrica di latenza per capability** | lexe-tools-it | Aggiungere Histogram Prometheus per ogni fase capability (classify_seconds, plan_seconds, execute_seconds, verify_seconds, synthesize_seconds). |
| G11 | **Loki riceve solo log Docker raw** | Tutti | Configurare Fluent Bit con parser JSON per estrarre campi strutturati (correlation_id, trace_id, level) come label Loki. |
| G12 | **Langfuse non ha Langfuse per lexe-memory** | lexe-memory | Le chiamate embedding/retrieval non sono tracciate in Langfuse. Aggiungere integrazione per memory operations. |
| G13 | **Prometheus target per LiteLLM down** | lexe-litellm | Il target `litellm:4000/metrics` e' down. Verificare se LiteLLM espone metriche Prometheus e configurare il path corretto. |
| G14 | **Alerting incompleto** | Tutti | Aggiungere alert per lexe-tools-it down, lexe-memory down, Langfuse ingestion failures. |
| G15 | **Grafana dashboard manca lexe-tools-it** | Grafana | Aggiungere pannelli per tool execution latency, tool invocations, cache hit rate dal metrics endpoint lexe-tools-it. |

---

## 6. Riepilogo per Servizio

### lexe-core: 7/10
- Langfuse: OK (custom spans + LiteLLM callback + evaluation scores)
- Prometheus: OK (15+ metriche, scraping attivo)
- OTEL: configurato ma non funzionante (G1)
- Loki: OK (via Fluent Bit)

### lexe-tools-it: 2/10
- Langfuse: assente (G5, G9)
- Prometheus: codice presente ma non scrappato (G3, G6)
- OTEL: codice presente ma ENV non passate (G2)
- Loki: OK (via Fluent Bit)
- Capabilities: zero tracing strutturato (G5-G7, G10)

### lexe-memory: 2/10
- Langfuse: assente (G12)
- Prometheus: codice presente ma HMAC-locked e non scrappato (G4)
- OTEL: codice presente ma disabilitato per default (G8)
- Loki: OK (via Fluent Bit)

### lexe-litellm: 4/10
- Langfuse: OK (callback automatico per tutte le LLM calls)
- Prometheus: target configurato ma down (G13)
- OTEL: non implementato
- Loki: OK (via Fluent Bit)

---

## 7. Quick Wins (Impatto massimo con sforzo minimo)

1. **Aggiungere lexe-tools-it a Prometheus** (G3) — 5 min: una riga in `prometheus.yml`
2. **Aggiungere ENV OTEL a lexe-tools-it** (G2) — 5 min: 2 righe in docker-compose override
3. **Usare `observe_tool_latency` in research_executor** (G6) — 15 min: wrappare tool calls
4. **Impostare `OTEL_ENABLED=true` per lexe-memory** (G8) — 2 min: una ENV var
5. **Aggiungere `/metrics` non-HMAC per lexe-memory** (G4) — 10 min: route pubblica interna

---

*Report generato automaticamente. Nessuna modifica e' stata apportata ai sistemi.*
