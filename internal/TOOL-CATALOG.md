---
owner: platform-team
audience: internal
source_of_truth: true
status: active
sensitivity: internal
last_verified: 2026-03-19
---

# LEXE Tool Catalog

Internal reference for all legal research tools available to the LEXE AI agent pipeline.

## Scope

**In scope:** Tool definitions, parameters, execution semantics, infrastructure (circuit breaker, budget, health monitoring, effectiveness tracking).

**Not in scope:** LLM model configuration (see `lexe-docs/llm-selection.md`), pipeline orchestration logic (see `orchestration_v2_progress.md`), frontend tool rendering.

---

## 1. normattiva_search

| Property | Value |
|----------|-------|
| **Description** | Search Italian legislation via Normattiva.it OpenData API. Retrieves full article text, checks vigenza (validity). Use for special laws, legislative decrees, DPR, and any act not in the 5 main codes. |
| **Purity Level** | L3 (External API -- deterministic lookup) |
| **Semaphore Category** | `official` (6 concurrent) |
| **Timeout** | 30s |
| **Retry** | 2 attempts, exponential backoff (1-10s jitter) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py` (definition), `lexe-core/src/lexe_core/tools/executor.py` (execution) |
| **Backend Endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/normattiva/search` |
| **Side Effects** | None. Read-only HTTP call to lexe-tools, which proxies Normattiva OpenData. |

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `act_type` | `string` | Yes | Type of act: legge, decreto legislativo, d.lgs, decreto legge, d.l., d.p.r., codice civile, codice penale, costituzione, etc. |
| `date` | `string` | No | Date of the act in YYYY-MM-DD or YYYY format. |
| `act_number` | `string` | No | Number of the act (e.g., "241" for Legge 241/1990). |
| `article` | `string` | No | Specific article number to retrieve (e.g., "1", "21-bis"). |

### Return Format

```json
{
  "text": "Full article text...",
  "urn": "urn:nir:stato:legge:1990-08-07;241",
  "title": "Legge 7 agosto 1990, n. 241",
  "article": "1",
  "vigente": true,
  "url": "https://www.normattiva.it/uri-res/..."
}
```

---

## 2. eurlex_search

| Property | Value |
|----------|-------|
| **Description** | Search EU legislation on EUR-Lex. Finds Regulations, Directives, Decisions, Treaties. Useful for GDPR, DSA, DMA, and other EU legal acts. |
| **Purity Level** | L3 (External API -- SPARQL + REST) |
| **Semaphore Category** | `official` (6 concurrent) |
| **Timeout** | 30s |
| **Retry** | 2 attempts, exponential backoff (1-10s jitter) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py`, `lexe-core/src/lexe_core/tools/executor.py` |
| **Backend Endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/eurlex/search` |
| **Side Effects** | None. |

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `act_type` | `string` (enum) | Yes | One of: `regolamento`, `direttiva`, `decisione`, `trattato`, `raccomandazione`. |
| `year` | `integer` | Yes | Year of publication (e.g., 2016 for GDPR). |
| `number` | `integer` | Yes | Number of the act (e.g., 679 for GDPR). |
| `article` | `string` | No | Specific article to retrieve (e.g., "17"). |

### Return Format

```json
{
  "text": "Article text...",
  "title": "Regolamento (UE) 2016/679",
  "celex": "32016R0679",
  "url": "https://eur-lex.europa.eu/legal-content/IT/TXT/HTML/?uri=CELEX:32016R0679"
}
```

---

## 3. infolex_search

| Property | Value |
|----------|-------|
| **Description** | Search Italian case law (giurisprudenza) and legal commentary from Brocardi. Returns article text, explanations, and court decision summaries (massime) from Cassazione, Corte Costituzionale, and other courts. |
| **Purity Level** | L4 (External scraping -- non-deterministic content) |
| **Semaphore Category** | `certified` (4 concurrent) |
| **Timeout** | 30s |
| **Retry** | 2 attempts, exponential backoff (1-10s jitter) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py`, `lexe-core/src/lexe_core/tools/executor.py` |
| **Backend Endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/infolex/search` |
| **Side Effects** | None. |

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `act_type` | `string` | Yes | Code type: codice civile, codice penale, codice procedura civile, codice procedura penale, etc. |
| `article` | `string` | Yes | Article number (e.g., "1321" for Nozione di contratto). |
| `include_massime` | `boolean` | No | Include case law summaries. Default: `true`. |

