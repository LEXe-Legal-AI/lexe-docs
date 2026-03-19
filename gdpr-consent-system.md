# GDPR Consent System — Documentazione Tecnica

> Sistema di consenso GDPR per LEXE Legal AI
> Reg. UE 2016/679 + D.Lgs. 196/2003 (mod. D.Lgs. 101/2018)

---

## Panoramica

Il Consent System di LEXE implementa le 4 fasi della compliance GDPR:

| Fase | Descrizione | Status |
|------|-------------|--------|
| **1. Informativa** | Termini di Servizio + Privacy Policy versionati | DONE |
| **2. Prova Consenso** | Registrazione IP/User-Agent + audit trail export | DONE |
| **3. Disclaimer** | AI disclaimer art. 1229 c.c., no-training, profiling | DONE |
| **4. Diritti** | Export dati (art. 15/20), cancellazione (art. 17), info panel | DONE |

---

## Architettura

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTEND                              │
│                                                              │
│  ConsentOverlay.tsx          PrivacyTab.tsx                   │
│  ┌──────────────────┐       ┌──────────────────────────────┐ │
│  │ 3 checkbox:       │       │ Accordions GDPR:             │ │
│  │ • terms (required)│       │ • Dati raccolti              │ │
│  │ • metrics (opt)   │       │ • Finalità trattamento       │ │
│  │ • pseudonymized   │       │ • Base giuridica (Art. 6)    │ │
│  │   (opt)           │       │ • Conservazione              │ │
│  │                   │       │ • Diritti (Art. 15-22)       │ │
│  │ Nome + Cognome    │       │                              │ │
│  │ (primo accesso)   │       │ [Esporta dati] [Cancella]    │ │
│  └────────┬─────────┘       └──────────────────────────────┘ │
│           │                                                   │
└───────────┼───────────────────────────────────────────────────┘
            │ POST /consent/accept
            ▼
