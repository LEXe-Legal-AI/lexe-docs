# RLS Activation Roadmap - Bivio Decisionale

> **Data:** 2026-02-02
> **Decisione:** Opzione A (RLS pronto, non attivo per app)
> **Motivo:** Priorità go-live su MAIN

---

## Stato Attuale

### Cosa È Stato Implementato (STAGE)

| Componente | Status | Note |
|------------|--------|------|
| Utente `lexe_app` | ✅ Creato | Non-superuser, password: `lexe_app_secure_2026!` |
| Permessi DB | ✅ Configurati | SELECT/INSERT/UPDATE/DELETE su core, memory, public |
| Funzioni helper | ✅ Create | `core.current_tenant_id()`, `core.is_rls_bypass()` |
| RLS su 6 tabelle | ✅ Abilitato | FORCE ROW LEVEL SECURITY attivo |
| Policy RLS | ✅ Create | `tenant_isolation_*` per ogni tabella |
| Test RLS | ✅ Passato | Funziona correttamente con `lexe_app` |

### Cosa NON È Stato Attivato

| Componente | Status | Motivo |
|------------|--------|--------|
| Connection string `lexe_app` | ❌ Non cambiata | Richiede modifiche al codice |
| Middleware tenant context | ❌ Non implementato | Richiede sviluppo e test |
| RLS attivo per app runtime | ❌ Non attivo | Dipende dai punti sopra |

---

## Il Bivio

```
                    RLS IMPLEMENTATO
                          │
                          ▼
           ┌──────────────────────────────┐
           │   BIVIO DECISIONALE          │
           │   2026-02-02                 │
           └──────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│  OPZIONE A    │       │  OPZIONE B    │
│  (SCELTA)     │       │  (BACKLOG)    │
│               │       │               │
│ RLS pronto    │       │ RLS attivo    │
│ ma non attivo │       │ per app       │
│ per app       │       │               │
│               │       │               │
│ Connection:   │       │ Connection:   │
│ lexe (super)  │       │ lexe_app      │
│               │       │               │
│ Protezione:   │       │ Protezione:   │
│ Solo Python   │       │ Python + DB   │
└───────────────┘       └───────────────┘
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│ GO-LIVE MAIN  │       │ POST GO-LIVE  │
│ ASAP          │       │ In STAGE      │
│               │       │ poi MAIN      │
│ Zero rischi   │       │               │
│ Zero downtime │       │ Test completi │
└───────────────┘       └───────────────┘
```

---

## Opzione A: Stato Attuale (SCELTA)

### Configurazione

```yaml
# docker-compose.yml - ATTUALE
services:
  lexe-core:
    environment:
      DATABASE_URL: postgresql://lexe:***@lexe-postgres:5432/lexe
      #                         ^^^^
      #                         SUPERUSER (bypassa RLS)
```

### Come Funziona Oggi

```
Request → JWT → tenant_id estratto
                    │
                    ▼
            ConversationService(tenant_id)
                    │
                    ▼
            Query Python con WHERE tenant_id = X
                    │
                    ▼
            PostgreSQL (lexe superuser)
                    │
                    ▼
            RLS BYPASSATO (superuser ignora policy)
                    │
                    ▼
            Filtro avviene SOLO nel codice Python
```

### Livello di Sicurezza

| Threat | Protetto? | Note |
|--------|-----------|------|
| Utente normale vede altri tenant | ✅ | Codice Python filtra |
| Bug nel codice Python | ❌ | Nessun filtro DB |
| SQL Injection | ❌ | Accesso a tutti i dati |
| Accesso diretto DB con `lexe` | ❌ | Vede tutto |
| Accesso diretto DB con `lexe_app` | ✅ | RLS attivo |

### Pro e Contro

| Pro | Contro |
|-----|--------|
| Zero modifiche al codice | SQL injection = data leak |
| Zero rischi go-live | Nessun audit a livello DB |
| Funziona subito | Doppia manutenzione filtri |
| Testato e stabile | Security non completa |

---

## Opzione B: RLS Attivo per App (BACKLOG)

### Configurazione Target

```yaml
# docker-compose.yml - TARGET FUTURO
services:
  lexe-core:
    environment:
      DATABASE_URL: postgresql://lexe_app:***@lexe-postgres:5432/lexe
      #                         ^^^^^^^^
      #                         NON-SUPERUSER (RLS attivo)

      # Per migrations/admin
      DATABASE_URL_ADMIN: postgresql://lexe:***@lexe-postgres:5432/lexe
```

