# LEXE WebGUI Admin - Design Document

> **Status:** DESIGN PHASE
> **Target:** Q2 2026
> **Prerequisito:** RLS implementato (vedi RLS-MULTITENANCY.md)

---

## Executive Summary

Questo documento definisce l'architettura e i pattern per la WebGUI Admin di LEXE, con focus su:
- Gestione multi-tenant sicura
- Ruoli e permessi granulari
- Integrazione con RLS PostgreSQL
- UX per Super Admin e Tenant Admin

---

## Ruoli e Permessi

### Gerarchia Ruoli

```
┌─────────────────────────────────────────────────────────────────┐
│                       SUPER ADMIN                                │
│  (Operatori LEXE interni)                                        │
│                                                                  │
│  Può:                                                            │
│    ✅ Vedere TUTTI i tenant                                      │
│    ✅ Creare/modificare/eliminare tenant                         │
│    ✅ Impersonare qualsiasi tenant (debug)                       │
│    ✅ Vedere metriche globali                                    │
│    ✅ Gestire configurazioni sistema                             │
│    ✅ Accesso a logs e audit cross-tenant                        │
│                                                                  │
│  DB: SET app.rls_bypass = 'true'                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       TENANT ADMIN                               │
│  (Admin dello studio legale / azienda cliente)                   │
│                                                                  │
│  Può:                                                            │
│    ✅ Vedere solo il SUO tenant                                  │
│    ✅ Gestire utenti del proprio tenant                          │
│    ✅ Configurare tool e feature del tenant                      │
│    ✅ Vedere metriche del proprio tenant                         │
│    ✅ Gestire Knowledge Base del tenant                          │
│    ❌ NON può vedere altri tenant                                │
│    ❌ NON può creare nuovi tenant                                │
│                                                                  │
│  DB: SET app.current_tenant_id = '<proprio_tenant>'              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         USER                                     │
│  (Avvocato, collaboratore, utente finale)                        │
│                                                                  │
│  Può:                                                            │
│    ✅ Usare webchat                                              │
│    ✅ Vedere proprie conversazioni                               │
│    ✅ Gestire propria memoria/preferenze                         │
│    ❌ NON può accedere ad admin panel                            │
│    ❌ NON può vedere dati altri utenti                           │
│                                                                  │
│  DB: SET app.current_tenant_id = '<tenant>' (via JWT)            │
└─────────────────────────────────────────────────────────────────┘
```

### Matrice Permessi Dettagliata

| Risorsa | Super Admin | Tenant Admin | User |
|---------|-------------|--------------|------|
| **Tenants** ||||
| List all tenants | ✅ | ❌ | ❌ |
| Create tenant | ✅ | ❌ | ❌ |
| Edit own tenant | ✅ | ✅ | ❌ |
| Delete tenant | ✅ | ❌ | ❌ |
| **Users (Contacts)** ||||
| List all users (cross-tenant) | ✅ | ❌ | ❌ |
| List tenant users | ✅ | ✅ | ❌ |
| Create user | ✅ | ✅ | ❌ |
| Edit user | ✅ | ✅ | ❌ |
| Delete user | ✅ | ✅ | ❌ |
| **Conversations** ||||
| View all (cross-tenant) | ✅ | ❌ | ❌ |
| View tenant conversations | ✅ | ✅ | ❌ |
| View own conversations | ✅ | ✅ | ✅ |
| Delete conversations | ✅ | ✅ | ✅ (own) |
| **Memory** ||||
| View all memories (cross-tenant) | ✅ | ❌ | ❌ |
| View tenant memories | ✅ | ✅ | ❌ |
| Clear tenant memory | ✅ | ✅ | ❌ |
| **Tools/Features** ||||
| Configure global tools | ✅ | ❌ | ❌ |
| Configure tenant tools | ✅ | ✅ | ❌ |
| **Metrics/Analytics** ||||
| Global dashboard | ✅ | ❌ | ❌ |
| Tenant dashboard | ✅ | ✅ | ❌ |
| **Audit Logs** ||||
| View all audit logs | ✅ | ❌ | ❌ |
| View tenant audit logs | ✅ | ✅ | ❌ |

---

## Architettura Backend

### Endpoint Structure

