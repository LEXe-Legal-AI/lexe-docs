---
owner: devops
audience: devops
source_of_truth: true
status: active
sensitivity: confidential
last_verified: 2026-03-19
---

# RUNBOOK-V2 -- Deploy & Rollback Operations

> Replaces: `RUNBOOK-deploy.md`, `RUNBOOK-rollback.md`, `DEPLOY-STAGE.md`
> Rule: where this doc and code diverge, **code wins**.

---

## Scope

**In scope:** Deploy procedures, rollback procedures, health checks, migration management, feature flag rollback for all 13 LEXE containers across staging and production environments.

**Not in scope:** Architecture details (see `ARCHITECTURE-V2.md`), pipeline internals (see `PIPELINE-REFERENCE.md`), security hardening (see `SECURITY-POSTURE.md`), KB ingestion procedures (see `KB-REFERENCE.md`), observability stack (see `OBSERVABILITY.md`).

---

## 1. Server Reference

| Env | IP | SSH Command | Infra Path |
|-----|-----|-------------|-----------|
| **Production** | 49.12.85.92 | `ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92` | `/opt/lexe-platform/lexe-infra` |
| **Staging** | 91.99.229.111 | `ssh -i ~/.ssh/id_stage_new root@91.99.229.111` | `/opt/lexe-platform/lexe-infra` |

**WARNING:** Production = 49.12.85.92, Staging = 91.99.229.111. Confusing them is a P0 incident.

---

## 2. Port Reference Table

| Container | Internal Port | Protocol | Role |
|-----------|--------------|----------|------|
| lexe-core | 8100 | HTTP | API Gateway, Auth, Agent pipeline |
| lexe-webchat | 3013 | HTTP (nginx) | User chat frontend |
| lexe-admin | 3014 | HTTP (nginx) | Admin panel SPA |
| lexe-orchestrator | 8102 | HTTP | ORCHIDEA pipeline (FF disabled) |
| lexe-memory | 8103 | HTTP | Memory L0-L4 |
| lexe-tools | 8021 | HTTP | Legal tools IT (Normattiva, EUR-Lex, InfoLex) |
| lexe-postgres | 5435 | PostgreSQL | System DB (core, memory, Logto, LiteLLM) |
| lexe-max | 5436 | PostgreSQL | KB Legal DB (normativa, massime) |
| lexe-valkey | 6381 | Redis protocol | Cache, session, limits counters |
| lexe-litellm | 4001 | HTTP | LLM Gateway (OpenRouter) |
| lexe-logto | 3304 (app) / 3305 (admin) | HTTP | Auth CIAM |
| lexe-temporal | 7234 | gRPC | Workflow orchestration |
| lexe-temporal-ui | 8180 | HTTP | Temporal dashboard |

**Networks:** `shared_public` (Traefik ingress, shared with LEO), `lexe_internal` (backend-only).

---

## 3. Deploy Commands

### Critical Rule

```
ALWAYS specify both -f flags. Without them, Docker auto-loads
docker-compose.override.yml which points to PRODUCTION.
```

| Environment | Command |
|-------------|---------|
| **Staging** | `docker compose -f docker-compose.yml -f docker-compose.override.stage.yml <action>` |
| **Production** | `docker compose -f docker-compose.yml -f docker-compose.override.prod.yml <action>` |

The override files provide:
- Traefik labels for routing (`*.stage.lexe.pro` or `*.lexe.pro`)
- Build args for Vite (`VITE_API_URL`, `VITE_OAUTH_ENABLED`, etc.)
- Logto issuer URL (`LEXE_LOGTO_ISSUER`)
- Network attachments (`shared_public`)
- Environment-specific secrets (HMAC keys, Langfuse keys)

**Without override = no Traefik labels + wrong env vars = 404 on all routes.**

### Git Workflow

```
stage branch -> push origin stage -> deploy staging -> test -> merge main -> deploy prod
```

