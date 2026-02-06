# LEXE Platform - Backlog

> Aggiornato: 2026-02-04

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

### Memory Extraction - Estrazione Fatti Errata
**Repo:** lexe-memory, lexe-core
**Segnalato:** 2026-02-06

**Problema:**
L'estrazione dei fatti dalla conversazione è completamente sbagliata.

**Esempio:**
- **Input utente:** "Buongiorno, sono rimasta vedova dal 24 marzo 2025..."
- **Fatto estratto:** `"L'utente si chiama rimasta"` ❌

Il sistema ha interpretato "sono rimasta" come un nome invece di capire il contesto.

**Impatto:**
- Memoria inquinata con fatti errati
- Risposte future basate su informazioni sbagliate
- Perdita di fiducia dell'utente

**Causa probabile:**
- LLM extractor con prompt troppo semplice
- Mancanza di validazione dei fatti estratti
- Possibile problema con il modello usato per l'estrazione

**Azioni:**
- [ ] Verificare prompt in `lexe-memory/extractors/llm_extractor.py`
- [ ] Verificare quale modello viene usato per l'estrazione
- [ ] Aggiungere validazione semantica dei fatti estratti
- [ ] Testare con diversi prompt per migliorare l'estrazione
- [ ] Considerare filtri per evitare fatti nonsense

**Files da investigare:**
- `lexe-core/gateway/customer_router.py` - `extract_and_store_facts()`
- `lexe-memory/src/lexe_memory/extraction/`
- `lexe-orchestrator/phases/extractors/llm_extractor.py`

---

### Risposte LLM Troncate - Investigare Limiti Token
**Repo:** lexe-webchat, lexe-core, lexe-orchestrator, LiteLLM
**Segnalato:** 2026-02-04

**Problema:**
Risposte LLM troncate in produzione. Utente riceve risposte incomplete.

**Analisi Frontend (lexe-webchat):**
Nel frontend **NON ci sono limiti espliciti di `max_tokens`**. I limiti sono gestiti dal backend.

| Limite | Valore | File | Potenziale Causa |
|--------|--------|------|------------------|
| Response Timeout | 30s | `SSEClient.ts:89` | ⚠️ Se risposta > 30s |
| Idle Timeout | 15s | `SSEClient.ts:90` | ⚠️ Se pause > 15s nello stream |
| Reconnect Attempts | 3 | `SSEClient.ts:88` | Dopo 3 tentativi, abort |

**Note:** Il typewriter effect (`chunkSize: 3`) è solo visualizzazione - non tronca dati.

**CAUSA IDENTIFICATA (2026-02-04):**

⚠️ **COLPEVOLE PRINCIPALE:** `max_tokens=1024` hardcoded in `phase3_drafting.py`

| File | Linee | Valore | Uso |
|------|-------|--------|-----|
| `lexe-orchestrator/phases/phase3_drafting.py` | 860, 928, 1438, 1548, 1694 | **1024** | Generazione risposta |
| `lexe-core/customer_router.py` | 952 | ~~2048~~ **4096** ✅ | LiteLLM direct path |
| `lexe-orchestrator/litellm.py` | 506 | 8000 | Streaming default |

**Altri limiti rilevanti:**
- `entity_extraction`: 500 tokens (phase2_semantic.py:357)
- `follow_up_suggestions`: 200 tokens (follow_up_service.py:95)
- `memory_extraction`: 1024 tokens (llm_extractor.py:300)
- `context_budget`: 2000 tokens (phase2_semantic.py:794)

**Timeout (OK - non sono il problema):**
- Backend chunk_timeout: 30s (stream_state.py:102)
- Backend stream_duration: 300s / 5 min (stream_state.py:101)
- Frontend responseTimeout: 30s (SSEClient.ts:89)
- Frontend idleTimeout: 15s (SSEClient.ts:90)

**Azioni:**
- [x] ~~Verificare `max_tokens` in lexe-core~~ → 2048 per tools
- [x] ~~Verificare `max_tokens` in lexe-orchestrator~~ → **1024 in drafting (PROBLEMA)**
- [x] ~~Verificare config LiteLLM~~ → Nessun limite hardcoded
- [x] ~~**FIX:** Aumentare `max_tokens` in phase3_drafting.py da 1024 a 4096~~ ✅ (2026-02-04)
- [ ] Considerare rendere `max_tokens` configurabile per tenant
- [ ] Test con prompt che generano risposte lunghe (>2000 tokens)

**Fix Raccomandato:**
```python
# phase3_drafting.py - linee 860, 928, 1438, 1548, 1694
max_tokens=4096  # Era 1024, troppo basso per risposte legali
```

---

### LLM Hallucination - Citazioni Giurisprudenziali Inventate
**Repo:** lexe-core, lexe-orchestrator
**Segnalato:** 2026-02-06

**Problema:**
Quando i legal tools sono disabilitati (o percepiti come tali nel frontend), l'LLM inventa citazioni di sentenze Cassazione con numeri specifici che potrebbero non esistere.

