# LEXE Platform - Status

> Stato attuale della piattaforma LEXE Legal AI

---

## Infrastructure Status (2026-02-13)

### Production (49.12.85.92)

| Container | Status | Note |
|-----------|--------|------|
| lexe-core | Healthy | Gateway + Auth OIDC + Admin API |
| lexe-orchestrator | Healthy | ORCHIDEA Pipeline (FF disabled) |
| lexe-memory | Healthy | L1-L3 Memory |
| lexe-webchat | Healthy | Frontend chat |
| lexe-logto | Healthy | CIAM Auth |
| lexe-litellm | Healthy | LLM Gateway |
| lexe-postgres | Healthy | DB + pgvector |
| lexe-valkey | Healthy | Cache L1 |
| lexe-max | Healthy | KB Legal (normativa, massime, embeddings) |
| lexe-tools | Healthy | Legal Tools API (Normattiva, EUR-Lex, InfoLex) |
| lexe-temporal | Healthy | Workflow orchestration |
| lexe-temporal-ui | Healthy | Temporal dashboard |

**Total:** 12 containers

### Staging (91.99.229.111)

| Container | Status | Note |
|-----------|--------|------|
| lexe-core | Healthy | stage branch |
| lexe-webchat | Healthy | Built from source via override |
| lexe-logto | Healthy | ENDPOINT=auth.stage.lexe.pro |
| lexe-litellm | Healthy | |
| lexe-tools | Healthy | |
| lexe-postgres | Healthy | |
| lexe-valkey | Healthy | |
| lexe-max | Healthy | |
| lexe-temporal | Healthy | |

---

## Services URLs

### Production

| Service | URL | Status |
|---------|-----|--------|
| Webchat | https://ai.lexe.pro | OK |
| API | https://api.lexe.pro | OK |
| Auth OIDC | https://auth.lexe.pro | OK |
| LLM Gateway | https://llm.lexe.pro | OK |

### Staging

| Service | URL | Status |
|---------|-----|--------|
| Webchat | https://stage-chat.lexe.pro | OK |
| API | https://api.stage.lexe.pro | OK |
| Auth OIDC | https://auth.stage.lexe.pro | OK |
| Auth Admin | https://admin.stage.lexe.pro | OK |
| LLM Gateway | https://llm.stage.lexe.pro | OK |

---

## Deploy Commands

```bash
# STAGING (91.99.229.111)
ssh -i ~/.ssh/id_stage_new root@91.99.229.111
cd /opt/lexe-platform/lexe-infra
git checkout stage && git pull origin stage
docker compose -f docker-compose.yml -f docker-compose.override.stage.yml up -d

# PRODUCTION (49.12.85.92)
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92
cd /opt/lexe-platform/lexe-infra
docker compose -f docker-compose.yml -f docker-compose.override.prod.yml up -d
```

> **CRITICO:** L'override file è OBBLIGATORIO. Senza override, Traefik labels mancano e Logto/lexe-core puntano all'ambiente sbagliato.

---

## Database Schema

### lexe-postgres (core)

| Schema | Tabelle | Note |
|--------|---------|------|
| `core` | tenants, contacts, conversations, conversation_messages, responder_personas, users, roles, permissions, role_permissions, tools, tenant_feature_flags, contact_groups, contact_group_members, contact_merges, channel_accounts, channel_policies, channel_routings, group_routings, service_conversations, verified_identifiers, pending_verifications, webhook_endpoints, audit_log, tenant_llm_models, email_verification_tokens, token_sessions | Multi-tenant con RLS |
| `memory` | memories | pgvector 1536d |

### lexe-max (KB Legal)

| Schema | Tabelle | Note |
|--------|---------|------|
| `kb` | normativa (6,335), normativa_chunk (10,246), annotation (13,281), work (69), embeddings | Asset principale |

---

## Completed Work

### Phase 0-4: Separazione da LEO
- [x] Backup completo
- [x] DNS configurato (prod + staging)
- [x] LEXE Logto su auth.lexe.pro (prod) e auth.stage.lexe.pro (staging)
- [x] LEXE LiteLLM su llm.lexe.pro
- [x] Containers LEXE separati e healthy
- [x] LEO references cleanup (whatsapp.py, aria_router.py, etc.)

### WS5: Legal Tools Integration (2026-02-12)
- [x] lexe-tools-it: Normattiva, EUR-Lex, InfoLex API tools
- [x] Tool executor con URL nei risultati LLM
- [x] System prompt persona con "Riferimenti Normativi" obbligatori in calce
- [x] E2E test: tool calls + LLM con persona prompt
- [x] EUR-Lex: URL incluso nel response dict

### WS6: Database Migrations (2026-02-13)
- [x] **Migration 001**: RLS policies (fn_get_tenant_id, fn_is_superadmin), audit_log, tenant_llm_models, conversation_messages.tenant_id
- [x] **Migration 002**: 13 tabelle mancanti create (RBAC, channel, verification, etc.)
- [x] **Migration 003**: responder_personas migrato da organization_id (VARCHAR) a tenant_id (UUID FK)
- [x] customer_router.py aggiornato: get_persona_prompt() usa JOIN con tenants
- [x] save_message() aggiornato: passa tenant_id per conversation_messages NOT NULL

