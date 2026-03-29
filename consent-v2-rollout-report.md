# Report: Adozione Graduale Nuove Regole Contrattuali — Consent v2.0

> Completato: 2026-03-27 → 2026-03-29 (3 giorni)
> Sprint: 19 (Context Engine v2)

---

## Obiettivo

Implementare l'adozione graduale delle nuove clausole contrattuali (protezione IP, anti-reverse engineering) mantenendo traccia di cosa ogni utente ha firmato, nel rispetto del GDPR.

## Risultati

### Tracciamento utenti iniziali
Gli utenti che hanno firmato i Terms v1.0 (lancio prod 2026-03-25) sono tracciati in modo immutabile:
- `core.consent_acceptance_log`: righe append-only con `terms_version='1.0'`, `content_hash` SHA-256, IP, user_agent, session_id, timestamp
- Esportabile via CSV dall'admin panel (`GET /admin/consent-audit/log/export`)
- Quando v2.0 è stata pubblicata, le righe v1.0 sono rimaste intatte — prova legale completa

### Nuovi termini v2.0
Tutti gli utenti vedono l'overlay di ri-accettazione al prossimo login con:
- **Clausola A** (Sez. 8): Divieto reverse engineering, prompt exfiltration, scraping, analisi competitiva
- **Clausola B** (Sez. 9): Obblighi utilizzo lecito, segreto commerciale D.Lgs. 63/2018
- **Clausola C** (Sez. 10): Sospensione immediata, risarcimento, azione cautelare, sopravvivenza 5 anni
- **Clausola D** (Sez. 11): Monitoraggio anti-abuse su base legittimo interesse Art. 6(1)(f)
- **7 vessatorie Art. 1341 c.2**: 3 esistenti + 4 nuove (reverse engineering, sospensione, sopravvivenza IP, monitoraggio)

---

## Fasi implementate

### Phase 1: Terms v2.0 + Clausole IP (2026-03-27)
| Deliverable | File |
|-------------|------|
| SQL terms v2.0 free | `sql/update_consent_terms_v2_free.sql` |
| SQL terms v2.0 paid | `sql/update_consent_terms_v2_paid.sql` |
| Vessatorie 7 voci | `lexe-webchat/src/components/auth/ConsentOverlay.tsx` |
| Fix IP X-Forwarded-For | `lexe-core/src/lexe_core/gateway/consent_router.py` |

### Phase 2: Rename consent fields + default OFF (2026-03-27)
| Deliverable | File |
|-------------|------|
| Migration 052a (add columns) | `lexe-core/migrations/052a_add_new_consent_columns.sql` |
| Migration 052b (drop old) | `lexe-core/migrations/052b_drop_old_consent_columns.sql` |
| Backend rename | `consent_router.py`, `consent_audit.py` |
| Frontend rename + default OFF | `consent.ts`, `ConsentOverlay.tsx` |
| Admin rename | `ConsentAuditPage.tsx` |

Campi rinominati:
- `consent_metrics` → `consent_analytics_enhanced`
- `consent_pseudonymized` → `consent_quality_improvement`
- `consent_training` → nascosto (sempre false, mantenuto in DB per storico)

### Phase 3: Anonymizer NER + consent gate (2026-03-29)
| Deliverable | File |
|-------------|------|
| Anonymizer (lexe-privacy client) | `lexe-improve-lab/src/lexe_improve/collector/anonymizer.py` |
| Consent gate collector | `lexe-improve-lab/src/lexe_improve/collector/conversations.py` |
| Debug audit log | `lexe-improve-lab/src/lexe_improve/collector/audit_log.py` |
| Config privacy URL | `lexe-improve-lab/src/lexe_improve/config.py` |

L'anonymizer usa **lexe-privacy** (Presidio + spaCy NER) via HTTP:
- Entità rilevate: PERSON, ORG, LOC, EMAIL, PHONE, CF, PIVA
- Legal entities italiane: Tribunale, Corte, Ministero
- Fallback regex se servizio non raggiungibile
- Due modalità: `collect_anonymous()` (tutto, anonimizzato) e `collect_pseudonymized()` (solo consenzienti)

