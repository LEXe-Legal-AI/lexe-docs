# RUNBOOK: Rollback Procedures

| Field | Value |
|-------|-------|
| **Last Updated** | 2026-02-21 |
| **Owner** | LEXE Core Team |
| **Estimated Time** | 5-15 minutes per component |
| **Risk Level** | Low-Medium |

## General Rollback Strategy

LEXE uses a **git-based rollback**: revert to the previous commit and rebuild the container.

```bash
# SSH to production
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

# General pattern:
cd /opt/lexe-platform/<repo>
git log --oneline -5          # Find the commit to revert to
git checkout <previous-commit>
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build <service> --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d <service> --force-recreate
```

---

## Component-Specific Rollback

### 1. lexe-core (API Gateway)

**Estimated time:** 3-5 minutes
**Impact during rollback:** SSE streaming interrupted for in-flight requests

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

cd /opt/lexe-platform/lexe-core
git log --oneline -5
# Identify the last known good commit
git checkout <good-commit-hash>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build lexe-core --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-core --force-recreate

# Verify
docker logs lexe-core --tail 20
curl -s https://api.lexe.pro/health
```

### 2. lexe-webchat (Frontend)

**Estimated time:** 3-5 minutes
**Impact during rollback:** Users see loading spinner briefly

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

cd /opt/lexe-platform/lexe-webchat
git log --oneline -5
git checkout <good-commit-hash>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build lexe-webchat --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-webchat --force-recreate

# Verify
curl -s -o /dev/null -w "%{http_code}" https://ai.lexe.pro
```

### 3. lexe-max (KB Database)

**Estimated time:** 5-15 minutes (depends on migration complexity)
**Impact during rollback:** Legal search degraded during rebuild

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

cd /opt/lexe-platform/lexe-max
git log --oneline -5
git checkout <good-commit-hash>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build lexe-max --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-max --force-recreate

# Verify KB is accessible
docker exec lexe-max psql -U lexe_kb -d lexe_kb -c "SELECT count(*) FROM kb.normativa;"
```

**WARNING:** If a DB migration was applied that is destructive (DROP TABLE, ALTER COLUMN), the rollback may require a database restore from backup. Contact the DBA.

### 4. lexe-litellm (LLM Gateway)

**Estimated time:** 2-3 minutes
**Impact during rollback:** All LLM calls fail during restart (~10s downtime)

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    restart lexe-litellm

# Verify
curl -s https://llm.lexe.pro/health
```

### 5. lexe-logto (Authentication)

**Estimated time:** 2-3 minutes
**Impact during rollback:** Users cannot log in during restart (~15s)

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    restart lexe-logto

# Verify
curl -s https://auth.lexe.pro/oidc/.well-known/openid-configuration | python3 -m json.tool
```

### 6. lexe-memory (Memory System)

**Estimated time:** 2-3 minutes
**Impact during rollback:** Memory retrieval/storage fails gracefully (system continues without memory)

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

cd /opt/lexe-platform/lexe-memory
git log --oneline -5
git checkout <good-commit-hash>

cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build lexe-memory --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-memory --force-recreate
```

---

## Emergency: Feature Flag Rollback

For pipeline-level issues that do not require container rebuild:

### Disable LEGIS Pipeline
```bash
# Edit the environment in the override file or container env
docker exec lexe-core env | grep FF_LEGIS

# Restart with flag disabled
cd /opt/lexe-platform/lexe-infra
# Edit docker-compose.override.prod.yml: set LEXE_FF_LEGIS_AGENT=false
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-core --force-recreate
```

### Disable LEXORC Pipeline
```bash
# Same approach: set LEXE_FF_LEXORC_ENABLED=false
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d lexe-core --force-recreate
```

---

## Emergency: Full Stack Rollback

If multiple components are broken and you need to revert everything:

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92

# Revert all repos to a known good state
for repo in lexe-core lexe-webchat lexe-max lexe-tools-it lexe-memory; do
    cd /opt/lexe-platform/$repo
    git log --oneline -3
    echo "Revert $repo? (enter commit hash or skip)"
    # git checkout <hash>
done

# Rebuild and restart everything
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    build --no-cache
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml \
    up -d --force-recreate

# Verify
docker ps --format 'table {{.Names}}\t{{.Status}}' | grep lexe
curl -s https://api.lexe.pro/health
curl -s -o /dev/null -w "%{http_code}" https://ai.lexe.pro
```

---

## Post-Rollback Checklist

- [ ] All containers are running (`docker ps | grep lexe`)
- [ ] Health endpoint returns 200 (`curl https://api.lexe.pro/health`)
- [ ] Frontend loads (`curl https://ai.lexe.pro`)
- [ ] Auth works (`curl https://auth.lexe.pro/oidc/.well-known/openid-configuration`)
- [ ] Send a test legal query and verify SSE streaming works
- [ ] Check logs for errors: `docker logs lexe-core --tail 100 2>&1 | grep ERROR`
- [ ] Notify the team about the rollback and root cause

## CRITICAL WARNINGS

1. **NEVER** run `docker compose down -v` -- this destroys all databases permanently
2. **ALWAYS** use `-f docker-compose.override.prod.yml` on production
3. **Database migrations** cannot be rolled back by git checkout alone -- they may need manual SQL
4. **Staging IP:** 91.99.229.111 | **Production IP:** 49.12.85.92 -- do NOT confuse them
