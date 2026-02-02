# LEXE Platform - Backlog

> Aggiornato: 2026-02-01

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

---

*Generato automaticamente - 2026-02-01*
