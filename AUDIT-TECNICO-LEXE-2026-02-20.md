# AUDIT TECNICO LEXe — 2026-02-20

> Audit completo di modelli, tool, prompt, routing e pipeline della piattaforma LEXe Legal AI.
> Generato su codebase live, non da documentazione.

---

## Indice

- [A. Modelli LiteLLM](#a-modelli-litellm)
- [B. Role Resolution Map](#b-role-resolution-map)
- [C. Tool — Schema, Timeout, Rate Limit](#c-tool--schema-timeout-rate-limit)
- [D. Prompt Templates](#d-prompt-templates)
- [E. Routing Rules](#e-routing-rules)
- [F. Pipeline End-to-End Maps](#f-pipeline-end-to-end-maps)
- [G. Problemi e Rischi](#g-problemi-e-rischi)
- [H. Proposte di Cambiamento](#h-proposte-di-cambiamento)

---

## A. Modelli LiteLLM

**File**: `lexe-infra/litellm/config.yaml`
**Provider**: Tutti via OpenRouter (`openrouter.ai/api/v1`)
**Settings globali**: `drop_params: true`, `set_verbose: false`

### A1. Catalogo modelli

| #   | Alias LiteLLM        | Modello reale                          | Tier        | Context | Tool calling   | Prezzo approx. (input/output per 1M tok) |
| --- | -------------------- | -------------------------------------- | ----------- | ------- | -------------- | ---------------------------------------- |
| 1   | `lexe-fast`          | `google/gemini-2.5-flash`              | Core        | 1M      | Si             | ~$0.15 / $0.60                           |
| 2   | `lexe-primary`       | `mistralai/mistral-large-2512`         | Core        | 128K    | Si             | ~$2.00 / $6.00                           |
| 3   | `lexe-complex`       | `openai/gpt-5.2`                       | Core        | 128K+   | Si (reasoning) | ~$15.00 / $60.00                         |
| 4   | `legal-tier1-gemini` | `google/gemini-2.5-flash`              | T1-Free     | 1M      | Si             | ~$0.15 / $0.60                           |
| 5   | `legal-tier2-haiku`  | `anthropic/claude-haiku-4-5-20251001`  | T2-Low      | 200K    | Si             | ~$0.80 / $4.00                           |
| 6   | `legal-tier2-gemini` | `google/gemini-2.5-flash`              | T2-Low      | 1M      | Si             | ~$0.15 / $0.60                           |
| 7   | `legal-tier3-sonnet` | `anthropic/claude-sonnet-4-6-20250514` | T3-Frontier | 200K    | Si             | ~$3.00 / $15.00                          |
| 8   | `legal-tier3-gpt5`   | `openai/gpt-5.2`                       | T3-Frontier | 128K+   | Si             | ~$15.00 / $60.00                         |
| 9   | `lexe-embedding`     | `openai/text-embedding-3-small`        | Embedding   | 8K      | No             | ~$0.02 / -                               |

### A2. Alias duplicati

| Modello reale             | Alias che puntano allo stesso modello                   |
| ------------------------- | ------------------------------------------------------- |
| `google/gemini-2.5-flash` | `lexe-fast`, `legal-tier1-gemini`, `legal-tier2-gemini` |
| `openai/gpt-5.2`          | `lexe-complex`, `legal-tier3-gpt5`                      |

**Impatto**: 3 modelli unici su 9 alias. I tier ridondanti occupano spazio nel catalog DB ma non causano errori.

### A3. Config JSONB per modello (migration 010)

| Modello             | temperature | max_completion_tokens | Note                             |
| ------------------- | ----------- | --------------------- | -------------------------------- |
| `lexe-primary`      | 0.7         | 4096                  | Chat + synthesis                 |
| `lexe-fast`         | 0.5         | **250**               | Follow-up, facts, classification |
| `lexe-complex`      | 0.1         | 2048                  | Reasoning/planning               |
| `legal-tier2-haiku` | 0.1         | 2048                  | Verification                     |

---

## B. Role Resolution Map

### B1. Catena di risoluzione

```
tenant_model_roles (per-tenant override)
    ↓ fallback
model_role_defaults (global DB seed)
    ↓ fallback
config.py hardcoded default
```

**Query runtime** (`customer_router.py:1080-1116`):

```sql
SELECT d.role,
       COALESCE(o.model_name, d.model_name) AS model_name,
       c.config
FROM core.model_role_defaults d
LEFT JOIN core.tenant_model_roles o
    ON o.tenant_id = $1 AND o.role = d.role
LEFT JOIN core.llm_model_catalog c
    ON c.model_name = COALESCE(o.model_name, d.model_name)
    AND c.deleted_at IS NULL
```

### B2. Ruoli nel DB (8 totali)

| Role                    | Default model       | Seeded in | Usato da                 | Admin override |
| ----------------------- | ------------------- | --------- | ------------------------ | -------------- |
| `chat_primary`          | `lexe-primary`      | mig 009   | ToolLoop (Studio mode)   | Si             |
| `chat_followup`         | `lexe-fast`         | mig 009   | `_generate_follow_ups()` | Si             |
| `fact_extraction`       | `lexe-fast`         | mig 009   | `fact_extractor_llm.py`  | Si             |
| `legis_planner`         | `lexe-complex`      | mig 010   | Pipeline Phase P         | Si             |
| `legis_synthesizer`     | `lexe-primary`      | mig 010   | Pipeline Phase S         | Si             |
| `legis_verifier`        | `legal-tier2-haiku` | mig 010   | Pipeline Phase V         | Si             |
| `legis_intent_detector` | `lexe-fast`         | mig 013   | Pipeline Phase 0         | Si             |
| `legis_planner_light`   | `lexe-fast`         | mig 013   | SIMPLE/STANDARD planner  | Si             |

### B3. Modelli hardcoded in `config.py` (NON nel role system)

| Setting                         | Default              | Env var                           | Usato da                                |
| ------------------------------- | -------------------- | --------------------------------- | --------------------------------------- |
| `lexorc_classifier_model`        | `lexe-fast`          | `LEXE_LEXORC_CLASSIFIER_MODEL`     | `classifier.py` + `_pre_intent_route()` |
| `lexorc_norm_agent_model`        | `lexe-primary`       | `LEXE_LEXORC_NORM_AGENT_MODEL`     | LEXORC NormAgent                         |
| `lexorc_caselaw_agent_model`     | `lexe-primary`       | `LEXE_LEXORC_CASELAW_AGENT_MODEL`  | LEXORC CaseLawAgent                      |
| `lexorc_doctrine_agent_model`    | `lexe-primary`       | `LEXE_LEXORC_DOCTRINE_AGENT_MODEL` | LEXORC DoctrineAgent                     |
| `lexorc_auditor_model`           | `legal-tier2-haiku`  | `LEXE_LEXORC_AUDITOR_MODEL`        | LEXORC Auditor                           |
| `lexorc_synthesizer_model`       | `lexe-primary`       | `LEXE_LEXORC_SYNTHESIZER_MODEL`    | LEXORC Synthesizer                       |
| `legis_researcher_model`        | `lexe-primary`       | `LEXE_LEGIS_RESEARCHER_MODEL`     | Non usato direttamente                  |
| `legis_verification_model`      | `legal-tier2-haiku`  | -                                 | Anti-hallucination check                |
| `legis_verification_high_model` | `legal-tier3-sonnet` | -                                 | High-confidence verification            |

**Problema**: Questi 8+ modelli NON sono nel role system DB, NON sono modificabili dal pannello admin, e NON partecipano alla query di risoluzione `model_role_defaults`.

### B4. Mappa completa utilizzo modelli per pipeline

```
TOOLLOOP (Studio mode):
  Chat → chat_primary (lexe-primary)
  Follow-up → chat_followup (lexe-fast)
  Fact extraction → fact_extraction (lexe-fast)

LEGIS Adaptive:
  Phase 0 (Intent) → legis_intent_detector (lexe-fast)
  Phase P (Plan DIRECT) → nessun LLM
  Phase P (Plan SIMPLE/STANDARD) → legis_planner_light (lexe-fast)
  Phase P (Plan COMPLEX) → legis_planner (lexe-complex / GPT-5.2)
  Phase R (Research) → tool calls only, no LLM
  Phase V (Verify) → legis_verifier (legal-tier2-haiku)
  Phase S (Synthesize) → legis_synthesizer (lexe-primary)

LEXORC (Deep/Strategic):
  Pre-intent classifier → lexorc_classifier_model (lexe-fast) ← HARDCODED
  NormAgent → lexorc_norm_agent_model (lexe-primary) ← HARDCODED
  CaseLawAgent → lexorc_caselaw_agent_model (lexe-primary) ← HARDCODED
  DoctrineAgent → lexorc_doctrine_agent_model (lexe-primary) ← HARDCODED
  Auditor → lexorc_auditor_model (legal-tier2-haiku) ← HARDCODED
  Synthesizer → lexorc_synthesizer_model (lexe-primary) ← HARDCODED

PRE-INTENT ROUTING:
  Classifier → lexorc_classifier_model (lexe-fast) ← HARDCODED
```

---

## C. Tool — Schema, Timeout, Rate Limit

**File definizioni**: `lexe-core/src/lexe_core/tools/definitions.py` (312 righe)
**File executor**: `lexe-core/src/lexe_core/tools/executor.py` (452 righe)
**Backend**: `lexe-tools:8021` (lexe-tools-it)
**SearXNG**: `shared-searxng:8080`

### C1. Tool completi

| #   | Tool                  | Endpoint                                 | Purity         | Timeout       | Retry                 | Default ON | Pipeline        |
| --- | --------------------- | ---------------------------------------- | -------------- | ------------- | --------------------- | ---------- | --------------- |
| 1   | `normattiva_search`   | `POST /api/v1/tools/normattiva/search`   | 3 (OFFICIAL)   | 30s           | No                    | Si         | LEGIS, LEXORC    |
| 2   | `eurlex_search`       | `POST /api/v1/tools/eurlex/search`       | 3 (OFFICIAL)   | 30s           | No                    | Si         | LEGIS, LEXORC    |
| 3   | `infolex_search`      | `POST /api/v1/tools/infolex/search`      | 4 (CERTIFIED)  | 30s           | No                    | Si         | LEGIS, LEXORC    |
| 4   | `web_search`          | `GET shared-searxng:8080/search`         | 5 (WEB)        | 30s           | No                    | **No**     | ToolLoop        |
| 5   | `kb_search`           | `POST /api/v1/tools/kb/search`           | 2 (MASSIMARIO) | **20s** (cap) | 1 retry (502/503/504) | Si         | LEGIS, LEXORC    |
| 6   | `kb_normativa_search` | `POST /api/v1/tools/kb/normativa/search` | 2 (MASSIMARIO) | **20s** (cap) | 1 retry               | Si         | LEGIS, LEXORC    |
| 7   | `lex_search`          | `POST /api/v1/tools/lex_search`          | 4 (CERTIFIED)  | **15s** (cap) | 1 retry               | Si         | LEGIS, LEXORC    |
| 8   | `lex_search_enriched` | `POST /api/v1/tools/lex_search/enriched` | 4 (CERTIFIED)  | 30s           | 1 retry               | **No**     | LEGIS (COMPLEX) |

### C2. Gerarchia purity

```
1 = LEXE_GRAPH     → Solo dati interni graph (non usato come tool)
2 = MASSIMARIO     → kb_search, kb_normativa_search
3 = OFFICIAL       → normattiva_search, eurlex_search
4 = CERTIFIED      → infolex_search, lex_search, lex_search_enriched
5 = WEB            → web_search
```

### C3. Schema parametri per tool

**normattiva_search**:

```json
{
  "act_type": "string (REQUIRED) — legge, d.lgs, codice civile, codice penale, costituzione, etc.",
  "date": "string — YYYY-MM-DD o YYYY",
  "act_number": "string — numero atto",
  "article": "string — numero articolo (es. '1', '21-bis')"
}
```

**eurlex_search**:

```json
{
  "act_type": "enum (REQUIRED) — regolamento|direttiva|decisione|trattato|raccomandazione",
  "year": "integer (REQUIRED) — anno pubblicazione",
  "number": "integer (REQUIRED) — numero atto",
  "article": "string — articolo specifico"
}
```

**infolex_search**:

```json
{
  "act_type": "string (REQUIRED) — codice civile, codice penale, etc.",
  "article": "string (REQUIRED) — numero articolo",
  "include_massime": "boolean (default: true)"
}
```

**kb_search**:

```json
{
  "query": "string (REQUIRED) — testo libero in italiano",
  "top_k": "integer (1-20, default: 5)",
  "materia": "string — civile, penale, lavoro, tributario, etc."
}
```

**kb_normativa_search**:

```json
{
  "query": "string (REQUIRED) — testo libero in italiano",
  "top_k": "integer (1-20, default: 5)",
  "codes": "array[string] — CC, CP, CPC, CPP, COST"
}
```

**lex_search** / **lex_search_enriched**:

```json
{
  "query": "string (REQUIRED) — testo libero"
}
```

**web_search**:

```json
{
  "query": "string (REQUIRED) — testo libero"
}
```

### C4. Tool policy enforcement

**File**: `lexe-core/src/lexe_core/tools/policy.py`

```
resolve_tool_policy(tenant_id, contact_id)
  → contact persona → tenant default persona → fallback (all tools)
  → Returns (allowed_set | None, blocked_set)
  → Applied AFTER frontend preset toggles (intersection)
```

### C5. Tool logging

**File**: `lexe-core/src/lexe_core/tools/logging.py`
**Tabella**: `core.tool_execution_logs` (migration 006)

Logging non-blocking: INSERT asincrono con `success|error|timeout|blocked`, `latency_ms`, `response_size`.

### C6. Disponibilita per scenario

| Tool                | ricerca | contratto_red | contratto_rev | parere | contenzioso | recupero_crediti | custom |
| ------------------- | ------- | ------------- | ------------- | ------ | ----------- | ---------------- | ------ |
| normattiva_search   | Si      | Si            | Si            | Si     | Si          | Si               | Si     |
| eurlex_search       | Si      | No            | No            | Si     | No          | No               | Si     |
| infolex_search      | Si      | Si            | Si            | Si     | Si          | Si               | Si     |
| kb_search           | Si      | No            | Si            | Si     | Si          | Si               | Si     |
| kb_normativa_search | Si      | No            | No            | Si     | Si          | No               | Si     |
| lex_search          | Si      | Si            | Si            | Si     | Si          | Si               | Si     |
| lex_search_enriched | No      | No            | No            | No     | No          | No               | Si     |
| web_search          | No      | No            | No            | No     | No          | No               | No     |

---

## D. Prompt Templates

**File principale**: `lexe-core/src/lexe_core/agent/prompts.py` (608 righe)
**File classifier**: `lexe-core/src/lexe_core/agent/classifier.py` (118 righe)
**File LEXORC synth**: `lexe-core/src/lexe_core/agent/agents/lexorc_synthesizer.py`
**File fact extractor**: `lexe-core/src/lexe_core/agent/fact_extractor_llm.py`
**File customer_router**: `lexe-core/src/lexe_core/gateway/customer_router.py`

### D1. Tabella riepilogativa prompt

| #   | Prompt                               | File:riga                     | Tipo          | Usato da                         |
| --- | ------------------------------------ | ----------------------------- | ------------- | -------------------------------- |
| 1   | `CLASSIFIER_SYSTEM_PROMPT`           | classifier.py:26-44           | System        | Pre-intent route + LEXORC         |
| 2   | `INTENT_DETECTOR_SYSTEM_PROMPT`      | prompts.py:14-67              | System        | LEGIS Phase 0                    |
| 3   | `INTENT_DETECTOR_USER_TEMPLATE`      | prompts.py:69-77              | User template | LEGIS Phase 0                    |
| 4   | `PLANNER_SYSTEM_PROMPT`              | prompts.py:334-450            | System        | LEGIS Phase P                    |
| 5   | `PLANNER_USER_TEMPLATE`              | prompts.py:452-460            | User template | LEGIS Phase P                    |
| 6   | `RESEARCHER_SYSTEM_PROMPT`           | prompts.py:467-482            | System        | LEGIS Phase R                    |
| 7   | `VERIFIER_SYSTEM_PROMPT`             | prompts.py:489-527            | System        | LEGIS Phase V                    |
| 8   | `SYNTHESIZER_SYSTEM_PROMPT`          | prompts.py:534-600            | System        | LEGIS Phase S (STANDARD/COMPLEX) |
| 9   | `SCENARIO_SYNTH_MINIMAL`             | prompts.py:288-304            | System        | LEGIS Phase S (DIRECT)           |
| 10  | `SCENARIO_SYNTH_CONCISE`             | prompts.py:307-327            | System        | LEGIS Phase S (SIMPLE)           |
| 11  | `SCENARIO_SYNTH_CONTRATTO_REDAZIONE` | prompts.py:84-111             | System        | Scenario contratto_redazione     |
| 12  | `SCENARIO_SYNTH_CONTRATTO_REVISIONE` | prompts.py:113-145            | System        | Scenario contratto_revisione     |
| 13  | `SCENARIO_SYNTH_PARERE`              | prompts.py:147-190            | System        | Scenario parere                  |
| 14  | `SCENARIO_SYNTH_CONTENZIOSO`         | prompts.py:192-236            | System        | Scenario contenzioso             |
| 15  | `SCENARIO_SYNTH_RECUPERO_CREDITI`    | prompts.py:238-285            | System        | Scenario recupero_crediti        |
| 16  | `SYNTHESIZER_SYSTEM_PROMPT` (LEXORC)  | lexorc_synthesizer.py:39-73    | System        | LEXORC synthesizer                |
| 17  | `_SYSTEM_PROMPT` (fact extractor)    | fact_extractor_llm.py:35-66   | System        | Fact extraction LLM              |
| 18  | Follow-up generation                 | customer_router.py:~2293-2308 | Inline        | `_generate_follow_ups()`         |
| 19  | Legal tools disabled warning         | customer_router.py:1465-1473  | Addendum      | ToolLoop when tools off          |
| 20  | `_get_default_prompt()`              | customer_router.py:180-205    | Function      | Default Studio persona           |

### D2. Prompt integrali

#### D2.1 — CLASSIFIER_SYSTEM_PROMPT (Pre-intent + LEXORC)

```
Sei un classificatore di complessita per query legali italiane.
Analizza la query dell'utente e classifica la complessita in una delle 4 categorie:

- FLASH: Query semplice, definizione di un concetto, domanda fattuale.
- STANDARD: Query media, 1-2 concetti legali, nessuna ambiguita.
- DEEP: Query complessa, multi-dominio, richiede ricerca approfondita.
- STRATEGIC: Query molto complessa, ambigua, alto impatto, richiede scelta strategica.

Criteri di scoring:
- Numero termini legali specifici (> 5 -> +2)
- Presenza citazioni normative esplicite (-> +1)
- Numero domini legali coinvolti (> 2 -> +2)
- Ambiguita della richiesta (alta -> +3)
- Complessita procedurale (-> +1)

Score: 0-1 = FLASH, 2-3 = STANDARD, 4-6 = DEEP, 7+ = STRATEGIC

Output JSON: {"complexity", "score", "reasoning", "agents_needed", "key_domains"}
```

#### D2.2 — INTENT_DETECTOR_SYSTEM_PROMPT (Phase 0)

Classifica in 2 dimensioni:

- **LEVEL**: DIRECT / SIMPLE / STANDARD / COMPLEX
- **SCENARIO**: ricerca / contratto_redazione / contratto_revisione / parere / contenzioso / recupero_crediti / custom

Output JSON con: level, scenario, intent_summary, legal_domain, jurisdiction, references[], key_concepts[], max_sources, reasoning.

Regole chiave:

- DIRECT: SOLO se cita articoli precisi e chiede testo
- "cosa prevede" / "cosa dice" un articolo -> DIRECT
- analisi/interpretazione -> almeno SIMPLE
- giurisprudenza + normativa -> STANDARD
- multi-dimensionale/comparativo -> COMPLEX
- Dubbio level -> STANDARD, dubbio scenario -> ricerca

#### D2.3 — PLANNER_SYSTEM_PROMPT (Phase P)

Contiene guida dettagliata per 8 tool con:

- Cosa fa, quando usarlo, cosa aspettarsi, args, esempi, limiti
- MAX 8 query prima iterazione
- Proattivo: norme correlate
- 2+ fonti diverse per cross-reference
- `deepening_suggestions`: 1-3 approfondimenti post-risposta

#### D2.4 — VERIFIER_SYSTEM_PROMPT (Phase V)

4 controlli:

1. Coerenza logica
2. Contraddizioni giurisprudenziali
3. Completezza normativa
4. Vigenza

Output: `{status: pass|warn|fail, confidence, checks_run, checks_passed, issues[], summary_it}`

#### D2.5 — SYNTHESIZER_SYSTEM_PROMPT (Phase S — STANDARD/COMPLEX)

7 sezioni obbligatorie:

- **A)** Sintesi esecutiva (5-10 righe, ogni affermazione con [N])
- **B)** Normativa rilevante (link cliccabili Normattiva/EUR-Lex)
- **C)** Giurisprudenza e massime
- **D)** Contraddizioni (da CONFLICTS nell'evidence pack)
- **E)** Evoluzione cronologica (da TIMELINE)
- **F)** Confidence Score
- **G)** Approfondimenti suggeriti (da DEEPENING SUGGESTIONS)

Regola assoluta: ogni affermazione DEVE avere [N] dall'evidence pack.

#### D2.6 — Prompt scenari dedicati

| Scenario                | Struttura output                                                                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **MINIMAL** (DIRECT)    | Solo testo articolo + vigenza + link                                                                                                  |
| **CONCISE** (SIMPLE)    | Sintesi 3-5 righe + normativa + giurisprudenza + confidence                                                                           |
| **CONTRATTO REDAZIONE** | 12 sezioni: premesse, definizioni, oggetto, obblighi, corrispettivo, durata, riservatezza, dati, responsabilita, penale, foro, finali |
| **CONTRATTO REVISIONE** | Playbook italiano (7 posizioni standard) + rating GREEN/YELLOW/RED per clausola                                                       |
| **PARERE**              | IRAC: quesito, normativa, giurisprudenza, dottrina, analisi, conclusioni (CERTO/PROBABILE/INCERTO/CONTRARIO), fonti                   |
| **CONTENZIOSO**         | Fatti, quadro normativo, precedenti (favorevoli+contrari), matrice rischio, strategia (negoziazione/giudizio/ADR), schema atto        |
| **RECUPERO CREDITI**    | Credito, prescrizione, procedura (monitoria/ordinaria/ADR), competenza, costi stimati, bozza atto                                     |

#### D2.7 — LEXORC SYNTHESIZER_SYSTEM_PROMPT

6 sezioni: A) Quadro Giuridico, B) Normativa, C) Giurisprudenza, D) Dottrina, E) Profili Critici, F) Nota di Verifica.
Citazioni in formato `(art. 2043 c.c., Normattiva)` invece di `[N]`.