### Come Funzionerà

```
Request → JWT → tenant_id estratto
                    │
                    ▼
            Middleware/Dependency
                    │
                    ▼
            SET app.current_tenant_id = 'uuid'
                    │
                    ▼
            Query (con o senza WHERE)
                    │
                    ▼
            PostgreSQL (lexe_app)
                    │
                    ▼
            RLS ATTIVO → filtra per tenant_id
                    │
                    ▼
            Solo dati del tenant ritornati
```

### Livello di Sicurezza

| Threat | Protetto? | Note |
|--------|-----------|------|
| Utente normale vede altri tenant | ✅ | Python + DB |
| Bug nel codice Python | ✅ | DB filtra comunque |
| SQL Injection | ✅ | RLS protegge |
| Accesso diretto DB con `lexe` | ❌ | Superuser bypassa |
| Accesso diretto DB con `lexe_app` | ✅ | RLS attivo |

### Cosa Serve Implementare

1. **Middleware Tenant Context**
   ```python
   # Da creare in lexe-core
   class TenantContextMiddleware:
       async def dispatch(self, request, call_next):
           tenant_id = get_tenant_from_jwt(request)
           await db.execute(f"SET app.current_tenant_id = '{tenant_id}'")
           return await call_next(request)
   ```

2. **Aggiornare Connection String**
   ```bash
   # In .env o docker-compose
   DATABASE_URL=postgresql://lexe_app:lexe_app_secure_2026!@lexe-postgres:5432/lexe
   ```

3. **Gestire Casi Admin**
   ```python
   # Per operazioni che richiedono bypass RLS
   await db.execute("SET app.rls_bypass = 'true'")
   ```

4. **Aggiornare Test**
   ```python
   # Tutti i test devono settare tenant context
   @pytest.fixture
   async def tenant_session(db_session, test_tenant):
       await db_session.execute(
           text("SET app.current_tenant_id = :tid"),
           {"tid": str(test_tenant.id)}
       )
       yield db_session
   ```

### Piano di Implementazione

```
FASE 1: Sviluppo (STAGE)
├── Creare middleware TenantContext
├── Creare dependency get_tenant_session
├── Aggiornare ConversationService
├── Aggiornare MemoryService
└── Unit test

FASE 2: Test Integrazione (STAGE)
├── Cambiare connection string a lexe_app
├── Test E2E webchat
├── Test API conversations
├── Test memory
└── Verificare nessuna regressione

FASE 3: Rollout (MAIN)
├── Merge codice
├── Cambiare connection string
├── Monitoring intensivo
└── Rollback plan pronto
```

---

## Rollback Plan (per Opzione B)

Se qualcosa va storto dopo l'attivazione:

```bash
# 1. Cambiare connection string back a lexe
docker exec lexe-core env | grep DATABASE_URL
# Edit .env: DATABASE_URL=postgresql://lexe:***@...

# 2. Restart containers
docker compose restart lexe-core lexe-memory

# 3. Verificare
docker logs lexe-core --tail 50

# RLS rimane configurato ma viene bypassato da superuser
```

---

## Checklist Pre-Attivazione (Opzione B)

Prima di attivare RLS per l'app in MAIN:

- [ ] Middleware implementato e testato in STAGE
- [ ] Tutti i service aggiornati per settare tenant context
- [ ] Test E2E passano con `lexe_app`
- [ ] Performance test (RLS aggiunge overhead minimo)
- [ ] Rollback plan documentato e testato
- [ ] Team informato del cambio
- [ ] Monitoring configurato per errori RLS
- [ ] Backup DB recente

---

## Riferimenti

| Documento | Contenuto |
|-----------|-----------|
| [RLS-MULTITENANCY.md](RLS-MULTITENANCY.md) | Setup completo RLS |
| [WEBGUI-ADMIN-DESIGN.md](WEBGUI-ADMIN-DESIGN.md) | Pattern per admin panel |
| [BACKLOG.md](BACKLOG.md) | Task per implementazione |

---

## Timeline Suggerita

| Milestone | Target | Note |
|-----------|--------|------|
| Go-live MAIN | ASAP | Con Opzione A (RLS non attivo) |
| Implementare Opzione B in STAGE | Post go-live | 1-2 settimane |
| Test completi STAGE | +1 settimana | E2E, performance |
| Rollout MAIN | Quando stabile | Con monitoring |

---

*Documento creato: 2026-02-02*
*Decisione: Opzione A (RLS pronto, non attivo)*
*Prossima review: Post go-live MAIN*
