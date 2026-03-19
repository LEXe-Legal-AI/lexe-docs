# LEGIS AGENT — Piano di Implementazione Definitivo

## Context

LEXe ha bisogno di un **agente specializzato per la ricerca legale** che superi il tool loop standard (max 5 iterazioni, 15 tool calls). L'avvocato oggi interagisce con la chat e i tool vengono invocati dal LLM "quando gli va", spesso in modo subottimale. Il LEGIS AGENT è un orchestratore multi-tool dedicato che:

- **Capisce** cosa vuole l'avvocato (analisi preliminare)
- **Cerca prolificamente** su tutte le fonti legali disponibili
- **Verifica** l'esistenza e la vigenza di ogni citazione
- **Produce** un "evidence pack" strutturato con score di affidabilità
- **Sintetizza** una risposta finale con citazioni verificabili

**Problema attuale**: Il tool loop standard è "reattivo" — il LLM decide se e quando chiamare un tool. Un avvocato che chiede "presupposti per il risarcimento ex art. 2043 c.c." potrebbe ricevere una risposta con citazioni inventate perché il LLM ha deciso di non usare Normattiva. Il LEGIS AGENT è "proattivo" — cerca SEMPRE su tutte le fonti pertinenti.

**Filosofia**: "Meglio sperperare token all'inizio ma avere risultati eccellenti" — la qualità delle fonti è prioritaria rispetto al costo. Ogni tool va testato preventivamente e migliorato se mostra anomalie.

---

## Architettura ad Alto Livello

```
┌─────────────────────────────────────────────────────────────────┐
│ customer_router.py                                              │
│   if request.legis_mode:                                        │
│     yield from legis_agent_pipeline(...)    ← NUOVO PERCORSO    │
│   else:                                                         │
│     yield from standard_tool_loop(...)      ← INVARIATO         │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ lexe_core/agent/pipeline.py   (PRVS State Machine)              │
│                                                                 │
│  ┌──────────┐   ┌────────────┐   ┌──────────┐   ┌───────────┐ │
│  │ PLANNER  │──▶│ RESEARCHER │──▶│ VERIFIER │──▶│SYNTHESIZER│ │
│  │ (LLM #1) │   │ (Tool Loop)│   │ (LLM #2) │   │ (LLM #3)  │ │
│  └──────────┘   └────────────┘   └──────────┘   └───────────┘ │
│       │               │               │               │        │
│  SSE: plan       SSE: tool_*     SSE: verify      SSE: token   │
│  SSE: confirm    SSE: evidence   SSE: verdict     SSE: done    │
└─────────────────────────────────────────────────────────────────┘
```

**Dove vive**: Nuovo package `lexe_core/agent/` dentro lexe-core. NON un servizio separato.
**Attivazione**: Feature flag `ff_legis_agent` + campo `legis_mode: true` nel request body.
**Interazione con tool loop esistente**: Percorso parallelo, non sostituzione. Il tool loop standard resta invariato.

---

## Scelta Modelli per Fase

> Principio: qualità massima dove serve (reasoning, synthesis), costo minimo dove basta (extraction, classification). Il budget è "generoso all'inizio" per garantire risultati eccellenti.

### Tabella Modelli per Funzione

| Fase                      | Funzione                                             | Modello Raccomandato               | Alias LEXe          | Perché                                                          | Costo stimato/call                | Fallback          |
| ------------------------- | ---------------------------------------------------- | ---------------------------------- | ------------------- | --------------------------------------------------------------- | --------------------------------- | ----------------- |
| **P — Planner**           | Analisi query, decomposizione in research plan       | `openai/gpt-4.1-mini`              | `lexe-complex`      | Reasoning forte, JSON strutturato, costo contenuto              | ~$0.003 (2K in + 1K out)          | `lexe-primary`    |
| **R — Researcher**        | Tool calling loop (chiama normattiva, eurlex, kb...) | `mistralai/mistral-large-2411`     | `lexe-primary`      | Unico alias con function calling testato in prod                | ~$0.010 (4K in + 2K out) × N iter | Nessuno (critico) |
| **V — Verifier**          | Cross-reference, coerenza logica, scoring            | `anthropic/claude-3.5-haiku`       | `legal-tier2-haiku` | Eccellente in analisi strutturata, supporta tool_use, economico | ~$0.004 (3K in + 1K out)          | `lexe-complex`    |
| **S — Synthesizer**       | Redazione risposta finale con citazioni              | `mistralai/mistral-large-2411`     | `lexe-primary`      | Streaming, qualità narrativa, tool refs inline                  | ~$0.015 (8K in + 4K out)          | Nessuno (critico) |
| **Citation Extractor**    | Estrazione citazioni da testo                        | Regex (NO LLM)                     | —                   | Deterministico, zero costo, zero latenza                        | $0.000                            | —                 |
| **Structural Check**      | Validazione formato URN, CELEX, articoli             | Regex (NO LLM)                     | —                   | Deterministico                                                  | $0.000                            | —                 |
| **Vigenza Recheck**       | Ri-verifica vigenza citazioni                        | Tool call (vigenza_fast)           | —                   | Cache L1/L2, <10ms se cached                                    | $0.000                            | normattiva_search |
| **Follow-up Suggestions** | Domande di follow-up                                 | `google/gemini-2.0-flash-lite-001` | `lexe-fast`         | Invariato dal sistema attuale                                   | ~$0.000 (free tier)               | —                 |
| **Embedding**             | Vettorizzazione query per kb_search                  | `openai/text-embedding-3-small`    | `lexe-embedding`    | Invariato, coerente con vettori in DB                           | ~$0.00002                         | —                 |

