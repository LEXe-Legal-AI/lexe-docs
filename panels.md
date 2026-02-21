# LEXE Admin Panel — Architettura a 3 Livelli

> Ultimo aggiornamento: 2026-02-16

La piattaforma LEXE espone tre esperienze distinte in base al ruolo dell'utente autenticato.
Tutte e tre condividono la stessa SPA (`lexe-webchat`) ma il contenuto visibile viene filtrato
**server-side** da `GET /api/v1/admin/me` e **client-side** dal hook `useAdminAccess`.

---

## 1. Panoramica Ruoli

| Ruolo            | Panel          | Chi lo usa                         | Come si assegna                              |
| ---------------- | -------------- | ---------------------------------- | -------------------------------------------- |
| **user** (basic) | MiniPanel      | Cliente dello studio legale        | Registrazione via Magic Link / Logto         |
| **admin**        | Tenant Admin   | Amministratore dello studio legale | Assegnato da superadmin in `core.users.role` |
| **superadmin**   | Platform Admin | Gestore della piattaforma LEXE     | `UPDATE core.users SET role='superadmin'`    |

Ereditarietà: ogni livello **include** tutte le funzionalità del livello precedente.

```
User (chat, sidebar, tool toggles)
  └─ Tenant Admin (+ gestione studio)
       └─ Platform Admin (+ gestione piattaforma, - dati clienti per privacy)
```

---

## 2. Funzionalità Comuni (Chat)

Queste funzionalità sono disponibili a **tutti** i ruoli autenticati:

| Funzionalità              | Descrizione                                                                           | Componente                |
| ------------------------- | ------------------------------------------------------------------------------------- | ------------------------- |
| **Chat AI**               | Conversazione con l'assistente legale LEXE, streaming SSE, markdown rendering         | `ChatPanel.tsx`           |
| **Sidebar conversazioni** | Lista cronologica delle proprie conversazioni, ricerca, rinomina, eliminazione        | `ConversationSidebar.tsx` |
| **Model selector**        | Scelta del modello LLM da usare (es. GPT-4o, Claude)                                  | `ModelSelector.tsx`       |
| **Tool toggles**          | Attivazione/disattivazione dei tool legali (Normattiva, vigenza, InfoLex, web search) | `ToolToggles.tsx`         |
| **Memory profile**        | Selettore profilo memoria per il contesto conversazionale                             | `MemorySelector.tsx`      |
| **File upload**           | Caricamento documenti nella chat                                                      | `FileUpload.tsx`          |
| **Follow-up suggestions** | Chip cliccabili con domande di follow-up generate dall'AI                             | `FollowUpChips.tsx`       |

---

## 3. MiniPanel (User Base)

Accessibile dal menu utente (icona ingranaggio). Non apre il pannello admin completo,
ma un pannello ridotto con sole impostazioni personali.

| Voce                        | Descrizione                                              | Endpoint                              |
| --------------------------- | -------------------------------------------------------- | ------------------------------------- |
| **Contatore conversazioni** | Numero totale di conversazioni dell'utente               | `GET /gateway/customer/conversations` |
| **Tema**                    | Toggle dark/light mode (persistito in localStorage)      | Client-side only                      |
| **Lingua**                  | Selettore lingua interfaccia (IT, EN, DE, ES, FR, PT-BR) | Client-side only                      |

**File:** `lexe-webchat/src/components/admin/mini/MiniPanel.tsx`

---

## 4. Sezioni Admin Panel — Matrice Completa

### Legenda

| Simbolo        | Significato                                                   |
| -------------- | ------------------------------------------------------------- |
| ✅ CRUD         | Lettura + Creazione + Modifica + Eliminazione                 |
| ✅ R            | Solo lettura                                                  |
| 👁️ R/O        | Visibile ma in sola lettura (input disabilitati, no Save)     |
| ❌ bloccato     | Nascosto dal privacy guard (superadmin non vede dati clienti) |
| ❌ non visibile | Nascosto dal filtro `panel` (non pertinente al ruolo)         |
| ⬜ mancante     | Previsto dal piano ma non ancora implementato nel frontend    |

