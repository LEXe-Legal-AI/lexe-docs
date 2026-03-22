# LEXe v4 — From Pipeline to Agentic Legal AI

> **"Il Manus del Legal AI"**
> Architettura Multi-Agente per Ragionamento Giuridico Avanzato

| Campo | Valore |
|---|---|
| Versione | DRAFT 1.0 |
| Data | 22 Marzo 2026 |
| Baseline | LEXe v3 — Sprint 13 PRVS + Sprint 14 NOVA RATIO |
| Obiettivo | LEXe v4 — Zero-Regex, Full-AI, Multi-Agent Orchestrated Pipeline |
| Research | 40+ papers post-Giugno 2025 analizzati |

---

## PARTE 1 — ARCHITETTURA ATTUALE (LEXe v3)

### 1.1 Overview

LEXe v3 usa una **pipeline sequenziale PRVS** (Plan → Research → Verify → Synthesize) con routing binario e depth-based budgeting. Non e' un sistema multi-agente: e' una **catena di funzioni LLM** con tool-use, dove ogni fase ha un singolo modello che produce un output consumato dalla fase successiva.

```
┌──────────────────────────────────────────────────────────────┐
│                    USER QUERY (HTTP POST)                      │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ PRE-PROCESSING                                                │
│ tenant_resolver → model_resolver → memory_client → persona    │
│ Budget: wall_clock 120s, tool_calls 30, model_calls 20        │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ INTENT DETECTION (lexe-fast, ~200ms, 12s timeout)             │
│ Output: PipelineLevel × ScenarioType × DepthLevel             │
│                                                                │
│ Binary Routing:                                                │
│   CHAT → ToolloopStrategy (generic LLM + tools)               │
│   DIRECT/SIMPLE/STANDARD/COMPLEX → LegalStrategy (PRVS)       │
└──────────────┬───────────────────────────┬───────────────────┘
               │                           │
     ┌─────────▼─────────┐      ┌─────────▼──────────────────┐
     │  TOOLLOOP          │      │  LEGIS PIPELINE (PRVS)      │
     │  Single LLM loop   │      │                              │
     │  with tool access   │      │  P: Planner (lexe-fast/     │
     │                     │      │     lexe-frontier, 30-45s)  │
     │  Max 5 iterations   │      │     → ResearchPlan          │
     │  Max 25 tool calls  │      │                              │
     │                     │      │  R: Researcher (parallel     │
     │  Model: lexe-primary│      │     tool dispatch, 8-40      │
     │                     │      │     calls, circuit breaker)  │
     │                     │      │     → EvidencePack           │
     │                     │      │                              │
     │                     │      │  V: Verifier (deterministic  │
     │                     │      │     + LLM coherence,         │
     │                     │      │     lexe-verifier, 15-20s)   │
     │                     │      │     → VerificationVerdict    │
     │                     │      │                              │
     │                     │      │  S: Synthesizer (streaming,  │
     │                     │      │     lexe-primary, 60s,       │
     │                     │      │     scenario-specific prompt)│
     │                     │      │     → AsyncGenerator[str]    │
     └─────────┬───────────┘      └──────────┬───────────────────┘
               │                              │
               └──────────────┬───────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│ POST-PROCESSING                                               │
│ confidence_calibration → follow_ups → persist_response        │
│ → EventSink → Langfuse spans → SSE done_event                │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 Knowledge Base (lexe-max) — Il Fondamento

La KB e' il vero asset differenziante di LEXe. Tutto il ragionamento giuridico si fonda su questa infrastruttura dati.

**Stack:** PostgreSQL 17 + pgvector 0.7.4 + Apache AGE + ParadeDB pg_search

```
┌─────────────────────────────────────────────────────────────┐
│                    lexe-max (porta 5436)                      │
│                    Schema: kb.*                               │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ NORMATIVA (Legislazione Italiana)                        │ │
│  │                                                           │ │
│  │ kb.work              69 codici/leggi (CC, CP, TUB...)    │ │
│  │ kb.normativa          10.154 articoli                    │ │
│  │ kb.normativa_chunk    41.256 chunks per retrieval        │ │
│  │ kb.normativa_chunk_embeddings  41.256 vectors (1536d)    │ │
│  │ kb.normativa_fts      tsvector Italian full-text search  │ │
│  │                                                           │ │
│  │ Retrieval: Hybrid (Dense pgvector + Sparse BM25 + RRF)  │ │
│  │ Index: HNSW cosine_ops (text-embedding-3-small, 1536d)  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ GIURISPRUDENZA (Massimario di Cassazione)                │ │
│  │                                                           │ │
│  │ kb.massime             38.718 massime attive             │ │
│  │ kb.embeddings          38.718 vectors (1536d)            │ │
│  │ kb.sentenza            Metadati sentenze                 │ │
│  │                                                           │ │
│  │ Retrieval: Hybrid (Dense + Sparse + RRF + Norm Boost)   │ │
│  │ Recall@10: 97.5%  |  MRR: 0.756                        │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ CITATION GRAPH (Relazioni tra fonti)                     │ │
│  │                                                           │ │
│  │ kb.graph_edges         58.737 edges (CITES, run_id=1)   │ │
│  │ kb.norms               4.128 norme uniche               │ │
│  │ kb.massima_norms       42.338 link massima→norma         │ │
│  │ kb.graph_runs          Versioning dei grafi              │ │
│  │                                                           │ │
│  │ Resolution cascade: rv_exact (47.5%) → sez_num_anno     │ │
│  │ (25%) → rv_text (14.5%) → sez_num_raw (8.3%)           │ │
│  │ Norm coverage: 60.3% (23.365 massime linkate a norme)   │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ DOTTRINA (Annotazioni e commenti)                        │ │
│  │                                                           │ │
│  │ kb.annotation          13.281 note dottrinali            │ │
│  │ kb.annotation_embeddings  13.180 vectors (1536d)         │ │
│  │ kb.annotation_link     Link annotazione→articolo         │ │
│  │                                                           │ │
│  │ Tipi: nota, commento, ratio decidendi, brocardo          │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ PRINCIPI (NOVA RATIO — Sprint 14)                        │ │
│  │                                                           │ │
│  │ kb.principle_anchors   24 principi fondamentali          │ │
│  │                        + embeddings (1536d)              │ │
│  │ kb.principle_massima_links  Link principio→massima       │ │
│  │                                                           │ │
│  │ Tipi: constitutional, general_clause, eu_principle       │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ CONCEPT HIERARCHY (Sprint 10)                            │ │
│  │                                                           │ │
│  │ kb.concept_hierarchy   Tassonomia concettuale            │ │
│  │                        per espansione semantica query     │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Numeri chiave:**
| Metrica | Valore |
|---|---|
| Articoli di legge indicizzati | 10.154 |
| Chunks con embeddings | 41.256 |
| Massime Cassazione | 38.718 |
| Citation graph edges | 58.737 |
| Norme uniche nel graph | 4.128 |
| Link massima→norma | 42.338 |
| Annotazioni dottrinali | 13.281 |
| Principi fondamentali | 24 |
| Codici/leggi coperti | 69 |
| Embedding model | text-embedding-3-small (1536d) |
| Vector index | HNSW (pgvector) |
| Full-text search | ParadeDB pg_search + Italian tsvector |
| Graph engine | Apache AGE (Cypher) + SQL relazionale |

