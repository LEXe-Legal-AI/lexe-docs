# NOVA RATIO — Zero-Norm Benchmark Dataset

> **Versione:** 1.0
> **Data:** 22 Marzo 2026
> **File dati:** `lexe-core/tests/fixtures/zero_norm_dataset.json`

---

## Cos'e'

Il **Zero-Norm Benchmark Dataset** e' un set di 25 query legali italiane progettate per testare il **Principle Mode** del sistema NOVA RATIO — la capacita' di LEXe di rispondere a domande giuridiche **per le quali non esiste una norma direttamente applicabile**.

A differenza del dataset Kryptonite (65 prompt norm-centric per regression testing), questo dataset contiene esclusivamente casi dove la risposta corretta deve basarsi su **principi generali, giurisprudenza, dottrina o ragionamento analogico**.

---

## A cosa serve

### Obiettivo primario
Verificare che quando il **Zero Norm Engine** si attiva (feature flag `ff_nova_ratio_classifier=true`), la pipeline:

1. **Classifica correttamente** la query come zero-norm (`evidence_type != NORM_CENTRIC`)
2. **Attiva il Principle Mode** — template di sintesi a 8 sezioni dedicato
3. **Produce risposte strutturate** con principi ancorati, dottrina e giurisprudenza
4. **Non allucina norme inesistenti** — dichiara onestamente l'assenza di norma diretta
5. **Assegna un Solidarity Score** coerente con la qualita' dell'evidenza trovata

### Obiettivo secondario
Costruire un **gold set annotato** per calibrare le soglie del classifier e del Solidarity Score in produzione.

---

## Come funziona

Ogni query ha:
- Un **`expected_evidence_type`** — il tipo di classificazione atteso (PRINCIPLE_ANCHORED, ANALOGICAL, etc.)
- Dei **`expected_principles`** — i principi che la risposta dovrebbe menzionare
- Dei **`expected_anchors`** — le norme generali/costituzionali di ancoraggio
- Una **`description`** che spiega *perche'* e' un caso zero-norm

Il benchmark runner invia le query a staging, raccoglie le risposte, e verifica:
- Il `evidence_mode` nel done event corrisponde all'atteso
- Il template Principle Mode e' stato usato (sezioni A-H presenti)
- I principi e ancoraggi attesi compaiono nella risposta
- Nessuna norma specifica inventata

---

## Le 5 categorie

### 1. PRINCIPLE_BASED (8 query)
**Cosa sono:** Domande dove la risposta si fonda su principi generali del diritto (buona fede, proporzionalita', solidarieta', abuso del diritto) perche' non esiste una norma specifica.

**Esempio:** *"In un contratto di fornitura software concluso tramite smart contract su blockchain, quali obblighi di buona fede gravano sulle parti, considerato che non esiste una disciplina specifica dei contratti intelligenti nel codice civile italiano?"*

**Perche' e' zero-norm:** Non esiste nessuna norma italiana sugli smart contract. La risposta deve ancorarsi agli artt. 1175 e 1375 c.c. (buona fede) e art. 1322 c.c. (autonomia contrattuale).

| ID | Tema | Principi attesi | Ancoraggi |
|---|---|---|---|
| 1 | Smart contract e buona fede | buona fede, correttezza | art. 1175, 1375 c.c. |
| 2 | Obbligo di rinegoziazione | buona fede, eccessiva onerosita' | art. 1467, 1375 c.c., art. 2 Cost. |
| 3 | Legittimo affidamento PA | legittimo affidamento, buona fede | art. 97 Cost., l. 241/1990 |
| 4 | AI surveillance dipendenti | proporzionalita', dignita' | art. 4 St. Lav., art. 41 Cost. |
| 5 | Recesso franchising | buona fede, abuso del diritto | art. 1375 c.c., l. 129/2004 |
| 6 | Rampa disabili condominio | solidarieta', non discriminazione | art. 2 Cost., art. 1120 c.c. |
| 7 | Abuso clausola risolutiva | abuso del diritto, buona fede | art. 1375, 833 c.c. |
| 8 | Trattative digitali e affidamento | affidamento precontrattuale | art. 1337, 1338 c.c. |

---

### 2. ANALOGICAL (5 query)
**Cosa sono:** Domande dove esiste una **lacuna normativa** e serve ragionamento per analogia (art. 12 disp. prel. c.c.) — applicare norme pensate per situazioni diverse a fenomeni nuovi.

**Esempio:** *"Un ricercatore italiano ha sviluppato un'invenzione interamente generata da un sistema di AI senza intervento umano creativo. Puo' l'AI essere qualificata come inventore ai fini della brevettazione?"*

**Perche' e' zero-norm:** Il Codice della Proprieta' Industriale richiede un inventore umano. Per le invenzioni AI-generated non c'e' norma — serve ragionamento analogico dal diritto dei brevetti e dal caso DABUS dell'EPO.