### Matrice

| Sezione                          | Tenant Admin   | Platform Admin | Stato impl.        | Backend endpoint(s)                                                                                                                                                                                            |
| -------------------------------- |:--------------:|:--------------:|:------------------:| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Dashboard**                    | ✅ R            | ✅ R            | REAL               | `GET /admin/dashboard/stats`                                                                                                                                                                                   |
| **Conversations**                | ✅ R            | ❌ bloccato     | REAL               | `GET /admin/conversations` `GET /admin/conversations/:id` `GET /admin/conversations/:id/messages`                                                                                                              |
| **Contacts**                     | ✅ CRUD         | ❌ bloccato     | REAL               | `GET /admin/contacts` `GET /admin/contacts/:id` `PUT /admin/contacts/:id` `POST /admin/workflow/contacts/invite` `POST /admin/workflow/contacts/:id/deactivate` `POST /admin/workflow/contacts/:id/reactivate` |
| **Users**                        | ✅ CRUD         | ❌ non visibile | REAL               | `GET /admin/users` `GET /admin/users/:id` `POST /admin/users/invite` `PUT /admin/users/:id` `DELETE /admin/users/:id` `POST /admin/users/:id/deactivate` `POST /admin/users/:id/reactivate`                    |
| **Tools**                        | ❌ non visibile | ✅ CRUD         | REAL               | `GET /admin/tools` `GET /admin/tools/:id` `POST /admin/tools` `PATCH /admin/tools/:id` `DELETE /admin/tools/:id`                                                                                               |
| **Models**                       | ❌ non visibile | ✅ CRUD         | REAL               | `GET /admin/models` `GET /admin/models/:id` `POST /admin/models` `PATCH /admin/models/:id` `DELETE /admin/models/:id`                                                                                          |
| **Personas**                     | ❌ non visibile | ✅ CRUD         | REAL               | `GET /admin/personas` `GET /admin/personas/:id` `POST /admin/personas` `PUT /admin/personas/:id` `DELETE /admin/personas/:id`                                                                                  |
| **Tenants**                      | ❌ non visibile | ✅ CRUD         | REAL               | `GET /admin/tenants` `GET /admin/tenants/:id` `POST /admin/tenants` `PATCH /admin/tenants/:id` `DELETE /admin/tenants/:id`                                                                                     |
| **API Health**                   | ❌ non visibile | ✅ R            | REAL               | Usa `healthProbeKey` dai vari endpoint del registry                                                                                                                                                            |
| **Settings** ↳ tab Settings      | ✅ R/W          | ✅ R/W          | REAL               | `GET /admin/settings` `PUT /admin/settings`                                                                                                                                                                    |
| **Settings** ↳ tab Limits        | 👁️ R/O        | ✅ R/W          | REAL               | `GET /admin/limits` `PUT /admin/limits`                                                                                                                                                                        |
| **Settings** ↳ tab Feature Flags | 👁️ R/O        | ✅ R/W          | REAL               | `GET /admin/settings/config/feature-flags` `PUT /admin/settings/config/feature-flags/:key`                                                                                                                     |
| **Settings** ↳ tab Prompt AI     | ⬜ mancante     | ❌ (403)        | BE OK, FE mancante | `GET /admin/settings/active-prompt`                                                                                                                                                                            |
| **Audit**                        | ✅ R            | ✅ R            | REAL               | `GET /admin/audit-log`                                                                                                                                                                                         |

---

## 5. Descrizione Dettagliata Sezioni

### 5.1 Dashboard

**Pannello:** both | **Permesso:** `settings.read` | **data_class:** `aggregate`

Mostra 9 metriche aggregate del tenant (o della piattaforma) in card statistiche:

