---
owner: platform-team
audience: internal
source_of_truth: true
status: active
sensitivity: internal
last_verified: 2026-03-19
---

# Multitenancy Reference -- LEXE Platform

> Complete reference for tenant model, provisioning, limits enforcement, RBAC, and user management.

---

## 1. Tenant Structure

Each tenant represents an isolated legal workspace (law firm, corporate legal department, solo practitioner).

### Schema: `core.tenants`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PK, default `gen_random_uuid()` | Tenant identifier |
| `name` | TEXT | NOT NULL | Display name (e.g. "Studio Legale Rossi") |
| `slug` | TEXT | UNIQUE, NOT NULL | URL-safe identifier (e.g. `studio-rossi`) |
| `settings` | JSONB | DEFAULT `'{}'` | Tenant-level configuration blob |
| `is_active` | BOOLEAN | DEFAULT `true` | Soft-delete / suspension flag |
| `logto_organization_id` | TEXT | NULLABLE | Mapped Logto organization for OIDC |
| `default_persona_id` | UUID | FK `core.responder_personas(id)` | Default system prompt persona |
| `litellm_virtual_key` | TEXT | NULLABLE | LiteLLM API key for cost tracking |
| `litellm_team_id` | TEXT | NULLABLE | LiteLLM team for budget grouping |
| `litellm_budget_monthly` | NUMERIC | NULLABLE | Monthly LLM spend cap (USD) |
| `created_at` | TIMESTAMPTZ | DEFAULT `now()` | Creation timestamp |
| `updated_at` | TIMESTAMPTZ | DEFAULT `now()` | Last modification |

### Settings JSONB Structure

```jsonc
{
  "plan": "free_demo",           // Plan preset key
  "max_users": 5,                // Structural limit
  "max_personas": 3,             // Structural limit
  "max_contacts": 10,            // Structural limit
  "max_conversations_per_day": 20,  // Temporal limit
  "max_tokens_per_day": 50000,      // Temporal limit
  "max_messages_per_conversation": 50, // Temporal limit
  "welcome_email_sent": true,
  "branding": {
    "logo_url": null,
    "primary_color": "#0A1628"
  }
}
```

### RLS Isolation

Row Level Security is active on all tenant-scoped tables. Policies use `fn_get_tenant_id()` (reads `app.current_tenant_id` from the session variable set per-request by lexe-core). Superadmin bypass via `fn_is_superadmin()`.

Protected tables: `tenants`, `contacts`, `conversations`, `conversation_messages`, `responder_personas`, `user_consents`.

---

## 2. Plan Presets

Three predefined plan tiers. Stored as constants in the backend and applied during provisioning.

| Plan | `max_users` | `max_personas` | `max_contacts` | `max_conversations_per_day` | `max_tokens_per_day` | `max_messages_per_conversation` |
|------|-------------|----------------|-----------------|----------------------------|---------------------|-------------------------------|
| **free_demo** | 5 | 3 | 10 | 20 | 50,000 | 50 |
| **starter** | 10 | 10 | 50 | 100 | 200,000 | 100 |
| **professional** | 50 | 25 | 500 | 500 | 1,000,000 | 200 |

Plan presets are applied at provisioning time and written into `settings` JSONB. They can be overridden per-tenant via the admin Tenant Editor.

---

## 3. Provisioning Flow

### Endpoint

```
POST /api/v1/admin/tenants/provision
```

**Auth**: AdminUser with `CanManageTenants` permission.

### Request Body

```json
{
  "name": "Studio Legale Rossi",
  "slug": "studio-rossi",
  "plan": "starter",
  "admin_email": "avv.rossi@studio-rossi.it",
  "admin_name": "Avv. Marco Rossi"
}
```

### Provisioning Saga (Non-Atomic)

The provisioning saga executes the following steps sequentially. Each step is idempotent for retry safety. Partial failures leave the tenant in a partially-provisioned state that can be completed manually.

