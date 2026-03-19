---
owner: platform-team
audience: internal
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: quarterly
sensitivity: confidential
last_verified: 2026-03-19
---

# Data Flows -- GDPR Compliance Reference

> Maps every category of personal data processed by LEXE, where it flows,
> how long it is retained, and what rights the data subject can exercise.
> Reg. UE 2016/679 + D.Lgs. 196/2003 (mod. D.Lgs. 101/2018).

---

## Scope

This document covers:

- All categories of personal data processed by the LEXE platform.
- Data flow maps: which system handles which data, in which direction.
- Retention policies per data category and memory tier.
- The consent system (4 phases, 3 toggles, audit trail).
- Data subject rights implementation: erasure (Art. 17), portability (Art. 20), access (Art. 15).
- Pseudonymization pipeline.
- Third-party data sharing and processing agreements.
- Known compliance gaps and remediation status.

## Non-Scope

- System architecture details (see `SYSTEM-BOUNDARIES.md`, `ARCHITECTURE-V2.md`).
- Confidence scoring (see `CONFIDENCE-PIPELINE.md`).
- Legal content of the terms of service (see `gdpr-consent-system.md` for full terms text).
- Security posture and hardening (see `SECURITY-POSTURE.md`).

---

## 1. Data Categories

| Category | Examples | Legal Basis (Art. 6) | Storage Location | RLS Protected |
|----------|---------|---------------------|-----------------|---------------|
| **User Identity** | Email, first name, last name, Logto sub | Contract (6.1.b) | lexe-logto (Logto DB), `core.contacts` | Yes (tenant_id) |
| **Authentication Data** | Password hash, JWT tokens, session data | Contract (6.1.b) | lexe-logto (password), lexe-valkey (sessions) | N/A (Logto-managed) |
| **Conversation Content** | User messages, AI responses, follow-ups | Contract (6.1.b) | `core.conversation_messages` | Yes (tenant_id) |
| **Legal Queries** | Questions about specific laws, contracts, cases | Contract (6.1.b) | `core.conversation_messages` | Yes (tenant_id) |
| **Evaluations** | User star ratings (1-5) per AI message | Consent (6.1.a) | `core.message_evaluations` | Yes (tenant_id) |
| **Memory Facts** | Extracted user preferences, legal context, profile | Contract (6.1.b) | `memory.profile_facts`, `memory.memories`, `memory.semantic_vectors` | Yes (tenant_id) |
| **Consent Records** | Terms acceptance, toggle preferences, IP, User-Agent | Legal obligation (6.1.c) | `core.user_consents`, `core.consent_terms` | Yes (tenant_id) |
| **Audit Trail** | Admin actions, system events, trace hashes | Legitimate interest (6.1.f) | `core.audit_log`, `core.conversation_events` | Yes (tenant_id) |
| **Anonymized Metrics** | Pipeline duration, confidence scores, tool counts | Consent (6.1.a, toggle `consent_metrics`) | Prometheus TSDB (no PII) | N/A (anonymous) |
| **Document Attachments** | Uploaded DOCX/PDF content (extracted text) | Contract (6.1.b) | `core.attachments` | Yes (tenant_id) |
| **Technical Metadata** | IP address (consent only), User-Agent, request timestamps | Legal obligation (6.1.c, for consent proof) | `core.user_consents` (IP/UA), application logs (ephemeral) | Yes (tenant_id) |

---

## 2. Data Flow Map

### Primary Data Flow

```
                                EXTERNAL
                                   |
                          [1] HTTPS + JWT
                                   |
                                   v
                           +---------------+
                           |   Traefik     |  TLS termination
                           +-------+-------+
                                   |
                    +--------------+----------------+
                    |              |                 |
                    v              v                 v
             lexe-webchat    lexe-admin        lexe-logto
             (SPA, no PII   (SPA, no PII      (OIDC provider)
              on server)     on server)              |
                    |              |           [2] JWT issued
                    +--------------+                 |
                           |                         |
                    [3] API calls (Bearer JWT)       |
                           |                         |
                           v                         |
                    +---------------+                |
                    |  lexe-core    |<---------------+
                    |  (gateway)    |   [4] Logto Mgmt API
                    +---+---+---+--+      (user provisioning)
                        |   |   |
           +------------+   |   +-------------+
           |                |                 |
    [5] Memory ops   [6] Tool calls    [7] LLM inference
           |                |                 |
           v                v                 v
    lexe-memory      lexe-tools-it      LiteLLM
           |                |                 |
           v                v                 v
    lexe-postgres      lexe-max       LLM Providers
    (memory schema)   (kb schema,     (Gemini, OpenAI)
           |          NO user PII)         |
           v                          [8] Streaming only
    lexe-valkey                       NO persistence
    (L0 cache,                        at provider
     counters)
```