#### D2.8 — FACT EXTRACTOR (_SYSTEM_PROMPT)

Estrae fatti stabili SOLO dal messaggio USER, ignora completamente le risposte ASSISTANT.
Max 8 fatti per turno, confidence >= 0.55.
Chiavi: identity, professional, location, preferences, etc.

#### D2.9 — FOLLOW-UP GENERATION (inline)

3 domande dal punto di vista dell'utente, 8-15 parole, collegate ai temi della risposta.
Output: JSON array di 3 stringhe.

#### D2.10 — DEFAULT PERSONA (_get_default_prompt)

```
Sei LEXe, assistente AI legale interno dell'ufficio legale di {org}.
- Professionale, preciso, sintetico
- Cita fonti normative complete
- Lista strumenti disponibili
- Suggerisce modalita LEGAL per ricerche approfondite
```

---

## E. Routing Rules

### E1. Flusso principale (`customer_router.py`)

```
Request in
  |
  +-- Orchestrator enabled? (ff_orchestrator_enabled) → stream_via_orchestrator()
  |     (attualmente DISABILITATO)
  |
  +-- LEGIS agent enabled? (ff_legis_agent)
  |     |
  |     +-- _pre_intent_route(message)
  |     |     |
  |     |     +-- msg < 30 char AND no legal hints → "toolloop"
  |     |     |
  |     |     +-- classify_complexity(query) → FLASH/STANDARD/DEEP/STRATEGIC
  |     |           |
  |     |           +-- FLASH + no legal domains → "toolloop"
  |     |           +-- FLASH/STANDARD → "legis"
  |     |           +-- DEEP/STRATEGIC → "lexorc"
  |     |
  |     +-- route == "lexorc" AND lexorc enabled for tenant?
  |     |     YES → LexeOrchestrator.run() → SSE stream
  |     |     FAIL → fallback to "legis"
  |     |
  |     +-- route == "legis"?
  |     |     YES → legis_agent_pipeline() → SSE stream
  |     |     FAIL → fallback to ToolLoop
  |     |
  |     +-- route == "toolloop"?
  |           YES → ToolLoop (LiteLLM direct + tool calling)
  |
  +-- ff_legis_agent OFF → ToolLoop directly
```

