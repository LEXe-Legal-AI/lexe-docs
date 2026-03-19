---
owner: platform-team
audience: internal
source_of_truth: true
status: active
sensitivity: internal
last_verified: 2026-03-19
---

# OBSERVABILITY -- Monitoring, Tracing & Alerting Reference

> Replaces: `observability-audit.md` (retained as historical baseline)
> Rule: where this doc and code diverge, **code wins**.

---

## Scope

**In scope:** Prometheus metrics and scrape targets, Grafana dashboards and datasources, Jaeger distributed tracing, Loki + Fluent Bit log aggregation, Langfuse LLM observability, EventSink system, trace hash lookup, assessment dashboard, alert rules and groups.

**Not in scope:** Deploy procedures (see `RUNBOOK-V2.md`), pipeline internals (see `PIPELINE-REFERENCE.md`), security hardening (see `SECURITY-POSTURE.md`), infrastructure provisioning.

---

## 1. Prometheus

### Overview

- **Container:** `shared-prometheus`
- **Port:** 9090
- **Network:** `shared_public` (to reach lexe-core on `lexe_internal`)
- **Scrape interval:** 15 seconds
- **Config:** `/etc/prometheus/prometheus.yml` (bind-mounted from lexe-infra)

### Scrape Targets (14)

| # | Job Name | Target | Port | Status | Metrics Path |
|---|----------|--------|------|--------|-------------|
| 1 | lexe-core | `lexe-core:8100` | 8100 | ACTIVE | `/metrics` |
| 2 | lexe-tools | `lexe-tools:8021` | 8021 | ACTIVE | `/metrics` |
| 3 | lexe-memory | `lexe-memory:8103` | 8103 | CODE (HMAC-locked) | `/metrics` |
| 4 | lexe-litellm | `lexe-litellm:4001` | 4001 | CONFIGURED (intermittent) | `/metrics` |
| 5 | lexe-orchestrator | `lexe-orchestrator:8102` | 8102 | CONFIGURED (FF disabled) | `/metrics` |
| 6 | node-exporter | `shared-node-exporter:9100` | 9100 | ACTIVE | `/metrics` |
| 7 | cadvisor | `shared-cadvisor:8080` | 8080 | ACTIVE | `/metrics` |
| 8 | traefik | `shared-traefik:8082` | 8082 | ACTIVE | `/metrics` |
| 9 | prometheus | `localhost:9090` | 9090 | ACTIVE (self) | `/metrics` |
| 10 | alertmanager | `shared-alertmanager:9093` | 9093 | ACTIVE | `/metrics` |
| 11 | grafana | `shared-grafana:3001` | 3001 | ACTIVE | `/metrics` |
| 12 | loki | `shared-loki:3100` | 3100 | ACTIVE | `/metrics` |
| 13 | temporal | `lexe-temporal:7234` | 7234 | CONFIGURED | `/metrics` |
| 14 | postgres-exporter | `shared-postgres-exporter:9187` | 9187 | ACTIVE | `/metrics` |

### lexe-core Metrics (15+ custom)

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `lexe_request_duration_seconds` | Histogram | method, path, status | HTTP request latency |
| `lexe_requests_total` | Counter | method, path, status | HTTP request count |
| `lexe_errors_total` | Counter | type, path | Error count by type |
| `lexe_sse_events_total` | Counter | event_type | SSE events emitted |
| `lexe_sse_duration_seconds` | Histogram | -- | SSE stream duration |
| `lexe_pipeline_routed_total` | Counter | strategy, depth | Pipeline routing decisions |
| `lexe_pipeline_duration_seconds` | Histogram | strategy, depth | Pipeline execution time |
| `lexe_confidence_score` | Histogram | band | Confidence score distribution |
| `lexe_evidence_items_total` | Counter | type | Evidence items by type |
| `lexe_memory_retrieval_total` | Counter | level, status | Memory retrieval ops |
| `lexe_memory_retrieval_seconds` | Histogram | level | Memory retrieval latency |
| `lexe_evaluations_total` | Counter | rating | User evaluation submissions |
| `lexe_event_sink_total` | Counter | event_type, status | EventSink persistence ops |
| `lexe_turn_envelope_seconds` | Histogram | -- | Turn envelope processing time |
| `lexe_limits_block_total` | Counter | type | Rate limit blocks (403/429) |
| `lexe_tenant_provision_total` | Counter | status | Tenant provisioning outcomes |

