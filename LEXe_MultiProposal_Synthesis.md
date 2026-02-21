# LEXe - Sintesi Multi-Proposta e Strategia di Implementazione Ottimale

> Documento generato il 20/02/2026
> Fonte: 5 analisi parallele (Gemini 3.1 Pro, Qwen 3.5 Plus, MiniMax M2.5, Qwen3-Max-Thinking, DeepSeek V3.2)
> Contesto: Vision LEXe + Audit Tecnico 20/02/2026

---

## 1. QUADRO SINOTTICO DELLE PROPOSTE

### 1.1 Matrice di confronto per dimensione

| Dimensione | Gemini 3.1 Pro | Qwen 3.5 Plus | MiniMax M2.5 | Qwen3-Max | DeepSeek V3.2 |
|---|---|---|---|---|---|
| **Profondita tecnica** | Eccellente - formula RRF, analisi matematica routing | Molto buona - schema SQL, code snippets Python | Eccellente - 8 fasi strutturate, dettaglio implementativo | Buona - focus architetturale, diagrammi Mermaid | Buona - sintesi chiara, pragmatica |
| **Vision strategica** | Alta - moat analysis, architettura deterministico-quantistica | Media - focus operativo, KPI solidi | Alta - AI First Maturity Model a 4 livelli | Molto alta - Unified Agent Framework, KB come nervous system | Media - focus su allineamento vision-implementazione |
| **Actionability** | Media - 4 step roadmap generici | Molto alta - sprint plan con ore stimate | Molto alta - 4 fasi con file e righe da modificare | Alta - 3 fasi con metriche di successo | Molto alta - 8 azioni ordinate per priorita |
| **Pricing/GTM** | Assente | Presente - tabella tier + costo per query | Assente | Assente | Assente |
| **Competitive analysis** | Limitata ma efficace | Presente - matrice vs Harvey/Casetext | Presente - benchmark claims vs readiness | Presente - metriche vs standard di mercato | Assente |
| **Originalita insight** | Formula RRF, self-correction loop | Costo per query per pipeline | AI First Maturity Model, jurisdiction expansion | KB come nervous system, Blockchain Legal Certificate | Feature flag ff_legis_agent come default |
| **Rischi identificati** | 4 rischi tecnici dettagliati | 7 problemi con soluzioni | 7 problemi con impatto UX mappato | 5 aree critiche con gap vs vision | 8 problemi ordinati per priorita |

### 1.2 Punteggio comparativo (scala 1-5)

| Criterio | Gemini | Qwen 3.5+ | MiniMax | Qwen3-Max | DeepSeek |
|---|---|---|---|---|---|
| Comprensione architettura | 5 | 4 | 5 | 4 | 4 |
| Qualita soluzioni tecniche | 5 | 4 | 5 | 3 | 4 |
| Roadmap eseguibile | 3 | 5 | 5 | 4 | 5 |
| Innovazione proposta | 4 | 3 | 4 | 5 | 3 |
| Completezza GTM | 1 | 4 | 2 | 2 | 1 |
| Allineamento con progetto LEXe | 4 | 4 | 5 | 4 | 4 |
| **MEDIA** | **3.7** | **4.0** | **4.3** | **3.7** | **3.5** |

**Classifica complessiva:** MiniMax M2.5 > Qwen 3.5 Plus > Gemini 3.1 Pro = Qwen3-Max > DeepSeek V3.2

---

## 2. BEST-OF-BREED: LE SOLUZIONI MIGLIORI SELEZIONATE E MIGLIORATE

### 2.1 PROBLEMA P1 - Double Classifier (CRITICO)

**Migliore proposta originale:** MiniMax M2.5 (dettaglio implementativo completo)
**Migliorata con:** Gemini (euristica a costo zero), Qwen3-Max (metriche di successo)

**Soluzione ottimale:**

Eliminare `classify_complexity()` come chiamata LLM separata. Mantenere `_pre_intent_route()` come filtro deterministico a costo zero, poi delegare tutto a `run_intent_detector()` con output esteso.

**Pre-filtro deterministico (0ms, $0):**