- `total_conversations` / `active_conversations` — Volume conversazionale
- `total_messages` — Messaggi scambiati
- `total_contacts` / `active_contacts` — Utenti finali (clienti dello studio)
- `total_users` / `active_users` — Staff dello studio (operatori, admin)
- `total_personas` / `active_personas` — Assistenti AI configurati

I dati sono aggregati server-side in `DashboardService` e non espongono dati personali,
quindi sono visibili sia al tenant admin che al platform admin.

**Frontend:** `DashboardPage.tsx` — skeleton loading, error state con retry.
**Backend:** `GET /admin/dashboard/stats` → `DashboardService.get_stats()`

---

### 5.2 Conversations

**Pannello:** tenant | **Permesso:** `conversations.read` | **data_class:** `customer_content`

Viewer admin delle conversazioni dei clienti. Layout a due pannelli:

- **Pannello sinistro:** lista conversazioni con filtri (status: all/active/closed, canale), ricerca, paginazione (20/page)
- **Pannello destro:** visualizzatore messaggi della conversazione selezionata (50/page), bubble chat user/assistant, token count

**Perche il superadmin non lo vede:**
Il `data_class=customer_content` attiva il **privacy guard**: il gestore della piattaforma
non deve poter leggere le conversazioni legali dei clienti degli studi. Questo e un requisito
GDPR/deontologico: i dati dei clienti appartengono allo studio, non al fornitore SaaS.

**Frontend:** `ConversationsPage.tsx`
**Backend:** `GET /admin/conversations` (list), `GET /:id` (detail), `GET /:id/messages` (messaggi paginati)

---

### 5.3 Contacts

**Pannello:** tenant | **Permesso:** `contacts.read` / `contacts.write` | **data_class:** `customer_pii`

Gestione anagrafica dei clienti dello studio legale:

- Lista con ricerca, filtro status (active/inactive/invited/suspended)
- Colonne: email, display_name, phone, status, created_at
- **Azioni CRUD:** invita (email + nome), modifica (display_name, phone), disattiva/riattiva
- Toast di conferma con link al log di audit (correlation_id)

**Perche il superadmin non lo vede:**
`data_class=customer_pii` — i dati anagrafici dei clienti sono PII protetti.
Il gestore della piattaforma non deve poter accedere ai nomi/email dei clienti degli studi.

**Workflow Logto:** invite/deactivate/reactivate passano per `router_workflow.py` che sincronizza
lo stato con Logto (CIAM) oltre che nel DB locale.

**Frontend:** `ContactsPage.tsx`
**Backend:** `contacts.py` (CRUD) + `router_workflow.py` (invite/deactivate/reactivate via Logto)

---

### 5.4 Users

**Pannello:** tenant | **Permesso:** `users.read` / `users.write` | **data_class:** `customer_pii`

Gestione del team dello studio legale (operatori, analisti, admin):

- DataTable ordinabile: display_name, email, role, status, last_login
- Filtri: ruolo (superadmin → viewer), status (active/inactive)
- **Azioni:** invita (email + nome + ruolo + checkbox invio email), modifica profilo, cambia ruolo, disattiva/riattiva, elimina
- Dialog dettaglio utente con tutte le info + avatar con iniziali

**Ruoli assegnabili:** superadmin, admin, manager, operator, analyst, viewer (6 livelli RBAC).

**Frontend:** `UsersPage.tsx`
**Backend:** `identity/users.py` montato sotto `/admin/users`

---

### 5.5 Tools

**Pannello:** platform | **Permesso:** `settings.read` / `settings.write` | **data_class:** `config`

Catalogo globale dei tool legali disponibili sulla piattaforma:

- Lista con CRUD completo: nome, display_name, tipo, stato attivo, configurazione JSON
- Toggle inline per attivare/disattivare un tool
- Validazione JSON sulla configurazione

I tool configurati qui (es. `verify_vigenza`, `fetch_article_text`, `llsearch`) diventano
disponibili per le personas dei tenant.

**Frontend:** `ToolsPage.tsx`
**Backend:** `tools.py` → `ToolService`

