# DPIA — Improve-Lab Analytics Pipeline

> Data Protection Impact Assessment ai sensi dell'Art. 35 GDPR
> Documento interno — IT Consulting S.r.l.
> Data: 2026-03-29 | Versione: 1.0

---

## 1. Descrizione del trattamento

### Natura
Raccolta e analisi di conversazioni tra utenti e il sistema LEXE Legal AI, finalizzata al miglioramento della qualità delle risposte legali (accuratezza, completezza, correttezza normativa).

### Ambito
- **Dati trattati**: messaggi utente/assistente, metadati conversazione (pipeline, confidence, latenza, tool calls), valutazioni utente
- **Soggetti interessati**: utenti della piattaforma LEXE che hanno fornito consenso esplicito
- **Volume stimato**: ~1000 conversazioni/settimana (rolling 7 giorni)
- **Frequenza**: raccolta periodica su richiesta (CLI manuale, non automatizzata)

### Finalità
Miglioramento del servizio AI: identificazione di pattern di errore (hallucination, bassa confidence, tool failure), ottimizzazione pipeline, benchmark qualità.

### Base giuridica
| Modalità | Base giuridica | Riferimento |
|----------|---------------|-------------|
| Raccolta anonima (senza consenso) | Legittimo interesse Art. 6(1)(f) | Dati completamente anonimizzati, fuori ambito GDPR (Considerando 26) |
| Raccolta pseudonimizzata (con consenso) | Consenso esplicito Art. 6(1)(a) | Toggle "Miglioramento qualità risposte" — opt-in, granulare, revocabile |
| Accesso debug (ticket-based) | Esecuzione contratto Art. 6(1)(b) | Supporto tecnico per bug resolution |

---

## 2. Necessità e proporzionalità

### Necessità
L'analisi delle conversazioni è necessaria per identificare e correggere pattern di errore nell'AI legale. Errori in ambito legale possono avere conseguenze gravi per gli utenti professionisti.

### Proporzionalità — Misure di minimizzazione
| Misura | Implementazione |
|--------|----------------|
| **Consenso granulare** | Due toggle separati: analytics_enhanced e quality_improvement. Default OFF (opt-in) |
| **Anonimizzazione pre-storage** | PII rimossi da NER (Presidio + spaCy) prima di qualsiasi storage analitico |
| **Tabelle separate** | `analytics_anonymous` (nessun ID) e `analytics_pseudonymized` (hash ID) fisicamente separate |
| **Retention limitata** | Pseudonimizzati: max 12 mesi. Anonimi: illimitati (non sono dati personali) |
| **Consent gate** | Query per-utente bloccate per utenti senza consenso |
| **k-Anonymity** | Tenant con < 10 utenti consenzienti → solo metriche aggregate cross-tenant |
| **Revoca immediata** | Flip toggle → purge automatico dati pseudonimizzati raw |

---

## 3. Rischi identificati

### R1: Re-identificazione da contenuto conversazionale
- **Probabilità**: Media
- **Impatto**: Alto (contenuto legale sensibile)
- **Scenario**: Nomi propri nei messaggi ("Il mio cliente Mario Rossi...") non completamente catturati dal NER
- **Mitigazione attuata**: NER via Presidio + spaCy (PERSON, ORG, LOC, CF, PIVA, EMAIL, PHONE). Modello `it_core_news_lg` per italiano
- **Rischio residuo**: Basso — NER cattura la maggior parte dei nomi italiani. Entità rare (nomi stranieri atipici, abbreviazioni) possono sfuggire
- **Piano mitigazione residuo**: Custom Presidio recognizer per pattern domain-specific (nomi parti processuali). Review periodica falsi negativi su campione

### R2: Re-identificazione da tenant piccoli
- **Probabilità**: Media (per tenant < 10 utenti)
- **Impatto**: Medio
- **Scenario**: Aggregati per-tenant con pochi utenti → inferenza identità
- **Mitigazione attuata**: k-Anonymity check (k ≥ 10). Sotto soglia → solo metriche cross-tenant
- **Rischio residuo**: Trascurabile

