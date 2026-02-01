# Mappa LEXE: lexe-docs

> Generato automaticamente - 2026-02-01
> Repo path: C:\PROJECTS\lexe-genesis\lexe-docs
> **Ambiente di riferimento: stage**
> Altri ambienti: prod (stabile attuale), local (sviluppo)
> Generato con: DocArchitect + 1 Explore parallelo

## 1. Architettura

### Stack Tecnologico
| Aspetto | Valore |
|---------|--------|
| Tipo | Documentation repository |
| Formato | Markdown (.md) |
| Hosting | GitHub |
| Porta | N/A |

### Struttura Principale
```
lexe-docs/
├── README.md          # 6.4 KB - Index principale
├── STATUS.md          # 3.1 KB - Stato piattaforma
└── .git/              # Git repository
```

### File Critici
| File | Funzione | Size | Priorita Analisi |
|------|----------|------|------------------|
| README.md | Index, architettura, repos | 6.4 KB | Alta |
| STATUS.md | Stato infra, roadmap | 3.1 KB | Alta |

## 2. Riferimenti LEO (Legacy)

### Critici (da rimuovere/rinominare)
| File | Contenuto | Azione |
|------|-----------|--------|
| STATUS.md | `LEO Logto migrato a logto.leo.itconsultingsrl.com` | Documentazione storica OK |
| README.md | `hetzner_leo_key` SSH key | Riferimento infra condiviso OK |

### Condivisi (OK - mantenere)
- Riferimenti a separazione da LEO (documentazione storica)
- SSH key `hetzner_leo_key` (server condiviso con LEO)
- Differenze architetturali LEO vs LEXE

### Statistiche
- File con riferimenti LEO: 2
- Riferimenti critici: 0 (tutti documentazione storica)
- Riferimenti condivisi: 5+

## 3. Contenuto Documentazione

### README.md - Index
Contiene:
- **Repository Map**: 10 repos con porte e descrizioni
- **Architecture Diagram**: ASCII flow webchat -> core -> orchestrator -> litellm
- **Memory Architecture**: L1 (Valkey) -> L2 (PostgreSQL) -> L3 (pgvector)
- **Legal Tools Table**: 4 strumenti (Normattiva, EUR-Lex, Massimari, Ricerca)
- **ORCHIDEA Pipeline**: 6 fasi (A, B, C, D, E, F)
- **Links**: A documenti futuri (ARCHITECTURE.md, API-REFERENCE.md)

### STATUS.md - Stato Piattaforma
Contiene:
- **Infrastructure Status**: 8 container healthy
- **Services URLs**: ai.lexe.pro, api.lexe.pro, auth.lexe.pro, llm.lexe.pro
- **Completed Work**: Phase 0-4 separazione da LEO
- **LEO References Cleanup**: 7 file disabilitati/refactorati
- **In Progress**: FASE 1-6 da POSTSTART.md
- **Known Issues**: 404 su endpoint LEO-specific

## 4. Documentazione Mancante

### Referenziata ma non esistente
| File | Referenziato in | Scopo previsto |
|------|-----------------|----------------|
| docs/ARCHITECTURE.md | README.md | Architettura dettagliata |
| docs/API-REFERENCE.md | README.md | Schema API completo |

### Suggerita ma non pianificata
- Schema/database diagram
- Deployment troubleshooting guide
- Tenant isolation implementation
- Tool development guide

## 5. Documentazione Correlata (in altri repos)

### In lexe-genesis root
| File | Size | Contenuto |
|------|------|-----------|
| CLAUDE.md | ~4 KB | Context progetto, stats, commands |
| STARTUP.md | ~2 KB | Istruzioni Claude, best-practices ref |
| POSTSTART.md | ~5 KB | Piano MVP 6 fasi, test checklist |
| best-practices.md | 11.5 KB | Anti-pattern SSH/Docker/DB/Traefik |

### In lexe-privacy
| File | Contenuto |
|------|-----------|
| api/ENDPOINTS.md | API documentation |
| api/EXAMPLES.md | Usage examples |
| api/IMPLEMENTATION_SUMMARY.md | Implementation details |
| engines/spacy/README.md | spaCy engine docs |
| pipeline/README.md | Pipeline architecture |
| benchmarking/README.md | Benchmarking guide |

### In lexe-max
| File | Contenuto |
|------|-----------|
| KB_SETUP.md | Knowledge base setup |

## 6. Architettura Documentata

### Stack Diagram (da README.md)
```
ai.lexe.pro (webchat)
    | SSE/REST
api.lexe.pro (lexe-core: Gateway + OIDC Logto + Identity)
    |
lexe-orchestrator (ORCHIDEA A->B->C->D->F)
    +-- lexe-max (Legal Tools)
    +-- lexe-memory (L1-L3)
        |
llm.lexe.pro (lexe-litellm: OpenRouter Gateway)
```

### Memory Layers
| Layer | Storage | TTL | Scopo |
|-------|---------|-----|-------|
| L1 Working | Valkey | 7d | Session context |
| L2 Episodic | PostgreSQL | Permanent | User history |
| L3 Semantic | pgvector | Permanent | Similarity search |

### ORCHIDEA Pipeline
| Fase | Status | Descrizione |
|------|--------|-------------|
| A | SI | Intent classification |
| B | SI | Entity capture |
| C | SI | KB + Memory retrieval |
| D | SI | LLM drafting |
| E | NO | Guardrails (P2) |
| F | SI | Memory writeback |

## 7. Note e Problemi

### Da Fixare (P1 - Urgente)
- [ ] Creare docs/ARCHITECTURE.md referenziato nel README
- [ ] Creare docs/API-REFERENCE.md referenziato nel README

### Da Fixare (P2 - Importante)
- [ ] Aggiungere schema database
- [ ] Documentare tenant isolation
- [ ] Aggiungere troubleshooting guide

### Suggerimenti Miglioramento
- Migrare a framework documentazione (Docusaurus, MkDocs)
- Aggiungere versioning per API changes
- Aggiungere changelog automatico

## 8. Qualita Documentazione

### Punti di Forza
- Documentazione pragmatica focalizzata su MVP
- Focus operativo (comandi, checklist, status)
- Aggiornata frequentemente (ultimo: 2026-02-01)
- Architettura ben visualizzata con ASCII diagrams

### Aree di Miglioramento
- Manca documentazione API formale
- Manca documentazione sviluppatori
- Manca onboarding guide
- Riferimenti a file futuri non ancora creati

---

*Mappa generata: 2026-02-01*
*Terminale 5 - DocArchitect*