---

### 5.6 Models

**Pannello:** platform | **Permesso:** `settings.read` / `settings.write` | **data_class:** `config`

Catalogo dei modelli LLM disponibili sulla piattaforma (routing via LiteLLM):

- CRUD completo: nome, provider, is_enabled, is_default, priority, configurazione JSON
- Stella per il modello default
- Toggle inline per abilitare/disabilitare

I modelli configurati qui vengono esposti ai tenant tramite il model selector nella chat.

**Frontend:** `ModelsPage.tsx`
**Backend:** `models.py` → `ModelService`

---

### 5.7 Personas

**Pannello:** platform | **Permesso:** `settings.read` / `settings.write` | **data_class:** `config`

Gestione centralizzata degli assistenti AI (personas/prompt):

- CRUD: name, display_name, slug (validato), description, language, style
- **system_prompt** (campo obbligatorio) — il prompt di sistema che definisce il comportamento dell'AI
- allowed_tools / blocked_tools — ChipInput per selezionare i tool abilitati
- is_active, is_default — toggle attivazione e selezione come default
- Mostra `contactCount` (quanti contatti usano questa persona)
- Blocco eliminazione se la persona e in uso

**Frontend:** `PersonasPage.tsx`
**Backend:** `identity/personas.py` montato sotto `/admin/personas`

---

### 5.8 Tenants

**Pannello:** platform | **Permesso:** `tenants.manage` | **data_class:** `config`

Gestione degli studi legali (tenant) sulla piattaforma:

- CRUD completo: name, slug (read-only dopo creazione), is_active
- Slug: validazione lowercase + hyphens + alfanumerico
- Data di creazione formattata
- Conferma eliminazione con nome del tenant

Ogni tenant rappresenta uno studio legale con il proprio spazio isolato
(conversazioni, contatti, utenti, impostazioni) grazie a Row-Level Security (RLS).

**Frontend:** `TenantsPage.tsx`
**Backend:** `admin/routers/tenants.py` → `TenantService` (router dedicato, non importa da `identity/api.py`)

---

### 5.9 API Health

**Pannello:** platform | **Permesso:** `system.manage` | **data_class:** `config`

Monitor dello stato di salute degli endpoint backend:

- Bottone "Run health check" per eseguire probe su tutti gli endpoint
- Raggruppamento per stato: live / degraded / offline
- Progress counter durante il check
- Integrazione con `apiHealthStore` per persistenza stato

Usa i `healthProbeKey` definiti nel registry (`admin-registry.ts`) per sapere
quali endpoint testare.

**Frontend:** `ApiHealthPage.tsx`
**Backend:** Nessun endpoint dedicato — il frontend chiama direttamente i vari `GET` degli endpoint registrati

---

### 5.10 Settings (4 tab)

**Pannello:** both | **Permesso:** `settings.manage` | **data_class:** `config`

La sezione Settings contiene 4 tab con comportamento diverso per ruolo:

#### Tab 1: Settings (Impostazioni generali)

Configurazione specifica del tenant. **R/W per entrambi i ruoli.**

| Campo                 | Tipo     | Descrizione                                                          |
| --------------------- | -------- | -------------------------------------------------------------------- |
| `welcome_message`     | textarea | Messaggio di benvenuto mostrato all'utente nella chat                |
| `default_language`    | dropdown | Lingua default (IT, EN, FR, ES, DE, PT-BR)                           |
| `auto_archive_days`   | number   | Giorni dopo cui archiviare automaticamente le conversazioni inattive |
| `enable_guest_access` | toggle   | Permette accesso senza autenticazione                                |
| `notification_email`  | email    | Email per notifiche amministrative                                   |

**Backend:** `GET/PUT /admin/settings` → `SettingsService`

#### Tab 2: Limits (Limiti)

Limiti operativi del tenant. **R/W per platform, READ-ONLY per tenant.**

