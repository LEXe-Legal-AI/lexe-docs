# LEXE Improve Lab — Architettura di Auto-Miglioramento

> Documento strategico per il modulo di auto-miglioramento della pipeline LEGIS AGENT.
> Basato su Kryptonite Bench Round 1 e dati di produzione.
> Versione 1.0 — 2026-03-21

---

## 1. Architettura

Event pipeline: conversation_events (21 tipi) + turn_envelopes (confidence) + message_evaluations (rating 1-5) → Event Aggregator → Signal Correlator → Insight Engine → Auto-Tuning / Alert / Report.

## 2. Data-Driven Pipeline Tuning

### 2.1 Researcher

- 7/65 prompt con 0 norme (10.8%) — collo di bottiglia principale
- Root cause: planner non emette key_norms per query implicite, scenari dedicati mancano kb_normativa_search
- Fix: fallback chain, query broadening, kb_normativa_search in tutti gli scenari

### 2.2 Verifier

- 26/65 high_risk (40%), di cui 4 false positive (15.4%)
- Root cause: grounding_score penalizza norme corrette ma non in evidence pack; hallucination_risk e un veto binario assoluto
- Fix P0: grounding normalization per evidence pack sparse
- Fix P1: demotare hallucination_risk da veto a segnale pesato
- Fix P2: normalizzazione act_type simmetrica tra extractor e evidence pack

### 2.3 Synthesizer

- FONTI present: 75% (target 95%)
- Cause: strip_leaked_content troppo aggressivo, template SIMPLE senza FONTI, token budget insufficiente

## 3. Cicli di Miglioramento

1. **Raccolta** (2 sett): accumula 200+ turn envelopes + 50+ evaluations
2. **Analisi** (2-3 gg): Kryptonite Bench + confronto baseline
3. **Tuning** (3-5 gg): test varianti parametri su bench
4. **Deploy** (1-2 gg): staging + gate validation + promote
5. **Continuo**: regression monitoring always-on (Z-score anomaly detection)

## 4. Experiment Framework

- A/B testing: hash-based tenant assignment, paired t-test, Wilcoxon
- Shadow mode: nuovo modello in parallelo, zero impatto utente
- Canary deployment: 5% traffico + benchmark gate
- Rollback automatico: confidence drop >10pt in 1h, error rate >5%

## 5. Actionable Insights Kryptonite R1

| Priority | Issue | Impact | Fix | Timeline |
|----------|-------|--------|-----|----------|
| P0 | 7 prompt con 0 norme | zero_norms 10.8% | Fallback chain + kb_normativa_search in scenari | 3gg |
| P1 | 4 false positive verifier | FP rate 15.4% | Grounding normalization + hallucination demotion | 3gg |
| P1 | FONTI 75% | Credibilita | Audit strip_leaked_content + FONTI obbligatorie | 2gg |
| P2 | IR solo 23% attivazione | Coverage bassa | Soglia IR ridotta (0.4 → 0.3) | 3gg |
| P2 | id=34 IR diluisce score | conf 80→79 | IR promoted = stessi attributi originali | 2gg |

## 6. KPI Dashboard

| Metrica | Target | Alert |
|---------|--------|-------|
| confidence_avg | >= 55 | < 45 |
| VERIFIED% | >= 50% | < 35% |
| high_risk% | <= 15% | > 25% |
| FONTI% | >= 95% | < 80% |
| zero_norms% | <= 3% | > 10% |
| ir_activation | >= 40% | < 15% |
| eval_avg_rating | >= 4.0 | < 3.0 |

## 7. Timeline

| Milestone | Sprint | Criteri |
|-----------|--------|---------|
| M1: Quick Wins | 12 | zero_norms < 5%, fonti > 90% |
| M2: Calibrated Pipeline | 13 | VERIFIED > 45%, high_risk < 20% |
| M3: Observable Pipeline | 14 | Dashboard live, alerting |
| M4: Self-Improving | 16 | A/B framework, primo ciclo completo |

---

*Owner: LEXE Platform Team — 2026-03-21*