```
/admin/
├── /tenants                     # Super Admin only
│   ├── GET    /                 # List all tenants
│   ├── POST   /                 # Create tenant
│   ├── GET    /{id}             # Get tenant details
│   ├── PUT    /{id}             # Update tenant
│   ├── DELETE /{id}             # Delete tenant (soft)
│   └── POST   /{id}/impersonate # Switch to tenant (debug)
│
├── /users                       # Tenant-scoped
│   ├── GET    /                 # List users (filtered by tenant)
│   ├── POST   /                 # Create user
│   ├── GET    /{id}             # Get user details
│   ├── PUT    /{id}             # Update user
│   └── DELETE /{id}             # Delete user (soft)
│
├── /conversations               # Tenant-scoped
│   ├── GET    /                 # List conversations
│   ├── GET    /{id}             # Get conversation + messages
│   ├── DELETE /{id}             # Delete conversation
│   └── GET    /{id}/export      # Export conversation (PDF/JSON)
│
├── /memory                      # Tenant-scoped
│   ├── GET    /stats            # Memory statistics
│   ├── GET    /search           # Search memories
│   └── DELETE /clear            # Clear all tenant memory
│
├── /tools                       # Tenant-scoped
│   ├── GET    /                 # List available tools
│   ├── PUT    /{tool}/enable    # Enable tool for tenant
│   └── PUT    /{tool}/disable   # Disable tool for tenant
│
├── /metrics                     # Role-based
│   ├── GET    /global           # Super Admin only
│   └── GET    /tenant           # Current tenant metrics
│
└── /audit                       # Role-based
    ├── GET    /                 # Audit logs (filtered)
    └── GET    /export           # Export audit logs
```

### FastAPI Router Example

```python
# admin/router.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from admin.dependencies import (
    require_super_admin,
    require_tenant_admin,
    get_admin_session
)
from admin.schemas import TenantCreate, TenantResponse
from admin.services import TenantService

router = APIRouter(prefix="/admin", tags=["admin"])

# ============================================================
# TENANT MANAGEMENT (Super Admin Only)
# ============================================================

@router.get("/tenants", response_model=list[TenantResponse])
async def list_tenants(
    current_user: AdminUser = Depends(require_super_admin),
    session: AsyncSession = Depends(get_admin_session)
):
    """
    Lista tutti i tenant.
    Solo Super Admin.
    """
    # RLS bypass automatico per Super Admin
    service = TenantService(session)
    return await service.list_all()


@router.post("/tenants", response_model=TenantResponse)
async def create_tenant(
    tenant: TenantCreate,
    current_user: AdminUser = Depends(require_super_admin),
    session: AsyncSession = Depends(get_admin_session)
):
    """
    Crea un nuovo tenant.
    Solo Super Admin.

    Steps:
    1. Crea record in core.tenants
    2. Crea organization in Logto
    3. Configura default tool policies
    4. Invia email di benvenuto (opzionale)
    """
    service = TenantService(session)

    # Verifica unicità slug
    existing = await service.get_by_slug(tenant.slug)
    if existing:
        raise HTTPException(400, f"Tenant with slug '{tenant.slug}' already exists")

    # Crea tenant
    new_tenant = await service.create(
        name=tenant.name,
        slug=tenant.slug,
        logto_organization_id=tenant.logto_organization_id or tenant.slug,
        settings=tenant.settings or default_tenant_settings()
    )

    # Crea organization in Logto
    await logto_client.create_organization(
        id=new_tenant.logto_organization_id,
        name=new_tenant.name
    )

    return new_tenant


@router.post("/tenants/{tenant_id}/impersonate")
async def impersonate_tenant(
    tenant_id: UUID,
    current_user: AdminUser = Depends(require_super_admin),
    session: AsyncSession = Depends(get_admin_session)
):
    """
    Permette a Super Admin di "impersonare" un tenant.
    Utile per debug e supporto.

    Returns un token temporaneo con tenant_id settato.
    """
    service = TenantService(session)
    tenant = await service.get(tenant_id)
    if not tenant:
        raise HTTPException(404, "Tenant not found")

    # Genera token temporaneo con tenant context
    impersonation_token = create_impersonation_token(
        admin_user_id=current_user.id,
        tenant_id=tenant_id,
        expires_in=3600  # 1 ora
    )

    # Audit log
    await audit_log(
        action="IMPERSONATE_TENANT",
        actor_id=current_user.id,
        target_tenant_id=tenant_id,
        details={"reason": "Admin debug"}
    )

    return {
        "token": impersonation_token,
        "tenant": tenant,
        "expires_in": 3600
    }


# ============================================================
# USER MANAGEMENT (Tenant-Scoped)
# ============================================================

@router.get("/users", response_model=list[UserResponse])
async def list_users(
    current_user: AdminUser = Depends(require_tenant_admin),
    session: AsyncSession = Depends(get_tenant_session)
):
    """
    Lista utenti del tenant corrente.
    Tenant Admin vede solo i propri utenti.
    Super Admin può vedere tutti (se impersona o filtra).
    """
    # RLS filtra automaticamente per tenant
    service = UserService(session)
    return await service.list_all()


@router.post("/users", response_model=UserResponse)
async def create_user(
    user: UserCreate,
    current_user: AdminUser = Depends(require_tenant_admin),
    session: AsyncSession = Depends(get_tenant_session)
):
    """
    Crea un nuovo utente nel tenant corrente.

    Steps:
    1. Crea utente in Logto
    2. Associa a organization in Logto
    3. Crea contact in core.contacts
    4. Invia email di invito
    """
    service = UserService(session)

    # Crea in Logto
    logto_user = await logto_client.create_user(
        email=user.email,
        name=user.display_name
    )

    # Associa a organization
    await logto_client.add_user_to_organization(
        user_id=logto_user.id,
        organization_id=current_user.tenant.logto_organization_id
    )

    # Crea contact locale
    contact = await service.create(
        tenant_id=current_user.tenant_id,
        external_user_id=logto_user.id,
        email=user.email,
        display_name=user.display_name
    )

    # Invia invito
    await send_invitation_email(user.email, current_user.tenant)

    return contact
```

