# LEXE - LLM Selection & Model Configuration

> Scelte modelli LLM basate su benchmark Phase 1 (6 run, 11 modelli, 2 judge, peer-review)
> *Ultimo aggiornamento: 2026-02-23*

---

## 1. Modelli scelti per ruolo LEXE

**Strategia: Gemini 3 Flash principale + DeepSeek/Qwen per fasi critiche**

| Alias LiteLLM | Ruolo | Modello Scelto | reasoning_effort | max_tokens | Motivazione |
|---|---|---|---|---|---|
| `lexe-fast` | Fast (follow-up, classification) | `gemini-3-flash-preview` | `medium` | 1024 | Best score 4.75, 2.7-9s, $0.05/$0.30 |
| `lexe-primary` | Primary (chat, consulenza) | `gemini-3-flash-preview` | `medium` | 8192 | Score 4.60, stabile, economico |
| `lexe-complex` | Complex (LEGIS, planner) | `gemini-3-flash-preview` | `medium` | 8192 | Score 4.35, 10s, domina tutti i run |
| `legal-tier2-haiku` | Verifier/audit | `claude-haiku-4.5` | `none` | 4096 | Veloce 3.6s, preciso, buon italiano |
| `legal-tier3-sonnet` | Writing quality | `claude-sonnet-4.6` | default | 8192 | Eccellente italiano |
| `lexe-embedding` | Embedding | `text-embedding-3-small` | N/A | N/A | Invariato |

---

## 2. reasoning_effort per modello

Il campo `reasoning_effort` è salvato nel JSONB `config` di `core.llm_model_catalog` e applicato automaticamente a runtime.

| Modello | reasoning_effort | Note |
|---|---|---|
| `gemini-3-flash-preview` | `medium` | Ottimo balance qualità/velocità |
| `claude-haiku-4.5` | `none` | Non supporta reasoning — viene strippato dal payload |
| `claude-sonnet-4.6` | default (non impostato) | Reasoning nativo, no override necessario |
| `gpt-5-mini` | `low` | **CRITICO!** Da score 1.00 a 4.75 con `low`. Mai usare senza. NO max_tokens. |
| `deepseek-v3.2` | `medium` | Peer score 5.00 FAST, 4.62 COMPLEX |
| `qwen-3.5-plus` | `medium` | Judge più calibrato, self-bias -0.06 |

---

## 3. Modelli candidati e alternativi

### Supporto per fasi critiche

| Modello | Ruolo supporto | Quando usare | Costo (in/$, out/$) |
|---|---|---|---|
| `deepseek-v3.2` | Fallback critical / 2nd opinion | Analisi complesse, fasi dove serve 2nd check | $0.14 / $0.28 |
| `qwen-3.5-plus` | Judge / verifica qualità | Valutazione risposte, peer-review interno | economico |
| `gpt-5-mini` | Fallback economico | Se Gemini non disponibile | $0.25 / $2.00 |

### Warning noti

- **DeepSeek V3.2**: eccellente candidato ma **BROKEN come judge** (assegna 1.00 a se stesso)
- **Qwen 3.5 Plus**: judge più calibrato nel peer-review, basso self-bias
- **GPT-5-mini**: **OBBLIGATORIO** `reasoning_effort=low` e **NO** `max_tokens` (causa score 1.00 se omesso)

---

## 4. Come funziona il routing

### Flusso runtime

```
Request → customer_router.py
  ├── JOIN core.model_role_defaults + core.llm_model_catalog
  │     → Risolve: ruolo → model_name → config JSONB
  ├── _apply_model_config(payload, config, defaults)
  │     → apply_config_override() da agent/config_utils.py
  │     → Mappa: reasoning_effort, max_tokens, temperature, etc.
  └── LiteLLM → Provider (OpenRouter / diretto)
        → reasoning_effort passato nel payload
```

### File coinvolti

| File | Ruolo |
|---|---|
| `admin/schemas/catalog.py` | Schema `ModelConfig` con `reasoning_effort: Literal["none","low","medium","high"]` |
| `agent/config_utils.py` | `apply_config_override()` — mappa config JSONB nel payload LiteLLM |
| `gateway/customer_router.py` | JOIN ruolo→modello→config, chiama `_apply_model_config()` |

### Logica reasoning_effort in `apply_config_override()`

```python
# reasoning_effort="none" → strip key (modelli che non lo supportano)
# reasoning_effort="low"/"medium"/"high" → passa al payload LiteLLM
# reasoning_effort non impostato → nessun override (usa default provider)
```

### Tenant override

I tenant possono sovrascrivere il modello per ruolo via `core.tenant_model_roles`. Il sistema usa la catena:

```
tenant_model_roles (override) → model_role_defaults (global) → llm_model_catalog (config)
```

---

## 5. Come cambiare un modello

### Via Admin Panel (consigliato)

1. **Catalog → Models**: verificare che il modello target esista nel catalogo
2. **Catalog → Model Roles → Grouped**: vista per workflow (chat, legis, lexorc, system)
3. Selezionare il ruolo da cambiare → assegnare nuovo modello
4. Se serve config specifica: **Catalog → Models → Edit** → sezione Config → `reasoningEffort`

### Via DB diretto

```sql
-- 1. Verificare catalogo
SELECT model_name, config FROM core.llm_model_catalog
WHERE is_active = true AND deleted_at IS NULL;

-- 2. Aggiornare ruolo
UPDATE core.model_role_defaults
SET model_name = 'nuovo-modello', updated_at = NOW()
WHERE role = 'nome_ruolo';

-- 3. Aggiornare config modello (reasoning_effort)
UPDATE core.llm_model_catalog
SET config = COALESCE(config, '{}')::jsonb || '{"reasoning_effort": "medium"}'::jsonb
WHERE model_name = 'nome-modello' AND deleted_at IS NULL;

-- 4. Restart container
docker restart lexe-core
```

### API (per automazione)

```bash
# PUT /api/v1/admin/catalog/model-roles
curl -X PUT https://api.stage.lexe.pro/api/v1/admin/catalog/model-roles \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"roles": [{"role": "chat_primary", "modelName": "gemini-3-flash-preview"}]}'
```

---

## 6. Benchmark summary

### Metodologia

- **6 run** (FAST×3, COMPLEX×3) con **11 modelli** su prompt legali IT reali
- **2 judge** (Claude Sonnet 4 + GPT-5-mini) con scoring 1-5 su 5 dimensioni
- **Peer-review**: ogni modello giudica gli altri → matrice self-bias
- Metriche: score medio, latenza, costo, stabilità cross-run

### Report dettagliati

| Report | Posizione |
|---|---|
| Benchmark consolidato | `lexe-docs/benchmark/` (se presente) |
| Piano master | `velvet-squishing-boole.md` (Phase 1-4) |
| Risultati raw | Sessioni benchmark nel transcript |

### Top 5 risultati

| # | Modello | Score Medio | Latenza | Note |
|---|---|---|---|---|
| 1 | Gemini 3 Flash | 4.57 | 2.7-10s | Domina FAST + COMPLEX |
| 2 | DeepSeek V3.2 | 4.81 (peer) | 5-12s | Best peer score, broken come judge |
| 3 | Claude Haiku 4.5 | 4.20 | 3.6s | Veloce, preciso |
| 4 | Qwen 3.5 Plus | 4.35 | 6-15s | Best judge calibration |
| 5 | GPT-5-mini | 4.75* | 4-8s | *Solo con reasoning=low |

---

*Documento generato da benchmark LLM Phase 1 — LEXE Platform 2026-02-23*
