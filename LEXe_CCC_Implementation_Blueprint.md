# LEXe - Piano Esecutivo di Implementazione
## Documento di Supporto per Claude Code CLI (CCC)

> **Versione:** 3.1 — CORRETTA vs architettura reale + verifica 21/02
> **Data:** 21/02/2026
> **Autore:** LEXe.OS - Orchestratore Operativo (v2.0) + Claude Code audit (v3.0)
> **Destinazione:** Claude Code CLI — team di agenti paralleli
> **Strategia selezionata:** BALANCED (B) - 4 sprint, 8 settimane
> **Ref architetturale:** `lexe-docs/ARCHITECTURE-REFERENCE.md` (source of truth)
> **Fonti originali:** 5 analisi AI parallele + 4 esecuzioni multi-scenario + ricerca web
> **Audit v3.1:** Verifica codebase completa (14 LEXE + 2 shared + 2 esterni = 18 servizi, 24 agent files, 13 migrations, 8 tools)

---

## 0. COME USARE QUESTO DOCUMENTO

Questo documento e la **Single Source of Truth** per CCC. Contiene tutto cio che serve per implementare le correzioni e i miglioramenti di LEXe in modo autonomo, senza ambiguita.

**Struttura:**
- Sezione 1: Contesto architetturale (cosa esiste oggi)
- Sezione 2: Problemi ordinati per criticita con soluzioni definitive
- Sezione 3: Roadmap sprint-by-sprint con ore, owner e criteri di accettazione
- Sezione 4: Metriche di gate e criteri go/no-go
- Sezione 5: Decisioni architetturali (ADR) e rationale
- Sezione 6: Innovazioni a medio termine
- Sezione 7: Checklist pre-implementazione
- Sezione 8: Convenzioni di codice e stile

**Regola d'oro:** ogni modifica deve essere protetta da feature flag. Se una metrica di gate non viene superata, si fa rollback del singolo componente senza impattare gli altri.

---

## 1. CONTESTO ARCHITETTURALE

> **Ref:** `lexe-docs/ARCHITECTURE-REFERENCE.md` per dettagli completi

### 1.1 Stack Tecnologico Corrente (verificato 2026-02-21)

#### LEXE Platform Services (14 container)

| Componente | Tecnologia | Porta ext→int | Ruolo |
|---|---|---|---|
| Chat Interface | **React 18 + Vite + TypeScript** (lexe-webchat) | 3013→80 | Frontend utente custom |
| Admin Panel | **React + Vite** (lexe-admin) | 3014→80 | Pannello admin separato |
| Backend API | FastAPI Python 3.11+ (lexe-core) | 8100→8100 | Gateway, Auth, Identity, Agent pipelines |
| Cache/Sessioni | **Valkey 8** (lexe-valkey) | 6381→6379 | L0/L1 memory, session store |
| DB Sistema | PostgreSQL 17 + pgvector (lexe-postgres) | 5435→5432 | core, memory, Logto, LiteLLM, Temporal |
| DB KB Legal | **PostgreSQL 17 + pgvector + Apache AGE + ParadeDB** (lexe-max) | 5436→5432 | Normativa, massime, embeddings, graph |
| LLM Gateway | LiteLLM (via OpenRouter) (lexe-litellm) | 4001→4000 | Routing multi-modello, cost tracking |
| Memory System | FastAPI Python (lexe-memory) | 8103→8103 | Memory L0-L4 |
| Legal Tools | FastAPI Python (lexe-tools-it) | 8021→8021 | 8 tool legali IT |
| Orchestrator | FastAPI Python (lexe-orchestrator) | 8102→8102 | ORCHIDEA Pipeline (**DISABLED default**) |
| Workflow | **Temporal 1.24** (lexe-temporal) | 7234→7233 | Workflow orchestration |
| Workflow UI | **Temporal UI 2.26** (lexe-temporal-ui) | 8180→8080 | Temporal dashboard |
| Auth | **Logto** (lexe-logto) | 3304→3001, 3305→3002 | CIAM (app + admin console) |
| Embedding | text-embedding-3-small via OpenRouter (lexe-embedding) | — | 1536 dimensioni |

#### Shared Infrastructure (2 container — condivisi con LEO platform)

| Componente | Tecnologia | Porta ext→int | Ruolo | Deploy |
|---|---|---|---|---|
| Reverse Proxy | **Traefik v2.11** (shared-traefik) | 80, 443, 8081→8080 | TLS termination, routing, Let's Encrypt | `shared-infra/docker-compose.yml` |
| Container Mgmt | **Portainer CE 2.21** (shared-portainer) | 9002→9000, 9445→9443 | Container management UI | `shared-infra/docker-compose.yml` |

#### Servizi Esterni Referenziati (NON in git, deployati su host)

| Componente | Hostname | Porta | Ruolo | Usato da |
|---|---|---|---|---|
| Web Search | **SearXNG** (shared-searxng) | 8080 | Motore di ricerca web | lexe-tools, lexe-core, lexe-orchestrator |
| Tracing | **Jaeger** (shared-jaeger) | 4317 | OpenTelemetry (opzionale) | lexe-core |

#### Servizi Disabilitati

| Componente | Motivo |
|---|---|
| lexe-memory-worker | Temporal worker non ancora implementato |

**Totale: 14 container LEXE + 2 shared + 2 esterni = 18 servizi.** NO Qdrant, NO Neo4j, NO Redis, NO Open WebUI — tutto PostgreSQL-centrico.

#### Network Topology

| Network | Tipo | Scopo |
|---|---|---|
| `lexe_internal` | Bridge (isolata) | Comunicazione interna tra servizi LEXE |
| `shared_public` | Bridge (esterna) | Condivisa con LEO, esposta via Traefik |

#### Traefik Routing (label-based)

| URL Staging | URL Produzione | Target |
|---|---|---|
| stage-chat.lexe.pro | ai.lexe.pro | lexe-webchat:80 |
| api.stage.lexe.pro | api.lexe.pro | lexe-core:8100 |
| auth.stage.lexe.pro | auth.lexe.pro | lexe-logto:3001 |
| logto-admin.stage.lexe.pro | auth-admin.lexe.pro | lexe-logto:3002 |
| admin.stage.lexe.pro | — | lexe-admin:80 |
| llm.stage.lexe.pro | llm.lexe.pro | lexe-litellm:4000 |

### 1.2 Pipeline di Elaborazione Query

**STATO ATTUALE:** Per default, tutte le pipeline avanzate sono **DISABILITATE**.
Il path attivo e il direct LiteLLM streaming (toolloop) con tool calling.

```
Utente -> lexe-webchat (React) -> POST /api/v1/gateway/customer/stream
                                         |
                                  [ff_orchestrator_enabled?]
                                    NO (default) |  YES -> lexe-orchestrator:8102
                                         |
                                  [ff_legis_agent?]
                                    NO (default) |  YES -> _pre_intent_route()
                                         |              |
                                  [Direct LiteLLM    [classify_complexity() via LLM]
                                   streaming +        |
                                   tool calls]   +----+----+----+
                                  (TOOLLOOP)     |    |    |    |
                                              FLASH STD DEEP STRATEGIC
                                                |    |    |    |
                                              toolloop legis lexorc(if enabled)
                                                       |    |
                                               [LEGIS  |  [LEXORC 6 componenti
                                                Adapt] |   (3 agenti paralleli)]
                                                  |
                                          Phase 0: Intent Detection (lexe-fast)
                                                  |
                                          +-------+-------+-------+
                                          |       |       |       |
                                        DIRECT  SIMPLE STANDARD COMPLEX
                                          |       |       |       |
                                        skip P  P(fast) P(fast) P(reasoning)
                                        skip V  skip V  V full  V+coherence
                                        S(min)  S(full) S(full) S(full+deep)
```