### Dependency Injection per Admin

```python
# admin/dependencies.py
from fastapi import Depends, HTTPException, Security
from fastapi.security import HTTPBearer
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

security = HTTPBearer()

class AdminUser:
    """Rappresenta un utente admin autenticato."""
    id: UUID
    email: str
    role: AdminRole
    tenant_id: UUID | None  # None per Super Admin cross-tenant
    tenant: Tenant | None


async def get_current_admin_user(
    token: str = Security(security),
    session: AsyncSession = Depends(get_db_session)
) -> AdminUser:
    """
    Estrae e valida l'utente admin dal JWT.
    """
    payload = verify_jwt(token.credentials)

    # Determina ruolo
    roles = payload.get("roles", [])
    if "super_admin" in roles:
        role = AdminRole.SUPER_ADMIN
        tenant_id = None
    elif "tenant_admin" in roles:
        role = AdminRole.TENANT_ADMIN
        org_id = payload.get("org_id")
        tenant_id = await get_tenant_id_from_org(session, org_id)
    else:
        raise HTTPException(403, "Admin role required")

    return AdminUser(
        id=UUID(payload["sub"]),
        email=payload.get("email"),
        role=role,
        tenant_id=tenant_id
    )


def require_super_admin(
    user: AdminUser = Depends(get_current_admin_user)
) -> AdminUser:
    """Richiede ruolo Super Admin."""
    if user.role != AdminRole.SUPER_ADMIN:
        raise HTTPException(403, "Super Admin role required")
    return user


def require_tenant_admin(
    user: AdminUser = Depends(get_current_admin_user)
) -> AdminUser:
    """Richiede almeno ruolo Tenant Admin."""
    if user.role not in [AdminRole.SUPER_ADMIN, AdminRole.TENANT_ADMIN]:
        raise HTTPException(403, "Tenant Admin role required")
    return user


async def get_admin_session(
    user: AdminUser = Depends(get_current_admin_user),
    session: AsyncSession = Depends(get_db_session)
) -> AsyncSession:
    """
    Configura la sessione DB in base al ruolo admin.
    """
    if user.role == AdminRole.SUPER_ADMIN:
        # Bypass RLS per vedere tutto
        await session.execute(text("SET app.rls_bypass = 'true'"))
    elif user.tenant_id:
        # Filtra per tenant
        await session.execute(
            text("SET app.current_tenant_id = :tid"),
            {"tid": str(user.tenant_id)}
        )

    return session


async def get_tenant_session(
    user: AdminUser = Depends(get_current_admin_user),
    session: AsyncSession = Depends(get_db_session)
) -> AsyncSession:
    """
    Sessione sempre filtrata per tenant (anche per Super Admin).
    Usare quando si opera su un tenant specifico.
    """
    tenant_id = user.tenant_id

    # Se Super Admin sta impersonando
    if user.role == AdminRole.SUPER_ADMIN:
        # Prendi tenant_id da header o query param
        tenant_id = get_impersonated_tenant_id()

    if not tenant_id:
        raise HTTPException(400, "Tenant context required")

    await session.execute(
        text("SET app.current_tenant_id = :tid"),
        {"tid": str(tenant_id)}
    )

    return session
```

