# LEXe — Legal AI Assistant

> *La legge a portata di AI*

---

## Cos'e LEXe

LEXe e un assistente legale AI progettato per studi legali e uffici legali d'impresa. Non e un motore di ricerca con interfaccia chatbot: e un **agente autonomo** che comprende la domanda giuridica, decide cosa cercare, dove cercarlo, verifica quello che trova, e produce un parere strutturato con fonti verificate e link cliccabili.

L'utente scrive una domanda in linguaggio naturale. LEXe fa tutto il resto.

---

## Il principio: zero friction, massima autonomia

L'interazione ideale con LEXe e una conversazione. Niente menu, niente dropdown, niente wizard a 5 step. L'avvocato scrive come scriverebbe a un collega esperto e LEXe risponde con lo stesso livello di profondita che ci si aspetterebbe da quel collega — ma con le fonti.

**L'interfaccia ha un campo di testo e poco altro.** Il resto lo decide l'agente.

### Cosa decide LEXe da solo

| Decisione                          | Come la prende                                                                 | Cosa vede l'utente                                                 |
| ---------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| **Complessita della domanda**      | Intent detector classifica in 4 livelli (DIRECT, SIMPLE, STANDARD, COMPLEX)    | Niente — e trasparente                                             |
| **Scenario giuridico**             | Scenario detector: ricerca, parere, contratto, contenzioso, recupero crediti   | Risposta gia nel formato giusto per lo scenario                    |
| **Pipeline da attivare**           | Routing automatico basato su complessita, scenario e piano                     | Un indicatore di fase (Pianificazione, Ricerca, Verifica, Sintesi) |
| **Quali tool usare**               | Il planner decide quali fonti interrogare e in che ordine                      | Badge con le fonti consultate                                      |
| **Quante fonti cercare**           | Budget tool calibrato sulla complessita                                        | Contatore fonti trovate                                            |
| **Se i risultati sono affidabili** | Verifier anti-allucinazione (strutturale + LLM)                                | Badge di confidenza (verde/giallo/rosso)                           |
| **Come strutturare la risposta**   | Synthesizer con template scenario-specifico (parere, contratto, ricerca, etc.) | Risposta strutturata con sezioni A-F                               |
| **Cosa approfondire dopo**         | Follow-up generator propone 3 domande contestuali                              | Chip cliccabili sotto la risposta                                  |

### Cosa NON chiede all'utente

- Non chiede "vuoi cercare su Normattiva o EUR-Lex?" — cerca su entrambi se serve
- Non chiede "quante massime vuoi?" — ne cerca quante ne servono
- Non chiede "che tipo di risposta vuoi?" — lo capisce dalla domanda
- Non chiede "vuoi verificare le fonti?" — verifica sempre, automaticamente
- Non chiede "in che formato vuoi la risposta?" — usa il formato giusto per lo scenario

### L'unica scelta esplicita

L'utente sceglie la **modalita conversazione** una sola volta, all'inizio:

- **Studio** — Assistente generico, memoria attiva, ricerca leggera. Per domande rapide, brainstorming, drafting.
- **Legal** — Agente di ricerca rigoroso, zero memoria cross-session, verifica fonti, audit trail immutabile. Per pareri, analisi, contenzioso.

