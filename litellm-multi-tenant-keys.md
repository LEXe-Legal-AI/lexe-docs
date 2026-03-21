# LiteLLM Multi-Tenant API Keys — Architettura

## Problema

Attualmente tutti i tenant LEXE condividono una singola API key OpenRouter.
Se un tenant fa burst di richieste, il rate limit colpisce tutti.
In prod serve isolamento per tenant: billing separato, rate limit indipendente, revoca granulare.

## Come LiteLLM supporta le virtual keys

LiteLLM Proxy ha un sistema di **Virtual Keys** integrato:

### 1. Virtual Keys (per-tenant)

```bash
# Crea una virtual key per un tenant
curl -X POST https://llm.lexe.pro/key/generate \
  -H "Authorization: Bearer LITELLM_MASTER_KEY" \
  -d '{
    "team_id": "tenant-uuid-here",
    "max_budget": 50.0,           # Budget mensile in USD
    "tpm_limit": 100000,          # Token per minute limit
    "rpm_limit": 60,              # Request per minute limit
    "models": ["lexe-fast", "lexe-primary", "lexe-complex"],
    "metadata": {"tenant_name": "Studio Rossi"}
  }'
```

Risposta:
```json
{
  "key": "sk-lexe-abc123...",
  "expires": "2026-04-21T00:00:00Z",
  "team_id": "tenant-uuid-here"
}
```

### 2. Per-Team Rate Limiting

Ogni virtual key ha rate limit indipendenti:

```yaml
# litellm config
general_settings:
  enable_team_rate_limiting: true

  # Default per tutti i team
  team_rpm_limit: 30        # 30 request/min per team
  team_tpm_limit: 50000     # 50k token/min per team

  # Budget tracking
  enable_budget_tracking: true
```

### 3. Integrazione con LEXE Tenant Resolver

Il `tenant_resolver.py` gia' risolve il tenant da Logto sub. Serve aggiungere:

```python
# In gateway/tenant_resolver.py
async def get_tenant_llm_key(tenant_id: str) -> str:
    """Get LiteLLM virtual key for this tenant."""
    # 1. Check Valkey cache
    cached = await valkey.get(f"lexe:llm_key:{tenant_id}")
    if cached:
        return cached

    # 2. Query DB
    key = await db.fetchval(
        "SELECT llm_api_key FROM core.tenants WHERE id = $1", tenant_id
    )

    # 3. If no key, create one via LiteLLM API
    if not key:
        key = await _create_tenant_llm_key(tenant_id)
        await db.execute(
            "UPDATE core.tenants SET llm_api_key = $1 WHERE id = $2",
            key, tenant_id
        )

    await valkey.set(f"lexe:llm_key:{tenant_id}", key, ex=3600)
    return key
```

### 4. Passaggio della key nel pipeline

```python
# In customer_router.py o pipeline entry point
tenant_llm_key = await get_tenant_llm_key(tenant_id)

# LiteLLM call con virtual key del tenant
headers = {"Authorization": f"Bearer {tenant_llm_key}"}
```

### 5. Monitoring per tenant

LiteLLM espone metriche per virtual key:

```
GET /spend/logs?api_key=sk-lexe-abc123
GET /team/info?team_id=tenant-uuid
```

Dashboard in Admin Panel:
- Spesa per tenant (giornaliera/mensile)
- Token consumati per modello
- Rate limit hit count
- Top queries per costo

## Piano di implementazione

| Fase | Cosa | Effort |
|------|------|--------|
| 1 | Aggiungere `llm_api_key` a `core.tenants` | Migration, 1h |
| 2 | Auto-provisioning key in tenant_resolver | Backend, 2h |
| 3 | Passare key nel pipeline LiteLLM call | Backend, 1h |
| 4 | Rate limit defaults nel plan preset (free=10rpm, pro=60rpm) | Config, 30min |
| 5 | Dashboard spesa per tenant in Admin Panel | Frontend, 4h |

## Costi OpenRouter

Con virtual keys, il billing di OpenRouter resta sulla singola API key master.
LiteLLM fa il tracking interno per tenant e applica budget limits.
Il billing reale per tenant si fa internamente via LiteLLM spend tracking.

Per billing reale separato (fattura diversa per tenant), servirebbe:
- OpenRouter non supporta sub-accounts
- Alternativa: Google AI diretto con service account per tenant (piu' complesso)
- Alternativa: budget tracking LiteLLM + fatturazione interna LEXE

## Rate Limits OpenRouter (riferimento)

| Piano | RPM | TPM | Note |
|-------|-----|-----|------|
| Free | 20 | 200k | Per test |
| Pay-as-you-go | 200 | 1M | Attuale LEXE |
| Enterprise | custom | custom | Negoziabile |

Con 10 tenant attivi e 30rpm ciascuno = 300rpm totale → serve piano Enterprise
o distribuzione su piu' API key (round-robin in LiteLLM).
