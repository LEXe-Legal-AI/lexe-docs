# LEXE Benchmark Suite — Documentazione Tecnica

## Indice

1. [Architettura del Benchmark System](#1-architettura-del-benchmark-system)
2. [Integrazione Admin Panel](#2-integrazione-admin-panel)
3. [Espansione del Dataset](#3-espansione-del-dataset)
4. [Metriche Avanzate](#4-metriche-avanzate)
5. [Automated Regression Detection](#5-automated-regression-detection)
6. [Benchmark Comparison](#6-benchmark-comparison)

---

## 1. Architettura del Benchmark System

### 1.1 Kryptonite Parallel Runner

Il Kryptonite runner esegue l'intero dataset di prompt legali attraverso la pipeline E2E di LEXE, simulando sessioni utente reali. L'architettura si basa su un pool di 10 worker asincroni.

**Risultati del primo run Kryptonite (2026-03-21)**:

| Metrica | Valore |
|---------|--------|
| Prompt totali | 65 |
| Avg confidence | 79.6 |
| VERIFIED | 40 (61.5%) |
| CAUTION | 25 (38.5%) |
| LOW_CONFIDENCE | 0 (0%) |
| IR triggered | 15/65 (23.1%) |
| IR promotion hit rate | 93% |
| Verifier high_risk | 26/65 (40%) |
| Avg latency | ~30s |
| Wall time (10 workers) | 218s |

### 1.2 Runner scripts

- `tests/kryptonite_parallel.py` — 10 worker async, metriche IR v3
- `tests/kryptonite_runner.py` — seriale legacy
- `tests/KRYPTONITE_BENCH.md` — istruzioni e best practices

### 1.3 Dataset

`tests/fixtures/kryptonite_dataset.json` — 65 prompt, 7 scenari, baseline pre-AEGIS.

---

## 2. Integrazione Admin Panel

### 2.1 API Endpoints

- `GET /admin/bench/pipeline-snapshot` — congela config corrente
- `POST /admin/bench/runs` — avvia run
- `GET /admin/bench/runs/{id}/compare/{other_id}` — delta tra due run

### 2.2 Visualizzazione

- Heatmap per scenario (confidence x scenario)
- Trend temporale (confidence over runs)
- Drill-down per prompt (risposta + norme attese vs trovate)

---

## 3. Espansione del Dataset

### 3.1 Categorie mancanti

| Area | Prompt stimati | Priorita |
|------|---------------|----------|
| Diritto tributario | 15-20 | P1 |
| Diritto societario | 12-15 | P1 |
| Diritto fallimentare/CCII | 10-12 | P1 |
| Diritto int. privato | 8-10 | P2 |
| Diritto famiglia | 8-10 | P2 |
| Diritto informatica | 6-8 | P2 |
| Diritto ambientale | 6-8 | P3 |
| Diritto sanitario | 5-8 | P3 |

Target: 200+ prompt per LexBench-IT v1.0.

---

## 4. Metriche Avanzate

- **IR**: trigger_rate, promotion_rate, cache_hit_ratio
- **Verifier**: false_positive_rate, true_positive_rate
- **Researcher**: avg_norms_per_query, tool_success_rate
- **Latency**: breakdown per fase (intent, research, verify, synthesize, IR)
- **Text quality**: FONTI presence, link validity, citation consistency, leak detection

---

## 5. Automated Regression Detection

- CI/CD: run benchmark su ogni merge a stage
- Alert su regressioni >10pt (Prometheus + Slack)
- Dashboard Grafana per trend storici

---

## 6. Benchmark Comparison

- Pre/post deploy comparison table
- A/B testing tra configurazioni (paired t-test, Wilcoxon, Bootstrap CI)
- Statistical significance con soglie decisionali

---

*Ultimo aggiornamento: 2026-03-21*