Never push directly to main without testing on staging first.

---

### 3a. lexe-core Changes (API Gateway)

```bash
cd /opt/lexe-platform/lexe-core
git checkout <branch> && git pull origin <branch>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build lexe-core --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d lexe-core --force-recreate
```

**Impact:** SSE streaming interrupted for in-flight requests (~5s downtime).
**Migrations:** lexe-core runs migrations on startup automatically. Check post-deploy:

```bash
docker exec lexe-postgres psql -U lexe -d lexe -c \
    "SELECT * FROM core.schema_migrations ORDER BY id DESC LIMIT 5;"
```

### 3b. lexe-webchat Changes (Frontend)

```bash
cd /opt/lexe-platform/lexe-webchat
git checkout <branch> && git pull origin <branch>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build lexe-webchat --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d lexe-webchat --force-recreate
```

**Impact:** Users see loading spinner briefly (~3s).
**Verify VITE_* build args** (build-time only, NOT runtime):

```bash
docker exec lexe-webchat sh -c 'cat /usr/share/nginx/html/assets/*.js' \
  | grep -oP 'auth\.\w+\.lexe\.pro|api\.\w+\.lexe\.pro' | sort -u
# Expected staging: auth.stage.lexe.pro, api.stage.lexe.pro
# Expected prod: auth.lexe.pro, api.lexe.pro
```

### 3c. lexe-max Changes (KB Legal)

```bash
cd /opt/lexe-platform/lexe-max
git checkout <branch> && git pull origin <branch>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build lexe-max --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d lexe-max --force-recreate
```

**Impact:** Legal search degraded during rebuild (~10-30s depending on migration).
**Verify KB:**

```bash
docker exec lexe-max psql -U lexe_kb -d lexe_kb -c "SELECT count(*) FROM kb.normativa;"
docker exec lexe-max psql -U lexe_kb -d lexe_kb -c "SELECT count(*) FROM kb.massime;"
```

### 3d. lexe-infra / Config Changes

```bash
cd /opt/lexe-platform/lexe-infra
git checkout <branch> && git pull origin <branch>

docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml pull
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml up -d
```

**Impact:** All containers may restart if compose definitions changed.

### 3e. lexe-admin (Admin Panel)

```bash
cd /opt/lexe-platform/lexe-admin
git checkout <branch> && git pull origin <branch>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build lexe-admin --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d lexe-admin --force-recreate
```

**Impact:** Admin panel unavailable for ~3s. No backend impact.
**Verify:** `curl -s -o /dev/null -w "%{http_code}" https://admin.lexe.pro` (prod) or `https://admin.stage.lexe.pro` (stage).

### 3f. lexe-tools-it (Legal Tools)

```bash
cd /opt/lexe-platform/lexe-tools-it
git checkout <branch> && git pull origin <branch>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build lexe-tools --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d lexe-tools --force-recreate
```

**Impact:** Tool calls (Normattiva, EUR-Lex, InfoLex) fail during restart (~5s). Pipeline degrades gracefully (skip_on_failure).

### 3g. lexe-memory (Memory System)

```bash
cd /opt/lexe-platform/lexe-memory
git checkout <branch> && git pull origin <branch>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build lexe-memory --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d lexe-memory --force-recreate
```

**Impact:** Memory retrieval/storage fails gracefully. System continues without memory context.

### Full Stack Deploy

```bash
# Pull all repos
for repo in lexe-core lexe-webchat lexe-admin lexe-tools-it lexe-memory lexe-max; do
    cd /opt/lexe-platform/$repo && git stash 2>/dev/null; git pull origin <branch>
done

cd /opt/lexe-platform/lexe-infra
git checkout <branch> && git pull origin <branch>

docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d --force-recreate
```

---

## 4. Health Check Verification

### Quick Health (run after every deploy)