┌───────────────────────────────────────────────────────────────┐
│                     BACKEND (lexe-core)                        │
│                                                                │
│  consent_router.py (customer-facing)                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ GET  /consent/status        → accepted? needs_update?     │  │
│  │ GET  /consent/terms         → body_html + existing prefs  │  │
│  │ POST /consent/accept        → upsert + capture IP/UA      │  │
│  │ PATCH /consent/preferences  → update toggles only         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  consent_audit.py (admin-facing, under /api/v1/admin)          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ GET /admin/consent-audit         → JSON audit records     │  │
│  │ GET /admin/consent-audit/export  → CSV download (DPO)     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│                    DATABASE (lexe-postgres)                     │
│                                                                │
│  core.consent_terms          core.user_consents                │
│  ┌─────────────────────┐     ┌──────────────────────────────┐  │
│  │ id (UUID PK)        │     │ id (UUID PK)                 │  │
│  │ version ("1.0")     │◄────│ terms_id (FK)                │  │
│  │ terms_type           │     │ tenant_id (UUID, NOT NULL)   │  │
│  │   (free/paid/demo)  │     │ user_sub (TEXT, Logto sub)   │  │
│  │ title               │     │ terms_accepted (BOOLEAN)     │  │
│  │ body_html           │     │ consent_metrics (BOOLEAN)    │  │
│  │ summary             │     │ consent_training (BOOLEAN)   │  │
│  │ published_at        │     │ consent_pseudonymized (BOOL) │  │
│  │ created_at          │     │ accepted_at (TIMESTAMPTZ)    │  │
│  └─────────────────────┘     │ ip_address (TEXT)            │  │
│                               │ user_agent (TEXT, max 500)   │  │
│                               │ created_at, updated_at       │  │
│                               └──────────────────────────────┘  │
│                                                                │
│  RLS: user_consents_tenant (tenant_id = fn_get_tenant_id())    │
│  RLS: user_consents_superadmin (fn_is_superadmin())            │
│  UNIQUE: (user_sub, terms_id) — one consent per user per ver.  │
│  INDEX: (user_sub, tenant_id) — fast lookup                    │
└───────────────────────────────────────────────────────────────┘
```

---

## Feature Flag

```
LEXE_FF_CONSENT_REQUIRED=true
```

| Valore | Comportamento |
|--------|---------------|
| `false` (default) | `GET /consent/status` ritorna `{accepted: true}` → nessun overlay |
| `true` | Controlla se l'utente ha accettato la versione corrente dei terms |

---

## Endpoints Customer

### `GET /consent/status`

Verifica rapida se l'utente ha accettato i terms correnti.

**Auth**: Bearer JWT (customer)

**Response** `200`:
```json
{
  "accepted": true,
  "terms_version": "1.0",
  "needs_update": false
}
```

**Logica**:
1. Se FF disabilitato → `accepted: true` immediatamente
2. Risolve tenant da JWT
3. Determina `plan_type` (free/paid/demo) dal tenant
4. Cerca i terms pubblicati più recenti per quel plan_type
5. Verifica se esiste un record `user_consents` per (user_sub, terms_id) con `terms_accepted=true`

---

### `GET /consent/terms`

Restituisce il contenuto HTML dei termini pubblicati, con le preferenze esistenti (se l'utente li aveva già accettati).

**Auth**: Bearer JWT (customer)

**Response** `200`:
```json
{
  "terms_id": "a1b2c3d4-...",
  "version": "1.0",
  "terms_type": "free",
  "title": "Termini di Servizio e Informativa Privacy — LEXE Legal AI",
  "body_html": "<h2>Termini Generali di Servizio...</h2>...",
  "summary": "Condizioni d'uso del piano Free, disclaimer AI...",
  "existing_consent": {
    "consent_metrics": true,
    "consent_training": false,
    "consent_pseudonymized": true
  }
}
```

---

### `POST /consent/accept`

Accetta i termini e registra la prova documentata del consenso.

**Auth**: Bearer JWT (customer)

**Request**:
```json
{
  "terms_id": "a1b2c3d4-...",
  "consent_metrics": true,
  "consent_training": false,
  "consent_pseudonymized": true,
  "first_name": "Mario",
  "last_name": "Rossi"
}
```

**Dati catturati automaticamente dal server** (non inviati dal client):
- `ip_address` — da `request.client.host`
- `user_agent` — da header `User-Agent` (troncato a 500 char)
- `accepted_at` — `datetime.now(UTC)`

**Comportamento**:
1. Verifica che `terms_id` esista e sia pubblicato
2. `UPSERT` su `user_consents` (chiave: `user_sub + terms_id`)
3. Se `first_name`/`last_name` forniti (primo accesso demo) → aggiorna nome su Logto + `core.contacts`
4. Log strutturato: `[Consent] User {sub} accepted terms {id} (metrics=..., training=..., pseudo=...)`

**Response** `201`:
```json
{ "status": "ok" }
```

---

### `PATCH /consent/preferences`

Aggiorna i toggle privacy senza ri-accettare i termini. Utile dalla pagina Profilo.

**Auth**: Bearer JWT (customer)

**Request**:
```json
{
  "consent_metrics": false,
  "consent_training": false,
  "consent_pseudonymized": true
}
```

**Response** `200`: `{ "status": "ok" }`
**Response** `404`: Se l'utente non ha mai accettato i terms

---

## Endpoints Admin (Audit Trail DPO)

Montati sotto `/api/v1/admin/consent-audit`.
**Auth**: AdminUser + permesso `settings.read`.
**Tenant isolation**: `TenantIdStrict` dal JWT → filtra solo i consent del tenant corrente.

### `GET /admin/consent-audit`

Lista JSON di tutti i consent records del tenant, con filtri opzionali.

**Query parameters**:

| Param | Tipo | Descrizione |
|-------|------|-------------|
| `user_sub` | string | Filtra per Logto sub specifico |
| `date_from` | date (YYYY-MM-DD) | Consent accettati dal (incluso) |
| `date_to` | date (YYYY-MM-DD) | Consent accettati fino al (incluso) |

**Response** `200`:
```json
{
  "items": [
    {
      "user_sub": "logto-user-id-123",
      "display_name": "Mario Rossi",
      "email": "mario.rossi@studio.it",
      "terms_version": "1.0",
      "terms_type": "free",
      "terms_accepted": true,
      "consent_metrics": true,
      "consent_training": false,
      "consent_pseudonymized": true,
      "accepted_at": "2026-03-16T10:30:00+00:00",
      "ip_address": "93.41.xxx.xxx",
      "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)...",
      "created_at": "2026-03-16T10:30:00+00:00"
    }
  ],
  "total": 1
}
```

**Query SQL** (semplificata):
```sql
SELECT uc.*, ct.version, ct.terms_type, c.display_name, c.email
FROM core.user_consents uc
JOIN core.consent_terms ct ON ct.id = uc.terms_id
LEFT JOIN core.contacts c ON c.external_user_id = uc.user_sub
WHERE uc.tenant_id = $1
ORDER BY uc.accepted_at DESC NULLS LAST
```

---

### `GET /admin/consent-audit/export`

Download CSV per audit trail DPO. Stessi filtri dell'endpoint JSON.

**Response**: `text/csv` con header `Content-Disposition: attachment; filename="consent-audit-{tenant_id}.csv"`

**Colonne CSV**:

| Colonna | Descrizione |
|---------|-------------|
| `user_sub` | Identificativo Logto dell'utente |
| `display_name` | Nome visualizzato (da `core.contacts`) |
| `email` | Email utente |
| `terms_version` | Versione dei termini accettati (es. "1.0") |
| `terms_type` | Tipo piano (free/paid/demo) |
| `terms_accepted` | `true` se accettato |
| `consent_metrics` | Toggle metriche aggregate anonime |
| `consent_training` | Toggle training AI (sempre `false`, non esposto in UI) |
| `consent_pseudonymized` | Toggle analisi pseudonimizzata per KPI |
| `accepted_at` | Timestamp ISO 8601 accettazione |
| `ip_address` | IP al momento dell'accettazione |
| `user_agent` | Browser/device (max 500 char) |
| `created_at` | Timestamp creazione record |

**Esempio output CSV**:
```csv
user_sub,display_name,email,terms_version,terms_type,terms_accepted,consent_metrics,consent_training,consent_pseudonymized,accepted_at,ip_address,user_agent,created_at
logto-abc123,Mario Rossi,mario@studio.it,1.0,free,True,True,False,True,2026-03-16T10:30:00+00:00,93.41.12.34,Mozilla/5.0...,2026-03-16T10:30:00+00:00
```

---

## Contenuto dei Termini (v1.0)

I termini sono versionati per piano (`free`/`paid`). Entrambi contengono:

### Parte 1 — Termini di Servizio

| Sezione | Contenuto |
|---------|-----------|
| **1. Oggetto** | Descrizione servizio LEXE, AI per ricerca normativa/giurisprudenziale |
| **2. Limiti AI** | Output non è consulenza legale, verifica autonoma obbligatoria |
| **3. Disclaimer** | Art. 1229 c.c., no responsabilità per allucinazioni AI, servizio "così com'è" |
| **4. Proprietà intellettuale** | IP di IT Consulting S.r.l., licenza tecnica su prompt utente |
| **5. Durata e recesso** | Free: tempo indeterminato. Paid: condizioni commerciali |
| **6. Modifiche** | Ri-accettazione richiesta per modifiche sostanziali |
| **7. Foro** | Bergamo (salvo foro consumatore inderogabile) |

### Parte 2 — Informativa Privacy

| Sezione | Contenuto |
|---------|-----------|
| **1. Titolare** | IT Consulting S.r.l., P.IVA IT03411290160, privacy@itconsultingsrl.com |
| **2. Dati raccolti** | Identificativi, conversazioni, metadati tecnici. Isolamento RLS per tenant |
| **3. Finalità** | Esecuzione contratto + consenso per metriche. **No training AI terze parti** |
| **4. Destinatari** | Hetzner EU, Logto self-hosted, LLM gateway (OpenAI/Google come responsabili) |
| **5. Conservazione** | Max 12 mesi post-cessazione account |
| **6. Diritti** | Art. 15 (accesso), 16 (rettifica), 17 (oblio), 18 (limitazione), 20 (portabilità), 21 (opposizione) |
| **7. Cookie** | Solo tecnici; analitici subordinati a consenso |
| **8. DPO** | dpo@itconsultingsrl.com |

### Aggiornamento termini

```bash
# Free plan (già applicato)
docker exec -i lexe-postgres psql -U lexe -d lexe < sql/update_consent_terms_v1.sql