| Campo                           | Tipo   | Descrizione                              |
| ------------------------------- | ------ | ---------------------------------------- |
| `max_contacts`                  | number | Numero massimo di contatti (clienti)     |
| `max_conversations_per_day`     | number | Conversazioni massime al giorno          |
| `max_messages_per_conversation` | number | Messaggi massimi per conversazione       |
| `max_tokens_per_month`          | number | Budget token LLM mensile                 |
| `max_users`                     | number | Numero massimo di utenti staff           |
| `max_personas`                  | number | Numero massimo di personas configurabili |

Il tenant admin vede i propri limiti ma **non puo modificarli** — sono imposti dal platform admin
come parte del piano commerciale. Input disabilitati, bottone Save nascosto, badge "Solo lettura".

**Backend:** `GET/PUT /admin/limits` → `LimitsService`

#### Tab 3: Feature Flags

Flags di attivazione funzionalita. **R/W per platform, READ-ONLY per tenant.**

Ogni flag mostra:

- Nome (font monospace)
- Badge sorgente: `tenant` (override locale) vs `global` (default piattaforma)
- Descrizione
- Toggle switch (disabilitato per tenant admin)

Il tenant admin vede quali feature sono attive sul proprio tenant ma non puo modificarle.
Il platform admin le gestisce centralmente.

**Backend:** `GET /admin/settings/config/feature-flags` + `PUT /:key` → `FeatureFlagService`

#### Tab 4: Prompt AI (FRONTEND NON ANCORA IMPLEMENTATO)

Visualizzazione read-only del system prompt attivo per il tenant. **Solo tenant admin.**

**Scopo:** Il tenant admin deve poter vedere quale prompt governa il comportamento
dell'assistente AI del proprio studio, senza poterlo modificare (la gestione e centralizzata
dal platform admin tramite la sezione Personas).

**Comportamento previsto:**

- Mostra il nome della persona attiva (`persona_name`, `display_name`)
- Mostra il `system_prompt` completo in un div scrollabile con font monospace
- Mostra lingua (`language`) e stile (`style`)
- Badge `is_default` se e la persona default
- Avviso: "Configurato dall'amministratore della piattaforma"
- **Errori:** 404 → "Nessun prompt attivo configurato"; 403 → "Non disponibile"; 500 → "Errore" + bottone Riprova

**Logica backend (gia implementata):**

1. Cerca `ResponderPersona` con `tenant_id=X, is_active=True, is_default=True`
2. Fallback: `is_active=True` ORDER BY `created_at DESC` LIMIT 1
3. Se nessuna persona attiva: 404
4. Se chiamato da superadmin: 403 (il platform admin gestisce le personas direttamente)

**Backend:** `GET /admin/settings/active-prompt` → `active_prompt.py` (FUNZIONANTE)
**Frontend:** `ActivePromptTab` in `SettingsPage.tsx` (DA IMPLEMENTARE)

---

## 6. Privacy Guard — Separazione Dati

Il platform admin (superadmin) **non puo** accedere a sezioni con dati dei clienti.
Questo e un requisito di design, non un bug:

| data_class         | Contenuto                                  | Tenant Admin | Platform Admin | Motivazione                                                                  |
| ------------------ | ------------------------------------------ |:------------:|:--------------:| ---------------------------------------------------------------------------- |
| `customer_content` | Conversazioni, messaggi                    | ✅            | ❌ bloccato     | Contenuto legale riservato: il fornitore SaaS non deve leggere le consulenze |
| `customer_pii`     | Contatti (email, nome, telefono), Utenti   | ✅            | ❌ bloccato     | Dati personali GDPR: il fornitore non deve accedere all'anagrafica clienti   |
| `config`           | Settings, tools, models, personas, tenants | ✅            | ✅              | Configurazione piattaforma, nessun dato personale                            |
| `aggregate`        | Dashboard stats, audit log                 | ✅            | ✅              | Dati aggregati/anonimi, nessuna PII                                          |