```python
# pipeline.py - _pre_intent_route() semplificato
_LEGAL_HINTS = {"art.", "articolo", "comma", "legge", "decreto", "sentenza",
                "cassazione", "codice", "tribunale", "corte", "giurisprudenza",
                "responsabilita", "risarcimento", "contratto", "inadempimento"}

def _pre_intent_route(message: str) -> str:
    msg_lower = message.lower()
    word_count = len(message.split())
    has_legal = any(h in msg_lower for h in _LEGAL_HINTS)

    if word_count < 8 and not has_legal:
        return "toolloop"  # bypass LLM: "Ciao", "Che ore sono?"
    return "intent_detector"  # passa sempre al detector unificato
```

**Intent Detector unificato (singola LLM call):**

```json
{
  "level": "DIRECT|SIMPLE|STANDARD|COMPLEX",
  "scenario": "ricerca|parere|contratto_redazione|contenzioso|recupero_crediti|generico",
  "route": "toolloop|legis|lexorc",
  "legal_domains": ["civile", "penale"],
  "intent_summary": "L utente chiede...",
  "confidence": 0.87,
  "complexity_signals": ["multi_dominio", "giurisprudenza_richiesta"]
}
```

**Metriche di successo post-implementazione:**

| Metrica | Attuale | Target | Verifica |
|---|---|---|---|
| Latenza Fase 0 (routing) | 2.5-3.5s | < 0.6s | P95 su 1000 query test |
| Costo routing/query | ~$0.0002 | ~$0.0001 | Dashboard OpenRouter |
| Parse error rate (JSON) | ~5% | < 1% | Log errori 7 giorni |
| Classificazione corretta | ~90% (stimata) | > 95% | Gold set 200 query |

**Owner:** Frisco (Tech Lead)
**Effort:** 16 ore
**Sprint:** 9 (immediato)

---

### 2.2 PROBLEMA P6 - Token Limit Insufficiente per lexe-fast

**Migliore proposta:** Consenso unanime (tutte e 5)
**Miglioramento:** aggiungere validazione JSON con fallback graceful

```sql
-- migration_015_fix_lexe_fast_tokens.sql
UPDATE core.model_catalog
SET config = jsonb_set(config, '{max_completion_tokens}', '1024')
WHERE alias = 'lexe-fast';
```

**Aggiunta: JSON validation con fallback**

```python
# utils/json_safe.py
import json
from typing import Any

def parse_intent_json(raw: str, fallback_level: str = "STANDARD") -> dict:
    """Parse JSON dell intent detector con fallback robusto."""
    try:
        # Rimuovi eventuale markdown wrapping
        cleaned = raw.strip()
        if cleaned.startswith("```"):
            cleaned = cleaned.split("\n", 1)[1].rsplit("```", 1)[0]
        return json.loads(cleaned)
    except json.JSONDecodeError:
        # Tentativo di estrazione parziale con regex
        import re
        level_match = re.search(r'"level"\s*:\s*"(\w+)"', raw)
        scenario_match = re.search(r'"scenario"\s*:\s*"(\w+)"', raw)
        return {
            "level": level_match.group(1) if level_match else fallback_level,
            "scenario": scenario_match.group(1) if scenario_match else "ricerca",
            "route": "legis",
            "confidence": 0.3,  # segnala bassa confidenza
            "_parse_fallback": True
        }
```

**Owner:** Frisco
**Effort:** 2 ore
**Sprint:** 9

---

### 2.3 PROBLEMA P2 - LEXORC Models Hardcoded

**Migliore proposta:** Qwen 3.5 Plus (migrazione SQL completa)
**Migliorata con:** MiniMax (config.py refactoring), DeepSeek (cleanup alias)

```sql
-- migration_016_lexorc_roles.sql
BEGIN;