```bash
# Container status
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep lexe

# Core health (internal)
curl -s http://localhost:8100/health | python3 -m json.tool

# Core health (via Traefik)
curl -sk https://api.<domain>/health | python3 -m json.tool

# Auth OIDC discovery
curl -sk https://auth.<domain>/oidc/.well-known/openid-configuration | python3 -m json.tool

# Webchat responds
curl -sk -o /dev/null -w "%{http_code}" https://<chat-domain>

# Admin panel responds
curl -sk -o /dev/null -w "%{http_code}" https://admin.<domain>

# LLM Gateway
curl -sk https://llm.<domain>/health

# Protected endpoint returns 401 (NOT 404)
curl -sk -o /dev/null -w "%{http_code}" https://api.<domain>/api/v1/gateway/customer/conversations
# Expected: 401 (requires auth). If 404 = Traefik routing broken.
```

### Per-Container Health Endpoints

| Container | Health Check | Expected |
|-----------|-------------|----------|
| lexe-core | `curl http://localhost:8100/health` | `{"status":"ok"}` with DB/Valkey/LiteLLM status |
| lexe-webchat | `curl http://localhost:3013/` | HTTP 200 (nginx serves SPA) |
| lexe-admin | `curl http://localhost:3014/` | HTTP 200 (nginx serves SPA) |
| lexe-tools | `curl http://localhost:8021/health` | `{"status":"ok"}` |
| lexe-memory | `curl http://localhost:8103/health` | `{"status":"ok"}` |
| lexe-litellm | `curl http://localhost:4001/health` | `{"status":"ok"}` |
| lexe-logto | `curl http://localhost:3304/api/status` | HTTP 200 |
| lexe-postgres | `docker exec lexe-postgres pg_isready -U lexe` | "accepting connections" |
| lexe-max | `docker exec lexe-max pg_isready -U lexe_kb` | "accepting connections" |
| lexe-valkey | `docker exec lexe-valkey valkey-cli ping` | `PONG` |
| lexe-temporal | `docker exec lexe-temporal tctl cluster health` | "SERVING" |
| lexe-temporal-ui | `curl http://localhost:8180/` | HTTP 200 |
| lexe-orchestrator | `curl http://localhost:8102/health` | `{"status":"ok"}` (FF disabled) |

### Post-Deploy Smoke Test

```bash
# Watch logs for errors (15 min)
docker logs lexe-core -f --since 1m 2>&1 | grep -E "(ERROR|WARN|500)"

# Verify SSE streaming works:
# Open the chat UI, send: "Cosa prevede l'art. 2043 c.c.?"
# Verify: tokens stream, tools execute, done event fires with trace hash

# Check Logto issuer is correct for environment
docker exec lexe-core env | grep LOGTO_ISSUER
# Staging: https://auth.stage.lexe.pro/oidc
# Prod: https://auth.lexe.pro/oidc
```

---

## 5. Rollback Procedures

### General Strategy

LEXE uses git-based rollback: revert to the previous commit and rebuild the container.

```bash
cd /opt/lexe-platform/<repo>
git log --oneline -5          # Find last known good commit
git checkout <good-commit>
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build <service> --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d <service> --force-recreate
```

### Per-Container Rollback

| Container | Repo | Estimated Time | Impact During Rollback |
|-----------|------|---------------|----------------------|
| lexe-core | lexe-core | 3-5 min | SSE interrupted for in-flight requests |
| lexe-webchat | lexe-webchat | 3-5 min | Users see loading spinner briefly |
| lexe-admin | lexe-admin | 2-3 min | Admin panel unavailable |
| lexe-max | lexe-max | 5-15 min | Legal search degraded |
| lexe-tools | lexe-tools-it | 3-5 min | Tool calls fail, pipeline degrades |
| lexe-memory | lexe-memory | 2-3 min | Memory retrieval fails gracefully |
| lexe-litellm | -- (config) | 2-3 min | All LLM calls fail (~10s) |
| lexe-logto | -- (image) | 2-3 min | Users cannot log in (~15s) |
| lexe-postgres | -- | N/A | DO NOT restart without DBA |
| lexe-max (DB) | -- | N/A | DO NOT restart without DBA |
| lexe-valkey | -- | 1-2 min | Cache miss spike, limits counters reset |
| lexe-temporal | -- (image) | 2-3 min | Workflow execution paused |
| lexe-temporal-ui | -- (image) | 1 min | Dashboard unavailable |

