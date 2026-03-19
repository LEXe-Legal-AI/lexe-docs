# LEXe — Piano di Orchestrazione Multi-Terminale per Claude Code CLI

> **Versione:** 1.0  
> **Data:** 21/02/2026  
> **Destinazione:** Claude Code CLI — esecuzione multi-terminale parallela  
> **Strategia:** 5 terminali × team di agenti specializzati, esecuzione sequenziale per step  
> **Fonti:** CCC Implementation Blueprint v3.2 + Proposte 2.0 + BACKLOG  
> **Regola d'oro:** Mai più di un terminale sullo stesso file. Conflitti = rollback.

---

## 0. COME USARE QUESTO DOCUMENTO

Questo documento orchestra **5 terminali paralleli**, ciascuno con un team di agenti Claude Code CLI specializzati. Ogni terminale lavora su un'area isolata del codebase per evitare conflitti.

**Ogni terminale segue lo stesso ciclo sequenziale obbligatorio:**

```
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: DEVELOP    → Esegui le modifiche                   │
│  STEP 2: TEST LOCAL → Testa localmente (lint, unit, type)   │
│  STEP 3: COMMIT     → Se OK → git commit con conventional   │
│  STEP 4: DEPLOY STG → Deploy su staging                     │
│  STEP 5: AUTO TEST  → Test automatici su staging            │
│  STEP 6: USER TEST  → Test manuale utente (Frisco)          │
│  STEP 7: DEPLOY PRD → Se OK utente → deploy su produzione   │
│  STEP 8: DOCUMENT   → Aggiorna docs e memorie               │
└─────────────────────────────────────────────────────────────┘
```

**STOP GATE:** Se uno step fallisce, il terminale si ferma, esegue il rollback del componente e riprova. Non si procede mai senza tutti i gate verdi. Gli altri terminali possono continuare indipendentemente.

**ORDINE DI AVVIO:** I terminali hanno dipendenze. Rispettare l'ordine di fase.

---

## 1. ARCHITETTURA DEI TERMINALI

```
                    ┌─────────────────────────────┐
                    │     FRISCO (Supervisore)     │
                    │  Approva USER TEST (Step 6)  │
                    │  Comanda DEPLOY PROD (Step 7)│
                    └──────────┬──────────────────┘
                               │
        ┌──────────┬───────────┼───────────┬──────────┐
        │          │           │           │          │
   ┌────▼────┐ ┌───▼────┐ ┌───▼────┐ ┌───▼────┐ ┌───▼────┐
   │  T1     │ │  T2    │ │  T3    │ │  T4    │ │  T5    │
   │ PIPELINE│ │ TOOLS  │ │ UI/UX  │ │ DATA   │ │ QA+DOC │
   │ & ROUTE │ │ & RESIL│ │ & PRMPT│ │ & DB   │ │ & MON  │
   └─────────┘ └────────┘ └────────┘ └────────┘ └────────┘
```

### Matrice di Ownership (chi tocca cosa)

| File/Area | T1 | T2 | T3 | T4 | T5 |
|---|:---:|:---:|:---:|:---:|:---:|
| `lexe-core/agent/pipeline.py` | ✅ | | | | |
| `lexe-core/agent/classifier.py` | ✅ | | | | |
| `lexe-core/agent/intent_detector.py` | ✅ | | | | |
| `lexe-core/agent/verifier.py` | ✅ | | | | |
| `lexe-core/agent/synthesizer.py` | ✅ | | | | |
| `lexe-core/agent/agents/*` | ✅ | | | | |
| `lexe-core/agent/models.py` | ✅ | | | | |
| `lexe-core/gateway/customer_router.py` | ✅ | | | | |
| `lexe-core/config.py` | ✅ | | | | |
| `lexe-core/tools/executor.py` | | ✅ | | | |
| `lexe-core/tools/policy.py` | | ✅ | | | |
| `lexe-core/tools/health.py` | | ✅ | | | |
| `lexe-core/tools/definitions.py` | | ✅ | | | |
| `lexe-tools-it/tools/*` | | ✅ | | | |
| `lexe-core/utils/json_safe.py` (NUOVO) | | ✅ | | | |
| `lexe-webchat/src/**` | | | ✅ | | |
| `lexe-core/prompts/v1/*` | | | ✅ | | |
| `lexe-core/agent/prompts.py` | | | ✅ | | |
| `lexe-admin/src/**` | | | ✅ | | |
| `lexe-core/migrations/*` | | | | ✅ | |
| `lexe-infra/docker-compose*` | | | | ✅ | |
| `lexe-infra/litellm/config.yaml` | | | | ✅ | |
| `lexe-memory/src/**` | | | | ✅ | |
| `lexe-docs/**` | | | | | ✅ |
| `tests/**` | | | | | ✅ |
| Gold set, metriche, monitoring | | | | | ✅ |

---

## 2. PROTOCOLLO COMUNE — Ogni Step in Dettaglio

### STEP 1: DEVELOP

```bash
# Ogni terminale lavora su un branch dedicato
git checkout -b <tipo>/<terminale>/<descrizione>
# Esempio: feat/t1-pipeline/unified-classifier

# Regola: un commit per modifica atomica
# Mai committare codice che non compila
```

### STEP 2: TEST LOCAL

```bash
# Python: lint + type check + unit test
cd lexe-core
ruff check src/
mypy src/ --ignore-missing-imports
pytest tests/ -x -v --tb=short

# Frontend (solo T3):
cd lexe-webchat
npm run lint
npm run type-check
npm run test
```

**Gate locale:**
- Zero errori lint
- Zero errori type (warning tollerati)
- Tutti gli unit test passano
- Se un test fallisce → fix e riprova, non committare