### lexe-tools Metrics (4 custom)

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `lexe_tool_latency_seconds` | Histogram | tool_name | Per-tool execution latency |
| `lexe_tool_invocations_total` | Counter | tool_name, status | Tool invocation count |
| `lexe_tool_cache_ops_total` | Counter | op, status | Cache hit/miss/set |
| `lexe_tool_errors_total` | Counter | tool_name, error_type | Tool errors by type |

### Known Gaps

- **lexe-memory**: `/metrics` endpoint exists but is HMAC-protected. Prometheus cannot scrape without auth header. Workaround: expose a separate unauthenticated `/metrics` route on internal network only.
- **lexe-litellm**: Target intermittently down. Verify LiteLLM exposes Prometheus-compatible metrics.

---

## 2. Grafana

### Overview

- **Container:** `shared-grafana`
- **Port:** 3001
- **Auth (staging):** admin / `LeoGrafanaStage2026!`
- **Auth (prod):** admin / (see LEXE-CREDENTIALS.md)

### Dashboards (6)

| # | Dashboard | UID | Panels | Description |
|---|-----------|-----|--------|-------------|
| 1 | LEXE Operations | `lexe-operations` | 8 | Core platform metrics: request rate, error rate, SSE duration, pipeline routing, confidence distribution, memory retrieval, evaluations, event sink |
| 2 | LEO Overview | `leo-overview` | 12 | Shared infrastructure overview (Traefik, node resources) |
| 3 | LEO Observability | `leo-observability` | 10 | Cross-platform observability (Loki queries, alert status) |
| 4 | Database | `database` | 8 | PostgreSQL metrics (connections, queries, replication, disk) |
| 5 | Memory Latency | `memory-latency` | 6 | lexe-memory operation latency (L0-L4 retrieval, writeback) |
| 6 | LLM Observability | `llm-observability` | 8 | LLM call latency, token usage, model distribution, cost |

### Datasources

| Datasource | Type | UID | URL | Description |
|------------|------|-----|-----|-------------|
| Prometheus | prometheus | `prometheus` | `http://shared-prometheus:9090` | Primary metrics |
| Loki | loki | `loki` | `http://shared-loki:3100` | Log aggregation |
| ClickHouse | clickhouse | `clickhouse` | `http://shared-langfuse-clickhouse:8123` | Langfuse analytics backend |

---

## 3. Jaeger

### Overview

- **Container:** `shared-jaeger`
- **Port:** 16686 (UI), 4317 (OTLP gRPC), 4318 (OTLP HTTP)
- **Storage:** Badger (embedded, local disk)
- **Retention:** 72 hours (default)

### OTLP Configuration

Services are configured to send traces via OpenTelemetry Protocol:

| Service | ENV Vars | Code Status |
|---------|----------|-------------|
| lexe-core | `OTEL_EXPORTER_OTLP_ENDPOINT=http://shared-jaeger:4317`, `OTEL_SERVICE_NAME=lexe-core` | `telemetry.py`: FastAPI + HTTPX instrumentor, `@traced()` decorator. **Currently inactive** (OTEL packages may not be installed in container) |
| lexe-tools | `OTEL_EXPORTER_OTLP_ENDPOINT=http://shared-jaeger:4317`, `OTEL_SERVICE_NAME=lexe-tools-it` | `telemetry.py`: same setup as lexe-core. **ENV vars not passed** in stage override |
| lexe-memory | `OTEL_EXPORTER_OTLP_ENDPOINT=http://shared-jaeger:4317` | `telemetry/tracing.py`: `create_span()` with NoOpSpan fallback. **Default `otel_enabled=false`** |