### E2. Legal hints (fast-path heuristic)

~70 keyword stems in `_LEGAL_HINTS`:

```
art, articolo, legge, decreto, codice, contratto, causa, norma, responsabil,
risarcimento, danno, parere, giurisp, cassazione, tribun, prescrizione,
inadempimento, recesso, locazione, fideiussione, ipoteca, obbligaz, illecit,
penale, civile, processual, ingiuntivo, esecuzione, fallimento, concorso,
socie, lavoro, licenziamento, gdpr, privacy, appalto, subappalto, mediazione,
arbitrato, impugnazione, appello, ricorso, sentenza, ordinanza, atto,
citazione, notifica, termine, clausola, vessatori, indennizzo, garanzia,
caparra, mora, interessi, credito, debitore, creditore, successione, eredit,
testamento, donazione, usucapione, servitu, propriet, possesso, condominio,
divorzio, separazione, affidamento, mantenimento, adozione, reato, dolo,
colpa, aggravante, attenuante, querela, denuncia, decadenza, redigi,
analizza, revisiona, bozza, nda, compliance
```

### E3. Pre-intent classifier mapping

| Complexity | Legal domains? | Route              |
| ---------- | -------------- | ------------------ |
| FLASH      | No             | `toolloop`         |
| FLASH      | Si             | `legis`            |
| STANDARD   | *              | `legis`            |
| DEEP       | *              | `lexorc`            |
| STRATEGIC  | *              | `lexorc`            |
| Error      | *              | `legis` (fallback) |