### STEP 3: COMMIT

```bash
# Conventional commit obbligatorio
git add -A
git commit -m "<tipo>(<scope>): <descrizione breve>

<corpo opzionale con dettagli tecnici>

Refs: <problema dal Blueprint, es. P1, P4, P7>
Terminal: T<N>
Sprint: S<N>"

# Tipi: fix | feat | refactor | docs | test | chore
# Scope: pipeline | tools | config | db | admin | ui | prompts | infra
```

### STEP 4: DEPLOY STAGING

```bash
# Push branch
git push origin <branch>

# Merge in staging (o branch develop)
git checkout staging
git merge <branch> --no-ff
git push origin staging

# Deploy staging via docker compose
ssh lexe-server "cd /opt/lexe && \
  docker compose -f docker-compose.yml \
  -f docker-compose.override.staging.yml \
  pull <servizio> && \
  docker compose -f docker-compose.yml \
  -f docker-compose.override.staging.yml \
  up -d <servizio>"

# Verifica container UP
ssh lexe-server "docker ps --filter name=lexe-<servizio> --format '{{.Status}}'"
```

### STEP 5: AUTO TEST SU STAGING

```bash
# Health check endpoint
curl -s https://api.stage.lexe.pro/health | jq .

# Test specifici per terminale (vedi sezioni T1-T5)
# Ogni terminale ha i propri test automatici da lanciare qui

# Gate automatico: TUTTI i test devono passare
# Se fallisce → rollback container a versione precedente:
ssh lexe-server "docker compose ... up -d --force-recreate <servizio>"
```

### STEP 6: USER TEST (Frisco)

```
⏸️ PAUSA — Attendi approvazione di Frisco

Frisco testa manualmente su stage-chat.lexe.pro:
- [ ] Funzionalità principale OK
- [ ] Nessuna regressione visibile
- [ ] Performance accettabile
- [ ] UX coerente

Segnale di GO: Frisco scrive "T<N> OK PROD" nella chat/canale
Segnale di STOP: Frisco scrive "T<N> ROLLBACK" → eseguire rollback Step 4
```

### STEP 7: DEPLOY PRODUZIONE

```bash
# Solo dopo OK esplicito di Frisco
# Snapshot DB pre-deploy
ssh lexe-server "pg_dump -h localhost -p 5435 -U lexe -d lexe_db \
  --schema=core --schema=kb > /backup/pre_deploy_$(date +%Y%m%d_%H%M).sql"

# Merge in main/production
git checkout main
git merge staging --no-ff
git push origin main

# Deploy produzione
ssh lexe-server "cd /opt/lexe && \
  docker compose -f docker-compose.yml \
  -f docker-compose.override.prod.yml \
  pull <servizio> && \
  docker compose -f docker-compose.yml \
  -f docker-compose.override.prod.yml \
  up -d <servizio>"

# Monitoring intensivo 30 minuti post-deploy
# Controllare: error rate, latenza, log errori
```

### STEP 8: DOCUMENT & UPDATE MEMORIES

```bash
# 1. Aggiornare CHANGELOG.md
echo "## $(date +%Y-%m-%d) - T<N> <descrizione>
- <cosa è cambiato>
- <impatto>
- <rollback se necessario>" >> CHANGELOG.md

# 2. Aggiornare BACKLOG.md (chiudere task completati)
# Marcare con ✅ e data

# 3. Aggiornare lexe-docs/ (se necessario)
# - ARCHITECTURE-REFERENCE.md
# - ADR se nuove decisioni

# 4. Aggiornare memorie Claude Code CLI
# Scrivere in CLAUDE.md o .claude/memory/:
# - Cosa è stato fatto
# - Decisioni prese
# - Problemi incontrati e soluzioni
# - Stato attuale post-modifica

# 5. Git commit documentazione
git add -A
git commit -m "docs(T<N>): aggiorna documentazione post-deploy <descrizione>"
git push
```

---

## 3. PIANO DI ROLLBACK UNIVERSALE

Ogni modifica ha un rollback immediato. **Tempo massimo rollback: 5 minuti.**

| Tipo Modifica | Rollback | Tempo |
|---|---|---|
| Feature flag | `LEXE_FF_<nome>=false` + restart container | < 1 min |
| Migration SQL | Eseguire comando rollback nel commento della migration | < 5 min |
| Codice Python | `git revert <commit>` + redeploy container | < 5 min |
| Frontend React | `git revert <commit>` + rebuild + redeploy | < 10 min |
| Config LiteLLM | Ripristinare `config.yaml` precedente + restart | < 2 min |
| Docker compose | `docker compose up -d --force-recreate <servizio>` con immagine precedente | < 3 min |

---

## 4. FASI DI ESECUZIONE

L'esecuzione è divisa in **4 fasi** per rispettare le dipendenze.

```
FASE 1 (Giorno 1-3)     FASE 2 (Giorno 4-7)     FASE 3 (Giorno 8-12)    FASE 4 (Giorno 13-16)
━━━━━━━━━━━━━━━━━━━     ━━━━━━━━━━━━━━━━━━━━     ━━━━━━━━━━━━━━━━━━━━     ━━━━━━━━━━━━━━━━━━━━
T4: DB migrations        T1: Unified classifier   T1: Self-Correction      T3: Confidence badges
T2: Rate limit+Retry     T3: System prompt v2     T3: Admin LEXORC roles   T5: SLA dashboard
T5: Gold set + baseline  T5: Auto test suite      T5: Load test 1000q/h    T5: Doc finale + KH
T1: FF legis + tokens    T2: Circuit breaker      T4: Alias cleanup        ALL: Hardening
```