### Sampling Strategies

Default: `const` sampler with param=1 (sample everything). In production, consider switching to `probabilistic` sampler with param=0.1 for high-throughput endpoints.

### Known Gap

Jaeger reports 0 services. Root cause: OTEL exporter packages likely not installed in container images, or import fails silently. Requires verification of `opentelemetry-exporter-otlp-proto-grpc` in each service's requirements.txt.

---

## 4. Loki + Fluent Bit

### Loki

- **Container:** `shared-loki`
- **Port:** 3100
- **Storage:** Local filesystem (`/loki/chunks`, `/loki/index`)
- **Retention:** 7 days (default)
- **Labels:** `docker` source label (container-level)

### Fluent Bit

- **Container:** `shared-fluent-bit`
- **Port:** 24224 (forward), 2020 (HTTP monitoring)
- **Input:** Docker log driver (JSON)
- **Output:** Loki (`http://shared-loki:3100/loki/api/v1/push`)

### Log Coverage

All 13 LEXE containers forward logs to Loki via Fluent Bit (Docker log driver). Logs are JSON-structured from Python services (lexe-core, tools, memory) and nginx access logs from frontends (webchat, admin).

### Structured Logging Fields

lexe-core and lexe-tools emit structured JSON logs with:

| Field | Description | Example |
|-------|-------------|---------|
| `level` | Log level | `INFO`, `ERROR`, `WARNING` |
| `message` | Log message | `"[PIPELINE] Routed to legal/STANDARD"` |
| `correlation_id` | Request correlation | `"abc123-def456"` |
| `tenant_id` | Tenant UUID | `"67a08dc0-..."` |
| `trace_hash` | 8-char trace hash | `"a1b2c3d4"` |
| `timestamp` | ISO 8601 | `"2026-03-19T10:30:00Z"` |

### Known Gap

Fluent Bit forwards raw Docker JSON logs. Structured fields are embedded in the `log` field as a JSON string. Loki queries require `| json` pipeline stage to extract fields. A Fluent Bit parser configuration for JSON extraction would improve query ergonomics.

---

## 5. Langfuse

### Overview

- **Container:** `shared-langfuse`
- **Port:** 3005
- **Backend:** ClickHouse (`shared-langfuse-clickhouse`)
- **Status:** Health "starting" (normal -- Langfuse reports this during warmup)

### Integration Points

| Service | Integration | Details |
|---------|-------------|---------|
| lexe-core | **Direct API** | `langfuse_integration.py`: `LangfuseSpanManager` singleton, batch ingestion. Trace per conversation turn, custom spans per pipeline phase. Scores for confidence + user evaluation. Controlled by `ff_langfuse_spans`. |
| lexe-litellm | **Callback** | `LITELLM_CALLBACKS=langfuse`. All LLM calls automatically traced with model, tokens, latency. No pipeline context (no parent trace_id). |
| lexe-tools | **None** | LLM calls traced indirectly via LiteLLM callback, but without parent trace or pipeline context. |
| lexe-memory | **None** | Embedding calls not traced in Langfuse. |

### Token Tracking

Langfuse tracks token usage per LLM call via LiteLLM callback:
- `prompt_tokens`, `completion_tokens`, `total_tokens`
- Model name and provider
- Latency (TTFT and total)
- Cost estimation (based on model pricing)

### Cost Analysis

Langfuse dashboard provides:
- Cost per model over time
- Cost per tenant (via trace metadata)
- Cost per pipeline strategy (via span tags)
- Token efficiency (tokens per quality score)

### ClickHouse Analytics