**Questa KB e' il motivo per cui LEXe puo' fare cio' che nessun altro fa**: retrieval ibrido grounded su fonti primarie italiane, con un citation graph che connette norme, massime e dottrina. Qualsiasi architettura multi-agente v4 si fonda su questa KB come source of truth.

### 1.3 Componenti Pipeline

| Componente | File | Modello LLM | Timeout | Funzione |
|---|---|---|---|---|
| Intent Detector | `intent_detector.py` | lexe-fast | 12s | Classifica livello, scenario, depth |
| Planner | `planner.py` | lexe-fast / lexe-frontier | 30-45s | Decompone in SourceQuery[] |
| Researcher | `researcher.py` | - (tool dispatch) | depth-based | Esegue tool, normalizza in EvidencePack |
| Verifier | `verifier.py` | lexe-verifier | 15-20s | Validazione strutturale + LLM coherence |
| Synthesizer | `synthesizer.py` | lexe-primary | 60s | Genera risposta streaming |
| Evidence Classifier | `evidence_classifier.py` | lexe-fast (fallback) | 5s | NOVA RATIO: classifica zero-norm |
| Solidarity Scorer | `scoring.py` | - | - | NOVA RATIO: scoring per principi |

### 1.4 Tool Disponibili

| Tool | Fonte | Latency P95 | Purity |
|---|---|---|---|
| normattiva_search | Normattiva.it API | ~2s | 3 |
| eurlex_search | EUR-Lex CELLAR | ~3s | 3 |
| infolex_search | Brocardi KB | ~1s | 4 |
| kb_search | Massimario (38K massime, hybrid) | ~150ms | 2 |
| kb_normativa_search | Codici (5.8K articoli, hybrid) | ~150ms | 2 |
| lex_search | SearXNG (13 siti legali) | ~5s | 4 |
| lex_search_enriched | SearXNG + LLM analysis | ~8s | 4 |
| vigenza_fast | Normattiva API (L1/L2 cache) | ~500ms | 3 |
| doctrine_search | KB annotations (13K, hybrid) | ~200ms | 2 |
| juris_verifier | Massime + graph cross-ref | ~300ms | 2 |
| principle_mapper | Principle anchors (24 seed) | ~100ms | 2 |

### 1.5 Limiti dell'Architettura Attuale