---

## 5. TERMINALE 1 — TEAM PIPELINE & ROUTING

**Scope:** Pipeline LEGIS, classifier unificato, self-correction, feature flags  
**Repo principali:** `lexe-core` (agent/, gateway/, config.py)  
**Agente Lead:** `pipeline-architect`  
**Agenti Support:** `routing-specialist`, `llm-integration`

### FASE 1 — Stabilizzazione (S9.1 + S9.2)

#### Task T1-F1-A: Fix token limit lexe-fast

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/config.py
# Verificare il default di max_completion_tokens per lexe-fast
# Se hardcoded, preparare la migration (coordinare con T4)
```

> **NOTA:** La migration SQL (018_fix_lexe_fast_tokens.sql) è ownership di T4.  
> T1 prepara il contenuto, T4 lo crea e applica.

**Step 2 - TEST LOCAL:**
```bash
# Verificare che il config.py non ha errori di sintassi
python -c "from lexe_core.config import Settings; s = Settings(); print(s.ff_legis_agent)"
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Test: 100 query all'intent detector, zero parse error
python tests/test_intent_detector_tokens.py --count=100 --target=stage
# Gate: parse_error_count == 0
```

#### Task T1-F1-B: Attivare ff_legis_agent default

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/config.py
# Cambiare:
#   ff_legis_agent: bool = False
# In:
    ff_legis_agent: bool = True
# Override rollback: LEXE_FF_LEGIS_AGENT=false
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Verificare che staging usa LEGIS come default
curl -s https://api.stage.lexe.pro/health | jq '.features.legis_agent'
# Deve ritornare: true

# Test 20 query miste (legali + generiche)
python tests/test_pipeline_routing.py --count=20 --target=stage
# Gate: zero errori, routing corretto per almeno 18/20
```

**Rollback:** `LEXE_FF_LEGIS_AGENT=false` + restart lexe-core (< 1 min)

**Monitoring post-deploy prod:** 48h su error rate e latenza.

---

### FASE 2 — Unified Classifier (S10.1 + S10.2 + S10.3)

#### Task T1-F2-A: Fast-path euristica pre-routing

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/gateway/customer_router.py
# Aggiungere funzione _fast_route_check() PRIMA di _pre_intent_route()
# 
# Logica: se query < 8 parole E nessun hint legale → toolloop diretto
# Costo: 0ms, $0, zero LLM call
#
# Lista _LEGAL_HINTS (da Blueprint sezione 2.2):
_LEGAL_HINTS = [
    "articolo", "art.", "comma", "legge", "decreto", "codice",
    "sentenza", "cassazione", "tribunale", "corte", "giurisprudenza",
    "normativa", "regolamento", "contratto", "causa", "ricorso",
    "atto", "citazione", "parere", "responsabilità", "risarcimento",
    "danno", "inadempimento", "risoluzione", "recesso", "termine",
    "prescrizione", "decadenza", "competenza", "giurisdizione",
    "diritto", "obbligo", "dovere", "divieto", "sanzione",
    "mutuo", "fideiussione", "ipoteca", "pignoramento", "esecuzione",
    "fallimento", "concordato", "liquidazione",
    "dlgs", "d.lgs", "dpr", "d.p.r", "dm", "d.m",
    "c.c.", "c.p.", "c.p.c.", "c.p.p.",
    "gdpr", "lgpd", "privacy",
]

def _fast_route_check(message: str) -> str:
    """Pre-routing euristico zero-cost. Ritorna 'toolloop' o 'intent_detector'."""
    msg_lower = message.lower()
    word_count = len(message.split())
    has_legal = any(hint in msg_lower for hint in _LEGAL_HINTS)
    if word_count < 8 and not has_legal:
        return "toolloop"
    return "intent_detector"
```

#### Task T1-F2-B: Intent Detector Unificato

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/agent/pipeline.py
# Sostituire lo schema di output dell'intent detector
# per includere sia livello (ex-classifier) sia intent (ex-intent_detector)
#
# Schema unificato (da Blueprint sezione 2.2):
INTENT_DETECTOR_SCHEMA = {
    "type": "object",
    "properties": {
        "level": {"type": "string", "enum": ["DIRECT", "SIMPLE", "STANDARD", "COMPLEX"]},
        "scenario": {"type": "string", "enum": [
            "ricerca", "parere", "contratto_redazione",
            "contratto_revisione", "contenzioso", "recupero_crediti", "custom"
        ]},
        "route": {"type": "string", "enum": ["toolloop", "legis", "lexorc"]},
        "legal_domains": {"type": "array", "items": {"type": "string"}},
        "intent_summary": {"type": "string"},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1},
        "complexity_signals": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["level", "scenario", "route", "confidence"]
}

# NOTA: Usare Structured Outputs (response_format json_schema) via LiteLLM
# se supportato dal modello corrente. Altrimenti instructor come fallback.
```

```python
# File: lexe-core/src/lexe_core/agent/classifier.py
# DEPRECARE: classify_complexity() — ora ridondante
# NON eliminare subito, aggiungere deprecation warning:
import warnings

def classify_complexity(...):
    warnings.warn(
        "classify_complexity() è deprecata. Usare intent_detector unificato.",
        DeprecationWarning, stacklevel=2
    )
    # ... codice esistente come fallback temporaneo
```