**Esempio dalla risposta:**
- `Cass. civ., sez. II, 10/05/2013, n. 11106`
- `Cass. civ., sez. II, 22/06/2017, n. 15535`
- `Cass. pen., sez. V, 12/03/2019, n. 10612`
- `Cass. civ., sez. II, 22/03/2016, n. 5678`

**Impatto:**
- **GRAVE per studi legali**: Avvocato cerca sentenza → non esiste → tempo perso
- **GRAVISSIMO**: Se l'avvocato cita la sentenza in un atto → figuraccia con giudice/cliente
- Perdita di credibilità della piattaforma

**Causa probabile:**
- LLM genera numeri plausibili ma non verificati
- Manca istruzione nel system prompt per evitare citazioni non verificabili
- Tools sembravano OFF nel frontend (da verificare se realmente disabilitati)

**Azioni:**
- [ ] Verificare se tools erano realmente OFF (check logs backend)
- [ ] Modificare system prompt: "NON citare numeri di sentenze specifici senza verifica con tools"
- [ ] Alternativa: "orientamento giurisprudenziale consolidato (da verificare su massimario)"
- [ ] Aggiungere disclaimer automatico quando tools OFF: "⚠️ Citazioni non verificate"
- [ ] Considerare flag `citations_verified: bool` nel metadata risposta

**Files da investigare:**
- System prompt in `core.responder_personas` (DB)
- `lexe-core/gateway/customer_router.py` - costruzione prompt
- Frontend: stato tools nel toggle bar

---

### Consigli Giuridici Errati - Responsabilità Solidale Mutuo
**Repo:** lexe-core (system prompt)
**Segnalato:** 2026-02-06

**Problema:**
La risposta suggerisce di "pagare solo la tua quota (50%)" del mutuo, ignorando che i mutui cointestati hanno tipicamente **responsabilità solidale** (art. 1292 c.c.).

**Citazione errata dalla risposta:**
> "Continua a pagare *solo la tua quota* (50%). Se gli altri chiamati rinunciano, la banca non può chiederti il pagamento della loro parte."

**Realtà giuridica:**
- Con responsabilità solidale, la banca può chiedere il **100%** a ciascun cointestatario
- La vedova rischia azione esecutiva per l'intero importo
- Consiglio errato potrebbe causare danni economici gravi

**Impatto:**
- **GRAVE**: Consiglio potenzialmente dannoso per il cliente dello studio
- Responsabilità professionale dell'avvocato che si affida alla risposta
- Perdita di fiducia nella piattaforma

**Causa probabile:**
- LLM semplifica eccessivamente concetti giuridici complessi
- Manca nel system prompt l'istruzione di essere cauto su obbligazioni solidali
- Manca disclaimer: "Verificare clausole contrattuali del mutuo"

**Azioni:**
- [ ] Aggiungere nel system prompt: "Per mutui/obbligazioni, SEMPRE menzionare possibile solidarietà"
- [ ] Aggiungere disclaimer automatico per argomenti "mutuo", "fideiussione", "coobbligato"
- [ ] Template: "⚠️ Verificare clausole contrattuali per responsabilità solidale vs pro-quota"
- [ ] Review del system prompt per altri potenziali consigli pericolosi

**Files da investigare:**
- System prompt in `core.responder_personas` (DB)
- Considerare knowledge base con "red flags" giuridiche

---

### System Prompt Update - Applicare Versione 2.0
**Repo:** lexe-core (DB), lexe-docs
**Segnalato:** 2026-02-06

**Problema:**
Il system prompt attuale non gestisce correttamente:
1. Citazioni giurisprudenziali quando tools sono OFF
2. Avvisi obbligatori su responsabilità solidale (mutui, fideiussioni)
3. Comportamento esplicito quando tools non disponibili

**Documenti Preparati:**
- `lexe-docs/system-prompts/lexe-legal-assistant-CURRENT.md` - Prompt attuale estratto dal DB
- `lexe-docs/system-prompts/lexe-legal-assistant-PROPOSED.md` - Versione 2.0 con fix
- `lexe-docs/PROMPT-OPTIMIZATION-PROPOSAL.md` - Analisi dettagliata

**Modifiche nella v2.0:**
| Sezione | Modifica |
|---------|----------|
| **NUOVA: Citazioni Giurisprudenziali** | NON citare numeri sentenze senza verifica tools |
| **NUOVA: Red Flags** | SEMPRE menzionare art. 1292 (solidarietà) per mutui |
| **Gestione Attendibilità** | Comportamento esplicito quando tools OFF |
| **Sintesi Iniziale** | Dichiarare stato tools nella risposta |

**Azioni:**
- [ ] Valutare modifiche proposte in `lexe-legal-assistant-PROPOSED.md`
- [ ] Testare nuovo prompt in sandbox/staging
- [ ] Applicare a DB con query:
  ```sql
  UPDATE core.responder_personas
  SET system_prompt = '<contenuto PROPOSED>'
  WHERE name = 'lexe-legal-assistant';
  ```
- [ ] Invalidare cache (restart lexe-core o endpoint dedicato)
- [ ] Verificare comportamento con tools ON e OFF

**Rischi se non implementato:**
- Continua hallucination di citazioni Cassazione
- Consigli errati su obbligazioni solidali
- Perdita credibilità con studi legali

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