### Costo Stimato per Request LEGIS AGENT

| Componente                                   | Token stimati   | Costo           |
| -------------------------------------------- | --------------- | --------------- |
| Planner (1 call)                             | ~3K tokens      | $0.003          |
| Researcher (3-5 iterazioni × 2-4 tool calls) | ~20K tokens     | $0.030          |
| Verifier (1 call)                            | ~4K tokens      | $0.004          |
| Synthesizer (streaming, 8K+ output)          | ~12K tokens     | $0.015          |
| Tool calls (Normattiva API, SearXNG, etc.)   | —               | $0.000          |
| **Totale per request**                       | **~39K tokens** | **~$0.05-0.08** |

Confronto: Una request chat standard costa ~$0.01-0.02. Il LEGIS AGENT costa 3-5x ma produce output con citazioni verificate.

### Modello per la Verifica Anti-Allucinazione (opzionale)

| Trigger                                          | Modello                              | Perché                                          |
| ------------------------------------------------ | ------------------------------------ | ----------------------------------------------- |
| Caso standard (3+ citazioni)                     | `legal-tier2-haiku` (Claude Haiku)   | Veloce, preciso per JSON strutturato            |
| Caso complesso (penale, termini, contraddizioni) | `legal-tier3-sonnet` (Claude Sonnet) | Reasoning superiore per coerenza logica         |
| Campionamento qualità (1% traffico)              | `legal-tier3-gpt4o` (GPT-4o)         | Second opinion indipendente da diverso provider |

### Configurazione in `config.py`

```python
# LEGIS AGENT Model Configuration
legis_planner_model: str = "lexe-complex"       # LEXE_LEGIS_PLANNER_MODEL
legis_researcher_model: str = "lexe-primary"     # LEXE_LEGIS_RESEARCHER_MODEL
legis_verifier_model: str = "legal-tier2-haiku"  # LEXE_LEGIS_VERIFIER_MODEL
legis_synthesizer_model: str = "lexe-primary"    # LEXE_LEGIS_SYNTHESIZER_MODEL
legis_verification_model: str = "legal-tier2-haiku"  # Anti-hallucination check
legis_verification_high_model: str = "legal-tier3-sonnet"  # Complex verification
```

Ogni modello è configurabile via env var per tenant, permettendo A/B testing e upgrade senza deploy.

---

## Pipeline PRVS — Dettaglio per Fase

### Fase P — PLANNER (30s timeout)

**Input**: Messaggio utente + conversazione recente (max 10 messaggi)
**Modello**: `lexe-complex` (gpt-4.1-mini), temp 0.1, max 2K tokens
**Output**: `ResearchPlan` (JSON strutturato)

```python
@dataclass
class ResearchPlan:
    query_understood: str           # Riepilogo di cosa ha capito
    legal_domain: str               # "diritto civile", "diritto penale", etc.
    temporal_scope: str | None      # "vigente" o "vigente al 2020-03-15"
    jurisdiction: str               # "IT", "EU", "IT+EU"
    key_norms: list[str]            # Norme candidate ["art. 2043 c.c.", "art. 2059 c.c."]
    key_concepts: list[str]         # Concetti chiave ["responsabilità extracontrattuale"]
    sources_to_query: list[SourceQuery]  # Tool calls pianificati
    disambiguation_needed: list[str] | None  # Domande minime se ambiguo
```

```python
@dataclass
class SourceQuery:
    tool: str                       # "normattiva_search", "kb_search", etc.
    arguments: dict                 # Args esatti per il tool
    reason: str                     # Perché questa query
    priority: int                   # 1=parallelo, 2=dopo i risultati di priority 1
    depends_on: list[str] | None    # ID di query prerequisite
```

**Prompt chiave**: Il Planner NON fa ricerca, solo pianificazione. Riceve la lista dei tool disponibili e le loro capabilities, e produce un piano di ricerca ottimale.

**SSE Events emessi**: `agent_phase("planning")`, `agent_plan(plan)`, opzionalmente `legis_confirm(plan)` se serve conferma utente.

### Fase R — RESEARCHER (120s timeout)

**Modello**: `lexe-primary` (Mistral Large), temp 0.3, max 4K tokens per iterazione
**Tool calling**: Sì — function calling nativo su Mistral Large
**Max iterazioni**: 10 (vs 5 del tool loop standard)
**Max tool calls totali**: 30 (vs 15 del standard)

#### Limiti Hard Anti-Runaway per Tool

| Tool | Max calls per request | Razionale |
|------|----------------------|-----------|
| `normattiva_search` | 8 | Max 8 articoli diversi in una singola ricerca |
| `eurlex_search` | 6 | Max 6 atti UE in parallelo |
| `infolex_search` | 6 | Brocardi rate limit implicito |
| `kb_search` | 8 | Max 8 query semantiche diverse |
| `lex_search` | 6 | SearXNG ha latenza variabile |
| `lex_search_enriched` | 4 | Include LLM call, costoso |
| `vigenza_fast` | 15 | Quasi gratis (cache), serve per tutte le norme |
| `one_search` | 3 | Fan-out pesante, max 3 query diverse |

**Comportamento al superamento**: Stop immediato del tool, chiusura con evidence pack parziale, status `WARN`, nota `"Limite chiamate raggiunto per {tool_name}, risultati parziali"`. MAI inventare dati per compensare.

**Implementazione**: Contatore `dict[str, int]` in `researcher.py`, check prima di ogni `execute_tool()`.