#### Task T1-F2-C: Eliminare double classifier da customer_router

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/gateway/customer_router.py
# In _pre_intent_route():
# RIMUOVERE la chiamata a classify_complexity()
# SOSTITUIRE con _fast_route_check() + run_intent_detector() unificato
#
# Flow nuovo:
# 1. _fast_route_check(message) → "toolloop" | "intent_detector"
# 2. Se "toolloop" → stream diretto LiteLLM (zero LLM call per routing)
# 3. Se "intent_detector" → UNA singola LLM call che ritorna level+scenario+route
# 4. Route verso toolloop | legis | lexorc basandosi su intent.route
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Test gold set 200 query (preparato da T5)
python tests/test_unified_classifier.py \
  --gold-set=tests/data/gold_set_200.json \
  --target=stage

# Gate:
# - Accuracy >= 95% su gold set
# - Latenza Fase 0 P95 < 0.6s
# - Parse error rate < 1%
# - Zero regression su query non-legali (tutte a ToolLoop)
```

**Rollback:** `LEXE_FF_LEGIS_AGENT=false` ripristina il vecchio routing.

---

### FASE 3 — Self-Correction Loop (S12.1 + S12.3)

#### Task T1-F3-A: Self-Correction nel Verifier

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/agent/verifier.py
# Aggiungere loop di self-correction:
#
# Synthesis → Verifier →
#   PASS → risposta + badge verde
#   FAIL → retry_with_reflection(query, synthesis, issues)
#          → Nuova Synthesis → Verifier (tentativo 2)
#              PASS → risposta + badge verde
#              FAIL → risposta + badge giallo + disclaimer

MAX_SELF_CORRECTION_RETRIES = 2
SELF_CORRECTION_TIMEOUT_S = 15.0

async def verify_with_self_correction(
    query: str,
    synthesis: str,
    sources: list[dict],
    tenant_id: str,
) -> VerificationResult:
    """Verifica con self-correction (max 2 tentativi, timeout 15s)."""
    
    for attempt in range(MAX_SELF_CORRECTION_RETRIES):
        result = await verify_synthesis(query, synthesis, sources, tenant_id)
        
        if result.passed:
            return VerificationResult(
                synthesis=synthesis,
                confidence_level="VERIFIED",  # verde
                sources_verified=result.verified_sources,
                attempt=attempt + 1,
            )
        
        if attempt < MAX_SELF_CORRECTION_RETRIES - 1:
            # Retry con reflection
            synthesis = await retry_with_reflection(
                query=query,
                original_synthesis=synthesis,
                issues=result.issues,
                sources=sources,
                tenant_id=tenant_id,
            )
    
    # Dopo max retry, consegna con badge giallo
    return VerificationResult(
        synthesis=synthesis,
        confidence_level="CAUTION",  # giallo
        disclaimer="Alcune informazioni potrebbero richiedere verifica manuale",
        sources_verified=result.verified_sources,
        attempt=MAX_SELF_CORRECTION_RETRIES,
    )
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Test verifier pass rate su gold set
python tests/test_self_correction.py --gold-set=tests/data/gold_set_200.json --target=stage

# Gate:
# - Verifier pass rate > 90%
# - Source grounding coverage > 85%
# - Tempo self-correction < 15s per il 99% delle query
```

---

## 6. TERMINALE 2 — TEAM TOOLS & RESILIENZA

**Scope:** Rate limiting, retry, circuit breaker, tool executor  
**Repo principali:** `lexe-core` (tools/), `lexe-tools-it`  
**Agente Lead:** `resilience-engineer`  
**Agenti Support:** `tool-specialist`, `performance-optimizer`

### FASE 1 — Rate Limiting + Retry (S9.3 + S9.4)

#### Task T2-F1-A: Installare dipendenza tenacity

**Step 1 - DEVELOP:**
```bash
# File: lexe-core/requirements.txt
# Aggiungere:
# tenacity>=8.2.0

cd lexe-core
pip install tenacity --break-system-packages
echo "tenacity>=8.2.0" >> requirements.txt
```

#### Task T2-F1-B: Rate Limiting differenziato

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/tools/executor.py
# Aggiungere semafori differenziati per categoria tool
# Codice completo nella sezione 2.3 del Blueprint
#
# Categorie:
#   official (normattiva, eurlex, vigenza): Semaphore(6)
#   certified (infolex, kb_search, lex_search, etc.): Semaphore(4)
#   web (web_*): Semaphore(3)
#   default: Semaphore(4)
#
# NOTA: Usare feature flag per bypass rapido
# ff_tool_rate_limiting: bool = True (config.py)
```

#### Task T2-F1-C: Retry con exponential backoff

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/tools/executor.py
# Wrappare l'esecuzione tool con @retry di tenacity
#
# @retry(
#     stop=stop_after_attempt(3),
#     wait=wait_random_exponential(multiplier=1, min=1, max=10),
#     retry=retry_if_exception_type((ConnectionError, TimeoutError, OSError))
# )
# async def execute_tool_with_resilience(tool_name, params, tenant_id=None):
#     ... (codice completo nel Blueprint sezione 2.3)
```

#### Task T2-F1-D: Circuit Breaker

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/tools/executor.py
# Aggiungere circuit breaker pattern:
# - Threshold: 5 fallimenti consecutivi
# - Reset: 60 secondi dopo ultimo fallimento
# - CircuitOpenError per rifiuto immediato
# Codice completo nel Blueprint sezione 2.3
```

**Step 2 - TEST LOCAL:**
```bash
cd lexe-core
# Unit test specifici per resilience
pytest tests/test_tool_resilience.py -v --tb=short

# Test da creare:
# - test_rate_limiting_concurrent (50 chiamate parallele)
# - test_retry_on_timeout (simula timeout, verifica 3 tentativi)
# - test_circuit_breaker_opens (5 fail → circuit open)
# - test_circuit_breaker_resets (60s → circuit closed)
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Test 500 esecuzioni tool in parallelo
python tests/test_tool_load.py --count=500 --concurrency=50 --target=stage

