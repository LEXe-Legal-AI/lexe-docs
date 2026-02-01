# LEXE Platform - Status

> Stato attuale della piattaforma LEXE Legal AI

---

## Infrastructure Status (2026-02-01)

| Container | Status | Note |
|-----------|--------|------|
| lexe-core | Healthy | Gateway + Auth OIDC |
| lexe-orchestrator | Healthy | ORCHIDEA Pipeline |
| lexe-memory | Healthy | L1-L3 Memory |
| lexe-webchat | Healthy | Frontend chat |
| lexe-logto | Healthy | CIAM Auth |
| lexe-litellm | Healthy | LLM Gateway |
| lexe-postgres | Healthy | DB + pgvector |
| lexe-valkey | Healthy | Cache L1 |

**Total:** 8 containers healthy

---

## Services URLs

| Service | URL | Status |
|---------|-----|--------|
| Webchat | https://ai.lexe.pro | OK |
| API | https://api.lexe.pro | OK |
| Auth OIDC | https://auth.lexe.pro | OK |
| LLM Gateway | https://llm.lexe.pro | OK |

---

## Traefik Routers

| Router | Domain | Status |
|--------|--------|--------|
| lexe-api | api.lexe.pro | Enabled |
| lexe-webchat | ai.lexe.pro | Enabled |
| lexe-logto | auth.lexe.pro | Enabled |
| lexe-logto-admin | 127.0.0.1:3304 | Enabled (SSH tunnel) |
| lexe-litellm | llm.lexe.pro | Enabled |

---

## Completed Work

### Phase 0-4 (Separazione da LEO)

- [x] Backup completo
- [x] DNS configurato
- [x] LEO Logto migrato a logto.leo.itconsultingsrl.com
- [x] LEXE Logto creato su auth.lexe.pro
- [x] LEXE LiteLLM creato su llm.lexe.pro
- [x] Containers LEXE separati e healthy

### LEO References Cleanup

- [x] whatsapp.py → P3/REMOVAL (requires lexe-channels)
- [x] aria_router.py → P3/REMOVAL (experimental bypass)
- [x] voxtral.py → Referer updated to lexe.pro
- [x] policy.py → Default to lexe-core:8000
- [x] telemetry.py → Default to lexe-core:8000
- [x] image_health.py → LEXE container names
- [x] benchmark.py → Referer updated to lexe.pro
- [x] welcome_email_service.py → Brand LEXE, URL ai.lexe.pro
- [x] email_provider.py → From noreply@lexe.pro
- [x] o2o_groups.py → P3/REMOVAL (requires lexe-channels)

---

## In Progress (POSTSTART.md)

### FASE 1: Verifica Base
- [ ] Health check completo
- [ ] Test login flow
- [ ] Test chat base

### FASE 2: Strumenti Legali
- [ ] Test Normattiva
- [ ] Test EUR-Lex
- [ ] Test Massimari
- [ ] Test Ricerca Web

### FASE 3: Memoria L1-L3
- [ ] L1 Working (Valkey)
- [ ] L2 Episodic (PostgreSQL)
- [ ] L3 Semantic (pgvector)

### FASE 4: Blindatura Security
- [ ] Tenant isolation
- [ ] User isolation
- [ ] Thread isolation
- [ ] JWT validation

### FASE 5: ORCHIDEA Minimo
- [ ] Phase A-D-F attive
- [ ] Tool execution
- [ ] Memory writeback

### FASE 6: Stabilizzazione
- [ ] Error handling
- [ ] Logging
- [ ] Performance < 5s

---

## P2/P3 Backlog

| Item | Priority | Note |
|------|----------|------|
| Mini pannello utente webchat | P2 | Profilo, preferenze |
| ORCHIDEA Phase E (Guardrails) | P2 | Validazione risposte |
| lexe-channels (WhatsApp/Email) | P3 | Non prioritario |
| Admin panel | P4 | Dashboard metriche |

---

## Known Issues

| Issue | Status | Workaround |
|-------|--------|------------|
| 404 su /api/v1/identity/access-requests | Open | Endpoint LEO-specific, rimuovere da webchat |

---

*Ultimo aggiornamento: 2026-02-01*