# Paid/Professional plan (già applicato)
docker exec -i lexe-postgres psql -U lexe -d lexe < sql/update_consent_terms_v1_paid.sql
```

Per nuove versioni: inserire un nuovo record con `version='1.1'` e `published_at=now()`. Gli utenti vedranno l'overlay e dovranno ri-accettare.

---

## Frontend

### ConsentOverlay (`lexe-webchat/src/components/auth/ConsentOverlay.tsx`)

Modale full-screen al primo accesso (o quando i terms vengono aggiornati):

- Campo **Nome** + **Cognome** (primo accesso demo)
- Testo espandibile: "Leggi i termini completi" → render `body_html` via `dangerouslySetInnerHTML`
- 3 checkbox:
  - **Termini di Servizio** (obbligatorio, blocca submit se non spuntato)
  - **Metriche di miglioramento** (opzionale, default on) — dati tecnici aggregati anonimi
  - **Analisi pseudonimizzata** (opzionale, default on) — KPI con hash irreversibile

### PrivacyTab (`lexe-webchat/src/components/profile/PrivacyTab.tsx`)

Tab nel profilo utente con:

- **Accordions informativi**: dati raccolti, finalità, base giuridica, conservazione, diritti GDPR
- **Esporta i tuoi dati**: download JSON di tutte le conversazioni (art. 15/20)
- **Cancellazione account**: link `mailto:privacy@lexe.pro` con subject pre-compilato (art. 17)
- **Contatto DPO**: `privacy@lexe.pro`
- **Privacy Policy completa**: link a `https://lexe.pro/privacy`

