# HANDOFF: 9 Legal Tools Webchat - Guida Completa

> Data: 2026-02-10
> Scope: Tutti i 9 tool della webchat LEXE - architettura, payload, timeout, retry, output

---

## Overview Architettura

```
[Webchat UI]                    [lexe-core]                    [lexe-tools-it]
ToolToggleBar.tsx               customer_router.py              /api/v1/tools/*
  |                                |                               |
  | configStore (Zustand)          | CustomerStreamRequest          |
  v                                v                               v
useStreaming.ts  --POST JSON-->  gateway/customer_router.py       endpoint HTTP
  body: {                          |                               |
    *_enabled: bool               definitions.py                   |
  }                                get_tool_definitions()          |
                                   |                               |
                                  executor.py                      |
                                   execute_tool() --httpx POST-->  |
                                   |                               |
                                  format_tool_result_for_llm()     |
                                   |                               |
                                  [LLM via LiteLLM]               |
```

### Flusso Completo

1. **UI Toggle** - L'utente clicca un bottone in `ToolToggleBar.tsx`
2. **Zustand Store** - Lo stato viene salvato in `configStore` (chiave `enabledTools`)
3. **Streaming Request** - `useStreaming.ts` legge `configState.enabledTools` e invia i toggle come `bool` nel body POST
4. **Gateway** - `customer_router.py` riceve `CustomerStreamRequest` con i toggle, li logga, li passa a `get_tool_definitions()`
5. **Tool Definitions** - `definitions.py` filtra le tool definitions OpenAI-format in base ai toggle attivi
6. **LLM Call** - LiteLLM riceve `messages + tools[]` e decide se chiamare un tool
7. **Execution** - `executor.py` chiama l'endpoint `lexe-tools-it` corrispondente via httpx
8. **Formatting** - `format_tool_result_for_llm()` formatta il risultato per il contesto LLM
9. **Response** - Il LLM usa il risultato formattato per comporre la risposta finale

---

## File Coinvolti

| File | Repo | Ruolo |
|------|------|-------|
| `src/components/ui/ToolToggleBar.tsx` | lexe-webchat | UI toggle buttons (9 bottoni) |
| `src/hooks/useStreaming.ts` | lexe-webchat | Invio toggle nel body POST (~riga 499-538) |
| `src/stores/configStore.ts` | lexe-webchat | Zustand store con `enabledTools` state |
| `src/lexe_core/gateway/customer_router.py` | lexe-core | Schema Pydantic + routing + orchestrator forward |
| `src/lexe_core/tools/definitions.py` | lexe-core | OpenAI tool definitions per LLM function calling |
| `src/lexe_core/tools/executor.py` | lexe-core | Executor HTTP + formatter output |

---

## I 9 Strumenti - Scheda Dettagliata

---

### 1. Web Search (`web_search`)

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `webSearch` |
| **Icona** | `Globe` (lucide-react) |
| **Default** | OFF (`false`) |
| **Toggle body key** | `web_search_enabled` |
| **Tool name (LLM)** | `web_search` |
| **Executor** | `_execute_web_search()` |
| **Backend endpoint** | `GET {SEARXNG_URL}/search` (shared-searxng:8080) |
| **Timeout** | 30s (default `execute_tool`) |
| **Retry** | Nessuno (usa httpx diretto, no `_post_with_retry`) |
| **Max risultati** | 5 |

**Parametri LLM:**
```json
{ "query": "string (required)" }
```

**Payload verso SearXNG:**
```json
{ "q": "query", "format": "json", "language": "it" }
```

**Risposta executor (OK):**
```json
{
  "success": true,
  "query": "...",
  "results": [
    { "title": "...", "url": "...", "content": "...(max 500 chars)" }
  ]
}
```

**Output LLM formatter:**
```
- Titolo: contenuto snippet
- Titolo 2: contenuto snippet 2
```

**Quando usarlo:** Ricerche generiche web, notizie recenti, informazioni non legali.

---