Anche questa scelta potrebbe essere automatizzata in futuro (l'agente capisce dal tono e dal contenuto se serve rigore documentale o conversazione fluida). Ma per ora e l'unico "pulsante" che conta.

---

## Come funziona sotto il cofano

### Una domanda, quattro fasi

```
"Responsabilita del medico per errore diagnostico: norme e giurisprudenza recente"
```

**Fase 0 — Intent Detection** (< 1s)
L'intent detector legge la domanda e decide: livello STANDARD, scenario PARERE, dominio diritto civile + sanitario, giurisdizione IT. Zero interazione utente.

**Fase 1 — Pianificazione** (2-3s)
Il planner produce un piano di ricerca strutturato. Per questo caso: 6 query su 4 fonti diverse. Normattiva per gli articoli chiave (art. 2043 c.c., art. 2236 c.c., L. 24/2017). KB Normativa per il testo vigente con embeddings. KB Massimario per massime di Cassazione pertinenti. InfoLex per annotazioni dottrinali. L'utente vede solo "Pianificazione in corso...".

**Fase 2 — Ricerca + Verifica** (5-15s)
Le query partono in parallelo su tutte le fonti. I risultati vengono aggregati in un evidence pack. Il verifier controlla: articoli esistenti? Vigenza corretta? Citazioni coerenti? Contraddizioni interne? L'utente vede i badge delle fonti che si riempiono uno a uno.

**Fase 3 — Sintesi** (5-10s)
Il synthesizer produce il parere strutturato in streaming, con template specifico per lo scenario (parere vs. contratto vs. ricerca). Ogni affermazione ha un tag [N] che rimanda all'evidence pack. Le norme hanno link cliccabili a Normattiva. Le massime hanno numero RV, sezione, anno. L'utente legge in tempo reale.

**Totale: 15-30 secondi** per un parere con 6-10 fonti verificate, link cliccabili, badge di confidenza. Senza aver cliccato nulla.

### Per domande semplici, ancora piu veloce

```
"Cosa dice l'art. 2043 del codice civile?"
```

LEXe riconosce che e un lookup diretto (livello DIRECT). Nessun planner, nessun verifier complesso. Il testo viene recuperato istantaneamente dalla Knowledge Base interna (embeddings + full-text), con link a Normattiva per il testo ufficiale vigente. Risposta in 3-5 secondi.

### Per domande complesse, scala automaticamente

```
"Analisi completa della responsabilita solidale tra committente e appaltatore
in caso di infortunio, con evoluzione giurisprudenziale e profili processuali"
```

LEXe riconosce che serve il pipeline avanzato (livello COMPLEX). Attiva agenti paralleli specializzati: NormAgent cerca gli articoli rilevanti su tutti i codici, CaseLawAgent interroga il massimario con ricerca semantica e citation graph, DoctrineAgent raccoglie dottrina da 13 fonti. L'Auditor verifica coerenza tra le fonti. Il Synthesizer produce un parere completo con timeline giurisprudenziale, profili critici, e next best actions. 30-60 secondi, nessun click.

### Sei scenari specializzati

LEXe non produce sempre lo stesso tipo di risposta. Lo scenario detector riconosce automaticamente cosa serve:

| Scenario                | Output                                        | Esempio                                              |
| ----------------------- | --------------------------------------------- | ---------------------------------------------------- |
| **Ricerca**             | Analisi normativa + giurisprudenza con fonti  | "Norme su appalti e subappalti dopo il nuovo codice" |
| **Parere**              | Parere strutturato A-F con conclusioni e NBA  | "Responsabilita del medico: analisi completa"        |
| **Redazione contratto** | Bozza contrattuale con clausole e riferimenti | "Bozza contratto di appalto per ristrutturazione"    |
| **Revisione contratto** | Analisi clausole, rischi, suggerimenti        | "Revisiona questo contratto di locazione"            |
| **Contenzioso**         | Strategia processuale con giurisprudenza      | "Il cliente ha subito un danno: come agire?"         |
| **Recupero crediti**    | Piano di recupero con strumenti giudiziali    | "Fattura non pagata da 12 mesi, come procedere?"     |

---

## Il motore della conoscenza: lexe-max

Sotto ogni risposta di LEXe c'e **lexe-max**, il motore di conoscenza giuridica proprietario. Non e un wrapper su API esterne: e un database vettoriale PostgreSQL dedicato, con dati estratti, classificati, chunkizzati e indicizzati internamente.

### Knowledge Base Normativa

**69 codici e testi unici** della legislazione italiana, estratti da PDF ufficiali con pipeline multi-fonte a 4 livelli di validazione:

1. **Estrazione primaria** — PDF Altalex con OCR e LLM-assisted extraction
2. **Validazione offline** — Cross-check con Studio Cataldi (24 codici) e Brocardi (4 codici)
3. **Classificazione 3 assi** — Ogni articolo classificato per identita (BASE/SUFFIX/SPECIAL), qualita (VALID_STRONG/SHORT/WEAK/EMPTY), e ciclo di vita (CURRENT/HISTORICAL/REPEALED)
4. **Fallback online** — Brocardi Online e Normattiva per articoli deboli o in watchlist

| Metrica                                    | Valore                                                             |
| ------------------------------------------ | ------------------------------------------------------------------ |
| Codici/leggi configurati                   | 69                                                                 |
| Articoli totali                            | 13.000+ (target: 15.000+)                                          |
| Chunks per retrieval                       | 10.000+                                                            |
| Embeddings (text-embedding-3-small, 1536d) | 10.000+                                                            |
| Full-text search index                     | Italian tsvector su ogni chunk                                     |
| Copertura                                  | CC, CP, CPC, CPP, COST + 64 tra TU, leggi speciali, regolamenti UE |

### Knowledge Base Massimario

Il **massimario della Corte di Cassazione** completo, con ricerca semantica e grafi di citazione:

| Metrica                                    | Valore    |
| ------------------------------------------ | --------- |
| Massime attive                             | 38.718    |
| Embeddings (text-embedding-3-small, 1536d) | 41.437    |
| Arco temporale                             | 1913-2025 |
| Recall@10                                  | 97.5%     |
| MRR (Mean Reciprocal Rank)                 | 0.756     |

### Citation Graph

Un **grafo di citazione** tra massime che permette di navigare la giurisprudenza come una rete di connessioni:

| Metrica                 | Valore |
| ----------------------- | ------ |
| Edge totali             | 58.737 |
| Massime sorgente uniche | 26.415 |
| Massime target uniche   | 20.868 |
| Tasso di risoluzione    | 44.8%  |

Il resolver a 5 livelli identifica le citazioni incrociando RV, sezione, numero e anno con cinque strategie progressive (rv_exact, sez_num_anno, rv_text_fallback, sez_num_anno_raw, num_anno).

### Norm Graph

Un **grafo delle norme** che mappa quali articoli di legge sono citati in quali massime:

| Metrica           | Valore                                                |
| ----------------- | ----------------------------------------------------- |
| Norme uniche      | 4.128                                                 |
| Edge totali       | 42.338                                                |
| Massime con norme | 23.365 (60.3%)                                        |
| Copertura         | CC, CP, CPC, CPP, COST, TUB, TUF, CAD, leggi speciali |

Quando l'utente chiede "giurisprudenza sull'art. 2043 c.c.", LEXe non fa solo ricerca semantica: interroga direttamente il norm graph per trovare tutte le massime che citano quell'articolo. Risultato: precisione massima, zero rumore.

### Hybrid Search

Ogni ricerca sulla Knowledge Base usa un sistema di **retrieval ibrido** a tre canali:

```
Query utente
    |
    +---> Dense search (vector cosine, HNSW, top-50)
    |         Cattura significato semantico
    |
    +---> Sparse search (BM25 Italian tsvector, top-50)
    |         Cattura termini esatti e codici
    |
    +---> Norm boost (se la query cita un articolo specifico)
              Bonus diretto dal norm graph
    |
    v
  RRF Fusion (Reciprocal Rank Fusion)
    |
    v
  Top-K risultati ordinati per rilevanza combinata
```

Questo approccio supera sia la ricerca puramente semantica (che perde termini tecnici esatti) sia la ricerca puramente lessicale (che non capisce sinonimi e parafrasi). La combinazione RRF garantisce il meglio di entrambi i mondi.

### Tecnologia sotto il cofano

| Componente   | Tecnologia                            | Dettaglio                                                             |
| ------------ | ------------------------------------- | --------------------------------------------------------------------- |
| Database     | PostgreSQL 17 + pgvector 0.7.4        | Vettori e relazioni nello stesso DB                                   |
| Vector index | HNSW (m=16, ef=64)                    | Ricerca sub-millisecondo su 40K+ vettori                              |
| Embeddings   | text-embedding-3-small (1536d)        | OpenAI via LiteLLM gateway                                            |
| Full-text    | Italian tsvector + unaccent           | BM25 con stemming italiano                                            |
| Graph engine | Relazioni PostgreSQL                  | Citation graph + norm graph                                           |
| Chunking     | Smart split (1000 chars, 150 overlap) | Boundary-aware (frase/virgola/spazio)                                 |
| LLM Gateway  | LiteLLM                               | Multi-provider (OpenAI, Anthropic, Google) con virtual key per tenant |

---

## Fonti e verificabilita

LEXe non e un chatbot che "sa cose". E un agente che **cerca, trova, verifica e cita**.

### Fonti interne (lexe-max)

| Fonte              | Contenuto                         | Copertura                       | Ricerca                      |
| ------------------ | --------------------------------- | ------------------------------- | ---------------------------- |
| **KB Normativa**   | Testo vigente di 69 codici/leggi  | 13.000+ articoli con embeddings | Hybrid: dense + sparse + RRF |
| **KB Massimario**  | 38.718 massime Cassazione         | 1913-2025, citation graph       | Semantic + norm graph + RRF  |
| **KB Annotations** | 13.281 note dottrinali (Brocardi) | CC, CP (copertura completa)     | Embedding search             |

### Fonti esterne (live)

| Fonte                  | Contenuto                            | Copertura                      | Metodo                     |
| ---------------------- | ------------------------------------ | ------------------------------ | -------------------------- |
| **Normattiva.it**      | Testo vigente ufficiale              | Tutti i codici, leggi, decreti | Scraping + parsing URN:NIR |
| **EUR-Lex**            | Regolamenti, direttive, decisioni UE | Normativa europea completa     | API + scraping             |
| **InfoLex (Brocardi)** | Note, commenti, massime online       | CC, CP, CPC, CPP               | Scraping con rate limit    |
| **13 siti giuridici**  | Altalex, DeJure, Diritto.it, etc.    | Dottrina, commenti, guide      | Web search contestuale     |

### Come si combinano

Quando LEXe riceve una domanda, il planner decide **quali fonti** interrogare e **in che ordine**. Per una domanda sull'art. 2043 c.c.:

1. **KB Normativa** — Testo dell'articolo istantaneo (gia in memoria, 0ms)
2. **KB Massimario** — Massime di Cassazione via norm graph (CC:2043 → 150+ massime) + ricerca semantica
3. **Normattiva.it** — Link ufficiale al testo vigente (per verifica e citazione)
4. **InfoLex** — Annotazioni dottrinali Brocardi sull'art. 2043

La KB interna serve per velocita e recall. Le fonti esterne servono per verificabilita e link cliccabili. Il risultato: risposte rapide **e** verificabili.

### Verificabilita

Ogni affermazione nella risposta ha un riferimento `[N]` all'evidence pack. Ogni norma ha un link cliccabile a Normattiva o EUR-Lex. Ogni massima ha numero RV, sezione, anno. Il badge di confidenza indica quanto l'agente si fida dei propri risultati.

**Se non trova fonti, lo dice.** Non inventa. Il verifier anti-allucinazione gira su ogni risposta.

---

## Privacy e isolamento

- **Modalita Legal**: zero memoria cross-session. Ogni conversazione e un mondo chiuso. Nessun dato personale dell'utente viene estratto o memorizzato. Audit trail immutabile.
- **Modalita Studio**: memoria attiva (preferenze, contesto professionale). Opt-in, trasparente, cancellabile.
- **Multi-tenant**: isolamento completo per tenant. RLS PostgreSQL, virtual key LiteLLM per tenant, nessun data leak possibile tra organizzazioni.
- **GDPR**: retention 90 giorni su evidence pack, right to erasure, no training su dati utente.
- **LLM isolation**: ogni tenant ha la propria virtual key LiteLLM con budget e rate limit indipendenti. Nessun dato transita tra tenant.

---

## La visione: l'avvocato scrive, LEXe lavora

L'obiettivo finale e un assistente che non ha bisogno di essere guidato. L'avvocato non deve sapere che esistono tool, pipeline, modelli, livelli di complessita. Non deve scegliere tra "ricerca base" e "ricerca avanzata". Non deve premere "verifica fonti".

**Scrive la domanda. Legge la risposta. Clicca i link per approfondire. Fine.**

Tutto il resto — la scelta del modello giusto, la calibrazione del budget, la parallelizzazione delle ricerche, la verifica anti-allucinazione, il formato della risposta — e responsabilita dell'agente.

Il vantaggio competitivo non e solo l'AI: e la **Knowledge Base proprietaria**. 69 codici estratti, classificati e indicizzati. 38.718 massime con citation graph. 58.737 connessioni tra pronunce. 42.338 link norma-massima. Tutto interrogabile in millisecondi con hybrid search. Questo e il moat che separa LEXe da qualsiasi wrapper su ChatGPT.

L'unica interfaccia che conta e quella piu antica: una domanda e una risposta.