INSERT INTO core.model_role_defaults (role, model_name, description, created_at) VALUES
  ('lexorc_classifier',     'lexe-fast',          'LEXORC: classificatore complessita query',     NOW()),
  ('lexorc_norm_agent',     'lexe-primary',       'LEXORC: agente normativo (codici, articoli)',   NOW()),
  ('lexorc_caselaw_agent',  'lexe-primary',       'LEXORC: agente giurisprudenza (massime)',       NOW()),
  ('lexorc_doctrine_agent', 'lexe-primary',       'LEXORC: agente dottrinale (annotazioni)',       NOW()),
  ('lexorc_auditor',        'legal-tier2-haiku',  'LEXORC: auditor coerenza e anti-hallucination', NOW()),
  ('lexorc_synthesizer',    'lexe-primary',       'LEXORC: sintetizzatore risposta finale',        NOW())
ON CONFLICT (role) DO NOTHING;

-- Cleanup alias duplicati (P3)
-- Verificare in litellm/config.yaml e rimuovere:
-- legal-tier1-gemini (duplicato di lexe-fast)
-- legal-tier2-gemini (duplicato di lexe-fast)
-- legal-tier3-gpt5   (duplicato di lexe-complex)

COMMIT;
```

**Refactoring config.py:**

```python
# config.py - PRIMA (hardcoded)
# lexorc_norm_agent_model: str = os.getenv("LEXE_LEXORC_NORM_AGENT_MODEL", "lexe-primary")

# config.py - DOPO (via role system)
def get_model_for_lexorc_role(role: str, tenant_id: str | None = None) -> str:
    """Risolve il modello per un ruolo LEXORC via DB, con fallback."""
    return resolve_model_for_role(f"lexorc_{role}", tenant_id=tenant_id)
```

**Impatto strategico:** consente ai tenant Enterprise di scegliere i propri modelli (es. solo Claude per compliance interna) senza richiedere riavvio o deploy.

**Owner:** Frisco
**Effort:** 8 ore
**Sprint:** 10

---

### 2.4 PROBLEMA P4 + P7 - Rate Limiting e Retry

**Migliore proposta:** MiniMax M2.5 (semafori differenziati per tool)
**Migliorata con:** Gemini (backoff esponenziale), Qwen3-Max (circuit breaker)

```python
# tools/executor.py - Rate limiting + Retry unificato

import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

# Semafori per categoria di tool
_semaphores = {
    "official": asyncio.Semaphore(6),   # normattiva, eurlex (alta priorita)
    "certified": asyncio.Semaphore(4),  # brocardi, infolex
    "web": asyncio.Semaphore(3),        # scraping generico
    "default": asyncio.Semaphore(4),
}

# Circuit breaker semplice
_failure_counts: dict[str, int] = {}
_CIRCUIT_THRESHOLD = 5
_CIRCUIT_RESET_SECONDS = 60

def _get_semaphore(tool_name: str) -> asyncio.Semaphore:
    if tool_name in ("normattiva_search", "eurlex_search"):
        return _semaphores["official"]
    elif tool_name in ("brocardi_search", "infolex_search"):
        return _semaphores["certified"]
    elif tool_name.startswith("web_"):
        return _semaphores["web"]
    return _semaphores["default"]

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type((ConnectionError, TimeoutError))
)
async def execute_tool_with_retry(tool_name: str, params: dict, tenant_id: str) -> dict:
    # Circuit breaker check
    if _failure_counts.get(tool_name, 0) >= _CIRCUIT_THRESHOLD:
        raise CircuitOpenError(f"Tool {tool_name} circuit open dopo {_CIRCUIT_THRESHOLD} fallimenti")

    sem = _get_semaphore(tool_name)
    async with sem:
        try:
            result = await _execute_tool_impl(tool_name, params, tenant_id)
            _failure_counts[tool_name] = 0  # reset on success
            return result
        except Exception as e:
            _failure_counts[tool_name] = _failure_counts.get(tool_name, 0) + 1
            raise