### Data Classification per Flow

| Flow | Label | Data Transmitted | PII Present | Encrypted | Retention at Destination |
|------|-------|-----------------|-------------|-----------|--------------------------|
| [1] | Browser -> Traefik | HTTP request + JWT | Yes (JWT contains sub, email) | TLS 1.2+ | None (proxy) |
| [2] | Logto -> Browser | JWT token, user profile | Yes | TLS | Browser sessionStorage |
| [3] | Browser -> lexe-core | Chat messages, commands, evaluations | Yes | TLS | Persisted in DB |
| [4] | lexe-core -> Logto | User provisioning, org management | Yes (email, name) | Internal HTTP | Logto DB (self-hosted) |
| [5] | lexe-core -> lexe-memory | Contact ID, conversation context, extracted facts | Yes (indirect: user facts) | Internal HTTP | memory schema in lexe-postgres |
| [6] | lexe-core -> lexe-tools | Legal query terms, tool parameters | No (legal refs only) | Internal HTTP | None (stateless) |
| [7] | lexe-core -> LiteLLM -> LLM | System prompt + conversation context + tool results | Yes (conversation content) | TLS to provider | **None** (streaming, no persistence per DPA) |
| [8] | LLM -> LiteLLM -> lexe-core | Generated response tokens | No PII (AI-generated) | TLS | Persisted as AI message in DB |

---

## 3. Retention Policies

### Memory Tiers

| Tier | Name | TTL | Storage | Content | Purge Mechanism |
|------|------|-----|---------|---------|-----------------|
| **L0** | Working Memory | 24 hours | lexe-valkey | Current conversation context, active tool results | Valkey TTL auto-expiry |
| **L1** | Short-term Memory | 7 days | `memory.memories` (type=episodic) | Recent conversation summaries, extracted entities | Background cleanup job |
| **L2** | Medium-term Memory | 90 days | `memory.memories` (type=semantic) | Consolidated facts, patterns, user preferences | Background cleanup job |
| **L3** | Long-term Memory | Permanent (until forget) | `memory.profile_facts` | Brainprint profile: stable user preferences, legal domain context | Manual forget or Art. 17 request |
| **L4** | Archival | Permanent (until forget) | `memory.semantic_vectors_v2` | Semantic embeddings for RAG retrieval | Cascade delete with source fact |

### Other Data Categories

| Data | Retention | Justification |
|------|-----------|---------------|
| Conversations + messages | Indefinite (until account deletion) | Contract performance; user expects chat history |
| Consent records | 5 years after last interaction | Legal obligation: proof of consent (Art. 7.1) |
| Audit log | 2 years | Legitimate interest: security, compliance |
| Conversation events | 1 year | Legitimate interest: quality monitoring |
| Message evaluations | 1 year | Consent (metrics toggle) |
| User identity (Logto) | Until account deletion + 12 months | Privacy policy commitment |
| Application logs | 30 days (logrotate) | Operational necessity |
| Prometheus metrics | 15 days (TSDB retention) | No PII; operational |
| Attachments | Conversation lifetime | Tied to conversation retention |

---

## 4. Consent System

### Overview

The LEXE consent system implements 4 GDPR compliance phases, all COMPLETE as of 2026-03-19.