**Feature flag attuali (tutti False di default):**
- `ff_legis_agent` → abilita LEGIS (Intent→Plan→Research→Verify→Synthesize)
- `ff_lexorc_enabled` → abilita LEXORC multi-agente (6 componenti: Classifier + 3 agenti paralleli + Auditor + Synthesizer)
- `ff_orchestrator_enabled` → abilita ORCHIDEA (8 fasi, Temporal-backed)

**Fasi LEGIS:** Phase 0 (Intent Detection) → P(lanning) → R(esearch) → V(erification) → S(ynthesis)
**Livelli LEGIS:** DIRECT (skip P+V) | SIMPLE (P fast, skip V) | STANDARD (P fast, V full) | COMPLEX (P reasoning, V+coherence)
**Scenari LEGIS:** ricerca | contratto_redazione | contratto_revisione | parere | contenzioso | recupero_crediti | custom
**Componenti LEXORC:** Classifier - NormAgent - CaselawAgent - DoctrineAgent - Auditor - Synthesizer

### 1.3 Knowledge Base Proprietaria (lexe-max)

**lexe-max e un container PostgreSQL 17** con estensioni specializzate (NON un servizio separato).

| Risorsa | Quantita | Storage | Note |
|---|---|---|---|
| Massime giurisprudenziali (`kb.massime`) | 38.718 active (46.767 totali) | PostgreSQL + pgvector | anni 1913-2929 |
| Norme (`kb.norms`) | 4.128 | PostgreSQL | LEGGE=1257, CC=995, CPC=520, DLGS=412, CPP=298, DL=229, CP=192 |
| Citation graph edges (`kb.graph_edges`) | 58.737 (CITES) | **Apache AGE** (graph PostgreSQL) | run_id=1 attivo |
| Norm-massima links (`kb.massima_norms`) | 42.338 | PostgreSQL | |
| Embeddings (`kb.embeddings`) | 41.437 | pgvector (1536d) | |
| Articoli normativa (`kb.normativa`) | ~4.117 (staging) | PostgreSQL | CC=3170, CP=947 |
| Category predictions (`kb.category_predictions_v2`) | 279.887 | PostgreSQL | |
| Sections (`kb.sections`) | 336 | PostgreSQL | |
| Categories (`kb.categories`) | 51 | PostgreSQL | |
| Hybrid search | RRF: score(d) = sum(1/(k + r_i(d))) | Dense HNSW + ParadeDB BM25 + Graph boost | |

**ATTENZIONE — Tabelle documentate ma NON presenti su staging:**
- `kb.work` (codici/leggi configurati) — NON esiste
- `kb.normativa_chunk` (chunks con embeddings) — NON esiste
- `kb.annotation` (annotazioni Brocardi) — NON esiste
- Queste tabelle potrebbero essere presenti solo in produzione o non ancora migrate.

### 1.4 Modelli LLM Configurati (litellm/config.yaml)

| Alias | Modello Sottostante | Uso | Note |
|---|---|---|---|
| lexe-fast | **Gemini 2.5 Flash** (openrouter) | Intent detection, follow-up, classificazione | |
| lexe-primary | **Mistral Large 2512** (openrouter) | Chat, synthesis, ricerca LEGIS/LEXORC | |
| lexe-complex | **GPT-5.2** (openrouter) | Reasoning, LEGIS planner | |
| legal-tier1-gemini | **Gemini 2.5 Flash** (openrouter) | Tier 1 fast | Alias di lexe-fast |
| legal-tier2-gemini | **Gemini 2.5 Flash** (openrouter) | Tier 2 | Alias di lexe-fast |
| legal-tier2-haiku | **Claude Haiku 4.5** (openrouter) | Auditor, verifier | |
| legal-tier3-sonnet | **Claude Sonnet 4.6** (openrouter) | High verification | |
| legal-tier3-gpt5 | **GPT-5.2** (openrouter) | Tier 3 frontier | Alias di lexe-complex |
| lexe-embedding | **text-embedding-3-small** (openrouter) | Embedding 1536d | |

**NOTA:** 3 alias duplicati (`legal-tier1-gemini`, `legal-tier2-gemini` → stesso modello di `lexe-fast`; `legal-tier3-gpt5` → stesso modello di `lexe-complex`). Vedi P3 per cleanup proposto.

### 1.5 File Chiave del Codebase (verificati)

```
lexe-core/src/lexe_core/
  config.py                    # Settings + 12 feature flags + model assignments
  main.py                      # FastAPI app + CORS
  gateway/
    customer_router.py         # Main SSE streaming + routing (_pre_intent_route)
    streaming/                 # SSE protocol, keepalive, limits, state
    services/llm_config.py     # LLM config resolution da DB
  agent/
    classifier.py              # classify_complexity() -> FLASH/STANDARD/DEEP/STRATEGIC
    intent_detector.py         # run_intent_detector() -> PipelineLevel + ScenarioType
    pipeline.py                # LEGIS pipeline orchestration
    planner.py                 # Generazione piano di ricerca (Fase P)
    researcher.py              # Esecuzione tool parallela (Fase R)
    verifier.py                # Verifica anti-hallucination (Fase V)
    synthesizer.py             # Generazione risposta con evidence (Fase S)
    advanced_orchestrator.py   # LEXORC orchestrator
    agents/                    # LEXORC: norm_agent, caselaw_agent, doctrine_agent, auditor, synthesizer
    blackboard.py              # Shared state Valkey-backed
    parallel_engine.py         # Esecuzione agenti parallela
    scoring.py                 # Confidence scoring
    models.py                  # PipelineLevel, ScenarioType, IntentResult
  prompts/v1/                  # 13 prompt template files (planner, researcher, synthesizer, verifier, etc.)
  # NOTA: prompts/v1/ coesiste con agent/prompts.py (608 LOC)
  # agent/prompts.py contiene: INTENT_DETECTOR_SYSTEM_PROMPT, PLANNER_SYSTEM_PROMPT,
  # SYNTHESIZER_SYSTEM_PROMPT, VERIFIER_SYSTEM_PROMPT, 7 scenario-specific synth prompts
  tools/
    executor.py                # Esecuzione tool
    policy.py                  # Policy resolution (tenant -> persona -> fallback)
    logging.py                 # Tool execution logging (non-blocking)
    definitions.py             # Tool schema definitions
    health.py                  # Tool health probes
  admin/
    routers/                   # 14 router (me, tenants, settings, models, tools, personas, catalogs, etc.)
    services/                  # 11 service (catalog, litellm_keys, dashboard, tenant, persona, etc.)
  identity/
    schemas/__init__.py        # TenantResponse, TenantCreate, TenantUpdate
    service.py                 # Tenant management + copy-on-create
  migrations/                  # 13 SQL files (001-013), prossima: 014

lexe-tools-it/src/lexe_tools_it/tools/
  normattiva.py                # Normattiva OpenData API
  eurlex.py                    # EUR-Lex
  infolex.py                   # InfoLex
  lex_search.py                # Legal search
  kb_search.py                 # KB semantic search
  one_search.py                # Unified search
  vigenza_fast.py              # Currency checking

lexe-memory/src/lexe_memory/
  layers/l0-l4                 # 5 memory layers (Valkey + PostgreSQL + AGE)
  rag/                         # RAG system
  profile/                     # Brainprint

lexe-infra/
  docker-compose.yml           # 14 servizi LEXE
  docker-compose.override.*.yml # Stage/Prod overrides (OBBLIGATORI)
  litellm/config.yaml          # Catalogo modelli LiteLLM (9 alias)

shared-infra/
  docker-compose.yml           # 2 servizi shared (Traefik + Portainer)
```

---

## 2. PROBLEMI E SOLUZIONI DEFINITIVE

I problemi sono ordinati per criticita di implementazione. Ogni soluzione include il codice finale, i file da modificare e i criteri di accettazione.