# Gate:
# - Tool failure rate < 2% (target < 1% post-fase 2)
# - Zero crash del backend
# - Circuit breaker si attiva correttamente
# - 2/3 dei timeout transitori recuperati via retry
```

**Rollback:**
- Rate limiting: feature flag `ff_tool_rate_limiting=false`
- Retry: feature flag disabilita decorator
- Circuit breaker: feature flag bypassa check

---

### FASE 2 — JSON Parser Robusto

#### Task T2-F2-A: Creare json_safe.py

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/utils/json_safe.py (NUOVO)
# Parser JSON a 3 livelli per output intent detector
# 1. JSON standard parse
# 2. Pulizia markdown wrapping + retry
# 3. Estrazione regex dei campi critici + flag _parse_fallback
# Codice completo nel Blueprint sezione 2.2 (Fase C)
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Test parse su output reali dell'intent detector (100 campioni)
python tests/test_json_safe.py --samples=100 --target=stage
# Gate: parse success rate > 99%
```

---

## 7. TERMINALE 3 — TEAM UI/UX & PROMPT

**Scope:** Frontend webchat, admin panel, system prompt, prompt templates  
**Repo principali:** `lexe-webchat`, `lexe-admin`, `lexe-core` (prompts/)  
**Agente Lead:** `ux-architect`  
**Agenti Support:** `prompt-engineer`, `frontend-dev`

### FASE 1 — System Prompt v2 (S9.5 + Backlog P0)

#### Task T3-F1-A: Aggiornare prompt Studio mode

**Step 1 - DEVELOP:**
```python
# File: lexe-core/src/lexe_core/agent/prompts.py
# Rimuovere riferimenti a UI rimossa (sidebar, bottoni specifici)
# Riferimento: Blueprint sezione 2.4
```

#### Task T3-F1-B: Applicare System Prompt v2

**Step 1 - DEVELOP:**
```sql
-- Preparare script SQL per aggiornamento prompt
-- Riferimento: lexe-docs/system-prompts/lexe-legal-assistant-PROPOSED.md
--
-- Modifiche chiave:
-- 1. NUOVA sezione Citazioni Giurisprudenziali:
--    "NON citare numeri di sentenze senza verifica con tools"
-- 2. NUOVA sezione Red Flags:
--    "SEMPRE menzionare art. 1292 c.c. (solidarietà) per mutui/fideiussioni"
-- 3. Gestione Attendibilità: comportamento esplicito quando tools OFF
-- 4. Disclaimer automatico quando tools disabilitati

-- NOTA: Questo SQL viene passato a T4 per esecuzione nella migration
-- T3 prepara il contenuto, T4 crea il file migration
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Test regressione prompt:
# 1. Query su mutuo cointestato → deve menzionare responsabilità solidale
# 2. Query con tools OFF → deve mostrare disclaimer citazioni
# 3. Query generica → nessun riferimento a UI rimossa
python tests/test_system_prompt_v2.py --target=stage
```

---

### FASE 3 — Confidence Badges UI (S11.3)

#### Task T3-F3-A: Componente ConfidenceBadge

**Step 1 - DEVELOP:**
```typescript
// File: lexe-webchat/src/components/chat/ConfidenceBadge.tsx (NUOVO)
// 
// 3 livelli di confidence con colori brand:
// VERIFIED  → Verde #10B981 → Icona: scudo con check
// CAUTION   → Giallo #F59E0B → Icona: triangolo attenzione  
// LOW_CONFIDENCE → Rosso #EF4444 → Icona: cerchio esclamativo
//
// Props: { level: "VERIFIED" | "CAUTION" | "LOW_CONFIDENCE", tooltip?: string }
// 
// Il badge appare accanto a ogni risposta LEGIS
// Feature flag frontend: ff_show_confidence_badge
```

```typescript
// File: lexe-webchat/src/components/chat/MessageBubble.tsx
// Integrare ConfidenceBadge nel rendering del messaggio
// Leggere confidence_level dal metadata della risposta SSE
```

#### Task T3-F3-B: Admin Panel — Ruoli LEXORC

**Step 1 - DEVELOP:**
```typescript
// File: lexe-admin/src/pages/ModelsPage.tsx
// Verificare che i nuovi ruoli LEXORC (lexorc_classifier, lexorc_norm_agent, etc.)
// siano visibili e modificabili nella tab Models del Tenant Editor
// La migration DB è responsabilità di T4 (019_lexorc_role_defaults.sql)
```

**Step 5 - AUTO TEST STAGING:**
```bash
# Test visivo su stage-chat.lexe.pro:
# 1. Inviare query legale → badge confidence visibile
# 2. Verificare colore corretto (verde/giallo/rosso)
# 3. Tooltip con spiegazione al hover

# Test admin su admin.stage.lexe.pro:
# 1. Aprire tenant → tab Models
# 2. Verificare ruoli LEXORC presenti
# 3. Cambiare modello per un ruolo LEXORC → salva → verifica
```

---

## 8. TERMINALE 4 — TEAM DATA & DB

**Scope:** Migrations SQL, configurazione LiteLLM, Docker, memory  
**Repo principali:** `lexe-core` (migrations/), `lexe-infra`, `lexe-memory`  
**Agente Lead:** `data-architect`  
**Agenti Support:** `dba-specialist`, `infra-ops`

### FASE 1 — Migrations Critiche

#### Task T4-F1-A: Migration 018 — Fix token lexe-fast

