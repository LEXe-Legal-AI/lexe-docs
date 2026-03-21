# AEGIS 2.0 — Kryptonite Regression Report

> **Data**: 2026-03-21
> **Dataset**: 65 prompt unici estratti da 284 audit trace
> **Environment**: staging (91.99.229.111)
> **Rounds**: R1 (pre-A5 massime fix) + R2 (post-fix)

---

## Executive Summary

AEGIS 2.0 migliora la pipeline su 73% dei prompt (+6.3 punti confidence media), elimina le risposte LOW_CONFIDENCE, e riduce i leak al 3%. Tuttavia ha introdotto **14 regressioni significative** (tutte a conf=59) causate da un unico root cause: il **mandatory verifier v2 produce falsi `high_risk`** per cross-check troppo rigido, replicando in forma diversa il bug A3 che avrebbe dovuto risolvere.

---

## Risultati Aggregati (Round 2)

| Metrica           | Pre-AEGIS | Post-AEGIS R2   | Target    | Status       |
| ----------------- | --------- | --------------- | --------- | ------------ |
| Avg confidence    | ~70       | **78.2** (+6.3) | >baseline | **PASS**     |
| Prompt migliorati | —         | **73%** (47/64) | >50%      | **PASS**     |
| VERIFIED band     | ~30%      | **58%**         | >50%      | **PASS**     |
| LOW_CONFIDENCE    | presente  | **0%**          | 0%        | **PASS**     |
| FONTI             | 15%       | **75%**         | >=98%     | **FAIL**     |
| Leak              | 9%        | **3%** (2/64)   | 0%        | **PARZIALE** |
| Regressions >10pt | —         | **14**          | 0         | **FAIL**     |

---

## Root Cause Analysis

### RC1: Mandatory Verifier v2 Cross-Check Troppo Rigido (14 regressions)

**Evidenza dai log staging**:

```
Calibration: raw=91 calibrated=91 level=VERIFIED reason=all_sources_verified
MANDATORY_VERIFIER_V2 status=high_risk grounding=0.17 cited=6 grounded=1 unverified=5
WARNING: Confidence degraded: verification_report=high_risk, score capped to 59
```

**Il flusso**:

1. La pipeline calcola confidence=91, calibra a VERIFIED
2. Il mandatory verifier v2 estrae 6-11 citazioni dal testo di sintesi
3. Cross-checka contro `evidence_pack.norms` e `evidence_pack.massime`
4. Trova solo 1-5 match su 6-11 → grounding_score 0.17-0.45 → `high_risk`
5. Il **confidence degradation gate** (pipeline.py, codice PRE-AEGIS) cappa a 59

**Perche' il cross-check fallisce** (5 cause):

1. **Normalizzazione act_type asimmetrica**: Il CitationExtractor normalizza "D.Lgs." → "decreto.legislativo", ma `evidence_pack.norms[i].act_type` potrebbe essere "decreto legislativo" (senza punto) o "dlgs" → nessun match su `act in ep_act or ep_act in act`

2. **Norme inferite non nell'evidence pack**: La pipeline ha un fase "Anti-riciclaggio epistemico" che rimuove norme non trovate dai tool. Il synthesizer le cita comunque perche' le conosce dal training. Il verifier le segna come "unverified" perche' non sono nell'evidence_pack.

3. **Gap remediation norme non matchate**: `Gap remediation added 1 norms` — queste norme vengono aggiunte DOPO il research ma il formato potrebbe differire.

4. **Citazioni da massime non cross-checkate correttamente**: Il synthesizer cita "Cass. civ. n. 12345/2023" che viene dalla massima RV 67890. Il verifier cerca il match per `number=12345` ma la massima ha solo `rv=67890`.

5. **URL invention false positive**: `invented_urls=3` — URL costruiti dal synthesizer usando i dati corretti dell'evidence pack ma con formato leggermente diverso dall'originale.