### 2.1 [P6] Token Limit lexe-fast - CRITICO, IMMEDIATO

**Problema:** `max_completion_tokens` per `lexe-fast` potrebbe essere troppo basso. L'output JSON dell'intent detector rischia troncamento.

**Causa root:** configurazione iniziale conservativa nel catalogo modelli.

**NOTA v3.0:** La tabella corretta e `core.llm_model_catalog` (NON `core.model_catalog`).
Il campo config e `config JSONB` aggiunto in migration 010. La prossima migration disponibile e **014**.

**Soluzione:**

```sql
-- File: lexe-core/migrations/014_fix_lexe_fast_tokens.sql
-- Descrizione: Aumenta token limit per lexe-fast
-- Rollback: UPDATE core.llm_model_catalog SET config = jsonb_set(COALESCE(config,'{}'), '{max_completion_tokens}', '250') WHERE model_name = 'lexe-fast';

UPDATE core.llm_model_catalog
SET config = jsonb_set(COALESCE(config, '{}'), '{max_completion_tokens}', '1024')
WHERE model_name = 'lexe-fast';
```

**Criteri di accettazione:**
- Zero parse error su 100 query di test dopo migrazione
- Nessun aumento significativo del costo per query (delta < 5%)
- Tempo di risposta intent detector invariato (delta < 100ms)

**Effort:** 2h | **Sprint:** 9 | **Owner:** Frisco

---

### 2.2 [P1] Double Classifier - ALTO (non CRITICO)

**Problema:** Quando `ff_legis_agent=True`, la funzione `_pre_intent_route()` in `customer_router.py:954` chiama `classify_complexity()`. Poi il LEGIS pipeline chiama separatamente `run_intent_detector()`. Due LLM call per il routing.

**NOTA v3.0:** Questi NON sono la stessa funzione:
- `classify_complexity()` (in `agent/classifier.py`) → ritorna FLASH/STANDARD/DEEP/STRATEGIC (complessita)
- `run_intent_detector()` (in `agent/intent_detector.py`) → ritorna PipelineLevel + ScenarioType (intent + scenario)
Sono stati aggiunti in sprint diversi (classifier prima, intent detector in Sprint 8/mig 013).
L'unificazione e corretta ma bisogna preservare entrambi gli output.

**Causa root:** evoluzione incrementale (classifier pre-esistente, intent detector aggiunto dopo).

**Soluzione definitiva - Fase A: Pre-filtro deterministico migliorato**

```python
# File: lexe-core/src/lexe_core/gateway/customer_router.py
# Funzione: _pre_intent_route() (linea ~954) - SOSTITUIRE interamente

_LEGAL_HINTS = frozenset({
    "art.", "articolo", "comma", "legge", "decreto", "sentenza",
    "cassazione", "codice", "tribunale", "corte", "giurisprudenza",
    "responsabilita", "risarcimento", "contratto", "inadempimento",
    "reato", "dolo", "colpa", "prescrizione", "ricorso", "appello",
    "procedura", "competenza", "giurisdizione", "notifica",
    "esecuzione", "pignoramento", "fallimento", "concordato",
    "successione", "donazione", "compravendita", "locazione",
    "appalto", "mandato", "fideiussione", "ipoteca", "servitu",
    "usucapione", "danno", "illecito", "responsabilita",
    "d.lgs", "d.l.", "l.n.", "c.c.", "c.p.", "c.p.c.", "c.p.p.",
    "t.u.", "d.p.r.", "d.m.", "reg.", "direttiva", "regolamento"
})

def _pre_intent_route(
    message: str,
    litellm_url: str,
    litellm_api_key: str,
    tenant_id: str
) -> str:
    """Filtro deterministico a costo zero per bypassare LLM su query banali.

    NOTA: La firma reale include litellm_url, litellm_api_key, tenant_id
    perche' nel caso non-toolloop chiama classify_complexity() che richiede
    accesso a LiteLLM. Il refactoring proposto (unificazione classifier)
    eliminera' classify_complexity() e semplifichera' questa firma.

    Regola: se la query ha meno di 8 parole E non contiene hint legali,
    va dritta al ToolLoop (Studio mode) senza chiamare l'intent detector.
    Tutto il resto passa all'intent detector unificato.

    Costo: 0ms, $0. Nessuna chiamata esterna (per il path toolloop).
    """
    msg_lower = message.lower()
    word_count = len(message.split())
    has_legal = any(hint in msg_lower for hint in _LEGAL_HINTS)

    if word_count < 8 and not has_legal:
        return "toolloop"
    return "intent_detector"
```

**Soluzione definitiva - Fase B: Intent Detector unificato**

```python
# File: lexe-core/src/lexe_core/agent/pipeline.py
# Funzione: run_intent_detector() - SOSTITUIRE output schema

# Output JSON atteso dall'intent detector (singola LLM call)
INTENT_DETECTOR_SCHEMA = {
    "type": "object",
    "properties": {
        "level": {
            "type": "string",
            "enum": ["DIRECT", "SIMPLE", "STANDARD", "COMPLEX"],
            "description": "Complessita della query legale"
        },
        "scenario": {
            "type": "string",
            "enum": [
                "ricerca", "parere", "contratto_redazione",
                "contratto_revisione", "contenzioso",
                "recupero_crediti", "custom"
            ],
            "description": "Tipo di scenario legale (allineato a agent/models.py ScenarioType)"
        },
        "route": {
            "type": "string",
            "enum": ["toolloop", "legis", "lexorc"],
            "description": "Pipeline di destinazione"
        },
        "legal_domains": {
            "type": "array",
            "items": {"type": "string"},
            "description": "Domini legali coinvolti"
        },
        "intent_summary": {
            "type": "string",
            "description": "Sintesi dell'intento utente in 1 frase"
        },
        "confidence": {
            "type": "number",
            "minimum": 0,
            "maximum": 1,
            "description": "Confidenza della classificazione"
        },
        "complexity_signals": {
            "type": "array",
            "items": {"type": "string"},
            "description": "Segnali di complessita rilevati"
        }
    },
    "required": ["level", "scenario", "route", "confidence"]
}
```

**Nota SOTA:** Se possibile, usare Structured Outputs (response_format json_schema) via LiteLLM/OpenRouter per forzare il formato a livello di tokenizzazione, azzerando i parse error. In alternativa, usare la libreria `instructor`. Verificare supporto del modello corrente.

**Soluzione definitiva - Fase C: JSON parser con fallback robusto**

```python
# File: lexe-core/src/lexe_core/utils/json_safe.py - NUOVO FILE

import json
import re
from typing import Any

def parse_intent_json(raw: str, fallback_level: str = "STANDARD") -> dict:
    """Parse JSON dell'intent detector con fallback robusto a 3 livelli.
    
    Livello 1: JSON standard parse
    Livello 2: Pulizia markdown wrapping + retry
    Livello 3: Estrazione regex dei campi critici + flag _parse_fallback
    
    Returns: dict con almeno level, scenario, route, confidence
    """
    # Livello 1: parse diretto
    try:
        return json.loads(raw.strip())
    except json.JSONDecodeError:
        pass
    
    # Livello 2: rimuovi markdown wrapping
    try:
        cleaned = raw.strip()
        if cleaned.startswith("```"):
            # Rimuovi ```json ... ``` o ``` ... ```
            cleaned = re.sub(r'^```(?:json)?\s*\n?', '', cleaned)
            cleaned = re.sub(r'\n?```\s*$', '', cleaned)
        return json.loads(cleaned)
    except (json.JSONDecodeError, ValueError):
        pass
    
    # Livello 3: estrazione regex dei campi critici
    level_match = re.search(r'"level"\s*:\s*"(\w+)"', raw)
    scenario_match = re.search(r'"scenario"\s*:\s*"(\w+)"', raw)
    route_match = re.search(r'"route"\s*:\s*"(\w+)"', raw)
    confidence_match = re.search(r'"confidence"\s*:\s*([\d.]+)', raw)
    
    return {
        "level": level_match.group(1) if level_match else fallback_level,
        "scenario": scenario_match.group(1) if scenario_match else "ricerca",
        "route": route_match.group(1) if route_match else "legis",
        "confidence": float(confidence_match.group(1)) if confidence_match else 0.3,
        "legal_domains": [],
        "intent_summary": "",
        "complexity_signals": [],
        "_parse_fallback": True  # segnala che il parse non e stato pulito
    }