**Step 1 - DEVELOP:**
```sql
-- File: lexe-core/src/lexe_core/migrations/018_fix_lexe_fast_tokens.sql
-- Descrizione: Aumenta token limit per lexe-fast da 250 a 1024
-- Rollback: UPDATE core.llm_model_catalog SET config = jsonb_set(COALESCE(config,'{}'), '{max_completion_tokens}', '250') WHERE model_name = 'lexe-fast';

BEGIN;

UPDATE core.llm_model_catalog
SET config = jsonb_set(COALESCE(config, '{}'), '{max_completion_tokens}', '1024')
WHERE model_name = 'lexe-fast';

COMMIT;
```

**Step 2 - TEST LOCAL:**
```bash
# Testare su DB locale/staging
psql -h localhost -p 5435 -U lexe -d lexe_db -f migrations/018_fix_lexe_fast_tokens.sql

# Verificare
psql -h localhost -p 5435 -U lexe -d lexe_db -c \
  "SELECT model_name, config->>'max_completion_tokens' FROM core.llm_model_catalog WHERE model_name = 'lexe-fast';"
# Deve ritornare: lexe-fast | 1024

# Testare rollback
psql -h localhost -p 5435 -U lexe -d lexe_db -c \
  "UPDATE core.llm_model_catalog SET config = jsonb_set(COALESCE(config,'{}'), '{max_completion_tokens}', '250') WHERE model_name = 'lexe-fast';"
```

#### Task T4-F1-B: Backup pre-operazioni

**Step 1 - DEVELOP:**
```bash
# PRIMA di qualsiasi migration, backup completo
ssh lexe-server "pg_dump -h localhost -p 5435 -U lexe -d lexe_db \
  --schema=core --schema=kb \
  | gzip > /backup/lexe_pre_sprint_$(date +%Y%m%d_%H%M).sql.gz"

# Backup config LiteLLM
ssh lexe-server "cp /opt/lexe/litellm/config.yaml \
  /backup/litellm_config_$(date +%Y%m%d_%H%M).yaml"
```

---

### FASE 3 — LEXORC Roles + Alias Cleanup

#### Task T4-F3-A: Migration 019 — LEXORC roles

**Step 1 - DEVELOP:**
```sql
-- File: lexe-core/src/lexe_core/migrations/019_lexorc_role_defaults.sql
-- Descrizione: Aggiunge i 6 ruoli LEXORC a model_role_defaults
-- Rollback: DELETE FROM core.model_role_defaults WHERE role LIKE 'lexorc_%';

BEGIN;

INSERT INTO core.model_role_defaults (role, model_name, display_label, description) VALUES
    ('lexorc_classifier',     'lexe-fast',         'LEXORC Classifier',    'Classificatore complessità query'),
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

#### Task T4-F3-B: Cleanup alias duplicati LiteLLM

**Step 1 - DEVELOP:**
```bash
# PRIMA: verificare se qualcuno usa gli alias da rimuovere
psql -h localhost -p 5435 -U lexe -d lexe_db -c \
  "SELECT DISTINCT model_name FROM core.tenant_model_roles
   WHERE model_name IN ('legal-tier1-gemini', 'legal-tier2-gemini', 'legal-tier3-gpt5');"

# Se risultati vuoti → safe to remove
# Se ci sono tenant che li usano → UPDATE prima di rimuovere

# File: lexe-infra/litellm/config.yaml
# Rimuovere alias duplicati (legal-tier1-gemini, legal-tier2-gemini, legal-tier3-gpt5)
# Mantenere solo: lexe-fast, lexe-primary, lexe-complex, legal-tier2-haiku, legal-tier3-sonnet, lexe-embedding
```

---

## 9. TERMINALE 5 — TEAM QA, MONITORING & DOCUMENTAZIONE

**Scope:** Gold set, test automatici, metriche, monitoring, documentazione  
**Repo principali:** `lexe-docs`, `tests/`, cross-repo  
**Agente Lead:** `qa-lead`  
**Agenti Support:** `metrics-analyst`, `doc-writer`

### FASE 1 — Baseline & Gold Set

#### Task T5-F1-A: Catturare baseline metriche

**Step 1 - DEVELOP:**
```bash
# Script per catturare baseline PRIMA delle modifiche
cat > tests/capture_baseline.py << 'EOF'
"""Cattura baseline metriche per confronto post-sprint."""
import asyncio
import json
import time
import httpx
from datetime import datetime

STAGE_URL = "https://api.stage.lexe.pro"
QUERIES_SAMPLE = [
    "Cos'è la responsabilità solidale?",
    "Art. 2043 codice civile",
    "Ciao, come stai?",
    "Quali sono i termini di prescrizione per danni da prodotto?",
    "Calcola il 22% di 1500",
    # ... espandere a 50 per baseline
]

async def measure():
    results = []
    async with httpx.AsyncClient(timeout=60) as client:
        for q in QUERIES_SAMPLE:
            start = time.time()
            try:
                r = await client.post(f"{STAGE_URL}/api/v1/gateway/customer/stream",
                    json={"message": q}, headers={"Authorization": "Bearer <token>"})
                latency = time.time() - start
                results.append({"query": q, "status": r.status_code, "latency_s": latency})
            except Exception as e:
                results.append({"query": q, "status": "error", "error": str(e)})
    
    baseline = {
        "timestamp": datetime.now().isoformat(),
        "total_queries": len(results),
        "errors": sum(1 for r in results if r["status"] != 200),
        "avg_latency_s": sum(r.get("latency_s", 0) for r in results) / len(results),
        "results": results,
    }
    
    with open("tests/data/baseline_metrics.json", "w") as f:
        json.dump(baseline, f, indent=2)
    print(f"Baseline salvata: {len(results)} query, {baseline['errors']} errori, {baseline['avg_latency_s']:.2f}s avg")

