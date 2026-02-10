# Deploy LEXE su Staging - Workflow Bulletproof

> Server: 91.99.229.111 | SSH Key: `~/.ssh/id_stage_new`
> URL: stage-chat.lexe.pro | API: api.stage.lexe.pro | Auth: auth.stage.lexe.pro

---

## REGOLA CRITICA N.1

```bash
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml <comando>
```

**SEMPRE** specificare i due file `-f`. Senza `-f`, Docker compose carica automaticamente
`docker-compose.override.yml` che punta a **PRODUZIONE** (api.lexe.pro, auth.lexe.pro).

| Comando | File caricati | Risultato |
|---------|---------------|-----------|
| `docker compose up` | `.yml` + `.override.yml` (auto) | **PROD** - SBAGLIATO su staging |
| `docker compose -f ... -f ...stage.yml up` | `.yml` + `.override.stage.yml` | **STAGE** - CORRETTO |

Il file `docker-compose.override.stage.yml` contiene:
- **Build args Vite** (`VITE_API_URL=https://api.stage.lexe.pro/api/v1`, ecc.)
- **Labels Traefik** per routing `*.stage.lexe.pro`
- **Logto stage** (`LEXE_LOGTO_ISSUER=https://auth.stage.lexe.pro/oidc`)
- **Networks** (`shared_public` per Traefik)

Senza questo file: niente labels Traefik, niente build-arg stage, niente routing.

---

## REGOLA CRITICA N.2

Le variabili `VITE_*` sono **build-time**, non runtime.

La sezione `environment:` nel compose **non ha effetto** su di esse.
Devono essere passate come `build: args:` (nel compose override) o come `--build-arg`.

**Verifica post-build:**
```bash
docker exec lexe-webchat sh -c 'cat /usr/share/nginx/html/assets/*.js' \
  | grep -oP 'auth\.\w+\.lexe\.pro|api\.\w+\.lexe\.pro' | sort -u
```
Atteso: `auth.stage.lexe.pro`, `api.stage.lexe.pro`
Errore: `auth.lexe.pro`, `api.lexe.pro` (= build senza override stage)

---

## Quick Reference

```bash
# SSH Alias (add to ~/.ssh/config)
Host lexe-stage
    HostName 91.99.229.111
    User root
    IdentityFile ~/.ssh/id_stage_new
```

---

## Compose files su staging

```
/opt/lexe-platform/lexe-infra/
  docker-compose.yml                   # Servizi principali
  docker-compose.override.yml          # AUTO-CARICATO = PROD (NON USARE su staging!)
  docker-compose.override.stage.yml    # Override stage (USARE QUESTO)
  docker-compose.override.prod.yml     # Override esplicito prod
  docker-compose.base.yml              # Base (non usato direttamente)
```

---

## Workflow Deploy Completo

### Step 1: Push da locale (Windows)

```bash
# Per ogni repo modificato:
cd lexe-webchat && git push origin stage
cd lexe-core && git push origin stage
```

### Step 2: SSH + Pull

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111
```

```bash
cd /opt/lexe-platform/lexe-webchat && git stash 2>/dev/null; git pull origin stage
cd /opt/lexe-platform/lexe-core && git stash 2>/dev/null; git pull origin stage
```

### Step 3: Build (con override stage!)

```bash
cd /opt/lexe-platform/lexe-infra

docker compose -f docker-compose.yml -f docker-compose.override.stage.yml \
  build lexe-webchat lexe-core --no-cache
```

### Step 4: Recreate

```bash
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml \
  up -d lexe-webchat lexe-core --force-recreate
```

### Step 5: Verifica (tutti i check!)

```bash
# 1. Container healthy
docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe

# 2. Core risponde internamente
curl -s -o /dev/null -w 'core internal: %{http_code}\n' http://localhost:8100/health
# Atteso: 200

# 3. API raggiungibile via Traefik
curl -sk -o /dev/null -w 'core traefik: %{http_code}\n' https://api.stage.lexe.pro/health
# Atteso: 200

# 4. Endpoint protetti richiedono JWT (non 404!)
curl -sk -o /dev/null -w 'customer auth: %{http_code}\n' https://api.stage.lexe.pro/api/v1/gateway/customer/conversations
# Atteso: 401 (richiede auth) -- se 404 = routing rotto!

# 5. Webchat carica
curl -sk -o /dev/null -w 'webchat: %{http_code}\n' https://stage-chat.lexe.pro
# Atteso: 200

# 6. Bundle ha URL stage (non prod!)
docker exec lexe-webchat sh -c 'cat /usr/share/nginx/html/assets/*.js' 2>/dev/null \
  | grep -oP 'auth\.\w+\.lexe\.pro|api\.\w+\.lexe\.pro' | sort -u
# Atteso: auth.stage.lexe.pro, api.stage.lexe.pro

# 7. Logto issuer corretto
docker exec lexe-core env | grep LOGTO_ISSUER
# Atteso: LEXE_LOGTO_ISSUER=https://auth.stage.lexe.pro/oidc

