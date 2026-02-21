# Proposta Nomi UI — LEXe Webchat

> Tutti i nomi visibili all'utente, dove stanno, e proposta di sostituzione.

---

## 1. Header Pipeline (visibile durante l'analisi)

### LEGIS Agent — `LegisResearchPanel.tsx:71`

| Attuale | Proposta | Note |
|---------|----------|------|
| **LEGIS Agent** | **Ricerca legale** | "Agent" e gergo tecnico. L'utente deve vedere cosa fa, non come si chiama |

### LEXORC — `OrchidPulse.tsx:104`

| Attuale | Proposta | Note |
|---------|----------|------|
| **LEXORC** | **Analisi approfondita** | Nome interno del sistema multi-agente. L'utente non sa cos'e "LEXORC" |

---

## 2. Fasi pipeline (label animate durante l'esecuzione)

### LEGIS phases — `LegisResearchPanel.tsx:33-38`

| Attuale | Proposta | Note |
|---------|----------|------|
| `Analisi della domanda...` | `Comprensione della domanda...` | Piu naturale |
| `Ricerca fonti legali...` | `Ricerca nelle fonti...` | Piu breve |
| `Verifica citazioni...` | `Verifica delle fonti...` | "Citazioni" e tecnico |
| `Sintesi della risposta...` | `Preparazione della risposta...` | "Sintesi" e troppo accademico |

### LEXORC phases — `OrchidPulse.tsx:19-65`

| Attuale | Proposta | Note |
|---------|----------|------|
| `Analisi richiesta...` | `Comprensione della domanda...` | Coerenza con LEGIS |
| `Pianificazione ricerca...` | `Pianificazione...` | Piu snello |
| `Ricerca in corso...` | `Ricerca nelle fonti...` | Coerenza |
| `Verifica fonti...` | `Verifica delle fonti...` | Coerenza |
| `Sintesi risposta...` | `Preparazione della risposta...` | Coerenza |
| `Completato` | `Completato` | OK cosi |

---

## 3. Nomi agenti LEXORC (task card e work plan board)

### Agent names — `WorkPlanBoard.tsx:34-40` + `TaskCard.tsx:154`

I nomi degli agenti arrivano dal backend (`work_plan.py:91-97`) via SSE e sono visualizzati direttamente nel frontend come `{node.agent}`.

| Attuale (backend) | Label backend (non usata in UI) | Proposta UI | File backend |
|--------------------|---------------------------------|-------------|--------------|
| **NormAgent** | Ricerca normativa | **Normativa** | `work_plan.py:92` |
| **CaseLawAgent** | Giurisprudenza | **Giurisprudenza** | `work_plan.py:93` |
| **DoctrineAgent** | Dottrina | **Dottrina** | `work_plan.py:94` |
| **Auditor** | Verifica evidenze | **Verifica** | `work_plan.py:95` |
| **Synthesizer** | Sintesi risposta | **Redazione** | `work_plan.py:96` |

**Nota**: Il frontend mostra `node.agent` (es. "NormAgent") invece di `node.detail` o `AGENT_DESCRIPTIONS[name].label`. La label italiana esiste gia nel backend ma non viene usata nel posto giusto.

**Due opzioni di fix**:
- **Opzione A (frontend)**: Creare un mapping `AGENT_DISPLAY_NAMES` in `WorkPlanBoard.tsx` che traduce il nome interno in label italiana
- **Opzione B (backend)**: Mandare un campo `display_name` nel SSE event e usare quello nel frontend

---

## 4. Task detail (sotto il nome agente nelle card)

### Task descriptions — `work_plan.py:143-162`

| Attuale | Proposta | Note |
|---------|----------|------|
| `{query utente}` (per ricerca) | `{query utente}` | OK, mostra cosa sta cercando |
| `Verifica anti-allucinazione` | `Verifica coerenza fonti` | "Anti-allucinazione" spaventa l'utente |
| `Sintesi parere strutturato` | `Redazione della risposta` | Piu semplice |
| `Verifica evidenze raccolte` | `Controllo delle fonti` | "Evidenze" e tecnico |

---

## 5. Complexity badge — `WorkPlanBoard.tsx:43-75`

| Attuale | Proposta A (italiano) | Proposta B (neutro) | Note |
|---------|----------------------|---------------------|------|
| **Flash** | Rapida | Flash | OK in entrambi |
| **Standard** | Standard | Standard | OK |
| **Deep** | Approfondita | Approfondita | "Deep" non dice nulla |
| **Strategic** | Strategica | Strategica | "Strategic" non dice nulla |

---

## 6. Verification badge — `LegisVerificationBadge.tsx:26-54`

| Attuale | Proposta | Note |
|---------|----------|------|
| **VERIFICATO** | **Fonti verificate** | Piu chiaro su cosa e stato verificato |
| **ATTENZIONE** | **Verificato con riserva** | "ATTENZIONE" e allarmante |
| **NON VERIFICATO** | **Verifica incompleta** | "NON VERIFICATO" suona come un fallimento |