asyncio.run(measure())
EOF
python tests/capture_baseline.py
```

#### Task T5-F1-B: Creare Gold Set 200 query

**Step 1 - DEVELOP:**
```bash
# Creare il gold set per validazione del classifier unificato
# Struttura: tests/data/gold_set_200.json
#
# Composizione (da Blueprint sezione 3 Sprint 10):
# - 80 query legali tipiche (ricerca normativa, giurisprudenza, pareri)
# - 40 query Studio mode (saluti, domande generiche, calcoli)
# - 30 query edge case (miste legale/generico, ambigue)
# - 20 query multi-dominio (cross-reference tra aree)
# - 15 query con riferimenti normativi espliciti (art. X, d.lgs. Y)
# - 15 query in linguaggio naturale senza hint legali espliciti
#
# Formato per ogni entry:
# {
#   "id": "GS-001",
#   "query": "Quali sono i presupposti dell'art. 2043 c.c.?",
#   "expected_route": "legis",
#   "expected_level": "STANDARD",
#   "expected_scenario": "ricerca",
#   "category": "legal_typical",
#   "notes": "Query standard di ricerca normativa"
# }
```

---

### FASE 2 — Suite di Test Automatici

#### Task T5-F2-A: Framework test E2E staging

**Step 1 - DEVELOP:**
```python
# File: tests/e2e/test_staging_suite.py
# Suite di test automatici da lanciare dopo ogni deploy su staging
#
# Test inclusi:
# 1. Health check tutti i servizi
# 2. Login flow (Logto)
# 3. Invio query legale → risposta con citazioni
# 4. Invio query generica → routing toolloop
# 5. Test tool execution (normattiva, kb_search)
# 6. Test streaming SSE (no troncamento)
# 7. Test memory extraction (fatto corretto, non "rimasta")
# 8. Test confidence badge (se feature flag attivo)
#
# Output: report JSON con pass/fail per ogni test
# Gate: TUTTI i test devono passare per procedere a Step 6
```

---

### FASE 3 — Load Test & SLA

#### Task T5-F3-A: Load test 1000 query/ora

**Step 1 - DEVELOP:**
```bash
# Usare locust o script custom per load test
pip install locust --break-system-packages

cat > tests/load/locustfile.py << 'EOF'
from locust import HttpUser, task, between

class LexeUser(HttpUser):
    wait_time = between(1, 5)
    
    @task(3)
    def legal_query(self):
        self.client.post("/api/v1/gateway/customer/stream",
            json={"message": "Quali sono i termini di prescrizione?"})
    
    @task(1)
    def generic_query(self):
        self.client.post("/api/v1/gateway/customer/stream",
            json={"message": "Ciao, come va?"})
EOF

# Esecuzione: 1000 query/ora = ~17 query/min = ~0.3 query/sec
locust -f tests/load/locustfile.py \
  --host=https://api.stage.lexe.pro \
  --users=5 --spawn-rate=1 --run-time=60m \
  --headless --csv=tests/results/load_test
```

**Gate:**
- Zero errori 5xx sotto carico
- Latenza P95 < 30s per query legali
- Latenza P95 < 5s per query generiche
- Memory usage stabile (no leak)

---

### FASE 4 — Documentazione Finale & Knowledge Hub

#### Task T5-F4-A: Creare knowledge hub

**Step 1 - DEVELOP:**
```bash
# Struttura lexe-docs/knowhow/
mkdir -p lexe-docs/knowhow/{adr,runbooks,postmortem}

# ADR da documentare:
# - ADR-D1: Unified classifier
# - ADR-D2: Semafori differenziati
# - ADR-D3: LEXORC roles via DB
# - ADR-D4: ff_legis_agent default true
# - ADR-D5: Confidence spectrum 3 livelli
# - ADR-D6: Self-correction max 2 retry

# Runbook operativi:
# - RUNBOOK-deploy.md: procedura deploy stage → prod
# - RUNBOOK-rollback.md: rollback per ogni componente
# - RUNBOOK-monitoring.md: cosa controllare e quando

# Template postmortem:
# - TEMPLATE-postmortem.md
```

#### Task T5-F4-B: Aggiornare memorie Claude Code

```bash
# File: .claude/memory/lexe-sprint-update.md
# Aggiornare con lo stato post-sprint:
# - Feature flag attivi e loro stato
# - Migration applicate (fino a 019)
# - Metriche post-sprint vs baseline
# - Problemi noti e workaround
# - Decisioni architetturali prese
```

---

## 10. MATRICE DIPENDENZE TRA TERMINALI

```
T4-F1-A (migration 018) ──────────► T1-F1-A (token fix richiede migration applicata)
T4-F1-B (backup)        ──────────► TUTTI (nessuno parte senza backup)
T5-F1-B (gold set)      ──────────► T1-F2-C (classifier unificato richiede gold set)
T1-F2-B (intent schema) ──────────► T2-F2-A (json_safe.py deve parsare il nuovo schema)
T1-F3-A (self-correction)─────────► T3-F3-A (confidence badge richiede confidence_level nel SSE)
T4-F3-A (migration 019) ──────────► T3-F3-B (admin LEXORC roles richiede DB seed)
```

**Ordine di avvio per fase:**

| Fase | Prima | Poi | Infine |
|---|---|---|---|
| **F1** | T4 (backup + migrations) | T2 (resilience) + T5 (baseline) | T1 (FF + config) + T3 (prompts) |
| **F2** | T5 (gold set pronto) | T1 (classifier) + T2 (json parser) | T3 (prompt v2 test) |
| **F3** | T4 (migration 019) | T1 (self-correction) | T3 (badges + admin) |
| **F4** | T5 (load test) | T3 (UI hardening) | T5 (doc finale) |

---

## 11. COMANDI RAPIDI PER TERMINALE

### Setup iniziale (da eseguire su OGNI terminale)

```bash
# Clone e setup
cd /opt/lexe
git fetch --all
git checkout staging
git pull origin staging