# 8. Logs puliti
docker logs lexe-core --tail 10
docker logs lexe-webchat --tail 5
```

---

## One-Liner da Windows (copia-incolla)

### Deploy webchat + core (completo)

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  cd /opt/lexe-platform/lexe-webchat && git stash 2>/dev/null; git pull origin stage &&
  cd /opt/lexe-platform/lexe-core && git stash 2>/dev/null; git pull origin stage &&
  cd /opt/lexe-platform/lexe-infra &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-webchat lexe-core --no-cache &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-webchat lexe-core --force-recreate &&
  sleep 15 &&
  docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe &&
  echo '--- Health checks ---' &&
  curl -s -o /dev/null -w 'core internal: %{http_code}\n' http://localhost:8100/health &&
  curl -sk -o /dev/null -w 'core traefik: %{http_code}\n' https://api.stage.lexe.pro/health &&
  curl -sk -o /dev/null -w 'customer auth: %{http_code}\n' https://api.stage.lexe.pro/api/v1/gateway/customer/conversations &&
  curl -sk -o /dev/null -w 'webchat: %{http_code}\n' https://stage-chat.lexe.pro
"
```

Output atteso:
```
core internal: 200
core traefik: 200
customer auth: 401
webchat: 200
```

### Deploy solo webchat

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  cd /opt/lexe-platform/lexe-webchat && git stash 2>/dev/null; git pull origin stage &&
  cd /opt/lexe-platform/lexe-infra &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-webchat --no-cache &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-webchat --force-recreate
"
```

### Deploy solo core

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  cd /opt/lexe-platform/lexe-core && git stash 2>/dev/null; git pull origin stage &&
  cd /opt/lexe-platform/lexe-infra &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-core --no-cache &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-core --force-recreate
"
```

### Deploy solo orchestrator

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  cd /opt/lexe-platform/lexe-orchestrator && git stash 2>/dev/null; git pull origin stage &&
  cd /opt/lexe-platform/lexe-infra &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-orchestrator --no-cache &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-orchestrator --force-recreate
"
```

### Deploy ALL services

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  for repo in lexe-core lexe-orchestrator lexe-memory lexe-webchat lexe-tools-it; do
    cd /opt/lexe-platform/\$repo && git stash 2>/dev/null; git pull origin stage
  done &&
  cd /opt/lexe-platform/lexe-infra &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build --no-cache &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d --force-recreate &&
  sleep 20 &&
  docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe
"
```

---

## Restart Senza Rebuild

Se serve solo riavviare un container (nessuna modifica codice):

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  cd /opt/lexe-platform/lexe-infra &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-core --force-recreate
"
```

---

## Trappole Conosciute

### 1. docker-compose.override.yml = PROD

Docker compose auto-carica `docker-compose.override.yml` se presente. Su staging questo file punta a PROD.
Non usare mai `docker compose` senza `-f` espliciti.

**Sintomi se usi il file sbagliato:**
- 404 su tutte le API call (niente labels Traefik)
- Webchat mostra form email invece di redirect Logto (VITE_OAUTH_ENABLED=false)
- Bundle JS contiene `auth.lexe.pro` invece di `auth.stage.lexe.pro`

### 2. VITE_* sono build-time

La sezione `environment:` del compose non funziona per variabili Vite.
Il Dockerfile ha `ARG VITE_OAUTH_ENABLED=false` come default.
Senza build-arg esplicito dal compose override, il bundle viene compilato con Logto disabilitato.

### 3. Logto viene ricreato come dependency

Quando si ricrea `lexe-webchat`, Docker puo' ricreare anche `lexe-logto` (dependency).
Normale - Logto riparte in pochi secondi, sessioni sopravvivono (state in DB).

### 4. Stash necessario se ci sono modifiche locali

Se qualcuno ha editato file sul server, `git pull` fallisce. `git stash` prima del pull.

### 5. Logto issuer PROD vs STAGE

| Env | Issuer |
|-----|--------|
| PROD | `https://auth.lexe.pro/oidc` |
| STAGE | `https://auth.stage.lexe.pro/oidc` |

Issuer sbagliato = 401 su tutte le chiamate autenticate.

### 6. health vs /api/v1/health

```
http://localhost:8100/health           → 200 (healthcheck interno)
https://api.stage.lexe.pro/health      → 200 (via Traefik, no auth)
https://api.stage.lexe.pro/api/v1/...  → 401 (richiede JWT)
```

Se `/api/v1/gateway/customer/stream` ritorna 404 invece di 401/405 = Traefik non sta routando a lexe-core.

---

## Rollback

```bash
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  cd /opt/lexe-platform/lexe-webchat && git log --oneline -5
"
# Trova il commit precedente, poi:
ssh -i ~/.ssh/id_stage_new root@91.99.229.111 "
  cd /opt/lexe-platform/lexe-webchat && git checkout COMMIT_HASH &&
  cd /opt/lexe-platform/lexe-infra &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-webchat --no-cache &&
  docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-webchat --force-recreate
"
```

---

## Tabella Diagnostica Rapida

| Sintomo | Causa probabile | Fix |
|---------|-----------------|-----|
| 404 su API | Build/up senza override stage | Rebuild con `-f ...stage.yml` |
| Form email invece di Logto | `VITE_OAUTH_ENABLED=false` nel bundle | Rebuild con override stage |
| Bundle ha `auth.lexe.pro` | Build senza override stage | Rebuild con override stage |
| 401 su tutto | Issuer PROD nel core | Recreate core con override stage |
| Logto riavviato | Dependency chain di docker compose | Normale, aspettare 30s |
| `git pull` fallisce | Modifiche locali sul server | `git stash` prima del pull |
| Container "Up" ma non raggiungibile | Network `shared_public` mancante | Override stage ha le labels |

---

*Creato 2026-02-10 - Dopo incident deploy senza override stage*
