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

### Fix Proposte

#### A — Arricchimento KB massime possessorie (P1)
Importare massime di Cassazione su art. 1168, 1169, 1170 c.c. in `kb.massime`. Fonte: Brocardi, DeJure, o scraping mirato.

- [ ] Identificare fonte massime possessorie (Brocardi ha sezione dedicata)
- [ ] Import in `kb.massime` con embeddings
- [ ] Verificare retrieval su domande test

#### B — Fallback web_search su KB miss (P1)
Quando NormAgent/DoctrineAgent non trovano risultati sufficienti (confidence <50%), il Synthesizer dovrebbe attivare `web_search` come fallback automatico.

- [ ] Aggiungere logica in LEXORC pipeline: se evidenze < soglia → trigger web_search
- [ ] Integrare risultati web nel parere con disclaimer fonte
- [ ] Aggiornare auditor per verificare anche fonti web

#### C — Confidence scoring calibration (P2)
Il confidence non deve essere >50% se il sistema non ha contenuto giurisprudenziale da citare. Aggiungere regola:

```
Se giurisprudenza_trovata == 0 AND domanda richiede_giurisprudenza:
    confidence = min(confidence, 40%)
```

- [ ] Aggiungere post-processing confidence nel Synthesizer
- [ ] Testare su batch di domande giurisprudenziali

#### D — Gap analysis KB sistematica (P2)
Mappare quali aree del diritto civile mancano nella KB per pianificare import mirati.

- [ ] Query su `kb.massime` per distribuzione articoli coperti
- [ ] Confronto con indice codice civile per identificare gap
- [ ] Prioritizzare: possessorio, obbligazioni, successioni, famiglia

---

*Ultimo aggiornamento: 2026-02-21*