### 2. Normattiva (`normattiva_search`)

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `normattiva` |
| **Icona** | `Scale` (lucide-react) |
| **Default** | ON (`true`) |
| **Toggle body key** | `normattiva_enabled` |
| **Tool name (LLM)** | `normattiva_search` |
| **Executor** | `_execute_normattiva()` |
| **Backend endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/normattiva/search` |
| **Timeout** | 30s (default) |
| **Retry** | Nessuno (usa httpx diretto) |

**Parametri LLM:**
```json
{
  "act_type": "string (required) - legge, d.lgs, codice civile, costituzione...",
  "date": "string (optional) - YYYY-MM-DD o YYYY",
  "act_number": "string (optional) - es. '241'",
  "article": "string (optional) - es. '1', '21-bis'"
}
```

**Payload verso lexe-tools:**
```json
{
  "act_type": "codice civile",
  "version": "vigente",
  "date": "1942",
  "act_number": "262",
  "article": "2043"
}
```
Nota: `version: "vigente"` viene aggiunto automaticamente.

**Risposta executor (OK):**
```json
{
  "text": "Art. 2043 - Risarcimento per fatto illecito...",
  "urn": "urn:nir:stato:regio.decreto:1942-03-16;262~art2043",
  "vigente": true
}
```

**Output LLM formatter:**
```
[Normattiva: urn:nir:... - VIGENTE]
Art. 2043 - Risarcimento per fatto illecito. Qualunque fatto doloso o colposo...
```

**Quando usarlo:** Ricerca legislazione italiana specifica (articoli, leggi, decreti). Verifica vigenza.

---

### 3. EUR-Lex (`eurlex_search`)

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `eurlex` |
| **Icona** | `EuroIcon` (SVG custom) |
| **Default** | ON (`true`) |
| **Toggle body key** | `eurlex_enabled` |
| **Tool name (LLM)** | `eurlex_search` |
| **Executor** | `_execute_eurlex()` |
| **Backend endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/eurlex/search` |
| **Timeout** | 30s (default) |
| **Retry** | Nessuno |

**Parametri LLM:**
```json
{
  "act_type": "string (required) - enum: regolamento|direttiva|decisione|trattato|raccomandazione",
  "year": "integer (required) - es. 2016",
  "number": "integer (required) - es. 679",
  "article": "string (optional) - es. '17'"
}
```

**Payload verso lexe-tools:**
```json
{
  "act_type": "regolamento",
  "year": 2016,
  "number": 679,
  "language": "ita",
  "article": "17"
}
```
Nota: `language: "ita"` viene aggiunto automaticamente.

**Risposta executor (OK):**
```json
{
  "text": "Articolo 17 - Diritto alla cancellazione...",
  "title": "Regolamento (UE) 2016/679 (GDPR)",
  "celex": "32016R0679"
}
```

**Output LLM formatter:**
```
[EUR-Lex: 32016R0679 - Regolamento (UE) 2016/679 (GDPR)]
Articolo 17 - Diritto alla cancellazione...
```

**Quando usarlo:** Normativa UE (GDPR, DSA, DMA, direttive, regolamenti).

---

### 4. InfoLex (`infolex_search`)

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `infolex` |
| **Icona** | `Gavel` (lucide-react) |
| **Default** | ON (`true`) |
| **Toggle body key** | `infolex_enabled` |
| **Tool name (LLM)** | `infolex_search` |
| **Executor** | `_execute_infolex()` |
| **Backend endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/infolex/search` |
| **Timeout** | 30s (default) |
| **Retry** | Nessuno |

**Parametri LLM:**
```json
{
  "act_type": "string (required) - codice civile, codice penale, c.p.c., c.p.p...",
  "article": "string (required) - es. '2043', '1321'",
  "include_massime": "boolean (optional, default true)"
}
```

**Payload verso lexe-tools:**
```json
{
  "act_type": "codice civile",
  "article": "2043",
  "include_massime": true,
  "include_relazioni": false,
  "include_footnotes": false
}
```

**Risposta executor (OK):**
```json
{
  "article_text": "Art. 2043 - Risarcimento per fatto illecito...",
  "spiegazione": "L'articolo 2043 disciplina la responsabilita'...",
  "massime": [
    { "autorita": "Cass. Civ., Sez. III", "testo": "La responsabilita'..." }
  ]
}
```

**Output LLM formatter:**
```
Testo articolo:
Art. 2043 - Risarcimento per fatto illecito...