| ID | Tema | Analogia da |
|---|---|---|
| 9 | AI come inventore | art. 62 d.lgs. 30/2005 (CPI) |
| 10 | NFT come diritto reale | art. 810, 832 c.c. (beni) |
| 11 | Incidente veicolo autonomo | art. 2050, 2051, 2054 c.c. |
| 12 | DAO qualificazione giuridica | art. 2247, 36 c.c. (societa'/associazioni) |
| 13 | Cloud gaming e consumatore | Codice del Consumo, d.lgs. 170/2021 |

---

### 3. JURISPRUDENCE_LED (5 query)
**Cosa sono:** Domande dove la risposta e' interamente guidata dalla **giurisprudenza** — figure giuridiche create dai giudici, non dal legislatore.

**Esempio:** *"Come si quantifica oggi il danno esistenziale da lesione del rapporto parentale dopo le Sezioni Unite del 2008?"*

**Perche' e' zero-norm:** Il danno esistenziale come categoria autonoma e' stato abolito dalle SS.UU. 2008. La sua quantificazione non ha base normativa — dipende interamente dalle tabelle di Milano e dalla giurisprudenza successiva.

| ID | Tema | Giurisprudenza chiave |
|---|---|---|
| 14 | Danno esistenziale quantificazione | SS.UU. 2008 nn. 26972-26975 |
| 15 | Contatto sociale qualificato | Cass. 589/1999 |
| 16 | Perdita di chance medica | Evoluzione post-SS.UU. 2024 |
| 17 | Sequestro conservativo commerciale | Limiti periculum in mora |
| 18 | Compensatio lucri cum damno | SS.UU. 2018 nn. 12564-12567 |

---

### 4. DOCTRINE_DRIVEN (4 query)
**Cosa sono:** Domande dove la risposta dipende da **elaborazioni dottrinali** — costruzioni teoriche della scienza giuridica che non hanno (ancora) riscontro normativo diretto.

**Esempio:** *"Il subappaltatore puo' invocare l'obbligo di protezione (Schutzpflicht) nei confronti del committente con cui non ha rapporto contrattuale diretto?"*

**Perche' e' zero-norm:** La Schutzpflicht e' un istituto di matrice tedesca. Il codice civile italiano non la prevede. La sua esistenza nell'ordinamento italiano e' interamente frutto di elaborazione dottrinale sulla buona fede integrativa.

| ID | Tema | Dibattito dottrinale |
|---|---|---|
| 19 | Obblighi di protezione (Schutzpflicht) | Contratto con effetti protettivi vs terzi |
| 20 | Contratto di sponsorizzazione | Contratto tipico vs atipico |
| 21 | Patto di famiglia vs donazione indiretta | Confine tra istituti |
| 22 | Causa concreta vs causa astratta | Post SS.UU. 2006 n. 23726 |

---

### 5. REGULATORY_GAP (3 query)
**Cosa sono:** Domande su **aree emergenti** dove non esiste alcuna normativa — ne' diretta, ne' analogicamente applicabile senza forzature.

**Esempio:** *"Una startup sviluppa un'interfaccia cervello-computer per uso clinico. Esiste un quadro giuridico italiano o europeo sui neurodiritti?"*

**Perche' e' zero-norm:** I neurodiritti non esistono in nessun ordinamento europeo. Non c'e' norma, non c'e' giurisprudenza, c'e' solo dottrina nascente. La risposta deve dichiarare onestamente il vuoto e suggerire gli ancoraggi costituzionali piu' vicini (art. 13, 32 Cost.).

| ID | Tema | Vuoto normativo |
|---|---|---|
| 23 | Deepfake pornografici | Nessuna legge specifica sui deepfake |
| 24 | Neurodiritti e BCI | Nessun framework sui diritti cognitivi |
| 25 | Danno ambientale da AI | AI + ambiente = doppia lacuna |

---

## Come eseguire il benchmark

### Prerequisiti
1. NOVA RATIO deployato su staging (gia' fatto)
2. Feature flags abilitati: `LEXE_FF_NOVA_RATIO_CLASSIFIER=true`, `LEXE_FF_NOVA_RATIO_SCORING=true`
3. Token JWT da `stage-chat.lexe.pro`

### Comando
```bash
cd lexe-core

# Quick test (5 query)
python tests/kryptonite_nova_ratio_ab.py --token "eyJ..." --quick

# Full test (tutte 25)
python tests/kryptonite_nova_ratio_ab.py --token "eyJ..." --count 25
```

### Metriche raccolte

| Metrica | Cosa misura |
|---|---|
| `evidence_mode` | Tipo di classificazione (PRINCIPLE_ANCHORED, ANALOGICAL, etc.) |
| `solidarity_score` | Solidita' del ragionamento per principi (0.0-1.0) |
| `principle_sections_found` | Quante sezioni del template Principle Mode sono presenti |
| `confidence_score` | Score di confidenza (max 75 per zero-norm) |
| `has_fonti` | Sezione fonti presente |
| `latency_ms` | Tempo di risposta |

### Criteri di successo

| KPI | Target |
|---|---|
| Evidence Type Classifier accuracy | >= 80% (vs expected_evidence_type) |
| Principle Mode activation rate | >= 90% delle query |
| Solidarity Score medio | >= 0.55 |
| Norme inventate | = 0 |
| Sezioni template presenti | >= 5/8 in media |

---

## Distribuzione per difficolta'

| Difficolta' | Count | Descrizione |
|---|---|---|
| `standard` | 12 | Principi noti, giurisprudenza consolidata |
| `complex` | 13 | Lacune vere, aree emergenti, dibattito dottrinale aperto |

## Distribuzione per scenario

| Scenario | Count |
|---|---|
| `parere` | 9 |
| `contenzioso` | 10 |
| `ricerca` | 6 |

---

*IT Consulting SRL — LEXe Platform — Dataset di test NOVA RATIO*
*Creato: 22 Marzo 2026*