### R3: Accesso non autorizzato ai dati pseudonimizzati
- **Probabilità**: Bassa
- **Impatto**: Alto
- **Scenario**: Accesso diretto al DB analitico
- **Mitigazione attuata**: PostgreSQL RLS, accesso solo via API autenticata (admin + RBAC), audit log immutabile per debug access
- **Rischio residuo**: Trascurabile

### R4: Correlazione cross-dataset
- **Probabilità**: Bassa
- **Impatto**: Medio
- **Scenario**: Hash deterministici potrebbero permettere correlazione tra dataset diversi
- **Mitigazione attuata**: Hash troncati (SHA256[:16] per user, [:8] per tenant). Senza la chiave originale, il brute-force è impraticabile
- **Rischio residuo**: Trascurabile

---

## 4. Misure tecniche e organizzative

### Pipeline di anonimizzazione
```
Conversazione DB (PII presente)
  ↓
lexe-improve-lab collector
  ↓
lexe-privacy NER (Presidio + spaCy it_core_news_lg)
  ├─ PERSON → [PERSONA]
  ├─ ORG → [ORG]
  ├─ LOC → [LUOGO]
  ├─ EMAIL → [EMAIL]
  ├─ PHONE → [TELEFONO]
  ├─ CF → [CF]
  └─ PIVA → [PIVA]
  ↓
  ├─ Percorso ANONIMO: ID azzerati, metadata stripped → analytics_anonymous
  └─ Percorso PSEUDONIMO: ID hashati, metadata preservati → analytics_pseudonymized
      (solo utenti con consent_quality_improvement = true)
```

### Consent lifecycle
```
Utente accetta terms v2.0
  → checkbox "Miglioramento qualità" OFF di default
  → utente opta in esplicitamente
  → consent_acceptance_log: action='accepted', IP, user_agent, session, hash

Utente revoca via profilo
  → consent_quality_improvement: true → false
  → consent_acceptance_log: action='preferences_updated'
  → purge automatico: DELETE analytics_pseudonymized WHERE pseudo_user_id = hash(user_sub)
  → consent_acceptance_log: action='data_purged'
```

### Audit trail
| Componente | Implementazione |
|------------|----------------|
| Consenso | `core.consent_acceptance_log` — append-only, content_hash SHA-256 |
| Debug access | `lexe-improve-lab/data/debug_audit.jsonl` — JSON lines, retention 2 anni |
| Purge | Registrato in `consent_acceptance_log` con action `data_purged` |

---

## 5. Valutazione complessiva del rischio

| Rischio | Pre-mitigazione | Post-mitigazione |
|---------|----------------|-----------------|
| R1: Re-identificazione contenuto | Alto | **Basso** |
| R2: Re-identificazione tenant piccoli | Medio | **Trascurabile** |
| R3: Accesso non autorizzato | Medio | **Trascurabile** |
| R4: Correlazione cross-dataset | Basso | **Trascurabile** |

**Rischio residuo complessivo: BASSO**

Il trattamento può procedere con le misure attuate. Non è necessaria consultazione preventiva dell'Autorità Garante (Art. 36 GDPR).

---

## 6. Approvazione

| Ruolo | Nome | Data | Firma |
|-------|------|------|-------|
| Titolare del trattamento | IT Consulting S.r.l. | ______ | ______ |
| DPO | ______ | ______ | ______ |
| Responsabile tecnico | ______ | ______ | ______ |

---

## 7. Revisione

Prossima revisione: **2026-09-29** (6 mesi) o al cambiamento significativo del trattamento.

Trigger per revisione anticipata:
- Introduzione di nuove fonti dati
- Cambio di engine NER
- Incidente di re-identificazione
- Feedback dell'Autorità Garante
