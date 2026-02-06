# LEXE STAGE Deployment Procedure

> Server: 91.99.229.111 (STAGE)
> SSH Key: `~/.ssh/id_stage_new`

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

## CRITICAL: Always Use Stage Override!

**NEVER** use just `docker compose` commands. **ALWAYS** include both files:

```bash
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml <command>
```

The stage override contains:
- STAGE Logto URLs (`auth.stage.lexe.pro`)
- STAGE API URLs (`api.stage.lexe.pro`)
- Traefik labels with CORS middleware
- Build args for webchat

---

## Deploy Procedures

### 1. Deploy lexe-webchat (Frontend)

```bash
# SSH to STAGE
ssh lexe-stage

# Pull latest code
cd /opt/lexe-platform/lexe-webchat
git pull origin stage

# Build and deploy WITH STAGE OVERRIDE
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-webchat --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-webchat --force-recreate

# Verify
docker ps | grep lexe-webchat
```

**Important:** The build step bakes in Vite env vars:
- `VITE_OAUTH_ENABLED=true`
- `VITE_LOGTO_ENDPOINT=https://auth.stage.lexe.pro`
- `VITE_API_URL=https://api.stage.lexe.pro/api/v1`

### 2. Deploy lexe-core (Backend API)

```bash
ssh lexe-stage

cd /opt/lexe-platform/lexe-core
git pull origin stage

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-core --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-core --force-recreate

# Verify
docker ps | grep lexe-core
docker logs lexe-core --tail 20
```

### 3. Deploy lexe-orchestrator

```bash
ssh lexe-stage

cd /opt/lexe-platform/lexe-orchestrator
git pull origin stage

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build lexe-orchestrator --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d lexe-orchestrator --force-recreate
```

### 4. Deploy All Services

```bash
ssh lexe-stage
cd /opt/lexe-platform/lexe-infra

# Pull all repos
for repo in lexe-core lexe-orchestrator lexe-memory lexe-webchat lexe-tools-it; do
  cd /opt/lexe-platform/$repo && git pull origin stage
done

# Rebuild and restart all
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d --force-recreate

# Verify all healthy
docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe
```

---

## Restart Without Rebuild

If you just need to restart a container (no code changes):

```bash
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d <service> --force-recreate
```

---

## Verification Checklist

After deploy, verify:

1. **Container Health**
   ```bash
   docker ps | grep lexe
   ```

2. **CORS Middleware** (for lexe-core)
   ```bash
   docker exec shared-traefik wget -qO- http://localhost:8080/api/http/routers | \
     jq '.[] | select(.name | test("lexe-api-stage")) | {name, middlewares}'
   ```
   Should show: `"middlewares": ["lexe-cors@file"]`

3. **Logto Issuer** (for lexe-core)
   ```bash
   docker exec lexe-core env | grep LOGTO_ISSUER
   ```
   Should show: `https://auth.stage.lexe.pro/oidc`

4. **Test API**
   ```bash
   curl -I https://api.stage.lexe.pro/api/v1/health
   ```

5. **Test Chat**
   - Open https://stage-chat.lexe.pro
   - Do hard refresh (Ctrl+Shift+R)
   - Login with Logto
   - Send a test message

---

## Troubleshooting

### CORS Errors
Check Traefik router has CORS middleware:
```bash
docker exec shared-traefik wget -qO- http://localhost:8080/api/http/routers | \
  jq '.[] | select(.name == "lexe-api-stage@docker")'
```

### JWT Invalid Issuer
Check lexe-core environment:
```bash
docker exec lexe-core env | grep LOGTO
```

### Logto Disabled in Frontend
Check if built with correct args - rebuild with stage override.

### Old JavaScript Bundle
Hard refresh browser (Ctrl+Shift+R) or clear cache.

---

## Files Reference

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Base configuration |
| `docker-compose.override.stage.yml` | STAGE-specific overrides |
| `docker-compose.override.prod.yml` | PRODUCTION overrides |

---

*Last updated: 2026-02-06*