---

## File di Riferimento

| File | Ruolo |
|------|-------|
| `lexe-core/migrations/036_consent_terms.sql` | Schema DB + seed iniziale |
| `lexe-core/src/lexe_core/gateway/consent_router.py` | Endpoints customer |
| `lexe-core/src/lexe_core/admin/routers/consent_audit.py` | Endpoints admin audit |
| `lexe-core/src/lexe_core/admin/router.py` | Wiring admin router |
| `sql/update_consent_terms_v1.sql` | Terms completi piano Free |
| `sql/update_consent_terms_v1_paid.sql` | Terms completi piano Professional |
| `lexe-webchat/src/components/auth/ConsentOverlay.tsx` | UI accettazione |
| `lexe-webchat/src/components/profile/PrivacyTab.tsx` | UI profilo privacy |
| `lexe-webchat/src/services/api/consent.ts` | API service frontend |

---

## Riferimenti Normativi

- **Reg. UE 2016/679** (GDPR) — Art. 5 (trasparenza), Art. 6 (base giuridica), Art. 7 (condizioni consenso), Art. 12-22 (diritti interessato), Art. 13 (informativa)
- **D.Lgs. 196/2003** mod. D.Lgs. 101/2018 — Codice Privacy italiano
- **Art. 1229 c.c.** — Limitazione clausole di esonero responsabilità
- **Massime Cassazione**: 669166, 638406, 639804, 628322, 645195, 648504

---

*Ultimo aggiornamento: 2026-03-19*