```

**Owner:** Frisco
**Effort:** 6 ore (rate limiting) + 4 ore (retry)
**Sprint:** 9-10

---

### 2.5 ARCHITETTURA - Evoluzione verso Unified Agent Framework

**Migliore proposta:** Qwen3-Max (vision architetturale)
**Migliorata con:** MiniMax (AI First Maturity Model), Gemini (self-correction loop)

Questa e una proposta a medio termine (Sprint 13+) che fonde le intuizioni piu innovative.

**Concetto: da LEGIS/LEXORC separati a LEXe Agent Platform unificata**

L'idea chiave di Qwen3-Max e trasformare la separazione LEGIS/LEXORC in un unico framework dove gli agenti (NormAgent, CaseLawAgent, DoctrineAgent, ContractAgent) vengono attivati dinamicamente dall'orchestratore in base alla complessita.

**AI First Maturity Model (da MiniMax) applicato alla roadmap:**

| Livello | Nome | Descrizione | Target |
|---|---|---|---|
| L1 | Rule-Based Routing | Euristica lessicale (attuale ToolLoop) | Presente |
| L2 | LLM-Driven Routing | Intent detector unificato decide pipeline | Sprint 9 (post H-A) |
| L3 | Self-Optimizing | Routing evolve in base a success rate storici | Sprint 16+ |
| L4 | Proactive Agentic | Il sistema anticipa i bisogni dell'utente | Q2 2027 |

**Self-Correction Loop (da Gemini) integrato nel Verifier:**

```
Verifier → {status: fail, issues: [...]}
       ↓
  retry_with_reflection(original_query, synthesis, issues)
       ↓
  Nuovo synthesis corretto → Verifier (max 2 tentativi)
       ↓
  Se ancora fail → risposta con disclaimer + badge giallo
```

**Knowledge Graph come Nervous System (da Qwen3-Max):**

Il citation graph (58.737 edge) e il norm graph (42.338 link) non devono servire solo la ricerca. Devono guidare le decisioni dell'orchestratore. Esempio: se la query contiene "art. 2043 c.c.", il sistema attiva CaseLawAgent con query pre-calcolata dal graph traversal, senza bisogno del planner LLM.

```python
# Pseudo-codice: graph-aware routing
async def graph_enhanced_routing(query: str, intent: dict) -> dict:
    """Arricchisce il piano di esecuzione con informazioni dal grafo."""
    # Estrai riferimenti normativi dalla query
    norm_refs = extract_norm_references(query)  # ["CC:2043", "DLgs:81/2008:art.18"]

    if norm_refs:
        # Traversa il grafo per trovare massime collegate
        related_massime = await norm_graph.get_connected_massime(norm_refs, max_hops=2)
        related_norms = await citation_graph.get_citing_norms(norm_refs)

        return {
            "pre_fetched_norms": norm_refs,
            "graph_massime_ids": [m.id for m in related_massime[:20]],
            "graph_related_norms": related_norms,
            "skip_broad_search": len(related_massime) > 10,  # gia abbastanza fonti
        }
    return {}
