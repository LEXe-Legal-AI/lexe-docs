# LEXe Hybrid Score — Piano Esecutivo

> **Obiettivo:** Un singolo score 0-100 che sia **giusto** — non iper-penalizza, non iper-premia.
> Risposte buone (la maggioranza) → 80+. Gap reali → 60-79. Rotto → < 50.

| Campo | Valore |
|---|---|
| Data | 22 Marzo 2026 |
| Target | Production first (vecchio orchestrator), poi staging |
| Prerequisito | Nessuna migrazione DB necessaria |

---

## 1. IL PROBLEMA

Abbiamo 3 score che non concordano:

| Score | Media reale | Problema |
|---|---|---|
| Pipeline Confidence | 97 (prod) | Inflazionato: misura fonti, non ragionamento. Floor 40, quasi tutto 100 |
| LLM Judge | 48 (staging) | Deflazionato: penalita' x0.4 per singola hallucination. 50 = "sufficiente" |
| Solidarity Score | max 75 | Penalizza zero-norm: capped a 75, come se fosse meno legale |

**Nessuno e' sbagliato**, ma nessuno racconta la storia giusta da solo.

---

## 2. HYBRID SCORE: FORMULA

```
HybridScore = (SourceQuality * 0.40) + (ReasoningQuality * 0.35) + (Completeness * 0.25)
```

### 2.1 SourceQuality (0-100) — dal confidence attuale, ricalibrato

```python
def source_quality(evidence_pack, vigenza_skipped, existence_failed):
    """Ricalibra il confidence score esistente per evitare inflazione."""

    # Riusa i componenti esistenti
    source_reliability = _score_source_reliability(ep)    # 0-1
    vigenza_match = _score_vigenza_match(ep)              # 0-1
    cross_source = _score_cross_source(ep)                # 0-1
    text_quality = _score_text_quality(ep)                # 0-1
    source_agreement = _calc_source_agreement(ep.norms)   # 0-1

    raw = (
        source_reliability * 0.35
        + vigenza_match * 0.25
        + cross_source * 0.20
        + text_quality * 0.10
        + source_agreement * 0.10
    )

    score = round(raw * 100)

    # Penalita' MORBIDE (non cap draconiani)
    if existence_failed:
        score = int(score * 0.85)    # -15%, non cap a 60
    if vigenza_skipped:
        score = int(score * 0.90)    # -10%, non cap a 55

    # Floor a 20 (non 40) — se c'e' almeno 1 evidence
    if total_items > 0 and score < 20:
        score = 20

    return max(0, min(100, score))
```

