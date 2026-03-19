# Memory Lite (Quick Fix v1)

> Sistema memoria semplificato per LEXE chat senza orchestrator.
> Implementato: 2026-02-03

---

## Overview

**Memory Lite** è l'implementazione memoria base integrata direttamente in `customer_router.py`.
Funziona senza dipendenza dall'orchestrator ORCHIDEA.

---

## Caratteristiche

### Retrieval (Semantico)
- Endpoint: `lexe-memory /memory/search`
- Similarity threshold: `0.3`
- Max memories: `5` per query
- Layer: L3 (semantic)
- Injection: aggiunto al system prompt

### Storage (Pattern-based)
- Estrazione: **solo regex** (no LLM)
- Pattern rilevati:
  - `mi chiamo X`, `sono X`, `io sono X`
  - `my name is X`, `I'm X`, `I am X`
  - `chiamami X`, `call me X`
- Target layer: L3 (semantic)
- Content type: `fact`

### Conversation Context
- **Messages per thread: 10** (configurabile)
- Ordine: cronologico (oldest first)
- Roles: user, assistant

---

## Configurazione

| Parametro | Valore | File |
|-----------|--------|------|
| `max_messages` | 10 | `customer_router.py:get_recent_messages()` |
| `limit` (memories) | 5 | `customer_router.py:retrieve_memory_context()` |
| `min_similarity` | 0.3 | `customer_router.py:retrieve_memory_context()` |
| `MEMORY_TIMEOUT` | 5.0s | `customer_router.py` |

---

## Limiti

| Aspetto | Memory Lite | ORCHIDEA (futuro) |
|---------|-------------|-------------------|
| Fact extraction | Regex (solo nome) | LLM-based |
| Memory layers | Solo L3 | L0→L1→L2→L3 |
| Retrieval | Semantic base | RAG + reranking |
| Consolidation | Nessuna | Temporal decay |

---

## Flow

```
User Message
    │
    ▼
┌─────────────────────────────────┐
│ 1. get_recent_messages(10)      │ ← Context window
│ 2. retrieve_memory_context(5)   │ ← Semantic search L3
│ 3. Build system prompt          │
│ 4. LiteLLM streaming            │
│ 5. store_memory_facts()         │ ← Pattern extraction
└─────────────────────────────────┘
    │
    ▼
Assistant Response
```

---

## Backlog

- [ ] **P2**: Aumentare `max_messages` a 20 per thread più lunghi
- [ ] **P2**: Configurare via env var `LEXE_MEMORY_CONTEXT_SIZE`
- [ ] **P1**: LLM-based fact extraction (richiede orchestrator)
- [ ] **P3**: Memory summary per conversazioni > 50 messaggi

---

*Ultimo aggiornamento: 2026-02-03*