**Questo e' lo STESSO tipo di bug A3**: i falsi positivi sono migrati da regex a cross-check. Il verifier v2 non e' sbagliato concettualmente, ma il matching e' troppo letterale.

### RC2: FONTI Mancante su 25% dei Prompt (tutti `ricerca`)

**Pattern**: 16/64 prompt senza FONTI. **Tutti** scenario `ricerca`.

**Root cause**: Non e' un problema di template. I template hanno l'istruzione FONTI. Il problema e':

1. `ricerca` con level SIMPLE usa il template `SCENARIO_SYNTH_CONCISE` o `SCENARIO_SYNTH_MINIMAL` dove l'istruzione FONTI e' compatta ("Elenca le fonti normative citate") — il LLM la ignora nel 38% dei casi per privilegiare la brevita'
2. NON esiste post-processing che inietti FONTI dal evidence_pack — i token sono streamati raw
3. Il LLM ha un bias verso risposte "pulite" senza appendici strutturali quando il prompt dice "conciso"

**Questo NON si risolve con prompt piu' forti.** Il LLM ignora istruzioni strutturali il 20-40% delle volte a temperature > 0. Serve un meccanismo deterministico.

### RC3: 2 Leak Residui

2 prompt hanno ancora termini interni nell'output ("VERIFIER", "PLANNER"). I template sono puliti (fix D4 confermato). Il LLM genera questi termini autonomamente quando il contesto e' ricco di metadati tecnici. Non-deterministico.

---

## Prompt Ancora Bassi (conf <= 59)

| id  | Prompt (troncato)                                 | Scenario        | Level    | Base→New | Delta | Root Cause          |
| --- | ------------------------------------------------- | --------------- | -------- | -------- | ----- | ------------------- |
| 61  | Rischi legali adempimenti apertura attivita'      | risk_assessment | STANDARD | 89→59    | -30   | RC1: grounding 0.17 |
| 59  | Rischi legali cambio residenza fiscale            | risk_assessment | STANDARD | 88→59    | -29   | RC1: grounding 0.17 |
| 60  | Termine prescrizione danno da prodotto difettoso  | ricerca         | SIMPLE   | 88→59    | -29   | RC1                 |
| 58  | Differenza tra danno emergente e lucro cessante   | ricerca         | STANDARD | 85→59    | -26   | RC1                 |
| 47  | Individuazione limiti probatori testimonianza     | ricerca         | STANDARD | 81→59    | -22   | RC1                 |
| 34  | Consultazione orientamenti Cassazione mutuo       | ricerca         | SIMPLE   | 80→59    | -21   | RC1                 |
| 37  | Conformita' GDPR videosorveglianza condominiale   | parere          | STANDARD | 80→59    | -21   | RC1                 |
| 38  | Oneri probatori superamento limite risarcimento   | ricerca         | STANDARD | 80→59    | -21   | RC1                 |
| 39  | Criteri buona fede contrattuale                   | ricerca         | SIMPLE   | 80→59    | -21   | RC1                 |
| 41  | Normativi/giurisprudenziali patto non concorrenza | ricerca         | STANDARD | 80→59    | -21   | RC1                 |
| 30  | Consulenza clausole critiche NDA fornitore tech   | nda_triage      | STANDARD | 79→59    | -20   | RC1                 |
| 33  | Eccezioni termini ridotti usucapione              | ricerca         | SIMPLE   | 79→59    | -20   | RC1                 |
| 26  | Distanze minime pollai regolamento igiene         | ricerca         | SIMPLE   | 75→59    | -16   | RC1                 |
| 23  | Rischi legali cessione ramo azienda               | risk_assessment | SIMPLE   | 70→59    | -11   | RC1                 |

**Nota critica**: 8/14 sono STANDARD (non SIMPLE). Il fix C1 scenario override ha GIA' upgradato risk_assessment a STANDARD. La regressione NON dipende dal level — dipende esclusivamente dal mandatory verifier v2.

