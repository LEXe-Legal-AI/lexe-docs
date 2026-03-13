# LEXE Orchestration v2 -- Architettura 3-Layer

> **Versione**: 1.0
> **Data**: 2026-03-12
> **Stato**: Phase 0 completata, Phase 1-5 pending
> **Repository**: `lexe-tools-it`, `lexe-orchestration` (futuro), `lexe-core`

---

## Indice

1. [Overview Architetturale](#1-overview-architetturale)
2. [Capability Table](#2-capability-table)
3. [Dispatcher Pattern](#3-dispatcher-pattern)
4. [Singleton Pattern](#4-singleton-pattern)
5. [Config e Feature Flags](#5-config-e-feature-flags)
6. [Endpoint HTTP](#6-endpoint-http)
7. [Pipeline PRVS](#7-pipeline-prvs)
8. [Super Tool Agent](#8-super-tool-agent)
9. [Domain Events](#9-domain-events)
10. [Strategy Pattern](#10-strategy-pattern)
11. [Degradation Rules](#11-degradation-rules)
12. [Migration Phases](#12-migration-phases)
13. [Import Patterns](#13-import-patterns)
14. [Testing -- curl examples](#14-testing----curl-examples)

---

## 1. Overview Architetturale

L'architettura LEXE e stata decomposta da un monolite (`customer_router.py`, 3357 righe in lexe-core) in 3 layer distinti con responsabilita ben definite.

```
                          UTENTE (Browser)
                              |
                              | HTTPS / SSE
                              v
+===================================================================+
|  LAYER 3 -- lexe-core (Edge Layer)              porta 8100       |
|                                                                   |
|  - Auth (Logto OIDC)                                             |
|  - Tenant resolution                                              |
|  - Conversation management (CRUD, persist)                        |
|  - Costruzione TurnContext                                        |
|  - Traduzione DomainEvent -> SSE (contratto frontend invariato)   |
|  - Memory L0-L4 integration                                      |
+===================================================================+
         |                                      |
         | import diretto (intra-processo)       | HTTP (inter-processo)
         v                                      v
+===================================================================+
|  LAYER 1 -- lexe-tools-it (Capability Server)   porta 8021      |
|                                                                   |
|  tools/           capabilities/                                   |
|  +-------------+  +-------------------------------------------+  |
|  | normattiva  |  | intent_classifier  (LLM, lexe-fast)      |  |
|  | eurlex      |  | research_planner   (LLM, lexe-primary)   |  |
|  | infolex     |  | research_executor  (no LLM, dispatcher)  |  |
|  | kb_search   |  | legal_verifier     (LLM, lexe-fast)      |  |
|  | lex_search  |  | legal_synthesizer  (LLM, lexe-primary)   |  |
|  | lex_enricher|  | super_tool_agent   (LLM, lexe-complex)   |  |
|  | vigenza_fast|  +-------------------------------------------+  |
|  | one_search  |                                                  |
|  +-------------+  dispatcher.py --> local dispatch (no HTTP)     |
+===================================================================+
         |
         | HTTP (LiteLLM gateway)
         v
+===================================================================+
|  lexe-litellm (LLM Gateway)                     porta 4000      |
|  20+ modelli: gemini-3-flash, claude-sonnet-4-6, gpt-5-mini...  |
+===================================================================+

+===================================================================+
|  LAYER 2 -- lexe-orchestration (FUTURO)                          |
|                                                                   |
|  - Contracts: TurnContext, IntentDecision, DecompositionPlan     |
|  - Strategy: ToolloopStrategy, LegisStrategy, SuperToolStrategy  |
|  - Decomposer deterministico                                      |
|  - Domain Events: PhaseEvent, TokenEvent, MetaEvent, DoneEvent   |
+===================================================================+
```

### Flusso dati principale (Pipeline LEGIS)

```
User Message
    |
    v
[lexe-core] Auth + TurnContext
    |
    v
[Intent Classifier] --> PipelineLevel + ScenarioType
    |
    v
[Research Planner] --> ResearchPlan (sources_to_query[])
    |
    v
[Research Executor] --> EvidencePack (norms, case_law, massime)
    |     ^
    |     |  dispatcher.execute()
    |     v
    |  [tools/] normattiva, eurlex, infolex, kb_search...
    |
    v
[Legal Verifier] --> VerificationVerdict (pass|warn|fail)
    |
    v
[Legal Synthesizer] --> Streaming text (SSE token events)
    |
    v
[lexe-core] DomainEvent -> SSE -> Frontend
```

### Flusso dati alternativo (Super Tool)

```
User Message
    |
    v
[lexe-core] Auth + TurnContext
    |
    v
[Super Tool Agent] --> LLM con tool calling (agentic loop)
    |     ^
    |     |  dispatcher.execute() / SearXNG
    |     v
    |  [tools/] normattiva, eurlex, infolex, kb_search...
    |
    v
SSE events: phase, tool_call, budget, usage, token
```

---

## 2. Capability Table

Ogni capability risiede in `lexe-tools-it/src/lexe_tools_it/capabilities/`.

| Capability | LLM? | Default Model | Input | Output | Endpoint HTTP | Source File |
|---|---|---|---|---|---|---|
| **Intent Classifier** | Si | `lexe-fast` | `user_message`, `conversation_context` | `IntentResult` (level, scenario, confidence) | `POST /api/v1/tools/intent/classify` | `intent_classifier.py` |
| **Research Planner** | Si | `lexe-primary` | `user_message`, `conversation_context` | `ResearchPlan` (sources_to_query[]) | `POST /api/v1/tools/research/plan` | `research_planner.py` |
| **Research Executor** | No | - | `ResearchPlan`, `execute_tool_fn` | `EvidencePack` | `POST /api/v1/tools/research/execute` | `research_executor.py` |
| **Legal Verifier** | Opzionale | `lexe-fast` | `EvidencePack`, `synthesis?` | `VerificationVerdict` / `VerificationResult` | `POST /api/v1/tools/verify/legal` | `legal_verifier.py` |
| **Legal Synthesizer** | Si | `lexe-primary` | `user_message`, `EvidencePack`, `VerificationVerdict` | `AsyncGenerator[str]` (streaming) | `POST /api/v1/tools/synthesize/legal` | `legal_synthesizer.py` |
| **Super Tool Agent** | Si | `lexe-complex` | `user_message`, `recent_messages`, tenant/contact/conversation IDs | `AsyncGenerator[str]` (SSE events) | `POST /api/v1/tools/agent/super_tool` | `super_tool_agent.py` |

### Source Tools (Layer 1 -- tools/)

| Tool | Feature Flag | Endpoint | Descrizione |
|---|---|---|---|
| `normattiva` | `ff_normattiva_enabled` | `/api/v1/tools/normattiva/search` | Legislazione italiana (Normattiva.it) |
| `eurlex` | `ff_eurlex_enabled` | `/api/v1/tools/eurlex/search` | Legislazione EU (EUR-Lex) |
| `infolex` | `ff_infolex_enabled` | `/api/v1/tools/infolex/search` | Giurisprudenza + dottrina (Brocardi) |
| `kb_search` | `ff_kb_search_enabled` | `/api/v1/tools/kb/search` | Massimario Cassazione (38K+ massime, hybrid search) |
| `kb_normativa_search` | `ff_kb_search_enabled` | `/api/v1/tools/kb/normativa/search` | KB articoli codice (CC, CP, CPC, CPP, COST) |
| `lex_search` | `ff_lex_search_enabled` | `/api/v1/tools/lex_search` | Ricerca web 13 siti legali (SearXNG) |
| `lex_enricher` | `ff_lex_search_enriched_enabled` | `/api/v1/tools/lex_search/enriched` | Ricerca + analisi LLM + vigenza |
| `vigenza_fast` | `ff_vigenza_fast_enabled` | `/api/v1/tools/vigenza/fast` | Verifica vigenza con cache L1/L2 |
| `one_search` | `ff_one_search_enabled` | `/api/v1/tools/one_search` | Orchestratore multi-source best-effort |

---

## 3. Dispatcher Pattern

Il `CapabilityDispatcher` (`dispatcher.py`) e il componente chiave che consente alle capability di invocare i source tool senza roundtrip HTTP quando eseguite nello stesso processo.

### Architettura

```
capability (es. research_executor)
    |
    v
dispatcher.execute("normattiva_search", {...})
    |
    +-- tool registrato localmente? --> dispatch diretto (in-process, ~0ms overhead)
    |
    +-- tool NON registrato?        --> HTTP fallback (POST http://localhost:8021/...)
```

### Implementazione

```python
class CapabilityDispatcher:
    _tools: dict[str, Callable[..., Awaitable[dict]]]  # registry locale
    _base_url: str = "http://localhost:8021"             # fallback HTTP

    def register(self, name: str, fn: Callable) -> None: ...
    async def execute(self, name: str, payload: dict) -> dict: ...
    async def _http_fallback(self, name: str, payload: dict) -> dict: ...
```

### Tool registrati localmente

Al primo accesso al singleton, `_register_local_tools()` registra:

| Tool Name | Callable Locale |
|---|---|
| `normattiva_search` | `normattiva_tool.search()` |
| `eurlex_search` | `eurlex_tool.search()` / `eurlex_tool.search_by_keyword()` |
| `infolex_search` | `infolex_tool.search()` |
| `lex_search` | `lex_search_tool.search()` |
| `lex_search_enriched` | `lex_enricher.search_enriched()` |
| `kb_search` | `kb_search_tool.search()` |

### Endpoint mapping per HTTP fallback

```python
_TOOL_ENDPOINTS = {
    "normattiva_search":    "/api/v1/tools/normattiva/search",
    "eurlex_search":        "/api/v1/tools/eurlex/search",
    "infolex_search":       "/api/v1/tools/infolex/search",
    "kb_search":            "/api/v1/tools/kb/search",
    "kb_normativa_search":  "/api/v1/tools/kb/normativa/search",
    "lex_search":           "/api/v1/tools/lex_search",
    "lex_search_enriched":  "/api/v1/tools/lex_search/enriched",
    "vigenza_fast":         "/api/v1/tools/vigenza/fast",
    "one_search":           "/api/v1/tools/one_search",
}
```

### Sicurezza

Il fallback HTTP supporta autenticazione HMAC opzionale (`internal_hmac_key` in settings). Se configurato, ogni richiesta HTTP viene firmata con `sign_request()`.

---

## 4. Singleton Pattern

Tutte le capability stateless utilizzano un pattern singleton a livello di modulo con lazy initialization.

### Pattern standard

```python
# In ogni capability file (es. intent_classifier.py):

_intent_classifier: IntentClassifier | None = None

def get_intent_classifier() -> IntentClassifier:
    """Get or create the module-level intent classifier singleton."""
    global _intent_classifier
    if _intent_classifier is None:
        _intent_classifier = IntentClassifier()
    return _intent_classifier
```

### Applicazione del pattern

| Capability | Singleton? | Getter | Motivo |
|---|---|---|---|
| IntentClassifier | Si | `get_intent_classifier()` | Stateless, riusa client config |
| ResearchPlanner | Si | `get_research_planner()` | Stateless |
| ResearchExecutor | Si | `get_research_executor()` | Stateless |
| LegalVerifier | Si | `get_legal_verifier()` | Stateless |
| LegalSynthesizer | Si | `get_legal_synthesizer()` | Stateless |
| CapabilityDispatcher | Si | `get_dispatcher()` | Registra tool una volta |
| **SuperToolAgent** | **No** | `create_super_tool_agent()` | **Per-request** (stato: messages, budget, metrics) |

Il `SuperToolAgent` e l'eccezione: ogni richiesta crea una nuova istanza perche mantiene stato request-specific (conversation messages, budget counters, metrics accumulator).

### Export via `__init__.py`

```python
# capabilities/__init__.py
__all__ = [
    "get_intent_classifier",
    "get_research_planner",
    "get_research_executor",
    "get_legal_verifier",
    "get_legal_synthesizer",
    "get_dispatcher",
    "SuperToolAgent",
    "SuperToolFallbackError",
]
```

---

## 5. Config e Feature Flags

Tutta la configurazione e centralizzata in `config.py` tramite `pydantic_settings.BaseSettings` con prefisso `LEXE_`.

### Feature Flags -- Source Tools

| Env Var | Default | Controlla |
|---|---|---|
| `LEXE_FF_NORMATTIVA_ENABLED` | `True` | Tool normattiva |
| `LEXE_FF_EURLEX_ENABLED` | `True` | Tool EUR-Lex |
| `LEXE_FF_INFOLEX_ENABLED` | `True` | Tool InfoLex/Brocardi |
| `LEXE_FF_LEX_SEARCH_ENABLED` | `True` | Tool lex_search (SearXNG) |
| `LEXE_FF_LEX_SEARCH_ENRICHED_ENABLED` | `True` | Tool lex_search enriched |
| `LEXE_FF_KB_SEARCH_ENABLED` | `True` | Tool kb_search + kb_normativa_search |
| `LEXE_FF_VIGENZA_FAST_ENABLED` | `True` | Tool vigenza_fast |
| `LEXE_FF_ONE_SEARCH_ENABLED` | `True` | Tool one_search |

### Feature Flags -- Capabilities

| Env Var | Default | Controlla |
|---|---|---|
| `LEXE_FF_INTENT_CLASSIFIER_ENABLED` | `True` | Intent Classifier capability |
| `LEXE_FF_RESEARCH_PLANNER_ENABLED` | `True` | Research Planner capability |
| `LEXE_FF_RESEARCH_EXECUTOR_ENABLED` | `True` | Research Executor capability |
| `LEXE_FF_LEGAL_VERIFIER_ENABLED` | `True` | Legal Verifier capability |
| `LEXE_FF_LEGAL_SYNTHESIZER_ENABLED` | `True` | Legal Synthesizer capability |
| `LEXE_FF_SUPER_TOOL_AGENT_ENABLED` | `True` | Super Tool Agent capability |

### Model Overrides

Ogni capability con LLM ha un model override. Se vuoto, usa il default indicato.

| Env Var | Default Fallback | Target |
|---|---|---|
| `LEXE_INTENT_CLASSIFIER_MODEL` | `lexe-fast` | Intent Classifier |
| `LEXE_RESEARCH_PLANNER_MODEL` | `lexe-primary` | Research Planner |
| `LEXE_LEGAL_VERIFIER_MODEL` | `lexe-fast` | Legal Verifier |
| `LEXE_LEGAL_SYNTHESIZER_MODEL` | `lexe-primary` | Legal Synthesizer |
| `LEXE_SUPER_TOOL_AGENT_MODEL` | `lexe-complex` | Super Tool Agent |

### Config Override (Catalog JSONB)

Il file `config_utils.py` fornisce `apply_config_override()` per applicare override dal catalog LLM:

```python
_CONFIG_KEYS = (
    "temperature",
    "max_completion_tokens",
    "top_p",
    "frequency_penalty",
    "presence_penalty",
    "reasoning_effort",
)
```

Trasformazioni:
- `max_completion_tokens` -> `max_tokens` (naming LiteLLM)
- `reasoning_effort="none"` -> chiave rimossa (modelli che non supportano reasoning)

### Altre configurazioni rilevanti

| Parametro | Default | Descrizione |
|---|---|---|
| `LEXE_LITELLM_URL` | `http://lexe-litellm:4000` | LLM Gateway |
| `LEXE_LITELLM_API_KEY` | `sk-lexe-internal` | API key LiteLLM |
| `LEXE_LITELLM_MODEL` | `lexe-primary` | Modello default |
| `LEXE_SEARXNG_URL` | `http://shared-searxng:8080` | SearXNG per lex_search |
| `LEXE_KB_DATABASE_URL` | `postgresql://...@lexe-max:5432/lexe_kb` | DB Knowledge Base |
| `LEXE_INTERNAL_HMAC_KEY` | `""` | HMAC inter-service (vuoto = disabilitato) |
| `LEXE_API_PORT` | `8021` | Porta API lexe-tools-it |

---

## 6. Endpoint HTTP

Tutti gli endpoint sono sotto il prefisso `/api/v1/tools` con tag `tools`.

### Capability Endpoints (Orchestration v2)

#### POST `/api/v1/tools/intent/classify`

Classifica l'intento utente per routing pipeline.

**Request**: `IntentClassifyRequest`
```json
{
  "user_message": "Art. 2043 codice civile",
  "conversation_context": "",
  "config_override": null
}
```

**Response**: `IntentClassifyResponse`
```json
{
  "level": "DIRECT",
  "scenario": "ricerca",
  "is_legal": true,
  "legal_domain": "civile",
  "confidence": 0.95,
  "reason": "citazione_norma riferimento_articolo",
  "intent_summary": "Testo art. 2043 codice civile",
  "key_concepts": ["responsabilita extracontrattuale", "danno ingiusto"],
  "references": [{"act_type": "codice civile", "article": "2043"}],
  "detector_model": "lexe-fast",
  "detector_duration_ms": 450
}
```

#### POST `/api/v1/tools/research/plan`

Genera un piano di ricerca strutturato.

**Request**: `ResearchPlanRequest`
```json
{
  "user_message": "Quali rischi corre un cofideiussore?",
  "conversation_context": "",
  "legal_domain": "civile",
  "config_override": null
}
```

**Response**: `ResearchPlanResponse`
```json
{
  "query_understood": "Rischi e responsabilita del cofideiussore",
  "legal_domain": "diritto civile",
  "jurisdiction": "IT",
  "key_norms": ["art. 1936 c.c.", "art. 1944 c.c.", "art. 1945 c.c."],
  "key_concepts": ["fideiussione", "obbligazione solidale"],
  "sources_to_query": [
    {"tool": "normattiva_search", "arguments": {"act_type": "codice civile", "article": "1936"}, "reason": "Testo art. 1936", "priority": 1},
    {"tool": "kb_search", "arguments": {"query": "cofideiussore responsabilita solidale", "top_k": 10}, "reason": "Orientamenti Cassazione", "priority": 1}
  ],
  "deepening_suggestions": [
    {"label": "Quali eccezioni puo opporre il cofideiussore?", "tools": ["kb_search"]}
  ]
}
```

#### POST `/api/v1/tools/research/execute`

Esegue il piano di ricerca e produce un evidence pack.

**Request**: `ResearchExecuteRequest`
```json
{
  "research_plan": { ... },
  "config_override": null
}
```

**Response**: `ResearchExecuteResponse`
```json
{
  "evidence_pack": {
    "id": "...",
    "query": "...",
    "norms": [...],
    "case_law": [...],
    "massime": [...],
    "coverage": {...},
    "confidence": {...}
  },
  "duration_ms": 3200
}
```

#### POST `/api/v1/tools/verify/legal`

Verifica integrita dell'evidence pack. Due modalita:

1. **Simple verification** (senza `synthesis`): check deterministici + opzionale LLM coherence
2. **Self-correction** (con `synthesis`): verifica + ri-sintesi se fallisce (max 2 tentativi, 15s timeout)

**Request**: `VerifyLegalRequest`
```json
{
  "evidence_pack": { ... },
  "synthesis": "",
  "use_llm_coherence": false,
  "config_override": null
}
```

**Response**: `VerifyLegalResponse`
```json
{
  "status": "pass",
  "confidence": 0.85,
  "checks_run": 5,
  "checks_passed": 4,
  "issues": [],
  "summary_it": "Verifica strutturale completata.",
  "model_used": "deterministic_only",
  "duration_ms": 120
}
```

#### POST `/api/v1/tools/synthesize/legal`

Sintetizza la risposta legale in streaming (SSE).

**Request**: `SynthesizeLegalRequest`
```json
{
  "user_message": "Art. 2043 codice civile",
  "evidence_pack": { ... },
  "verification": { ... },
  "synth_prompt_key": "default",
  "memory_context": null,
  "config_override": null
}
```

**Response**: `StreamingResponse` (text/event-stream) con chunk di testo.

Prompt scenarios disponibili per `synth_prompt_key`:

| Key | Scenario | Descrizione |
|---|---|---|
| `default` | Generico | Risposta completa con sezioni A-G |
| `minimal` | DIRECT | Lookup puntuale, solo articolo + vigenza + link |
| `concise` | SIMPLE | Sintesi breve 3-5 righe |
| `parere` | Parere | Struttura IRAC con matrice rischio |
| `contratto_redazione` | Redazione | Bozza contratto 12 sezioni |
| `contratto_revisione` | Revisione | Report clausola per clausola con rating |
| `contenzioso` | Contenzioso | Analisi strategica con matrice rischio |
| `recupero_crediti` | Recupero crediti | Procedura monitoria/ordinaria + costi |

#### POST `/api/v1/tools/agent/super_tool`

Esegue il Super Tool Agent in streaming (SSE).

**Request**: `SuperToolRequest`
```json
{
  "user_message": "Spiegami la responsabilita del medico",
  "recent_messages": [],
  "tenant_id": "...",
  "contact_id": "...",
  "conversation_id": "...",
  "correlation_id": "req-123",
  "trace_id": "abc",
  "memory_context": null
}
```

**Response**: `StreamingResponse` (text/event-stream) con SSE events (vedi sezione 9).

---

## 7. Pipeline PRVS

PRVS = **P**lanner -> **R**esearcher -> **V**erifier -> **S**ynthesizer

### 7.1 Intent Classifier (Phase 0)

Determina `PipelineLevel` e `ScenarioType` con una singola LLM call.

**PipelineLevel** (5 livelli crescenti di complessita):

| Level | max_sources | skip_verifier | skip_graph | synthesizer_depth | planner_model |
|---|---|---|---|---|---|
| `CHAT` | 0 | Si | Si | `none` | None |
| `DIRECT` | 3 | Si | Si | `minimal` | None |
| `SIMPLE` | 4 | Si | Si | `concise` | `lexe-fast` |
| `STANDARD` | 6 | No | No | `full` | `lexe-fast` |
| `COMPLEX` | 8 | No | No | `full` | None (usa default) |

**ScenarioType** (8 scenari):

| Scenario | Synth Prompt | Descrizione |
|---|---|---|
| `chat` | - | Conversazione non legale |
| `ricerca` | `default` | Ricerca normativa/giurisprudenziale |
| `contratto_redazione` | `contratto_redazione` | Bozza di contratto |
| `contratto_revisione` | `contratto_revisione` | Revisione documento legale |
| `parere` | `parere` | Parere pro-veritate |
| `contenzioso` | `contenzioso` | Analisi contenzioso |
| `recupero_crediti` | `recupero_crediti` | Procedura recupero crediti |
| `custom` | `default` | Altro |

**Fallback**: in caso di errore LLM, timeout, o JSON invalido -> `CHAT` + `is_legal=false` (fail safe).

**DIRECT plan builder**: per `PipelineLevel.DIRECT`, il piano viene costruito deterministicamente dai riferimenti estratti senza LLM call aggiuntiva (`build_direct_plan()`).

### 7.2 Research Planner (Phase P)

Riceve la query utente e produce un `ResearchPlan` con tool call pianificate.

**Output**: `ResearchPlan`
```
query_understood     str     -- comprensione della query
legal_domain         str     -- dominio legale
temporal_scope       str?    -- ambito temporale
jurisdiction         str     -- giurisdizione (default "IT")
key_norms            list    -- norme chiave identificate
key_concepts         list    -- concetti chiave
sources_to_query     list    -- SourceQuery[] (tool + arguments + priority)
deepening_suggestions list?  -- approfondimenti suggeriti
disambiguation_needed list?  -- domande di disambiguazione
```

**SourceQuery** contiene:
- `tool`: nome del tool da invocare
- `arguments`: parametri del tool
- `reason`: motivazione
- `priority`: 1 (parallelo) o 2 (sequenziale)
- `depends_on`: dipendenze opzionali

**Truncation repair**: se il LLM produce JSON troncato (`finish_reason=length`), viene applicato `_repair_truncated_json()` che analizza bracket nesting e chiude i blocchi aperti.

**Fallback plan**: in caso di errore -> piano con `kb_search` + `lex_search` + `infolex_search`.

### 7.3 Research Executor (Phase R)

Esegue il piano chiamando i tool via dispatcher e costruisce un `EvidencePack`.

**Strategia di esecuzione**:
1. Priority 1: esecuzione **parallela** (`asyncio.gather`)
2. Priority 2: esecuzione **sequenziale**
3. Budget: max 20 tool call totali (configurabile)

**Limiti per-tool** (hard anti-runaway):

| Tool | Max Chiamate |
|---|---|
| `normattiva_search` | 8 |
| `eurlex_search` | 6 |
| `infolex_search` | 6 |
| `kb_search` | 8 |
| `lex_search` | 6 |
| `lex_search_enriched` | 4 |
| `vigenza_fast` | 15 |
| `one_search` | 3 |
| `web_search` | 3 |

**Deduplicazione**: ogni tool call genera un `dedup_key` (SHA-256 di `tool_name:args_json`). Le chiamate duplicate vengono silenziosamente ignorate.

**Normalizzazione risultati**: ogni tool ha un normalizzatore dedicato che converte i risultati raw in oggetti tipizzati:

| Tool | Tipo Output | Normalizzatore |
|---|---|---|
| `normattiva_search` | `NormRef` | `_normalize_normattiva()` |
| `eurlex_search` | `NormRef` | `_normalize_eurlex()` |
| `infolex_search` | `CaseLawRef` (massime) | `_normalize_infolex()` |
| `kb_search` | `MassimaRef` | `_normalize_kb_search()` |
| `lex_search` / `lex_search_enriched` | `CaseLawRef` | `_normalize_lex_search()` |

**EvidencePack** -- struttura finale:

```
EvidencePack
  norms:       list[NormRef]        -- norme con testo, vigenza, URN/CELEX
  case_law:    list[CaseLawRef]     -- giurisprudenza
  massime:     list[MassimaRef]     -- massime Cassazione
  conflicts:   list[Conflict]       -- contraddizioni tra fonti
  timeline:    list[TimelineEvent]  -- evoluzione cronologica
  coverage:    dict[str, SourceCoverage]  -- status per fonte
  confidence:  ConfidenceScore      -- score aggregato
```

### 7.4 Legal Verifier (Phase V)

Verifica integrita dell'evidence pack in due step:

**Step 1 -- Check deterministici**:
- Presenza di almeno una evidenza
- Vigenza verificata per ogni norma
- Status: `pass` (0 errori), `warn` (1+ issue), `fail` (2+ error)

**Step 2 -- LLM coherence check** (opzionale, automatico se):
- 5+ norme nell'evidence pack
- Dominio sensibile (penale, lavoro, privacy, tributario, fiscale)
- Temporal scope non-vigente
- Mix fonti IT + EU
- Vigenza confidence < 0.70 per qualche norma

**Self-correction loop** (quando `synthesis` e fornita):
1. Attempt 1: verifica completa (budget: 60% del timeout di 15s)
2. Se fail -> Attempt 2: reflection LLM per ri-sintetizzare correggendo le issue specifiche
3. Ri-verifica della sintesi corretta
4. Output: `VerificationResult` con `confidence_level`: `VERIFIED` | `CAUTION` | `LOW_CONFIDENCE`

### 7.5 Legal Synthesizer (Phase S)

Produce la risposta finale in streaming via LLM.

**Costruzione prompt**:
1. Selezione template da `synth_prompt_key` (mappa scenario -> template)
2. Formattazione evidence pack: ogni item diventa un blocco `[N] TYPE: title / excerpt / source / confidence / url`
3. Inclusione conflicts, timeline, deepening suggestions
4. Verifica status e confidence nel prompt
5. Opzionale: memory context appeso in coda

**Streaming**: connessione SSE a LiteLLM, parsing chunk-by-chunk delle `data:` lines, yield di ogni `delta.content`.

**Parametri LLM default**:
- `temperature`: 0.3
- `max_tokens`: 16384
- Timeout: 120s

---

## 8. Super Tool Agent

Il Super Tool Agent e un agente LLM con tool calling nativo, alternativo alla pipeline PRVS per query complesse.

### Architettura

```
SuperToolAgent.run() -- AsyncGenerator[str]
    |
    for round in range(max_rounds=8):
    |
    +-- check budget (tool calls, wall clock)
    +-- LLM call (non-streaming, with tools[])
    |     |
    |     +-- tool_calls? --> bridge validate --> dispatcher.execute()
    |     |                                       --> SSE tool_call event
    |     |                                       --> append tool result to messages
    |     +-- no tool_calls? --> extract text --> SSE token events
    |                                          --> break
    |
    +-- SSE usage event + tool_run_end event
```

### Budget System

```python
class SimpleBudget:
    max_tool_calls: int = 12
    _dedup_keys: set[str]   # dedup cache
    _cache: dict[str, Any]  # result cache per dedup hit
```

| Risorsa | Limite Default | Comportamento a esaurimento |
|---|---|---|
| Wall clock | 120s | Degraded output con fonti raccolte |
| API rounds | 8 | Force `tool_choice=none` (solo testo) |
| Tool calls | 12 | Errore budget nel tool result |
| Stdout bytes | 64KB | Troncamento output |

### Tool Bridge

Ogni tool call passa per `_execute_tool_bridge()` che:

1. **Valida il nome tool** contro `VALID_TOOL_NAMES` (frozenset di 8 tool)
2. **Valida dimensione input** (max 4KB JSON)
3. **Denylist check**: URL non consentiti in input (eccetto `web_search`)
4. **Dispatch**: `web_search` via SearXNG HTTP, tutti gli altri via local dispatcher
5. **Envelope**: risultato wrappato in `ToolBridgeResult(ok, tool, latency_ms, payload|error)`

### Tool Definitions

I tool sono definiti in formato Anthropic/OpenAI-compatible (`super_tool_definitions.py`):

| Tool | Purity Level | Abilitato per Default |
|---|---|---|
| `normattiva_search` | 3 | Si |
| `eurlex_search` | 3 | Si |
| `infolex_search` | 4 | Si |
| `kb_search` | 2 | Si |
| `kb_normativa_search` | 2 | Si |
| `lex_search` | 4 | Si |
| `web_search` | 5 | **No** |
| `lex_search_enriched` | 4 | **No** |

Purity level: 1 = piu affidabile, 5 = meno affidabile (metrica audit/telemetry).

### SSE Events emessi

| Event Type | Quando | Payload |
|---|---|---|
| `super_tool_phase` | Transizione fase | `phase`, `detail`, `ts` |
| `super_tool_tool_call` | Dopo ogni tool call | `tool`, `ok`, `latency_ms`, `error_code?` |
| `super_tool_budget` | Ogni round | `api_rounds`, `tool_calls`, `remaining_calls`, `remaining_time_ms` |
| `super_tool_usage` | Fine run | `model`, `input_tokens`, `output_tokens`, `cost_usd_est`, `stop_reason` |
| `token` | Chunk testo | `content` |
| `tool_run_end` | Fine run | `status`, `duration_ms`, `tools_executed`, `tools_failed` |

### Fasi del Super Tool

1. `analyzing` -- Analisi query e pianificazione
2. `tooling` -- Esecuzione tool call (ripetuta per ogni tool)
3. `synthesizing` -- Generazione risposta testuale
4. `finalizing` -- Wrap-up e metriche

### Pricing v1

| Model | Input ($/1M tok) | Output ($/1M tok) |
|---|---|---|
| `claude-sonnet-4-6` | 3.00 | 15.00 |
| `claude-opus-4-6` | 5.00 | 25.00 |

### Degraded Output

Se il budget si esaurisce prima di produrre testo finale, l'agent emette output parziale con:
- Testo parziale raccolto fino a quel punto
- Nota con motivo del degradamento
- Lista fonti consultate con status (OK/Errore) e latenza

### Fallback

`SuperToolFallbackError` viene lanciata su errori irrecuperabili (es. LiteLLM 500). Il router in lexe-core la cattura e fallisce sul pipeline LEGIS.

---

## 9. Domain Events

### Attuale (Phase 0): SSE diretto

Nella fase attuale, gli eventi SSE sono generati direttamente dalle capability e inviati al frontend.

#### Formato eventi Super Tool

```
event: <event_type>
data: {"type": "<event_type>", "correlation_id": "...", "trace_id": "...", ...}
```

#### Formato eventi token (streaming testo)

```
event: token
data: {"content": "chunk di testo"}
```

### Futuro (Layer 2): Domain Events

Il Layer 2 (`lexe-orchestration`) introdurra Domain Events tipizzati e la traduzione in SSE avverra nel Layer 3.

**Tipi pianificati**:

| Domain Event | Descrizione | Traduzione SSE |
|---|---|---|
| `PhaseEvent` | Transizione di fase pipeline | `event: phase` |
| `TokenEvent` | Chunk di testo streaming | `event: token` |
| `MetaEvent` | Metadati (evidence pack, verification) | `event: meta` |
| `DoneEvent` | Completamento con metriche | `event: done` |

**Principio**: i Domain Events NON sono SSE. Sono oggetti Python/dataclass emessi dalle strategy. La traduzione DomainEvent -> SSE avviene esclusivamente in lexe-core (Layer 3), mantenendo il contratto frontend invariato.

---

## 10. Strategy Pattern

### Attuale (Phase 0)

La selezione della strategia e implicita nel router di lexe-core basata su `PipelineLevel`:

```
CHAT     --> direct LLM response (no pipeline)
DIRECT   --> build_direct_plan() --> PRVS (skip verifier)
SIMPLE   --> Planner(lexe-fast) --> PRVS (skip verifier)
STANDARD --> Planner(lexe-fast) --> PRVS (full)
COMPLEX  --> Planner + PRVS (full, max 8 sources)
```

Super Tool: attivato da `PipelineLevel >= super_tool_min_level` o prefisso `/supertool`.

### Futuro (Layer 2): Strategy ABC

```python
class OrchestratorStrategy(ABC):
    @abstractmethod
    async def execute(self, turn: TurnContext) -> AsyncGenerator[DomainEvent]: ...

class ToolloopStrategy(OrchestratorStrategy): ...   # CHAT/DIRECT
class LegisStrategy(OrchestratorStrategy): ...       # SIMPLE/STANDARD/COMPLEX (PRVS)
class SuperToolStrategy(OrchestratorStrategy): ...   # Super Tool agentic
class LexorcStrategy(OrchestratorStrategy): ...      # LEXORC multi-fase
```

**Decomposer deterministico**: seleziona la strategy basandosi su `IntentResult`:

```python
def select_strategy(intent: IntentResult) -> OrchestratorStrategy:
    if not intent.is_legal:
        return ToolloopStrategy()
    if intent.level == PipelineLevel.DIRECT:
        return ToolloopStrategy()
    if user_requested_super_tool:
        return SuperToolStrategy()
    return LegisStrategy()
```

**Candidate scoring** (futuro): per i casi borderline tra LEGIS e SuperTool, un sistema di scoring pesato valutera:
- Complessita query (numero vincoli, sotto-domande)
- Numero fonti previste
- Dominio legale (penale/tributario -> LEGIS per maggior rigore)
- Budget disponibile

---

## 11. Degradation Rules

### Per-Capability Failure

| Capability | Failure Mode | Degradazione |
|---|---|---|
| **Intent Classifier** | LLM timeout/error, JSON invalido | Fallback a `CHAT` + `is_legal=false` (fail safe conservativo) |
| **Research Planner** | LLM timeout/error, JSON invalido | Fallback plan generico: `kb_search` + `lex_search` + `infolex_search` |
| **Research Planner** | JSON troncato (`finish_reason=length`) | Repair automatico con bracket nesting analysis |
| **Research Executor** | Tool individuale fallisce | Continua con altri tool, marca coverage come `partial`/`failed` |
| **Research Executor** | Budget tool call esaurito | Stop esecuzione, ritorna evidence pack parziale |
| **Research Executor** | Dedup hit | Silenziosamente ignorato (non conta come errore) |
| **Legal Verifier** | LLM coherence timeout | Ritorna solo check deterministici (graceful) |
| **Legal Verifier** | Self-correction timeout | Ritorna sintesi originale con `CAUTION` |
| **Legal Synthesizer** | LLM timeout | Messaggio di errore nel stream + suggerimento di consultare evidence pack |
| **Legal Synthesizer** | LLM error | Messaggio di errore nel stream |
| **Super Tool Agent** | Wall clock esaurito | Degraded output con fonti raccolte |
| **Super Tool Agent** | Budget tool call esaurito | Force `tool_choice=none`, sintetizza con quello che ha |
| **Super Tool Agent** | LLM irrecuperabile (500) | `SuperToolFallbackError` -> fallback a LEGIS |
| **Super Tool Agent** | Stdout limit | Troncamento output |

### Per-Tool Failure (Source Tools)

| Tool | Circuit Breaker | Fallback |
|---|---|---|
| `normattiva` | Si | Segnalato come `failed` in coverage |
| `eurlex` | Si | Segnalato come `failed` in coverage |
| `infolex` | Si | Segnalato come `failed` in coverage |
| `kb_search` | No (DB-based) | Errore propagato |
| `lex_search` | No (SearXNG) | Risposta con `success=false` |
| `vigenza_fast` | No | Eccezione propagata |

### LLM Coherence Check Trigger Automatico

Il verifier attiva automaticamente il check LLM quando rileva condizioni ad alto rischio:

1. 5+ norme nell'evidence pack (complessita)
2. Domini sensibili: penale, lavoro, privacy, tributario, fiscale
3. Temporal scope non-vigente (storico)
4. Mix fonti IT + EU (cross-giurisdizione)
5. Vigenza confidence < 0.70 su qualche norma

---

## 12. Migration Phases

### Phase 0 -- Capability Extraction (COMPLETATA)

**Stato**: Completata.

**Cosa e stato fatto**:
- Estratte tutte le capability da lexe-core in lexe-tools-it/capabilities/
- Intent Classifier, Research Planner, Research Executor, Legal Verifier, Legal Synthesizer
- Super Tool Agent migrato con dispatcher locale
- Dispatcher pattern implementato (local dispatch + HTTP fallback)
- Tutti gli endpoint HTTP capability funzionanti
- Prompt templates migrati in `prompts.py`
- Models migrati in `models.py`
- Config override migrato in `config_utils.py`
- Super Tool definitions e events migrati

**File migrati da lexe-core**:
- `agent/intent_detector.py` -> `capabilities/intent_classifier.py`
- `agent/planner.py` -> `capabilities/research_planner.py`
- `agent/researcher.py` -> `capabilities/research_executor.py`
- `agent/verifier.py` -> `capabilities/legal_verifier.py`
- `agent/synthesizer.py` -> `capabilities/legal_synthesizer.py`
- `agent/super_tool.py` -> `capabilities/super_tool_agent.py`
- `agent/super_tool_definitions.py` -> `capabilities/super_tool_definitions.py`
- `agent/super_tool_events.py` -> `capabilities/super_tool_events.py`
- `agent/prompts.py` -> `capabilities/prompts.py`
- `agent/models.py` -> `capabilities/models.py`
- `agent/config_utils.py` -> `capabilities/config_utils.py`

### Phase 1 -- lexe-core Thinning (PENDING)

**Obiettivo**: ridurre `customer_router.py` a thin edge layer.

**Task**:
- [ ] lexe-core importa capability direttamente da lexe-tools-it (import Python, non HTTP)
- [ ] Rimuovere logica duplicata in lexe-core/agent/
- [ ] TurnContext costruito in lexe-core, passato alle capability
- [ ] SSE translation rimane in lexe-core

### Phase 2 -- Contract Definition (PENDING)

**Obiettivo**: definire i contratti formali per il Layer 2.

**Task**:
- [ ] Definire `TurnContext` dataclass
- [ ] Definire `IntentDecision` dataclass
- [ ] Definire `DecompositionPlan` dataclass
- [ ] Definire `Capability` enum
- [ ] Definire `DomainEvent` hierarchy (PhaseEvent, TokenEvent, MetaEvent, DoneEvent)

### Phase 3 -- Strategy Pattern (PENDING)

**Obiettivo**: implementare il decomposer e le strategy nel Layer 2.

**Task**:
- [ ] `OrchestratorStrategy` ABC
- [ ] `ToolloopStrategy` (CHAT/DIRECT)
- [ ] `LegisStrategy` (SIMPLE/STANDARD/COMPLEX)
- [ ] `SuperToolStrategy` (agentic)
- [ ] Decomposer deterministico con candidate scoring
- [ ] Domain Event emission dalle strategy

### Phase 4 -- lexe-orchestration Repository (PENDING)

**Obiettivo**: creare il repository `lexe-orchestration` come package indipendente.

**Task**:
- [ ] Setup repository con pyproject.toml
- [ ] Spostare contracts, strategies, decomposer
- [ ] lexe-core dipende da lexe-orchestration (import Python)
- [ ] lexe-orchestration dipende da lexe-tools-it (capabilities)

### Phase 5 -- HTTP Decoupling (PENDING)

**Obiettivo**: lexe-orchestration comunica con lexe-tools-it via HTTP (microservizi separati).

**Task**:
- [ ] Rimuovere dipendenza Python diretta
- [ ] Comunicazione inter-servizio via HTTP con HMAC
- [ ] Service discovery / health check
- [ ] Circuit breaker inter-servizio

---

## 13. Import Patterns

### Pattern attuale: import diretto (Python)

lexe-core importa le capability come package Python:

```python
# In lexe-core
from lexe_tools_it.capabilities import (
    get_intent_classifier,
    get_research_planner,
    get_research_executor,
    get_legal_verifier,
    get_legal_synthesizer,
    get_dispatcher,
    SuperToolAgent,
)

# Uso
classifier = get_intent_classifier()
result = await classifier.classify(query="Art. 2043 codice civile")
```

### Pattern: import di modelli

```python
from lexe_tools_it.capabilities.models import (
    IntentResult,
    PipelineLevel,
    ScenarioType,
    ResearchPlan,
    SourceQuery,
    EvidencePack,
    VerificationVerdict,
    VerificationResult,
)
```

### Pattern: uso del dispatcher per esecuzione tool

```python
from lexe_tools_it.capabilities.dispatcher import get_dispatcher

dispatcher = get_dispatcher()

# Esecuzione locale (in-process, nessun HTTP)
result = await dispatcher.execute("normattiva_search", {
    "act_type": "codice civile",
    "article": "2043",
})
```

### Pattern: Research Executor con dispatcher

Il Research Executor richiede un `execute_tool_fn` callable:

```python
executor = get_research_executor()
dispatcher = get_dispatcher()

evidence_pack = await executor.execute(
    plan=research_plan,
    execute_tool_fn=dispatcher.execute,  # callable(tool_name, args) -> dict
    max_total_calls=20,
)
```

### Pattern: Super Tool Agent (per-request)

```python
agent = SuperToolAgent(
    user_message="...",
    recent_messages=[...],
    tenant_id=uuid,
    contact_id=uuid,
    conversation_id=uuid,
    litellm_url="http://lexe-litellm:4000",
    litellm_api_key="sk-lexe-internal",
    correlation_id="req-123",
    trace_id="abc",
)

async for event in agent.run():
    # event e una stringa SSE formattata
    yield event
```

### Pattern: config override dal catalog

```python
from lexe_tools_it.capabilities.config_utils import apply_config_override

# Il catalog config JSONB del modello contiene:
config = {"reasoning_effort": "medium", "temperature": 0.1}

defaults = {"temperature": 0.3, "max_tokens": 4096}
apply_config_override(defaults, config)
# defaults ora = {"temperature": 0.1, "max_tokens": 4096, "reasoning_effort": "medium"}
```

---

## 14. Testing -- curl examples

Tutti gli esempi assumono `lexe-tools-it` in esecuzione su `localhost:8021`.

### Intent Classifier

```bash
curl -s -X POST http://localhost:8021/api/v1/tools/intent/classify \
  -H "Content-Type: application/json" \
  -d '{
    "user_message": "Art. 2043 codice civile",
    "conversation_context": ""
  }' | jq .
```

### Research Planner

```bash
curl -s -X POST http://localhost:8021/api/v1/tools/research/plan \
  -H "Content-Type: application/json" \
  -d '{
    "user_message": "Quali rischi corre un cofideiussore?",
    "conversation_context": "",
    "legal_domain": "civile"
  }' | jq .
```

### Research Executor

```bash
curl -s -X POST http://localhost:8021/api/v1/tools/research/execute \
  -H "Content-Type: application/json" \
  -d '{
    "research_plan": {
      "query_understood": "Rischi del cofideiussore",
      "legal_domain": "civile",
      "jurisdiction": "IT",
      "key_norms": ["art. 1936 c.c."],
      "key_concepts": ["fideiussione"],
      "sources_to_query": [
        {
          "tool": "normattiva_search",
          "arguments": {"act_type": "codice civile", "article": "1936"},
          "reason": "Testo art. 1936",
          "priority": 1
        },
        {
          "tool": "kb_search",
          "arguments": {"query": "cofideiussore responsabilita solidale", "top_k": 5},
          "reason": "Orientamenti Cassazione",
          "priority": 1
        }
      ]
    }
  }' | jq .
```

### Legal Verifier (simple)

```bash
curl -s -X POST http://localhost:8021/api/v1/tools/verify/legal \
  -H "Content-Type: application/json" \
  -d '{
    "evidence_pack": {
      "query": "art. 2043 c.c.",
      "norms": [{
        "act_type": "codice civile",
        "article": "2043",
        "text_excerpt": "Qualunque fatto doloso o colposo...",
        "source": "normattiva",
        "vigenza": {"is_vigente": true, "verified": true, "confidence": 0.9}
      }],
      "case_law": [],
      "massime": []
    },
    "use_llm_coherence": false
  }' | jq .
```

### Legal Synthesizer (streaming)

```bash
curl -s -N -X POST http://localhost:8021/api/v1/tools/synthesize/legal \
  -H "Content-Type: application/json" \
  -d '{
    "user_message": "Art. 2043 codice civile",
    "evidence_pack": {
      "query": "art. 2043 c.c.",
      "norms": [{
        "act_type": "codice civile",
        "article": "2043",
        "text_excerpt": "Qualunque fatto doloso o colposo che cagiona ad altri un danno ingiusto...",
        "source": "normattiva",
        "url": "https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:regio.decreto:1942-03-16;262~art2043",
        "vigenza": {"is_vigente": true, "verified": true, "confidence": 0.9}
      }],
      "case_law": [],
      "massime": []
    },
    "verification": {"status": "pass", "confidence": 0.9},
    "synth_prompt_key": "minimal"
  }'
```

### Super Tool Agent (streaming SSE)

```bash
curl -s -N -X POST http://localhost:8021/api/v1/tools/agent/super_tool \
  -H "Content-Type: application/json" \
  -d '{
    "user_message": "Spiegami la responsabilita del medico per errore diagnostico",
    "recent_messages": [],
    "tenant_id": "00000000-0000-0000-0000-000000000001",
    "contact_id": "00000000-0000-0000-0000-000000000002",
    "conversation_id": "00000000-0000-0000-0000-000000000003",
    "correlation_id": "test-123",
    "trace_id": "test-abc"
  }'
```

### Source Tools

```bash
# Normattiva
curl -s -X POST http://localhost:8021/api/v1/tools/normattiva/search \
  -H "Content-Type: application/json" \
  -d '{"act_type": "codice civile", "article": "2043"}' | jq .

# EUR-Lex (structured)
curl -s -X POST http://localhost:8021/api/v1/tools/eurlex/search \
  -H "Content-Type: application/json" \
  -d '{"act_type": "regolamento", "year": 2016, "number": 679, "article": "6"}' | jq .

# EUR-Lex (keyword)
curl -s -X POST http://localhost:8021/api/v1/tools/eurlex/search \
  -H "Content-Type: application/json" \
  -d '{"keyword": "GDPR"}' | jq .

# InfoLex/Brocardi
curl -s -X POST http://localhost:8021/api/v1/tools/infolex/search \
  -H "Content-Type: application/json" \
  -d '{"act_type": "codice civile", "article": "2043", "include_massime": true}' | jq .

# KB Massimario
curl -s -X POST http://localhost:8021/api/v1/tools/kb/search \
  -H "Content-Type: application/json" \
  -d '{"query": "danno ingiusto responsabilita extracontrattuale", "top_k": 5}' | jq .

# KB Normativa
curl -s -X POST http://localhost:8021/api/v1/tools/kb/normativa/search \
  -H "Content-Type: application/json" \
  -d '{"query": "risarcimento danno ingiusto", "top_k": 5, "codes": ["CC"]}' | jq .

# Lex Search
curl -s -X POST http://localhost:8021/api/v1/tools/lex_search \
  -H "Content-Type: application/json" \
  -d '{"query": "responsabilita medica errore diagnostico", "scope": "it"}' | jq .

# Lex Search Enriched
curl -s -X POST http://localhost:8021/api/v1/tools/lex_search/enriched \
  -H "Content-Type: application/json" \
  -d '{"query": "responsabilita medica", "scope": "all"}' | jq .

# Vigenza Fast
curl -s -X POST http://localhost:8021/api/v1/tools/vigenza/fast \
  -H "Content-Type: application/json" \
  -d '{"urn": "urn:nir:stato:regio.decreto:1942-03-16;262~art2043"}' | jq .

# One Search
curl -s -X POST http://localhost:8021/api/v1/tools/one_search \
  -H "Content-Type: application/json" \
  -d '{"query": "responsabilita medica", "scope": "all", "expert_mode": true}' | jq .

# Tool Status
curl -s http://localhost:8021/api/v1/tools/status | jq .

# Vigenza Stats
curl -s http://localhost:8021/api/v1/tools/vigenza/stats | jq .
```

---

## Appendice A: Data Models Reference

### IntentResult

| Campo | Tipo | Descrizione |
|---|---|---|
| `level` | `PipelineLevel` | CHAT, DIRECT, SIMPLE, STANDARD, COMPLEX |
| `scenario` | `ScenarioType` | chat, ricerca, contratto_redazione, contratto_revisione, parere, contenzioso, recupero_crediti, custom |
| `is_legal` | `bool` | Query legale o conversazionale |
| `legal_domain` | `str` | civile, penale, lavoro, tributario, amministrativo, privacy, societario, consumatore, famiglia, altro |
| `routing_confidence` | `float` | 0.0-1.0 |
| `references` | `list[ExtractedReference]` | Riferimenti normativi estratti |
| `key_concepts` | `list[str]` | 2-5 concetti chiave |
| `max_sources` | `int` | Max fonti per il planner |
| `skip_verifier` | `bool` | Skip verifier per livelli bassi |
| `skip_graph` | `bool` | Skip graph expansion |
| `synthesizer_depth` | `str` | none, minimal, concise, full |

### EvidencePack

| Campo | Tipo | Descrizione |
|---|---|---|
| `id` | `UUID` | ID univoco |
| `query` | `str` | Query originale |
| `norms` | `list[NormRef]` | Norme con testo, vigenza, URN/CELEX |
| `case_law` | `list[CaseLawRef]` | Giurisprudenza (autorita, principio) |
| `massime` | `list[MassimaRef]` | Massime Cassazione (RV, testo, score) |
| `conflicts` | `list[Conflict]` | Contraddizioni tra fonti |
| `timeline` | `list[TimelineEvent]` | Evoluzione cronologica |
| `coverage` | `dict[str, SourceCoverage]` | Status per fonte (ok, partial, failed, skipped) |
| `confidence` | `ConfidenceScore` | Score: overall, normativa, giurisprudenza, vigenza, coverage, coherence |

### VerificationVerdict

| Campo | Tipo | Descrizione |
|---|---|---|
| `status` | `str` | pass, warn, fail |
| `confidence` | `float` | 0.0-1.0 |
| `checks_run` | `int` | Numero check eseguiti |
| `checks_passed` | `int` | Numero check superati |
| `issues` | `list[VerificationIssue]` | Issue trovate (tipo, severita, descrizione) |
| `summary_it` | `str` | Riassunto italiano |

### VerificationIssueType

| Tipo | Descrizione |
|---|---|
| `CITATION_NOT_FOUND` | Citazione non trovata |
| `VIGENZA_MISMATCH` | Vigenza non verificata o errata |
| `STRUCTURAL_INVALID` | Struttura evidence pack invalida |
| `CROSS_REF_BROKEN` | Riferimento incrociato rotto |
| `COHERENCE_ISSUE` | Problema di coerenza logica |
| `ORIENTAMENTO_SUPERATO` | Orientamento giurisprudenziale superato |

---

*Ultimo aggiornamento: 2026-03-12*