Spiegazione:
L'articolo 2043 disciplina la responsabilita'...

Massime giurisprudenziali:
- Cass. Civ., Sez. III: La responsabilita'... (max 200 chars)
```
Nota: Max 3 massime nel formatter.

**Quando usarlo:** Commento dottrinale e massime giurisprudenziali da Brocardi.it per articoli specifici.

---

### 5. KB Search (`kb_search`) -- NUOVO

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `kbSearch` |
| **Icona** | `BookOpen` (lucide-react) |
| **Default** | ON (`true`) |
| **Toggle body key** | `kb_search_enabled` |
| **Tool name (LLM)** | `kb_search` |
| **Executor** | `_execute_kb_search()` |
| **Backend endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/kb/search` |
| **Timeout** | **20s** (capped, vector search puo' essere lento) |
| **Retry** | 1 retry su 502/503/504 con 1s backoff (via `_post_with_retry`) |
| **Max risultati nel formatter** | 8 |
| **Max chars per snippet** | 1000 |

**Parametri LLM:**
```json
{
  "query": "string (required) - domanda legale in linguaggio naturale",
  "top_k": "integer (optional, default 5, max 20)",
  "materia": "string (optional) - es. 'responsabilita civile', 'contratti', 'lavoro'"
}
```

**Payload verso lexe-tools:**
```json
{
  "query": "risarcimento danni per responsabilita extracontrattuale",
  "top_k": 5,
  "materia": "responsabilita civile"
}
```
Nota: `top_k` viene cappato a max 20 dall'executor. `materia` viene incluso solo se presente.

**Risposta executor (OK) - shape attesa da lexe-tools:**
```json
{
  "results": [
    {
      "riferimento": "Cass. civ., Sez. III, 15/03/2024, n. 12345",
      "id": "massima_12345",
      "testo": "In tema di responsabilita' extracontrattuale...",
      "text": "fallback se 'testo' mancante",
      "materia": "Responsabilita' civile",
      "rrf_score": 0.87,
      "score": 0.87
    }
  ],
  "meta": { "latency_ms": 450, "source": "kb_search", "endpoint": "..." }
}
```
Nota: Il formatter legge sia `riferimento/id`, `testo/text`, `rrf_score/score` (fallback su chiave alternativa).

**Output LLM formatter:**
```
[KB Massimario: 5 risultati]
- [Cass. civ., Sez. III, 15/03/2024, n. 12345] (Responsabilita' civile) score=0.87: In tema di responsabilita' extracontrattuale...
- [Cass. civ., Sez. Unite, ...] ...
```

**Quando usarlo:** Ricerca precedenti giurisprudenziali nel Massimario di Cassazione (38K+ massime). Semantic search con vector embeddings.

**Differenza da InfoLex:** InfoLex cerca commenti per articolo specifico (art. 2043 CC). KB Search cerca nel massimario per concetto ("responsabilita' extracontrattuale danno biologico").

---

### 6. Lex Search (`lex_search`) -- NUOVO

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `lexSearch` |
| **Icona** | `Search` (lucide-react) |
| **Default** | ON (`true`) |
| **Toggle body key** | `lex_search_enabled` |
| **Tool name (LLM)** | `lex_search` |
| **Executor** | `_execute_lex_search()` |
| **Backend endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/lex_search` |
| **Timeout** | **15s** (capped, SearXNG legal) |
| **Retry** | 1 retry su 502/503/504 con 1s backoff (via `_post_with_retry`) |
| **Max risultati nel formatter** | 8 |
| **Max chars per title** | 100 |
| **Max chars per content** | 800 |

**Parametri LLM:**
```json
{
  "query": "string (required) - ricerca legale in linguaggio naturale"
}
```

**Payload verso lexe-tools:**
```json
{
  "query": "ultime sentenze GDPR diritto all'oblio"
}
```

**Risposta executor (OK) - shape attesa:**
```json
{
  "results": [
    {
      "title": "Diritto all'oblio: la Cassazione conferma...",
      "url": "https://altalex.com/documents/...",
      "content": "La Suprema Corte ha confermato che il diritto all'oblio..."
    }
  ],
  "meta": { "latency_ms": 800, "source": "lex_search", "endpoint": "..." }
}
```

**Output LLM formatter:**
```
[Legal Search: 5 risultati]
- Diritto all'oblio: la Cassazione conferma... (https://altalex.com/...): La Suprema Corte ha confermato...
- Titolo 2 (https://brocardi.it/...): Snippet...
```

**Quando usarlo:** Ricerca aperta su siti legali IT/EU (normattiva.it, brocardi.it, altalex.com, dejure.it, eur-lex.europa.eu, cortedicassazione.it). Come Google ma filtrato su fonti legali autorevoli.

**Differenza da Web Search:** `web_search` cerca su tutto il web via SearXNG generico. `lex_search` passa per l'endpoint di lexe-tools-it che usa SearXNG con category/engines filtrati su fonti legali.

---

### 7. Lex Search Enriched (`lex_search_enriched`) -- NUOVO

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `lexSearchEnriched` |
| **Icona** | `Sparkles` (lucide-react) |
| **Default** | **OFF** (`false`) |
| **Badge UI** | "Analisi" (amber badge) |
| **Toggle body key** | `lex_search_enriched_enabled` |
| **Tool name (LLM)** | `lex_search_enriched` |
| **Executor** | `_execute_lex_search_enriched()` |
| **Backend endpoint** | `POST {LEXE_TOOLS_URL}/api/v1/tools/lex_search/enriched` |
| **Timeout** | **30s** (capped, include LLM analysis lato lexe-tools) |
| **Retry** | 1 retry su 502/503/504 con 1s backoff (via `_post_with_retry`) |
| **Max chars analysis** | 3000 |
| **Max risultati nel formatter** | 8 |

**Parametri LLM:**
```json
{
  "query": "string (required) - ricerca legale che richiede analisi approfondita"
}
```

**Payload verso lexe-tools:**
```json
{
  "query": "evoluzione giurisprudenziale responsabilita medica 2020-2025"
}
```

**Risposta executor (OK) - 2 shape possibili:**

Shape A - Analisi stringa:
```json
{
  "analysis": "## Analisi\nLa giurisprudenza sulla responsabilita' medica ha subito...",
  "meta": { "latency_ms": 12000, "source": "lex_search_enriched" }
}
```

Shape B - Risultati strutturati:
```json
{
  "results": [
    {
      "title": "Cass. civ. 2024/1234",
      "url": "https://...",
      "content": "...",
      "vigenza": "vigente"
    }
  ],
  "enriched_results": "fallback per analysis"
}
```

**Output LLM formatter:**

Per Shape A:
```
[Legal Search Enriched]
## Analisi
La giurisprudenza sulla responsabilita' medica ha subito... (max 3000 chars)
```

Per Shape B:
```
[Legal Search Enriched: 5 risultati]
- Cass. civ. 2024/1234 (https://...): contenuto... [vigenza: vigente]
```

**Quando usarlo:** Analisi legali approfondite. Fa la stessa ricerca di `lex_search` ma poi arricchisce con:
- Estrazione automatica riferimenti normativi
- Verifica vigenza via Normattiva
- Analisi strutturata LLM

**ATTENZIONE:** Consuma piu' token (LLM call lato lexe-tools-it). Per questo e' OFF per default e ha il badge "Analisi" in UI.

---

### 8. Memory Retrieval (`use_memory_retrieval`)

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `memory` (in `retrievalSources`) |
| **Icona** | `Brain` (lucide-react) |
| **Default** | ON (`true`) |
| **Toggle body key** | `use_memory_retrieval` |
| **Tipo** | Retrieval Source (non tool LLM) |
| **Gestione** | Diretta nel gateway (`customer_router.py`) |

**Non e' un tool LLM.** Viene gestito nel gateway prima della chiamata LLM:
- Se `use_memory_retrieval=true`, il gateway chiama `retrieve_memory_context()` (lexe-memory)
- Il contesto viene iniettato nel system prompt come `## Memoria dell'utente`
- L'LLM non "chiama" questo tool, lo riceve gia' nel prompt

---

### 9. RAG v3 Retrieval (`use_rag_v3_retrieval`)

| Campo | Valore |
|-------|--------|
| **Frontend ID** | `ragV3` (in `retrievalSources`) |
| **Icona** | `Database` (lucide-react) |
| **Default** | OFF (`false`) |
| **Toggle body key** | `use_rag_v3_retrieval` |
| **Tipo** | Retrieval Source (non tool LLM) |
| **Gestione** | Diretta nel gateway |

**Non e' un tool LLM.** Come Memory, viene gestito prima della chiamata LLM per iniettare contesto RAG conversazionale.

---

## Tabella Riassuntiva Timeout e Retry

| Tool | Timeout | Retry | Retry Backoff | Retry Status Codes |
|------|---------|-------|---------------|--------------------|
| `web_search` | 30s | NO | - | - |
| `normattiva_search` | 30s | NO | - | - |
| `eurlex_search` | 30s | NO | - | - |
| `infolex_search` | 30s | NO | - | - |
| **`kb_search`** | **20s** | **SI (1x)** | **1s** | **502, 503, 504** |
| **`lex_search`** | **15s** | **SI (1x)** | **1s** | **502, 503, 504** |
| **`lex_search_enriched`** | **30s** | **SI (1x)** | **1s** | **502, 503, 504** |

I 3 nuovi tool usano `_post_with_retry()` che:
- Misura latency in ms (campo `meta.latency_ms`)
- Riprova 1 volta su 502/503/504 dopo 1s sleep
- Su timeout httpx, riprova 1 volta dopo 1s
- Su qualsiasi altra eccezione, fail immediato
- Aggiunge `meta: {latency_ms, source, endpoint}` alla risposta

---

## Tabella Riassuntiva Default e Body Keys

| # | UI ID | Body Key | Default | Tipo |
|---|-------|----------|---------|------|
| 1 | `webSearch` | `web_search_enabled` | `false` | Tool LLM |
| 2 | `normattiva` | `normattiva_enabled` | `true` | Tool LLM |
| 3 | `eurlex` | `eurlex_enabled` | `true` | Tool LLM |
| 4 | `infolex` | `infolex_enabled` | `true` | Tool LLM |
| 5 | `kbSearch` | `kb_search_enabled` | `true` | Tool LLM |
| 6 | `lexSearch` | `lex_search_enabled` | `true` | Tool LLM |
| 7 | `lexSearchEnriched` | `lex_search_enriched_enabled` | `false` | Tool LLM |
| 8 | `memory` | `use_memory_retrieval` | `true` | Retrieval Source |
| 9 | `ragV3` | `use_rag_v3_retrieval` | `false` | Retrieval Source |

---

## Error Handling

### Tutti i tool (executor generico)

Se `execute_tool()` catcha un'eccezione non gestita:
```json
{ "error": "messaggio eccezione", "success": false }
```

### format_tool_result_for_llm - Error shaping

Se `result.success == false`:
```
[Tool kb_search failed: Timeout after 20000ms]
```

L'LLM riceve questo messaggio e puo' informare l'utente che il tool non ha risposto.

### Nessun risultato

Ogni tool ha un messaggio specifico per "nessun risultato":
- `[KB Massimario: Nessun risultato]`
- `[Legal Search: Nessun risultato]`
- `[Legal Search Enriched: Nessun risultato]`
- `[Web search: Nessun risultato]`
- `[InfoLex: Nessun risultato]`

---

## Anti-Hallucination: legal_tools_disabled

Se **tutti** i tool legali sono disabilitati (`normattiva + eurlex + infolex + kb_search + lex_search` tutti OFF), viene iniettato un warning nel system prompt:

```
## ATTENZIONE: Strumenti legali disabilitati
Gli strumenti di ricerca normativa sono attualmente DISABILITATI.
NON PUOI accedere a fonti legali per verificare articoli, leggi o normative.
Se l'utente chiede informazioni su articoli di legge specifici:
- Rispondi: "Non ho accesso agli strumenti di ricerca normativa..."
- NON inventare contenuti di articoli o leggi
- NON fornire analisi legali senza fonti verificate
```

Nota: `lex_search_enriched` NON e' nel check perche' e' un enrichment, non una fonte primaria.

---

## Logging

### Backend log compatto (customer_router.py)

```
[CustomerStream] enabled_tools: normattiva=1, eurlex=1, infolex=1, web=0, kb=1, lex=1, lex_enriched=0
```

I valori sono `0`/`1` (int cast di bool) per log compatto e greppabile.

### Executor log (_post_with_retry)

```
[kb_search] Got 503, retrying (attempt 1)
[lex_search] Timeout, retrying (attempt 1)
```

---

## Orchestrator Forward-Compatibility

In `stream_via_orchestrator()`, il payload include tutti i 7 toggle:

```python
payload = {
    ...
    "web_search_enabled": ...,
    "normattiva_enabled": ...,
    "eurlex_enabled": ...,
    "infolex_enabled": ...,
    "kb_search_enabled": ...,
    "lex_search_enabled": ...,
    "lex_search_enriched_enabled": ...,
}
```

Quando l'orchestrator verra' riattivato, avra' gia' i toggle nel payload. Dovra' implementare la logica equivalente di `get_tool_definitions()` + `execute_tool()`.

---

## Backward Compatibility

Tutti i nuovi campi hanno default nel Pydantic schema:
- `kb_search_enabled: bool = True`
- `lex_search_enabled: bool = True`
- `lex_search_enriched_enabled: bool = False`

Un payload vecchio (senza questi campi) funziona identicamente perche' i default sono applicati automaticamente da Pydantic.

---

## Endpoint lexe-tools-it (Referenza)

| Endpoint | Metodo | Usato da |
|----------|--------|----------|
| `/api/v1/tools/normattiva/search` | POST | `normattiva_search` |
| `/api/v1/tools/eurlex/search` | POST | `eurlex_search` |
| `/api/v1/tools/infolex/search` | POST | `infolex_search` |
| `/api/v1/tools/kb/search` | POST | `kb_search` |
| `/api/v1/tools/lex_search` | POST | `lex_search` |
| `/api/v1/tools/lex_search/enriched` | POST | `lex_search_enriched` |

Base URL: `LEXE_TOOLS_URL` env var (default `http://lexe-tools:8021`)

SearXNG per `web_search`: `SEARXNG_URL` env var (default `http://shared-searxng:8080`)

---

## Smoke Test Checklist (Stage)

1. **Network tab**: POST body contiene tutti i 9 toggle
2. **Backend log**: `enabled_tools: normattiva=1, eurlex=1, infolex=1, web=0, kb=1, lex=1, lex_enriched=0`
3. **Toggle ON** → tool appare nella lista `tools[]` inviata al LLM
4. **Toggle OFF** → tool NON appare
5. **kbSearch**: "massime sulla responsabilita' extracontrattuale" → risultati dal massimario
6. **lexSearch**: "ultime sentenze GDPR" → risultati da siti legali
7. **lexSearchEnriched**: domanda complessa → analisi LLM nel risultato
8. **Tutti OFF**: warning anti-hallucination nel system prompt
9. **Badge "Analisi"**: visibile su lexSearchEnriched (amber tag)
10. **Backward compat**: client vecchio (senza nuovi toggle) non rompe

---

*Handoff generato il 2026-02-10*