### Stateless Services (restart only)

For lexe-litellm, lexe-logto, lexe-temporal, lexe-temporal-ui:

```bash
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    restart <service>
```

### Full Stack Rollback (Emergency)

```bash
# Revert all application repos
for repo in lexe-core lexe-webchat lexe-admin lexe-tools-it lexe-memory lexe-max; do
    cd /opt/lexe-platform/$repo
    echo "=== $repo ==="
    git log --oneline -5
    # git checkout <known-good-hash>
done

# Rebuild and restart everything
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    build --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d --force-recreate

# Verify
docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe
curl -s https://api.<domain>/health
```

---

## 6. Migration Rollback

### Pre-Rollback: Always Backup First

```bash
# System DB backup
docker exec lexe-postgres pg_dump -U lexe -d lexe -Fc > /tmp/lexe_backup_$(date +%Y%m%d_%H%M).dump

# KB DB backup
docker exec lexe-max pg_dump -U lexe_kb -d lexe_kb -Fc > /tmp/lexe_kb_backup_$(date +%Y%m%d_%H%M).dump
```

### Identify Destructive Migrations

```bash
# Check for destructive statements in pending migrations
grep -n -E "DROP|ALTER.*DROP|TRUNCATE|DELETE FROM" /opt/lexe-platform/lexe-core/migrations/*.sql
grep -n -E "DROP|ALTER.*DROP|TRUNCATE|DELETE FROM" /opt/lexe-platform/lexe-max/migrations/*.sql
```

If destructive operations found: **stop and assess**. Migration rollback may require restoring from backup.

### Core Migrations (lexe-core)

- Location: `/opt/lexe-platform/lexe-core/migrations/`
- Tracking table: `core.schema_migrations`
- Current: 035+ migrations
- Applied automatically on lexe-core startup

```bash
# Check applied migrations
docker exec lexe-postgres psql -U lexe -d lexe -c \
    "SELECT * FROM core.schema_migrations ORDER BY id DESC LIMIT 10;"

# Manual rollback (write inverse SQL)
docker exec -i lexe-postgres psql -U lexe -d lexe < /path/to/rollback.sql
```

### KB Migrations (lexe-max)

- Location: `/opt/lexe-platform/lexe-max/migrations/`
- 26 migration files (numbered 001-080, non-contiguous)
- Applied on lexe-max startup

```bash
# Check KB migration state
docker exec lexe-max psql -U lexe_kb -d lexe_kb -c \
    "SELECT * FROM kb.schema_migrations ORDER BY id DESC LIMIT 10;"
```

---

## 7. Feature Flag Rollback

Feature flags allow rapid rollback without container rebuild. Edit the override file and recreate lexe-core.

### Feature Flag Reference

| Flag | Default | Description |
|------|---------|-------------|
| `ff_orchestration_v2` | `true` | Orchestration v2 (3-layer architecture) |
| `ff_multi_agent_research` | `false` | Multi-agent research "La Bomba" (3-wave parallel) |
| `ff_consent_required` | `false` | GDPR consent gate before chat |
| `ff_memory_v2` | `true` | Memory L0-L4 system |
| `ff_lexorc_enabled` | `false` | LEXORC pipeline (disabled) |
| `ff_legis_agent` | `true` | LEGIS agent pipeline |
| `ff_langfuse_spans` | `true` | Langfuse LLM tracing |
| `ff_event_persistence` | `true` | EventSink conversation events |
| `ff_strict_tenant_no_fallback` | `true` | Strict tenant resolution (no fallback) |

