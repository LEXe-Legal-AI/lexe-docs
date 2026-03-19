---
owner: platform-team
audience: internal
source_of_truth: true
status: active
sensitivity: confidential
last_verified: 2026-03-19
---

# LEXE Security Posture

Internal security architecture documentation for the LEXE platform.

## Scope

**In scope:** Authentication, authorization, tenant isolation, data protection, rate limiting, transport security, GDPR compliance mechanisms, and known gaps.

**Not in scope:** Infrastructure hardening (OS, SSH, firewall rules), Hetzner network policies, SSL certificate management (handled by Traefik + Let's Encrypt), third-party dependency vulnerability scanning.

---

## 1. HMAC Inter-Service Authentication

Internal service-to-service calls (lexe-core to lexe-tools, lexe-core to lexe-memory) are authenticated via HMAC-SHA256 request signing.

Source: `lexe-core/src/lexe_core/gateway/internal_auth.py`

### Headers

| Header | Description |
|--------|-------------|
| `X-Forwarded-By` | Always `lexe-core` (identifies the calling service) |
| `X-Gateway-Ts` | Unix timestamp (float) of the request |
| `X-Gateway-Sig` | HMAC-SHA256 hex digest |
| `X-Tenant-ID` | Tenant UUID for the request |

### Signing Algorithm

```
message = tenant_id + timestamp + method + path
signature = HMAC-SHA256(shared_key, message).hexdigest()
```

### Verification

| Property | Value |
|----------|-------|
| Algorithm | SHA-256 |
| Timestamp tolerance | 30 seconds (`tolerance_s=30.0`) |
| Comparison | Constant-time via `hmac.compare_digest()` |
| Replay protection | Requests older than 30s are rejected |
| Key source | `settings.internal_hmac_key` (environment variable) |

### Activation

HMAC signing is conditional: headers are only added when both `hmac_key` is configured AND `tenant_id` is provided. If `internal_hmac_key` is empty or unset, calls proceed unsigned.

Source: `executor.py:_hmac_headers()` and `internal_auth.py:sign_request()`

### Webhook HMAC (External)

A separate HMAC system exists for incoming webhook verification from external channels (WhatsApp, Telegram, Email, Zammad).

Source: `lexe-core/src/lexe_core/gateway/middleware/hmac_verify.py`

Features:
- Per-channel secrets (configurable via environment)
- Constant-time comparison (`hmac.compare_digest`)
- Timestamp freshness (300s tolerance for webhooks)
- Development mode bypass when no secret configured
- Channel-specific verification: WhatsApp (`X-Hub-Signature-256`), Telegram (`X-Telegram-Bot-Api-Secret-Token`), generic HMAC for others

---

## 2. Row-Level Security (RLS)

PostgreSQL RLS enforces tenant isolation at the database layer as defense-in-depth.

### Helper Functions

Defined in migration `001_admin_panel.sql`:

**`core.fn_get_tenant_id()`** -- Returns the current session tenant UUID from `app.tenant_id` config variable. Set by Python via `SET LOCAL app.tenant_id = :tid`.

**`core.fn_is_superadmin()`** -- Returns boolean from `app.is_superadmin` config variable. Used to allow cross-tenant queries for admin endpoints.

### Policy Pattern

All tenant-scoped tables use the same policy:

```sql
ALTER TABLE core.<table> ENABLE ROW LEVEL SECURITY;
ALTER TABLE core.<table> FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON core.<table>
    USING (tenant_id = core.fn_get_tenant_id() OR core.fn_is_superadmin());
```

`FORCE ROW LEVEL SECURITY` ensures RLS applies even to table owners.

### Protected Tables

RLS is enabled and forced on the following tables (across 10 migration files, 37 total `ENABLE/FORCE` statements):

| Table | Migration | FORCE |
|-------|-----------|-------|
| `core.contacts` | 001 | Yes (pre-existing) |
| `core.conversations` | 001 | Yes (pre-existing) |
| `core.conversation_messages` | 001 | Yes |
| `core.audit_log` | 001 | Yes |
| `core.tenant_llm_models` | 001 | Yes |
| `core.users` | 002 | Yes |
| `core.roles` | 002 | Yes |
| `core.permissions` | 002 | Yes |
| `core.role_permissions` | 002 | Yes |
| `core.tools` | 002 | Yes |
| `core.tenant_feature_flags` | 002 | Yes |
| `core.responder_personas` | 003 | Yes |
| `core.tool_execution_logs` | 006 | Yes |
| `core.evidence_packs` | 008 | Enabled only |
| `core.verification_logs` | 008 | Enabled only |
| `core.manus_sessions` (now `lexorc_sessions`) | 012 | Enabled only |
| `core.legis_usage_patterns` | 013 | Enabled only |
| `core.attachments` | 035 | Yes |
| `core.consent_terms` / related | 036 | Yes |
| `core.conversation_events` | 039 | Yes |
| `core.message_evaluations` | 039 | Yes |
| `core.turn_envelopes` | 039 | Yes |

---

## 3. RBAC (Role-Based Access Control)

Source: `lexe-core/src/lexe_core/auth/rbac.py`

### Roles (7)

| Role | Priority | Scope |
|------|----------|-------|
| `superadmin` | Highest | Cross-tenant, full platform access |
| `admin` | High | Full tenant access, user management |
| `agent` | Medium | Conversation handling, contact management |
| `operator` | Medium | Legacy alias for agent (identical permissions) |
| `user` | Low | Conversation participation, read contacts |
| `viewer` | Lowest | Read-only access |
| (custom) | Tenant-defined | Per-tenant custom roles via DB |

### Permissions (22 codes)

Organized by resource:

| Resource | Actions |
|----------|---------|
| `users` | `manage`, `read`, `create`, `update`, `delete` |
| `roles` | `manage`, `read` |
| `conversations` | `manage`, `read`, `create`, `update`, `delete`, `respond` |
| `contacts` | `manage`, `read`, `create`, `update`, `delete` |
| `settings` | `manage`, `read` |
| `billing` | `manage`, `read` |
| `webhooks` | `manage`, `read` |
| `memory` | `manage`, `read` |
| `tenants` | `manage`, `read` |
| `personas` | `manage`, `read` |
| `system` | `manage` (superadmin only) |

### Enforcement

Three FastAPI dependency decorators:

- `require_permission(code)` -- Single permission check
- `require_any_permission(*codes)` -- OR logic
- `require_all_permissions(*codes)` -- AND logic

Type aliases for common checks (e.g., `CanManageTenants`, `CanReadUsers`) provide clean dependency injection.

### Resolution Order

1. `user.role_ref` (eager-loaded role relationship)
2. `user.role_id` (DB lookup by role UUID)
3. `user.role` (legacy name string lookup, tenant-specific then system)
4. Hardcoded legacy permission map (final fallback)

---

## 4. Tenant Isolation

### Database Layer

- **All data tables** have a `tenant_id UUID NOT NULL` column
- RLS policies enforce row-level filtering (see Section 2)
- Python sets `app.tenant_id` via `SET LOCAL` on every DB session
- Indexes on `tenant_id` for all tenant-scoped tables

### Cache Layer (Valkey)

Key namespacing pattern:
- `lexe:limits:{tenant_id}` -- Tenant plan limits
- `lexe:daily_convs:{tenant_id}:{date}` -- Daily conversation counter
- `lexe:daily_tokens:{tenant_id}:{date}` -- Daily token counter
- `lexe:daily_msgs:{tenant_id}:{date}` -- Daily message counter

### Identity Layer (Logto)

- Tenants map to Logto organizations
- Tenant resolver (`gateway/tenant_resolver.py`) uses sub-based fallback with auto-heal and TTL cache (5 minutes)
- OIDC tokens contain organization claims

### Application Layer

- `conversation_messages.tenant_id` is `NOT NULL` -- must be passed in `save_message()`
- All API routes resolve tenant from authenticated user context
- Admin endpoints use `app.is_superadmin` for cross-tenant access

---

## 5. GDPR Compliance

### Consent System

Feature flag: `ff_consent_required` (default: `false`)

Source: `lexe-core/src/lexe_core/gateway/consent_router.py`, `lexe-core/src/lexe_core/admin/routers/consent_audit.py`

#### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/consent/status` | Check if user accepted current terms |
| `GET` | `/consent/terms` | Get published terms for user's plan type |
| `POST` | `/consent/accept` | Accept terms with privacy toggles |
| `PATCH` | `/consent/preferences` | Update privacy toggles without re-accepting |

#### Privacy Toggles

| Toggle | Default | Description |
|--------|---------|-------------|
| `consent_metrics` | `true` | Allow usage analytics collection |
| `consent_training` | `false` | Allow data use for model training |
| `consent_pseudonymized` | `true` | Allow pseudonymized data processing |

#### Compliance Matrix

| GDPR Article | Status | Implementation |
|-------------|--------|----------------|
| Art. 5 (Principles) | Compliant | Data minimization, purpose limitation |
| Art. 6 (Lawfulness) | Compliant | Consent-based processing |
| Art. 7 (Consent conditions) | Compliant | Granular, revocable consent |
| Art. 13 (Information) | Compliant | Terms content served via API |
| Art. 15 (Right of access) | Partial | Export endpoint (`/conversations/{id}/export`) |
| Art. 17 (Right to erasure) | Manual | No automated deletion pipeline |
| Art. 20 (Data portability) | Compliant | JSON/MD/HTML export formats |
| Art. 21 (Right to object) | Partial | Consent preferences update |

#### Phases

1. **Phase 1** (Complete): DB schema, consent router, admin audit
2. **Phase 2** (P1, backlog): Consent proof with cryptographic signatures
3. **Phase 3** (P2, backlog): Security hardening
4. **Phase 4** (P2, backlog): Full data subject rights (Art. 15-21 automated)

### Audit Trail

- `core.audit_log` table (RLS-protected)
- `core.conversation_events` table with 21 event types
- Consent acceptance recorded with timestamp, IP, user agent

---

## 6. Budget and Rate Limiting

### Per-Turn Budget (BudgetController)

Source: `lexe-core/src/lexe_core/gateway/budget.py`

| Limit | Default | Enforcement |
|-------|---------|-------------|
| Wall clock | 120s | `BudgetExhausted` exception |
| Model calls | 20 per turn | `BudgetExhausted` exception |
| Tool calls | 30 per turn | `BudgetExhausted` exception |

### Per-Tenant Rate Limiting

Enforced via Valkey counters with daily reset.

| Limit Type | HTTP Status | Semantics |
|------------|------------|-----------|
| `max_users` | 403 | Structural -- cannot exceed plan |
| `max_personas` | 403 | Structural -- cannot exceed plan |
| `max_conversations_per_day` | 429 | Temporal -- resets daily |
| `max_messages_per_day` | 429 | Temporal -- resets daily |
| `max_tokens_per_day` | 429 | Temporal -- resets daily (default 50K) |

Plan presets: `free_demo`, `starter`, `professional` with escalating limits.

Source: Tenant Management Board (`project_tmb.md` in memory), Valkey key pattern `lexe:limits:{tenant_id}`

---

## 7. CSP and Security Headers

Configured via Traefik middleware labels in Docker Compose override files.

Source: `lexe-infra/docker-compose.override.prod.yml`, `lexe-infra/docker-compose.override.stage.yml`

### Headers Applied

| Header | Value |
|--------|-------|
| `Content-Security-Policy` | `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self' data:; connect-src 'self' {api_url} {auth_url} {wss_url}; frame-ancestors 'none'` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `X-Frame-Options` | `DENY` (via `frameDeny=true`) |
| `X-Content-Type-Options` | `nosniff` (via `contentTypeNosniff=true`) |
| `X-XSS-Protection` | `1; mode=block` (via `browserXssFilter=true`) |

### Applied To

- `lexe-webchat` (customer-facing chat)
- `lexe-admin` (admin panel)

Separate middleware instances per service with environment-specific `connect-src` domains:
- **Production**: `api.lexe.pro`, `auth.lexe.pro`, `wss://api.lexe.pro`
- **Staging**: `api.stage.lexe.pro`, `auth.stage.lexe.pro`, `wss://api.stage.lexe.pro`

---

## 8. Known Gaps

### Critical

| Gap | Impact | Mitigation | Priority |
|-----|--------|------------|----------|
| **`lexe_app` DB user not deployed** | RLS is effectively inert -- application connects as superuser (`lexe`), bypassing all RLS policies. FORCE ROW LEVEL SECURITY only applies to the table owner, but `lexe` is the database superuser. | Defense-in-depth only. Application-level tenant filtering is the primary enforcement. | P1 |

### High

| Gap | Impact | Mitigation | Priority |
|-----|--------|------------|----------|
| **HMAC disabled by default** | `internal_hmac_key` not set in production. Service-to-service calls are unauthenticated. Any container on the Docker network can call internal APIs. | Docker network isolation (`lexe_internal`) limits exposure. Only containers on the network can reach internal services. | P1 |
| **CSP `style-src 'unsafe-inline'`** | Allows inline styles, which weakens CSP protection against style-based attacks. | Required by current frontend framework (React + styled components). Low actual risk since `script-src` is strict. | P3 |
| **Export endpoint access levels** | `/conversations/{id}/export` has 3 access levels but no automated PII redaction for non-owner access. | Manual review process for data exports. | P2 |
| **Tenant resolver fallback** | Sub-based fallback with auto-heal could theoretically assign a user to a wrong tenant if Logto data is corrupted. | TTL cache (5 min) limits blast radius. Auto-heal logs all reassignments. | P2 |

### Medium

| Gap | Impact | Mitigation | Priority |
|-----|--------|------------|----------|
| **Consent FF default `false`** | GDPR consent collection is not enforced unless explicitly enabled per tenant. New tenants start without consent gates. | Feature flag can be enabled per tenant. Consent tables and endpoints exist, just gated. | P2 |
| **GDPR Art. 17 manual deletion** | No automated "right to erasure" pipeline. Data deletion requires manual DB operations. | Documented manual procedure. Backlog item for automated deletion. | P2 |
| **localStorage token storage** | Logto SDK stores access tokens in browser localStorage, vulnerable to XSS. | CSP `script-src 'self'` mitigates. No `eval()` or dynamic script loading. DOMPurify for user content. | P2 |
| **Some tables ENABLE without FORCE** | `evidence_packs`, `verification_logs`, `manus_sessions`, `legis_usage_patterns` have `ENABLE ROW LEVEL SECURITY` but not `FORCE`. Table owner bypasses RLS. | These tables are less sensitive (operational logs, not PII). Will be fixed in next migration. | P3 |
| **CORS allow_headers manual list** | Starlette returns 400 if request headers are not in `allow_headers`. `PATCH` must be explicitly in `allow_methods`. | Documented pattern in MEMORY.md. Integration tests cover common header combinations. | P3 |

---

*Last updated: 2026-03-19*