```

---

### 2.6 GTM e PRICING

**Unica proposta con pricing:** Qwen 3.5 Plus
**Migliorata con:** dati dal project knowledge (Ops/Backlog e Sales Kit)

La proposta Qwen 3.5 Plus suggerisce tier da 49-499 euro, ma il progetto LEXe ha gia stabilito un posizionamento mid-premium allineato a Lexroom (~85 euro/utente/mese base). La sintesi ottimale integra i costi per pipeline calcolati da Qwen 3.5 Plus con i prezzi gia definiti.

**Costo per query per pipeline (validato):**

| Pipeline | LLM Calls post-H-A | Token medi | Costo stimato |
|---|---|---|---|
| ToolLoop (Studio) | 1-2 | ~2K | ~$0.0004 |
| LEGIS DIRECT | 1 | ~1.5K | ~$0.0003 |
| LEGIS STANDARD | 3 | ~6K | ~$0.0012 |
| LEGIS COMPLEX | 4 | ~12K | ~$0.0024 |
| LEXORC | 6+ | ~20K | ~$0.0040 |

**Margine lordo stimato per tier (allineato a prezzi LEXe correnti):**

Ipotizzando ~85 euro/utente/mese (Starter) e un mix medio di 50 query LEGIS/giorno per utente: costo LLM mensile stimato per utente ~2-4 euro, con margine lordo > 95%. Questo conferma la sostenibilita del modello di pricing attuale.

---

### 2.7 PROPOSTE INNOVATIVE DA INTEGRARE NELLA ROADMAP

Queste idee non risolvono problemi dell'audit ma aggiungono differenziazione competitiva.

| Idea | Fonte | Impatto | Fattibilita | Priorita suggerita |
|---|---|---|---|---|
| Self-Correction Loop nel Verifier | Gemini | Qualita output +15-20% | Media (Sprint 13) | Alta |
| Graph-Aware Routing (KB come nervous system) | Qwen3-Max | Latenza -30%, qualita +20% | Alta (Sprint 14) | Alta |
| Blockchain Legal Certificate (audit trail) | Qwen3-Max | Differenziazione enterprise | Bassa (Q2 2027) | Media-Bassa |
| Jurisdiction Expansion Engine (UE/USA) | Qwen3-Max | Mercato 10x | Bassa (12+ mesi) | Media |
| Confidence Spectrum (verde/giallo/rosso) | Qwen3-Max | UX trust +30% | Alta (Sprint 12) | Alta |
| API pubblica per DMS integration | Qwen 3.5 Plus | Ecosystem lock-in | Media (Sprint 15) | Alta |
| Attivare ff_legis_agent come default | DeepSeek | Adozione nuova pipeline | Immediata | Critica |
| Citation resolution da 44.8% a 75%+ | Gemini | Moat expansion | Media (Q1-Q2) | Alta |

---

## 3. ROADMAP CONSOLIDATA E OTTIMIZZATA

### Sprint 9 (immediato, settimane 1-2)

| ID | Azione | Fonte | Effort | Owner |
|---|---|---|---|---|
| S9.1 | Fix max_completion_tokens lexe-fast (P6) | Tutte | 2h | Frisco |
| S9.2 | Attivare ff_legis_agent come default | DeepSeek | 1h | Frisco |
| S9.3 | Retry universale per tool esterni (P7) | MiniMax + Gemini | 4h | Frisco |
| S9.4 | Rate limiting tool executor (P4) | MiniMax + Qwen3-Max | 6h | Frisco |
| S9.5 | Aggiornare Studio prompt obsoleto (P5) | DeepSeek + MiniMax | 2h | Frisco |
| **Totale Sprint 9** | | | **15h** | |

### Sprint 10 (settimane 3-4)

| ID | Azione | Fonte | Effort | Owner |
|---|---|---|---|---|
| S10.1 | Unificare Classifier (H-A, elimina P1) | MiniMax + Gemini | 16h | Frisco |
| S10.2 | JSON validation con fallback robusto | Miglioramento originale | 4h | Frisco |
| S10.3 | Gold set 200 query per validazione routing | Gemini | 8h | Frisco |
| **Totale Sprint 10** | | | **28h** | |

### Sprint 11-12 (settimane 5-8)

| ID | Azione | Fonte | Effort | Owner |
|---|---|---|---|---|
| S11.1 | LEXORC roles nel DB (H-B, elimina P2) | Qwen 3.5 Plus | 8h | Frisco |
| S11.2 | Cleanup alias duplicati (P3) | DeepSeek | 4h | Frisco |
| S11.3 | Confidence Spectrum (badge verde/giallo/rosso) | Qwen3-Max | 12h | Frisco |
| S12.1 | Self-Correction Loop nel Verifier | Gemini | 16h | Frisco |
| S12.2 | SLA monitoring per tool latency | Qwen 3.5 Plus | 8h | Frisco |
| **Totale Sprint 11-12** | | | **48h** | |

### Sprint 13-16 (settimane 9-16, Fase 3 - Differenziazione)

| ID | Azione | Fonte | Effort | Owner |
|---|---|---|---|---|
| S13.1 | Graph-Aware Routing (KB as nervous system) | Qwen3-Max | 24h | Frisco |
| S14.1 | Citation resolution migliorata (44.8% -> 75%) | Gemini | 40h | Frisco |
| S15.1 | Espansione KB a 100+ codici | Qwen 3.5 Plus | 60h | Frisco + team |
| S16.1 | API pubblica per DMS integration | Qwen 3.5 Plus | 40h | Frisco |

---

## 4. METRICHE DI SUCCESSO CONSOLIDATE

Fusione delle metriche migliori proposte da tutti i modelli.

### Performance (fonte: Qwen 3.5 Plus + MiniMax)

| Metrica | Baseline | Post-Sprint 10 | Post-Sprint 12 | Target 12 mesi |
|---|---|---|---|---|
| Latenza media LEGIS STANDARD | 20-30s | < 18s | < 15s | < 12s |
| Latenza media LEGIS DIRECT | 5-8s | < 3s | < 2s | < 1.5s |
| LLM calls per query legale | 3-4 | 2 | 2 | 2 |
| Tool failure rate | ~3% | < 1% | < 0.5% | < 0.2% |
| Parse error rate JSON | ~5% | < 1% | < 0.5% | < 0.1% |

### Qualita (fonte: Gemini + Qwen3-Max)

| Metrica | Baseline | Target Sprint 12 | Target 12 mesi |
|---|---|---|---|
| Verifier pass rate | Da monitorare | > 90% | > 95% |
| Citation resolution rate | 44.8% | 55% | 75% |
| Legal accuracy rate | Da monitorare | > 92% | > 95% |
| Confidenza media risposte | Da monitorare | > 0.80 | > 0.85 |

### Business (fonte: Qwen 3.5 Plus + Project Knowledge)

| Metrica | Target Q1 2026 | Target Q2 2026 | Target 12 mesi |
|---|---|---|---|
| Costo medio per query | < $0.002 | < $0.0015 | < $0.001 |
| Uptime | 99% | 99.5% | 99.9% |
| NPS utenti beta | 40+ | 50+ | 60+ |
| Tenant enterprise | 3 | 10 | 25+ |
| Demo -> Trial conversion | 20% | 25% | 30% |

---

## 5. DECISIONI CHIAVE E RATIONALE

| # | Decisione | Alternativa scartata | Motivazione |
|---|---|---|---|
| D1 | Unificare classifier in singola LLM call | Mantenere doppia classificazione con cache | La cache non risolve il costo, e aggiunge complessita |
| D2 | Semafori differenziati per categoria tool | Semaforo globale unico | Le fonti ufficiali hanno priorita diversa dallo scraping |
| D3 | LEXORC roles via DB (non env vars) | Mantenere env vars con restart | Incompatibile con multi-tenant enterprise |
| D4 | Attivare ff_legis_agent come default | Mantenere opt-in | La pipeline legacy non puo essere la UX primaria |
| D5 | Confidence Spectrum (3 livelli) vs badge binario | Badge verde/rosso | Il giallo cattura le zone grigie fondamentali nel legale |
| D6 | Self-correction max 2 retry | Retry illimitati | Costo e latenza devono restare sotto controllo |

---

## 6. FONTI E RIFERIMENTI

| Codice | Fonte |
|---|---|
| GEM | Gemini 3.1 Pro Preview - analisi architetturale e formula RRF |
| QW35 | Qwen 3.5 Plus - roadmap operativa, pricing, KPI |
| MM25 | MiniMax M2.5 - implementazione dettagliata, AI First Maturity Model |
| QW3M | Qwen3-Max-Thinking - vision strategica, Unified Agent Framework |
| DS32 | DeepSeek V3.2 Speciale - allineamento vision-implementazione, feature flags |
| PKB | Project Knowledge Base LEXe (Ops/Backlog, Sales Kit, Brand Book) |

---

# APPENDICE A: PROMPT STRATEGICO PER PIANIFICAZIONE MULTI-SCENARIO

Il prompt seguente e progettato per essere utilizzato con qualsiasi LLM avanzato per generare piani di implementazione multipli e confrontabili.

---

```
SYSTEM PROMPT: LEXe Implementation Strategy Planner