The ClickHouse backend (`shared-langfuse-clickhouse`) enables:
- Fast aggregation over large trace volumes
- Custom SQL queries on trace/span/score data
- Accessible via Grafana ClickHouse datasource

---

## 6. EventSink

### Overview

The EventSink system persists conversation events for analytics, audit, and quality assessment.

- **Feature flag:** `ff_event_persistence` (default: `true`)
- **Table:** `core.conversation_events`
- **Pattern:** Fire-and-forget (async, non-blocking)

### Event Points (21 `_persist_event()` calls)

All wired in `customer_router.py:stream_to_orchestrator()`:

| # | Event Type | Trigger | Data |
|---|-----------|---------|------|
| 1 | `turn_start` | Conversation turn begins | tenant_id, conversation_id, user_message |
| 2 | `intent_detected` | Intent classifier returns | intent, confidence, model |
| 3 | `pipeline_routed` | Strategy selected | strategy, depth_level |
| 4 | `tool_call_start` | Tool execution begins | tool_name, args |
| 5 | `tool_call_end` | Tool execution completes | tool_name, result_size, duration_ms |
| 6 | `tool_call_error` | Tool execution fails | tool_name, error |
| 7 | `llm_call_start` | LLM call begins | model, role |
| 8 | `llm_call_end` | LLM call completes | model, tokens, duration_ms |
| 9 | `sse_token` | SSE token emitted | token_count (batched) |
| 10 | `sse_tool_start` | SSE tool_start event | tool_name |
| 11 | `sse_tool_result` | SSE tool_result event | tool_name, items |
| 12 | `sse_evidence` | SSE evidence event | evidence_count |
| 13 | `sse_confidence` | SSE confidence event | score, band |
| 14 | `sse_follow_up` | SSE follow_up event | suggestions |
| 15 | `sse_done` | SSE done event | trace_hash, total_tokens |
| 16 | `sse_error` | SSE error event | error_code, message |
| 17 | `memory_retrieve` | Memory context retrieved | level, items, duration_ms |
| 18 | `memory_store` | Memory stored | level, operation |
| 19 | `verification_result` | Verifier completes | status, confidence |
| 20 | `evaluation_received` | User submits rating | rating (1-5) |
| 21 | `turn_end` | Conversation turn completes | total_duration_ms, tokens |

### Schema

```sql
CREATE TABLE core.conversation_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES core.tenants(id),
    conversation_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    event_data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 7. Trace System

### Overview

Every conversation turn generates an 8-character trace hash for end-to-end correlation.

- **Generation:** SHA-256 of `{conversation_id}:{turn_index}:{timestamp}`, truncated to 8 chars
- **Propagation:** SSE `done` event includes `trace_hash` field
- **Frontend:** Stored in `ChatMessage.traceHash`, displayed as trace badge
- **Lookup:** `GET /admin/trace/{hash}`

### Trace Lookup Endpoint

```
GET /api/v1/admin/trace/{hash}
Authorization: Bearer <admin-token>
```

**Response:**

```json
{
    "trace_hash": "a1b2c3d4",
    "conversation_id": "uuid",
    "tenant_id": "uuid",
    "timeline": [
        {
            "event_type": "turn_start",
            "timestamp": "2026-03-19T10:30:00Z",
            "data": {}
        },
        {
            "event_type": "pipeline_routed",
            "timestamp": "2026-03-19T10:30:01Z",
            "data": {"strategy": "legal", "depth": "STANDARD"}
        }
    ],
    "metadata": {
        "total_duration_ms": 4200,
        "total_tokens": 1850,
        "strategy": "legal",
        "depth": "STANDARD",
        "confidence": 52,
        "band": "VERIFIED",
        "tools_called": ["kb_search", "normattiva_search"],
        "evaluation": null
    }
}
```

### Correlation Flow

```
User message
  -> lexe-core generates trace_hash
  -> All _persist_event() calls include trace_hash in event_data
  -> Langfuse trace created with trace_hash as external_id
  -> SSE done event emits trace_hash to frontend
  -> Frontend stores in ChatMessage.traceHash
  -> Admin can lookup via /admin/trace/{hash}
  -> Timeline reconstructed from conversation_events ordered by created_at