```python
PER_TOOL_LIMITS = {
    "normattiva_search": 8, "eurlex_search": 6, "infolex_search": 6,
    "kb_search": 8, "lex_search": 6, "lex_search_enriched": 4,
    "vigenza_fast": 15, "one_search": 3,
}
```

**Funzionamento**:

1. Riceve il `ResearchPlan` dal Planner
2. Esegue le `SourceQuery` di priority 1 in **parallelo** via `asyncio.gather()`
3. Raccoglie i risultati, li normalizza nell'Evidence Pack
4. Se il piano prevedeva query di priority 2 (dipendenti), le esegue ora con i risultati come contesto
5. Il LLM può aggiungere tool calls extra se i risultati suggeriscono norme correlate
6. **Deduplicazione cross-fonte**: Prima di aggiungere all'evidence pack, dedup per URN (norme IT), CELEX (norme EU), e `(numero, anno, sezione)` per giurisprudenza Cassazione. Se duplicato, merge dei campi (migliore snippet, link più autorevole) in un unico EvidenceItem

**Tool calls disponibili** (tutti quelli esistenti + vigenza_fast):

- `normattiva_search` — legislazione italiana (max 8)
- `eurlex_search` — legislazione UE (max 6)
- `infolex_search` — massime e commento da Brocardi (max 6)
- `kb_search` — Massimario Cassazione (38K+ massime, ricerca ibrida) (max 8)
- `lex_search` — ricerca web su 13 siti legal approvati (max 6)
- `vigenza_fast` — verifica vigenza con cache L1/L2/L3/L4 (max 15)
- `one_search` — fan-out parallelo multi-fonte (per query esplorative) (max 3)

**SSE Events emessi**: `tool_call`, `tool_result`, `tool_end` (riuso eventi esistenti), più `legis_evidence(item)` per ogni risultato normalizzato.

### Fase V — VERIFIER (30s timeout)

**Modello**: `legal-tier2-haiku` (Claude Haiku), temp 0.1, max 2K tokens
**Input**: Evidence Pack completo + tool results grezzi
**Output**: `VerificationVerdict`

**Controlli eseguiti**:

1. **Esistenza citazioni** (tool call): Per ogni norma nell'evidence pack, ri-chiama `normattiva_search`/`eurlex_search` con parametri esatti per conferma
2. **Vigenza** (tool call cached): Chiama `vigenza_fast` per ogni norma — quasi gratis grazie alla cache L1/L2
3. **Coerenza cross-riferimento** (LLM): Il LLM verifica che le conclusioni seguano logicamente dalle fonti
4. **Validazione strutturale** (regex, no LLM): URL formato, CELEX formato, range articoli plausibili
5. **Contraddizioni** (LLM): Identifica orientamenti giurisprudenziali contrastanti

**Output**:

```python
@dataclass
class VerificationVerdict:
    status: Literal["pass", "warn", "fail"]
    confidence: float               # 0.0-1.0
    checks_run: int
    checks_passed: int
    issues: list[VerificationIssue]
    summary_it: str                 # Riassunto in italiano per l'avvocato
```

**Trigger**: Sempre attivo in LEGIS mode. Per il layer anti-allucinazione avanzato (LLM call extra con Sonnet), trigger solo se: citation_count >= 5, oppure materia sensibile (penale/lavoro/privacy), oppure data storica.

**SSE Events emessi**: `agent_phase("verifying")`, `legis_verify(verdict)`

### Fase S — SYNTHESIZER (120s timeout)

**Modello**: `lexe-primary` (Mistral Large), temp 0.3, max 16K tokens, **streaming**
**Input**: Messaggio originale + Evidence Pack + VerificationVerdict
**Output**: Risposta testuale con citazioni inline [1], [2], etc.

#### REGOLA DURA: Evidence Pack come Single Source of Truth

Il Synthesizer opera sotto un vincolo fondamentale:

> **NESSUNA affermazione senza un EvidenceItem corrispondente.**

- Ogni affermazione normativa DEVE citare `[N]` che punta a un NormRef/CaseLawRef/MassimaRef nell'evidence pack
- Se l'LLM "sa" qualcosa ma non è nell'evidence pack → DEVE scrivere: *"Questo aspetto non è coperto dalle fonti recuperate nella presente ricerca."*
- Se l'evidence pack è vuoto o parziale per un punto → il Synthesizer lo segnala esplicitamente, non lo omette
- Anche affermazioni "ovvie" (es. "l'art. 2043 c.c. disciplina la responsabilità extracontrattuale") richiedono il riferimento `[N]`

**Prompt enforcement** (in `prompts.py`):
```
REGOLA ASSOLUTA: Ogni affermazione giuridica nella tua risposta DEVE essere supportata
da un elemento dell'evidence pack. Cita sempre con [N]. Se un punto non è coperto
dalle fonti, scrivi esplicitamente: "Questo aspetto non è coperto dalle fonti
recuperate nella presente ricerca." NON fare affermazioni basate sulla tua conoscenza
se non sono confermate dall'evidence pack.
```

**Il prompt di sintesi include**:

- Il messaggio originale dell'avvocato
- L'evidence pack strutturato (norme, giurisprudenza, massime, contraddizioni)
- Lo score di affidabilità per ogni fonte
- La regola "zero affermazioni senza evidence"
- Istruzioni per segnalare esplicitamente i punti non coperti

**Formato output standard**:

- A) Sintesi esecutiva (5-10 righe, ogni punto con citazione [N])
- B) Normativa rilevante (con link Normattiva/EUR-Lex)
- C) Giurisprudenza e massime (con identificativi)
- D) Contraddizioni e incertezze
- E) Aspetti non coperti dalle fonti (se presenti)
- F) Confidence score