### E4. LEGIS Pipeline Level routing

| Level    | Planner                          | Graph | Verifier             | Synthesizer      |
| -------- | -------------------------------- | ----- | -------------------- | ---------------- |
| DIRECT   | Nessun LLM (`build_direct_plan`) | Skip  | Skip                 | MINIMAL          |
| SIMPLE   | `lexe-fast`                      | Skip  | Skip                 | CONCISE          |
| STANDARD | `lexe-fast`                      | Si    | Si (LLM)             | FULL (7 sezioni) |
| COMPLEX  | `lexe-complex` (reasoning)       | Si    | Si (LLM + coherence) | FULL (7 sezioni) |

### E5. Scenario routing

| Scenario              | Plan builder                  | Synth prompt          | Verifier    | Graph       |
| --------------------- | ----------------------------- | --------------------- | ----------- | ----------- |
| `ricerca`             | Level-based                   | Level-based           | Level-based | Level-based |
| `contratto_redazione` | `_plan_contratto_redazione()` | 12 clausole           | Si          | **No**      |
| `contratto_revisione` | `_plan_contratto_revisione()` | Playbook rating       | Si          | **No**      |
| `parere`              | `_plan_parere()`              | IRAC                  | Si          | Si          |
| `contenzioso`         | `_plan_contenzioso()`         | Strategia processuale | Si          | Si          |
| `recupero_crediti`    | `_plan_recupero_crediti()`    | Procedurale           | Si          | Si          |
| `custom`              | Level-based                   | Level-based           | Si          | Si          |

