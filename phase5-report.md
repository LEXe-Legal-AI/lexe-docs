# LEXE Phase 5 — Experiment Lifecycle & LLM Judge

## Report di Implementazione

**Data**: 2026-02-23
**Durata sessione**: ~45 minuti
**Ambiente target**: Staging (91.99.229.111)
**Status**: Completato e verificato E2E su staging

---

## 1. Obiettivo

Chiudere il loop auto-improve con:
- Experiment lifecycle CRUD (create, list, get, promote, reject)
- LLM-as-judge per scoring qualitativo delle risposte
- Pipeline end-to-end (collect → detect → propose → create → judge → report)
- Admin API REST per gestione esperimenti dal pannello admin

---

## 2. Decisioni Architetturali

| Decisione | Motivazione |
|-----------|-------------|
| **NO Temporal** | Zero integrazione in lexe-core, effort eccessivo |
| **NO A/B testing live** | Richiede modifiche a customer_router — defer Phase 6 |
| **LLM-as-judge** per replay | Alternativa pragmatica alla ri-esecuzione completa pipeline |
| **Promozione manuale** | Nessuna applicazione automatica di config in prod |
| **asyncpg diretto** | Segue pattern assessment_router (no ORM) |
| **Schemas self-contained** | Nessuna dipendenza di lexe-core da lexe-improve-lab |

---

## 3. File Prodotti

### lexe-improve-lab (7 file, 5 commit)

| File | Azione | Righe | Step |
|------|--------|-------|------|
| `src/lexe_improve/config.py` | MODIFICA | 49 | 1 |
| `src/lexe_improve/experiments/__init__.py` | NUOVO | 6 | 2 |
| `src/lexe_improve/experiments/schemas.py` | NUOVO | 58 | 2 |
| `src/lexe_improve/experiments/manager.py` | NUOVO | 201 | 2 |
| `src/lexe_improve/replay/judge.py` | NUOVO | 130 | 3 |
| `src/lexe_improve/replay/runner.py` | MODIFICA | 132 | 4 |
| `src/lexe_improve/cli.py` | MODIFICA | 393 | 5 |
| `tests/test_experiments.py` | NUOVO | 252 | 7 |
| `tests/test_judge.py` | NUOVO | 161 | 7 |
| `tests/test_pipeline.py` | NUOVO | 159 | 7 |
| **Subtotale** | | **1,541** | |

### lexe-core (2 file, 3 commit)

| File | Azione | Righe | Step |
|------|--------|-------|------|
| `src/lexe_core/admin/routers/experiments_router.py` | NUOVO | 223 | 6 |
| `src/lexe_core/admin/router.py` | MODIFICA (+4 righe) | — | 6 |
| **Subtotale** | | **223** | |

### Totale

| Metrica | Valore |
|---------|--------|
| **Righe di codice prodotte** | ~1,764 |
| **Righe di test** | 572 (32% del totale) |
| **File nuovi** | 8 |
| **File modificati** | 4 |
| **Test totali** | 63 (34 esistenti + 29 nuovi) |
| **Test pass rate** | 100% |

---

## 4. Commit History

### lexe-improve-lab → `stage`

| Hash | Messaggio |
|------|-----------|
| `8f0905f` | feat: Phase 5 — experiment lifecycle, LLM-as-judge, pipeline CLI |
| `ad3fa6b` | fix: add LiteLLM API key support to LLM judge |
| `881dab3` | fix: use stdlib logger format strings in judge parse_error |
| `ff37bdb` | fix: allow None for warnings field in ExperimentRecord |
| `63496d7` | fix: align experiment schemas to actual DB columns |

### lexe-core → `stage`

| Hash | Messaggio |
|------|-----------|
| `31ac191` | feat: Phase 5 — experiments admin router for auto-improve lifecycle |
| `88094b8` | fix: allow None for warnings field in ExperimentResponse |
| `7f5b807` | fix: align experiments router schemas to actual DB columns |

---

## 5. Verifica E2E su Staging

| Test | Risultato |
|------|-----------|
| `pytest tests/` (63 test) | 63/63 passed |
| `lexe-improve experiment list` | Lista vuota → poi con esperimento |
| `lexe-improve experiment show <ID>` | JSON completo con tutti i campi DB |
| `lexe-improve experiment promote <ID>` | draft → promoted |
| `lexe-improve experiment reject <ID>` (su promoted) | Correttamente rifiutato |
| `lexe-improve pipeline run --period 7d --dry-run` | 78 conv, 0 issues, 5 HTTP 200, 3 improvements, 1 regression |
| OpenAPI `/admin/experiments` | 4 endpoint registrati (GET list, GET detail, POST promote, POST reject) |
| lexe-core container | healthy |

### Pipeline E2E Output (staging)

```
[1/5] Collecting conversations (period=7d)... → 78 conversations
[2/5] Detecting issues...                     → 0 issues found
[3/5] Proposing fixes...                      → 0 proposals
[4/5] DRY RUN — skipping experiment creation
[5/5] Judging 10 conversations via LLM...     → Regressions: 1, Improvements: 3, Unchanged: 6
```

---

## 6. Componenti Implementati

### LLM-as-Judge (`judge.py`)