### Phase 4: Revoca handler + purge (2026-03-28)
| Deliverable | File |
|-------------|------|
| Migration 053 | `lexe-core/migrations/053_consent_purge_action.sql` |
| Purge job | `lexe-core/src/lexe_core/jobs/consent_purge.py` |
| Revoca detection | `consent_router.py` (PATCH /consent/preferences) |

Quando utente disattiva "Miglioramento qualità risposte":
1. Detect flip `true → false`
2. Purge automatico dati pseudonimizzati raw
3. Log `data_purged` in `consent_acceptance_log`
4. Aggregati mantenuti (già anonimi)

### Phase 5: Privacy Policy v2.1 — notifica in-app (2026-03-29)
| Deliverable | File |
|-------------|------|
| SQL privacy update | `sql/update_privacy_policy_v2_1.sql` |
| Banner + modal | `lexe-webchat/src/components/ui/PrivacyUpdateBanner.tsx` |
| Mount in App | `lexe-webchat/src/App.tsx` |

Privacy policy aggiornata con sezioni:
- Finalità granulari (analytics enhanced vs quality improvement)
- Anonimizzazione pre-storage via NER
- Pseudonimizzazione SHA-256 in tabelle separate
- Retention: 12 mesi pseudo, illimitato anonimo, 6 mesi monitoring
- Revoca immediata + purge automatico

Banner non-blocking: l'utente non deve ri-accettare, solo notifica dismissable.

### Phase 6: Analytics gate + DPIA (2026-03-29)
| Deliverable | File |
|-------------|------|
| Consent gate helper | `lexe-core/src/lexe_core/admin/routers/insights/consent_gate.py` |
| Consent gate conversations | `conversations_insights.py` (1 query) |
| Consent gate activity | `activity.py` (3 query) |
| Consent gate engagement | `engagement.py` (1 query) |
| Retention job | `lexe-core/src/lexe_core/jobs/data_retention.py` |
| DPIA | `lexe-docs/internal/DPIA-improve-lab.md` |

6 query per-utente ora filtrate per `consent_quality_improvement = true`:
- `_query_longest_conversations`, `_query_top_users`, `_query_top_rated_messages`
- `_query_top_users_by_score`, `_query_top_engaged`

Query aggregate (COUNT, AVG) invariate — già anonime per natura.

k-Anonymity: tenant con < 10 utenti consenzienti → solo metriche aggregate cross-tenant.

---

## Commit history

| Repo | Branch | Commits |
|------|--------|---------|
| lexe-core | stage → main | 6 commit (IP fix, Phase 2, Phase 4, Phase 6) |
| lexe-webchat | stage → main | 4 commit (vessatorie, Phase 2, banner, banner fix) |
| lexe-admin | stage → main | 1 commit (audit page rename) |
| lexe-improve-lab | stage → main | 3 commit (anonymizer, NER client, config) |
| lexe-docs | stage | 1 commit (DPIA) |

## Migrations eseguite

| # | File | Staging | Prod |
|---|------|---------|------|
| 052a | `add_new_consent_columns.sql` | Done | Done |
| 052b | `drop_old_consent_columns.sql` | Done | Done |
| 053 | `consent_purge_action.sql` | Done | Done |

## SQL data eseguiti

| File | Staging | Prod |
|------|---------|------|
| `update_consent_terms_v2_free.sql` | Done | Done |
| `update_consent_terms_v2_paid.sql` | Done | Done |
| `update_privacy_policy_v2_1.sql` | Done | Done |

---

## Backlog residuo

- **6e (NER fine-tuning)**: lasciato in backlog. Presidio è engine primario, fine-tuning spaCy non pratico. Se servisse migliorare NER → aggiungere custom Presidio recognizer.
- **Dynamic Contract Management**: `change_type` (minor/major) e `target_role` (admin/user) per gestione contratti da admin panel. Vedi `backlog_contract_management.md`.

---

*Documento generato automaticamente — 2026-03-29*