**SSE Events emessi**: `agent_phase("synthesizing")`, `token` (streaming standard), `done` (con evidence_pack e verification nel payload)

---

## Evidence Pack — Schema Dati

### Modelli Pydantic

```python
class EvidencePack(BaseModel):
    id: UUID
    query: str
    legal_domain: str
    temporal_scope: str | None
    jurisdiction: str
    norms: list[NormRef]
    case_law: list[CaseLawRef]
    massime: list[MassimaRef]
    conflicts: list[Conflict]
    timeline: list[TimelineEvent] | None
    coverage: dict[str, SourceCoverage]
    confidence: ConfidenceScore
    created_at: datetime

class NormRef(BaseModel):
    id: str                         # UUID corto per citazione [N]
    act_type: str                   # "codice civile", "regolamento UE"
    article: str | None
    title: str
    text_excerpt: str               # Estratto minimo rilevante
    urn: str | None                 # Normattiva URN
    celex: str | None               # EUR-Lex CELEX
    url: str                        # Link ufficiale verificato
    source: str                     # "normattiva" | "eurlex"
    vigenza: VigenzaStatus
    confidence: float               # 0.0-1.0
    dedup_key: str | None           # URN o CELEX per deduplicazione cross-fonte
    raw_tool_result: dict           # Risultato grezzo del tool (solo audit, vedi nota privacy)

class VigenzaStatus(BaseModel):
    is_vigente: bool | None
    vigenza_at_fact_date: bool | None  # Vigenza all'epoca del fatto
    fact_date: date | None
    abrogato_da: str | None
    modificato_da: list[str] | None
    verified: bool                  # True se verificato via API
    confidence: float               # 0.0-1.0
    source: str                     # "normattiva_api" | "vigenza_fast_l1" | "inference"

class CaseLawRef(BaseModel):
    id: str
    authority: str                  # "Cass. civ., Sez. III"
    number: str
    year: int
    date: date | None
    principle: str                  # Principio di diritto o massima
    url: str | None
    source: str                     # "kb_search" | "infolex" | "lex_search"
    pertinence: int                 # 1-5
    confidence: float

class MassimaRef(BaseModel):
    id: str
    massima_id: UUID | None         # Da KB
    rv: str | None                  # Identificativo RV
    text: str
    sezione: str | None
    materia: str | None
    anno: int | None
    source: str
    score: float                    # RRF score da kb_search
    confidence: float

class Conflict(BaseModel):
    description: str
    items: list[str]                # ID dei NormRef/CaseLawRef in conflitto
    severity: Literal["info", "warning", "critical"]
    resolution_hint: str | None

class ConfidenceScore(BaseModel):
    overall: int                    # 0-100
    normativa: float                # 0.0-1.0
    giurisprudenza: float
    vigenza: float
    coverage: float
    cross_source_coherence: float
    breakdown_note: str             # Spiegazione breve
```

### Algoritmo di Scoring (0-100)

```
raw_score = round(
    source_reliability   * 0.30 +    # Fonti primarie vs secondarie
    vigenza_match        * 0.25 +    # Vigenza verificata via API
    link_verification    * 0.20 +    # Link ufficiale presente e verificato
    cross_source         * 0.15 +    # Fonti concordanti
    text_quality         * 0.10      # Testo estratto (non solo URL)
) * 100

# PENALITÀ HARD: se anche UNA SOLA citazione primaria fallisce existence check
if any_primary_citation_failed_existence:
    overall = min(raw_score, 60)     # Cap a 60 anche se tutto il resto è perfetto
else:
    overall = raw_score
```

**Razionale penalità**: Se anche una sola norma citata dall'evidence pack non passa l'existence check, c'è un rischio strutturale di allucinazione. Cap a 60 forza lo status WARN e avvisa l'avvocato.

| Componente         | 1.0 (max)                                   | 0.5                                | 0.0 (min)               |
| ------------------ | ------------------------------------------- | ---------------------------------- | ----------------------- |
| source_reliability | Tutte fonti primarie (Normattiva/EUR-Lex)   | Mix primarie + secondarie          | Solo web search         |
| vigenza_match      | API verification per tutte le norme         | Alcune verificate, alcune inferite | Nessuna verifica        |
| link_verification  | Tutti i link Normattiva/EUR-Lex funzionanti | Alcuni link, alcuni mancanti       | Nessun link             |
| cross_source       | 3+ fonti concordano                         | 2 fonti concordano                 | Fonti in contraddizione |
| text_quality       | Testo articolo estratto da fonte primaria   | Solo snippet/excerpt               | Solo titolo/URL         |

### Vigenza Storica come Prima Classe

La vigenza storica NON è un'opzione — è **obbligatoria** quando l'utente indica una data.

#### Trigger automatico vigenza storica

Rilevamento nel Planner (Fase P) di:
- Date esplicite: "nel 2019", "all'epoca dei fatti (2018)", "prima della riforma del 2024"
- Espressioni temporali: "all'epoca", "versione precedente", "prima della modifica", "nella versione vigente al"
- Pattern regex: `\b(nel|anno|al)\s*\d{4}\b`, `all'epoca`, `versione\s*(precedente|storica|originale)`

#### Comportamento

