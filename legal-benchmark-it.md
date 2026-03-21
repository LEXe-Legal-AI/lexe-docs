# LexBench-IT: Legal Benchmark per il Diritto Italiano

> Design document per un benchmark sistematico di valutazione AI legale italiana.
> Versione 0.1 — 2026-03-21

---

## 1. Gap: nessun benchmark per il diritto italiano

I benchmark esistenti (LegalBench Stanford, LEXTREME, LexGLUE) sono per common law / multilingue generico. Il diritto italiano richiede valutazione specifica su: vigenza, gerarchia fonti, codificazione, giurisprudenza con peso differenziato (SU > Cassazione > Appello > Tribunale).

## 2. Dimensioni di valutazione

| Dimensione | Peso | Cosa misura |
|-----------|------|-------------|
| Accuratezza normativa | 30% | Norme corrette, esistenti, vigenti |
| Completezza (Recall) | 25% | Norme rilevanti citate |
| Grounding | 20% | Affermazioni ancorate a fonti |
| Qualita risposta | 15% | Utilita professionale |
| Assenza hallucination | 10% | Norme/sentenze inventate |

## 3. Tassonomia

- 8 aree: CIV, PEN, AMM, LAV, TRI, EUR, SOC, FAM
- 6 tipi: parere, ricerca, contenzioso, compliance, risk assessment, redazione
- 3 complessita: SIMPLE (1-2 norme), STANDARD (3-6), COMPLEX (7+)
- Matrice target: 144 celle, 288+ prompt

## 4. Dataset methodology

- 40% audit traces reali (anonimizzati)
- 25% prompt da avvocati
- 20% esami di stato / concorsi
- 15% casi giurisprudenziali noti
- Gold standard: annotazione umana da 2 giuristi per 50 prompt critici

## 5. Metriche specifiche diritto italiano

- **Vigenza**: norma abrogata citata senza indicazione = errore grave
- **Gerarchia fonti**: Costituzione > legge > regolamento > UE nel suo ambito
- **Autorita giurisprudenziale**: Corte Cost = 5, SU = 4, Cass = 3, Appello = 2, Tribunale = 1
- **Cross-reference accuracy**: rimandi tra norme corretti

## 6. Kryptonite Round 1 baseline

Avg conf 79.6, ricerca forte (88.3), parere debole (67.4), compliance gap (71.2).

## 7. Roadmap

- Phase 1 (Q2 2026): 200 prompt + gold standard parziale
- Phase 2 (Q3 2026): annotazione umana 50 prompt critici
- Phase 3 (Q4 2026): CI/CD automated regression testing
- Phase 4 (Q1 2027): multi-model comparison (LEXE vs ChatGPT vs Gemini vs Claude)

---

*LexBench-IT v0.1 — 2026-03-21*