---

## Architettura Frontend

### Stack Tecnologico

```
React 18+ (o Vue 3+)
├── TanStack Query (data fetching/caching)
├── Zustand (state management)
├── React Router (routing)
├── Tailwind CSS (styling)
├── Shadcn/ui (components)
└── React Hook Form + Zod (forms/validation)
```

### Route Structure

```
/admin
├── /                           # Dashboard
├── /tenants                    # Super Admin: lista tenant
│   ├── /new                    # Crea tenant
│   └── /:id                    # Dettaglio tenant
│       ├── /settings           # Configurazione
│       ├── /users              # Utenti del tenant
│       └── /metrics            # Metriche tenant
│
├── /users                      # Lista utenti (tenant-scoped)
│   ├── /new                    # Crea utente
│   └── /:id                    # Dettaglio utente
│
├── /conversations              # Lista conversazioni
│   └── /:id                    # Dettaglio conversazione
│
├── /tools                      # Configurazione tool
│
├── /metrics                    # Dashboard metriche
│
├── /audit                      # Audit logs
│
└── /settings                   # Impostazioni
```

### Component Architecture

```
src/
├── components/
│   ├── layout/
│   │   ├── AdminLayout.tsx       # Layout principale
│   │   ├── Sidebar.tsx           # Navigazione laterale
│   │   ├── Header.tsx            # Header con user menu
│   │   └── TenantSelector.tsx    # Dropdown tenant (Super Admin)
│   │
│   ├── tenants/
│   │   ├── TenantList.tsx
│   │   ├── TenantForm.tsx
│   │   ├── TenantCard.tsx
│   │   └── TenantMetrics.tsx
│   │
│   ├── users/
│   │   ├── UserList.tsx
│   │   ├── UserForm.tsx
│   │   └── UserCard.tsx
│   │
│   └── common/
│       ├── DataTable.tsx
│       ├── SearchBar.tsx
│       ├── Pagination.tsx
│       └── ConfirmDialog.tsx
│
├── hooks/
│   ├── useAuth.ts               # Auth state
│   ├── useTenant.ts             # Tenant context
│   └── usePermissions.ts        # Permission checks
│
├── services/
│   ├── api.ts                   # API client
│   ├── tenants.ts               # Tenant API calls
│   └── users.ts                 # User API calls
│
└── store/
    ├── authStore.ts
    └── tenantStore.ts
```

### Tenant Context Management

```typescript
// store/tenantStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface TenantState {
  // Tenant corrente (per Tenant Admin è fisso, per Super Admin è selezionabile)
  currentTenantId: string | null;

  // Lista tenant (solo Super Admin)
  tenants: Tenant[];

  // Modalità: 'all' (vedi tutto) o 'single' (vedi un tenant)
  viewMode: 'all' | 'single';

  // Actions
  setCurrentTenant: (tenantId: string | null) => void;
  setViewMode: (mode: 'all' | 'single') => void;
  loadTenants: () => Promise<void>;
}

export const useTenantStore = create<TenantState>()(
  persist(
    (set, get) => ({
      currentTenantId: null,
      tenants: [],
      viewMode: 'all',

      setCurrentTenant: (tenantId) => {
        set({ currentTenantId: tenantId });
        // Aggiorna API client headers
        apiClient.setTenantContext(tenantId);
      },

      setViewMode: (mode) => set({ viewMode: mode }),

      loadTenants: async () => {
        const tenants = await tenantsApi.list();
        set({ tenants });
      }
    }),
    { name: 'lexe-admin-tenant' }
  )
);
```

### TenantSelector Component