| Phase | Description | Status | Implementation |
|-------|-----------|--------|----------------|
| **1. Informativa** | Terms of Service + Privacy Policy, versioned per plan type (free/paid/demo) | DONE | `core.consent_terms` table, `body_html` field |
| **2. Prova Consenso** | Documented proof: IP, User-Agent, timestamp, audit trail export | DONE | `core.user_consents` table, admin CSV export |
| **3. Disclaimer** | AI disclaimer per Art. 1229 c.c., explicit no-training statement, DPO contact, 12-month retention | DONE | Embedded in terms v1.0 for both free and paid plans |
| **4. Diritti Interessato** | Data export (Art. 15/20), account deletion request (Art. 17), privacy info panel | DONE | PrivacyTab in webchat, export endpoints |

### Consent Toggles

| Toggle | Field | Required | Default | What It Controls |
|--------|-------|----------|---------|-----------------|
| **Terms of Service** | `terms_accepted` | **Mandatory** | -- | Blocks platform access if not accepted |
| **Metrics** | `consent_metrics` | Optional | ON | Aggregate anonymous usage metrics (pipeline stats, tool counts) |
| **Pseudonymized Analysis** | `consent_pseudonymized` | Optional | ON | KPI analysis with irreversible hash (no re-identification) |

Note: `consent_training` field exists in DB but is always `false` and not exposed in UI. LEXE explicitly states "NO training AI terze parti" in the privacy policy.

### Feature Flag

```
LEXE_FF_CONSENT_REQUIRED=true
```

When `false`: `GET /consent/status` returns `{accepted: true}` immediately -- no overlay shown.

### Consent Endpoints

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/consent/status` | GET | Customer JWT | Check if user accepted current terms version |
| `/consent/terms` | GET | Customer JWT | Retrieve terms HTML + existing preferences |
| `/consent/accept` | POST | Customer JWT | Accept terms, capture IP/UA proof |
| `/consent/preferences` | PATCH | Customer JWT | Update optional toggles without re-accepting terms |
| `/admin/consent-audit` | GET | Admin JWT + `settings.read` | List consent records (JSON, tenant-filtered) |
| `/admin/consent-audit/export` | GET | Admin JWT + `settings.read` | Download CSV for DPO audit |

### Audit Trail

Every consent action is recorded with:
- `user_sub` (Logto identifier)
- `tenant_id` (multi-tenant isolation)
- `terms_id` (which version was accepted)
- `accepted_at` (server-side UTC timestamp)
- `ip_address` (from `request.client.host`)
- `user_agent` (from HTTP header, truncated to 500 chars)
- All three toggle values at time of acceptance

---

## 5. Right to Erasure (Art. 17)

### Current Implementation

| Component | Mechanism | Status |
|-----------|----------|--------|
| Memory facts | `POST /memory/forget` | Implemented (soft delete, hard purge after retention) |
| Conversations | Manual deletion by admin | Admin can delete via admin panel |
| User identity (Logto) | Manual via Logto admin panel | Requires admin action |
| Consent records | Retained for legal obligation (Art. 17.3.e) | Exempt from erasure |
| Full account deletion | Email to `privacy@lexe.pro` | **Manual process** |

### Erasure Workflow (Current)

```
User Request
    |
    v
Email to privacy@lexe.pro
    |
    v
DPO verifies identity (reply to registered email)
    |
    v
Admin actions:
  1. DELETE conversations (core.conversation_messages, core.conversations)
  2. DELETE memory (memory.memories, memory.profile_facts, memory.semantic_vectors)
  3. DELETE evaluations (core.message_evaluations)
  4. DELETE attachments (core.attachments)
  5. DELETE user from Logto (Management API)
  6. DELETE contact from core.contacts
  7. RETAIN consent records (legal obligation)
    |
    v
