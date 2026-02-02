# Logto Multitenancy Setup - Runbook Operativo

> Guida per configurare Logto con organization-based multitenancy
> Eseguito su STAGE: 2026-02-02
> Da eseguire su PROD: TBD

---

## Prerequisiti

- Accesso SSH al server
- Logto container running e healthy
- PostgreSQL container running

---

## Variabili per Ambiente

| Variabile | STAGE | PROD |
|-----------|-------|------|
| `SERVER_IP` | 91.99.229.111 | 49.12.85.92 |
| `SSH_KEY` | ~/.ssh/id_stage_new | ~/.ssh/hetzner_leo_key |
| `DB_USER` | lexe_logto | lexe_logto |
| `DB_NAME` | lexe_logto | lexe_logto |
| `DB_CONTAINER` | lexe-postgres | lexe-postgres |
| `API_RESOURCE_ID` | lexe-api-stage | lexe-api-prod |
| `API_INDICATOR` | https://api.stage.lexe.pro | https://api.lexe.pro |
| `ORG_ID` | lexe-default-stage | vij1h45hy26m (esistente) |
| `M2M_APP_ID` | lexe-mgmt-m2m-stage | lexe-mgmt-m2m |
| `LOGTO_ENDPOINT` | auth.stage.lexe.pro | auth.lexe.pro |

---

## Fase 1: Verificare Stato Attuale

### 1.1 Check Container Status

```bash
# STAGE
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 \
  "docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe"

# PROD
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92 \
  "docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe"
```

### 1.2 Verificare Risorse Esistenti

```bash
# API Resources
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -t -c \
   "SELECT id, indicator, name FROM resources;"'

# Organizations
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -t -c \
   "SELECT id, name FROM organizations;"'

# Applications (M2M)
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -t -c \
   "SELECT id, name, type FROM applications WHERE type = '\''MachineToMachine'\'';"'
```

---

## Fase 2: Creare API Resource

> **Nota:** Se l'API Resource esiste già (es. PROD ha già `https://api.lexe.pro`), SALTARE questo step.

```bash
# Sostituire variabili con valori ambiente
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -c \
   "INSERT INTO resources (tenant_id, id, indicator, name, is_default) \
    VALUES ('\''default'\'', '\''$API_RESOURCE_ID'\'', '\''$API_INDICATOR'\'', '\''LEXE API'\'', false);"'
```

### Esempio STAGE (eseguito)

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 \
  'docker exec lexe-postgres psql -U lexe_logto -d lexe_logto -c \
   "INSERT INTO resources (tenant_id, id, indicator, name, is_default) \
    VALUES ('\''default'\'', '\''lexe-api-stage'\'', '\''https://api.stage.lexe.pro'\'', '\''LEXE Stage API'\'', false);"'
```

### Verifica

```bash
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -t -c \
   "SELECT id, indicator FROM resources WHERE indicator LIKE '\''%lexe%'\'';"'
```

---

## Fase 3: Creare Organization

> **Nota:** Se l'Organization esiste già (es. PROD ha `vij1h45hy26m`), SALTARE questo step.

```bash
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -c \
   "INSERT INTO organizations (tenant_id, id, name, description) \
    VALUES ('\''default'\'', '\''$ORG_ID'\'', '\''LEXE Default'\'', '\''Default organization for LEXE users'\'');"'
```

### Esempio STAGE (eseguito)

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 \
  'docker exec lexe-postgres psql -U lexe_logto -d lexe_logto -c \
   "INSERT INTO organizations (tenant_id, id, name, description) \
    VALUES ('\''default'\'', '\''lexe-default-stage'\'', '\''LEXE Default'\'', '\''Default organization for LEXE users'\'');"'
```

### Verifica

```bash
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -t -c \
   "SELECT id, name FROM organizations WHERE name LIKE '\''%LEXE%'\'';"'
```

---

## Fase 4: Creare M2M Application

> **Nota:** Se M2M esiste già (es. PROD ha `lexe-mgmt-m2m`), SALTARE questo step o verificare che abbia i permessi corretti.

```bash
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -c \
   "INSERT INTO applications (tenant_id, id, name, secret, description, type, \
    oidc_client_metadata, custom_client_metadata, custom_data, is_third_party, created_at) \
    VALUES ('\''default'\'', '\''$M2M_APP_ID'\'', '\''LEXE Management API Client'\'', \
    md5(random()::text || clock_timestamp()::text), '\''M2M for lexe-core backend'\'', \
    '\''MachineToMachine'\'', '\''{}'\''::jsonb, '\''{}'\''::jsonb, '\''{}'\''::jsonb, false, NOW());"'
```