**Differenze dal confidence attuale:**
- **No floor a 40** — floor a 20 (piu' onesto)
- **Penalita' moltiplicative** (-15%, -10%) invece di cap (60, 55)
- **No inflation bonus** — source_agreement e' parte della formula, non additivo

### 2.2 ReasoningQuality (0-100) — nuovo, lightweight

```python
def reasoning_quality(evidence_pack, synthesis_text, evidence_classification):
    """Valuta la qualita' del ragionamento senza LLM judge (troppo costoso).

    Heuristics deterministiche + segnali strutturali.
    """
    score = 50  # baseline: "sufficiente" se la pipeline ha prodotto output

    # Bonus: citazioni presenti e verificabili (+20 max)
    citations_in_text = count_citation_markers(synthesis_text)  # [1], [2], etc.
    if citations_in_text >= 5: score += 20
    elif citations_in_text >= 3: score += 15
    elif citations_in_text >= 1: score += 10

    # Bonus: diversita' evidence (+15 max)
    has_norms = len(ep.norms) > 0
    has_juris = len(ep.massime) > 0 or len(ep.case_law) > 0
    has_doctrine = any(n.source == "doctrine_search" for n in ep.norms + ep.case_law)
    source_types = sum([has_norms, has_juris, has_doctrine])
    score += source_types * 5  # +5 per tipo (max +15)

    # Bonus: risposta strutturata (+10 max)
    has_sections = synthesis_text.count("###") >= 2 or synthesis_text.count("**") >= 4
    if has_sections: score += 10

    # Bonus: lunghezza adeguata (+5)
    if 1000 < len(synthesis_text) < 15000: score += 5

    # Penalita': risposta troppo corta (-10)
    if len(synthesis_text) < 300: score -= 10

    # Penalita': nessuna citazione in testo legale (-15)
    if citations_in_text == 0 and evidence_classification != "CHAT": score -= 15

    return max(0, min(100, score))
```

**Costo: ZERO** — niente LLM call, solo heuristics strutturali.

### 2.3 Completeness (0-100) — coverage delle fonti

```python
def completeness_score(evidence_pack, intent):
    """Quanto la ricerca ha coperto l'argomento."""
    score = 50

    # Norme trovate
    norms = len(ep.norms)
    if norms >= 5: score += 20
    elif norms >= 3: score += 15
    elif norms >= 1: score += 10

    # Giurisprudenza
    juris = len(ep.massime) + len(ep.case_law)
    if juris >= 5: score += 15
    elif juris >= 2: score += 10
    elif juris >= 1: score += 5

    # Cross-source (fonti diverse)
    sources = len(set(n.source for n in ep.norms))
    if sources >= 3: score += 10
    elif sources >= 2: score += 5

    # Penalita': zero evidence
    if norms == 0 and juris == 0: score -= 20

    # Penalita': solo 1 fonte
    if sources <= 1 and norms > 0: score -= 5

    return max(0, min(100, score))
```

---

## 3. BAND MAPPING — GIUSTO, NON DRACONIANO

| Score | Band | Colore | Significato |
|---|---|---|---|
| **80-100** | **VERIFIED** | Verde | Risposta solida, fonti verificate, ragionamento strutturato |
| **60-79** | **GOOD** | Verde chiaro | Risposta buona ma con margini di miglioramento |
| **40-59** | **CAUTION** | Giallo | Gap significativi — fonti parziali o ragionamento debole |
| **< 40** | **LOW** | Rosso | Qualcosa e' rotto — mancano fonti o la risposta e' incoerente |

**Nota:** CAUTION non e' "attenzione la risposta e' sbagliata". E' "la risposta potrebbe essere migliorata con piu' ricerca". Solo LOW segnala un problema reale.

### 3.1 Stima Score su Dati Reali

Con la formula ibrida, le query del benchmark produrrebbero:

| Query tipo | SourceQ | ReasonQ | ComplQ | **Hybrid** | Band |
|---|---|---|---|---|---|
| Norm-centric standard (art. 2043 cc) | 95 | 85 | 90 | **90** | VERIFIED |
| Norm-centric con vigenza skip | 80 | 80 | 85 | **81** | VERIFIED |
| Zero-norm (neurodiritti) | 60 | 80 | 70 | **69** | GOOD |
| Zero-norm con buona giuris. | 65 | 85 | 80 | **76** | GOOD |
| Query con existence_failed | 70 | 75 | 75 | **73** | GOOD |
| Risposta corta senza citazioni | 50 | 30 | 40 | **41** | CAUTION |
| Timeout / zero evidence | 0 | 20 | 0 | **7** | LOW |

**Le risposte buone stanno sopra 80. Le zero-norm stanno in GOOD (60-79), non in CAUTION. I problemi veri stanno sotto 40.**

---

## 4. ZERO-NORM = LEGALE

### Il cambio concettuale

Il sistema attuale tratta le zero-norm come "meno affidabili" (cap 75, CAUTION). **Sbagliato.** Una risposta basata su principi costituzionali, giurisprudenza SS.UU., e dottrina autorevole e' una risposta legale a pieno titolo.

### Le regole

1. **Nessun cap per zero-norm** — il Solidarity Score cap a 75 viene rimosso
2. **Zero-norm con giurisprudenza forte = VERIFIED** — se ci sono 3+ massime SS.UU. e principi costituzionali ancorati, e' VERIFIED quanto una norma diretta
3. **Il band non dipende dal tipo di evidenza** — dipende dalla qualita' complessiva
4. **Il done_event indica `evidence_mode`** ma il frontend non lo usa per degradare il badge

### Modifica al Solidarity Score

```python
# PRIMA (penalizzante):
overall = min(overall, 75)  # cap fisso

# DOPO (giusto):
# Nessun cap. Il score riflette la qualita' reale.
# Una risposta zero-norm con fonti forti puo' raggiungere 90+.
```

---

## 5. PRODUCTION FIRST — DOVE MODIFICARE

### 5.1 File da modificare

Il production usa il vecchio orchestrator. I file coinvolti:

| File | Modifica |
|---|---|
| `agent/scoring.py` | Aggiungere `compute_hybrid_score()` + rimuovere cap 75 da Solidarity |
| `agent/pipeline.py` | Usare `compute_hybrid_score()` al posto di `compute_confidence()` |
| `gateway/sse_contracts.py` | Il done_event gia' ha `confidence_score` — cambia solo il valore |
| `config.py` | FF `ff_hybrid_score: bool = False` — rollout graduale |

### 5.2 Non servono migration

Lo score e' calcolato runtime e scritto nel done_event + turn_envelope. Nessuna nuova colonna.

### 5.3 Rollout

1. **FF off**: produzione usa `compute_confidence()` (attuale, inflazionato)
2. **FF on staging**: `compute_hybrid_score()` per testing
3. **FF on prod 5%**: canary con metriche Prometheus
4. **FF on prod 100%**: se i numeri sono giusti

---

## 6. IMPLEMENTAZIONE `compute_hybrid_score()`

```python
def compute_hybrid_score(
    evidence_pack: EvidencePack,
    synthesis_text: str = "",
    any_existence_failed: bool = False,
    vigenza_skipped: bool = False,
    evidence_classification: EvidenceClassification | None = None,
) -> ConfidenceScore:
    """Hybrid Score: SourceQuality (40%) + ReasoningQuality (35%) + Completeness (25%).

    Giusto, non draconiano. 80+ per risposte buone. < 40 solo se rotto.
    """
    sq = _source_quality(evidence_pack, any_existence_failed, vigenza_skipped)
    rq = _reasoning_quality(evidence_pack, synthesis_text, evidence_classification)
    cq = _completeness_score(evidence_pack)

    overall = round(sq * 0.40 + rq * 0.35 + cq * 0.25)
    overall = max(0, min(100, overall))

    # Band mapping
    if overall >= 80:
        level = "VERIFIED"
    elif overall >= 60:
        level = "GOOD"
    elif overall >= 40:
        level = "CAUTION"
    else:
        level = "LOW"

    # Breakdown per transparency
    notes = []
    notes.append(f"source={sq} reasoning={rq} completeness={cq}")
    if evidence_classification and evidence_classification.is_zero_norm:
        notes.append(f"evidence_mode={evidence_classification.evidence_type.value}")
    if any_existence_failed:
        notes.append("existence_check_penalty=-15%")
    if vigenza_skipped:
        notes.append("vigenza_skip_penalty=-10%")

    return ConfidenceScore(
        overall=overall,
        normativa=sq / 100,
        giurisprudenza=_score_giurisprudenza(evidence_pack),
        vigenza=_score_vigenza_match(evidence_pack) if not vigenza_skipped else 0.5,
        coverage=cq / 100,
        cross_source_coherence=_score_cross_source(evidence_pack),
        breakdown_note="; ".join(notes),
    )
```

### 6.1 Per production (vecchio orchestrator)

Il vecchio orchestrator in `advanced_orchestrator.py` chiama `compute_confidence()`. Il fix:

```python
# In advanced_orchestrator.py, dove viene calcolato il confidence:
if settings.ff_hybrid_score:
    confidence = compute_hybrid_score(
        evidence_pack, synthesis_text, existence_failed, vigenza_skipped,
    )
else:
    confidence = compute_confidence(evidence_pack, existence_failed, vigenza_skipped)
```

---

## 7. TIMELINE

| Step | Durata | Deliverable |
|---|---|---|
| 1 | 1 giorno | `compute_hybrid_score()` in scoring.py + test |
| 2 | 1 giorno | Integrazione in pipeline.py (staging) + advanced_orchestrator.py (prod) |
| 3 | 1 giorno | Rimuovere cap 75 da Solidarity Score |
| 4 | 1 giorno | Deploy staging, benchmark, calibrazione pesi |
| 5 | 1 giorno | Deploy prod con FF, canary 5% → 100% |

**5 giorni totali.**

---

## 8. METRICHE DI SUCCESSO

| KPI | Target |
|---|---|
| Score medio su Kryptonite 65 | 80-90 |
| Score medio su zero-norm 25 | 65-80 |
| Items sotto 40 (genuinamente rotti) | < 5% |
| Items a 100 (sospetto inflazione) | < 20% |
| Correlazione con LLM Judge | > 0.5 (vs 0.1 attuale) |

---

*"Non voglio rosso, non voglio giallo — voglio giusto."*
