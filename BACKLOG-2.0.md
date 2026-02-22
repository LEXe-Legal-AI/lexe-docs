# LEXE Platform - Backlog 2.0

> Aggiornato: 2026-02-21 — Post Benchmark Sprint

---

## BK-001 — Truncation a 8192 token su risposte complesse (follow-up)

**Severità:** Low (edge case)
**Trovato da:** Mini Benchmark 2026-02-21 (prompt #11, responsabilità medica Gelli-Bianco)
**Componente:** lexe-core / customer_router.py

### Problema

Il prompt #11 (complex tier) produce risposte che superano 8192 completion token sui turn di follow-up (T2 e T3). Il modello risponde con `finish_reason: "length"` — la risposta è troncata.

**Dati benchmark:**

| Turn | Tok In | Tok Out | Finish | Latency |
|------|--------|---------|--------|---------|
| T1   | 2,903  | 706     | stop   | 13s     |
| T2   | 3,092  | 8,192   | length | 90s     |
| T3   | 10,763 | 8,192   | length | 89s     |

Il T1 risponde in 706 token, ma i follow-up ("Come si ripartisce l'onere della prova?" e "Qual è il ruolo delle linee guida?") generano analisi molto dettagliate che superano il budget.

**Nota:** Sugli altri 3 tier (trivial, medium, very_complex) il limite 8192 non viene mai raggiunto. È un edge case su follow-up che richiedono trattazione esaustiva.

### Contesto

- `max_tokens` era 4096, alzato a 8192 dopo benchmark del 2026-02-21
- 4/20 prompt del benchmark completo troncavano a 4096
- Ora solo 2/12 turn del mini benchmark troncano a 8192 (entrambi sullo stesso prompt)

### Fix Proposte

#### Opzione A — Istruzione nel system prompt (Raccomandata)
Aggiungere al persona prompt un vincolo di lunghezza:

```
Mantieni le risposte entro ~4000 parole. Se l'argomento richiede una trattazione
più ampia, struttura la risposta con i punti principali e offri approfondimenti
specifici come follow-up.
```

**Pro:** Zero costi aggiuntivi, nessun cambio codice, risposte più concise e utili
**Contro:** Il modello potrebbe non rispettare sempre il vincolo

#### Opzione B — Alzare max_tokens a 12288 o 16384
Modificare `customer_router.py` linea 1552.

**Pro:** Risolve il troncamento
**Contro:** Costi token più alti (~2x su risposte lunghe), latenze 90s+ già ora

#### Opzione C — max_tokens dinamico per tier
Il sistema potrebbe usare `max_tokens` diversi in base al tipo di domanda (semplice vs complessa), ma richiede classificazione upstream.

**Pro:** Ottimizza costi per domande semplici
**Contro:** Complessità architetturale, richiede intent classification

#### Decisione
**Lasciare 8192** — il troncamento è un edge case su 2/12 turn e le risposte restano comunque complete al 95%. L'utente riceve comunque follow-up per approfondire.

---

## BK-002 — Follow-up dinamici fallback su reasoning model

**Severità:** Fixed
**Trovato da:** Benchmark 2026-02-21
**Componente:** lexe-core / customer_router.py, core.llm_model_catalog

### Problema (Risolto)

Il modello `gpt-5-mini-reasoning` per i follow-up aveva `max_tokens: 250`. I token di reasoning interni consumavano l'intero budget, lasciando `content: ""` e `finish_reason: "length"`. Il frontend cadeva sui `DEFAULT_FOLLOW_UPS` statici.

### Fix Applicata

1. **DB staging:** `max_completion_tokens: 1024` per `gpt-5-mini-reasoning` in `core.llm_model_catalog`
2. **Codice:** Default hardcoded da 250 a 1024 in `customer_router.py:2333`
3. **Timeout:** Follow-up client_timeout da 8s a 12s in `customer_router.py:2305`

### Da Fare

- [ ] Applicare fix DB anche su **production** (schema diverso: `core.tenant_llm_models` invece di `core.llm_model_catalog`)
- [ ] Verificare follow-up dinamici su production dopo deploy

---

## BK-003 — Benchmark system prompt allineato a produzione

**Severità:** Fixed
**Trovato da:** Review benchmark 2026-02-21
**Componente:** benchmarks/benchmark_chat.py, benchmark_mini.py

### Problema (Risolto)

Gli script benchmark usavano un system prompt di 2 righe:
```
"Sei LEXE, un assistente legale italiano esperto e affidabile..."
```
Invece del prompt reale da ~10,000 caratteri con istruzioni di verifica normativa, formati risposta, tool usage, ecc.

### Fix Applicata

Tutti i benchmark ora leggono il prompt da `core.responder_personas` (stessa tabella usata in produzione). Fallback al prompt minimo solo se il DB non è raggiungibile.

---

## BK-004 — Deploy fix su Production

**Severità:** Medium
**Stato:** TODO
**Componente:** lexe-core, lexe-postgres (prod)

### Da fare

- [ ] Push `lexe-core` stage → main e deploy su production (49.12.85.92)
- [ ] Fix DB production per `gpt-5-mini-reasoning` max_completion_tokens
  - **Attenzione:** Production usa `core.tenant_llm_models`, non `core.llm_model_catalog`
  - Verificare schema e adattare la query UPDATE
- [ ] Verificare follow-up dinamici su https://ai.lexe.pro dopo deploy
- [ ] Eseguire `benchmark_mini.py` su production per validare

---

## BK-005 — Latenza elevata su tier very_complex

**Severità:** Low (monitorare)
**Trovato da:** Mini Benchmark 2026-02-21

### Osservazione

Il prompt #16 (very_complex) ha T1 a 108s — quasi 2 minuti per la prima risposta. La media per very_complex è ~68s per turn.

**Dati:**

| Tier | Avg Inference | Avg Tok Out |
|------|---------------|-------------|
| trivial | 18s | 1,109 |
| medium | 24s | 1,354 |
| complex | 64s | 5,697 |
| very_complex | 69s | 4,079 |

### Considerazioni

- gpt-5-2 via OpenRouter introduce overhead di routing
- Il system prompt da 10K char aggiunge ~2,900 token input a ogni richiesta
- Per UX: considerare streaming (già attivo in prod) che mitiga la percezione di attesa
- Non è un bug — è il costo di risposte legali approfondite su temi complessi

---

## BK-006 — KB lacunosa su azioni possessorie e giurisprudenza consolidata

**Severità:** Medium
**Stato:** TODO
**Trovato da:** Test manuale webchat 2026-02-21 (domande su art. 1168 c.c.)
**Componente:** lexe-max (KB), lexe-core (pipeline tool)

### Problema

Testando 3 domande sulle azioni possessorie (spoglio, animus spoliandi, spoglio clandestino/violento), il sistema ha evidenziato:

1. **KB senza massime possessorie:** `kb.massime` non contiene giurisprudenza su art. 1168-1170 c.c. (azioni di reintegrazione e manutenzione). Il NormAgent trova l'articolo ma il DoctrineAgent non trova massime/dottrina.

2. **Giurisprudenza consolidata non recuperata:** Principi noti e pacifici non vengono forniti:
   - Spoglio continuato: il termine annuale decorre dall'**ultimo atto**, non dal primo
   - Animus spoliandi **in re ipsa**: si desume dal fatto oggettivo, non serve dolo specifico
   - Spoglio clandestino: dies a quo dalla **scoperta**, non dall'atto (Cass. SS.UU. 1984/1395)

3. **web_search non attivato come fallback:** Quando la KB non copre un tema, il sistema dovrebbe usare `web_search` per colmare la lacuna. Non lo fa — risponde con "non sono emerse evidenze" senza tentare fonti esterne.

4. **Confidence scoring incoerente:** La seconda risposta (animus spoliandi) ha confidence 70% pur ammettendo di non avere contenuto. Dovrebbe essere ≤40%.

### Dati dal test

| Domanda | Confidence | Normativa | Giurisprudenza | web_search |
|---------|-----------|-----------|----------------|------------|
| Dies a quo spoglio (primo/ultimo) | 55% | art. 1168 OK | Non trovata | Non usato |
| Prova animus spoliandi | 70% | art. 1168 OK | Non trovata | Non usato |
| Animus in spoglio clandestino/violento | 30% | art. 1168 OK | Non trovata | Non usato |

### Fix Applicate (2026-02-22)

#### A — KB ha già 1222 massime possessorie (Non serve import!)
**Scoperta:** La KB contiene già massime rilevanti:
- 52 massime con "spoglio"
- 253 con "reintegrazione"
- 914 con "possesso"
- 3 con "art. 1168" esplicito
- art. 1168, 1169, 1170 presenti in `kb.normativa`

**Root cause reale:** `kb_search.py` era completamente non funzionale a causa di 4 bug:

1. **`model_name` mismatch** (CRITICO): SQL filtrava `e.model_name = 'text-embedding-3-small'`
   ma il DB contiene `'openai/text-embedding-3-small'` → dense search restituiva 0 risultati per TUTTE le query
2. **Embedding serialization** (CRITICO): Passava `list[float]` Python ad asyncpg che richiede
   stringa `"[0.1,0.2,...]"` → hybrid search crashava con 500 error
3. **`plainto_tsquery` AND logic** (ALTO): Query multi-termine richiedeva TUTTI i termini presenti →
   "spoglio art 1168" = 0 risultati perché nessuna massima contiene tutti i termini
4. **`litellm_embedding_model` errato** (CRITICO): Config aveva `"lexe-embedding"` (alias inesistente) →
   embedding API restituiva 400, nessun embedding generato → solo sparse search (broken anch'esso per bug #3)

**Effetto combinato:** 38,718 massime esistevano ma non venivano MAI restituite dalla search.

- [x] ~~Import massime~~ → Non necessario, già presenti
- [x] **RISOLTO:** 4 bug critici fixati in `kb_search.py` e `config.py` (commits `0b9a75d`, `2109b2f`)
- [x] Testato su staging: 4/4 query restituiscono 10 risultati in modalità **hybrid**
- [x] Dense scores 0.60-0.71 (semantic similarity funzionante)

#### B — Fallback web_search su KB miss (Applicata)
Implementato in `advanced_orchestrator.py`: dopo la fase di ricerca parallela,
se `case_law` e `massime` sono vuoti E `internet_opt_in=True`, viene eseguito
automaticamente `web_search` come fallback con query giurisprudenziale.

- [x] Logica fallback in orchestrator (Phase 3b)
- [x] Risultati integrati come `CaseLawRef` con source="web_search"
- [x] Log quando `internet_opt_in=False` per tracciabilità
- [ ] **TODO:** Audit fonti web (confidence bassa per web_search)

#### C — Confidence scoring calibration (Applicata)
Implementato in `scoring.py`: se non ci sono `case_law` né `massime` ma ci sono `norms`,
il confidence è cappato a 40%.

- [x] Cap 40% quando giurisprudenza assente in `compute_confidence()`
- [x] Nota esplicita nel breakdown: "cap 40%: giurisprudenza assente"

#### D — Gap analysis KB sistematica (Completata)
**Risultati:** 38,718 massime attive totali. La copertura è ampia ma il retrieval
è il collo di bottiglia, non la quantità di dati.

- [x] Query distribuzione massime
- [x] Search quality investigata → Root cause trovata (vedi Fix A)

#### E — NBA "Abilita ricerca web" (Nuova, Applicata)
Quando la giurisprudenza manca, il sistema ora suggerisce "Abilita ricerca web per giurisprudenza"
come Next Best Action nel pannello LEXORC.

- [x] Aggiunto in `generate_nba()` in `lexorc_synthesizer.py`

---

## BK-007 — kb_search completamente non funzionale (4 bug critici)

**Severità:** Critical (RISOLTO)
**Trovato da:** Investigazione BK-006 Fix A, 2026-02-22
**Componente:** lexe-tools-it / kb_search.py, config.py

### Problema (Risolto)

`kb_search` (hybrid massime search) era completamente non funzionale su staging e production.
38,718 massime esistevano nel DB ma non venivano mai restituite. 4 bug separati:

| Bug | File | Descrizione | Effetto |
|-----|------|-------------|---------|
| model_name mismatch | kb_search.py:274 | `'text-embedding-3-small'` vs DB `'openai/text-embedding-3-small'` | Dense = 0 risultati |
| Embedding serialization | kb_search.py:325 | `list[float]` non serializzabile da asyncpg | 500 error |
| plainto_tsquery AND | kb_search.py:288,292 | Tutti i termini richiesti | Sparse = 0 per query complesse |
| Config model name | config.py:71 | `"lexe-embedding"` (inesistente in LiteLLM) | Embedding API 400 |

### Fix Applicate

1. `e.model_name = 'openai/text-embedding-3-small'` (allineato al DB)
2. `embedding_str = "[" + ",".join(str(x) for x in embedding) + "]"` (serializzazione corretta)
3. `websearch_to_tsquery` al posto di `plainto_tsquery` (OR-style matching)
4. `litellm_embedding_model = "text-embedding-3-small"` (nome corretto)

### Verifica

Testato su staging 2026-02-22:

| Query | Mode | Risultati | Latency |
|-------|------|-----------|---------|
| azione possessoria spoglio art 1168 | hybrid | 10 | 0.88s |
| animus spoliandi prova in re ipsa | hybrid | 10 | 0.48s |
| responsabilità art 2043 (control) | hybrid | 10 | 0.83s |
| spoglio (simple) | hybrid | 10 | 0.49s |

Dense scores 0.60-0.71. RRF fusion funzionante con contributi da entrambi dense e sparse.

### Da Fare

- [ ] Applicare stesse fix su production (`lexe-tools-it` config.py e kb_search.py)
- [ ] Verificare che `normativa_kb_search.py` config.py model name sia allineato

---

*Ultimo aggiornamento: 2026-02-22*