**Implementazione server-side** (`me.py:79`):

```python
_SUPERADMIN_BLOCKED_DATA_CLASSES = frozenset({"customer_pii", "customer_content"})
```

Le sezioni con `data_class` bloccato vengono **omesse** dalla risposta di `GET /admin/me`
prima ancora che il frontend le veda. Doppio filtro: server non le ritorna + client non le renderizza.

---

## 7. Pipeline di Filtro Sezioni

Quando un utente chiama `GET /admin/me`, il backend applica 4 filtri in cascata:

```
_SECTIONS (11 definizioni)
    │
    ├─ 1. Panel filter ──────── panel="tenant"|"platform"|"both" vs ruolo utente
    │                            → rimuove sezioni non pertinenti al pannello
    │
    ├─ 2. Privacy guard ─────── data_class in {"customer_pii","customer_content"}
    │                            → rimuove sezioni con dati sensibili per superadmin
    │
    ├─ 3. Feature flag gate ─── feature_flag attivo/disattivo
    │                            → rimuove sezioni con FF disabilitato
    │
    └─ 4. RBAC permission ───── permission richiesta vs permessi utente
                                 → include la sezione ma con enabled=false + motivo
```

Risultato per **tenant admin** (ruolo `admin`):
→ dashboard, conversations, contacts, settings, audit, users (6 sezioni)

Risultato per **platform admin** (ruolo `superadmin`):
→ dashboard, tools, models, settings, audit, personas, tenants, api_health (8 sezioni)

---

## 8. Endpoint Registry Completo

Tutti gli endpoint sono registrati in `admin-registry.ts` con metadata tipizzati.
Totale: **41 endpoint** + 1 meta-endpoint (`admin.me`).