```

**Eliminare:** la funzione `classify_complexity()` in `agent/classifier.py` (ora ridondante — l'intent detector unificato copre entrambi gli output).

**Criteri di accettazione:**
- Latenza Fase 0 (routing) P95 < 0.6s su 1000 query di test
- Costo routing per query < $0.0001
- Parse error rate < 1% su 7 giorni di produzione
- Classificazione corretta >= 95% su gold set 200 query
- Zero regression su query non-legali (devono andare a ToolLoop)

**Effort:** 16h | **Sprint:** 10 | **Owner:** Frisco

---

### 2.3 [P4+P7] Rate Limiting e Retry Tool Esterni - ALTO

**Problema:** Nessun rate limiting nel tool executor. Nessun retry per normattiva/eurlex/infolex. Tool failure rate ~3%.

**Causa root:** implementazione iniziale senza gestione errori di rete.

**Soluzione definitiva: Rate Limiting differenziato + Retry con backoff + Circuit Breaker**

```python
# File: lexe-core/src/lexe_core/tools/executor.py - AGGIUNGERE

import asyncio
import time
from tenacity import (
    retry, stop_after_attempt, wait_random_exponential,
    retry_if_exception_type
)

# --- RATE LIMITING ---
# Semafori differenziati per categoria di tool.
# Motivazione: le fonti ufficiali (normattiva, eurlex) hanno API piu robuste
# e meritano piu slot concorrenti. Lo scraping web e piu fragile.

_TOOL_CATEGORIES = {
    # Nomi reali dei tool da definitions.py (con suffix _search)
    "normattiva_search": "official",
    "eurlex_search": "official",
    "vigenza_fast": "official",
    "infolex_search": "certified",
    "kb_search": "certified",
    "lex_search": "certified",
    "kb_normativa_search": "certified",
    "one_search": "certified",
    "lex_search_enriched": "certified",
}

_SEMAPHORES = {
    "official": asyncio.Semaphore(6),
    "certified": asyncio.Semaphore(4),
    "web": asyncio.Semaphore(3),
    "default": asyncio.Semaphore(4),
}

def _get_category(tool_name: str) -> str:
    if tool_name in _TOOL_CATEGORIES:
        return _TOOL_CATEGORIES[tool_name]
    if tool_name.startswith("web_"):
        return "web"
    return "default"

# --- CIRCUIT BREAKER ---
# Se un tool fallisce 5 volte consecutive, lo disabilita per 60 secondi.
# Evita di martellare un servizio down e spreca tempo/costo.

_CIRCUIT_STATE: dict[str, dict] = {}
_CIRCUIT_THRESHOLD = 5
_CIRCUIT_RESET_SECONDS = 60

class CircuitOpenError(Exception):
    """Errore quando il circuit breaker e aperto per un tool."""
    pass

def _check_circuit(tool_name: str) -> None:
    state = _CIRCUIT_STATE.get(tool_name)
    if not state:
        return
    if state["failures"] >= _CIRCUIT_THRESHOLD:
        elapsed = time.time() - state["last_failure"]
        if elapsed < _CIRCUIT_RESET_SECONDS:
            raise CircuitOpenError(
                f"Circuit open per {tool_name}: "
                f"{state['failures']} fallimenti, "
                f"reset tra {int(_CIRCUIT_RESET_SECONDS - elapsed)}s"
            )
        # Reset dopo il periodo di cooldown
        _CIRCUIT_STATE[tool_name] = {"failures": 0, "last_failure": 0}

def _record_failure(tool_name: str) -> None:
    if tool_name not in _CIRCUIT_STATE:
        _CIRCUIT_STATE[tool_name] = {"failures": 0, "last_failure": 0}
    _CIRCUIT_STATE[tool_name]["failures"] += 1
    _CIRCUIT_STATE[tool_name]["last_failure"] = time.time()

def _record_success(tool_name: str) -> None:
    _CIRCUIT_STATE[tool_name] = {"failures": 0, "last_failure": 0}

# --- EXECUTOR UNIFICATO ---

@retry(
    stop=stop_after_attempt(3),
    wait=wait_random_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type((ConnectionError, TimeoutError, OSError))
)
async def execute_tool_with_resilience(
    tool_name: str,
    params: dict,
    tenant_id: str | None = None
) -> dict:
    """Esecuzione tool con rate limiting, retry e circuit breaker.
    
    Flow:
    1. Check circuit breaker (se aperto, errore immediato)
    2. Acquisisci semaforo per categoria
    3. Esegui tool con timeout
    4. Su successo: reset circuit breaker
    5. Su fallimento: incrementa contatore, tenacity riprova fino a 3x
    """
    # 1. Circuit breaker
    _check_circuit(tool_name)
    
    # 2. Rate limiting
    category = _get_category(tool_name)
    sem = _SEMAPHORES[category]
    
    async with sem:
        try:
            # 3. Esecuzione con timeout globale di 30s
            result = await asyncio.wait_for(
                _execute_tool_impl(tool_name, params, tenant_id),
                timeout=30.0
            )
            # 4. Successo
            _record_success(tool_name)
            return result
        except Exception as e:
            # 5. Fallimento
            _record_failure(tool_name)
            raise
```

**Nota scalabilita:** Per deployment multi-pod (Kubernetes), sostituire `asyncio.Semaphore` con rate limiting distribuito via Valkey/Redis (Token Bucket). Lo stack ha Valkey 8 (Redis-compatible). Pattern:

```python
# Futuro: rate limiting distribuito via Redis
# import valkey.asyncio as avalkey  # o redis.asyncio
# async def check_rate_limit_redis(client, tool_name, limit=5, window=1):
#     key = f"ratelimit:{tool_name}"
#     current = await client.incr(key)
#     if current == 1:
#         await client.expire(key, window)
#     return current <= limit
```

**Dipendenza:** `pip install tenacity` (aggiungere a requirements.txt)

**Criteri di accettazione:**
- Tool failure rate < 1% su 500 esecuzioni di test
- Zero crash del backend sotto carico (50 query concorrenti)
- Circuit breaker si attiva correttamente dopo 5 fallimenti consecutivi
- Retry funziona: 2/3 dei timeout transitori vengono recuperati

**Effort:** 10h (6h rate limiting + 4h retry/circuit breaker) | **Sprint:** 9 | **Owner:** Frisco

---

### 2.4 [P5] Prompt Studio Obsoleto - MEDIO

**Problema:** Il prompt di Studio mode contiene riferimenti a elementi UI rimossi (sidebar, bottoni specifici).

**Soluzione:** Aggiornare il prompt in `prompts.py` rimuovendo i riferimenti obsoleti. Verificare con 10 query di tipo "studio" che il comportamento sia invariato.

**Criteri di accettazione:**
- Nessun riferimento a UI rimossa nel prompt
- UX review positiva da 5 utenti beta

**Effort:** 2h | **Sprint:** 9 | **Owner:** Frisco

---

### 2.5 [P2] Modelli LEXORC nel DB - ~~ALTO~~ GIA PARZIALMENTE IMPLEMENTATO

**NOTA v3.0:** Questa feature e **GIA IMPLEMENTATA** al 70%.

**Stato attuale verificato:**
- `core.model_role_defaults` (PK: `role`) **ESISTE** — migration 009
- `core.tenant_model_roles` (UNIQUE: `tenant_id, role`) **ESISTE** — migration 009
- Seed iniziale: `chat_primary`, `chat_followup`, `fact_extraction` — migration 009
- Aggiunta Sprint 8: `legis_intent_detector`, `legis_planner_light` — migration 013
- **Admin panel ModelsPage** gia gestisce `tenant_model_roles` con LEGIS roles
- Backfill automatico su tutti i tenant esistenti (migration 009)

**Cosa MANCA:**
I 6 ruoli LEXORC non sono ancora nel DB (`model_role_defaults`). Sono in `config.py` come env vars:
- `lexorc_classifier_model`, `lexorc_norm_agent_model`, `lexorc_caselaw_agent_model`
- `lexorc_doctrine_agent_model`, `lexorc_auditor_model`, `lexorc_synthesizer_model`

**Soluzione (solo la parte mancante):**

```sql
-- File: lexe-core/migrations/014_lexorc_role_defaults.sql (o successiva)
-- Descrizione: Aggiunge i ruoli LEXORC a model_role_defaults
-- Rollback: DELETE FROM core.model_role_defaults WHERE role LIKE 'lexorc_%';