---

## F. Pipeline End-to-End Maps

### F1. ToolLoop (Studio mode)

```
User message
  → resolve persona prompt (DB + cache 5min)
  → resolve memory context (v1/v2)
  → resolve tool definitions (preset-based)
  → apply tool policy (allowed/blocked)
  → LiteLLM streaming (chat_primary model)
     → if tool_calls: execute_tool() → follow-up LLM call
  → save assistant message
  → _generate_follow_ups() (chat_followup model)
  → _extract_facts_if_enabled() (fact_extraction model)
  → SSE done event
```

**LLM calls**: 1-2 (chat) + 1 (follow-up) + 0-1 (facts) = 2-4 calls

### F2. LEGIS DIRECT

```
User: "dammi l'art. 2043 c.c."
  → Phase 0: Intent detector (lexe-fast, ~500ms)
     → level=DIRECT, scenario=ricerca
  → Phase P: build_direct_plan() (0ms, no LLM)
     → 2 queries: normattiva_search + infolex_search
  → Phase R: execute tools (~2-5s)
  → Phase V: SKIP (structural pass only)
  → Phase S: MINIMAL synth (lexe-primary, ~2s)
  → Total: ~5-8s, 2 LLM calls, 2 tool calls
```

### F3. LEGIS STANDARD

