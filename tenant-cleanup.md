# Tenant Cleanup — Guida Operativa

Procedura per rimuovere completamente un tenant e i suoi utenti da tutti i sistemi LEXE.

## Sistemi coinvolti

| # | Sistema | Cosa contiene | Cascade? |
|---|---------|---------------|----------|
| 1 | **lexe-postgres** (`core` schema) | Tenant, users, conversations, contacts, tools, personas, models, feature flags, model roles | Si (FK cascade su `DELETE tenant`) |
| 2 | **Logto** (auth) | User account, Organization, Org membership | No — cleanup manuale via M2M API |
| 3 | **LiteLLM** | Virtual key + team | No — cleanup manuale (se provisioned) |
| 4 | **Valkey** (cache) | Rate limit keys, session cache | TTL auto-expire, ma cleanup immediato possibile |

---

## Step-by-step

### 1. Identificare il tenant

```bash
# Trova tenant per nome
docker exec lexe-postgres psql -U lexe -d lexe -c \
  "SELECT id, name, slug, logto_organization_id, litellm_team_id FROM core.tenants WHERE name ILIKE '%NOME%';"

# Trova utenti del tenant
docker exec lexe-postgres psql -U lexe -d lexe -c \
  "SELECT id, email, external_auth_id, role FROM core.users WHERE tenant_id = 'TENANT_ID';"
```

### 2. Audit risorse collegate (opzionale)

```bash
docker exec lexe-postgres psql -U lexe -d lexe -c "
SELECT 'users' as tbl, count(*) FROM core.users WHERE tenant_id = 'TENANT_ID'
UNION ALL SELECT 'conversations', count(*) FROM core.conversations WHERE tenant_id = 'TENANT_ID'
UNION ALL SELECT 'contacts', count(*) FROM core.contacts WHERE tenant_id = 'TENANT_ID'
UNION ALL SELECT 'tenant_llm_models', count(*) FROM core.tenant_llm_models WHERE tenant_id = 'TENANT_ID'
UNION ALL SELECT 'tools', count(*) FROM core.tools WHERE tenant_id = 'TENANT_ID'
UNION ALL SELECT 'responder_personas', count(*) FROM core.responder_personas WHERE tenant_id = 'TENANT_ID'
UNION ALL SELECT 'tenant_model_roles', count(*) FROM core.tenant_model_roles WHERE tenant_id = 'TENANT_ID'
UNION ALL SELECT 'feature_flags', count(*) FROM core.tenant_feature_flags WHERE tenant_id = 'TENANT_ID';
"
```

### 3. Logto — Rimuovere utenti e organizzazione

Per ogni utente con `external_auth_id` non vuoto:

```bash
docker exec lexe-core python3 -c "
import asyncio
from lexe_core.auth.logto_management_client import get_logto_management_client

async def main():
    c = get_logto_management_client()

    # Per ogni utente: rimuovi da org + elimina
    users = [
        # ('LOGTO_USER_ID', 'LOGTO_ORG_ID'),
    ]
    for user_id, org_id in users:
        try:
            await c.remove_user_from_organization(user_id, org_id)
            print(f'Removed {user_id} from org {org_id}')
        except Exception as e:
            print(f'Remove from org: {e}')
        try:
            await c.delete_user(user_id)
            print(f'User {user_id} deleted')
        except Exception as e:
            print(f'Delete user: {e}')

    # Elimina organizzazione
    org_id = 'LOGTO_ORG_ID'
    try:
        await c._request('DELETE', f'/organizations/{org_id}')
        print(f'Org {org_id} deleted')
    except Exception as e:
        print(f'Delete org: {e}')

asyncio.run(main())
"
```

> **Nota**: Se l'utente era condiviso con altri tenant (raro), NON eliminarlo da Logto — rimuoverlo solo dall'organizzazione.

### 4. LiteLLM — Rimuovere virtual key e team

Solo se `litellm_team_id` era popolato:

```bash
# Trova il team_id dal DB (step 1)
TEAM_ID="..."
LITELLM_KEY="$(grep LITELLM_MASTER_KEY /opt/lexe-platform/lexe-infra/.env.lexe | cut -d= -f2)"

# Elimina team (elimina anche le virtual keys associate)
curl -X DELETE "http://lexe-litellm:4000/team/delete" \
  -H "Authorization: Bearer $LITELLM_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"team_ids\": [\"$TEAM_ID\"]}"
```

### 5. Database — Eliminare tenant (cascade)

```bash
docker exec lexe-postgres psql -U lexe -d lexe -c \
  "DELETE FROM core.tenants WHERE id = 'TENANT_ID';"
```

Questo elimina a cascata:
- `core.users` (FK `tenant_id` CASCADE)
- `core.conversations` + `core.conversation_messages`
- `core.contacts` + `core.channel_accounts`
- `core.tools`
- `core.responder_personas`
- `core.tenant_llm_models`
- `core.tenant_model_roles`
- `core.tenant_feature_flags`
- `core.webhook_endpoints`

> **ATTENZIONE**: `DELETE FROM core.tenants` NON chiede conferma. Verificare l'ID due volte.

### 6. Valkey — Pulire cache (opzionale)

I rate limit keys scadono con TTL, ma per cleanup immediato:

```bash
docker exec lexe-valkey valkey-cli KEYS "lexe:*TENANT_ID*"
# Se ci sono risultati:
docker exec lexe-valkey valkey-cli DEL "lexe:limits:TENANT_ID" \
  "lexe:daily_convs:TENANT_ID:$(date +%Y-%m-%d)" \
  "lexe:daily_tokens:TENANT_ID:$(date +%Y-%m-%d)"
```

### 7. Verifica finale

```bash
# DB: deve ritornare 0 righe
docker exec lexe-postgres psql -U lexe -d lexe -c \
  "SELECT id FROM core.tenants WHERE id = 'TENANT_ID';"

# Logto: deve ritornare null
docker exec lexe-core python3 -c "
import asyncio
from lexe_core.auth.logto_management_client import get_logto_management_client
async def main():
    c = get_logto_management_client()
    org = await c.get_organization('LOGTO_ORG_ID')
    print('Org:', org)  # Deve essere None
asyncio.run(main())
"

# Valkey: deve ritornare empty list
docker exec lexe-valkey valkey-cli KEYS "lexe:*TENANT_ID*"
```

---

## Ordine di esecuzione

```
1. Identificare (DB query)
2. Logto: remove users from org → delete users → delete org
3. LiteLLM: delete team (se esiste)
4. DB: DELETE FROM core.tenants (cascade fa il resto)
5. Valkey: DEL keys (opzionale, TTL auto-expire)
6. Verifica finale
```

> **Importante**: Logto e LiteLLM vanno puliti PRIMA del DB, perche' dopo il DELETE i riferimenti (external_auth_id, logto_organization_id, litellm_team_id) non sono piu' disponibili.

---

*Ultimo aggiornamento: 2026-03-18*
