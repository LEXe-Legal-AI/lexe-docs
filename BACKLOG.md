# LEXE Platform - Backlog

> Aggiornato: 2026-02-02

---

## P0 - URGENTI

### lexe-channels - Verificare necessità
**Repo:** lexe-orchestrator
**Files:**
- `src/lexe_orchestrator/tools/whatsapp.py`
- `src/lexe_orchestrator/tools/whatsapp_v1.py`
- `src/lexe_orchestrator/services/whatsapp_notify.py`

**Descrizione:**
I moduli WhatsApp puntano a `lexe-channels:8010` (rinominato da `leo-channels`).
Verificare se questo servizio è necessario per la webchat o solo per altri canali (WhatsApp, Telegram, Email).

**Azioni:**
- [ ] Verificare se lexe-webchat usa questi endpoint
- [ ] Se NO → commentare come P3 (altri canali stile LEO)
- [ ] Se SI → implementare lexe-channels

**Stato attuale:** Modulo `whatsapp.py` già disabilitato (flag `WHATSAPP_TOOLS_DISABLED = True`)

---

## P1 - IMPORTANTI

### LEO References Cleanup
Vedi reports in ogni repo:
- `lexe-core/LEO-REFS-REMAINING.md`
- `lexe-orchestrator/mappa-lexe-orchestrator.md`
- `lexe-memory/LEO-REFERENCES-REPORT.md`
- `LEO-REFERENCES-T4.md` (lexe-max)
- `LEO-REFERENCES-T5.md` (lexe-webchat)

---

## P1.5 - POST GO-LIVE (Security Enhancement)

### RLS Activation per App Runtime

> **Riferimento:** [RLS-ACTIVATION-ROADMAP.md](RLS-ACTIVATION-ROADMAP.md)

**Stato Attuale (2026-02-02):**
- ✅ RLS implementato e testato in PostgreSQL (STAGE)
- ✅ Utente `lexe_app` creato con permessi corretti
- ✅ Policy RLS su 6 tabelle (contacts, conversations, memory.*)
- ❌ App usa ancora `lexe` (superuser) → RLS bypassato

**Decisione:** Go-live con Opzione A (RLS pronto ma non attivo per app)

**Motivo:** Priorità go-live su MAIN senza rischi

**Task da Implementare (POST GO-LIVE in STAGE):**

1. **Middleware Tenant Context** `[lexe-core]`
   - [ ] Creare `TenantContextMiddleware`
   - [ ] Estrarre `tenant_id` dal JWT
   - [ ] Eseguire `SET app.current_tenant_id` ad ogni request
   - [ ] Unit test middleware

2. **Dependency Injection** `[lexe-core]`
   - [ ] Creare `get_tenant_db_session()` dependency
   - [ ] Aggiornare router per usare la nuova dependency
   - [ ] Gestire casi admin con `app.rls_bypass`

3. **Aggiornare Services** `[lexe-core, lexe-memory]`
   - [ ] `ConversationService` - settare tenant context
   - [ ] `MemoryService` - settare tenant context
   - [ ] Verificare altri service con accesso DB

4. **Test E2E in STAGE**
   - [ ] Cambiare connection string a `lexe_app`
   - [ ] Test webchat completo
   - [ ] Test API conversations
   - [ ] Test memory system
   - [ ] Verificare nessuna regressione

5. **Rollout MAIN**
   - [ ] Merge codice middleware
   - [ ] Cambiare connection string in MAIN
   - [ ] Monitoring intensivo 24h
   - [ ] Rollback plan testato

**Effort Stimato:** 1-2 settimane (sviluppo + test)

**Documenti Correlati:**
- [RLS-MULTITENANCY.md](RLS-MULTITENANCY.md) - Setup tecnico completo
- [RLS-ACTIVATION-ROADMAP.md](RLS-ACTIVATION-ROADMAP.md) - Decisione e piano
- [WEBGUI-ADMIN-DESIGN.md](WEBGUI-ADMIN-DESIGN.md) - Pattern per admin

**Benefici Post-Attivazione:**
- SQL injection non può accedere a dati cross-tenant
- Audit a livello database
- Defense in depth (Python + DB)
- Compliance security best practices

---

## P2 - NORMALI

### ~~Storage Keys Rename (lexe-webchat)~~ ✅ COMPLETATO
~~Rinominare localStorage keys da `leo-*` a `lexe-*`~~
**Completato:** 2026-02-01 (commit 4a54180)
- 26 file modificati, 131 sostituzioni
- Tutti i riferimenti P0/P1 rinominati

### Tailwind Design System Colors (lexe-webchat)
Rinominare classi CSS Tailwind da `leo-*` a `lexe-*`:
- `leo-primary` → `lexe-primary`
- `leo-secondary` → `lexe-secondary`
- `leo-accent` → `lexe-accent`
- `leo-dark` → `lexe-dark`
- `leo-light` → `lexe-light`
- `leo-gray` → `lexe-gray`

**Files coinvolti:** 70+ componenti
**Impatto:** Solo cosmetico (nomi classi CSS)
**Effort:** Alto (refactoring massivo)
**Priorita:** Bassa - funziona comunque, solo naming consistency

---

## P2 - NORMALI

### Admin Panel - Gestione Persona/Prompt
**Repo:** lexe-admin (da creare) o lexe-webchat
**Database:** `core.responder_personas`

**Descrizione:**
Creare interfaccia admin per gestire le persona AI (system prompt) per ogni organization.
Attualmente il prompt è nel DB (`core.responder_personas`) e viene letto da `customer_router.py`.

**Features richieste:**
- [ ] CRUD persona per organization
- [ ] Editor system prompt con preview
- [ ] Variabili template: `{{organization_name}}`, etc.
- [ ] Selezione modello LLM di default
- [ ] Gestione tools abilitati/disabilitati
- [ ] Test prompt in sandbox

**Schema DB esistente:**
```sql
core.responder_personas (
  id, organization_id, name, display_name,
  system_prompt, language, style,
  allowed_tools, blocked_tools, is_default
)
```

**Note:**
- Il prompt supporta variabili come `{{organization_name}}`
- Cache in-memory con TTL 5 min in `customer_router.py`
- Per invalidare cache: restart container o implementare endpoint

---

## P3 - NICE TO HAVE

### Altri Canali (stile LEO)
Se necessario implementare canali aggiuntivi:
- WhatsApp via Evolution API
- Telegram
- Email

Riferimento: moduli in `lexe-orchestrator/tools/whatsapp*.py`

### WebGUI Admin Panel

> **Riferimento:** [WEBGUI-ADMIN-DESIGN.md](WEBGUI-ADMIN-DESIGN.md)

**Descrizione:** Interfaccia web per gestione tenant, utenti, e configurazioni.

**Features:**
- [ ] Dashboard Super Admin (cross-tenant)
- [ ] Gestione Tenant (CRUD)
- [ ] Gestione Utenti per tenant
- [ ] Configurazione Tools per tenant
- [ ] Visualizzazione Conversazioni
- [ ] Audit Logs
- [ ] Metriche e Analytics

**Prerequisiti:**
- RLS attivo per app (vedi P1.5)
- Auth Logto con ruoli admin

**Effort:** 4-6 settimane

---

*Ultimo aggiornamento: 2026-02-02*