Confirmation email to user within 30 days (Art. 12.3)
```

### Known Gaps

- **Manual process**: No self-service "delete my account" button. Requires email to DPO.
- **No automated cascade**: Admin must delete across multiple tables manually.
- **Soft delete timing**: Memory `forget` is soft delete; hard purge timing not enforced by cron.
- **Backup retention**: Database backups may retain deleted data for the backup retention window.

---

## 6. Data Portability (Art. 20)

### Available Export Endpoints

| Endpoint | Format | Content | Auth |
|----------|--------|---------|------|
| `GET /conversations/{id}/export?format=json` | JSON | Full conversation with messages, metadata, evaluations | Customer JWT (own conversations) |
| `GET /conversations/{id}/export?format=md` | Markdown | Human-readable conversation transcript | Customer JWT |
| `GET /conversations/{id}/export?format=html` | HTML | Branded standalone HTML (navy/gold/green theme) | Customer JWT |
| `GET /memory/contact/{id}/export` | JSON/CSV | Memory facts, profile, preferences | Customer JWT (own data) |

### Access Levels

The export endpoint supports 3 access levels:
1. **Customer**: Own conversations only (JWT-enforced).
2. **Admin**: Tenant-scoped conversations (RBAC `conversations.read`).
3. **Superadmin**: Any conversation (cross-tenant).

### PrivacyTab (Frontend)

The webchat `PrivacyTab` component provides:
- "Esporta i tuoi dati" button -- downloads JSON of all conversations (Art. 15/20).
- GDPR informational accordions: data collected, purposes, legal basis, retention, rights.
- "Cancellazione account" -- mailto link to `privacy@lexe.pro` with pre-filled subject.
- DPO contact information.
- Link to full privacy policy at `https://lexe.pro/privacy`.

---

## 7. Pseudonymization

### lexe-privacy Engine

| Component | Technology | Purpose |
|-----------|-----------|---------|
| PII Detection | spaCy (Italian model) + custom regex engines | Identify names, fiscal codes, addresses, phone numbers in conversation text |
| Pseudonymization | Irreversible hash (SHA-256 + per-tenant salt) | Replace PII with hash tokens for analytics |
| Consent Gate | `consent_pseudonymized` toggle | Only process if user opted in |

### Data Flow

```
Conversation message
    |
    | (if consent_pseudonymized = true)
    v
lexe-privacy PII detection
    |
    v
Identified entities: [NAME: "Mario Rossi", CF: "RSSMRA80A01H501Z"]
    |
    v
Hash replacement: [NAME: "h_a3f2...c1", CF: "h_7b9e...d4"]
    |
    v
Pseudonymized record stored for KPI analysis
```

**Guarantees:**
- Hash is irreversible (no re-identification possible).
- Salt is per-tenant (cross-tenant correlation impossible).
- Raw PII never leaves the tenant boundary during pseudonymization.
- If `consent_pseudonymized = false`, no processing occurs.

---

## 8. Audit Trail

### Tables

| Table | Schema | Content | Retention |
|-------|--------|---------|-----------|
| `core.audit_log` | `core` | Admin actions: CRUD operations, config changes, user management | 2 years |
| `core.conversation_events` | `core` | Pipeline events: tool calls, synthesis, verification, SSE events (21 event types) | 1 year |
| `core.message_evaluations` | `core` | User quality ratings (1-5 stars) per AI response | 1 year |
| `core.user_consents` | `core` | Consent acceptance records with IP/UA proof | 5 years |
| `memory.audit_log` | `memory` | Memory operations: create, update, forget, delta tracking | 2 years |

All audit tables are RLS-protected by `tenant_id`.

---

## 9. Third-Party Data Sharing

### LLM Providers

| Provider | Data Sent | Persistence | DPA Status | Notes |
|----------|----------|-------------|------------|-------|
| **Google (Gemini)** | Conversation context, system prompts, tool results | **None** (streaming, API mode) | Covered by Google Cloud DPA | Primary provider for all 6 model aliases |
| **OpenAI** | Embedding input text (for `text-embedding-3-small`); conversation context (for `gpt-5-mini`) | **None** (API mode, data not used for training per API ToS) | Covered by OpenAI API DPA | gpt-5-mini is secondary/benchmark model only |
| **Anthropic** | Conversation context (for `sonnet-4-6`) | **None** (API mode) | Covered by Anthropic API terms | Available as fallback; not primary |
| **DeepSeek** | Conversation context (for `deepseek-v3.2`) | Per their API terms | **Review needed** | Available as fallback; data may transit China |

**Key principle:** No user identity data (email, name, tenant) is sent to LLM providers. Only conversation content and system prompts are transmitted via LiteLLM streaming. The terms of service explicitly state: "NO training AI terze parti".