### Return Format

```json
{
  "article_text": "Testo dell'articolo...",
  "spiegazione": "Commentary...",
  "massime": [
    {
      "autorita": "Cass. civ., Sez. III, 12/03/2024",
      "testo": "Summary of the decision..."
    }
  ],
  "act_type": "codice civile",
  "article": "2043"
}
```

---

## 4. kb_search

| Property | Value |
|----------|-------|
| **Description** | Search the Italian Supreme Court case law database (Massimario di Cassazione). Contains 38,000+ case summaries (massime) with hybrid dense + sparse + RRF ranking. |
| **Purity Level** | L2 (Internal database -- deterministic, owned data) |
| **Semaphore Category** | `certified` (4 concurrent) |
| **Timeout** | 20s (capped via `min(timeout, 20.0)`) |
| **Retry** | 2 attempts, exponential backoff (1-10s jitter) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py`, `lexe-core/src/lexe_core/tools/executor.py` |
| **Backend Endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/kb/search` |
| **Side Effects** | None. Read-only against lexe-max PostgreSQL (port 5436). |

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `query` | `string` | Yes | Natural language query describing the legal question. |
| `top_k` | `integer` | No | Results to return (1-20). Default: 5. |
| `materia` | `string` | No | Subject matter filter (e.g., "responsabilita civile", "contratti", "lavoro"). |
| `anno_min` | `integer` | No | Minimum year filter for massime. |
| `anno_max` | `integer` | No | Maximum year filter for massime. |
| `sezione` | `string` | No | Section filter (e.g., "unite" for Sezioni Unite, "3" for Sezione III). |

### Return Format

```json
{
  "success": true,
  "results": [
    {
      "riferimento": "Cass. civ., Sez. III, 15/02/2024, n. 12345",
      "testo": "Massima text...",
      "materia": "responsabilita civile",
      "rrf_score": 0.87
    }
  ],
  "meta": { "latency_ms": 145, "source": "kb_search" }
}
```

---

## 5. kb_normativa_search

| Property | Value |
|----------|-------|
| **Description** | Search the 5 main Italian legal codes (CC, CP, CPC, CPP, Costituzione) from local database. Fast (~150ms). Contains ~5,800 articles with hybrid semantic + keyword search. |
| **Purity Level** | L2 (Internal database -- deterministic, owned data) |
| **Semaphore Category** | `certified` (4 concurrent) |
| **Timeout** | 20s (capped via `min(timeout, 20.0)`) |
| **Retry** | 2 attempts, exponential backoff (1-10s jitter) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py`, `lexe-core/src/lexe_core/tools/executor.py` |
| **Backend Endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/kb/normativa/search` |
| **Side Effects** | None. Read-only against lexe-max PostgreSQL (port 5436). |

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `query` | `string` | Yes | Natural language query in Italian describing the legal concept. |
| `top_k` | `integer` | No | Results to return (1-20). Default: 5. |
| `codes` | `array[string]` | No | Filter by code: CC, CP, CPC, CPP, COST. Omit for all codes. |

### Return Format

```json
{
  "success": true,
  "results": [
    {
      "code": "CC",
      "article": "2043",
      "rubrica": "Risarcimento per fatto illecito",
      "text": "Qualunque fatto doloso o colposo...",
      "score": 0.92
    }
  ],
  "meta": { "latency_ms": 148, "source": "kb_normativa_search" }
}
```

---

## 6. lex_search

| Property | Value |
|----------|-------|
| **Description** | Search Italian and European legal websites via SearXNG metasearch engine. Restricts queries to 14 authoritative legal portals. |
| **Purity Level** | L4 (External web scraping -- non-deterministic) |
| **Semaphore Category** | `certified` (4 concurrent) |
| **Timeout** | 15s (capped via `min(timeout, 15.0)`) |
| **Retry** | 2 attempts, exponential backoff (1-10s jitter) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py`, `lexe-core/src/lexe_core/tools/executor.py`, `lexe-tools-it/src/lexe_tools_it/tools/lex_search.py` |
| **Backend Endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/lex_search` |
| **Side Effects** | None. |