### Esempio STAGE (eseguito)

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 \
  'docker exec lexe-postgres psql -U lexe_logto -d lexe_logto -c \
   "INSERT INTO applications (tenant_id, id, name, secret, description, type, \
    oidc_client_metadata, custom_client_metadata, custom_data, is_third_party, created_at) \
    VALUES ('\''default'\'', '\''lexe-mgmt-m2m-stage'\'', '\''LEXE Management API Client'\'', \
    md5(random()::text || clock_timestamp()::text), '\''M2M for lexe-core backend'\'', \
    '\''MachineToMachine'\'', '\''{}'\''::jsonb, '\''{}'\''::jsonb, '\''{}'\''::jsonb, false, NOW());"'
```

### Recuperare Secret (IMPORTANTE!)

```bash
ssh -i $SSH_KEY root@$SERVER_IP \
  'docker exec $DB_CONTAINER psql -U $DB_USER -d $DB_NAME -t -c \
   "SELECT id, secret FROM applications WHERE id = '\''$M2M_APP_ID'\'';"'
```

> **SALVARE IL SECRET** - Non è recuperabile dopo! Salvare in file credenziali locale.

---

## Fase 5: Aggiornare Frontend DOMAIN_ORG_MAP

Modificare due file in `lexe-webchat/src/components/auth/`:

### LoginPage.tsx

```typescript
const DOMAIN_ORG_MAP: Record<string, string> = {
  // Stage domains
  "stage-chat.lexe.pro": "lexe-default-stage",
  // Production domain
  "ai.lexe.pro": "vij1h45hy26m",  // ← PROD org ID
  // Localhost
  localhost: "lexe-default-stage",
};
```

### LogtoStreamingProvider.tsx

```typescript
const DOMAIN_ORG_MAP: Record<string, string> = {
  // Stage domains
  "stage-chat.lexe.pro": "lexe-default-stage",
  // Production domain
  "ai.lexe.pro": "vij1h45hy26m",  // ← PROD org ID
  // Localhost
  localhost: "lexe-default-stage",
};
```

### Commit e Push

```bash
cd lexe-webchat
git add src/components/auth/LoginPage.tsx src/components/auth/LogtoStreamingProvider.tsx
git commit -m "feat(auth): update DOMAIN_ORG_MAP for multitenancy

- Map stage-chat.lexe.pro to lexe-default-stage
- Map ai.lexe.pro to vij1h45hy26m (PROD org)
- Update localhost to use stage org for dev

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>"
git push origin main
```

---

## Fase 6: Rebuild e Deploy Webchat

### STAGE

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 \
  "cd /opt/lexe-platform/lexe-webchat && git pull origin main && \
   cd /opt/lexe-platform/lexe-infra && \
   docker compose -f docker-compose.yml -f docker-compose.override.stage.yml \
   build lexe-webchat --no-cache && \
   docker compose -f docker-compose.yml -f docker-compose.override.stage.yml \
   up -d lexe-webchat --force-recreate"
```

### PROD

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92 \
  "cd /opt/lexe-platform/lexe-webchat && git pull origin main && \
   cd /opt/lexe-platform/lexe-infra && \
   docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
   build lexe-webchat --no-cache && \
   docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
   up -d lexe-webchat --force-recreate"
```

### Verifica Deploy

```bash
# Check container status
ssh -i $SSH_KEY root@$SERVER_IP \
  "docker ps --format '{{.Names}}\t{{.Status}}' | grep webchat"

# Check container creation time (deve essere recente)
ssh -i $SSH_KEY root@$SERVER_IP \
  "docker inspect lexe-webchat --format '{{.Created}}'"
```

---

## Fase 7: Creare Utente Test in Organization

### Opzione A: Via Logto Admin Console

1. Accedere a https://auth-admin.stage.lexe.pro (o auth-admin.lexe.pro per PROD)
2. Users → Create User
3. Organizations → Selezionare "LEXE Default"
4. Assegnare ruolo `admin` o `collaborator`

### Opzione B: Via Management API

```bash
# 1. Ottenere M2M token
TOKEN=$(curl -s -X POST "https://$LOGTO_ENDPOINT/oidc/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=$M2M_APP_ID" \
  -d "client_secret=$M2M_SECRET" \
  -d "resource=https://default.logto.app/api" \
  -d "scope=all" | jq -r ".access_token")

# 2. Creare utente
curl -X POST "https://$LOGTO_ENDPOINT/api/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "primaryEmail": "test@lexe.pro",
    "name": "Test User"
  }'