BEGIN;

INSERT INTO core.model_role_defaults (role, model_name, display_label, description) VALUES
    ('lexorc_classifier',     'lexe-fast',         'LEXORC Classifier',    'Classificatore complessita query'),
    ('lexorc_norm_agent',     'lexe-primary',      'LEXORC Norm Agent',    'Agente normativo (codici, articoli)'),
    ('lexorc_caselaw_agent',  'lexe-primary',      'LEXORC Caselaw Agent', 'Agente giurisprudenza (massime)'),
    ('lexorc_doctrine_agent', 'lexe-primary',      'LEXORC Doctrine Agent','Agente dottrinale (annotazioni)'),
    ('lexorc_auditor',        'legal-tier2-haiku',  'LEXORC Auditor',      'Auditor coerenza e anti-hallucination'),
    ('lexorc_synthesizer',    'lexe-primary',       'LEXORC Synthesizer',  'Sintetizzatore risposta finale')
ON CONFLICT (role) DO NOTHING;

-- Backfill: copy LEXORC defaults to ALL existing tenants
INSERT INTO core.tenant_model_roles (tenant_id, role, model_name)
SELECT t.id, d.role, d.model_name
FROM core.tenants t
CROSS JOIN core.model_role_defaults d
WHERE d.role LIKE 'lexorc_%'
ON CONFLICT (tenant_id, role) DO NOTHING;

COMMIT;
```

**Poi:** refactorare `advanced_orchestrator.py` per leggere da `tenant_model_roles` instead of `settings.lexorc_*_model`.
Pattern gia usato da `services/llm_config.py` per LEGIS roles.

**Criteri di accettazione:**
- Admin panel mostra ruoli LEXORC nel Tenant Editor tab Models
- Un tenant puo cambiare modello LEXORC senza restart
- Zero breaking change (env vars restano come fallback in config.py)

**Effort:** 4h (ridotto — infra gia esiste) | **Sprint:** 11 | **Owner:** Frisco

---

### 2.6 [P3] Alias Duplicati nel Catalogo LiteLLM - BASSO

**Problema:** Alcuni alias puntano allo stesso modello sottostante:
- `legal-tier1-gemini` e `legal-tier2-gemini` → entrambi Gemini 2.5 Flash (come `lexe-fast`)
- `legal-tier3-gpt5` → GPT-5.2 (come `lexe-complex`)

**NOTA v3.0:** Questi alias sono definiti in DUE posti:
1. `lexe-infra/litellm/config.yaml` (LiteLLM routing)
2. `core.llm_model_catalog` (DB, migration 007 seed)

Bisogna pulire entrambi. Verificare prima `core.tenant_model_roles`:

```sql
SELECT DISTINCT model_name FROM core.tenant_model_roles
WHERE model_name IN ('legal-tier1-gemini', 'legal-tier2-gemini', 'legal-tier3-gpt5');
```

**Criteri di accettazione:**
- Zero alias duplicati nel catalogo
- Zero breaking change per tenant

**Effort:** 4h | **Sprint:** 11 | **Owner:** Frisco

---

### 2.7 [FF] Feature Flag ff_legis_agent Default - CRITICO, IMMEDIATO

**Problema:** La nuova pipeline LEGIS (multi-agente) e dietro feature flag disabilitato. Gli utenti usano la pipeline legacy (ToolLoop) come default, che e inferiore.

**Soluzione:** Cambiare il default di `ff_legis_agent` da `false` a `true` in config.py. Mantenere il flag attivo per rollback rapido.

```python
# File: lexe-core/src/lexe_core/config.py
# NOTA: config.py usa pydantic-settings (NOT os.getenv).
# Tutti i campi hanno prefisso env LEXE_ via model_config.
#
# Cambiare il default da False a True:
# Attuale:
#   ff_legis_agent: bool = False
# Nuovo:
ff_legis_agent: bool = True
# Override via env var: LEXE_FF_LEGIS_AGENT=false (per rollback)
```

**Criteri di accettazione:**
- Tutti i tenant vedono la pipeline LEGIS come default
- Monitoring attivo su error rate per 48h post-attivazione
- Rollback possibile in < 1 minuto via env var

**Effort:** 1h | **Sprint:** 9 | **Owner:** Frisco

---

## 3. ROADMAP SPRINT-BY-SPRINT

### Sprint 9 - "Stabilizzazione Base" (Settimane 1-2, 15h)

| ID | Azione | Problema | Ore | Priorita | Criterio di Accettazione |
|---|---|---|---|---|---|
| S9.1 | Fix max_completion_tokens | P6 | 2h | Critico | 0 parse error su 100 query test |
| S9.2 | Attivare ff_legis_agent default | FF | 1h | Critico | Pipeline LEGIS attiva, monitoring OK 48h |
| S9.3 | Retry universale tool esterni | P7 | 4h | Alto | 2/3 timeout recuperati, failure < 2% |
| S9.4 | Rate limiting tool executor | P4 | 6h | Alto | 0 crash sotto 50 query concorrenti |
| S9.5 | Aggiornare prompt Studio | P5 | 2h | Medio | 0 riferimenti a UI rimossa |

**Gate di uscita Sprint 9:**
- Tool failure rate < 2% su 500 esecuzioni
- Zero crash sotto carico di test
- ff_legis_agent attivo senza regressioni per 48h
- Se un gate non passa: estendere sprint di 3 giorni, non procedere a Sprint 10

---

### Sprint 10 - "Unified Routing" (Settimane 3-4, 28h)

| ID | Azione | Problema | Ore | Priorita | Criterio di Accettazione |
|---|---|---|---|---|---|
| S10.1 | Unificare Classifier (Fase A+B+C) | P1 | 16h | Critico | Latenza routing P95 < 0.6s |
| S10.2 | JSON validation con fallback | P1 | 4h | Alto | Parse error < 1% |
| S10.3 | Gold set 200 query + validazione | - | 8h | Alto | Accuracy >= 95% su gold set |

**Composizione gold set (200 query):**
- 80 query legali tipiche (ricerca normativa, giurisprudenza, pareri)
- 40 query Studio mode (saluti, domande generiche, calcoli)
- 30 query edge case (miste legale/generico, ambigue)
- 20 query multi-dominio (cross-reference tra aree)
- 15 query con riferimenti normativi espliciti (art. X, d.lgs. Y)
- 15 query in linguaggio naturale senza hint legali espliciti

**Gate di uscita Sprint 10:**
- Latenza Fase 0 P95 < 0.6s su 1000 query
- Parse error rate < 1%
- Routing accuracy >= 95% su gold set
- Se accuracy < 90%: ritoccare prompt intent detector, estendere 1 settimana
- Se accuracy 90-95%: accettabile, migliorare nel sprint successivo

---

### Sprint 11 - "Enterprise Readiness" (Settimane 5-6, 32h)

| ID | Azione | Problema | Ore | Priorita | Criterio di Accettazione |
|---|---|---|---|---|---|
| S11.1 | LEXORC roles nel DB (completamento) | P2 | 4h | Alto | Ruoli LEXORC in model_role_defaults + admin panel |
| S11.2 | Cleanup alias duplicati | P3 | 4h | Basso | 0 alias duplicati, 0 breaking change |
| S11.3 | Confidence Spectrum (badge 3 livelli) | Innovazione | 12h | Alto | Badge verde/giallo/rosso visibile in UI |
| S11.4 | Wiki interna know-how tecnico | Protezione | 8h | Medio | Struttura /lexe-knowhow/ creata con 6+ ADR |

**Confidence Spectrum - Specifica:**

| Livello | Colore | Condizione | Icona |
|---|---|---|---|
| VERIFIED | Verde (#10B981) | 2+ fonti coerenti + norme vigenti verificate + confidence > 0.85 | Scudo con check |
| CAUTION | Giallo (#F59E0B) | Contraddizioni rilevate OPPURE confidence 0.6-0.85 OPPURE 1 sola fonte | Triangolo attenzione |
| LOW_CONFIDENCE | Rosso (#EF4444) | Confidence < 0.6 OPPURE 0 fonti verificate OPPURE self-correction fallita | Cerchio esclamativo |

**Gate di uscita Sprint 11:**
- Admin panel funzionante per LEXORC roles
- Badge confidence visibili su almeno il 90% delle risposte LEGIS
- Nessun breaking change per tenant esistenti

---

### Sprint 12 - "Quality Assurance" (Settimane 7-8, 32h)

| ID | Azione | Problema | Ore | Priorita | Criterio di Accettazione |
|---|---|---|---|---|---|
| S12.1 | Self-Correction Loop nel Verifier | Innovazione | 16h | Alto | Verifier pass rate > 90% |
| S12.2 | SLA monitoring dashboard | Operazioni | 8h | Medio | Dashboard funzionale con alert |
| S12.3 | Enhanced Verifier con source grounding | Qualita | 8h | Alto | 85%+ affermazioni con citazione |

**Self-Correction Loop - Specifica:**

```
Synthesis generata (Fase S)
        |
        v
    Verifier (Fase V)
        |
   +----+----+
   |         |
  PASS      FAIL (issues: [...])
   |         |
   v         v