| Sezione       | Endpoint Key             | Method | Path                                        | Permission            |
| ------------- | ------------------------ | ------ | ------------------------------------------- | --------------------- |
| Dashboard     | `dashboard.stats`        | GET    | `/admin/dashboard/stats`                    | `dashboard:read`      |
| Contacts      | `contacts.list`          | GET    | `/admin/contacts`                           | `contacts:read`       |
|               | `contacts.get`           | GET    | `/admin/contacts/:id`                       | `contacts:read`       |
|               | `contacts.update`        | PUT    | `/admin/contacts/:id`                       | `contacts:write`      |
|               | `contacts.invite`        | POST   | `/admin/workflow/contacts/invite`           | `contacts:write`      |
|               | `contacts.deactivate`    | POST   | `/admin/workflow/contacts/:id/deactivate`   | `contacts:write`      |
|               | `contacts.reactivate`    | POST   | `/admin/workflow/contacts/:id/reactivate`   | `contacts:write`      |
| Conversations | `conversations.list`     | GET    | `/admin/conversations`                      | `conversations:read`  |
|               | `conversations.get`      | GET    | `/admin/conversations/:id`                  | `conversations:read`  |
|               | `conversations.messages` | GET    | `/admin/conversations/:id/messages`         | `conversations:read`  |
| Tools         | `tools.list`             | GET    | `/admin/tools`                              | `tools:read`          |
|               | `tools.get`              | GET    | `/admin/tools/:id`                          | `tools:read`          |
|               | `tools.create`           | POST   | `/admin/tools`                              | `tools:write`         |
|               | `tools.update`           | PATCH  | `/admin/tools/:id`                          | `tools:write`         |
|               | `tools.delete`           | DELETE | `/admin/tools/:id`                          | `tools:write`         |
| Models        | `models.list`            | GET    | `/admin/models`                             | `models:read`         |
|               | `models.get`             | GET    | `/admin/models/:id`                         | `models:read`         |
|               | `models.create`          | POST   | `/admin/models`                             | `models:write`        |
|               | `models.update`          | PATCH  | `/admin/models/:id`                         | `models:write`        |
|               | `models.delete`          | DELETE | `/admin/models/:id`                         | `models:write`        |
| Limits        | `limits.get`             | GET    | `/admin/limits`                             | `limits:read`         |
|               | `limits.update`          | PUT    | `/admin/limits`                             | `limits:write`        |
| Settings      | `settings.list`          | GET    | `/admin/settings`                           | `settings:read`       |
|               | `settings.get`           | GET    | `/admin/settings/:key`                      | `settings:read`       |
|               | `settings.update`        | PUT    | `/admin/settings`                           | `settings:write`      |
|               | `settings.activePrompt`  | GET    | `/admin/settings/active-prompt`             | `settings:read`       |
| Audit         | `audit.list`             | GET    | `/admin/audit-log`                          | `audit:read`          |
| Users         | `users.list`             | GET    | `/admin/users`                              | `users:read`          |
|               | `users.get`              | GET    | `/admin/users/:id`                          | `users:read`          |
|               | `users.invite`           | POST   | `/admin/users/invite`                       | `users:write`         |
|               | `users.update`           | PUT    | `/admin/users/:id`                          | `users:write`         |
|               | `users.delete`           | DELETE | `/admin/users/:id`                          | `users:write`         |
|               | `users.deactivate`       | POST   | `/admin/users/:id/deactivate`               | `users:write`         |
|               | `users.reactivate`       | POST   | `/admin/users/:id/reactivate`               | `users:write`         |
| Personas      | `personas.list`          | GET    | `/admin/personas`                           | `personas:read`       |
|               | `personas.get`           | GET    | `/admin/personas/:id`                       | `personas:read`       |
|               | `personas.create`        | POST   | `/admin/personas`                           | `personas:write`      |
|               | `personas.update`        | PUT    | `/admin/personas/:id`                       | `personas:write`      |
|               | `personas.delete`        | DELETE | `/admin/personas/:id`                       | `personas:write`      |
| Tenants       | `tenants.list`           | GET    | `/admin/tenants`                            | `tenants:read`        |
|               | `tenants.get`            | GET    | `/admin/tenants/:id`                        | `tenants:read`        |
|               | `tenants.create`         | POST   | `/admin/tenants`                            | `tenants:write`       |
|               | `tenants.update`         | PATCH  | `/admin/tenants/:id`                        | `tenants:write`       |
|               | `tenants.delete`         | DELETE | `/admin/tenants/:id`                        | `tenants:write`       |
| Feature Flags | `featureFlags.list`      | GET    | `/admin/settings/config/feature-flags`      | `feature_flags:read`  |
|               | `featureFlags.toggle`    | PUT    | `/admin/settings/config/feature-flags/:key` | `feature_flags:write` |
| Admin Me      | `admin.me`               | GET    | `/admin/me`                                 | `admin:read`          |

---

## 9. Stato Implementazione

| Componente        | Backend | Frontend   | Note                                                 |
| ----------------- |:-------:|:----------:| ---------------------------------------------------- |
| Dashboard         | ✅ REAL  | ✅ REAL     | Dati aggregati dal DB                                |
| Conversations     | ✅ REAL  | ✅ REAL     | Read-only viewer con paginazione                     |
| Contacts          | ✅ REAL  | ✅ REAL     | CRUD + workflow Logto (invite/deactivate/reactivate) |
| Users             | ✅ REAL  | ✅ REAL     | CRUD completo con 6 ruoli RBAC                       |
| Tools             | ✅ REAL  | ✅ REAL     | CRUD + validazione JSON config                       |
| Models            | ✅ REAL  | ✅ REAL     | CRUD + toggle default                                |
| Personas          | ✅ REAL  | ✅ REAL     | CRUD + blocco delete se in uso                       |
| Tenants           | ✅ REAL  | ✅ REAL     | CRUD + validazione slug                              |
| API Health        | ✅ REAL  | ✅ REAL     | Health probe su tutti gli endpoint                   |
| Settings tab      | ✅ REAL  | ✅ REAL     | R/W per entrambi i ruoli                             |
| Limits tab        | ✅ REAL  | ✅ REAL     | R/W platform, R/O tenant                             |
| Feature Flags tab | ✅ REAL  | ✅ REAL     | R/W platform, R/O tenant                             |
| **Prompt AI tab** | ✅ REAL  | ⬜ MANCANTE | Backend funzionante, frontend da creare              |
| Audit             | ✅ REAL  | ✅ REAL     | Log con filtri + correlation_id                      |
| MiniPanel (user)  | ✅ REAL  | ✅ PARTIAL  | Count conversazioni reale; tema/lingua client-side   |