| Caso | Azione | Confidence |
|------|--------|-----------|
| Data rilevata + vigenza storica verificata via API | `vigenza_at_fact_date = true/false`, `fact_date = date` | 0.90 |
| Data rilevata + API non supporta vigenza storica | `vigenza_at_fact_date = null`, nota "vigenza storica non verificabile via API" | 0.50 (penalizzato) |
| Data rilevata + timeout/errore API | `vigenza_at_fact_date = null`, nota "verifica fallita" | 0.40 (fortemente penalizzato) |
| Nessuna data rilevata | Solo vigenza attuale, `fact_date = null` | Standard |

**Regola**: Se la vigenza storica è richiesta ma non verificabile, il `VigenzaStatus.confidence` scende a 0.40-0.50 e l'intero overall score ne risente proporzionalmente via il componente `vigenza_match`.

---

## Enhancement dei Tool Pre-Integrazione

### Tier 1 — BLOCCANTI (da fare prima di LEGIS AGENT)

| #   | Enhancement                                                                                        | Tool              | File                                                    | Effort     |
| --- | -------------------------------------------------------------------------------------------------- | ----------------- | ------------------------------------------------------- | ---------- |
| 1   | **Estrazione testo articoli** da Normattiva (il LEGIS AGENT ha bisogno del testo, non solo URL)    | normattiva_search | `lexe-tools-it/.../normattiva.py`                       | 1 giorno   |
| 2   | **Mappatura struttura CPC + CPP** per Brocardi (2 dei 5 codici maggiori non funzionano)            | infolex_search    | `lexe-tools-it/.../infolex.py`, `scrapers/selectors.py` | 1 giorno   |
| 3   | **Cache embedding** per kb_search (evita chiamate ridondanti a LiteLLM, risparmia latenza + costo) | kb_search         | `lexe-tools-it/.../kb_search.py`                        | 0.5 giorni |
| 4   | **Verifica bug dataGU** (formato YYYY-DD-MM vs YYYY-MM-DD per vigenza detail)                      | normattiva_search | `lexe-tools-it/.../normattiva_api.py`                   | 0.5 giorni |
| 5   | **Test suite golden per tutti gli 8 tool** (~70 test cases)                                        | tutti             | `lexe-tools-it/tests/`                                  | 3 giorni   |

### Tier 2 — ALTA PRIORITÀ (migliorano significativamente la qualità)

| #   | Enhancement                                                | Tool           | Effort      |
| --- | ---------------------------------------------------------- | -------------- | ----------- |
| 6   | Ricerca EU per titolo/keyword (non solo anno+numero)       | eurlex_search  | 1.5 giorni  |
| 7   | Force-refresh + vigenza storica (data specifica)           | vigenza_fast   | 0.75 giorni |
| 8   | Resilienza HTML multi-selettore per Brocardi               | infolex_search | 1 giorno    |
| 9   | Output JSON strutturato (response_format) per LLM enricher | lex_enricher   | 0.5 giorni  |
| 10  | Registry atti EU noti (GDPR, NIS2, AI Act, DSA, DMA)       | eurlex_search  | 0.5 giorni  |

### Tier 3 — NICE TO HAVE

| #   | Enhancement                                                             | Effort      |
| --- | ----------------------------------------------------------------------- | ----------- |
| 11  | Intent detection LLM-based per one_search                               | 0.5 giorni  |
| 12  | Deduplicazione cross-fonte in one_search                                | 0.25 giorni |
| 13  | Query expansion per kb_search (aggiungere testo articolo come contesto) | 0.5 giorni  |
| 14  | Retry + dedup per SearXNG (lex_search)                                  | 0.5 giorni  |
| 15  | Registry articoli abrogati noti (vigenza_fast)                          | 0.5 giorni  |

**Totale stimato: ~13 giorni** (Tier 1: 6gg, Tier 2: 4.25gg, Tier 3: 2.25gg)

---

## Layer Anti-Allucinazione — Dettaglio

### Posizione nella Pipeline

Post-processing, tra la Fase V (Verifier) e la Fase S (Synthesizer). Opzionalmente anche post-Synthesizer come controllo finale.

### Trigger (qualsiasi condizione è sufficiente)

| Condizione                                              | Soglia                                    | Modello usato        |
| ------------------------------------------------------- | ----------------------------------------- | -------------------- |
| citation_count >= 3                                     | 3+ norme distinte                         | `legal-tier2-haiku`  |
| Materia sensibile (penale, lavoro, privacy, tributario) | keyword detection                         | `legal-tier3-sonnet` |
| Data storica referenziata                               | `!vig=YYYY-MM-DD` o "all'epoca dei fatti" | `legal-tier2-haiku`  |
| Cross-jurisdizione (IT + EU)                            | Entrambe presenti                         | `legal-tier2-haiku`  |
| vigenza_confidence < 0.70 su qualsiasi norma            | Tool ha segnalato incertezza              | `legal-tier2-haiku`  |
| Tenant setting `force_verification`                     | Sempre                                    | Configurabile        |

### Controlli Eseguiti

1. **Esistenza citazione** (tool call): Ri-chiama normattiva_search/eurlex_search — costo: solo latenza API
2. **Vigenza re-check** (vigenza_fast): Cache hit quasi garantito — costo: ~0ms
3. **Validazione strutturale** (regex): Range articoli, formato URL, formato CELEX — costo: 0
4. **Cross-reference** (regex + tool): Se "modificato da X", verifica che X esista — costo: 1 tool call
5. **Coerenza logica** (LLM): Solo per priority "high" — costo: ~$0.004 (Haiku) o ~$0.06 (Sonnet)

### Verdetto