```
User: "responsabilita medica per errore diagnostico"
  → Phase 0: Intent detector (lexe-fast, ~800ms)
     → level=STANDARD, scenario=ricerca
  → Phase P: run_planner(lexe-fast, ~2s)
     → 5-6 queries planned
  → Phase R: execute tools (~5-10s)
     + graph enrichment (relationships + timeline)
  → Phase V: run_verifier(legal-tier2-haiku, ~3s)
  → Phase S: FULL synth 7 sezioni (lexe-primary, ~8-15s)
  → Total: ~15-30s, 4 LLM calls, 5-6 tool calls
```

### F4. LEGIS Scenario (parere)

```
User: "parere sulla responsabilita del medico"
  → Phase 0: Intent detector → level=STANDARD, scenario=parere
  → Phase P: build_scenario_plan(parere) (0ms, template)
     → normattiva + infolex + kb_search + lex_search
  → Phase R: execute tools (~5-10s)
     + graph enrichment
  → Phase V: run_verifier (~3s)
  → Phase S: PARERE synth IRAC (lexe-primary, ~10-15s)
  → Total: ~20-30s, 3 LLM calls, 4-6 tool calls
```

### F5. LEXORC (Deep/Strategic)

```
User: "analisi completa risarcimento danni da perdita di chance sanitario"
  → Pre-intent: classify_complexity() → DEEP
  → LexeOrchestrator 6 phases:
     1. Classify (lexe-fast)
     2. WorkPlan (lexe-complex)
     3. ParallelEngine (3 agents in asyncio.gather):
        - NormAgent (lexe-primary, normattiva+eurlex)
        - CaseLawAgent (lexe-primary, kb_search+infolex)
        - DoctrineAgent (lexe-primary, lex_search+infolex)
     4. Audit L1 (structural) + L2 (legal-tier2-haiku)
     5. Synthesize (lexe-primary, A-F sections)
     6. SSE streaming
  → Total: ~30-60s, 7+ LLM calls, 8-15 tool calls
```

---

## G. Problemi e Rischi

### P1 — CRITICAL: Double Classifier

**Dove**: `_pre_intent_route()` (customer_router.py:954) + `run_intent_detector()` (pipeline.py:114)

Il flusso per query LEGIS fa **2 classificazioni LLM sequenziali**:

1. Pre-intent: `classify_complexity()` → FLASH/STANDARD/DEEP/STRATEGIC (~1.5s)
2. Intent detector: `run_intent_detector()` → DIRECT/SIMPLE/STANDARD/COMPLEX + scenario (~1s)

**Impatto**: +1.5-2.5s latenza, costo raddoppiato per ogni query legale.

**Root cause**: Pre-intent classifier (Sprint 8 auto-routing) e intent detector (Sprint 8 adaptive pipeline) fanno overlap funzionale.

### P2 — MEDIUM: Modelli LEXORC non nel role system