```
Step 1: INSERT into core.tenants (slug uniqueness enforced)
    |
Step 2: Create Logto Organization via Management API (M2M token)
    |  -> stores logto_organization_id on tenant
    |
Step 3: Create admin user in Logto (email + temporary password)
    |  -> password expires in 72 hours
    |
Step 4: Assign admin user to Logto Organization with admin role
    |
Step 5: INSERT into core.contacts (admin user record)
    |
Step 6: Apply plan preset limits to tenant settings JSONB
    |
Step 7: Send welcome email
         -> Navy #0A1628 + Cyan #06B6D4 branding
         -> Contains: login URL, temporary password, 72h expiry notice
         -> Template: HTML with LEXE branding
```

### Idempotency

If provisioning fails mid-saga:
- The tenant row exists with `is_active=true` but may lack `logto_organization_id`
- Re-calling provision with the same slug returns a conflict error
- Admin must complete provisioning manually or delete and retry

---

## 4. Limits Enforcement

Two categories of limits with distinct HTTP semantics.

### Structural Limits (403 Forbidden)

These limits are checked before resource creation. Exceeding them is a hard block.

| Limit | Check Point | Error Response |
|-------|-------------|----------------|
| `max_users` | User invite endpoint | `403 HTTP_403 Limit exceeded: max_users` |
| `max_personas` | Persona creation | `403 HTTP_403 Limit exceeded: max_personas` |
| `max_contacts` | Contact creation | `403 HTTP_403 Limit exceeded: max_contacts` |

**Implementation**: Query `COUNT(*)` from the relevant table filtered by `tenant_id`, compare against the limit in `settings` JSONB.

### Temporal Limits (429 Too Many Requests)

These limits reset daily (UTC midnight) or per-conversation. Tracked in Valkey for performance.

| Limit | Valkey Key Pattern | TTL | Error Response |
|-------|-------------------|-----|----------------|
| `max_conversations_per_day` | `lexe:daily_convs:{tenant_id}:{YYYY-MM-DD}` | 48h | `429 Rate limit: daily conversations` |
| `max_tokens_per_day` | `lexe:daily_tokens:{tenant_id}:{YYYY-MM-DD}` | 48h | `429 Rate limit: daily tokens` |
| `max_messages_per_conversation` | Checked via DB count | -- | `429 Rate limit: messages per conversation` |

**Valkey Key Structure**:

```
lexe:limits:{tenant_id}          -> HASH of current limit values (cached from DB, TTL 5min)
lexe:daily_convs:{tid}:{date}    -> INCR on each new conversation
lexe:daily_tokens:{tid}:{date}   -> INCRBY on each LLM response (token count from LiteLLM)
```

**Accounting pattern**: Pre-check before operation, post-accounting after completion. Token counts are incremented after the LLM response is received (not estimated upfront).

---

## 5. Auth Flow

### Customer Authentication (Logto OIDC)

```
Browser -> Logto Authorization Endpoint (PKCE)
    -> User authenticates (email OTP or password)
    -> Logto issues JWT access token + refresh token
    -> Browser stores tokens in LocalStorage
    -> Each API request: Authorization: Bearer <JWT>
```

| Parameter | Value |
|-----------|-------|
| **Protocol** | OIDC Authorization Code + PKCE |
| **JWT signing** | HS256 |
| **Access token expiry** | 60 minutes |
| **Refresh token expiry** | 7 days |
| **Token audience** | `https://api.lexe.pro` (prod) / `https://api.stage.lexe.pro` (stage) |
| **Organization claim** | `organization_id` in JWT payload |
| **Issuer** | `https://auth.lexe.pro/oidc` (prod) / `https://auth.stage.lexe.pro/oidc` (stage) |

### Admin Authentication (Internal)

Admins authenticate via the same Logto instance but through the `lexe-admin-spa` OIDC application. The JWT contains admin-specific scopes and the user is resolved against `core.contacts` with an admin role.

### Token Validation (lexe-core)