```tsx
// components/layout/TenantSelector.tsx
import { useTenantStore } from '@/store/tenantStore';
import { useAuth } from '@/hooks/useAuth';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue
} from '@/components/ui/select';

export function TenantSelector() {
  const { user } = useAuth();
  const {
    currentTenantId,
    tenants,
    viewMode,
    setCurrentTenant,
    setViewMode
  } = useTenantStore();

  // Solo Super Admin può switchare tenant
  if (user?.role !== 'super_admin') {
    return (
      <div className="px-3 py-2 text-sm text-muted-foreground">
        {user?.tenant?.name}
      </div>
    );
  }

  return (
    <div className="flex items-center gap-2">
      <Select
        value={viewMode === 'all' ? 'all' : currentTenantId || ''}
        onValueChange={(value) => {
          if (value === 'all') {
            setViewMode('all');
            setCurrentTenant(null);
          } else {
            setViewMode('single');
            setCurrentTenant(value);
          }
        }}
      >
        <SelectTrigger className="w-[200px]">
          <SelectValue placeholder="Select tenant..." />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="all">
            <span className="flex items-center gap-2">
              <GlobeIcon className="h-4 w-4" />
              All Tenants
            </span>
          </SelectItem>
          {tenants.map((tenant) => (
            <SelectItem key={tenant.id} value={tenant.id}>
              <span className="flex items-center gap-2">
                <BuildingIcon className="h-4 w-4" />
                {tenant.name}
              </span>
            </SelectItem>
          ))}
        </SelectContent>
      </Select>

      {viewMode === 'single' && (
        <span className="text-xs text-orange-500 font-medium">
          Viewing as tenant
        </span>
      )}
    </div>
  );
}
```

### Permission-Aware Components

```tsx
// hooks/usePermissions.ts
import { useAuth } from './useAuth';
import { useTenantStore } from '@/store/tenantStore';

export function usePermissions() {
  const { user } = useAuth();
  const { currentTenantId } = useTenantStore();

  const isSuperAdmin = user?.role === 'super_admin';
  const isTenantAdmin = user?.role === 'tenant_admin' || isSuperAdmin;

  return {
    isSuperAdmin,
    isTenantAdmin,

    canListTenants: isSuperAdmin,
    canCreateTenant: isSuperAdmin,
    canDeleteTenant: isSuperAdmin,

    canListUsers: isTenantAdmin,
    canCreateUser: isTenantAdmin,
    canDeleteUser: isTenantAdmin,

    canViewAllConversations: isSuperAdmin && !currentTenantId,
    canViewTenantConversations: isTenantAdmin,

    canViewGlobalMetrics: isSuperAdmin,
    canViewTenantMetrics: isTenantAdmin,

    canViewAuditLogs: isSuperAdmin || isTenantAdmin,
  };
}

// Uso in component
function TenantsPage() {
  const { canListTenants, canCreateTenant } = usePermissions();

  if (!canListTenants) {
    return <Navigate to="/admin" />;
  }

  return (
    <div>
      <h1>Tenants</h1>
      {canCreateTenant && (
        <Button onClick={() => navigate('/admin/tenants/new')}>
          Create Tenant
        </Button>
      )}
      <TenantList />
    </div>
  );
}
```

---

## Flussi Operativi

### 1. Creazione Nuovo Tenant

```
┌─────────────┐    ┌──────────────┐    ┌────────────┐    ┌─────────┐
│ Super Admin │    │  WebGUI      │    │  Backend   │    │   DB    │
└──────┬──────┘    └──────┬───────┘    └─────┬──────┘    └────┬────┘
       │                   │                  │                 │
       │ 1. Click "New Tenant"               │                 │
       │──────────────────>│                  │                 │
       │                   │                  │                 │
       │ 2. Fill form      │                  │                 │
       │  - name           │                  │                 │
       │  - slug           │                  │                 │
       │  - features       │                  │                 │
       │──────────────────>│                  │                 │
       │                   │                  │                 │
       │                   │ 3. POST /admin/tenants             │
       │                   │─────────────────>│                 │
       │                   │                  │                 │
       │                   │                  │ 4. Validate     │
       │                   │                  │    - slug unique│
       │                   │                  │    - required   │
       │                   │                  │                 │
       │                   │                  │ 5. INSERT tenant│
       │                   │                  │────────────────>│
       │                   │                  │<────────────────│
       │                   │                  │                 │
       │                   │                  │ 6. Create Logto │
       │                   │                  │    organization │
       │                   │                  │                 │
       │                   │                  │ 7. Setup defaults│
       │                   │                  │    - tools      │
       │                   │                  │    - persona    │
       │                   │                  │                 │
       │                   │ 8. Return tenant │                 │
       │                   │<─────────────────│                 │
       │                   │                  │                 │
       │ 9. Redirect to    │                  │                 │
       │    tenant detail  │                  │                 │
       │<──────────────────│                  │                 │
       │                   │                  │                 │
```