# 3. Assegnare a organization (dopo aver ottenuto user_id)
curl -X POST "https://$LOGTO_ENDPOINT/api/organizations/$ORG_ID/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "userIds": ["USER_ID_HERE"]
  }'
```

---

## Fase 8: Verificare Token JWT

### Test Login e Decode Token

1. Accedere a https://stage-chat.lexe.pro (o ai.lexe.pro)
2. Login con utente test
3. Aprire DevTools → Application → Session Storage
4. Copiare access_token
5. Decodificare:

```bash
echo "TOKEN_QUI" | cut -d. -f2 | base64 -d | jq .
```

### Claims Attesi

```json
{
  "aud": "https://api.stage.lexe.pro",
  "organization_id": "lexe-default-stage",
  "scope": "openid profile email read:data write:data",
  "sub": "user-uuid",
  "iss": "https://auth.stage.lexe.pro/oidc"
}
```

---

## Fase 9: Salvare Credenziali

Creare file locale (NON in repo):

```bash
# STAGE: C:\PROJECTS\LEXE-CREDENTIALS-STAGE.md
# PROD:  C:\PROJECTS\LEXE-CREDENTIALS-PROD.md
```

Contenuto:

```markdown
# LEXE [STAGE/PROD] Credentials

## M2M Application
- client_id: [ID]
- client_secret: [SECRET]

## API Resource
- indicator: https://api.[stage.]lexe.pro

## Organization
- id: [ORG_ID]
- name: LEXE Default

## Environment Variables
LEXE_LOGTO_ISSUER=https://auth.[stage.]lexe.pro/oidc
LEXE_LOGTO_RESOURCE=https://api.[stage.]lexe.pro
LEXE_LOGTO_MGMT_CLIENT_ID=[M2M_ID]
LEXE_LOGTO_MGMT_CLIENT_SECRET=[SECRET]
```

---

## Checklist Finale

### Per Ambiente

- [ ] API Resource creato/verificato
- [ ] Organization creata/verificata
- [ ] M2M Application creata/verificata
- [ ] Secret M2M salvato in file credenziali
- [ ] Frontend DOMAIN_ORG_MAP aggiornato
- [ ] Webchat rebuilt e deployed
- [ ] Utente test creato e assegnato a org
- [ ] Token JWT verificato con organization_id
- [ ] OIDC discovery funzionante

### Test Negativi (Post-Setup)

- [ ] Token senza org_id → rifiutato (401/403)
- [ ] Token con aud errata → rifiutato
- [ ] Accesso cross-tenant → nessun dato esposto

---

## Troubleshooting

### Container non healthy

```bash
docker logs lexe-logto --tail 100
docker logs lexe-webchat --tail 100
```

### OIDC non raggiungibile

```bash
curl -v https://auth.stage.lexe.pro/oidc/.well-known/openid-configuration
```

### Token non contiene organization_id

1. Verificare che utente sia assegnato a organization
2. Verificare DOMAIN_ORG_MAP nel frontend
3. Verificare che signIn() passi organization_id hint

### M2M token fallisce

1. Verificare client_id e client_secret
2. Verificare che M2M abbia scopes per Management API
3. Verificare resource = `https://default.logto.app/api`

---

## Stato Esecuzione

| Ambiente | Fase 1 | Fase 2 | Fase 3 | Fase 4 | Fase 5 | Fase 6 | Fase 7 | Fase 8 |
|----------|--------|--------|--------|--------|--------|--------|--------|--------|
| STAGE | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ⏳ |
| PROD | ✅ | ✅ già | ✅ già | ✅ già | ✅ | ✅ | ✅ | ⏳ |

### Test User Creato (2026-02-02)

| Ambiente | User ID | Email | Organization |
|----------|---------|-------|--------------|
| STAGE | w8oj4n7ledh9 | ftrani@gmail.com | lexe-default-stage |
| PROD | 8xsr0dycoz2o | ftrani@gmail.com | vij1h45hy26m |

> Login method: **OTP via email** (nessuna password)

### PROD - Risorse Esistenti (verificato 2026-02-02)

| Risorsa | ID | Valore |
|---------|-----|--------|
| API Resource | `ei85yfg2dlp9wupjtefg9` | `https://api.lexe.pro` |
| Organization | `vij1h45hy26m` | LEXE Default |
| M2M App | `lexe-mgmt-m2m` | LEXE Management API Client |

> **PROD non richiede creazione risorse** - solo deploy frontend e test.

---

*Runbook: Logto Multitenancy Setup*
*Versione: 1.0*
*Data: 2026-02-02*
