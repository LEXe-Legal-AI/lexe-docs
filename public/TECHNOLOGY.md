---
owner: product-team
audience: public
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: monthly
sensitivity: public
last_verified: 2026-03-19
---

# LEXe Legal AI - Technology Stack

## Backend

| Component | Technology |
|-----------|-----------|
| **Language** | Python 3.12 |
| **Framework** | FastAPI (async, high-performance) |
| **Database Driver** | asyncpg (native PostgreSQL async) |
| **Data Validation** | Pydantic v2 |
| **Task Orchestration** | Custom multi-agent pipeline |

## Frontend

| Component | Technology |
|-----------|-----------|
| **Framework** | React 18 with TypeScript |
| **State Management** | Zustand (lightweight, selector-based) |
| **Data Fetching** | TanStack Query (caching, deduplication) |
| **Styling** | Tailwind CSS |
| **Build** | Vite |

## Database Layer

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Primary Database** | PostgreSQL 17 | Relational data, tenant isolation |
| **Vector Search** | pgvector | Dense embedding similarity search |
| **Graph Queries** | Apache AGE | Citation graphs, norm relationships |
| **Full-Text Search** | ParadeDB (BM25) | Keyword search with legal tokenization |
| **Hybrid Search** | RRF Fusion | Combines dense vectors + BM25 for optimal recall |

## Cache and Session

| Component | Technology |
|-----------|-----------|
| **Cache** | Valkey (Redis-compatible, open source) |
| **Session Store** | Valkey with TTL |
| **Rate Limiting** | Valkey counters with expiry |

## Authentication and Authorization

| Component | Technology |
|-----------|-----------|
| **Identity Provider** | Logto (self-hosted) |
| **Protocol** | OAuth 2.0 / OpenID Connect |
| **Token Format** | JWT with RS256 signing |
| **Access Control** | Role-Based Access Control (RBAC) |
| **Tenant Isolation** | Row-Level Security (RLS) at database level |

## LLM Integration

| Component | Technology |
|-----------|-----------|
| **Gateway** | LiteLLM (multi-provider abstraction) |
| **Providers** | Multiple tier-1 LLM providers |
| **Model Selection** | Benchmark-driven, role-specific assignment |
| **Streaming** | Server-Sent Events (SSE) |
| **Embeddings** | OpenAI text-embedding-3-small |

## Observability

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Metrics** | Prometheus | Time-series metrics, alerting |
| **Dashboards** | Grafana | Operational visibility, SLA tracking |
| **Tracing** | Jaeger | Distributed request tracing |
| **Logs** | Loki | Centralized log aggregation |
| **LLM Tracing** | Langfuse | Model performance, cost tracking |

## Infrastructure

| Component | Technology |
|-----------|-----------|
| **Containerization** | Docker Compose |
| **Reverse Proxy** | Traefik v2.11 |
| **TLS** | Let's Encrypt (automatic certificate renewal) |
| **Hosting** | Dedicated servers (EU, GDPR-compliant) |

## Search Architecture

LEXe uses a hybrid search strategy that combines three retrieval methods for maximum recall and precision:

1. **Dense Vector Search** (pgvector) -- captures semantic similarity even when exact terms differ
2. **BM25 Full-Text Search** (ParadeDB) -- excels at exact legal terminology matching
3. **Reciprocal Rank Fusion (RRF)** -- merges results from both methods into a single ranked list

This hybrid approach ensures that both conceptually related and terminologically precise results surface in every query.

## Graph-Enhanced Knowledge

Apache AGE enables citation graph traversal across the legal knowledge base. Norms, jurisprudence, and doctrine are connected through typed edges (CITES, AMENDS, REPEALS, INTERPRETS), allowing the system to follow citation chains and discover related authorities that keyword search alone would miss.

---

*Built for legal professionals. Engineered for reliability.*
