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

# LEXe Legal AI - Features

## Multi-Agent Legal Research

LEXe deploys specialized research agents in parallel across three domains: **norms**, **jurisprudence**, and **doctrine**. Each agent searches its domain independently, then results are fused into a single, coherent legal analysis. This three-wave architecture ensures comprehensive coverage while keeping response times practical.

## 3-Band Confidence System

Every response includes a transparent confidence assessment:

| Band | Meaning |
|------|---------|
| **VERIFIED** | Strong source support from multiple authoritative references |
| **CAUTION** | Partial support; some sources found but coverage is incomplete |
| **LOW CONFIDENCE** | Limited source backing; the user should verify independently |

Confidence is computed from source authority, cross-reference consistency, and citation verification -- never from the language model's self-assessment alone.

## Interactive Citation Badges

Cited sources appear as numbered badges inline with the response text. Clicking a badge opens a popover with the original excerpt, source metadata, and a direct link to the authoritative publication. Every claim is traceable back to its origin.

## Specialized Legal Tools

LEXe integrates eight dedicated legal data sources:

- **Normattiva** -- official Italian legislation database
- **EUR-Lex** -- European Union law and case law
- **Brocardi** -- annotated Italian legal codes with commentary
- **KB Massimario** -- over 38,000 curated maxims from the Italian Court of Cassation
- **KB Normativa** -- 5,800+ indexed articles from codified Italian law
- **Legal Web Search** -- targeted web retrieval for doctrine and commentary
- **Graph Explorer** -- citation graph traversal across norms and jurisprudence
- **Vigenza Checker** -- verification of norm validity and amendment history

## GDPR-Compliant Consent System

A built-in consent management system with full audit trail. Users must accept terms before data processing begins. Every consent event is logged immutably for compliance reporting.

## Multitenancy with Plan-Based Limits

Each organization operates in a fully isolated tenant with configurable resource limits:

- **User seats** -- maximum active users per tenant
- **Daily conversations** -- conversation caps per billing period
- **Token budgets** -- daily token consumption limits
- **Custom personas** -- tenant-specific AI behavior profiles

Three standard plans are available (Free Demo, Starter, Professional), with custom enterprise plans on request.

## Admin Panel

A comprehensive management interface with 15 dedicated pages:

- Dashboard with real-time usage metrics
- Tenant provisioning and configuration
- User management with role-based access
- Model catalog and routing configuration
- Tool enablement per tenant
- Persona editor
- Conversation browser
- Audit log viewer
- Evaluation analytics
- System health monitoring

## Evaluation System

Users rate responses on a 1-to-5 star scale. Ratings feed into a continuous improvement loop that tracks quality trends across models, tools, and query types. Aggregated evaluation data is available in the admin panel for quality assurance.

## Memory System

LEXe maintains context across five memory layers:

| Layer | Scope |
|-------|-------|
| **L0** | Current conversation turn |
| **L1** | Full conversation history |
| **L2** | Cross-conversation user context |
| **L3** | Tenant-wide shared knowledge |
| **L4** | Knowledge graph (entities, relations, precedents) |

Memory enables the assistant to recall prior research, avoid redundant lookups, and build on previous analyses within and across sessions.

## Real-Time Streaming

Responses are delivered via Server-Sent Events (SSE) with pipeline transparency. Users see which tools are being invoked, which sources are being consulted, and how the final response is assembled -- all in real time.

---

*LEXe Legal AI -- La legge a portata di AI.*