| Limite | Descrizione | Impatto |
|---|---|---|
| **Sequenziale** | PRVS e' una catena lineare: ogni fase aspetta la precedente | Latency alto (P95 ~40s STANDARD) |
| **Single-Model** | Ogni fase usa UN singolo modello LLM | No diversita' di prospettiva |
| **Regex-Heavy** | Intent detector, citation extractor, norm parser usano regex | Fragile, non generalizza |
| **No Self-Correction** | Il verifier puo' segnalare problemi ma non corregge la sintesi | Errori propagati all'utente |
| **No Debate** | Nessun meccanismo adversarial per sfidare le conclusioni | Consenso acritico |
| **Budget Statico** | Depth budget fisso per livello (8/25/40 tool calls) | No adattamento runtime |
| **No Principled Reasoning** | Ragionamento solo norm-centric (NOVA RATIO e' un branch, non un agente) | Debole su casi complessi |

---

## PARTE 2 — SISTEMI MULTI-AGENTE: TASSONOMIA E PAPERS

### 2.1 Tassonomia delle Architetture Multi-Agente

```
MULTI-AGENT SYSTEMS
│
├── ORCHESTRATED (Central Controller)
│   ├── Static Orchestration (pipeline predefinita)
│   ├── Dynamic Orchestration (routing adattivo)
│   └── Hierarchical (planner → executor → verifier)
│
├── DECENTRALIZED (Peer-to-Peer)
│   ├── Debate / Adversarial (agenti si sfidano)
│   ├── Swarm (emergenza dal basso)
│   └── Consensus (negoziazione fino ad accordo)
│
├── ENSEMBLE (Multiple Models → Fusion)
│   ├── Mixture-of-Agents (layers di modelli)
│   ├── Self-MoA (stesso modello, multiple samples)
│   └── Routing-Based (modello giusto per il task)
│
└── HYBRID (Combinazioni)
    ├── Orchestrator + Debate (planner delega, agents dibattono)
    ├── Hierarchical + Reflection (self-correction multi-livello)
    └── Tool-Augmented Debate (debate con retrieval grounded)
```

### 2.2 Papers: Sistemi Orchestrati

| Paper | Autori/Fonte | Data | Architettura | Rilevanza LEXe |
|---|---|---|---|---|
| **L-MARS** | arXiv 2509.00761 | Set 2025 | Decomposizione + ricerca eterogenea + Judge Agent | **MOLTO ALTA** — quasi identico a LEXe |
| **DAAO** (Difficulty-Aware) | arXiv 2509.11079 | Set 2025 | VAE difficulty estimation + workflow dinamico | **MOLTO ALTA** — formalizza il depth routing |
| **Evolving Orchestration** | arXiv 2505.19591 | Mag 2025 | RL-trained orchestrator dinamico | ALTA — next-gen routing |
| **SOLAR** | arXiv 2509.00710, CIKM 2025 | Set 2025 | Knowledge formalization + symbolic reasoning | ALTA — KB enhancement |
| **L4L** | arXiv 2511.21033 | Nov 2025 | Role-differentiated agents + SMT verification | ALTA — formal verification |
| **GoalAct** | arXiv 2504.16563, NCIIP 2025 | Apr 2025 | Global planning + hierarchical execution | ALTA — gap remediation |
| **AgentSpawn** | arXiv 2602.07072 | Feb 2026 | Dynamic agent spawning via complexity detection | ALTA — adaptive depth |
| **SALLMA** | IEEE ICSE 2025 | 2025 | Dual-layer (Operational + Knowledge) | MEDIA |
| **MASLegalBench** | arXiv 2509.24922 | Set 2025 | Benchmark per MAS legali (950 casi) | MEDIA |
| **LRAS** | arXiv 2601.07296 | Gen 2026 | Agentic search + legal reasoning | ALTA |

**L-MARS** e' il paper piu' vicino a LEXe: decompone query legali in sub-problemi, lancia ricerche eterogenee (web, RAG locale, DB giurisprudenziale), e usa un **Judge Agent** per verificare sufficienza, giurisdizione e validita' temporale prima della sintesi. Iterativo.

**DAAO** formalizza il nostro depth routing con un VAE per stimare la difficolta' della query e generare workflow dinamici. Potrebbe sostituire l'intent detector basato su regex/prompt.

### 2.3 Papers: Sistemi Decentralizzati e Debate

| Paper | Autori/Fonte | Data | Pattern | Rilevanza LEXe |
|---|---|---|---|---|
| **Multiagent Debate** | Du et al., ICML 2024 | 2023-2025 | Peer-to-peer debate, convergenza | ALTA — riduce allucinazioni |
| **DebateCV** | WWW '26 | 2026 | 2 debaters + 1 judge, claim verification | ALTA — mirror processo legale |
| **Tool-MAD** | arXiv 2601.04742 | Gen 2026 | Debate con retrieval tool durante i round | **MOLTO ALTA** — debate grounded |
| **L4L** | arXiv 2511.21033 | Nov 2025 | Agenti accusatore/difensore + SMT solver | **MOLTO ALTA** — Legal AI nativo |
| **A-HMAD** | J. King Saud Univ., 2025 | 2025 | Debate eterogeneo con modelli diversi | MEDIA |
| **MAD-Fact** | arXiv 2510.22967 | 2025 | Debate per long-form factuality | MEDIA |
| **Multi-Agent Reflexion (MAR)** | arXiv 2512.20845 | Dic 2025 | Cross-agent reflection (rompe loop) | ALTA |
| **Self-MoA** | arXiv 2502.00674, NeurIPS 2025 | Feb 2025 | Stesso modello, multiple samples, aggregazione | ALTA |
| **Confidence-Aware Routing** | arXiv 2601.04861 | Gen 2026 | Per-turn model escalation via confidence | ALTA — mappa su depth |
| **MasRouter** | ACL 2025 | 2025 | Learned meta-controller per topology selection | MEDIA |

**Tool-MAD** e' il paper piu' rilevante: estende il debate con retrieval di evidenza reale durante i round. Gli agenti citano statuti e giurisprudenza dal database, non dalla memoria parametrica. Perfetto per LEXe.

**L4L** e' l'unico paper che combina agenti differenziati per ruolo (accusatore, difensore, giudice) con **verifica formale SMT**. Garantisce consistenza logica delle conclusioni legali.

### 2.4 Papers: Agentic RAG, Tool-Use e Knowledge Graph

| Paper | Autori/Fonte | Data | Pattern | Rilevanza LEXe |
|---|---|---|---|---|
| **HalluGraph** | arXiv 2512.01659 | Dic 2025 | Audit allucinazioni via KG alignment (AUC 0.979) | **MOLTO ALTA** — verifica citazioni |
| **Reasoning RAG Sys1/Sys2** | arXiv 2506.10408, IJCNLP-AACL 2025 | Giu 2025 | Predefined (fast) vs Agentic (deliberativo) RAG | **MOLTO ALTA** — formalizza routing |
| **MA-RAG** | arXiv 2505.20096 | Mag 2025 | 4 agenti specializzati (Planner/Definer/Extractor/QA) | ALTA — blueprint multi-wave |
| **Microsoft GraphRAG** | arXiv 2404.16130 | Apr 2024 | Community detection su KG → global summarization | ALTA — sfruttare citation graph |
| **CRAG** (Corrective RAG) | arXiv 2401.15884, ICLR 2025 | Gen 2024 | Confidence-triggered re-retrieval | ALTA — mappa su 3-band |
| **A-RAG** (Hierarchical) | arXiv 2602.03442 | Feb 2026 | Keyword + semantic + chunk read come tool | ALTA — multi-granularity |
| **ToolTree** | arXiv 2603.12740, ICLR 2026 | Mar 2026 | MCTS per tool planning (+10% F1) | MEDIA — tool selection |
| **LexRAG** | SIGIR 2025 | 2025 | Benchmark multi-turn legal RAG (1.013 dialoghi) | MEDIA — benchmark |
| **Legal KG+Vector+NMF** | arXiv 2502.20364 | Feb 2025 | Hybrid vector + KG + topic hierarchy | ALTA — allineato con lexe-max |
| **Reasoning Trap** | arXiv 2510.22977 | Ott 2025 | Reasoning forte → piu' tool hallucination | ALTA — warning critico |
| **UQ/Calibration Survey** | arXiv 2503.15850, KDD 2025 | Mar 2025 | Post-hoc calibration per confidence LLM | ALTA — migliorare scoring |
| **LLM Uncertainty** | ICLR 2025 | 2025 | Tutti i frontier model sono overconfident (ECE 0.12+) | ALTA — serve calibrazione |
| **Hallucination Study** | arXiv 2405.20362, JELS 2025 | 2025 | Lexis+ AI e Westlaw AI allucinano 17-33% | ALTA — baseline da battere |

**HalluGraph** e' il paper piu' impattante per LEXe: usa allineamento con Knowledge Graph per quantificare allucinazioni con due metriche — *Entity Grounding* (l'entita' citata esiste nella fonte?) e *Relation Preservation* (la relazione asserita e' supportata?). AUC 0.979 su documenti legali strutturati. Perfetto per verificare che quando LEXe cita "art. 35 c.p.", il contenuto corrisponda alla fonte.

**Reasoning Trap** e' un warning critico: modelli con reasoning forte (reasoning_effort=high) allucinano **piu'** tool calls spurie. LEXe usa `lexe-frontier` con reasoning=high per il Deep path — serve tool availability grounding esplicito.

**ICLR 2025 Calibration**: Tutti i frontier model sono sistematicamente overconfident (Claude Opus 4.5: ECE=0.120). LEXe dovrebbe applicare calibrazione post-hoc (temperature scaling) al confidence score anziche' usare output raw.

### 2.5 Insight Chiave dalla Letteratura

1. **Il debate pattern riduce le allucinazioni** — Du et al. dimostrano che agenti che si sfidano convergono verso risposte piu' accurate. Per il legal, dove l'allucinazione e' inaccettabile, il debate e' fondamentale.

2. **Il debate deve essere grounded** — Tool-MAD dimostra che il debate puro (parametrico) non basta. Gli agenti devono poter cercare evidenza reale durante il dibattito.

3. **Self-MoA > MoA per consistenza** — NeurIPS 2025 dimostra che mescolare modelli diversi spesso *peggiora* la qualita'. Per il legal, dove la consistenza e' critica, meglio multiple samples dallo stesso modello con aggregazione.

4. **La difficulty estimation deve essere learned, non rule-based** — DAAO usa un VAE; il nostro intent detector usa regex+prompt. Un classifier trainato su dati reali sarebbe piu' robusto.

5. **L'architettura ottimale e' ibrida** — Google (Marzo 2026) conclude che non esiste un singolo pattern migliore: serve orchestrazione centralizzata per analisi strutturata, decentralizzata per ricerca esplorativa.

---

## PARTE 3 — TRE VARIANTI PER LEXe v4

### Principi Comuni a Tutte le Varianti

- **Zero Regex**: Nessun pattern matching hardcoded. Classificazione, estrazione citazioni, routing — tutto via LLM o modelli trainati.
- **Full AI**: Ogni decisione e' presa da un modello (LLM, classificatore, o embeddings), non da regole.
- **Grounded**: Ogni affermazione dell'agente deve essere ancorata a una fonte verificabile.
- **Fallback**: LEXe v3 (PRVS + NOVA RATIO) resta come fallback sicuro.

---

### VARIANTE A — "TRIBUNALE" (Adversarial Debate + Judge)

> Ispirata a: L4L, DebateCV, Tool-MAD

```
User Query
  │
  ▼
┌──────────────────────────────────────────────────────────────┐
│ CLERK AGENT (Cancelliere)                                     │
│ Modello: lexe-fast                                            │
│                                                                │
│ Funzioni:                                                      │
│ - Classifica la query (difficulty, domain, tipo evidenza)      │
│ - Assegna il "caso" alla formazione giudicante appropriata    │
│ - Decide il budget (simple → 1 round, complex → 3 rounds)    │
│ - Prepara il fascicolo (memory, context, precedenti)          │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│ DEBATE CHAMBER (Camera di Consiglio)                          │
│                                                                │
│ ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│ │ ADVOCATE AGENT   │  │ CRITIC AGENT     │  │ SCHOLAR AGENT │  │
│ │ (Avvocato)       │  │ (Contraddittore) │  │ (Dottrinario) │  │
│ │                  │  │                  │  │               │  │
│ │ Costruisce la    │  │ Sfida ogni       │  │ Fornisce      │  │
│ │ tesi migliore    │  │ affermazione:    │  │ background    │  │
│ │ a favore della   │  │ "Questa norma    │  │ dottrinale:   │  │
│ │ risposta.        │  │  e' abrogata?"   │  │ orientamenti, │  │
│ │                  │  │ "Questo          │  │ principi,     │  │
│ │ Tool: normattiva │  │  principio si    │  │ interpretaz.  │  │
│ │ eurlex, kb_norm  │  │  applica?"       │  │               │  │
│ │                  │  │                  │  │ Tool: doctrine│  │
│ │ Modello:         │  │ Tool: vigenza,   │  │ kb_search,    │  │
│ │ lexe-primary     │  │ juris_verifier   │  │ lex_search    │  │
│ │                  │  │                  │  │               │  │
│ │                  │  │ Modello:         │  │ Modello:      │  │
│ │                  │  │ lexe-frontier    │  │ lexe-complex  │  │
│ └────────┬─────────┘  └────────┬─────────┘  └───────┬───────┘  │
│          │                     │                     │          │
│          └─────────────────────┼─────────────────────┘          │
│                                │                                │
│                    ROUND 1 → ROUND 2 → ROUND 3                 │
│                    (ogni agente vede output degli altri)        │
│                                │                                │
└────────────────────────────────┼────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────┐
│ JUDGE AGENT (Giudice)                                         │
│ Modello: lexe-frontier (reasoning=high)                       │
│                                                                │
│ Funzioni:                                                      │
│ - Valuta la completezza del dibattito                         │
│ - Verifica che ogni citazione sia grounded                    │
│ - Assegna Solidarity Score                                    │
│ - Produce la sentenza finale (risposta strutturata)           │
│ - Dichiara il livello di certezza                             │
│                                                                │
│ Policy: REQUIRE_GROUNDED_CITATION — ogni affermazione deve    │
│         essere supportata da almeno 1 fonte verificata        │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
                   SSE Response
```

**Vantaggi:**
- Naturalmente adversarial — il Critic Agent sfida ogni affermazione, riducendo allucinazioni
- Mirror del processo legale reale (avvocato, contraddittore, giudice)
- Grounded: ogni agente ha accesso ai tool e deve citare fonti
- Il Judge Agent ha l'ultima parola — singolo punto di decisione

**Svantaggi:**
- Latency: 3 round × 3 agenti = 9 LLM calls minimo (vs 4 nel PRVS)
- Costo: ~3× il costo attuale per query STANDARD
- Complessita': gestire stato condiviso tra round di debate
- Rischio di convergenza prematura se gli agenti sono troppo simili

**Riferimenti paper:**
- L4L (arXiv 2511.21033) — adversarial agents + SMT verification
- DebateCV (WWW '26) — two debaters + judge
- Tool-MAD (arXiv 2601.04742) — debate con tool retrieval
- Multiagent Debate (Du et al., ICML 2024) — foundational debate paper

---

### VARIANTE B — "STUDIO LEGALE" (Hierarchical Multi-Agent Orchestration)

> Ispirata a: L-MARS, DAAO, GoalAct, AgentSpawn, CrewAI

```
User Query
  │
  ▼
┌──────────────────────────────────────────────────────────────┐
│ MANAGING PARTNER (Orchestratore Centrale)                     │
│ Modello: lexe-frontier (reasoning=high)                       │
│                                                                │
│ Funzioni:                                                      │
│ - Analisi profonda della query (non solo intent detection)    │
│ - Decomposizione in sub-task con dipendenze (DAG)             │
│ - Stima difficolta' (learned, non rule-based)                 │
│ - Assegnazione task agli associate specializzati              │
│ - Monitoring progresso e budget                               │
│ - Decisione di spawning agenti aggiuntivi se necessario       │
│ - Fusione risultati e produzione risposta finale              │
└──────────────────────┬───────────────────────────────────────┘
                       │
          ┌────────────┼────────────┬───────────┐
          │            │            │           │
          ▼            ▼            ▼           ▼
┌──────────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐
│ NORM AGENT   │ │ JURIS    │ │ DOCTRINE │ │ PRINCIPLE    │
│ (Normativista)│ │ AGENT    │ │ AGENT    │ │ AGENT        │
│              │ │(Giurispr)│ │(Dottrina)│ │(Principi)    │
│              │ │          │ │          │ │              │
│ Specializzato│ │ Cerca    │ │ Cerca    │ │ Identifica   │
│ in ricerca   │ │ massime, │ │ commenti,│ │ principi     │
│ normativa:   │ │ sentenze,│ │ note,    │ │ generali,    │
│ codici,      │ │ orienta- │ │ ratio    │ │ costituz.,   │
│ leggi spec., │ │ menti    │ │ decidendi│ │ EU, clausole │
│ regolamenti  │ │          │ │          │ │ generali     │
│              │ │ Tool:    │ │ Tool:    │ │              │
│ Tool:        │ │ kb_search│ │ doctrine │ │ Tool:        │
│ normattiva,  │ │ juris_   │ │ _search, │ │ principle_   │
│ eurlex,      │ │ verifier │ │ lex_     │ │ mapper,      │
│ kb_normativa │ │          │ │ search   │ │ kb_normativa │
│              │ │ Modello: │ │          │ │              │
│ Modello:     │ │ lexe-    │ │ Modello: │ │ Modello:     │
│ lexe-primary │ │ primary  │ │ lexe-    │ │ lexe-primary │
│              │ │          │ │ primary  │ │              │
└──────┬───────┘ └────┬─────┘ └────┬─────┘ └──────┬───────┘
       │              │            │               │
       └──────────────┼────────────┼───────────────┘
                      │            │
                      ▼            ▼
            ┌──────────────────────────────────┐
            │ EVIDENCE FUSION LAYER             │
            │                                    │
            │ - Merge evidence packs             │
            │ - Dedup (same norm/massima)        │
            │ - Conflict detection               │
            │ - Source authority ranking          │
            │ - Citation graph expansion         │
            └──────────────┬─────────────────────┘
                           │
                           ▼
            ┌──────────────────────────────────┐
            │ QUALITY GATE                      │
            │ (Verifier + Self-Correction)      │
            │                                    │
            │ - Structural verification          │
            │ - Citation grounding check         │
            │ - Vigenza verification             │
            │ - Coherence assessment             │
            │ - If FAIL: → loop back to agents   │
            │   con feedback specifico           │
            │                                    │
            │ Modello: lexe-verifier             │
            │ Max retry: 2                       │
            └──────────────┬─────────────────────┘
                           │
                           ▼
            ┌──────────────────────────────────┐
            │ SENIOR PARTNER (Synthesis)        │
            │                                    │
            │ - Riceve evidence fusa e verificata│
            │ - Produce risposta strutturata     │
            │ - Scenario-aware template          │
            │ - Streaming SSE                    │
            │                                    │
            │ Modello: lexe-frontier             │
            └──────────────────────────────────┘
```

**Vantaggi:**
- Agenti specializzati: ogni agente e' esperto nel suo dominio
- Parallelismo: Norm/Juris/Doctrine/Principle agents girano in parallelo
- Scalabile: il Managing Partner puo' spawning agenti aggiuntivi runtime (AgentSpawn)
- Evidence Fusion: merge intelligente da fonti diverse
- Self-correction loop: il Quality Gate puo' rimandare agli agenti

**Svantaggi:**
- Single point of failure: il Managing Partner deve essere molto capace
- Overhead di coordinazione: merge evidence da 4 agenti e' complesso
- Nessun meccanismo adversarial: gli agenti non si sfidano tra loro
- Il Managing Partner potrebbe non catturare conflitti tra le fonti

**Riferimenti paper:**
- L-MARS (arXiv 2509.00761) — query decomposition + heterogeneous search + Judge
- DAAO (arXiv 2509.11079) — difficulty-aware workflow generation
- GoalAct (arXiv 2504.16563) — continuously updated global planning
- AgentSpawn (arXiv 2602.07072) — dynamic agent spawning
- CrewAI — role-based orchestration pattern

---

### VARIANTE C — "CORTE D'APPELLO" (Hybrid: Orchestration + Debate + Reflection)

> Ispirata a: L-MARS + Tool-MAD + MAR + Confidence-Aware Routing

**Questa e' la variante raccomandata.** Combina i vantaggi delle Varianti A e B eliminando i rispettivi svantaggi.

```
User Query
  │
  ▼
┌──────────────────────────────────────────────────────────────┐
│ CORTE AGENT (Learned Router)                                  │
│ Modello: fine-tuned classifier (non LLM) — ~50ms             │
│                                                                │
│ Funzioni:                                                      │
│ - Classifica difficolta' (learned, non regex)                 │
│ - Classifica evidence type (NORM/PRINCIPLE/JURIS/DOCTRINE)    │
│ - Seleziona formazione:                                       │
│   SIMPLE → Fast Track (singolo agente, v3 pipeline)           │
│   STANDARD → Full Bench (3 agenti paralleli + debate 1 round)│
│   COMPLEX → Gran Camera (4 agenti + debate 2 round + reflect)│
│ - Budget adattivo basato su confidence mid-stream             │
│                                                                │
│ Zero regex: il classifier e' trainato su 500+ query annotate  │
└──────────────────────┬───────────────────────────────────────┘
                       │
         ┌─────────────┼─────────────────┐
         │             │                 │
         ▼             ▼                 ▼
   ┌──────────┐  ┌───────────┐  ┌────────────────┐
   │FAST TRACK│  │FULL BENCH │  │GRAN CAMERA     │
   │(SIMPLE)  │  │(STANDARD) │  │(COMPLEX)       │
   └─────┬────┘  └─────┬─────┘  └───────┬────────┘
         │             │                 │
         ▼             ▼                 ▼

═══════════════════════════════════════════════════════
FAST TRACK (query semplici, ~15s)
═══════════════════════════════════════════════════════
  Single Agent + Tool Loop (identico a v3 PRVS MINIMAL)
  → Risposta diretta
  → Fallback: v3 pipeline

═══════════════════════════════════════════════════════
FULL BENCH (query standard, ~30s)
═══════════════════════════════════════════════════════

Phase 1: PARALLEL RESEARCH (3 agenti, ~10s)
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ NORM AGENT    │ │ JURIS AGENT   │ │ CONTEXT AGENT │
│               │ │               │ │               │
│ normattiva,   │ │ kb_search,    │ │ doctrine,     │
│ eurlex,       │ │ juris_verify, │ │ lex_search,   │
│ kb_normativa  │ │ graph expand  │ │ memory, docs  │
│               │ │               │ │               │
│ → NormPack    │ │ → JurisPack   │ │ → ContextPack │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                          ▼
Phase 2: EVIDENCE FUSION + MINI-DEBATE (1 round, ~8s)
┌──────────────────────────────────────────────────────┐
│ FUSION AGENT: merge, dedup, conflict detection       │
│                                                        │
│ If conflicts detected:                                 │
│   ADVOCATE: "Art. 2043 cc si applica perche'..."      │
│   CRITIC:   "Ma l'orientamento SS.UU. 2018 dice..."  │
│   → Risoluzione in 1 round                            │
│                                                        │
│ → Fused EvidencePack                                   │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
Phase 3: VERIFY + SYNTHESIZE (~12s)
┌──────────────────────────────────────────────────────┐
│ VERIFIER: grounding check, vigenza, coherence        │
│ SYNTHESIZER: scenario-aware streaming                │
│                                                        │
│ If confidence < threshold:                             │
│   → Reflect: "Cosa manca? Quali fonti non ho cercato?"│
│   → Loop back a Phase 1 con query mirate              │
│   → Max 1 retry                                       │
└──────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════
GRAN CAMERA (query complesse, ~60s)
═══════════════════════════════════════════════════════

Phase 1: DEEP RESEARCH (4 agenti paralleli, ~15s)
  Norm + Juris + Doctrine + Principle agents
  Ognuno con budget esteso (10 tool calls)

Phase 2: ADVERSARIAL DEBATE (2 round, ~15s)
  Round 1: Advocate vs Critic (con tool access)
  Round 2: Advocate ribatte + Scholar contribuisce
  → Transcript del dibattito

Phase 3: JUDGE + REFLECTION (~15s)
  JUDGE AGENT valuta il dibattito
  Se Judge score < 7.0/10:
    → REFLECTION: cross-agent reflection (MAR)
    → Agenti riflettono sugli errori reciproci
    → Phase 2 ripetuta con feedback (max 1 retry)

Phase 4: SYNTHESIS (~15s)
  SENIOR PARTNER sintetizza basandosi su:
  - Evidence fusa e verificata
  - Transcript del dibattito
  - Verdict del Judge
  - Reflection notes (se applicate)
  → Risposta strutturata con massima qualita'
```

**Vantaggi:**
- **Adattivo**: query semplici → fast track (15s), complesse → debate pieno (60s)
- **Adversarial dove serve**: il debate si attiva solo per STANDARD+ con conflitti
- **Grounded**: ogni agente ha tool access, niente ragionamento parametrico puro
- **Self-correcting**: reflection loop rompe i cicli di errore
- **Zero regex**: classifier learned, non rule-based
- **Retrocompatibile**: Fast Track = v3 pipeline, zero regressione
- **Cost-efficient**: ~1.5× per STANDARD, ~3× solo per COMPLEX (vs 3× per tutto in Variante A)

**Svantaggi:**
- Complessita' di implementazione significativa
- Il learned router richiede un dataset di training (500+ query annotate)
- La gestione dello stato tra round di debate e' non banale
- Testing richiede gold set per ogni formazione (Fast/Full/Gran)

**Riferimenti paper:**
- L-MARS (arXiv 2509.00761) — decomposition + heterogeneous search
- Tool-MAD (arXiv 2601.04742) — debate con tool retrieval
- MAR (arXiv 2512.20845) — multi-agent reflexion
- Confidence-Aware Routing (arXiv 2601.04861) — per-turn model escalation
- DAAO (arXiv 2509.11079) — difficulty-aware orchestration
- Self-MoA (arXiv 2502.00674) — single-model aggregation per consistency

---

## PARTE 4 — CONFRONTO VARIANTI

### 4.1 Matrice Comparativa

| Dimensione | v3 (Attuale) | A: Tribunale | B: Studio Legale | C: Corte d'Appello |
|---|---|---|---|---|
| **Paradigma** | Pipeline sequenziale | Adversarial debate | Hierarchical orchestration | Hybrid adaptive |
| **Agenti** | 0 (funzioni) | 4 (Clerk+Advocate+Critic+Judge) | 5 (Partner+4 Associates) | 3-5 (adattivo) |
| **Regex** | Pesante | Zero | Zero | Zero |
| **Adversarial** | No | Si (core) | No | Si (on-demand) |
| **Self-correction** | No | No | Si (Quality Gate loop) | Si (Reflection) |
| **Latency SIMPLE** | ~15s | ~30s (overkill) | ~20s | ~15s (Fast Track) |
| **Latency STANDARD** | ~35s | ~45s | ~35s | ~30s |
| **Latency COMPLEX** | ~50s | ~60s | ~55s | ~60s |
| **Cost multiplier** | 1× | 3× | 2× | 1.5× avg |
| **Hallucination risk** | Medio | Basso | Medio-Basso | Basso |
| **Implementation** | Fatto | 8 settimane | 10 settimane | 12 settimane |
| **Fallback** | N/A | v3 | v3 | v3 (Fast Track = v3) |

### 4.2 Raccomandazione

**Variante C ("Corte d'Appello")** e' la scelta raccomandata perche':

1. **Non spreca risorse su query semplici** — Fast Track e' identico a v3, zero overhead
2. **Debate solo dove serve** — conflitti nelle fonti, query complesse, zero-norm
3. **Self-correction** — il reflection loop rompe i cicli di errore che v3 non puo' correggere
4. **Zero regex** — il learned router e' la base per eliminare tutti i pattern matching hardcoded
5. **Costo controllato** — 1.5× in media, non 3× come la Variante A
6. **Retrocompatibile** — il Fast Track E' la pipeline v3, garantisce zero regressione

### 4.3 Roadmap Implementativa (Variante C)

| Sprint | Durata | Deliverable |
|---|---|---|
| Sprint 15 | 2 settimane | Learned Router (classifier trainato su 500+ query dal Kryptonite + zero-norm dataset) |
| Sprint 16 | 2 settimane | Agent Framework (Norm/Juris/Context agents paralleli, Evidence Fusion) |
| Sprint 17 | 2 settimane | Mini-Debate (Advocate + Critic, 1 round, tool-grounded) |
| Sprint 18 | 2 settimane | Quality Gate + Reflection Loop |
| Sprint 19 | 2 settimane | Gran Camera (2-round debate + Judge Agent) |
| Sprint 20 | 2 settimane | Canary rollout + regression testing |

**Prerequisiti:**
- NOVA RATIO (Sprint 14) validato su staging ← **gia' fatto**
- Dataset annotato 500+ query per training del router
- Gold set per ogni formazione (Fast/Full/Gran Camera)

---

## PARTE 5 — GLOSSARIO DEI PAPERS CITATI

| Ref | Paper | Venue | Anno | arXiv/URL |
|---|---|---|---|---|
| [1] | Multi-Agent Collaboration via Evolving Orchestration | arXiv | 2025 | 2505.19591 |
| [2] | Difficulty-Aware Agentic Orchestration (DAAO) | arXiv | 2025 | 2509.11079 |
| [3] | L-MARS: Legal Multi-Agent with Orchestrated Reasoning | arXiv | 2025 | 2509.00761 |
| [4] | SOLAR: Structured Ontological Legal Analysis Reasoner | CIKM 2025 | 2025 | 2509.00710 |
| [5] | L4L: Trustworthy Legal AI via LLM Agents + Formal Reasoning | arXiv | 2025 | 2511.21033 |
| [6] | GoalAct: Global Planning + Hierarchical Execution | NCIIP 2025 | 2025 | 2504.16563 |
| [7] | AgentSpawn: Dynamic Spawning for Long-Horizon Tasks | arXiv | 2026 | 2602.07072 |
| [8] | SALLMA: Software Architecture for LLM-Based MAS | ICSE 2025 | 2025 | IEEE 11029425 |
| [9] | MASLegalBench: Benchmarking MAS in Legal Reasoning | arXiv | 2025 | 2509.24922 |
| [10] | LRAS: Advanced Legal Reasoning with Agentic Search | arXiv | 2026 | 2601.07296 |
| [11] | Improving Factuality through Multiagent Debate | ICML 2024 | 2023 | 2305.14325 |
| [12] | DebateCV: Debate-driven Claim Verification | WWW '26 | 2026 | 2507.19090 |
| [13] | Tool-MAD: Multi-Agent Debate with Tool Use | arXiv | 2026 | 2601.04742 |
| [14] | A-HMAD: Adaptive Heterogeneous Multi-Agent Debate | J. King Saud | 2025 | Springer |
| [15] | MAD-Fact: Multi-Agent Debate for Long-Form Factuality | arXiv | 2025 | 2510.22967 |
| [16] | Multi-Agent Reflexion (MAR) | arXiv | 2025 | 2512.20845 |
| [17] | Reflexion: Language Agents with Verbal Reinforcement | NeurIPS 2023 | 2023 | 2303.11366 |
| [18] | Self-Refine: Iterative Refinement with Self-Feedback | arXiv | 2023 | 2303.17651 |
| [19] | Agent-R: Training Agents to Reflect via MCTS | arXiv | 2025 | 2501.11425 |
| [20] | Self-MoA / Rethinking Mixture-of-Agents | NeurIPS 2025 | 2025 | 2502.00674 |
| [21] | Mixture-of-Agents (MoA) | Together AI | 2024 | 2406.04692 |
| [22] | Confidence-Aware Routing for Multi-Scale Models | arXiv | 2026 | 2601.04861 |
| [23] | MasRouter: Learning to Route for MAS | ACL 2025 | 2025 | ACL Anthology |
| [24] | SYMPHONY: Synergistic Multi-agent Planning | NeurIPS 2025 | 2025 | OpenReview |
| [25] | Google A2A Protocol | Google | 2025 | a2a-protocol.org |
| [26] | Anthropic MCP | Anthropic / Linux Foundation | 2024-2025 | modelcontextprotocol.io |
| [27] | Microsoft Agent Framework (AutoGen + Semantic Kernel) | Microsoft | 2025 | learn.microsoft.com |
| [28] | OpenAI Agents SDK | OpenAI | 2025 | github.com/openai |
| [29] | LangGraph | LangChain | 2024-2025 | langchain.com/langgraph |
| [30] | CrewAI | CrewAI Inc | 2024-2025 | crewai.com |
| [31] | Legal Agents Survey | OAE Publishing | 2025 | oaepublish.com |
| [32] | Google Scaling Principles for Agentic Architectures | InfoQ | 2026 | infoq.com |

---

*IT Consulting SRL — LEXe Platform*
*"Il Manus del Legal AI"*
*Draft 1.0 — 22 Marzo 2026*