Sei un consulente tecnico-strategico esperto di architetture AI multi-agente, legal tech e go-to-market per prodotti SaaS B2B verticali. Il tuo compito e generare strategie di implementazione multiple per il progetto LEXe, ciascuna con trade-off espliciti.

CONTESTO DEL PROGETTO:
- LEXe e un assistente legale AI per il mercato italiano
- Architettura: multi-agente (LEGIS + LEXORC), KB proprietaria (69 codici, 38K massime, citation graph 58K edge)
- Stack: Docker, Open WebUI, FastAPI, Redis, PostgreSQL, Qdrant, Neo4j, LiteLLM
- Team: 1 Tech Lead/Product Owner (Frisco), supporto part-time
- Fase: transizione da beta a commercial launch
- Competitor: Lexroom, Lisia, Aptus, Simpliciter
- Pricing target: mid-premium (~85 euro/utente/mese base)

PROBLEMI APERTI DA RISOLVERE (ordinati per criticita):
1. [P1] Double Classifier: 2 LLM call di routing per ogni query legale (+2.5s, costo doppio)
2. [P6] Token limit lexe-fast a 250 (causa troncamento JSON)
3. [P4] Nessun rate limiting nel tool executor
4. [P7] Nessun retry per normattiva/eurlex/infolex
5. [P2] Modelli LEXORC hardcoded in config.py (non configurabili da admin)
6. [P5] Prompt Studio obsoleto (riferimenti a UI rimossa)
7. [P3] Alias duplicati nel catalogo LiteLLM
8. Feature flag ff_legis_agent disabilitato di default