8 modelli LEXORC sono hardcoded in `config.py`, non nel `model_role_defaults` DB, non modificabili dal pannello admin.

**Impatto**: L'admin non puo cambiare modello per agenti LEXORC senza variabile d'ambiente + restart container.

### P3 — LOW: Alias duplicati LiteLLM

3 alias (`legal-tier1-gemini`, `legal-tier2-gemini`) puntano a `gemini-2.5-flash`, gia coperto da `lexe-fast`.
`legal-tier3-gpt5` punta a `gpt-5.2`, gia coperto da `lexe-complex`.

**Impatto**: Confusione nel catalog, nessun errore funzionale.

### P4 — MEDIUM: Nessun rate limiting tool executor

`executor.py` non ha rate limiting. Un planner con max 8 query puo generare 8 chiamate simultanee a `lexe-tools:8021`.

**Impatto**: Sotto carico, rischio di saturare lexe-tools o SearXNG.

### P5 — LOW: Studio prompt obsoleto

`_get_default_prompt()` suggerisce "passare in modalita LEGAL usando il selettore nel pannello laterale" — ma con auto-routing il selettore non esiste piu (default mode e "legal").

**Impatto**: UX confusa, nessun errore tecnico.

### P6 — MEDIUM: max_completion_tokens=250 per lexe-fast

Il seed nella migration 010 imposta `max_completion_tokens: 250` per `lexe-fast`.
Questo modello e usato per intent detection (output JSON ~200-500 token) e planner light (output JSON ~500-1000 token).

**Impatto**: Possibile troncamento JSON → parse error → fallback a STANDARD + RICERCA.

### P7 — LOW: No retry per normattiva/eurlex/infolex

`normattiva_search`, `eurlex_search`, `infolex_search` usano `httpx.AsyncClient` diretto senza `_post_with_retry()`.
Solo `kb_search`, `kb_normativa_search`, `lex_search`, `lex_search_enriched` hanno retry.

**Impatto**: Un 502 transitorio su Normattiva causa fallimento tool senza retry.

### P8 — INFO: Costo doppio routing

P1 implica 2 LLM call per ogni query legale:

- Pre-intent classifier: ~$0.0001 (lexe-fast, ~200 input + ~50 output tokens)
- Intent detector: ~$0.0001 (lexe-fast, ~500 input + ~200 output tokens)

Su 1000 query/giorno: ~$0.20 extra/giorno = ~$6/mese. Non critico ma evitabile.

---

## H. Proposte di Cambiamento

### H-A: Unificare Classifier (elimina P1 + P8)

**Proposta**: Eliminare `_pre_intent_route()` e spostare la decisione toolloop/legis/lexorc nell'intent detector.

**Come**: Aggiungere un campo `route: "toolloop"|"legis"|"lexorc"` all'output di `run_intent_detector()`. Se `level=DIRECT` e il messaggio non ha legal hints, `route=toolloop`. Se `level=COMPLEX`, `route=lexorc`.

**Tradeoff**: Aumenta latenza del fast-path (anche "ciao" passa per intent detector) vs. elimina 1 LLM call per ogni query legale.

**Mitigazione**: Mantenere l'heuristic `_LEGAL_HINTS` come pre-filter (0ms) prima dell'intent detector.

### H-B: LEXORC roles nel DB (elimina P2)

**Proposta**: Aggiungere 6 nuovi role defaults:

```sql
INSERT INTO core.model_role_defaults (role, model_name, description) VALUES
  ('lexorc_classifier', 'lexe-fast', 'LEXORC complexity classifier'),
  ('lexorc_norm_agent', 'lexe-primary', 'LEXORC NormAgent'),
  ('lexorc_caselaw_agent', 'lexe-primary', 'LEXORC CaseLawAgent'),
  ('lexorc_doctrine_agent', 'lexe-primary', 'LEXORC DoctrineAgent'),
  ('lexorc_auditor', 'legal-tier2-haiku', 'LEXORC Auditor'),
  ('lexorc_synthesizer', 'lexe-primary', 'LEXORC Synthesizer');
```

**Tradeoff**: +6 righe nella query di role resolution (negligibile). Full admin control su tutti i modelli.

### H-C: Cleanup alias duplicati (elimina P3)

**Proposta**: Rimuovere `legal-tier1-gemini`, `legal-tier2-gemini`, `legal-tier3-gpt5` da `config.yaml`.
Aggiornare eventuali riferimenti nel catalog DB per usare `lexe-fast` e `lexe-complex`.

**Tradeoff**: Breaking change se qualche tenant ha overrides che puntano a questi alias.

**Mitigazione**: Verificare `tenant_model_roles` prima della rimozione.

### H-D: Rate limiting tool executor (elimina P4)

**Proposta**: `asyncio.Semaphore(4)` globale in `executor.py` per limitare chiamate concorrenti a lexe-tools.

**Tradeoff**: Potenziale rallentamento pipeline se >4 tool call simultanee.

### H-E: Fix max_completion_tokens lexe-fast (elimina P6)

**Proposta**: Aggiornare il seed config per `lexe-fast`:

```sql
UPDATE core.llm_model_catalog SET config = '{"temperature": 0.5, "max_completion_tokens": 1024}'
WHERE model_name = 'lexe-fast';
```

**Tradeoff**: Nessuno. 250 e insufficiente per JSON strutturato.

### H-F: Retry per normattiva/eurlex/infolex (elimina P7)

**Proposta**: Refactor `_execute_normattiva()`, `_execute_eurlex()`, `_execute_infolex()` per usare `_post_with_retry()`.

