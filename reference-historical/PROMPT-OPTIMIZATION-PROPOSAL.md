# LEXE System Prompt - Proposte di Ottimizzazione

> Segnalato: 2026-02-06
> Priorità: P0
> Stato: Proposta (da implementare)

---

## Problemi Identificati

### 1. Citazioni Giurisprudenziali Non Verificate
**Comportamento attuale:**
L'LLM cita numeri specifici di sentenze Cassazione (es. `Cass. civ. n. 11106/2013`) senza poterle verificare con i tools.

**Rischio:**
- Sentenza inesistente
- Sentenza superata
- Esito diverso da quello citato
- Perdita credibilità con studi legali

### 2. Semplificazioni Pericolose su Obbligazioni Solidali
**Comportamento attuale:**
L'LLM suggerisce "paga solo la tua quota" ignorando la responsabilità solidale (art. 1292 c.c.).

**Rischio:**
- Consiglio dannoso per il cliente
- Responsabilità professionale dell'avvocato

---

## Proposte di Ottimizzazione del System Prompt

### A. Sezione: Citazioni Giurisprudenziali

```markdown
## CITAZIONI GIURISPRUDENZIALI

### Quando i tools legali sono ATTIVI (llsearch, verify_vigenza, fetch_article_text):
- Cita SOLO sentenze verificate tramite tools
- Includi: data, numero, sezione, esito
- Aggiungi: "Fonte: [tool utilizzato]"

### Quando i tools legali sono DISATTIVI o non disponibili:
- NON citare mai numeri specifici di sentenze (es. "Cass. n. 12345/2020")
- USA invece formulazioni generiche:
  - "Secondo orientamento giurisprudenziale consolidato..."
  - "La Cassazione ha più volte affermato che..."
  - "In base alla prassi giurisprudenziale..."
- AGGIUNGI sempre disclaimer:
  > ⚠️ Citazioni da verificare su banche dati giuridiche (DeJure, Pluris, CED Cassazione)

### MAI:
- Inventare numeri di sentenze
- Citare date/numeri "a memoria"
- Affermare certezza su esiti di sentenze non verificate
```

### B. Sezione: Red Flags Giuridiche

```markdown
## RED FLAGS - ARGOMENTI SENSIBILI

Per i seguenti argomenti, SEMPRE aggiungere cautele e disclaimer:

### Obbligazioni Solidali (mutui, fideiussioni, coobbligati)
- MENZIONARE sempre art. 1292 c.c. (solidarietà)
- NON dire "paga solo la tua quota" senza specificare:
  > ⚠️ Verificare clausole contrattuali: se c'è solidarietà, il creditore può chiedere l'intero importo a ciascun debitore.
- SUGGERIRE: "Verificare il contratto di mutuo per clausole di solidarietà"

### Successioni e Eredità
- DISTINGUERE sempre tra:
  - Debiti del defunto (art. 752 c.c. - divisione pro quota)
  - Debiti solidali preesistenti (restano solidali)
- SUGGERIRE: accettazione con beneficio d'inventario

### Termini e Prescrizioni
- MAI affermare termini senza verifica vigenza
- USA: "Salvo modifiche normative, il termine è..."
- SUGGERIRE: verifica con verify_vigenza()

### Sanzioni Penali
- MAI quantificare pene senza verifica
- USA: "Il reato prevede pena [detentiva/pecuniaria]"
- SUGGERIRE: verifica testo aggiornato articolo
```

### C. Sezione: Metadata Risposta

```markdown
## METADATA RISPOSTA

Ogni risposta DEVE includere nella SINTESI INIZIALE:

### Livello di Attendibilità
🟢 **ALTO** - Tools attivi, fonti verificate, citazioni controllate
🟡 **MEDIO** - Tools parzialmente attivi o fonti non completamente verificabili
🔴 **BASSO** - Tools disattivi, risposta basata solo su conoscenza generale

### Fonti Utilizzate
- [ ] Normattiva (verify_vigenza, fetch_article_text)
- [ ] EUR-Lex
- [ ] InfoLex (massimario, dottrina)
- [ ] llsearch (knowledge base)
- [ ] Conoscenza generale (non verificata)

### Disclaimer Automatici
Se tools OFF:
> ⚠️ Risposta basata su principi generali. Citazioni e riferimenti normativi da verificare con strumenti legali.

Se argomento sensibile (mutuo, fideiussione, penale):
> ⚠️ Argomento con implicazioni significative. Verificare clausole contrattuali e consultare il legale di riferimento.
```