Risposta   retry_with_reflection(query, synthesis, issues)
+ badge        |
verde         v
          Nuova Synthesis
              |
              v
          Verifier (tentativo 2)
              |
         +----+----+
         |         |
        PASS      FAIL
         |         |
         v         v
    Risposta    Risposta + disclaimer
    + badge     + badge GIALLO
    verde       "Alcune informazioni potrebbero
                 richiedere verifica manuale"
```

**Regole Self-Correction:**
- Massimo 2 tentativi (mai di piu, per costo e latenza)
- Timeout totale per self-correction: 15 secondi
- Se il secondo tentativo fallisce, la risposta viene comunque consegnata con badge giallo
- Log dettagliato di ogni self-correction per analisi

**Gate di uscita Sprint 12:**
- Verifier pass rate > 90% su gold set
- Source grounding coverage > 85%
- SLA dashboard attiva con alert configurati
- Load test 1000 query/ora senza degradazione

---

### Sprint 13-16 - "Differenziazione" (Settimane 9-16, stimato)

| ID | Azione | Effort | Sprint |
|---|---|---|---|
| S13.1 | Graph-Aware Routing (KB as nervous system) | 24h | 13 |
| S13.2 | Advanced RAG: query expansion + cross-encoder re-ranking | 20h | 13 |
| S14.1 | Citation resolution da 44.8% a 75%+ | 40h | 14 |
| S14.2 | Source grounding avanzato (snippet evidenziabili) | 16h | 14 |
| S15.1 | Espansione KB a 100+ codici | 60h | 15 |
| S16.1 | API pubblica per DMS integration | 40h | 16 |

**Graph-Aware Routing (S13.1) - Concept:**

```python
# Pseudo-codice: il citation graph guida le decisioni dell'orchestratore
async def graph_enhanced_routing(query: str, intent: dict) -> dict:
    """Se la query contiene riferimenti normativi espliciti,
    pre-fetch massime collegate dal graph traversal.
    Evita broad search quando il grafo ha gia le informazioni."""
    
    norm_refs = extract_norm_references(query)
    # Esempio: ["CC:2043", "DLgs:81/2008:art.18"]
    
    if norm_refs:
        related_massime = await norm_graph.get_connected_massime(
            norm_refs, max_hops=2
        )
        related_norms = await citation_graph.get_citing_norms(norm_refs)
        
        return {
            "pre_fetched_norms": norm_refs,
            "graph_massime_ids": [m.id for m in related_massime[:20]],
            "graph_related_norms": related_norms,
            "skip_broad_search": len(related_massime) > 10,
        }
    return {}