**Tradeoff**: +1s latenza nel worst case (retry su timeout). Consistenza con gli altri tool.

---

## Appendice — Feature Flags

| Flag                       | Env var                         | Default | Effetto                                    |
| -------------------------- | ------------------------------- | ------- | ------------------------------------------ |
| `ff_legis_agent`           | `LEXE_FF_LEGIS_AGENT`           | `false` | Abilita pipeline LEGIS + auto-routing      |
| `ff_legis_allow_bypass`    | `LEXE_FF_LEGIS_ALLOW_BYPASS`    | `false` | Permette all'utente di disabilitare LEGIS  |
| `ff_lexorc_enabled`         | `LEXE_FF_LEXORC_ENABLED`         | `false` | Abilita LEXORC globalmente                  |
| `ff_lexorc_tenants`         | `LEXE_FF_LEXORC_TENANTS`         | `""`    | UUIDs comma-separated per LEXORC per-tenant |
| `ff_orchestrator_enabled`  | `LEXE_FF_ORCHESTRATOR_ENABLED`  | `false` | Abilita orchestrator (DISABILITATO)        |
| `ff_memory_v2`             | `LEXE_FF_MEMORY_V2`             | `false` | Memory v2 con intent-aware retrieval       |
| `ff_memory_profile_evolve` | `LEXE_FF_MEMORY_PROFILE_EVOLVE` | `false` | Profile evolution con LLM summaries        |
| `ff_facts_llm`             | `LEXE_FF_FACTS_LLM`             | `false` | Estrazione fatti via LLM                   |

## Appendice — Timeout completi

| Componente          | Setting                     | Default   | Env var                          |
| ------------------- | --------------------------- | --------- | -------------------------------- |
| Intent detector     | `legis_intent_timeout`      | 5.0s      | `LEXE_LEGIS_INTENT_TIMEOUT`      |
| Planner             | `legis_planner_timeout`     | 60.0s     | `LEXE_LEGIS_PLANNER_TIMEOUT`     |
| Researcher          | `legis_researcher_timeout`  | 120.0s    | `LEXE_LEGIS_RESEARCHER_TIMEOUT`  |
| Verifier            | `legis_verifier_timeout`    | 30.0s     | `LEXE_LEGIS_VERIFIER_TIMEOUT`    |
| Synthesizer         | `legis_synthesizer_timeout` | 120.0s    | `LEXE_LEGIS_SYNTHESIZER_TIMEOUT` |
| LEXORC classifier    | `lexorc_classifier_timeout`  | 5.0s      | `LEXE_LEXORC_CLASSIFIER_TIMEOUT`  |
| LEXORC parallel      | `lexorc_parallel_timeout`    | 30.0s     | `LEXE_LEXORC_PARALLEL_TIMEOUT`    |
| LEXORC agent         | `lexorc_agent_timeout`       | 20.0s     | `LEXE_LEXORC_AGENT_TIMEOUT`       |
| LEXORC auditor       | `lexorc_auditor_timeout`     | 10.0s     | `LEXE_LEXORC_AUDITOR_TIMEOUT`     |
| LEXORC synthesizer   | `lexorc_synthesizer_timeout` | 120.0s    | `LEXE_LEXORC_SYNTHESIZER_TIMEOUT` |
| Memory context      | `memory_context_timeout`    | 3.0s      | `LEXE_MEMORY_CONTEXT_TIMEOUT`    |
| Memory delta        | `memory_delta_timeout`      | 5.0s      | `LEXE_MEMORY_DELTA_TIMEOUT`      |
| Facts LLM           | `facts_llm_timeout`         | 2.5s      | `LEXE_FACTS_LLM_TIMEOUT`         |
| kb_search           | executor hardcoded          | 20.0s cap | -                                |
| kb_normativa_search | executor hardcoded          | 20.0s cap | -                                |
| lex_search          | executor hardcoded          | 15.0s cap | -                                |
| lex_search_enriched | executor hardcoded          | 30.0s cap | -                                |

## Appendice — Migrations DB

| #   | File                                     | Contenuto                                               |
| --- | ---------------------------------------- | ------------------------------------------------------- |
| 001 | `001_admin_panel.sql`                    | RLS, tenant_id su conversation_messages                 |
| 002 | `002_rbac_tables.sql`                    | Tabelle RBAC                                            |
| 003 | `003_personas_rls.sql`                   | RLS su personas                                         |
| 004 | `004_persona_indexes.sql`                | Indici personas                                         |
| 005 | `005_personas_permissions.sql`           | Permessi personas                                       |
| 006 | `006_tool_logging_litellm_isolation.sql` | tool_execution_logs + 3 colonne LiteLLM su tenants      |
| 007 | `007_global_catalogs.sql`                | llm_model_catalog, tool_catalog, persona_catalog        |
| 008 | `008_legis_agent.sql`                    | Tabelle LEGIS agent                                     |
| 009 | `009_model_roles.sql`                    | model_role_defaults + tenant_model_roles (3 ruoli chat) |
| 010 | `010_model_config.sql`                   | config JSONB su catalog + 3 ruoli LEGIS                 |
| 011 | `011_conversation_mode.sql`              | Modalita conversazione                                  |
| 012 | `012_lexorc_foundation.sql`               | lexorc_sessions                                          |
| 013 | `013_intent_detector.sql`                | legis_usage_patterns + 2 ruoli Sprint 8                 |

---

*Generato il 2026-02-20 da audit automatico su codebase LEXe Genesis.*