### Quick FF Toggle

```bash
cd /opt/lexe-platform/lexe-infra

# Edit the override file (use vi/nano, NOT sed -i which breaks Docker bind mounts)
vi docker-compose.override.<env>.yml
# Change: LEXE_FF_<FLAG_NAME>=false

# Recreate lexe-core (no rebuild needed)
docker compose -f docker-compose.yml -f docker-compose.override.<env>.yml \
    up -d lexe-core --force-recreate

# Verify
docker exec lexe-core env | grep FF_
```

**WARNING:** `sed -i` creates a new inode and breaks Docker bind mounts. Always use an interactive editor, then `docker restart lexe-core` if bind-mounted, or `--force-recreate` if not.

### Per-Tenant FF Override

Feature flags can be overridden per-tenant via the admin API:

```bash
curl -X PUT https://api.<domain>/api/v1/admin/tenants/<tenant-id>/feature-flags \
    -H "Authorization: Bearer <admin-token>" \
    -H "Content-Type: application/json" \
    -d '{"ff_multi_agent_research": true}'
```

---

## 8. Known Traps and Gotchas

| Trap | Description | Prevention |
|------|-------------|------------|
| Missing override file | `docker compose up` without `-f` loads `docker-compose.override.yml` = PROD config | Always use explicit `-f` flags |
| VITE_* at runtime | `environment:` section does NOT affect Vite variables | Must be `build: args:` in override |
| Logto dependency chain | Recreating webchat may also recreate logto | Normal. Logto restarts in ~15s, sessions survive (state in DB) |
| Local edits on server | `git pull` fails if someone edited files on server | `git stash` before `git pull` |
| sed -i breaks mounts | `sed -i` creates new inode, Docker bind mount breaks | Use interactive editor + `docker restart` |
| lexe-core port 8100 | Internal port is 8100, NOT 8000 as some old docs say | Always use 8100 |
| DB credentials lexe-max | Both envs use `lexe_kb`/`lexe_kb` (password: `lexe_kb_secret`) | Old docs said `lexe_max`/`lexe_max` for prod -- NOT true |

---

## 9. Post-Deploy / Post-Rollback Checklist

- [ ] All 13 containers running: `docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe`
- [ ] Health endpoint returns 200: `curl https://api.<domain>/health`
- [ ] Frontend loads: `curl https://<chat-domain>`
- [ ] Admin panel loads: `curl https://admin.<domain>`
- [ ] Auth works: `curl https://auth.<domain>/oidc/.well-known/openid-configuration`
- [ ] Protected endpoint returns 401 (not 404): `curl https://api.<domain>/api/v1/gateway/customer/conversations`
- [ ] SSE streaming works: send a test legal query and verify response
- [ ] Logto issuer matches environment: `docker exec lexe-core env | grep LOGTO_ISSUER`
- [ ] No ERROR in recent logs: `docker logs lexe-core --tail 100 2>&1 | grep ERROR`
- [ ] Webchat bundle has correct domain: verify VITE_* URLs in JS bundle
- [ ] Migrations applied (if applicable): check `core.schema_migrations`
- [ ] Notify team about deploy/rollback and any root cause

---

## 10. Critical Warnings

1. **NEVER** run `docker compose down -v` -- this permanently destroys all databases.
2. **ALWAYS** use the environment-specific override file (`-f docker-compose.override.<env>.yml`).
3. **NEVER** confuse staging (91.99.229.111) with production (49.12.85.92).
4. **ALWAYS** deploy to staging first, test, then merge to main and deploy prod.
5. **NEVER** push directly to main without testing on stage branch first.
6. **NEVER** restart lexe-postgres or lexe-max without DBA review.
7. **ALWAYS** backup databases before applying migrations with DROP/ALTER statements.

---

*Last updated: 2026-03-19*