| Condizione                                                                                | Status                 |
| ----------------------------------------------------------------------------------------- | ---------------------- |
| Tutto verificato                                                                          | **PASS** (badge verde) |
| 1 citazione non trovata OPPURE 1 vigenza mismatch                                         | **WARN** (badge ambra) |
| 2+ citazioni non trovate OPPURE 2+ vigenza mismatch OPPURE contraddizione cross-reference | **FAIL** (badge rosso) |

### Su FAIL

- **Modalità A (default)**: Mostra la risposta con disclaimer rosso + dettagli
- **Modalità B (strict, opt-in per tenant)**: Una ri-generazione con istruzioni di correzione (max 1 retry, MAI ricorsivo)

### Budget Performance

| Priorità | Timeout totale | Latenza P50 attesa | Latenza P95 attesa |
| -------- | -------------- | ------------------ | ------------------ |
| Normal   | 3s             | ~500ms             | ~2.5s              |
| High     | 5s             | ~2s                | ~4.5s              |

---

## Frontend — UX per l'Avvocato

### Attivazione

- **Toggle "Scala"** nell'input bar, accanto al globe di web search
- Icona `Scale` (lucide-react), colore oro quando attivo
- Persistito in `configStore.enabledTools.legisMode`
- Controllabile per tenant via feature flag `ff_legis_agent_enabled`
- **In LEGAL mode: toggle ON by default e NON spegnibile dall'utente** — questo garantisce che ogni risposta in modalità LEGAL passi dal LEGIS AGENT con verifica fonti. L'utente non può bypassare i tool per ottenere risposte "veloci ma senza fonti"
- Il toggle diventa spegnibile solo se il tenant ha `ff_legis_allow_bypass=true` (default: false)

### Flusso UX

1. **Piano di Ricerca** (0-3s): Card con riepilogo query, fonti pianificate, tempo stimato. Bottoni [Conferma] [Modifica] [Annulla]. Skip se tenant ha `auto_confirm`.

2. **Progresso Fonti** (3-15s): Barre di progresso per ogni fonte (Normattiva, EUR-Lex, Massimario, Dottrina). Status live: ricerca → trovati N risultati → fallito.

3. **Evidence Pack** (incrementale): Card per ogni risultato, raggruppate per categoria (Normativa, Giurisprudenza, Massime, Contraddizioni). Ogni card ha: titolo, snippet, badge vigenza (verde/rosso/ambra), confidence %, link fonte.

4. **Verifica** (1-5s): Badge complessivo [87% PASS], checklist verifica, dettagli espandibili.

5. **Sintesi Finale** (streaming): Risposta con citazioni [1], [2] cliccabili che scrollano all'evidence pack.

6. **Post-risposta**: L'evidence pack si compatta in un accordion "87% PASS — 10 fonti trovate [Espandi]".

### Nuovi SSE Events

| Event                | Quando                      | Payload chiave                                                        |
| -------------------- | --------------------------- | --------------------------------------------------------------------- |
| `agent_phase`        | Transizione di fase         | `phase: "planning" \| "researching" \| "verifying" \| "synthesizing"` |
| `agent_plan`         | Piano di ricerca pronto     | `ResearchPlan` JSON                                                   |
| `legis_evidence`     | Ogni risultato normalizzato | `NormRef \| CaseLawRef \| MassimaRef`                                 |
| `legis_verify`       | Verdetto verifica           | `VerificationVerdict`                                                 |
| `legis_disambiguate` | Serve chiarimento           | `question, options[], timeout_ms`                                     |

Gli eventi `tool_call`, `tool_result`, `tool_end`, `token`, `done` restano invariati.

### Nuovi Componenti Frontend

```
src/components/legis/
  LegisResearchPanel.tsx       # Container principale
  LegisResearchPlan.tsx        # Card piano + conferma
  LegisSourceProgress.tsx      # Progresso per fonte
  LegisEvidencePack.tsx        # Container evidence raggruppato
  LegisEvidenceItem.tsx        # Singola card evidence
  LegisVigenzaBadge.tsx        # Badge vigenza (verde/rosso/ambra)
  LegisVerificationBadge.tsx   # Score + PASS/WARN/FAIL
  LegisConfidenceBar.tsx       # Barra/cerchio confidence
  LegisDisambiguation.tsx      # Chips chiarimento inline
  LegisTimeline.tsx            # Timeline cambiamenti normativi

src/stores/legisStore.ts       # Zustand store per stato LEGIS
```

---

## Struttura File Backend

```
lexe-core/src/lexe_core/agent/
  __init__.py
  config.py                    # Costanti e settings LEGIS-specific
  models.py                    # ResearchPlan, EvidencePack, VerificationVerdict, etc.
  pipeline.py                  # Orchestratore PRVS (entry point)
  planner.py                   # Fase P — analisi e decomposizione query
  researcher.py                # Fase R — esecuzione tool e raccolta evidence
  verifier.py                  # Fase V — verifica citazioni e vigenza
  synthesizer.py               # Fase S — sintesi finale streaming
  prompts.py                   # Tutti i prompt template per ogni fase
  events.py                    # Helper SSE per eventi LEGIS-specific
  scoring.py                   # Algoritmo confidence scoring

lexe-core/src/lexe_core/verification/
  __init__.py
  extractor.py                 # Estrazione citazioni da testo (regex)
  structural.py                # Validazione strutturale (formato URL, range articoli)
  runner.py                    # Orchestratore verifiche
  models.py                    # ExtractedCitation, VerificationIssue, etc.
```

### File Esistenti da Modificare