1. Verify JWT signature (HS256 with Logto secret)
2. Check `exp` claim (reject expired tokens)
3. Check `aud` claim (must match API resource indicator)
4. Extract `sub` (Logto user ID) and `organization_id`
5. Resolve tenant via `gateway/tenant_resolver.py`:
   - Primary: match `organization_id` against `core.tenants.logto_organization_id`
   - Fallback: `sub`-based lookup against `core.contacts.external_user_id`
   - Auto-heal: if fallback succeeds, update the tenant's `logto_organization_id`
   - Cache: TTL 5 minutes in process memory

---

## 6. RBAC

### Roles (7)

| Role | Scope | Description |
|------|-------|-------------|
| `superadmin` | Global | Full platform access, cross-tenant operations |
| `admin` | Tenant | Manage tenant settings, users, personas, limits |
| `operator` | Tenant | Manage conversations, view reports, moderate |
| `viewer` | Tenant | Read-only access to conversations and reports |
| `developer` | Tenant | API access, tool configuration, feature flags |
| `agent` | Tenant | AI agent role (internal use for M2M agent calls) |
| `user` | Tenant | Standard end-user (chat, profile, consent) |

### Permissions (22)

Permissions are checked via `require_permission()` decorators on route handlers using the `_perm: CanXxx` parameter pattern (NOT `dependencies=[Depends()]` -- see gotchas).

| Permission | Roles |
|------------|-------|
| `CanManageTenants` | superadmin |
| `CanViewTenants` | superadmin, admin |
| `CanManageUsers` | superadmin, admin |
| `CanViewUsers` | superadmin, admin, operator |
| `CanManagePersonas` | superadmin, admin |
| `CanViewPersonas` | superadmin, admin, operator, viewer |
| `CanManageModels` | superadmin, admin, developer |
| `CanViewModels` | superadmin, admin, developer, operator |
| `CanManageTools` | superadmin, admin, developer |
| `CanViewTools` | superadmin, admin, developer, operator |
| `CanManageFeatureFlags` | superadmin, admin, developer |
| `CanViewFeatureFlags` | superadmin, admin, developer |
| `CanManageConversations` | superadmin, admin, operator |
| `CanViewConversations` | superadmin, admin, operator, viewer |
| `CanDeleteConversations` | superadmin, admin |
| `CanExportData` | superadmin, admin |
| `CanViewReports` | superadmin, admin, operator |
| `CanManageContacts` | superadmin, admin |
| `CanViewContacts` | superadmin, admin, operator |
| `CanViewAuditLog` | superadmin, admin |
| `CanManageSettings` | superadmin, admin |
| `CanViewSettings` | superadmin, admin, operator |

### RBAC Double-Wrap Gotcha

The permission check pattern uses a parameter-based dependency injection:

```python
# CORRECT
@router.get("/tenants")
async def list_tenants(_perm: CanManageTenants):
    ...

# WRONG -- double-wraps and breaks FastAPI dependency resolution
@router.get("/tenants", dependencies=[Depends(require_permission(CanManageTenants))])
async def list_tenants():
    ...
```

---

## 7. Feature Flags per Tenant

### Schema: `core.tenant_feature_flags`

| Column | Type | Constraints |
|--------|------|-------------|
| `id` | UUID | PK |
| `tenant_id` | UUID | FK `core.tenants(id)`, NOT NULL |
| `flag_name` | TEXT | NOT NULL |
| `enabled` | BOOLEAN | DEFAULT `false` |
| `created_at` | TIMESTAMPTZ | DEFAULT `now()` |

**Unique constraint**: `(tenant_id, flag_name)` -- one override per flag per tenant.

### Resolution Order

1. Check `core.tenant_feature_flags` for tenant-specific override
2. If no override, use global feature flag default from `config.py` / environment variable
3. Environment variable pattern: `LEXE_FF_{FLAG_NAME}` (uppercase)

### Active Feature Flags