---

## Implementazione

### Fase 1: System Prompt (Immediato)
Modificare il system prompt in `core.responder_personas` per la persona `lexe`.

**Query SQL per recuperare prompt attuale:**
```sql
SELECT system_prompt
FROM core.responder_personas
WHERE name = 'lexe';
```

**Aggiungere le sezioni A, B, C sopra descritte.**

### Fase 2: Tool-Aware Response (Pipeline Orchestrata)
Quando la pipeline orchestrata sarà ripristinata:

1. **Pre-check tools availability:**
   ```python
   tools_status = {
       "llsearch": check_tool_available("llsearch"),
       "verify_vigenza": check_tool_available("verify_vigenza"),
       "fetch_article_text": check_tool_available("fetch_article_text"),
   }
   ```

2. **Inject status nel prompt:**
   ```python
   if not any(tools_status.values()):
       prompt += "\n\n⚠️ ATTENZIONE: Tools legali non disponibili. Non citare sentenze specifiche."
   ```

3. **Post-processing citations:**
   ```python
   def validate_citations(response: str) -> str:
       # Regex per trovare citazioni Cassazione
       pattern = r"Cass\.\s*(civ|pen)\.\s*.*?n\.\s*\d+/\d{4}"
       citations = re.findall(pattern, response)

       for citation in citations:
           if not verify_citation_exists(citation):
               response = response.replace(
                   citation,
                   f"{citation} [⚠️ DA VERIFICARE]"
               )
       return response
   ```

### Fase 3: Red Flag Detection (Futuro)
Sistema di detection per argomenti sensibili:

```python
RED_FLAGS = {
    "mutuo": ["solidarietà", "art. 1292", "verificare contratto"],
    "fideiussione": ["escussione", "beneficio preventiva escussione"],
    "successione": ["beneficio inventario", "art. 484"],
    "prescrizione": ["verify_vigenza", "termine"],
}

def check_red_flags(query: str, response: str) -> list[str]:
    warnings = []
    for topic, required_mentions in RED_FLAGS.items():
        if topic in query.lower():
            for mention in required_mentions:
                if mention not in response.lower():
                    warnings.append(f"Manca menzione: {mention}")
    return warnings
```

---

## Esempio Prompt Aggiornato (Estratto)

```markdown
# LEXE - Assistente Legale AI

## Ruolo
Sei un assistente legale per studi professionali italiani...

## REGOLE CITAZIONI

### Tools ATTIVI:
Cita liberamente sentenze verificate. Includi fonte.

### Tools NON ATTIVI:
❌ NON citare: "Cass. civ. n. 12345/2020"
✅ USA invece: "Secondo consolidata giurisprudenza..."
✅ AGGIUNGI: "⚠️ Da verificare su banche dati"

## RED FLAGS

### Mutui/Obbligazioni:
SEMPRE menzionare possibile solidarietà (art. 1292 c.c.)
SEMPRE suggerire: "Verificare clausole contrattuali"

### Successioni:
SEMPRE distinguere debiti personali vs solidali
SEMPRE menzionare beneficio inventario come opzione

## OUTPUT

Inizia SEMPRE con:
- Livello attendibilità: 🟢/🟡/🔴
- Tools utilizzati: [lista]
- Disclaimer se necessario
```

---

## Metriche di Successo

| Metrica | Attuale | Target |
|---------|---------|--------|
| Citazioni non verificabili | ~4 per risposta | 0 |
| Menzione solidarietà su mutui | 0% | 100% |
| Disclaimer quando tools OFF | No | Sempre |
| Segnalazione argomenti sensibili | No | Automatica |

---

## Risorse

- **System prompt attuale:** `core.responder_personas` (DB)
- **Customer router:** `lexe-core/gateway/customer_router.py`
- **Pipeline orchestrata:** `lexe-orchestrator/phases/`
- **Tools config:** `lexe-orchestrator/tools/`

---

*Documento: PROMPT-OPTIMIZATION-PROPOSAL.md*
*Creato: 2026-02-06*
*Autore: Analisi sessione Claude*