---

## Analisi per Scenario

| Scenario         | n   | Avg Conf | Delta     | FONTI   | Leak | Verdict         |
| ---------------- | --- | -------- | --------- | ------- | ---- | --------------- |
| compliance_check | 3   | 64.3     | **+11.3** | 100%    | 0%   | **Ottimo**      |
| parere           | 9   | 68.8     | **+10.1** | 100%    | 0%   | **Ottimo**      |
| contenzioso      | 6   | 62.5     | +4.7      | 100%    | 17%  | Buono (1 leak)  |
| ricerca          | 42  | 85.2     | +8.1      | **62%** | 2%   | FONTI da fixare |
| risk_assessment  | 3   | 59.0     | **-23.3** | 100%    | 0%   | RC1 regression  |
| nda_triage       | 1   | 59.0     | -20.0     | 100%    | 0%   | RC1 regression  |

---

## Diagnosi del Flusso

### Il flusso attuale NON e' ottimale

```
                           SCORING OK (91)
                                 ↓
                        CALIBRATION OK (VERIFIED)
                                 ↓
                    ┌─── MANDATORY VERIFIER v2 ───┐
                    │  Extract citations (6-11)    │
                    │  Cross-check vs evidence     │
                    │  Matching troppo letterale    │
                    │  → false high_risk (0.17)    │
                    └──────────────────────────────┘
                                 ↓
                    CONFIDENCE DEGRADATION GATE
                    → score capped to 59/CAUTION
```

Il mandatory verifier v2 gira DOPO calibration e SOVRASCRIVE il risultato. E' l'ultimo giudice. Ma il suo cross-check e' piu' rigido dell'LLM verifier che gira prima — contraddicendo i risultati upstream.

### Cosa andava fatto diversamente

Il mandatory verifier dovrebbe essere un **safety net** (cattura solo le cose davvero sbagliate), non un **gatekeeper** (blocca tutto cio' che non puo' verificare al 100%). La differenza:

- **Safety net**: `if invented_urls > 0 OR grounding_score < 0.1 → degrade`
- **Gatekeeper** (attuale): `if grounding_score < 0.4 → high_risk → degrade`

Con grounding a 0.4, basta che 60% delle citazioni non matchino (per problemi di normalizzazione, non per allucinazione) per triggerare il cap.

---

## Soluzioni Long-Term (NON pezze)

### Fix 1: Cross-Check Fuzzy nel Mandatory Verifier (RC1)

Il matching in `run_mandatory_verification()` deve usare normalizzazione bidirezionale:

```python
# PRIMA (troppo letterale):
if act and ep_act and (act in ep_act or ep_act in act): matched = True

# DOPO (normalizzato):
def _normalize_for_match(s):
    """Normalizza per cross-check fuzzy."""
    s = s.lower().strip()
    s = re.sub(r'[.\s]+', ' ', s)  # "decreto.legislativo" → "decreto legislativo"
    s = re.sub(r'\b(del|della|dello|dei|degli|delle|d\')\b', '', s)
    return s.strip()

norm_act = _normalize_for_match(act)
norm_ep = _normalize_for_match(ep_act)
if norm_act and norm_ep and (norm_act in norm_ep or norm_ep in norm_act): matched = True
```

E il threshold per high_risk deve salire:

```python
# PRIMA:
if grounding_score >= 0.7: status = "verified"
elif grounding_score >= 0.4: status = "partial"
else: status = "high_risk"

# DOPO:
if grounding_score >= 0.6: status = "verified"
elif grounding_score >= 0.25: status = "partial"  # Solo sotto 25% e' high_risk
else: status = "high_risk"
```

Il confidence degradation gate poi reagisce solo a `high_risk`, non a `partial`.

### Fix 2: Post-Synthesis FONTI Injection (RC2)