| File                                                 | Modifica                                       | Complessità |
| ---------------------------------------------------- | ---------------------------------------------- | ----------- |
| `lexe-core/src/lexe_core/gateway/customer_router.py` | Aggiungere routing `if legis_mode` (~30 righe) | Bassa       |
| `lexe-core/src/lexe_core/gateway/sse_contracts.py`   | Aggiungere 4 nuovi event types                 | Bassa       |
| `lexe-core/src/lexe_core/config.py`                  | Aggiungere ff_legis_agent + model config       | Bassa       |
| `lexe-core/src/lexe_core/tools/executor.py`          | Esporre `vigenza_fast` come tool callable      | Media       |
| `lexe-core/src/lexe_core/tools/definitions.py`       | Aggiungere `vigenza_fast` alla tool list       | Bassa       |
| `lexe-webchat/src/hooks/useStreaming.ts`             | Gestire nuovi event types                      | Media       |
| `lexe-webchat/src/stores/configStore.ts`             | Aggiungere `legisMode` toggle                  | Bassa       |
| `lexe-webchat/src/components/chat/ChatMessage.tsx`   | Integrare `LegisResearchPanel`                 | Media       |

### Nuova Migration

`lexe-core/migrations/008_legis_agent.sql`:

- `core.evidence_packs` — storage opzionale evidence pack per audit
- `core.verification_logs` — log verifica anti-allucinazione
- Feature flags in `core.tenant_feature_flags`

---

## Sequenza di Implementazione

### Sprint 1 — Test & Enhancement Tool (6 giorni)

1. **Test suite golden** per tutti gli 8 tool (3 giorni)
   
   - `test_normattiva.py` (12 golden + 5 edge)
   - `test_eurlex.py` (6 golden + 4 edge)
   - `test_infolex.py` (9 golden + 5 edge)
   - `test_kb_search.py` (8 golden + 5 edge)
   - `test_lex_search.py` (4 golden)
   - `test_vigenza_fast.py` (5 golden)
   - `test_cross_tool.py` (5 consistency)
   - `test_reliability.py` (11 failure scenarios con mock)

2. **Fix bloccanti** (3 giorni)
   
   - Estrazione testo articoli da Normattiva
   - Mappatura CPC + CPP per Brocardi
   - Cache embedding per kb_search
   - Verifica bug dataGU

### Sprint 2 — LEGIS AGENT Core Backend (5 giorni)

1. Migration 008 + config (0.5 giorni)
2. `agent/models.py` — tutti i Pydantic models (1 giorno)
3. `agent/planner.py` — Fase P con prompt (1 giorno)
4. `agent/researcher.py` — Fase R con tool loop esteso (1.5 giorni)
5. `agent/pipeline.py` + `events.py` — orchestrazione + SSE (1 giorno)

### Sprint 3 — Verifier + Synthesizer (4 giorni)

1. `verification/extractor.py` + `structural.py` — regex citation extraction (1 giorno)
2. `agent/verifier.py` — Fase V con verifica vigenza + cross-ref (1.5 giorni)
3. `agent/synthesizer.py` — Fase S streaming (1 giorno)
4. `agent/scoring.py` — algoritmo confidence (0.5 giorni)

### Sprint 4 — Integrazione + Frontend (5 giorni)

1. Routing in `customer_router.py` + SSE events (1 giorno)
2. `legisStore.ts` + `useStreaming.ts` estensione (1 giorno)
3. Componenti frontend (3 giorni):
   - LegisAgentToggle, LegisResearchPlan, LegisSourceProgress
   - LegisEvidencePack, LegisEvidenceItem, LegisVigenzaBadge
   - LegisVerificationBadge, LegisConfidenceBar
   - Integrazione in ChatMessage.tsx

### Sprint 5 — Enhancement Tier 2 + Polish (4 giorni)

1. Ricerca EU per keyword/titolo (1.5 giorni)
2. Vigenza storica (data specifica) (0.75 giorni)
3. Resilienza HTML Brocardi (1 giorno)
4. Test end-to-end + fix (0.75 giorni)

**Totale: ~24 giorni lavorativi** (5 sprint)

---

## Verifica End-to-End

### Test Manuali su Staging

| #   | Query                                                              | Verifica                                                                                                                     |
| --- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| 1   | "Presupposti risarcimento danno ex art. 2043 c.c."                 | Piano mostra Normattiva + KB + Dottrina. Evidence pack ha art. 2043, 2059, massime Cassazione. Vigenza verified. Score > 80. |
| 2   | "Art. 6 GDPR base giuridica trattamento dati"                      | EUR-Lex trovato con CELEX 32016R0679. Normattiva per D.Lgs. 196/2003. Cross-jurisdiction detected.                           |
| 3   | "Licenziamento per giusta causa nel 2019 — cosa prevede la legge?" | Data storica 2019 rilevata. Vigenza verificata al 2019. Art. 2119 CC + L. 300/1970 art. 18.                                  |
| 4   | "Art. 99999 codice civile"                                         | Strutturale FAIL — articolo fuori range. Badge rosso.                                                                        |
| 5   | "Differenza tra responsabilità contrattuale ed extracontrattuale"  | Multi-norma: art. 1218 + 2043 CC. Evidence pack con confronto. Score > 75.                                                   |

### Test Automatici

```bash
# Tool golden tests
cd lexe-tools-it && pytest tests/ -m gold --tb=short

# Reliability tests (mock)
cd lexe-tools-it && pytest tests/ -m reliability --tb=short

# LEGIS AGENT pipeline test
cd lexe-core && pytest tests/agent/ -v

# Frontend component tests
cd lexe-webchat && npm test -- --testPathPattern=legis
```

### Monitoring in Produzione