### WS7: Staging Infrastructure Fix (2026-02-13)
- [x] docker-compose.override.stage.yml: extra_hosts per auth.stage.lexe.pro
- [x] Logto admin router corretto: admin.stage.lexe.pro (match DNS)
- [x] .env.stage allineato con override (auth.stage.lexe.pro)
- [x] Deploy con override: tutti i servizi healthy
- [x] Traefik routers: webchat, API, auth, admin, litellm tutti enabled
- [x] Auth flow E2E funzionante su staging

### WS8: UX Fixes (2026-02-13)
- [x] Follow-up suggestions: prompt corretto per generare domande dal POV utente
- [x] Indagine Normattiva permalink: confermato che deep-link a singolo articolo NON è supportato

### Admin Panel Backend (2026-02-13)
- [x] 33 endpoints REST montati su /api/v1/admin
- [x] RBAC: 6 ruoli (superadmin, admin, agent, operator, user, viewer)
- [x] Audit log con tenant isolation
- [x] FF_STRICT_TENANT_NO_FALLBACK=true

---

## Architettura Auth

### Logto OIDC Flow

```
Browser → stage-chat.lexe.pro → Logto (auth.stage.lexe.pro)
  → JWT con organization_id claim
  → lexe-core valida: issuer + audience (api.stage.lexe.pro)
  → risolve tenant via core.tenants.logto_organization_id
```

### Staging Logto Config

| Parametro | Valore |
|-----------|--------|
| App ID | dbj6ca04zxzwz7m61aww3 |
| App Name | LEXE Webchat STAGE |
| Redirect URI | https://stage-chat.lexe.pro/callback |
| API Resource | https://api.stage.lexe.pro |
| Organization | lexe-default-stage |
| Tenant UUID | 67a08dc0-4677-423d-b962-f890c6d6b5a9 |

---

## In Progress

### Admin Panel Frontend (Task #6)
- [ ] Scaffolding React app (lexe-admin)
- [ ] Dashboard con metriche conversazioni
- [ ] CRUD Personas
- [ ] Gestione utenti/ruoli
- [ ] Audit log viewer
- [ ] Configurazione modelli LLM per tenant

---

## Known Issues

| Issue | Status | Note |
|-------|--------|------|
| Normattiva deep-link articolo | Fixed | Serviva suffisso annex `:2` per codici (CC, CP, CPC, CPP, C.Nav). Senza `:2`, `~artN` puntava all'art del decreto contenitore |
| Jaeger traces export | Minor | `shared-jaeger` non attivo su staging, warning nei log ma nessun impatto funzionale |
| Let's Encrypt rate limit | Minor | Alcuni domini legacy (play.lexe.pro, leo.itconsultingsrl.com) in rate limit |
| 404 su /api/v1/identity/access-requests | Open | Endpoint LEO-specific, rimuovere da webchat |

---

## Normattiva Permalink - Deep Link Articoli

### Regola fondamentale: Codici vs Atti normali

I **codici** (CC, CP, CPC, CPP, C.Nav.) sono allegati a decreti. L'URN deve includere il suffisso annex `:2` per raggiungere gli articoli del codice:

```
# CODICI: servono :2 prima di ~art (articoli nell'allegato del decreto)
# Senza :2, ~artN punta all'articolo del decreto contenitore (sempre art. 1 di approvazione)

# Codice Civile, Art. 2043 (vigente)
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:regio.decreto:1942-03-16;262:2~art2043!vig=

# Codice Penale, Art. 575 (vigente)
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:regio.decreto:1930-10-19;1398:2~art575!vig=

# ATTI NORMALI: ~art funziona direttamente (no annex)
# D.Lgs. 152/2006, Art. 74
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:decreto.legislativo:2006-04-03;152~art74!vig=

# Vigente a data specifica
https://www.normattiva.it/uri-res/N2Ls?urn:nir:stato:decreto.legge:2008-11-10;180~art2!vig=2009-11-10
```

### Suffissi annex per codice

| Codice | Decreto | Annex |
|--------|---------|-------|
| Codice Civile | R.D. 262/1942 | `:2` |
| Codice Penale | R.D. 1398/1930 | `:2` |
| Codice Proc. Civile | R.D. 1443/1940 | `:2` |
| Codice Proc. Penale | D.P.R. 447/1988 | `:2` |
| Codice Navigazione | R.D. 327/1942 | `:2` |
| Costituzione | - | nessuno (atto autonomo) |

Il tool (`normattiva.py`) usa `CODICI_PREDEFINITI[code]["urn_annex"]` per inserire automaticamente il suffisso corretto.

---

*Ultimo aggiornamento: 2026-02-13*