### Legal Sites (14 total)

**Italy (10):** normattiva.it, brocardi.it, altalex.com, studiocataldi.it, dejure.it, cortedicassazione.it, giurcost.org, giustizia-amministrativa.it, ilcaso.it, iusexplorer.it

**EU (4):** eur-lex.europa.eu, curia.europa.eu, hudoc.echr.coe.int, europarl.europa.eu

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `query` | `string` | Yes | Legal search query in natural language. |

### Return Format

```json
{
  "success": true,
  "results": [
    {
      "title": "Art. 2043 Codice Civile - Risarcimento...",
      "url": "https://brocardi.it/codice-civile/libro-quarto/...",
      "content": "Snippet text..."
    }
  ],
  "meta": { "latency_ms": 980, "source": "lex_search" }
}
```

---

## 7. lex_search_enriched

| Property | Value |
|----------|-------|
| **Description** | Advanced legal search with LLM analysis. Performs lex_search, then enriches results with automatic law reference extraction, vigenza verification via Normattiva, and structured analysis. |
| **Purity Level** | L4 (External web + LLM enrichment) |
| **Semaphore Category** | `certified` (4 concurrent) |
| **Timeout** | 30s (capped via `min(timeout, 30.0)`) |
| **Retry** | 2 attempts, exponential backoff (1-10s jitter) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py`, `lexe-core/src/lexe_core/tools/executor.py` |
| **Backend Endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/lex_search/enriched` |
| **Side Effects** | Triggers LLM inference call (lexe-fast) for result analysis. Consumes model budget. |

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `query` | `string` | Yes | Legal search query requiring in-depth analysis. |

### Return Format

```json
{
  "success": true,
  "analysis": "Structured LLM analysis text...",
  "results": [
    {
      "title": "...",
      "url": "...",
      "content": "...",
      "vigenza": "vigente"
    }
  ],
  "meta": { "latency_ms": 4200, "source": "lex_search_enriched" }
}
```

---

## 8. web_search

| Property | Value |
|----------|-------|
| **Description** | General web search via SearXNG. Use for recent news, current events, or information not found in legal databases. Not restricted to legal sites. |
| **Purity Level** | L5 (Open web -- lowest reliability) |
| **Semaphore Category** | `web` (3 concurrent) |
| **Timeout** | 30s |
| **Retry** | No retry (direct httpx call, not through `_post_with_retry`) |
| **Source File** | `lexe-core/src/lexe_core/tools/definitions.py`, `lexe-core/src/lexe_core/tools/executor.py` |
| **Backend Endpoint** | `GET {SEARXNG_URL}/search` (direct SearXNG, not via lexe-tools) |
| **Side Effects** | None. |
| **Default Enabled** | No (`include_web_search=False` in `get_tool_definitions()`) |

### Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `query` | `string` | Yes | Search query in natural language. |

### Return Format

```json
{
  "success": true,
  "query": "search terms",
  "results": [
    {
      "title": "Page Title",
      "url": "https://example.com/...",
      "content": "Snippet (max 500 chars)..."
    }
  ]
}
```

---

## Infrastructure

### Purity Hierarchy

Tools are classified by data reliability. The pipeline uses purity to weight confidence scoring.

| Level | Name | Description | Tools |
|-------|------|-------------|-------|
| L1 | Cached/Static | Pre-computed, immutable data | (none currently) |
| L2 | Internal DB | Owned databases, deterministic queries | `kb_search`, `kb_normativa_search` |
| L3 | External API | Official APIs, deterministic lookup | `normattiva_search`, `eurlex_search` |
| L4 | External Scraping | Web sources, non-deterministic | `infolex_search`, `lex_search`, `lex_search_enriched` |
| L5 | Open Web | General web, lowest reliability | `web_search` |

### Semaphore Rate Limiting

Concurrency is controlled via per-category asyncio semaphores defined in `executor.py`. These prevent overloading upstream sources.

| Category | Max Concurrent | Tools |
|----------|---------------|-------|
| `official` | 6 | `normattiva_search`, `eurlex_search` |
| `certified` | 4 | `infolex_search`, `kb_search`, `kb_normativa_search`, `lex_search`, `lex_search_enriched` |
| `web` | 3 | `web_search` |
| `default` | 4 | Any unregistered tool |