```

---

## 4. METRICHE DI SUCCESSO E GATE

### 4.1 Performance

| Metrica | Baseline | Post-Sprint 10 | Post-Sprint 12 | Target 12 mesi |
|---|---|---|---|---|
| Latenza LEGIS STANDARD | 20-30s | < 18s | < 15s | < 12s |
| Latenza LEGIS DIRECT | 5-8s | < 3s | < 2s | < 1.5s |
| Latenza Fase 0 (routing) | 2.5-3.5s | < 0.6s | < 0.6s | < 0.3s |
| LLM calls per query legale | 3-4 | 2 | 2 | 2 |
| Tool failure rate | ~3% | < 1% | < 0.5% | < 0.2% |
| Parse error rate JSON | ~5% | < 1% | < 0.5% | < 0.1% |

### 4.2 Qualita

| Metrica | Baseline | Post-Sprint 12 | Target 12 mesi |
|---|---|---|---|
| Verifier pass rate | Da misurare | > 90% | > 95% |
| Citation resolution rate | 44.8% | 55% | 75% |
| Source grounding coverage | Da misurare | > 85% | > 95% |
| Confidenza media risposte | Da misurare | > 0.80 | > 0.85 |
| Legal accuracy rate | Da misurare | > 92% | > 95% |

### 4.3 Business

| Metrica | Target Q1 2026 | Target Q2 2026 | Target 12 mesi |
|---|---|---|---|
| Costo medio per query | < $0.002 | < $0.0015 | < $0.001 |
| Uptime | 99% | 99.5% | 99.9% |
| NPS utenti beta | 40+ | 50+ | 60+ |
| Tenant enterprise | 3 | 10 | 25+ |
| Demo -> Trial conversion | 20% | 25% | 30% |

### 4.4 Costo per Pipeline (validato)

| Pipeline | LLM Calls (post-unificazione) | Token medi | Costo stimato |
|---|---|---|---|
| ToolLoop (Studio) | 1-2 | ~2K | ~$0.0004 |
| LEGIS DIRECT | 1 | ~1.5K | ~$0.0003 |
| LEGIS STANDARD | 3 | ~6K | ~$0.0012 |
| LEGIS COMPLEX | 4 | ~12K | ~$0.0024 |
| LEXORC | 6+ | ~20K | ~$0.0040 |

A ~85 euro/utente/mese con 50 query/giorno, il costo LLM mensile per utente e ~2-4 euro. Margine lordo > 95%. Pricing sostenibile confermato.

---

## 5. DECISIONI ARCHITETTURALI (ADR)

### ADR-D1: Unificare classifier in singola LLM call

| Campo | Valore |
|---|---|
| Status | Approvato |
| Alternativa scartata | Mantenere doppia classificazione con cache |
| Motivazione | La cache non risolve il costo, aggiunge complessita. Una singola call con output esteso e piu semplice e veloce. |
| Rischio residuo | Edge case di classificazione errata (mitigato da gold set) |

### ADR-D2: Semafori differenziati per categoria tool

| Campo | Valore |
|---|---|
| Status | Approvato |
| Alternativa scartata | Semaforo globale unico |
| Motivazione | Le fonti ufficiali (normattiva, eurlex) hanno API piu robuste e meritano piu slot concorrenti. Un semaforo unico limita inutilmente le fonti affidabili quando solo lo scraping web e sotto pressione. |
| Evoluzione futura | Rate limiting distribuito via Redis per multi-pod |

### ADR-D3: LEXORC roles via DB (non env vars)

| Campo | Valore |
|---|---|
| Status | **Parzialmente implementato** (migration 009 + 013 hanno infrastruttura, mancano solo i seed LEXORC) |
| Alternativa scartata | Mantenere env vars con restart |
| Motivazione | Incompatibile con multi-tenant enterprise. Un tenant deve poter scegliere i propri modelli (es. solo Claude per compliance interna) senza richiedere restart o deploy. |
| Nota v3.0 | `core.model_role_defaults` e `core.tenant_model_roles` gia esistono. Servono solo INSERT dei 6 ruoli LEXORC + refactoring di `advanced_orchestrator.py`. |

### ADR-D4: Attivare ff_legis_agent come default

| Campo | Valore |
|---|---|
| Status | Approvato |
| Alternativa scartata | Mantenere opt-in |
| Motivazione | La pipeline legacy non puo essere la UX primaria. Contraddice la vision "AI-First" e indebolisce il posizionamento multi-agente. |
| Rollback | Env var LEXE_FF_LEGIS_AGENT=false, tempo < 1 minuto |

### ADR-D5: Confidence Spectrum (3 livelli) vs badge binario

| Campo | Valore |
|---|---|
| Status | Approvato |
| Alternativa scartata | Badge verde/rosso binario |
| Motivazione | Il giallo cattura le zone grigie fondamentali nel dominio legale. Un avvocato deve sapere quando una risposta e "probabilmente corretta ma da verificare" vs "verificata" vs "inaffidabile". |

### ADR-D6: Self-correction max 2 retry

| Campo | Valore |
|---|---|
| Status | Approvato |
| Alternativa scartata | Retry illimitati |
| Motivazione | Costo e latenza devono restare sotto controllo. 2 retry e il punto di equilibrio tra qualita e performance. Oltre, il costo marginale supera il beneficio. |

### ADR-D7: Investire in knowledge base interna per il team

| Campo | Valore |
|---|---|
| Status | Approvato |
| Alternativa scartata | Non documentare (status quo) |
| Motivazione | Bus factor = 1 (Frisco). Preserva know-how, riduce rischio dipendenza da singolo, accelera onboarding futuri sviluppatori. |

### ADR-D8: Proteggere KB e algoritmi come segreto commerciale

| Campo | Valore |
|---|---|
| Status | Proposto |
| Alternativa scartata | Non proteggere formalmente |
| Motivazione | La KB proprietaria (lexe-max) e il citation/norm graph sono il vero moat competitivo. Art. 98 CPI tutela segreti commerciali senza registrazione, ma richiede misure attive di protezione (NDA, accesso compartmentalizzato, audit logging). |
| Asset da proteggere | Architettura multi-agente, lexe-max KB, citation graph, norm graph, formula RRF, prompt templates proprietari, tool execution policy |

---

## 6. AI-FIRST MATURITY MODEL

Roadmap di evoluzione architetturale a lungo termine.

| Livello | Nome | Descrizione | Target |
|---|---|---|---|
| L1 | Rule-Based Routing | Euristica lessicale (attuale ToolLoop legacy) | Superato |
| L2 | LLM-Driven Routing | Intent detector unificato decide la pipeline | Sprint 10 |
| L3 | Self-Optimizing | Routing evolve in base ai success rate storici | Sprint 16+ |
| L4 | Proactive Agentic | Il sistema anticipa i bisogni dell'utente | Q2 2027 |

**KCI - Know-How Codification Index (metrica composita):**

```
KCI = (Fonti verificate in KB / Totale fonti consultate)
      x Citation Resolution Rate
      x Confidenza Media
```

- Baseline stimata: ~0.45
- Target Sprint 12: > 0.65
- Target 12 mesi: > 0.85

---

## 7. CHECKLIST PRE-IMPLEMENTAZIONE

### Prima di iniziare Sprint 9

**Technical:**
- [ ] Backup completo database PostgreSQL (pg_dump)
- [ ] Snapshot infrastruttura Docker (docker commit per ogni container)
- [ ] Preparare gold set iniziale (almeno 50 query prioritarie, espandere a 200 in Sprint 10)
- [ ] Catturare baseline metriche attuali (latenza, error rate, costo per query)
- [ ] Verificare che feature flag system funzioni (test toggle ff_legis_agent)
- [ ] Installare tenacity: `pip install tenacity` e aggiungere a requirements.txt

**Legal/Protezione:**
- [ ] Inventario asset know-how (lista completa di cosa e proprietario)
- [ ] Verifica NDA attivi con tutti i contractor
- [ ] Audit logging attivo per accessi a codice sensibile

**Comunicazione:**
- [ ] Notifica ai tenant beta sulle modifiche in arrivo
- [ ] Canale di feedback rapido attivo (Slack o email dedicata)

### Prima di ogni deploy

- [ ] Tutti i test automatici passano
- [ ] Gold set validato (accuracy >= target del gate)
- [ ] Feature flag configurato per rollback
- [ ] Snapshot DB pre-deploy
- [ ] Monitoring attivo su metriche critiche
- [ ] Piano di rollback documentato e testato

---

## 8. CONVENZIONI DI CODICE E STILE

### Python

- Type hints su tutti i parametri e return values
- Docstring su ogni funzione pubblica (stile Google)
- async/await per tutte le operazioni I/O
- Pydantic per validazione dati (modelli in `models/`)
- Logging strutturato (JSON) con livelli appropriati

### SQL Migrations

- Naming: `NNN_descrizione.sql` (es. `014_lexorc_role_defaults.sql`) — NO prefisso "migration_"
- Sempre includere commento con: descrizione, comando di rollback
- Sempre wrappare in `BEGIN; ... COMMIT;`
- Testare rollback prima di applicare

### Feature Flags

- Naming: `ff_nome_feature`
- Default in config.py, override via env var `LEXE_FF_NOME_FEATURE`
- Documentare in commento: cosa controlla, quando rimuovere

### Commit Messages

```
tipo(scope): descrizione breve

tipo: fix | feat | refactor | docs | test | chore
scope: pipeline | tools | config | db | admin | ui