**Riepilogo:** 14/15 componenti fully REAL. Unico mancante: `ActivePromptTab` nel frontend.

---

## 10. File di Riferimento

### Backend (lexe-core)

| File                             | Scopo                                                 |
| -------------------------------- | ----------------------------------------------------- |
| `admin/routers/me.py`            | Sezioni, filtro panel/privacy/FF/RBAC                 |
| `admin/schemas/me.py`            | PanelType, AdminMeResponse, AdminSection              |
| `admin/router.py`                | Aggregatore — monta tutti i sub-router                |
| `admin/routers/dashboard.py`     | Stats aggregate                                       |
| `admin/routers/contacts.py`      | CRUD contatti                                         |
| `admin/routers/conversations.py` | Viewer conversazioni                                  |
| `admin/routers/tools.py`         | CRUD tools                                            |
| `admin/routers/models.py`        | CRUD modelli LLM                                      |
| `admin/routers/settings.py`      | Settings tenant                                       |
| `admin/routers/limits.py`        | Limiti tenant                                         |
| `admin/routers/audit.py`         | Audit log                                             |
| `admin/routers/active_prompt.py` | Prompt AI (tenant-only, 403 per superadmin)           |
| `admin/routers/tenants.py`       | CRUD tenants (dedicato, no import da identity/api.py) |
| `identity/users.py`              | CRUD utenti (montato sotto /admin)                    |
| `identity/personas.py`           | CRUD personas (montato sotto /admin)                  |
| `identity/router_workflow.py`    | Invite/deactivate/reactivate via Logto                |
| `gateway/api/feature_flags.py`   | Feature flags (montato sotto /admin/settings)         |

### Frontend (lexe-webchat)

| File                                             | Scopo                                               |
| ------------------------------------------------ | --------------------------------------------------- |
| `components/admin/AdminPanel.tsx`                | Shell principale, sidebar nav, routing sezioni      |
| `components/admin/AdminGuard.tsx`                | Guard: admin → AdminPanel, user → MiniPanel         |
| `components/admin/hooks/useAdminAccess.ts`       | Hook: chiama /admin/me, espone panelType + sections |
| `components/admin/mini/MiniPanel.tsx`            | Pannello ridotto per user base                      |
| `components/admin/pages/DashboardPage.tsx`       | Dashboard                                           |
| `components/admin/pages/ConversationsPage.tsx`   | Viewer conversazioni                                |
| `components/admin/pages/ContactsPage.tsx`        | Gestione contatti                                   |
| `components/admin/pages/UsersPage.tsx`           | Gestione utenti                                     |
| `components/admin/pages/ToolsPage.tsx`           | Gestione tools                                      |
| `components/admin/pages/ModelsPage.tsx`          | Gestione modelli                                    |
| `components/admin/pages/PersonasPage.tsx`        | Gestione personas                                   |
| `components/admin/pages/TenantsPage.tsx`         | Gestione tenants                                    |
| `components/admin/pages/ApiHealthPage.tsx`       | Health check                                        |
| `components/admin/pages/SettingsPage.tsx`        | Settings (4 tab)                                    |
| `components/admin/pages/AuditPage.tsx`           | Audit log                                           |
| `services/api/admin-registry.ts`                 | Registry 41 endpoint tipizzati                      |
| `i18n/locales/{it,en,de,es,fr,pt-BR}/admin.json` | Traduzioni admin                                    |
