# RUNBOOK: Deploy Staging -> Production

| Field | Value |
|-------|-------|
| **Last Updated** | 2026-02-21 |
| **Owner** | LEXE Core Team |
| **Estimated Time** | 15-30 minutes |
| **Risk Level** | Medium |

## Prerequisites

- SSH access to both staging (91.99.229.111) and production (49.12.85.92)
- All tests passing on staging
- No active incidents

## Step 1: Verify Staging is Healthy

```bash
# SSH to staging
ssh -i ~/.ssh/id_stage_new root@91.99.229.111

# Check all containers are running
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep lexe

# Expected: all 8+ containers "Up" with healthy status
# lexe-core, lexe-webchat, lexe-postgres, lexe-max, lexe-valkey,
# lexe-logto, lexe-litellm, lexe-memory (optional: lexe-orchestrator)

# Quick health check
curl -s https://api.stage.lexe.pro/health | python3 -m json.tool

# Check recent logs for errors
docker logs lexe-core --tail 100 --since 1h 2>&1 | grep -i error | head -20
```

## Step 2: Merge stage -> main on GitHub

For each repository that has changes:

```bash
# On your local machine
cd C:/PROJECTS/lexe-genesis/<repo-name>

# Ensure stage is up to date
git checkout stage
git pull origin stage

# Merge to main
git checkout main
git pull origin main
git merge stage

# Push main
git push origin main
```

Repositories to check (in order of dependency):
1. `lexe-infra` (if docker-compose changes)
2. `lexe-core` (backend API)
3. `lexe-max` (if KB migrations)
4. `lexe-webchat` (frontend)
5. `lexe-tools-it` (if tool changes)
6. `lexe-memory` (if memory changes)
7. `lexe-docs` (documentation)

## Step 3: Deploy to Production

```bash
# SSH to production
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

# Navigate to infra
cd /opt/lexe-platform/lexe-infra

# Pull latest main
git checkout main
git pull origin main
```

### 3a: If only backend code changed (lexe-core)

```bash
cd /opt/lexe-platform/lexe-core
git checkout main
git pull origin main

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build lexe-core --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-core --force-recreate
```

### 3b: If frontend changed (lexe-webchat)

```bash
cd /opt/lexe-platform/lexe-webchat
git checkout main
git pull origin main

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build lexe-webchat --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-webchat --force-recreate
```

### 3c: If infra/docker-compose changed

```bash
cd /opt/lexe-platform/lexe-infra
git checkout main
git pull origin main

docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    pull
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d
```

### 3d: If database migrations needed (lexe-core or lexe-max)

```bash
# For lexe-core migrations (system DB)
cd /opt/lexe-platform/lexe-core
git checkout main
git pull origin main

# Check which migrations are pending
ls migrations/*.sql | sort

# Apply manually or via startup migration runner
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-core --force-recreate

# Verify migration applied
docker exec lexe-postgres psql -U lexe -d lexe -c \
    "SELECT * FROM core.schema_migrations ORDER BY id DESC LIMIT 5;"
```

```bash
# For lexe-max migrations (KB DB)
cd /opt/lexe-platform/lexe-max
git checkout main
git pull origin main

docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build lexe-max --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-max --force-recreate
```

## Step 4: Verify Production

```bash
# Health check
curl -s https://api.lexe.pro/health | python3 -m json.tool

# Check containers
docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe

# Check logs for startup errors
docker logs lexe-core --tail 50 2>&1 | head -30
docker logs lexe-webchat --tail 50 2>&1 | head -30

# Smoke test: visit https://ai.lexe.pro in browser
# Send a test legal query and verify response
```

## Step 5: Post-Deploy Monitoring (15 minutes)

```bash
# Watch logs for errors
docker logs lexe-core -f --since 1m 2>&1 | grep -E "(ERROR|WARN|500)"

# Check no circuit breakers tripped
docker logs lexe-core --tail 200 2>&1 | grep -i "circuit"

# Verify SSE streaming works
# Open https://ai.lexe.pro, send: "Cosa prevede l'art. 2043 c.c.?"
# Verify: tokens stream, tools execute, done event fires
```

## CRITICAL WARNINGS

1. **NEVER** run `docker compose down -v` -- this deletes all databases
2. **ALWAYS** use the production override file: `-f docker-compose.override.prod.yml`
3. **NEVER** confuse staging (91.99.229.111) with production (49.12.85.92)
4. **ALWAYS** deploy to staging first and test before production
5. **NEVER** push directly to main without testing on stage branch first