OBIETTIVI STRATEGICI:
- Posizionare LEXe come primo legal assistant AI-First con architettura multi-agente
- Raggiungere latenza < 15s per query STANDARD, < 5s per DIRECT
- Uptime 99.5%+, citation accuracy 95%+
- 10+ tenant enterprise entro Q2 2026
- Conformita GDPR e AI Act by design

VINCOLI:
- Budget limitato (1 persona full-time per sviluppo)
- Sprint bisettimanali
- Non rompere la UX attuale durante la migrazione
- Backward compatibility per tenant esistenti

ISTRUZIONI:
Genera ESATTAMENTE 3 strategie di implementazione, ciascuna con un approccio diverso al bilanciamento rischio/velocita/qualita. Per ogni strategia fornisci:

1. NOME e FILOSOFIA (una frase)
2. TIMELINE (quanti sprint, settimane totali)
3. SEQUENZA OPERAZIONI (cosa viene fatto in ogni sprint)
4. TRADE-OFF ESPLICITI: cosa sacrifica questa strategia rispetto alle altre
5. RISCHI SPECIFICI di questa strategia
6. METRICHE DI GATE per decidere se procedere o pivotare
7. COSTO STIMATO (ore di sviluppo totali)
8. PROBABILITA DI SUCCESSO stimata (%)

Le 3 strategie devono essere:
A) CONSERVATIVE: massima sicurezza, minimo rischio, piu lenta
B) BALANCED: compromesso ottimale tra velocita e qualita
C) AGGRESSIVE: massima velocita, accetta rischi controllati

Per ciascuna strategia, includi anche:
- Piano di rollback se qualcosa va storto
- Criteri di go/no-go per ogni fase
- Impatto sul GTM e sul posizionamento competitivo

FORMATO OUTPUT:
Usa markdown strutturato con tabelle per i confronti. Includi una tabella finale di confronto tra le 3 strategie su 8 dimensioni (tempo, rischio, costo, qualita, UX impact, competitive impact, team stress, reversibilita).

NOTA: Non usare trattini lunghi (em dash). Usa trattini normali (-) o punteggiatura.
```

---

# APPENDICE B: CHECKLIST PRE-IMPLEMENTAZIONE

Prima di iniziare lo Sprint 9, verificare:

- [ ] Backup completo del database PostgreSQL
- [ ] Snapshot dell'infrastruttura Docker corrente
- [ ] Gold set di 200 query (100 legali, 50 studio, 50 edge case) per regression test
- [ ] Baseline metriche attuali (latenza, error rate, costo per query)
- [ ] Feature flag per rollback rapido su ogni modifica
- [ ] Comunicazione ai tenant beta attivi sulle modifiche in arrivo
- [ ] Documentazione ADR (Architecture Decision Record) per ogni decisione D1-D6

---

> Documento compilato da LEXe.OS - Orchestratore operativo
> Prossima revisione: fine Sprint 10
> Changelog: v1.0 - 20/02/2026 - Prima sintesi multi-proposta
