---
owner: product-team
audience: public
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: monthly
sensitivity: public
last_verified: 2026-03-20
---

# LEXe Legal AI - Platform Overview

## What It Is

LEXe is an AI-first legal research platform built for real legal work, not generic chat demos.

- Ask a legal question in natural language.
- LEXe decides which sources to query, which agents to activate, and how much verification is required.
- The platform returns a grounded answer with citations, confidence, and traceable evidence.
- No manual tool picking in the core user experience.
- No operator-style search consoles for the end user.
- No black-box "trust me" output.

### Marketing-ready description

- Legal AI that does the research, shows the evidence, and makes its confidence explicit.
- Multi-agent legal analysis with source-backed answers instead of single-model improvisation.
- An AI-native platform for legal teams that want speed without giving up rigor.
- Built for transparent legal reasoning, not just polished text generation.

## Core Value Proposition

- **AI-first by design**: the user asks; the platform orchestrates.
- **Multi-agent orchestration**: intent detection, planning, research, verification, and synthesis are separated into specialized capabilities.
- **LLM-agnostic architecture**: model routing is abstracted behind LiteLLM and role aliases, so provider choice is operational, not structural.
- **Measured, not hand-waved**: model selection, latency, confidence, tool usage, and quality signals are benchmarked and observed.
- **Transparent by default**: citations, confidence bands, tool progress, and trace hashes make the system inspectable.
- **Grounded on authoritative sources**: official legislation, legal knowledge bases, jurisprudence, doctrine, and vigenza checks.

## Key Features

- **Natural-language legal research**
  - Users start from a question, not from a database or a tool menu.
  - LEXe maps the request to the right research path automatically.

- **Adaptive pipeline depth**
  - Simple requests stay fast.
  - Complex requests trigger full multi-source research and verification.

- **Multi-source legal retrieval**
  - Normattiva for Italian legislation.
  - EUR-Lex for EU law.
  - Brocardi and legal commentary sources for doctrinal context.
  - Internal knowledge bases for normative articles and Cassazione massime.

- **Citation-backed responses**
  - Every important claim can be linked back to a source.
  - Citations are part of the product logic, not decorative UI.

- **3-band confidence system**
  - `VERIFIED`
  - `CAUTION`
  - `LOW CONFIDENCE`
  - Confidence is derived from evidence quality, coverage, consistency, and verification outcomes.

- **Parallel legal research**
  - Multiple sources are queried concurrently.
  - Evidence is fused, deduplicated, and scored before synthesis.

- **Vigenza-aware answers**
  - The platform checks whether a norm is in force and handles amendment-sensitive questions more safely.

- **Real-time streaming**
  - Users see the answer form in real time.
  - The platform can expose research progress, tool activity, and completion metadata while streaming.

- **Tenant-aware platform controls**
  - Per-tenant model roles, plan limits, enabled sources, personas, and feature flags.
  - Isolation is enforced at the platform level, not bolted on later.

- **Memory and continuity**
  - Conversation memory and cross-session context improve continuity without forcing the user to restate everything.

- **Compliance-native foundations**
  - GDPR consent flow.
  - Auditability.
  - EU-hosted infrastructure.
  - Tenant isolation and role-based access control.

## AI-First Means No Manual Tool UX

- The primary interaction is conversation, not tool operation.
- Users do not choose "run Normattiva" or "run KB search" manually.
- Users do not need to know which database contains the answer.
- The orchestration layer selects tools, controls depth, merges evidence, and verifies outputs automatically.
- Administrative controls exist for governance and configuration, not as a substitute for the product experience.

## How It Works

### Architecture At A Glance

| Layer | What It Does | Why It Matters |
|------|------|------|
| **Interface Layer** | Webchat, API, streaming responses, auth-aware request entry | Gives users a simple conversational entry point |
| **Core Platform Layer** | Tenant resolution, conversation handling, memory integration, SSE translation, limits enforcement | Keeps the product coherent, secure, and multi-tenant |
| **Orchestration Layer** | Strategy selection and pipeline control for different query depths | Routes each request to the right level of intelligence |
| **Capability Layer** | Intent classifier, planner, executor, verifier, synthesizer, agentic loops | Splits legal work into specialized AI functions |
| **Tool and Knowledge Layer** | Normattiva, EUR-Lex, commentary sources, internal KB, graph search, vigenza checks | Grounds the answer in retrievable legal evidence |
| **LLM Gateway Layer** | LiteLLM role routing across providers and models | Makes the stack provider-flexible and benchmark-driven |
| **Observability and Governance Layer** | Metrics, traces, evaluations, audit logs, assessment dashboards | Makes quality, cost, and reliability visible |

### Compact Request Flow

1. **User asks a question**
   - The request enters through chat or API.

2. **LEXe builds context**
   - Auth, tenant, conversation state, memory context, and request metadata are assembled.

3. **Intent and depth are classified**
   - LEXe decides whether the request is conversational, direct, simple, standard, or complex.