- **3 dimensioni di scoring**: relevance, accuracy, completeness (0-100)
- **Composite pesato**: accuracy×0.5 + relevance×0.3 + completeness×0.2
- **Concurrency**: asyncio.Semaphore(5) per batch scoring
- **Resilienza**: timeout handling, markdown fencing strip, out-of-range clamping, JSON parse errors
- **Auth**: Bearer token via `LEXE_IMPROVE_LITELLM_API_KEY`

### ExperimentManager (`manager.py`)

- **CRUD completo**: create, create_from_proposal, get, list, update, promote, reject
- **JSONB parsing**: 7 colonne JSONB gestite (`json.loads()` per asyncpg string return)
- **State machine**: draft → running → completed → promoted/rejected
- **Guard clauses**: promote solo da completed/draft, reject impossibile se promoted

### Pipeline CLI (`cli.py`)

- **5 fasi**: collect → detect → propose → create experiments → judge replay
- **Dry-run mode**: esegue tutto senza persistere esperimenti
- **Report finale**: conteggi di conversations, issues, proposals, experiments, replayed

### Admin Router (`experiments_router.py`)

- **4 endpoint**: GET list (con filtro status + pagination), GET detail, POST promote, POST reject
- **RBAC**: CanReadSettings per lettura, CanManageSettings per promote/reject
- **409 Conflict**: feedback chiaro su stato incompatibile (es. "Cannot promote: current status is 'rejected'")

---

## 7. Modalita di Sviluppo

### Agenti Utilizzati

| Agente | Tipo | Scopo | Contesto |
|--------|------|-------|----------|
| **Claude Opus 4.6** | Main | Implementazione diretta di tutti i 7 step | Foreground |
| **SynchronizerOptimizer** | Subagent | Allineamento schemas al DB reale (4 file in parallelo) | Foreground |
| **Explore** | Subagent | Verifica finale compliance piano vs implementazione | Foreground |
| **Totale agenti** | **3** | | |

### Strumenti di Sviluppo

| Tool | Utilizzo | Note |
|------|----------|------|
| **Write** | Creazione 8 file nuovi | judge.py, manager.py, schemas.py, router, 3 test |
| **Edit** | Modifiche chirurgiche a 4 file | config.py, cli.py, runner.py, admin/router.py |
| **Read** | Lettura file esistenti pre-modifica | assessment_router (pattern), models.py, fixes.py |
| **Bash** | 30+ comandi SSH per staging | git pull, docker build, pip install, curl test |
| **Glob/Grep** | Esplorazione codebase | Verifica struttura, RBAC types, file esistenti |
| **pytest** | Test runner locale | 5 esecuzioni, da 58 fail a 63/63 pass |
| **SSH** | Deploy staging remoto | Pull, build, restart container |
| **Docker Compose** | Deploy con override staging | `--build lexe-core` per rebuild image |
| **curl** | API testing | OpenAPI verification, LiteLLM models, token flow |

### Workflow

```
1. Read file esistenti (pattern assessment_router, models.py, fixes.py)
2. Implementazione Step 1-3 in parallelo (config + schemas/manager + judge)
3. Implementazione Step 4+6 in parallelo (runner + admin router)
4. Implementazione Step 5 (CLI — dipende da 2+4)
5. Mount router in admin/router.py
6. Creazione 3 file test in parallelo
7. Test locale → fix mock (AsyncMock→MagicMock per httpx) → 63/63 pass
8. Commit selettivo (solo file Phase 5, non altri WIP)
9. Push → SSH pull → Docker build → Deploy
10. E2E staging: experiment lifecycle + pipeline + judge
11. 4 fix iterativi (LiteLLM auth, logger format, nullable warnings, schema alignment)
12. Verifica finale completa
```

### Bug Incontrati e Risolti in Sessione

| Bug | Causa | Fix |
|-----|-------|-----|
| AsyncMock.json() returns coroutine | httpx Response.json() è sincrono | Usare MagicMock per response |
| Logger kwargs error | `logging.warning(msg, key=val)` non supportato | Format string `%s` |
| 401 Unauthorized dal judge | LiteLLM richiede API key | Aggiunto `litellm_api_key` + Bearer header |
| `warnings` field validation error | DB ha NULL, Pydantic vuole `list[str]` | `list[str] \| None` |
| `updated_at` column not found | Schema DB ha `completed_at`, non `updated_at` | Allineamento completo a 4 file |

---

## 8. Stima Token e Costi

| Metrica | Stima |
|---------|-------|
| **Token input** (contesto + file letti) | ~150,000 |
| **Token output** (codice + messaggi) | ~30,000 |
| **Token subagent SynchronizerOptimizer** | ~44,000 |
| **Token subagent Explore** | ~49,000 |
| **Token totali sessione** | ~273,000 |
| **Chiamate LLM judge su staging** | ~15 (5+10 in 2 pipeline run) |

---

## 9. Prossimi Passi (Phase 6)

- A/B testing live con traffic splitting in customer_router
- Dashboard admin per visualizzazione esperimenti
- Automazione scheduling pipeline (cron o task periodico)
- Metriche Prometheus per experiment outcomes
- Integrazione Langfuse per tracing delle chiamate judge

---

*Generato automaticamente — 2026-02-23*