| Flag | Default | Description |
|------|---------|-------------|
| `ff_legis_agent` | `true` | LEGIS legal research pipeline |
| `ff_lexorc_enabled` | `false` | LEXORC advanced orchestrator (deprecated) |
| `ff_orchestration_v2` | `false` | Orchestration v2 3-layer architecture |
| `ff_multi_agent_research` | `false` | Multi-agent parallel research ("La Bomba") |
| `ff_consent_required` | `false` | GDPR consent overlay requirement |
| `ff_super_tool` | `false` | Super Tool agent (deprecated, files retained) |

---

## 8. User Management

### Invite Flow

```
Admin clicks "Invite User" in Tenant Editor
    |
    v
POST /api/v1/admin/tenants/{tid}/users/invite
    body: { email, name, role }
    |
    v
Create user in Logto (Management API, M2M token)
    -> Temporary password (72h expiry)
    |
    v
Assign user to Logto Organization with role
    |
    v
INSERT into core.contacts (external_user_id = Logto sub)
    |
    v
Send welcome email
    -> Navy #0A1628 background + Cyan #06B6D4 accents
    -> Login URL, temporary password, expiry notice
    -> LEXE branding with logo
```

### User Lifecycle

| Action | Endpoint | Effect |
|--------|----------|--------|
| **Invite** | `POST .../users/invite` | Create in Logto + DB, send email |
| **Deactivate** | `PATCH .../users/{uid}/deactivate` | Suspend in Logto, set `is_active=false` in contacts |
| **Reactivate** | `PATCH .../users/{uid}/reactivate` | Unsuspend in Logto, set `is_active=true` |
| **Change role** | `PATCH .../users/{uid}/role` | Update Logto org role + DB role |
| **Delete** | Not implemented | GDPR art. 17 handled via manual DPO process |

### Session Management

Sessions are managed by Logto. lexe-core does not maintain server-side sessions. Token refresh is handled client-side via the Logto SDK. Valkey is used for rate-limit counters, not session storage.

---

## 9. Tenant Editor (Admin Panel)

The Tenant Editor in `lexe-admin` provides a 6-tab interface for managing individual tenants.

| Tab | Contents |
|-----|----------|
| **General** | Name, slug (read-only after creation), is_active toggle, Logto org ID, creation date |
| **Settings** | Plan selection dropdown, custom settings JSONB editor |
| **Limits** | Editable fields for all structural and temporal limits, current usage display with progress bars |
| **Models** | Override model aliases per tenant (`tenant_model_roles`), temperature/max_tokens per role |
| **Tools** | Enable/disable specific tools per tenant, tool execution log viewer |
| **Feature Flags** | Toggle per-tenant feature flag overrides, shows global default vs. tenant override |

### Usage Bars

The Limits tab displays real-time usage bars for temporal limits:
- Conversations today: `N / max_conversations_per_day`
- Tokens today: `N / max_tokens_per_day`
- Data sourced from Valkey counters + Prometheus metrics

---

## 10. Tenant Resolution

### File: `gateway/tenant_resolver.py`

The tenant resolver maps incoming JWT claims to a `core.tenants` row. It is called on every authenticated request.

### Resolution Algorithm

```
1. Extract organization_id from JWT
2. Cache lookup: tenant_cache[organization_id]
   -> If hit and TTL < 5min: return cached tenant
3. DB lookup: SELECT * FROM core.tenants WHERE logto_organization_id = $1
   -> If found: cache and return
4. Fallback: Extract sub from JWT
   -> DB lookup: SELECT t.* FROM core.tenants t
      JOIN core.contacts c ON c.tenant_id = t.id
      WHERE c.external_user_id = $1
   -> If found:
      a. Auto-heal: UPDATE core.tenants SET logto_organization_id = $org_id WHERE id = $tid
      b. Cache and return
5. If all lookups fail: raise 401 Unauthorized
```

### Cache Properties

| Property | Value |
|----------|-------|
| **Storage** | In-process Python dict |
| **TTL** | 5 minutes |
| **Eviction** | TTL-based, no LRU |
| **Invalidation** | Automatic on tenant update via admin API |

---

*Last updated: 2026-03-19*
