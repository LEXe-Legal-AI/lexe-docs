# LEXE Documentation

> Documentazione tecnica per LEXE Legal AI Platform

---

## Quick Links

| Documento | Descrizione |
|-----------|-------------|
| [STATUS.md](STATUS.md) | Stato attuale piattaforma |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | Architettura sistema |
| [API-REFERENCE.md](docs/API-REFERENCE.md) | Riferimento API |

---

## Repository Map

| Repository | Descrizione | Stato |
|------------|-------------|-------|
| **lexe-core** | Gateway, Auth OIDC, API | Active |
| **lexe-orchestrator** | ORCHIDEA Pipeline, Tools | Active |
| **lexe-memory** | Memory L1-L3 (Valkey, PG, pgvector) | Active |
| **lexe-max** | Legal Tools (Normattiva, EUR-Lex, Massimari) | Active |
| **lexe-webchat** | User Chat Interface | Active |
| **lexe-infra** | Docker Compose Infrastructure | Active |
| **lexe-docs** | Documentation | This repo |

---

## Production URLs

| Service | URL |
|---------|-----|
| Webchat | https://ai.lexe.pro |
| API | https://api.lexe.pro |
| Auth (Logto) | https://auth.lexe.pro |
| LLM Gateway | https://llm.lexe.pro |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     ai.lexe.pro                              │
│                   (lexe-webchat)                             │
└──────────────────────────┬──────────────────────────────────┘
                           │ SSE/REST
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    api.lexe.pro                              │
│                    (lexe-core)                               │
│  ┌─────────┐  ┌──────────────┐  ┌─────────────────────┐    │
│  │ Gateway │  │ OIDC (Logto) │  │ Identity/Contacts   │    │
│  └────┬────┘  └──────────────┘  └─────────────────────┘    │
└───────┼─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                   lexe-orchestrator                          │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                 ORCHIDEA Pipeline                     │   │
│  │  A:Intent → B:Capture → C:Retrieve → D:Draft → F:FB  │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌────────────────┐  ┌───────────────┐                      │
│  │  Legal Tools   │  │  lexe-memory  │                      │
│  │  (lexe-max)    │  │  L1-L2-L3     │                      │
│  └────────────────┘  └───────────────┘                      │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                    llm.lexe.pro                              │
│                   (lexe-litellm)                             │
│            ┌──────────────────────────┐                      │
│            │ OpenRouter Gateway       │                      │
│            │ - lexe-primary (Mistral) │                      │
│            │ - lexe-fast (Gemini)     │                      │
│            └──────────────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Memory Architecture (L1-L3)

```
L1 Working (Valkey)     → Cache sessione, context window, 7d TTL
L2 Episodic (PostgreSQL) → Fatti, topic summaries, permanent
L3 Semantic (pgvector)   → Embeddings per similarity search
```

---

## Legal Tools (lexe-max)

| Tool | Fonte | Descrizione |
|------|-------|-------------|
| Normattiva | normattiva.it | Legislazione italiana |
| EUR-Lex | eur-lex.europa.eu | Legislazione EU |
| Massimari | KB interna | Giurisprudenza annotata |
| Ricerca Web | DuckDuckGo | Ricerca generalista |

---

## ORCHIDEA Pipeline (Minimo Sindacale)

| Fase | Nome | Attiva | Descrizione |
|------|------|--------|-------------|
| A | Intent | SI | Classificazione intent utente |
| B | Capture | SI | Estrazione entità |
| C | Retrieve | SI | KB + Memory retrieval |
| D | Drafting | SI | Generazione risposta LLM |
| E | Verify | NO | Guardrails (P2) |
| F | Feedback | SI | Memory writeback |

---

## Development

```bash
# Clone all repos
for repo in lexe-core lexe-orchestrator lexe-memory lexe-max lexe-webchat lexe-infra; do
  git clone git@github.com:LEXe-Legal-AI/$repo.git
done

# Local development
cd lexe-infra
docker compose up -d
```

---

## Deploy (Manual)

```bash
ssh -i ~/.ssh/hetzner_leo_key root@49.12.85.92
cd /opt/lexe-platform/lexe-infra
docker compose pull && docker compose up -d
```

---

*Ultimo aggiornamento: 2026-02-01*