### 2. Invito Nuovo Utente

```
┌──────────────┐    ┌────────────┐    ┌─────────┐    ┌─────────┐
│ Tenant Admin │    │  Backend   │    │  Logto  │    │  Email  │
└──────┬───────┘    └─────┬──────┘    └────┬────┘    └────┬────┘
       │                   │                │              │
       │ POST /admin/users │                │              │
       │  - email          │                │              │
       │  - name           │                │              │
       │  - role           │                │              │
       │──────────────────>│                │              │
       │                   │                │              │
       │                   │ Create user    │              │
       │                   │───────────────>│              │
       │                   │<───────────────│              │
       │                   │                │              │
       │                   │ Add to org     │              │
       │                   │───────────────>│              │
       │                   │<───────────────│              │
       │                   │                │              │
       │                   │ Create contact │              │
       │                   │ (local DB)     │              │
       │                   │                │              │
       │                   │ Send invite    │              │
       │                   │───────────────────────────────>│
       │                   │                │              │
       │  User created     │                │              │
       │<──────────────────│                │              │
       │                   │                │              │
```

---

## Security Considerations

### API Security Checklist

- [ ] Tutti gli endpoint admin richiedono autenticazione
- [ ] JWT validato con issuer Logto
- [ ] Ruoli verificati prima di ogni operazione
- [ ] RLS attivo su tutte le query
- [ ] Rate limiting su endpoint sensibili
- [ ] Audit log per tutte le operazioni admin
- [ ] CORS configurato correttamente
- [ ] HTTPS obbligatorio

### Audit Logging

Ogni operazione admin deve essere loggata:

```python
@router.delete("/users/{user_id}")
async def delete_user(
    user_id: UUID,
    current_user: AdminUser = Depends(require_tenant_admin),
    session: AsyncSession = Depends(get_tenant_session)
):
    # ... delete logic ...

    # Audit log
    await create_audit_log(
        session=session,
        event_type="USER_DELETED",
        actor_id=current_user.id,
        actor_role=current_user.role,
        tenant_id=current_user.tenant_id,
        target_type="contact",
        target_id=user_id,
        details={
            "user_email": user.email,
            "reason": request.reason
        },
        ip_address=request.client.host,
        user_agent=request.headers.get("user-agent")
    )

    return {"message": "User deleted"}
```

### Data Retention

```python
# Configurazione per tenant
TENANT_DEFAULT_SETTINGS = {
    "data_retention": {
        "conversations_days": 365,      # 1 anno
        "audit_logs_days": 730,         # 2 anni (GDPR)
        "memory_days": 180,             # 6 mesi
        "deleted_data_days": 30         # 30 giorni soft delete
    },
    "gdpr": {
        "auto_anonymize": True,
        "export_format": "json",
        "deletion_confirmation": True
    }
}
```

---

## Roadmap Implementazione

### Fase 1: Backend Core (2 settimane)

- [ ] Setup progetto FastAPI per admin
- [ ] Implementare auth middleware con ruoli
- [ ] CRUD Tenants (Super Admin)
- [ ] CRUD Users (Tenant-scoped)
- [ ] Integrazione Logto per user management
- [ ] Audit logging base

### Fase 2: Frontend Base (2 settimane)

- [ ] Setup progetto React/Vite
- [ ] Layout admin con sidebar
- [ ] Login flow con Logto
- [ ] Tenant selector per Super Admin
- [ ] Lista e dettaglio tenants
- [ ] Lista e form utenti

### Fase 3: Features Avanzate (2 settimane)

- [ ] Dashboard metriche
- [ ] Visualizzazione conversazioni
- [ ] Gestione tool per tenant
- [ ] Audit logs viewer
- [ ] Export dati (GDPR)

### Fase 4: Polish & Security (1 settimana)

- [ ] Review sicurezza completa
- [ ] Performance optimization
- [ ] Documentation
- [ ] Testing E2E

---

*Design Document creato: 2026-02-02*
*Ultimo aggiornamento: 2026-02-02*
*Autore: Claude Code / LEXE Team*