# Variabili comuni
export LEXE_STAGE_URL="https://api.stage.lexe.pro"
export LEXE_PROD_URL="https://api.lexe.pro"
export LEXE_DB_HOST="localhost"
export LEXE_DB_PORT="5435"
export LEXE_DB_USER="lexe"
export LEXE_DB_NAME="lexe_db"
```

### Comandi per deploy rapido

```bash
# Deploy singolo servizio su staging
deploy_stage() {
  local service=$1
  ssh lexe-server "cd /opt/lexe && \
    docker compose -f docker-compose.yml \
    -f docker-compose.override.staging.yml \
    up -d --build $service"
}

# Deploy singolo servizio su produzione (SOLO dopo OK utente)
deploy_prod() {
  local service=$1
  echo "⚠️  DEPLOY PRODUZIONE: $service"
  echo "Conferma? (y/N)"
  read confirm
  if [ "$confirm" = "y" ]; then
    ssh lexe-server "cd /opt/lexe && \
      docker compose -f docker-compose.yml \
      -f docker-compose.override.prod.yml \
      up -d --build $service"
  fi
}

# Rollback rapido (ripristina immagine precedente)
rollback() {
  local service=$1
  ssh lexe-server "cd /opt/lexe && \
    docker compose -f docker-compose.yml \
    -f docker-compose.override.prod.yml \
    up -d --force-recreate $service"
}

# Health check
check_health() {
  local url=$1
  curl -s "$url/health" | jq '.status'
}
```

---

## 12. CHECKLIST PRE-AVVIO

### Prima di aprire qualsiasi terminale:

- [ ] Backup DB completo (`pg_dump` core + kb)
- [ ] Backup `litellm/config.yaml`
- [ ] Snapshot Docker containers (`docker commit`)
- [ ] Verificare spazio disco server (> 20GB liberi)
- [ ] Verificare che staging è allineato con prod
- [ ] Canale comunicazione rapido con Frisco attivo
- [ ] Tutti i feature flag documentati e testabili
- [ ] `tenacity` installato in lexe-core
- [ ] Gold set iniziale pronto (almeno 50 query per Fase 1)
- [ ] Baseline metriche catturata

### Dopo ogni fase completata:

- [ ] Tutti i gate di fase superati
- [ ] Zero errori in produzione per 24h
- [ ] BACKLOG.md aggiornato (task chiusi)
- [ ] CHANGELOG.md aggiornato
- [ ] Memorie Claude Code aggiornate
- [ ] Metriche confrontate con baseline

---

## 13. METRICHE DI SUCCESSO (da Blueprint)

| Metrica | Baseline | Target Post-Sprint | Misurato |
|---|---|---|---|
| Latenza Fase 0 (routing) | 2.5-3.5s | < 0.6s | ☐ |
| LLM calls per query legale | 3-4 | 2 | ☐ |
| Tool failure rate | ~3% | < 1% | ☐ |
| Parse error rate JSON | ~5% | < 1% | ☐ |
| Verifier pass rate | TBD | > 90% | ☐ |
| Source grounding coverage | TBD | > 85% | ☐ |
| Costo medio per query | TBD | < $0.002 | ☐ |
| Gold set accuracy | TBD | >= 95% | ☐ |

---

## 14. RIEPILOGO SPRINT → TERMINALI

| Sprint | Azione | T1 | T2 | T3 | T4 | T5 |
|---|---|:---:|:---:|:---:|:---:|:---:|
| **S9** | Token fix lexe-fast | | | | ✅ | |
| **S9** | Attivare ff_legis_agent | ✅ | | | | |
| **S9** | Retry tool esterni | | ✅ | | | |
| **S9** | Rate limiting executor | | ✅ | | | |
| **S9** | Prompt Studio update | | | ✅ | | |
| **S9** | Baseline metriche | | | | | ✅ |
| **S9** | Gold set 50 query | | | | | ✅ |
| **S10** | Unified classifier A+B+C | ✅ | | | | |
| **S10** | JSON validation fallback | | ✅ | | | |
| **S10** | Gold set 200 query | | | | | ✅ |
| **S10** | System Prompt v2 | | | ✅ | | |
| **S10** | Test suite E2E | | | | | ✅ |
| **S11** | Self-correction loop | ✅ | | | | |
| **S11** | LEXORC roles DB | | | | ✅ | |
| **S11** | Confidence badges UI | | | ✅ | | |
| **S11** | Load test 1000q/h | | | | | ✅ |
| **S11** | Alias cleanup LiteLLM | | | | ✅ | |
| **S12** | Enhanced verifier | ✅ | | | | |
| **S12** | Admin LEXORC roles UI | | | ✅ | | |
| **S12** | SLA dashboard | | | | | ✅ |
| **S12** | Knowledge hub + docs | | | | | ✅ |

---

> **Documento compilato per Claude Code CLI**  
> Esecuzione: 5 terminali paralleli, 4 fasi sequenziali, ~16 giorni lavorativi  
> Ogni terminale è autonomo ma rispetta le dipendenze della matrice (Sezione 10)  
> Rollback disponibile per ogni singola modifica (Sezione 3)  
> Frisco approva Step 6 (User Test) prima di ogni deploy in produzione  
