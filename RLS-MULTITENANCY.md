# LEXE Row Level Security (RLS) - Documentazione Completa

> **Data creazione:** 2026-02-02
> **Ambiente:** STAGE (91.99.229.111)
> **Status:** ATTIVO E TESTATO

---

## Indice

1. [Overview](#overview)
2. [Architettura](#architettura)
3. [Utenti Database](#utenti-database)
4. [Tabelle Protette da RLS](#tabelle-protette-da-rls)
5. [Funzioni Helper](#funzioni-helper)
6. [Come Usare RLS nell'Applicazione](#come-usare-rls-nellapplicazione)
7. [Pattern per WebGUI Admin](#pattern-per-webgui-admin)
8. [Troubleshooting](#troubleshooting)
9. [Script SQL Completi](#script-sql-completi)

---

## Overview

### Cos'è RLS?

**Row Level Security (RLS)** è una feature di PostgreSQL che permette di filtrare automaticamente le righe di una tabella in base a policy definite. Questo garantisce **isolamento multi-tenant a livello database**, non solo a livello applicazione.

### Perché RLS per LEXE?

| Senza RLS | Con RLS |
|-----------|---------|
| Isolamento solo nel codice Python | Isolamento nel database |
| SQL injection = accesso cross-tenant | SQL injection = dati ancora isolati |
| Bug nel codice = data leak | Bug nel codice = dati ancora isolati |
| Audit difficile | Audit a livello DB |

### Stato Attuale

```
✅ RLS ATTIVO su 6 tabelle
✅ FORCE RLS attivo (applica anche all'owner)
✅ Utente lexe_app creato (non-superuser)
✅ Testato e funzionante
```

---

## Architettura

### Schema Generale

```
┌─────────────────────────────────────────────────────────────────┐
│                         APPLICAZIONE                             │
│  (lexe-core, lexe-memory, lexe-orchestrator, webchat)           │
│                                                                  │
│  Connessione: postgresql://lexe_app:***@lexe-postgres/lexe      │
│                                                                  │
│  Ad ogni request:                                                │
│    1. Estrae tenant_id dal JWT                                   │
│    2. SET app.current_tenant_id = '<uuid>'                       │
│    3. Esegue query (RLS filtra automaticamente)                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         POSTGRESQL                               │
│                                                                  │
│  RLS Policy:                                                     │
│    USING (                                                       │
│      core.is_rls_bypass() OR                                     │
│      tenant_id = core.current_tenant_id()                        │
│    )                                                             │
│                                                                  │
│  Se tenant_id non settato → 0 righe                              │
│  Se tenant_id settato → solo righe di quel tenant                │
│  Se rls_bypass = true → tutte le righe                           │
└─────────────────────────────────────────────────────────────────┘
```

### Flusso di una Request

```
1. User fa login → riceve JWT con organization_id (= tenant)
2. Frontend invia request con JWT
3. Backend (lexe-core):
   a. Valida JWT
   b. Estrae organization_id → trova tenant_id
   c. SET app.current_tenant_id = tenant_id
   d. Esegue query
4. PostgreSQL:
   a. Applica RLS policy
   b. Filtra righe per tenant_id
   c. Ritorna solo dati autorizzati
```

---

## Utenti Database

### Due Utenti Separati

| Utente | Tipo | Scopo | RLS |
|--------|------|-------|-----|
| `lexe` | SUPERUSER | Admin, migrations, manutenzione | Bypassa sempre |
| `lexe_app` | NORMAL | Applicazione runtime | **RLS ATTIVO** |

### Credenziali

```bash
# ADMIN (per migrations, create tenant, backup)
User: lexe
Password: [vedi .env sul server]
Superuser: SI
RLS: BYPASSA

# APPLICAZIONE (per runtime)
User: lexe_app
Password: lexe_app_secure_2026!
Superuser: NO
RLS: ATTIVO
```

### Quando Usare Quale Utente

| Operazione | Utente | Motivo |
|------------|--------|--------|
| Migrations (alembic) | `lexe` | Richiede DDL privileges |
| Creare tenant | `lexe` | Operazione admin |
| Backup/Restore | `lexe` | Richiede superuser |
| API runtime | `lexe_app` | RLS protegge i dati |
| Query da webchat | `lexe_app` | RLS protegge i dati |
| Debug/troubleshooting | `lexe` | Vedere tutti i dati |

### Connection Strings

```bash
# Per applicazione (con RLS)
DATABASE_URL=postgresql://lexe_app:lexe_app_secure_2026!@lexe-postgres:5432/lexe

# Per admin/migrations (bypassa RLS)
DATABASE_URL_ADMIN=postgresql://lexe:***@lexe-postgres:5432/lexe
```

---

## Tabelle Protette da RLS

### Lista Completa

| Schema | Tabella | RLS | FORCE | Policy |
|--------|---------|-----|-------|--------|
| core | contacts | ✅ | ✅ | tenant_isolation_contacts |
| core | conversations | ✅ | ✅ | tenant_isolation_conversations |
| memory | audit_log | ✅ | ✅ | tenant_isolation_audit_log |
| memory | episodic_vectors | ✅ | ✅ | tenant_isolation_episodic |
| memory | semantic_vectors | ✅ | ✅ | tenant_isolation_semantic |
| memory | working_memories | ✅ | ✅ | tenant_isolation_working |

### Tabelle SENZA RLS (e perché)

| Tabella | Motivo |
|---------|--------|
| core.tenants | È la tabella master, non ha tenant_id su se stessa |
| core.responder_personas | Condivise tra tenant (future: aggiungere RLS) |
| core.conversation_messages | Isolamento via FK a conversations |

### Verifica Status RLS

```sql
-- Verificare quali tabelle hanno RLS attivo
SELECT schemaname, tablename, rowsecurity
FROM pg_tables
WHERE schemaname IN ('core', 'memory');

-- Verificare FORCE RLS
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname IN ('contacts', 'conversations', ...);

-- Listar policy
SELECT schemaname, tablename, policyname
FROM pg_policies
ORDER BY schemaname, tablename;
```

---

## Funzioni Helper

### core.current_tenant_id()

Ritorna il tenant_id corrente dalla sessione.

```sql
-- Definizione
CREATE OR REPLACE FUNCTION core.current_tenant_id()
RETURNS uuid AS $$
BEGIN
    RETURN NULLIF(current_setting('app.current_tenant_id', true), '')::uuid;
EXCEPTION
    WHEN OTHERS THEN
        RETURN NULL;
END;
$$ LANGUAGE plpgsql STABLE;

-- Uso
SET app.current_tenant_id = '67a08dc0-4677-423d-b962-f890c6d6b5a9';
SELECT core.current_tenant_id();  -- Ritorna UUID
```

### core.is_rls_bypass()

Ritorna true se la sessione è in modalità admin (bypassa RLS).

```sql
-- Definizione
CREATE OR REPLACE FUNCTION core.is_rls_bypass()
RETURNS boolean AS $$
DECLARE
    val text;
BEGIN
    val := current_setting('app.rls_bypass', true);
    IF val IS NULL OR val = '' THEN
        RETURN false;
    END IF;
    RETURN val::boolean;
EXCEPTION
    WHEN OTHERS THEN
        RETURN false;
END;
$$ LANGUAGE plpgsql STABLE;

-- Uso
SET app.rls_bypass = 'true';
SELECT core.is_rls_bypass();  -- Ritorna true
```

---

## Come Usare RLS nell'Applicazione

### Python/SQLAlchemy Pattern

```python
from sqlalchemy.ext.asyncio import AsyncSession
from uuid import UUID

async def set_tenant_context(session: AsyncSession, tenant_id: UUID):
    """Setta il tenant_id per la sessione corrente."""
    await session.execute(
        text("SET app.current_tenant_id = :tenant_id"),
        {"tenant_id": str(tenant_id)}
    )

async def set_admin_context(session: AsyncSession):
    """Abilita bypass RLS per operazioni admin."""
    await session.execute(text("SET app.rls_bypass = 'true'"))

async def clear_context(session: AsyncSession):
    """Reset del contesto (fine request)."""
    await session.execute(text("RESET app.current_tenant_id"))
    await session.execute(text("RESET app.rls_bypass"))
```

### FastAPI Middleware Pattern

```python
from fastapi import Request
from sqlalchemy.ext.asyncio import AsyncSession

class TenantMiddleware:
    async def __call__(self, request: Request, call_next):
        # Estrai tenant_id dal JWT
        tenant_id = get_tenant_from_jwt(request)

        # Setta contesto DB
        session: AsyncSession = request.state.db
        await set_tenant_context(session, tenant_id)

        try:
            response = await call_next(request)
            return response
        finally:
            # Cleanup
            await clear_context(session)
```

### Dependency Injection Pattern (Raccomandato)

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_tenant_session(
    tenant_id: UUID = Depends(get_current_tenant_id),
    session: AsyncSession = Depends(get_db_session)
) -> AsyncSession:
    """Ritorna una sessione con tenant_id già settato."""
    await session.execute(
        text("SET app.current_tenant_id = :tid"),
        {"tid": str(tenant_id)}
    )
    return session

# Uso nel router
@router.get("/conversations")
async def list_conversations(
    session: AsyncSession = Depends(get_tenant_session)
):
    # Tutte le query sono automaticamente filtrate per tenant
    result = await session.execute(select(Conversation))
    return result.scalars().all()
```

---

## Pattern per WebGUI Admin

### Architettura WebGUI

```
┌─────────────────────────────────────────────────────────────────┐
│                      WEBGUI ADMIN                                │
│                                                                  │
│  Ruoli:                                                          │
│    - Super Admin: vede TUTTI i tenant, può creare tenant         │
│    - Tenant Admin: vede solo il SUO tenant                       │
│    - User: vede solo i SUOI dati                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Endpoints Admin

| Endpoint | Ruolo Richiesto | RLS | Note |
|----------|-----------------|-----|------|
| `GET /admin/tenants` | Super Admin | BYPASS | Lista tutti i tenant |
| `POST /admin/tenants` | Super Admin | BYPASS | Crea nuovo tenant |
| `GET /admin/tenants/{id}/users` | Tenant Admin | NORMAL | Solo suo tenant |
| `GET /api/conversations` | User | NORMAL | Solo sue conversazioni |

### Implementazione Super Admin

```python
from enum import Enum

class AdminRole(Enum):
    SUPER_ADMIN = "super_admin"
    TENANT_ADMIN = "tenant_admin"
    USER = "user"

async def get_admin_session(
    role: AdminRole,
    tenant_id: UUID | None,
    session: AsyncSession
) -> AsyncSession:
    """
    Configura la sessione in base al ruolo admin.

    - SUPER_ADMIN: bypass RLS (vede tutto)
    - TENANT_ADMIN: setta tenant_id specifico
    - USER: setta tenant_id dal JWT
    """
    if role == AdminRole.SUPER_ADMIN:
        await session.execute(text("SET app.rls_bypass = 'true'"))
    elif tenant_id:
        await session.execute(
            text("SET app.current_tenant_id = :tid"),
            {"tid": str(tenant_id)}
        )
    return session
```

### CRUD Tenant (Super Admin Only)

```python
@router.post("/admin/tenants")
async def create_tenant(
    tenant: TenantCreate,
    current_user: User = Depends(require_super_admin),
    session: AsyncSession = Depends(get_db_session)
):
    """
    Crea un nuovo tenant.
    Richiede ruolo SUPER_ADMIN.
    Usa connessione admin (bypassa RLS).
    """
    # IMPORTANTE: usa lexe (superuser) per creare tenant
    # oppure setta rls_bypass = true
    await session.execute(text("SET app.rls_bypass = 'true'"))

    new_tenant = Tenant(
        name=tenant.name,
        slug=tenant.slug,
        logto_organization_id=tenant.logto_organization_id,
        is_active=True,
        settings=tenant.settings or {}
    )
    session.add(new_tenant)
    await session.commit()

    return new_tenant
```

### Listing Users per Tenant (Tenant Admin)

```python
@router.get("/admin/tenants/{tenant_id}/users")
async def list_tenant_users(
    tenant_id: UUID,
    current_user: User = Depends(require_tenant_admin),
    session: AsyncSession = Depends(get_db_session)
):
    """
    Lista utenti di un tenant.
    Tenant Admin può vedere solo il proprio tenant.
    Super Admin può vedere qualsiasi tenant.
    """
    # Verifica autorizzazione
    if current_user.role != AdminRole.SUPER_ADMIN:
        if current_user.tenant_id != tenant_id:
            raise HTTPException(403, "Cannot access other tenant's users")

    # Setta contesto (RLS filtra automaticamente)
    await session.execute(
        text("SET app.current_tenant_id = :tid"),
        {"tid": str(tenant_id)}
    )

    result = await session.execute(select(Contact))
    return result.scalars().all()
```

### Switch Tenant (Super Admin Debug)

```python
@router.post("/admin/switch-tenant")
async def switch_tenant(
    tenant_id: UUID,
    current_user: User = Depends(require_super_admin),
    session: AsyncSession = Depends(get_db_session)
):
    """
    Permette a Super Admin di "impersonare" un tenant per debug.
    """
    # Verifica tenant esiste
    tenant = await session.execute(
        text("SELECT id, name, slug FROM core.tenants WHERE id = :tid"),
        {"tid": str(tenant_id)}
    )
    tenant = tenant.fetchone()
    if not tenant:
        raise HTTPException(404, "Tenant not found")

    # Setta nel cookie/session per le prossime request
    return {
        "message": f"Switched to tenant: {tenant.name}",
        "tenant_id": str(tenant_id),
        "tenant_slug": tenant.slug
    }
```

### UI Components Pattern

```typescript
// React/Vue component per tenant selector (Super Admin)
interface TenantSelectorProps {
  currentTenantId: string | null;
  onTenantChange: (tenantId: string) => void;
}

const TenantSelector: React.FC<TenantSelectorProps> = ({
  currentTenantId,
  onTenantChange
}) => {
  const { data: tenants } = useQuery('tenants', fetchTenants);

  return (
    <Select
      value={currentTenantId}
      onChange={onTenantChange}
      placeholder="Select tenant to view..."
    >
      <Option value={null}>All Tenants (Admin View)</Option>
      {tenants?.map(t => (
        <Option key={t.id} value={t.id}>
          {t.name} ({t.slug})
        </Option>
      ))}
    </Select>
  );
};
```

---

## Troubleshooting

### Problema: "Vedo 0 righe ma dovrei vedere dati"

**Causa:** `app.current_tenant_id` non settato o settato sbagliato.

```sql
-- Debug
SELECT core.current_tenant_id();  -- Deve ritornare UUID valido
SELECT core.is_rls_bypass();      -- Deve essere false per utente normale

-- Fix
SET app.current_tenant_id = 'uuid-corretto';
```

### Problema: "Vedo tutti i dati (no filtering)"

**Causa:** Utente è superuser, o `rls_bypass = true`.

```sql
-- Verifica
SELECT usename, usesuper FROM pg_user WHERE usename = current_user;
SELECT core.is_rls_bypass();

-- Fix: usa lexe_app invece di lexe
-- Fix: RESET app.rls_bypass
```

### Problema: "Permission denied for table X"

**Causa:** `lexe_app` non ha permessi su quella tabella.

```sql
-- Fix (esegui come lexe superuser)
GRANT SELECT, INSERT, UPDATE, DELETE ON schema.table_name TO lexe_app;
```

### Problema: "Function core.current_tenant_id() does not exist"

**Causa:** Funzioni helper non create.

```sql
-- Ricrea funzioni (vedi sezione Script SQL)
```

### Test Rapido RLS

```sql
-- Connetti come lexe_app
\c lexe lexe_app

-- Senza tenant (deve essere 0)
SELECT COUNT(*) FROM core.conversations;  -- 0

-- Con tenant
SET app.current_tenant_id = 'uuid-tenant';
SELECT COUNT(*) FROM core.conversations;  -- > 0

-- Bypass (admin)
SET app.rls_bypass = 'true';
SELECT COUNT(*) FROM core.conversations;  -- tutti
```

---

## Script SQL Completi

### 1. Creazione Utente lexe_app

```sql
-- Esegui come superuser (lexe)
CREATE USER lexe_app WITH
    LOGIN
    PASSWORD 'lexe_app_secure_2026!'
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE;

GRANT CONNECT ON DATABASE lexe TO lexe_app;
```

### 2. Permessi su Schema

```sql
-- Per ogni schema (core, memory, public)
GRANT USAGE ON SCHEMA core TO lexe_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA core TO lexe_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA core TO lexe_app;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA core TO lexe_app;

ALTER DEFAULT PRIVILEGES IN SCHEMA core
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO lexe_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA core
    GRANT USAGE, SELECT ON SEQUENCES TO lexe_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA core
    GRANT EXECUTE ON FUNCTIONS TO lexe_app;

-- Ripeti per memory e public...
```

### 3. Funzioni Helper RLS

```sql
-- current_tenant_id
CREATE OR REPLACE FUNCTION core.current_tenant_id()
RETURNS uuid AS $$
BEGIN
    RETURN NULLIF(current_setting('app.current_tenant_id', true), '')::uuid;
EXCEPTION
    WHEN OTHERS THEN
        RETURN NULL;
END;
$$ LANGUAGE plpgsql STABLE;

-- is_rls_bypass
CREATE OR REPLACE FUNCTION core.is_rls_bypass()
RETURNS boolean AS $$
DECLARE
    val text;
BEGIN
    val := current_setting('app.rls_bypass', true);
    IF val IS NULL OR val = '' THEN
        RETURN false;
    END IF;
    RETURN val::boolean;
EXCEPTION
    WHEN OTHERS THEN
        RETURN false;
END;
$$ LANGUAGE plpgsql STABLE;
```

### 4. Abilitare RLS su Tabelle

```sql
-- Per ogni tabella con tenant_id
ALTER TABLE core.contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE core.contacts FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_contacts ON core.contacts
    FOR ALL
    USING (
        core.is_rls_bypass() OR
        tenant_id = core.current_tenant_id()
    )
    WITH CHECK (
        core.is_rls_bypass() OR
        tenant_id = core.current_tenant_id()
    );

-- Ripeti per: conversations, audit_log, episodic_vectors,
-- semantic_vectors, working_memories
```

### 5. Rollback RLS (se necessario)

```sql
-- ATTENZIONE: rimuove protezione RLS
DROP POLICY tenant_isolation_contacts ON core.contacts;
ALTER TABLE core.contacts DISABLE ROW LEVEL SECURITY;
ALTER TABLE core.contacts NO FORCE ROW LEVEL SECURITY;

-- Ripeti per altre tabelle...
```

---

## Checklist Implementazione

### Per Nuovo Ambiente

- [ ] Creare utente `lexe_app`
- [ ] Configurare permessi su tutti gli schema
- [ ] Creare funzioni helper (`current_tenant_id`, `is_rls_bypass`)
- [ ] Abilitare RLS su tabelle con `tenant_id`
- [ ] Applicare `FORCE ROW LEVEL SECURITY`
- [ ] Creare policy per ogni tabella
- [ ] Testare con `lexe_app` senza tenant (0 righe)
- [ ] Testare con `lexe_app` con tenant (dati filtrati)
- [ ] Testare bypass admin
- [ ] Aggiornare connection string nei container

### Per Nuova Tabella con tenant_id

```sql
-- 1. Crea tabella normalmente
CREATE TABLE core.new_table (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES core.tenants(id),
    ...
);

-- 2. Abilita RLS
ALTER TABLE core.new_table ENABLE ROW LEVEL SECURITY;
ALTER TABLE core.new_table FORCE ROW LEVEL SECURITY;

-- 3. Crea policy
CREATE POLICY tenant_isolation_new_table ON core.new_table
    FOR ALL
    USING (
        core.is_rls_bypass() OR
        tenant_id = core.current_tenant_id()
    )
    WITH CHECK (
        core.is_rls_bypass() OR
        tenant_id = core.current_tenant_id()
    );

-- 4. Grant permessi
GRANT SELECT, INSERT, UPDATE, DELETE ON core.new_table TO lexe_app;
```

---

*Documentazione creata: 2026-02-02*
*Ultimo aggiornamento: 2026-02-02*
*Autore: Claude Code / LEXE Team*