### Circuit Breaker

Two circuit breaker implementations exist:

**1. Executor-level (`executor.py:_CircuitBreaker`)**
- Threshold: 5 consecutive failures
- Reset window: 60 seconds
- Tracks failures per source name string
- Returns error dict immediately when open

**2. Guardrails-level (`gateway/guardrails.py:CircuitBreaker`)**
- Threshold: 3 transient failures in a 2-minute window
- Half-open after 30 seconds, probes to verify recovery
- Distinguishes transient (5xx, timeout) from permanent (4xx) failures
- Only transient failures count toward circuit opening
- Used by health probes

### Retry Policy

Retry is handled by `tenacity` in `_post_with_retry_inner()`:

| Property | Value |
|----------|-------|
| Max attempts | 2 (initial + 1 retry) |
| Wait strategy | Random exponential (1-10s) |
| Retryable errors | `httpx.TimeoutException`, `httpx.ConnectError`, `OSError`, HTTP 500/502/503/504 |
| Non-retryable | HTTP 4xx, parse errors |
| Logging | Structured log on each retry via `_before_retry_log` |

### Budget Integration

The `BudgetController` (`gateway/budget.py`) enforces per-turn resource limits. Tools check budget before executing.

| Budget Dimension | Default Limit |
|-----------------|---------------|
| Wall clock | 120 seconds |
| Model calls | 20 per turn |
| Tool calls | 30 per turn |

Features:
- **Pre-execution check**: `budget.check_budget()` raises `BudgetExhausted` before semaphore acquisition
- **Deduplication cache**: Identical tool calls (same name + params hash) return cached results
- **Dedup key format**: `{tool_name}:{md5(sorted_params)[:8]}`

Source: `lexe-core/src/lexe_core/gateway/budget.py`

### Tool Health Monitoring

Endpoint: `GET /api/v1/tools/health`

Runs real probes in parallel for each source (not just container liveness). Source: `lexe-core/src/lexe_core/tools/health.py`

| Source | Probe Query | Degraded Threshold |
|--------|------------|-------------------|
| normattiva | Art. 2043 c.c. | >2000ms latency |
| eurlex | GDPR (Reg. 2016/679) | >2000ms or SPARQL slow |
| infolex | Art. 1 c.c. | >2000ms latency |
| massimario | "responsabilita extracontrattuale" | >2000ms or <3 results |
| web_search | "GDPR compliance" | >2000ms or 0 results |

Status values: `ok`, `degraded`, `down`

Degraded reasons (enum): `sparql_slow`, `rate_limited`, `partial_data`, `cache_stale`, `high_latency`, `circuit_half_open`

Response includes `_circuit_breakers` field with all circuit statuses.

### Tool Effectiveness Tracking

Source: `lexe-core/src/lexe_core/tools/effectiveness.py`

The `ToolEffectivenessTracker` records tool usage outcomes to `core.tool_effectiveness` for few-shot prompt injection (ICRL-inspired).

Tracked fields:
- `tenant_id`, `conversation_id`
- `tool_name`, `arguments`
- `query_text`, `query_domain`, `query_level`
- `results_count`, `confidence_contribution`
- `latency_ms`, `success`

Recording is fire-and-forget (no pipeline blocking on write failure).

### Tool Defaults

`get_tool_definitions()` default flags:

| Tool | Default Enabled |
|------|----------------|
| `normattiva_search` | Yes |
| `eurlex_search` | Yes |
| `infolex_search` | Yes |
| `kb_search` | Yes |
| `kb_normativa_search` | Yes |
| `lex_search` | Yes |
| `lex_search_enriched` | No |
| `web_search` | No |

### Internal Networking

| Service | Internal URL | Port |
|---------|-------------|------|
| lexe-tools | `http://lexe-tools:8021` | 8021 |
| SearXNG | `http://shared-searxng:8080` | 8080 |

All tool calls from lexe-core to lexe-tools are signed with HMAC headers when `internal_hmac_key` is configured (see SECURITY-POSTURE.md).

---

*Last updated: 2026-03-19*