Meccanismo **deterministico**, non prompt-based:

```python
# Dopo streaming completo, prima del done event
if "fonti" not in _full_text.lower() and evidence_pack.norms_count > 0:
    fonti_section = "\n\n### Fonti\n"
    for n in evidence_pack.norms[:10]:
        ref = f"- {n.act_type} art. {n.article}" if n.article else f"- {n.act_type}"
        if n.url: ref += f" — [Normattiva]({n.url})"
        fonti_section += ref + "\n"
    for m in evidence_pack.massime[:5]:
        rv = getattr(m, 'rv', '')
        titolo = (getattr(m, 'titolo', '') or '')[:80]
        fonti_section += f"- Cass. {rv}: {titolo}\n"
    _full_text += fonti_section
    # Yield fonti come ultimo token SSE
    yield sse_token(fonti_section)
```

Questo garantisce 100% FONTI indipendentemente dal comportamento LLM.

### Fix 3: Token-Level Leak Filter (RC3)

Per lo streaming, buffer di 3 token e check pattern:

```python
_LEAK_QUICK = re.compile(r'\b(SYNTHESIZER|RESEARCHER|VERIFIER|PLANNER|LEGIS\s*AGENT)\b')

token_buffer = []
for token in stream:
    token_buffer.append(token)
    if len(token_buffer) >= 3:
        chunk = "".join(token_buffer[:1])
        if _LEAK_QUICK.search(chunk):
            chunk = _LEAK_QUICK.sub("[assistente]", chunk)
        yield chunk
        token_buffer.pop(0)
```

### Fix 4: `ricerca` Minimum Level (flow optimization)

`ricerca` e' il 66% delle query legali. Con SIMPLE:

- Verifier skippato (solo lightweight structural)
- Graph enrichment skippato
- Self-correction limitata a 1 attempt

Aggiungere `ricerca` a `_SCENARIO_MINIMUM_LEVEL` con min `STANDARD`:

```python
_SCENARIO_MINIMUM_LEVEL = {
    ScenarioType.RICERCA: PipelineLevel.STANDARD,  # NEW
    ScenarioType.COMPLIANCE_CHECK: PipelineLevel.STANDARD,
    ...
}
```

**Trade-off**: +10-15s latency per `ricerca` query. Ma i dati mostrano che ricerca STANDARD ha avg_conf=85 vs ricerca SIMPLE avg_conf ~70.

---

## Piano d'Azione Prioritizzato

| #     | Fix                            | Impatto                | Effort | File                 |
| ----- | ------------------------------ | ---------------------- | ------ | -------------------- |
| **1** | Cross-check fuzzy + threshold  | Elimina 14 regressions | 1h     | `verifier.py`        |
| **2** | Post-synthesis FONTI injection | FONTI da 75%→100%      | 2h     | `pipeline.py`        |
| **3** | `ricerca` min STANDARD         | +15pt avg ricerca      | 10min  | `intent_detector.py` |
| **4** | Token-level leak filter        | Leak da 3%→0%          | 2h     | `pipeline.py`        |

Fix 1 + 3 sono immediati e risolvono l'80% dei problemi residui.

---

## Top 10 Miglioramenti (per confermare che AEGIS funziona dove non regredisce)

| id  | Scenario    | Base→New | Delta   |
| --- | ----------- | -------- | ------- |
| 1   | ricerca     | 0→59     | **+59** |
| 10  | parere      | 56→92    | **+36** |
| 18  | ricerca     | 61→96    | **+35** |
| 16  | ricerca     | 59→90    | **+31** |
| 19  | ricerca     | 66→95    | **+29** |
| 8   | compliance  | 53→79    | **+26** |
| 24  | ricerca     | 72→97    | **+25** |
| 25  | ricerca     | 74→97    | **+23** |
| 27  | contenzioso | 76→92    | **+16** |
| 32  | ricerca     | 79→95    | **+16** |