- `core.tool_execution_logs` — latenza e failure rate per tool
- `core.evidence_packs` — score distribuzione, coverage
- `core.verification_logs` — PASS/WARN/FAIL ratio, hallucination rate
- Dashboard admin: sezione "LEGIS Agent Quality" con trend 7 giorni

---

## Audit Trail e Privacy by Design

### raw_tool_result — Accesso Controllato

Il campo `raw_tool_result` in ogni EvidenceItem contiene il risultato grezzo del tool per audit trail completo. Regole:

1. **Storage**: Salvato in `core.evidence_packs.pack_data` (JSONB) — NON esposto al frontend
2. **Accesso**: Solo ruoli `superadmin` e `audit` possono leggere `raw_tool_result` via endpoint admin dedicato
3. **Pseudonimizzazione**: Se `raw_tool_result` contiene dati utente (es. query con nomi propri), applicare pseudonimizzazione prima dello storage:
   - Sostituire nomi propri rilevati con token `[PERSONA_N]`
   - Mantenere hash reversibile per audit autorizzato
   - Seguire pattern `lexe-privacy` (modulo GDPR esistente)
4. **Retention**: 90 giorni, poi eliminazione automatica (coerente con GDPR Art. 5.1.e)
5. **Frontend**: L'evidence pack nel payload SSE e nel `done_event` NON include `raw_tool_result` — solo i campi normalizzati (titolo, snippet, URN, URL, vigenza, confidence)

### Coerenza con Brand

Questo design è allineato con:
- **"Evidenze prima delle promesse"** — ogni affermazione ha una fonte verificabile
- **Citation tracking** — audit trail completo per ogni citazione
- **Trasparenza** — l'avvocato vede sempre confidence, vigenza e link
- **Sicurezza** — raw data accessibile solo a chi ha diritto

---

## Evidence Dedup Cross-Fonte

### Strategia di Deduplicazione

Quando fonti diverse restituiscono la stessa norma/sentenza, merge in un unico EvidenceItem:

| Tipo | Dedup Key | Merge Strategy |
|------|----------|----------------|
| Norme IT | `URN` (es. `urn:nir:stato:...;262:2~art2043`) | Tieni snippet più lungo, URL Normattiva (primario), confidence più alta |
| Norme EU | `CELEX` (es. `32016R0679`) | Tieni snippet più lungo, URL EUR-Lex (primario) |
| Giurisprudenza | `(numero, anno, sezione)` tuple | Merge massime, tieni principio più completo |
| Massime KB | `massima_id` UUID | Già unique, nessun merge |
| Web results | `URL` normalizzato | Tieni primo trovato, ignora duplicati |

### Implementazione

In `researcher.py`, metodo `_add_to_evidence_pack()`:

```python
def _dedup_key(item: NormRef | CaseLawRef | MassimaRef) -> str:
    if isinstance(item, NormRef):
        return item.urn or item.celex or f"{item.act_type}:{item.article}"
    elif isinstance(item, CaseLawRef):
        return f"cass:{item.number}/{item.year}/{item.section}"
    elif isinstance(item, MassimaRef):
        return str(item.massima_id) or f"massima:{item.rv}"
```

Se `dedup_key` già presente → merge (update campi nulli, scegli confidence migliore). Se assente → insert.

---

## Definition of Done

### Backend — Criteri PASS per Rollout

| Criterio | Target | Metrica |
|----------|--------|---------|
| PRVS end-to-end su 10 query gold | Tutte PASS o WARN motivato | Test automatici |
| Zero citazioni inventate | 0 NormRef/CaseLawRef con existence_check = false nei test | Test automatici |
| Zero affermazioni non supportate | Ogni frase nella sintesi ha `[N]` o "non coperto" | Review manuale su 10 query |
| P95 latenza planner | < 3s | Benchmark |
| P95 latenza researcher | < 15s | Benchmark |
| P95 latenza verifier | < 5s | Benchmark |
| P95 latenza synthesizer | < 30s (streaming first token < 3s) | Benchmark |
| P95 latenza totale | < 45s | Benchmark |
| Fallback controllati | Su API failure: evidence pack parziale + WARN, MAI crash | Test reliability |
| Per-tool limits rispettati | Nessun tool supera il max, chiusura graceful | Test automatici |

### Qualità — Criteri per il Prodotto

| Criterio | Target | Come misurare |
|----------|--------|--------------|
| PASS ratio su query civili standard | >= 85% | 20 query gold civile, contare PASS |
| WARN spiegato e riproducibile | Ogni WARN ha `summary_it` chiaro e `issues[]` dettagliati | Review manuale |
| FAIL rarissimo ma gestito | < 5% su query gold; output safe con disclaimer | Test automatici |
| Vigenza storica corretta | 100% su 5 query con data esplicita | Test dedicati |
| Dedup funzionante | 0 card duplicate nell'evidence pack | Test cross-fonte |
| Evidence pack come SSoT | 0 affermazioni senza `[N]` nella sintesi | Audit automatico post-synthesis |

### Prodotto e Posizionamento

| Criterio | Verifica |
|----------|---------|
| Evidence pack visibile e navigabile nell'UI | Review UX su staging |
| Verification badge (PASS/WARN/FAIL) prominente | Screenshot review |
| Link alle fonti ufficiali funzionanti | Click test su tutte le URL nell'evidence pack |
| Coerente col brand "evidenze prima delle promesse" | Nessuna promessa senza fonte, nessun numero senza link |
| Confidence score leggibile e spiegato | Tooltip/expand con breakdown |
| Scale toggle funzionante e non bypassabile in LEGAL mode | Test UX |