4. **A research plan is generated**
   - The system identifies relevant norms, domains, and sources to query.

5. **Research agents execute in parallel**
   - Legal sources are queried concurrently across legislation, jurisprudence, doctrine, and knowledge bases.

6. **Evidence is fused and checked**
   - Duplicates are merged.
   - Coverage is measured.
   - Conflicts and weak support are surfaced.
   - Verification logic decides whether the answer is solid, partial, or risky.

7. **A grounded answer is synthesized**
   - The response is generated from evidence, not from raw model recall alone.
   - Citations and confidence travel with the output.

8. **The turn is measured and logged**
   - Traceability, latency, tool usage, quality signals, and evaluation hooks are preserved for audit and improvement.

### Main Components

- **lexe-core**
  - API gateway, auth middleware, tenant-aware runtime, streaming contract, limits, and orchestration entry point.

- **lexe-tools-it**
  - Legal tools and capabilities for research, verification, and execution against legal sources.

- **lexe-memory**
  - Multi-layer memory for turn context, conversation continuity, and retrieval-augmented context.

- **lexe-max**
  - Legal knowledge base with PostgreSQL, vector search, graph traversal, and full-text retrieval.

- **LiteLLM gateway**
  - Central model abstraction, provider routing, and spend-aware LLM access.

- **Webchat and admin**
  - Webchat for the AI-first product experience.
  - Admin for governance, configuration, and observability.

### Data Flow Summary

```text
User question
  -> LEXe interface
  -> tenant/auth/context resolution
  -> intent + strategy selection
  -> planning
  -> parallel legal research
  -> evidence fusion + verification
  -> grounded synthesis
  -> streaming response with citations + confidence
  -> trace + metrics + evaluation signals
```

## Why LEXe Feels Different From Competitors

- **Not a generic chatbot with a legal prompt**
  - LEXe is built around a legal pipeline, not just a stronger system prompt.

- **Not a manual search cockpit**
  - The product does not ask users to operate a toolbox.
  - The AI orchestrates the workflow end to end.

- **Not single-agent by default**
  - Different phases handle classification, planning, retrieval, verification, and synthesis.
  - This separation improves control, debuggability, and output quality.

- **Not tied to one model vendor**
  - The architecture is provider-agnostic.
  - The active production choice is benchmark-driven, but the platform is not structurally locked to it.

- **Not opaque**
  - Confidence is explicit.
  - Citations are retrievable.
  - Tool activity and trace identifiers make the system inspectable.

- **Not blind to quality**
  - Benchmarking, metrics, evaluations, alerting, and traceability are first-class platform concerns.

## Competitive Differentiators

| Dimension | LEXe | Typical AI Assistant |
|------|------|------|
| **Primary UX** | Conversational, AI-orchestrated legal workflow | Generic chat with optional tools |
| **Research model** | Multi-agent, multi-phase, multi-source | Usually single-pass generation |
| **Source grounding** | Built into pipeline and citations | Often partial or post-hoc |
| **Confidence model** | Explicit 3-band evidence-based confidence | Implicit or model-style certainty |
| **Provider strategy** | LLM-agnostic architecture with role routing | Often tied to one provider or one model |
| **Observability** | Metrics, trace hashes, evaluations, dashboards | Limited visibility |
| **Governance** | Tenant isolation, plan limits, feature flags, admin controls | Often shallow workspace settings |
| **Legal specialization** | Italian legal sources and workflows | General-purpose knowledge |

## Transparency and Measurement

- **Benchmark-driven model selection**
  - 11 candidate models evaluated.
  - 6 benchmark runs.
  - 2 independent judges.
  - Role-specific model assignment instead of one-model-for-everything.

- **Operational observability**
  - Core request, pipeline, SSE, memory, and confidence metrics are tracked.
  - Tool execution is measured separately.
  - Dashboards and alerts monitor latency, error rates, confidence drift, and system health.

- **Traceable support and QA**
  - Each completed turn can carry a trace hash.
  - Quality assessments, evaluations, and event histories support debugging and product improvement.

- **Visible confidence**
  - Confidence is shown as a product signal.
  - It is tied to evidence quality and verification, not to rhetorical certainty.

## What Buyers and Partners Should Understand Quickly

- LEXe is a platform, not a wrapper.
- The AI is the interface, not an add-on.
- The orchestration is multi-agent and evidence-driven.
- The model layer is abstracted, benchmarked, and replaceable.
- The product is designed to make legal AI inspectable, governable, and operationally real.

## Short Positioning

- **For legal teams**: faster research with visible evidence and less black-box risk.
- **For firms and enterprises**: configurable, multi-tenant legal AI with governance built in.
- **For partners and integrators**: API-ready legal intelligence on top of a modular, provider-flexible architecture.

---

*LEXe Legal AI: AI-first legal research with orchestration, evidence, and transparency built into the platform itself.*