Formato attuale: `{confidence}% VERIFICATO` → Proposta: `{confidence}% Fonti verificate`

---

## 7. Verification checklist — `VerificationChecklist.tsx:168-169`

| Attuale | Proposta | Note |
|---------|----------|------|
| `Verifica evidenze` | `Controllo fonti` | Coerenza con il resto |
| `({revealed}/{total})` | `({revealed}/{total})` | OK |

---

## 8. Progress counter — `WorkPlanBoard.tsx:140`

| Attuale | Proposta | Note |
|---------|----------|------|
| `{n}/{total} task completati` | `{n}/{total} completati` | "task" e gergo tecnico |

---

## 9. Evidence count — `TaskCard.tsx:167`

| Attuale | Proposta | Note |
|---------|----------|------|
| `{n} fonti` | `{n} fonti` | OK |

---

## 10. Approfondimenti suggeriti (NBA) — `DeepeningChips.tsx:44`

| Attuale | Proposta | Note |
|---------|----------|------|
| `Approfondimenti suggeriti` | `Vuoi approfondire?` | Piu diretto, invita all'azione |

### Label NBA dal backend — `manus_synthesizer.py:generate_nba()`

| Attuale | Proposta | Note |
|---------|----------|------|
| `Cerca giurisprudenza di merito` | `Quali sono i precedenti di merito?` | Formulato come domanda dell'utente |
| `Approfondisci normativa applicabile` | `Quali norme si applicano?` | Domanda, non comando |
| `Amplia ricerca Cassazione` | `Cosa dice la Cassazione?` | Piu naturale |
| `Verifica normativa europea` | `Ci sono norme europee rilevanti?` | Domanda |
| `Approfondisci art. X ...` | `Cosa prevede l'art. X ...?` | Domanda dall'utente |

**Nota**: Le NBA dovrebbero seguire le stesse regole dei follow-up di LEGIS — formulate dal punto di vista dell'utente, come domande che cliccherebbe.

---

## 11. Riepilogo modifiche per file

### Frontend (lexe-webchat)

| File | Riga | Modifica |
|------|------|----------|
| `LegisResearchPanel.tsx` | 71 | `LEGIS Agent` → label proposta |
| `LegisResearchPanel.tsx` | 33-38 | phaseLabels → label proposte |
| `OrchidPulse.tsx` | 104 | `LEXORC` → label proposta |
| `OrchidPulse.tsx` | 19-65 | phaseConfig labels → proposte |
| `WorkPlanBoard.tsx` | 34-40 | Aggiungere `AGENT_DISPLAY_NAMES` mapping |
| `WorkPlanBoard.tsx` | 43-75 | complexity labels → proposte |
| `WorkPlanBoard.tsx` | 140 | `task completati` → proposta |
| `LegisVerificationBadge.tsx` | 26-54 | statusConfig labels → proposte |
| `VerificationChecklist.tsx` | 169 | `Verifica evidenze` → proposta |
| `DeepeningChips.tsx` | 44 | `Approfondimenti suggeriti` → proposta |

### Backend (lexe-core)

| File | Riga | Modifica |
|------|------|----------|
| `work_plan.py` | 153 | `Verifica anti-allucinazione` → proposta |
| `work_plan.py` | 162 | `Sintesi parere strutturato` → proposta |
| `work_plan.py` | 151 | `Verifica evidenze raccolte` → proposta |
| `manus_synthesizer.py:generate_nba()` | 327-400 | Label NBA → domande utente |

---

## 12. Bug spinner infinito

**Problema**: I cerchi di caricamento girano all'infinito su NormAgent/Auditor/Synthesizer.

**Causa**: Lo spinner di `TaskCard.tsx:68` gira quando `node.status === "running"`. Si ferma solo quando il backend invia un SSE event che cambia lo status a `"completed"` o `"failed"`. Se il backend non invia correttamente l'event di completamento (o il frontend non lo processa), lo spinner resta attivo per sempre.

**Dove indagare**:
1. `useStreaming.ts` — handler di `LEXORC_PLAN_NODE` event: verifica che aggiorni correttamente lo status del nodo nel `workplanStore`
2. `advanced_orchestrator.py` — verifica che emetta `LEXORC_PLAN_NODE` con `status: "completed"` per ogni agente al termine
3. `workplanStore.ts` — verifica che `updatePlanNode()` sostituisca correttamente lo status
4. Race condition possibile: se `LEXORC_PHASE: "complete"` arriva PRIMA degli eventi di completamento dei singoli nodi, i nodi restano in `"running"` per sempre

**Fix probabile**: Quando `LEXORC_PHASE` diventa `"complete"`, fare un bulk update di tutti i nodi ancora in `"running"` → `"completed"` nel `workplanStore`.