### Infrastructure

| Provider | Data Stored | Location | DPA Status |
|----------|-----------|----------|------------|
| **Hetzner** | All platform data (databases, logs, caches) | EU (Germany: Falkenstein/Nuremberg) | Standard DPA (Art. 28) |

### No Other Third-Party Sharing

- No analytics services (Google Analytics, Mixpanel, etc.).
- No advertising networks.
- No data brokers.
- Cookies: technical only; analytics subordinated to `consent_metrics` toggle.

---

## 10. Known Gaps and Remediation

| Gap | GDPR Article | Severity | Status | Remediation |
|-----|-------------|----------|--------|-------------|
| **Art. 17 deletion is manual (email-based)** | Art. 17 | HIGH | Open | Planned: self-service delete button + automated cascade across all tables |
| **Art. 16 rectification minimal** | Art. 16 | MEDIUM | Open | User can update name via Logto; no UI for correcting conversation content |
| **lexe_app DB user not deployed** | Art. 25 (data protection by design) | LOW | Open | Application currently uses `lexe` superuser; RLS is defense-in-depth only |
| **No automated hard purge** | Art. 17 | MEDIUM | Open | Memory `forget` is soft delete; need cron job to hard-purge after retention window |
| **DeepSeek DPA review** | Art. 46 (international transfers) | MEDIUM | Open | DeepSeek is backup model; review DPA and data residency before promoting to primary |
| **Backup data retention** | Art. 17 | LOW | Accepted risk | DB backups may contain deleted user data; retention window is 7 days (Hetzner snapshots) |
| **No DPO formally appointed** | Art. 37 | LOW | Open | DPO email exists (`dpo@itconsultingsrl.com`); formal appointment letter pending |

### Completed Remediations (2026-03-19)

- Phase 1: Informativa -- terms versioned per plan, `body_html` served via API.
- Phase 2: Prova consenso -- IP/UA capture, admin CSV export for DPO audit.
- Phase 3: Disclaimer -- Art. 1229 c.c. disclaimer, no-training statement, DPO contact, 12-month retention in both free and paid terms.
- Phase 4: Diritti -- export endpoints (JSON/MD/HTML), PrivacyTab with GDPR accordions, deletion request flow.
- Consent audit admin endpoints with tenant isolation.
- RLS on `user_consents` table (migration 036).

---

## Regulatory References

| Regulation | Articles | Relevance |
|-----------|----------|-----------|
| **Reg. UE 2016/679 (GDPR)** | Art. 5 (principles), 6 (lawfulness), 7 (consent conditions), 12-22 (data subject rights), 13 (information), 25 (data protection by design), 28 (processor), 33-34 (breach notification), 37 (DPO), 46 (international transfers) | Primary regulation |
| **D.Lgs. 196/2003** (mod. D.Lgs. 101/2018) | Italian implementation of GDPR | National specifics |
| **Art. 1229 c.c.** | Limitation of liability clauses | AI disclaimer in terms |
| **AI Act (Reg. UE 2024/1689)** | Art. 52 (transparency for AI systems) | Future: AI system classification and disclosure requirements |

---

## File References

| File | Role |
|------|------|
| `lexe-core/migrations/036_consent_terms.sql` | DB schema + seed for consent |
| `lexe-core/src/lexe_core/gateway/consent_router.py` | Customer consent endpoints |
| `lexe-core/src/lexe_core/admin/routers/consent_audit.py` | Admin audit endpoints |
| `lexe-webchat/src/components/auth/ConsentOverlay.tsx` | Consent acceptance UI |
| `lexe-webchat/src/components/profile/PrivacyTab.tsx` | Privacy info + export UI |
| `lexe-docs/gdpr-consent-system.md` | Detailed consent system documentation |
| `sql/update_consent_terms_v1.sql` | Full text of free plan terms |
| `sql/update_consent_terms_v1_paid.sql` | Full text of paid plan terms |
| `lexe-privacy/` | PII detection engine (spaCy + custom) |
| `lexe-core/src/lexe_core/conversation/export_router.py` | Conversation export (Art. 15/20) |

---

*Ultimo aggiornamento: 2026-03-19*