```

---

## 8. Assessment Dashboard

### Overview

The assessment endpoint provides aggregated platform quality metrics.

- **Endpoint:** `GET /api/v1/admin/assessment?period=1d|7d|30d`
- **Auth:** Admin token required
- **Cache:** Hybrid in-memory + DB, cached 60 seconds, survives restarts

### Response Sections

| Section | Contents |
|---------|----------|
| `pipeline_distribution` | Breakdown by strategy (toolloop/legal) and depth (MINIMAL/STANDARD/DEEP), with counts and percentages |
| `evaluation_stats` | Average rating, rating distribution (1-5), total evaluations, response rate |
| `confidence_stats` | Average confidence, band distribution (VERIFIED/CAUTION/LOW_CONFIDENCE), percentiles |
| `tool_stats` | Most used tools, average tool calls per turn, tool error rate |
| `latency_stats` | p50/p90/p99 latency by strategy, SSE TTFT, total duration |
| `token_stats` | Average tokens per turn, token efficiency (tokens per confidence point) |
| `anomalies` | Detected anomalies: error rate spikes, latency outliers, confidence drops |
| `sla_compliance` | SLA target vs actual for availability, latency, error rate |

### SLA Persistence

SLA metrics use a hybrid approach:
- **In-memory:** Rolling window counters (availability, error rate, latency percentiles)
- **DB fallback:** Periodic snapshots to `core.conversation_events` aggregates
- **Cache TTL:** 60 seconds
- **Restart recovery:** Recomputes from DB on startup (last 24h of events)

---

## 9. Alert Groups

### Overview

- **Alertmanager:** `shared-alertmanager` on port 9093
- **Alert rules:** Configured in Prometheus via `/etc/prometheus/alert_rules.yml`
- **Notification:** (configured per environment -- Slack webhook, email, or PagerDuty)

### Alert Groups (7 groups, 72 rules total)

#### Group 1: Service Availability (12 rules)

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `LexeCoreDown` | critical | `up{job="lexe-core"} == 0` | 1m |
| `LexeToolsDown` | critical | `up{job="lexe-tools"} == 0` | 1m |
| `LexeMemoryDown` | warning | `up{job="lexe-memory"} == 0` | 2m |
| `LexeWebchatDown` | critical | Traefik backend health | 1m |
| `LexeAdminDown` | warning | Traefik backend health | 2m |
| `LexeLitellmDown` | critical | `up{job="lexe-litellm"} == 0` | 1m |
| `LexeLogtoDown` | critical | Logto health check fails | 1m |
| `LexePostgresDown` | critical | `pg_up == 0` | 30s |
| `LexeMaxDown` | critical | `pg_up == 0` (KB DB) | 30s |
| `LexeValkeyDown` | critical | Valkey ping fails | 30s |
| `LexeTemporalDown` | warning | `up{job="temporal"} == 0` | 2m |
| `LexeOrchestratorDown` | info | `up{job="lexe-orchestrator"} == 0` | 5m |

#### Group 2: Error Rates (10 rules)

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `LexeHighErrorRate` | critical | `rate(lexe_errors_total[5m]) > 0.1` | 2m |
| `LexeHigh5xxRate` | critical | 5xx rate > 5% of requests | 2m |
| `LexeSSEErrors` | warning | `rate(lexe_sse_events_total{event_type="error"}[5m]) > 0.05` | 3m |
| `LexeToolErrors` | warning | Tool error rate > 10% | 3m |
| `LexeEventSinkErrors` | warning | EventSink failure rate > 5% | 5m |
| `LexeAuthErrors` | critical | 401 rate spike (> 20% of requests) | 2m |
| `LexeMemoryErrors` | warning | Memory retrieval error rate > 10% | 3m |
| `LexeLimitsBlockRate` | info | Limits block rate > 10/min | 5m |
| `LexeProvisionFailures` | warning | Provision failure rate > 0 | 1m |
| `LexePipelineErrors` | warning | Pipeline error rate > 5% | 3m |

#### Group 3: LLM Performance (12 rules)

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `LexeHighLatency` | warning | `lexe_request_duration_seconds{quantile="0.95"} > 10` | 3m |
| `LexeSSELatency` | warning | SSE stream p95 > 30s | 3m |
| `LexePipelineLatency` | warning | Pipeline p95 > 20s for STANDARD depth | 3m |
| `LexeLLMLatencyHigh` | warning | LLM call p95 > 15s | 3m |
| `LexeTokenBudgetExceeded` | info | Token budget exceeded per turn | 5m |
| `LexeConfidenceDrop` | warning | Avg confidence < 30 over 30m window | 10m |
| `LexeLowVerifiedRate` | warning | VERIFIED band < 40% of turns | 15m |
| `LexeHighLowConfRate` | critical | LOW_CONFIDENCE band > 30% of turns | 10m |
| `LexeTTFTHigh` | warning | Time-to-first-token p95 > 5s | 3m |
| `LexeToolCallBudget` | info | Tool calls exceed depth budget | 5m |
| `LexeDepthEscalation` | info | Depth escalation rate > 20% | 15m |
| `LexeSelfCorrectionRate` | warning | Self-correction triggers > 10% | 10m |

#### Group 4: Resource Utilization (12 rules)

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `HighCPUUsage` | warning | Container CPU > 80% | 5m |
| `HighMemoryUsage` | warning | Container memory > 85% | 5m |
| `DiskSpaceLow` | critical | Disk < 10% free | 2m |
| `DiskSpaceWarning` | warning | Disk < 20% free | 5m |
| `HighNetworkTraffic` | info | Network > 100MB/s | 10m |
| `ContainerRestarting` | warning | Container restart count > 3 in 15m | 1m |
| `OOMKilled` | critical | Container OOM killed | 0s |
| `HighLoadAverage` | warning | Load average > CPU count * 2 | 5m |
| `SwapUsageHigh` | warning | Swap > 50% | 5m |
| `InodeUsageHigh` | warning | Inode usage > 80% | 5m |
| `TraefikHighLatency` | warning | Traefik backend p95 > 5s | 3m |
| `TraefikHighErrorRate` | warning | Traefik 5xx > 2% | 3m |

#### Group 5: Database & Cache (12 rules)

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `PostgresConnectionsHigh` | warning | Active connections > 80% of max | 3m |
| `PostgresConnectionPoolExhausted` | critical | Active connections > 95% of max | 1m |
| `PostgresLongRunningQueries` | warning | Queries running > 60s | 1m |
| `PostgresDeadlocks` | warning | Deadlock rate > 0 | 1m |
| `PostgresReplicationLag` | critical | Replication lag > 30s | 2m |
| `PostgresDiskUsageHigh` | warning | DB size > 80% of disk | 5m |
| `ValkeyMemoryHigh` | warning | Valkey memory > 80% of maxmemory | 3m |
| `ValkeyEvictionsHigh` | warning | Eviction rate > 100/s | 5m |
| `ValkeyConnectionsHigh` | warning | Connected clients > 500 | 3m |
| `ValkeySlowLog` | info | Slow commands > 10 in 5m | 5m |
| `KBDatabaseSlow` | warning | lexe-max query p95 > 2s | 3m |
| `KBDatabaseConnections` | warning | lexe-max connections > 50 | 3m |

#### Group 6: Temporal (8 rules)

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `TemporalWorkflowFailures` | warning | Workflow failure rate > 5% | 5m |
| `TemporalWorkflowTimeout` | warning | Workflow timeout rate > 2% | 5m |
| `TemporalActivityFailures` | warning | Activity failure rate > 5% | 5m |
| `TemporalScheduleLag` | warning | Schedule lag > 30s | 3m |
| `TemporalPendingActivities` | warning | Pending activities > 100 | 5m |
| `TemporalPendingWorkflows` | info | Pending workflows > 50 | 5m |
| `TemporalPersistenceLag` | critical | Persistence lag > 10s | 2m |
| `TemporalFrontendLatency` | warning | Frontend p95 > 5s | 3m |

#### Group 7: Self-Monitoring (6 rules)

| Alert | Severity | Condition | For |
|-------|----------|-----------|-----|
| `PrometheusTargetDown` | warning | Any scrape target down | 3m |
| `PrometheusScrapeErrors` | warning | Scrape error rate > 10% | 5m |
| `GrafanaDown` | warning | Grafana health check fails | 2m |
| `LokiIngestionErrors` | warning | Loki push failures > 0 | 3m |
| `AlertmanagerDown` | critical | Alertmanager unreachable | 1m |
| `LangfuseIngestionFailure` | warning | Langfuse batch ingestion errors | 5m |

---

## 10. Observability Coverage Matrix

### Per-Service Coverage

| Service | Prometheus | Langfuse | OTEL/Jaeger | Loki | Score |
|---------|-----------|----------|-------------|------|-------|
| lexe-core | ACTIVE (15+ metrics) | ACTIVE (spans + scores) | CODE (inactive) | ACTIVE | 7/10 |
| lexe-tools | ACTIVE (4 metrics) | NONE (indirect via LiteLLM) | CODE (ENV missing) | ACTIVE | 3/10 |
| lexe-memory | CODE (HMAC-locked) | NONE | CODE (disabled) | ACTIVE | 2/10 |
| lexe-litellm | CONFIGURED (intermittent) | ACTIVE (callback) | NONE | ACTIVE | 4/10 |
| lexe-orchestrator | CONFIGURED (FF off) | NONE | NONE | ACTIVE | 1/10 |
| lexe-webchat | N/A (frontend) | N/A | N/A | ACTIVE (nginx) | N/A |
| lexe-admin | N/A (frontend) | N/A | N/A | ACTIVE (nginx) | N/A |
| lexe-postgres | ACTIVE (exporter) | N/A | N/A | ACTIVE | 8/10 |
| lexe-max | ACTIVE (exporter) | N/A | N/A | ACTIVE | 8/10 |
| lexe-valkey | ACTIVE (via node-exp) | N/A | N/A | ACTIVE | 6/10 |
| lexe-logto | N/A (third-party) | N/A | N/A | ACTIVE | N/A |
| lexe-temporal | CONFIGURED | N/A | N/A | ACTIVE | 5/10 |
| lexe-temporal-ui | N/A (UI) | N/A | N/A | ACTIVE | N/A |

### Priority Gaps (from observability-audit.md)

| Priority | Gap | Service | Status |
|----------|-----|---------|--------|
| P0 | G1: Jaeger receives 0 traces | All | OPEN -- verify OTEL packages in containers |
| P0 | G3: lexe-tools not scraped by Prometheus | lexe-tools | FIXED (added to prometheus.yml) |
| P0 | G4: lexe-memory HMAC-locked metrics | lexe-memory | OPEN -- expose unauthenticated internal route |
| P1 | G5: Tools capabilities lack Langfuse spans | lexe-tools | OPEN |
| P1 | G8: lexe-memory OTEL disabled by default | lexe-memory | OPEN |
| P1 | G9: LLM calls in tools lack parent trace | lexe-tools | OPEN |
| P2 | G11: Loki receives raw Docker JSON | All | OPEN -- configure Fluent Bit JSON parser |

---

*Last updated: 2026-03-19*