Esempio:
fix(pipeline): unifica classifier in singola LLM call (P1)
feat(tools): aggiunge rate limiting differenziato per categoria
```

### Brand (per UI e messaggi utente)

- Palette: Navy #1E3A5F (primario), Cyan #0CCBEE (accent), White #FFFFFF (sfondo)
- Font: Inter (Regular 400, SemiBold 600, Bold 700)
- Tono: professionale ma accessibile, orientato al risultato, no hype
- Colori semantici: Success #10B981, Warning #F59E0B, Error #EF4444, Info #3B82F6

---

## 9. PIANO DI ROLLBACK PER COMPONENTE

| Modifica | Meccanismo di Rollback | Tempo | Impatto |
|---|---|---|---|
| P6 (token limit) | `UPDATE core.llm_model_catalog SET config = jsonb_set(config, '{max_completion_tokens}', '250') WHERE model_name = 'lexe-fast';` | < 5 min | Nessuno (torna a troncamento) |
| FF (legis agent) | `LEXE_FF_LEGIS_AGENT=false` + restart container | < 1 min | Utenti tornano a ToolLoop |
| P7 (retry) | Feature flag disabilita retry decorator | < 1 min | Tool senza retry (come prima) |
| P4 (rate limiting) | Feature flag bypassa semafori | < 1 min | Tool senza rate limit (come prima) |
| P1 (unified classifier) | `LEXE_FF_LEGIS_AGENT=false` + riabilita classify_complexity() | < 5 min | Torna a double classifier |
| P2 (LEXORC roles DB) | `DELETE FROM core.model_role_defaults WHERE role LIKE 'lexorc_%'` + env vars fallback in config.py | < 5 min | Modelli tornano da env vars |
| Confidence Spectrum | Feature flag nasconde badge in UI | < 1 min | Badge non visibili |
| Self-Correction Loop | Feature flag bypassa loop nel verifier | < 1 min | Verifier senza retry |

---

## 10. RIEPILOGO CONFRONTO STRATEGIE

Questo documento implementa la **Strategia B (BALANCED)**. Per riferimento, il confronto con le alternative:

| Dimensione | Conservativa (A) | **BALANCED (B)** | Aggressiva (C) |
|---|---|---|---|
| Durata | 12 settimane | **8 settimane** | 4 settimane |
| Rischio | Basso | **Medio (controllato)** | Alto |
| Costo | ~120h | **~107h** | ~70h |
| Qualita | Eccellente | **Alta** | Media |
| Impatto UX | Neutrale | **Positivo** | Variabile |
| Impatto Competitivo | Difensivo | **Equilibrato** | Offensivo (rischioso) |
| Team Stress | Basso | **Medio** | Alto |
| Reversibilita | Alta | **Alta (feature flags)** | Bassa |
| Probabilita Successo | ~90% | **~85%** | ~65% |

**Condizione critica per successo:** Attivare ff_legis_agent come default nello Sprint 9 (S9.2). Senza questo passo, la roadmap perde significato strategico.

**Contingenza:** Se metriche post-Sprint 10 non raggiungono i gate, attivare "modalita ibrida" (classifier unificato solo per query con legal hints) per 2 settimane.

---

> **Documento compilato da LEXe.OS - Orchestratore Operativo**
> Changelog:
>   v2.0 - 20/02/2026 - Sintesi definitiva best-of-breed per CCC
>   v3.0 - 20/02/2026 - Audit architetturale completo vs codebase reale (Claude Code)
>   v3.1 - 21/02/2026 - Verifica codebase: fix 12 incongruenze (conteggi, nomi tool, scenari, modelli, KB, FF pattern)
> Prossima revisione: fine Sprint 10 (circa meta marzo 2026)
> Fonti originali: GEM, QW35, MM25, QW3M, DS32, PKB, WEB1-4
> Audit v3.0: Esplorazione diretta di 6 repository, 13 container, 24 agent files, 13 migrations

---

## APPENDICE: ERRATA v2.0 → v3.0

Errori fattuali corretti rispetto alla versione 2.0 del documento.

| # | Claim v2.0 | Realta verificata | Impatto |
|---|-----------|------------------|---------|
| 1 | "Open WebUI" come Chat Interface | Custom **React 18 + Vite + TypeScript** (lexe-webchat) | Sezione 1.1 riscritta |
| 2 | "Redis Stack" per cache | **Valkey 8** (Redis-compatible) | Sezione 1.1 corretta |
| 3 | "Qdrant" per vector DB | **PostgreSQL pgvector** (in lexe-postgres e lexe-max) | Sezione 1.1, 1.3 corretta |
| 4 | "Neo4j 5" per graph DB | **Apache AGE** (estensione PostgreSQL in lexe-max) | Sezione 1.1, 1.3 corretta |
| 5 | 8 container | **14 LEXE + 2 shared + 2 esterni = 18 servizi** (+ Temporal, Temporal-UI, lexe-admin, lexe-tools, shared-traefik, shared-portainer, shared-searxng, shared-jaeger) | Sezione 1.1 corretta |
| 6 | `lexe-primary` = "Claude 3.5 Sonnet / GPT-4o" | **Mistral Large 2512** (openrouter) | Sezione 1.4 corretta |
| 7 | `legal-tier2-haiku` = "Claude 3 Haiku" | **Claude Haiku 4.5** | Sezione 1.4 corretta |
| 8 | `core.model_catalog` | Tabella reale: **`core.llm_model_catalog`** (mig. 007) | Sezioni 2.1, 9 corrette |
| 9 | `lexe-admin/panel.py` | lexe-admin e un'app **React** separata (porta 3014) | Sezione 1.5 corretta |
| 10 | `migration_015`, `migration_016` | Prossima migration: **014** | Sezioni 2.1, 2.5 corrette |
| 11 | LEXORC roles "non nel DB" | `model_role_defaults` + `tenant_model_roles` **GIA ESISTONO** (mig. 009) | Sezione 2.5 ridimensionata |
| 12 | Tools: `brocardi_search`, `dejure_search`, `gazzetta_ufficiale` | **NON esistono** | Sezione 2.3 corretta |
| 13 | Tools senza suffix `_search` | Nomi reali **CON** suffix: `normattiva_search`, `eurlex_search`, `infolex_search`, `kb_search`, `lex_search`, `lex_search_enriched`, `kb_normativa_search`, `one_search` (da `definitions.py`) | Sezione 2.3 corretta |
| 14 | `tools/registry.py` | NON esiste — tool definitions in `tools/definitions.py` | Sezione 1.5 corretta |
| 15 | `models/intent.py` | Intent models in `agent/models.py` (PipelineLevel, ScenarioType) | Sezione 1.5 corretta |
| 16 | Pipeline come path primario | Default: **direct LiteLLM streaming** (toolloop). LEGIS/LEXORC/ORCHIDEA tutti **DISABLED** | Sezione 1.2 riscritta |
| 17 | Nessuna menzione di Temporal | **lexe-temporal** attivo (porta 7234) con UI (8180) | Sezione 1.1 aggiunta |
| 18 | Double classifier come "CRITICO" | classifier e intent_detector sono moduli **diversi** aggiunti in sprint separati | Sezione 2.2 ridimensionata |
| 19 | Naming migration `migration_NNN_*` | Pattern reale: **`NNN_descrizione.sql`** | Sezione 8 corretta |
| 20 | Admin: "11 router, 9 service" | Reale: **14 router, 11 service** (verificato con glob su codebase) | Sezione 1.5 corretta (v3.1) |
| 21 | LEXORC: "5 agenti paralleli" | **6 componenti** (Classifier + 3 agenti paralleli + Auditor + Synthesizer) | Sezione 1.2 corretta (v3.1) |
| 22 | Modelli: tabella con 7 alias | **9 alias** nel config.yaml (mancavano `legal-tier1-gemini`, `legal-tier2-gemini`) | Sezione 1.4 corretta (v3.1) |
| 23 | kb.work, kb.normativa_chunk, kb.annotation | **NON esistono su staging** (potrebbero essere solo in prod) | Sezione 1.3 corretta (v3.1) |
| 24 | Scenari: "generico", "normativa", "giurisprudenza", "dottrina" | Reali: `contratto_revisione`, `recupero_crediti`, `custom` (da `agent/models.py` ScenarioType) | Sezione 2.2 corretta (v3.1) |
| 25 | `_pre_intent_route(message)` senza params | Firma reale: `(message, litellm_url, litellm_api_key, tenant_id)` | Sezione 2.2 corretta (v3.1) |
| 26 | ff_legis_agent via `os.getenv()` | Usa **pydantic-settings** (`ff_legis_agent: bool = False`) | Sezione 2.7 corretta (v3.1) |
