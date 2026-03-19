---
owner: platform-team
audience: internal
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: quarterly
sensitivity: internal
last_verified: 2026-03-19
---

# LEXE Pipeline & Orchestration Reference

Definitive technical reference for the LEXE V2 orchestration architecture, the LEGIS legal pipeline (PRVS), multi-agent research, confidence scoring, and tool runtime hardening.


## Scope / Non-Scope

### In Scope

- V2 3-layer orchestration (`orchestration/`, `gateway/`, `agent/`)
- Binary routing decomposer (CHAT vs Legal)
- DomainEvent system and SSE translation
- Strategies: ToolLoop (terminal) and Legal (delegates to LEGIS)
- Degradation policy and fallback chains
- LEGIS pipeline phases: Intent Detection, Planning, Research, Verification, Synthesis
- Confidence scoring algorithm and 3-band thresholds
- Authority classification for jurisprudence
- Depth budgets (MINIMAL / STANDARD / DEEP)
- Multi-agent research with 3-wave parallel execution (FF-gated)
- Gap remediation and evidence fusion
- Tool runtime hardening (WS1)

### Non-Scope

- Frontend SSE consumption (`lexe-webchat` / `lexe-admin`)
- LiteLLM proxy configuration and model catalog
- Logto authentication and RBAC
- Database schema and migrations
- Deployment and infrastructure (see `lexe-infra`)
- LEXORC pipeline (disabled, files retained but disconnected)
- Super Tool pipeline (disabled, files retained but disconnected)


## 1. Overview: V2 3-Layer Orchestration

The LEXE pipeline uses a strict 3-layer architecture that separates concerns into domain-pure orchestration, gateway boundary adapters, and agent pipeline execution.

```
                         GATEWAY BOUNDARY
                    (HTTP, SSE, persistence)
                              |
                              v
   +--------------------------------------------------+
   |              ORCHESTRATOR (Layer 1)               |
   |    orchestration/orchestrator.py                  |
   |                                                   |
   |  TurnContext --> decompose() --> Strategy.execute()|
   |                                                   |
   |  DomainEvent stream (pure, no SSE)                |
   +--------+-----------------------------------------+
            |
            v
   +--------+-----------------------------------------+
   |              STRATEGIES (Layer 2)                 |
   |    orchestration/strategies/                      |
   |                                                   |
   |  toolloop.py  -- terminal, tool-calling loop      |
   |  legal.py     -- delegates to LEGIS via legis_v2  |
   |                                                   |
   +--------+-----------------------------------------+
            |
            v
   +--------+-----------------------------------------+
   |           AGENT PIPELINE (Layer 3)                |
   |    agent/pipeline.py + agent/*.py                 |
   |                                                   |
   |  Phase 0: Intent Detection                        |
   |  Phase P: Planning                                |
   |  Phase R: Research                                |
   |  Phase V: Verification                            |
   |  Phase S: Synthesis                               |
   |                                                   |
   +--------------------------------------------------+
```

**Data flow**: The gateway builds a `TurnContext` (immutable dataclass, domain-pure) and passes it to `Orchestrator.run()`. The orchestrator calls `decompose()` for strategy selection, then delegates to the chosen strategy's `execute()` method. Strategies yield `DomainEvent` instances (never SSE strings). The gateway boundary's SSE adapter converts `DomainEvent` objects to SSE wire format.

### Key Design Principles

- **Domain purity**: `orchestration/` has zero imports from `gateway/`. All HTTP, SSE, and persistence concerns live at the gateway boundary.
- **Protocol-based adapters**: Strategies consume `ToolKit`, `LLMClient`, `BudgetHandle` protocols (defined in `contracts.py`), implemented in `gateway/adapters.py`.
- **Event-driven**: All communication from strategies to the gateway is via the `DomainEvent` hierarchy. No direct SSE emission from orchestration code.


## 2. Binary Routing

**File**: `orchestration/decomposer.py`

The decomposer implements a binary routing decision: CHAT or Legal. There is no intermediate routing logic.

```
User Message
     |
     v
decompose(ctx: TurnContext)
     |
     +-- pipeline_override? --------> return override strategy
     |
     +-- intent_result present? ----> reuse (avoid double LLM call)
     |
     +-- classify_intent() ----------> IntentResult
     |        |
     |        +-- level == "CHAT" --> "toolloop" strategy
     |        |
     |        +-- else ------------> "legal" strategy
     |
     +-- exception -----------------> "toolloop" (fail-safe)
```

**Routing metadata** returned alongside the strategy name includes: `routing_reason`, `intent_level`, `depth`, `scenario`, and any raw fields from the classifier.

**Fail-safe behavior**: If intent classification fails (timeout, LLM error), the decomposer routes to `"toolloop"` with `uncertain: true`. False-chat is recoverable (user can rephrase); false-legal is a worse UX error.


## 3. DomainEvent System

**File**: `orchestration/events.py`

All orchestration strategies communicate with the gateway through a typed event hierarchy. Events are pure dataclasses with no SSE formatting.

### 13 Core Event Types

| Event Class | `event_type` | Category | Description |
|---|---|---|---|
| `MetaEvent` | `meta` | Lifecycle | Turn metadata (conversation_id, model, mode) |
| `PhaseEvent` | `phase` | Lifecycle | Pipeline phase transition (intent, research, verify, synthesize) |
| `PreprocessingEvent` | `preprocessing` | Lifecycle | Preprocessing steps (memory, persona, tools) |
| `TokenEvent` | `token` | Content | Streaming text token from LLM |
| `ToolCallEvent` | `tool_call` | Content | Tool invocation started (tool_id, name, arguments) |
| `ToolResultEvent` | `tool_result` | Content | Tool execution completed (status, result, duration_ms) |
| `ToolEndEvent` | `tool_end` | Content | Safety belt: exactly one per tool_id |
| `ToolRunStartEvent` | `tool_run_start` | Content | Batch of tool calls starting (count) |
| `ToolRunEndEvent` | `tool_run_end` | Content | Entire tool run complete (stats) |
| `ErrorEvent` | `error` | Error | Orchestration error (code, recoverable flag) |
| `DoneEvent` | `done` | Lifecycle | Turn completed successfully |
| `PipelineDataEvent` | `pipeline_data` | Pipeline | Pipeline-specific data (legis_intent, agent_plan, legis_verify, legis_source_progress) |

### Event Hierarchy

```
DomainEvent (base)
  |
  +-- MetaEvent
  +-- PhaseEvent
  +-- PreprocessingEvent
  +-- TokenEvent
  +-- ToolCallEvent
  +-- ToolResultEvent
  +-- ToolEndEvent
  +-- ToolRunStartEvent
  +-- ToolRunEndEvent
  +-- ErrorEvent
  +-- DoneEvent
  +-- PipelineDataEvent
```

**PipelineDataEvent** carries the original SSE payload dict for frontend backward compatibility. The SSE adapter serializes `event.data` directly. This bridges the LEGIS pipeline (which still emits SSE strings internally) to the DomainEvent system via `legis_v2.py`.


## 4. Strategies

### 4.1 ToolLoop Strategy (Terminal)

**File**: `orchestration/strategies/toolloop.py`

- **Role**: General-purpose conversational tool-calling loop for CHAT-classified queries.
- **Fallback chain**: Empty (terminal strategy). Cannot fall back further.
- **Requires**: `llm_client` + `tool_kit` on `TurnContext`.
- **Execution**: Resolves tool definitions and policy, builds message history, delegates to `run_tool_loop()` in `tool_runtime.py`.
- **Model**: Uses `model_roles.primary` (currently `lexe-primary`).

### 4.2 Legal Strategy

**File**: `orchestration/strategies/legal.py`

- **Role**: Unified legal pipeline. Replaces the 3 deprecated strategies (legis, lexorc, super_tool).
- **Fallback chain**: `["toolloop"]` -- on total failure, degrades to conversational.
- **On total failure**: `"toolloop"`.
- **Execution**: Delegates to `legis_pipeline_native()` in `agent/legis_v2.py`, which wraps the full LEGIS PRVS pipeline and converts SSE output to DomainEvents.

### 4.3 Per-Capability Degradation (Legal)

| Capability | Fallback | Max Retries | Timeout | Circuit Breaker | Skip on Failure |
|---|---|---|---|---|---|
| `planner` | `direct_plan` | 1 | 15,000ms | 3 | No |
| `researcher` | -- | 2 | 30,000ms | 5 | No |
| `verifier` | -- | -- | 10,000ms | -- | **Yes** |
| `synthesizer` | -- | 1 | 60,000ms | -- | No |

The verifier is the only capability that can be silently skipped on failure. This is intentional: a missing verification produces a CAUTION-level result rather than a hard failure.


## 5. Degradation Policy

**File**: `orchestration/strategy.py`

The degradation system operates at two levels:

### Strategy-Level Fallback

```
Legal fails
  |
  +-- Walk fallback_chain: ["toolloop"]
  |     |
  |     +-- toolloop succeeds --> done
  |     |
  |     +-- toolloop fails --> on_total_failure
  |
  +-- on_total_failure: "toolloop" (already tried)
  |
  +-- ALL_STRATEGIES_FAILED error event
```

The orchestrator tracks `tried` strategies in a set to prevent infinite loops. Each fallback transition emits a `PhaseEvent(phase="fallback")`.

### Per-Capability Degradation

Within the Legal strategy, each capability (planner, researcher, verifier, synthesizer) has its own `CapabilityDegradation` config:

```python
@dataclass
class CapabilityDegradation:
    fallback: str | None     # Alternative capability
    max_retries: int         # Retry count
    timeout_ms: int          # Per-call timeout
    circuit_breaker_threshold: int  # Consecutive failures before opening
    skip_on_failure: bool    # Skip entirely vs hard fail
```


## 6. LEGIS Pipeline (PRVS)

**Entry point**: `agent/pipeline.py:legis_agent_pipeline()`

The LEGIS pipeline implements a 5-phase legal research pipeline:

```
+------------------+     +------------------+     +------------------+
|   Phase 0        |     |   Phase P        |     |   Phase R        |
|   INTENT         |---->|   PLANNING       |---->|   RESEARCH       |
|   DETECTION      |     |                  |     |                  |
|                  |     |  DIRECT: determ.  |     |  P1: parallel    |
|  Level + Scenario|     |  SIMPLE: fast    |     |  P2: sequential  |
|  Depth           |     |  STD+: reasoning |     |  Gap remediation |
+------------------+     +------------------+     +--------+---------+
                                                           |
                                                           v
+------------------+     +------------------+     +--------+---------+
|   Phase S        |     |   Phase V        |     |   Evidence Pack  |
|   SYNTHESIS      |<----|   VERIFICATION   |<----|   (EvidencePack)  |
|                  |     |                  |     +------------------+
|  Scenario prompt |     |  5-step check    |
|  7 interventions |     |  Self-correction |
|  Streaming SSE   |     |  15s budget      |
+------------------+     +------------------+

Output: Streaming SSE tokens + confidence score + citations
```

### Level-Based Branching

```
Intent Level      Planning                Research       Verify      Synthesize
-----------       --------                --------       ------      ----------
DIRECT            build_direct_plan()     R              --          S(minimal)
SIMPLE            run_planner(fast)       R              --          S(concise)
STANDARD          run_planner(fast)       R + graph      V           S(full)
COMPLEX           run_planner(reasoning)  R + graph      V + LLM     S(full)
```


## 7. Intent Detection (Phase 0)

**File**: `agent/intent_detector.py`

**Function**: `run_intent_detector(user_message, recent_messages, ...)`

**Target**: < 3 seconds latency.

### IntentResult Structure

```python
@dataclass
class IntentResult:
    level: PipelineLevel          # CHAT | DIRECT | SIMPLE | STANDARD | COMPLEX
    scenario: ScenarioType        # ricerca | parere | contenzioso | contratto_* | ...
    is_legal: bool                # Binary decision gate
    routing_confidence: float     # 0.0 - 1.0
    depth: DepthLevel             # MINIMAL | STANDARD | DEEP
    intent_summary: str
    legal_domain: str
    jurisdiction: str             # Default: "IT"
    references: list[ExtractedReference]
    key_concepts: list[str]
    max_sources: int              # From _LEVEL_CONFIG
    planner_model: str | None     # Model for planning phase
    skip_verifier: bool           # Whether to skip verification
    skip_graph: bool              # Whether to skip graph enrichment
    synthesizer_depth: str        # "none" | "minimal" | "concise" | "full"
```

### Two-Step Enforcement (ACL-1)

1. **Binary decision**: `is_legal` determines CHAT vs legal pipeline.
2. **Level consistency**: If `is_legal=false`, force `level=CHAT` regardless of LLM output. If `is_legal=true` but `level=CHAT`, force `is_legal=false` (resolve inconsistency).

### Level Configuration

| Level | Planner Model | Verifier | Graph | Synth Depth | Max Sources |
|---|---|---|---|---|---|
| CHAT | None | Skip | Skip | none | 0 |
| DIRECT | None | Skip | Skip | minimal | 3 |
| SIMPLE | lexe-fast | Skip | Skip | concise | 4 |
| STANDARD | lexe-fast | Active | Active | full | 6 |
| COMPLEX | settings.legis_planner_model | Active | Active | full | 8 |

### Fallback Behavior (ACL-2)

On any failure (timeout, LLM error, parse error), the intent detector returns a CHAT-level fallback. Rationale: false-chat is recoverable (user rephrases), false-legal produces poor output.

**Retry**: 1 retry on timeout (handles LiteLLM cold start). Max 2 attempts total.


## 8. Planning (Phase P)

**File**: `agent/planner.py`

### Three Planning Modes

| Intent Level | Mode | Model | Description |
|---|---|---|---|
| DIRECT | Deterministic | None | `build_direct_plan()`: constructs plan from extracted references without LLM |
| SIMPLE | Fast | `lexe-fast` | `run_planner()`: LLM plans with low latency |
| STANDARD+ | Reasoning | `lexe-fast` or settings override | `run_planner()`: LLM plans with `reasoning_effort=medium` |
| COMPLEX | Deep reasoning | settings.legis_planner_model | Full reasoning with `reasoning_effort` from catalog config |

### ResearchPlan Output

```python
@dataclass
class ResearchPlan:
    query_understood: str
    legal_domain: str
    temporal_scope: str | None
    jurisdiction: str
    key_norms: list[str]
    key_concepts: list[str]
    sources_to_query: list[SourceQuery]
    disambiguation_needed: str | None
    deepening_suggestions: list[DeepeningSuggestion] | None
```

### Tool Diversity Enforcement

`enforce_tool_diversity()` injects missing tool queries for STANDARD+ levels:

1. If `key_norms` present but no `normattiva_search` planned, inject normattiva queries.
2. If no `kb_search` at all, inject massimario search.
3. If CC/CP norms but no `infolex_search`, inject doctrine queries.

### Truncated JSON Repair

When `finish_reason=length`, the planner attempts bracket-aware JSON repair by tracking open brackets/braces (respecting string literals) and closing them in reverse order.


## 9. Research (Phase R)

**File**: `agent/researcher.py`

### Execution Model

**P1 (Parallel)**: All `SourceQuery` items with `priority=1` execute concurrently via `asyncio.gather()`.

**P2 (Sequential)**: Items with `priority=2` execute sequentially after P1 completes. Typically vigenza verification and cross-referencing that depends on P1 results.

### Per-Tool Call Limits

| Tool | Max Calls |
|---|---|
| `normattiva_search` | 8 |
| `eurlex_search` | 6 |
| `infolex_search` | 6 |
| `kb_search` | 8 |
| `kb_normativa_search` | 6 |
| `lex_search` | 6 |
| `lex_search_enriched` | 4 |
| `vigenza_fast` | 15 |
| `one_search` | 3 |
| `web_search` | 3 |

### Budget Enforcement

The `Researcher` respects both local per-tool limits and the global `BudgetController`. The effective cap is `min(max_total_calls, budget.remaining_tool_calls)`.

### Evidence Pack

All research results are accumulated into a single `EvidencePack` containing:
- `norms: list[NormRef]` -- legislative text with vigenza status
- `case_law: list[CaseLawRef]` -- judicial decisions
- `massime: list[MassimaRef]` -- case law summaries
- `coverage: dict[str, SourceCoverage]` -- per-source status tracking
- `conflicts: list[...]` -- cross-reference conflicts


## 10. Verification (Phase V)

**File**: `agent/verifier.py`

### 5-Step Verification Process

```
Step 1: Structural validation (regex, no LLM)
     |
     v
Step 2: Vigenza consistency check
     |
     v
Step 3: Cross-reference integrity
     |
     v
Step 4: LLM coherence check (conditional)
     |
     v
Step 5: Self-correction loop (DEEP only, max 2 attempts)
```

### LLM Coherence Triggers

The LLM coherence check (Step 4) is triggered when:
- 5+ norms in evidence pack
- Sensitive legal domains (penale, lavoro, privacy, tributario, fiscale)
- Historical date reference (non-vigente temporal scope)
- Cross-jurisdiction (IT + EU norms)
- Low vigenza confidence (< 0.70) on any norm

### Self-Correction Loop

**Function**: `verify_with_self_correction()`

**Total budget**: 15 seconds (hard timeout).

```
Attempt 1 (60% of budget = ~9s):
  run_verifier()
    |
    +-- PASS --> VerificationResult(confidence_level="VERIFIED")
    |
    +-- FAIL -->
         |
         +-- All issues are coverage_gap? --> downgrade to WARN, no retry
         |
         +-- Non-gap issues exist -->
              |
              Attempt 2 (remaining budget):
                retry_with_reflection() --> re-synthesize
                run_verifier() on corrected text
                  |
                  +-- PASS --> VerificationResult(confidence_level="VERIFIED", attempt=2)
                  +-- FAIL --> VerificationResult(confidence_level="CAUTION", disclaimer=...)
```

**Key detail**: `coverage_gap` issues from the LLM verifier are informational only. They do not trigger self-correction and do not produce hard penalties.


## 11. Synthesis (Phase S)

**File**: `agent/synthesizer.py`

### Scenario-Specific Prompts

| Scenario | Prompt Key | Synth Depth |
|---|---|---|
| ricerca | default | full |
| parere | SCENARIO_SYNTH_PARERE | full |
| contenzioso | SCENARIO_SYNTH_CONTENZIOSO | full |
| contratto_redazione | SCENARIO_SYNTH_CONTRATTO_REDAZIONE | full |
| contratto_revisione | SCENARIO_SYNTH_CONTRATTO_REVISIONE | full |
| recupero_crediti | SCENARIO_SYNTH_RECUPERO_CREDITI | full |
| nda_triage | SCENARIO_SYNTH_NDA_TRIAGE | full |
| compliance_check | SCENARIO_SYNTH_COMPLIANCE_CHECK | full |
| risk_assessment | SCENARIO_SYNTH_RISK_ASSESSMENT | full |

### 7 Prompt Interventions (A-G)

Defined in `agent/prompts.py`:

| ID | Intervention | Purpose |
|---|---|---|
| A | Source citation format | Enforce [N] numbered citation badges |
| B | Vigenza marking | Mark verified/unverified vigenza status |
| C | Confidence disclosure | Include confidence band in output |
| D | Authority weighting | Weight jurisprudence by court authority |
| E | Gap disclosure | "Limitazioni della ricerca" section |
| F | Research gaps injection | Auto-computed gaps from `gap_reporter` |
| G | Follow-up generation | User-perspective follow-up questions |

### Streaming Output

The synthesizer streams tokens via SSE. The caller wraps each chunk in a `token_event()`. The evidence pack is the Single Source of Truth (SSoT) -- the synthesizer never fabricates citations.


## 12. Confidence Scoring

**File**: `agent/scoring.py`

### 5 Weighted Components

| Component | Weight | Source |
|---|---|---|
| `source_reliability` | **0.30** | Primary (normattiva, eurlex) vs secondary sources ratio |
| `vigenza_match` | **0.25** | Vigenza verification status per norm |
| `link_verification` | **0.20** | URL tier quality (tier 1=1.0, tier 2=0.7, tier 3=0.4) |
| `cross_source` | **0.15** | Source diversity + authority-weighted jurisprudence |
| `text_quality` | **0.10** | Text excerpt availability (> 50 chars) |

### Formula

```
raw_score = source_reliability * 0.30
          + vigenza_match     * 0.25
          + link_verification * 0.20
          + cross_source      * 0.15
          + text_quality      * 0.10

overall = round(raw_score * 100)
```

### Hard Penalties

| Condition | Penalty |
|---|---|
| Any primary citation failed deterministic existence check | Cap at 60 |
| Vigenza was skipped (early exit) | Cap at 55 |
| No evidence found (zero norms + zero case_law + zero massime) | Force 0 |

### Floor

When evidence exists (`total_items > 0`) but score < 40, the floor is applied: `overall = 40`. This prevents misleadingly low scores when valid evidence was collected.

### Jurisprudence Absence

Absence of jurisprudence does NOT reduce confidence. Jurisprudence is informational enrichment, not a verification gate. (BK-006 Fix C)


## 13. 3-Band Thresholds

| Band | Range | Label | Frontend Badge |
|---|---|---|---|
| **VERIFIED** | >= 45 | Verified | Green |
| **CAUTION** | 25 - 44 | Caution | Yellow |
| **LOW_CONFIDENCE** | < 25 | Low Confidence | Red |


## 14. Authority Classification

**File**: `agent/scoring.py`

### Classification Table

| Authority Level | Weight | Detection Pattern |
|---|---|---|
| `sezioni_unite` | **1.0** | "unite" in sezione, "su" sezione, "sezioni unite" in authority |
| `cassazione` | **0.8** | Numeric sezione (1-6, I-VI), "cassazione" in authority |
| `corte_appello` | **0.5** | "appello" or "corte d'appello" in authority |
| `tribunale` | **0.3** | "tribunale" in authority |
| `isolato` | **0.2** | Default fallback |

### Cross-Source Scoring with Authority

The `_score_cross_source()` function blends source diversity (70%) with authority quality (30%):

```
blended = diversity_score * 0.7 + avg_authority * 0.3
```

**Conformity bonus**: When 3+ massime from authoritative sources (weight >= 0.5) agree, a `CONFORMITY_BONUS = 0.15` is applied:

```
if authoritative_count >= 3:
    blended = min(1.0, blended + 0.15)
```


## 15. Depth Budgets

**File**: `agent/depth_budget.py`

### Budget Allocation

| Parameter | MINIMAL | STANDARD | DEEP |
|---|---|---|---|
| `total_tool_calls` | 8 | 25 | 40 |
| `wave1_budget` | 8 | 14 | 20 |
| `wave2_budget` | 0 | 4 | 10 |
| `wave3_budget` | 0 | 7 (+5 gap) | 10 (+5 gap) |
| `timeout_s` | 20 | 40 | 55 |
| `planner_model` | "none" (deterministic) | "lexe-fast" | "lexe-frontier" |
| `verifier_enabled` | No | Yes | Yes |
| `audit_layers` | 0 | 1 | 2 |
| `self_correct` | No | No | Yes |

### Depth Resolution

```
IntentResult.depth (explicit)
     |
     +-- Present? --> use directly
     |
     +-- Absent?  --> PipelineLevel fallback:
                        CHAT/DIRECT/SIMPLE --> MINIMAL
                        STANDARD           --> STANDARD
                        COMPLEX            --> DEEP
```


## 16. Multi-Agent Research

**Feature flag**: `ff_multi_agent_research`

**File**: `agent/intent_decomposer.py`

### WorkPlan Decomposition

For SIMPLE+ queries (when FF enabled), `run_intent_decomposer()` produces a `WorkPlan` with sub-tasks grouped into parallel waves:

```python
@dataclass
class SubTask:
    id: str              # "st-001"
    aspect: str          # normativa | giurisprudenza | dottrina | vigenza | eu | serendipity
    question: str        # Atomic research question
    tools: list[str]     # Tools to use
    priority: int        # Wave: 1=parallel, 2=sequential, 3=remediation
    depends_on: list[str]  # SubTask IDs that must complete first
    budget: int          # Max tool calls for this task
```

### 3-Wave Parallel Execution

```
Wave 1 (parallel)                   Wave 2 (sequential)         Wave 3 (remediation)
+-------------------+               +------------------+        +------------------+
| Norm Agent        |               | Vigenza Agent    |        | Gap Analyst      |
| (normattiva,      |--+            | (normattiva re-  |        | (auto-fetch      |
|  kb_normativa)    |  |            |  verification)   |        |  missing norms)  |
+-------------------+  |            +------------------+        +------------------+
                       |
+-------------------+  |  gather()  +------------------+
| Juris Agent       |--+----------->| Cross-ref check  |
| (kb_search,       |  |            +------------------+
|  lex_search)      |  |
+-------------------+  |
                       |
+-------------------+  |
| Doctrine Agent    |--+
| (infolex_search)  |
+-------------------+
         |
         | (COMPLEX only)
+-------------------+
| Serendipity Agent |
| (cross-domain,    |
|  budget=2)        |
+-------------------+
```

### Budget Configuration (Multi-Agent)

| Depth | Total | Wave 1 | Wave 2 | Wave 3 | Timeout |
|---|---|---|---|---|---|
| MINIMAL | 8 | 8 | 0 | 0 | 20s |
| STANDARD | 20 | 14 | 4 | 2 | 35s |
| DEEP | 35 | 20 | 10 | 5 | 50s |

### 5 Research Agents + Serendipity

| Agent | File | Aspect | Tools | Wave |
|---|---|---|---|---|
| `NormAgentV2` | `research/norm_agent_v2.py` | normativa | normattiva_search, kb_normativa_search | 1 |
| `JurisAgent` | `research/juris_agent.py` | giurisprudenza | kb_search, lex_search | 1 |
| `DoctrineAgentV2` | `research/doctrine_agent_v2.py` | dottrina | infolex_search | 1 |
| `VigenzaAgent` | `research/vigenza_agent.py` | vigenza | normattiva_search | 2 |
| `GapAnalyst` | `research/gap_analyst.py` | remediation | kb_normativa_search, normattiva_search | 3 |
| `SerendipityAgent` | `research/serendipity_agent.py` | serendipity | kb_search | 1 (COMPLEX only) |

### Agent Base Protocol

All agents extend `ResearchAgent` (in `research/base.py`):

```python
class ResearchAgent(ABC):
    MAX_ITERATIONS = 2
    MIN_QUALITY = 0.5

    async def run(question, budget, context) -> AgentResult:
        for iteration in range(MAX_ITERATIONS):
            partial = await _search(question, iteration, accumulated, context)
            _merge(accumulated, partial)
            quality = _evaluate(accumulated)
            if quality >= MIN_QUALITY: break
            question = _refine_query(original, accumulated)
        return accumulated
```

Each agent iterates up to `MAX_ITERATIONS`, evaluating quality after each pass. If quality exceeds `MIN_QUALITY`, the agent stops early. Otherwise, it refines the query and retries.


## 17. Gap Remediation

**File**: `agent/gap_reporter.py`

### `compute_research_gaps(evidence, plan)`

Identifies gaps between the research plan and collected evidence:

1. **Failed or partial tool coverage**: Sources that returned errors or zero results.
2. **No norms found**: Missing legislative evidence.
3. **No jurisprudence**: Missing case law and massime.
4. **Unverified vigenza**: Norms without API-verified vigenza status.
5. **Norms cited in massime but missing**: Regex-extracted citations (`art. N codice`) from massime text that are not present as autonomous evidence items.

### `extract_missing_norms(evidence)`

Returns a structured list of `MissingNorm` objects (max 5, sorted by citation frequency):

```python
@dataclass
class MissingNorm:
    article: str     # "2043"
    code: str        # "codice civile"
    tool: str        # "kb_normativa_search" or "normattiva_search"
    arguments: dict  # Ready-to-use tool call arguments
```

For the 5 main codes (CC, CP, CPC, CPP, Costituzione), uses `kb_normativa_search` (fast, ~150ms). For other acts, falls back to `normattiva_search`.

### `format_gaps_for_prompt(gaps)`

Formats gaps as Italian text for injection into the synthesizer prompt (Intervention F):

```
Lacune rilevate automaticamente:
- Normativa: fonte temporaneamente non disponibile
- Vigenza non verificata via API per: art. 2043 codice civile
```


## 18. Evidence Fusion

**File**: `agent/fusion.py`

### `fuse_evidence(evidence_pack)`

Post-processes the evidence pack after multi-agent research:

1. **Dedup norms by dedup_key**: URN or `{act_type}:{article}` composite key. Keeps the instance with highest confidence.
2. **Dedup massime by RV number**: Keeps the instance with highest confidence. Massime without RV are retained.
3. **Sort norms by confidence** (descending).
4. **Confidence adjustments**:
   - `+5` if 3+ sources contributed results (multi-source confirmation)
   - `+5` if cross-reference CONFIRMS relations exist
   - `-10` if zero norms found (critical gap)


## 19. Tool Runtime Hardening (WS1)

**File**: `orchestration/tool_runtime.py`

### WS1 Configuration

```python
@dataclass(frozen=True)
class ToolLoopConfig:
    max_calls_per_iteration: int = 8      # WS1C: per-iteration cap
    max_calls_total: int = 25             # WS1C: total cap
    max_args_size: int = 4096             # WS1D: max argument string size
    max_json_depth: int = 5               # WS1D: max JSON nesting depth
    max_blocked_iterations: int = 2       # Max iterations with all tools blocked
    max_iterations: int = 5               # Max loop iterations
    max_wall_clock_s: float = 120.0       # Wall-clock timeout
```

### Hardening Controls

| Control | Code | Description |
|---|---|---|
| **Hard clamp** | WS1A | If `force_text_only=true`, tool_calls are dropped even if LLM returns them |
| **Circuit breaker** | WS1B | 2+ consecutive identical `tool_name:arguments` calls force text-only mode |
| **Per-iteration cap** | WS1C | Truncate to `max_calls_per_iteration` (8) calls per LLM response |
| **Total cap** | WS1C | After `max_calls_total` (25) calls, switch to text-only |
| **Argument validation** | WS1D | Reject calls with args > 4096 bytes or JSON depth > 5 |
| **Allowlist enforcement** | -- | Tool must be in `allowed_tool_names` set (from policy) |
| **Wall-clock timeout** | -- | Hard stop after `max_wall_clock_s` (120s) |

### Recovery Behavior

When all tools in an iteration are blocked (invalid args, not in allowlist), the runtime:
1. Rolls back messages to before the tool calls.
2. Injects a user message: "I tool richiesti non sono disponibili. Rispondi con le informazioni che hai, senza usare tool."
3. Switches to `force_text_only` mode.

When the circuit breaker triggers, the same rollback and text-only fallback occurs.


## 20. Key Files

### Orchestration Layer (1,614 LOC)

| File | LOC | Purpose |
|---|---|---|
| `orchestration/orchestrator.py` | 125 | Main orchestration loop with fallback |
| `orchestration/contracts.py` | 245 | TurnContext, Protocol interfaces, DTOs |
| `orchestration/decomposer.py` | 66 | Binary routing: CHAT vs Legal |
| `orchestration/events.py` | 155 | 13 DomainEvent types |
| `orchestration/strategy.py` | 84 | Abstract strategy + DegradationPolicy |
| `orchestration/tool_runtime.py` | 397 | Shared tool-loop with WS1 hardening |
| `orchestration/strategies/toolloop.py` | 124 | Terminal tool-calling strategy |
| `orchestration/strategies/legal.py` | 79 | Unified legal strategy |

### Agent Pipeline (6,954 LOC)

| File | LOC | Purpose |
|---|---|---|
| `agent/pipeline.py` | -- | LEGIS orchestrator (PRVS flow) |
| `agent/intent_detector.py` | 426 | Phase 0: Intent Detection |
| `agent/planner.py` | 428 | Phase P: Planning |
| `agent/researcher.py` | -- | Phase R: Research execution |
| `agent/verifier.py` | 594 | Phase V: Verification + self-correction |
| `agent/synthesizer.py` | -- | Phase S: Streaming synthesis |
| `agent/scoring.py` | 307 | Confidence scoring + authority classification |
| `agent/unified_pipeline.py` | 168 | Depth-based pipeline wrapper |
| `agent/depth_budget.py` | 65 | Depth budget configuration |
| `agent/gap_reporter.py` | 211 | Gap detection + missing norm extraction |
| `agent/intent_decomposer.py` | 407 | WorkPlan decomposition (multi-agent) |
| `agent/fusion.py` | 76 | Evidence dedup + fusion |
| `agent/legis_v2.py` | 88 | DomainEvent bridge for LEGIS |
| `agent/models.py` | -- | Pydantic models for all pipeline DTOs |
| `agent/prompts.py` | -- | All prompt templates (PRVS + scenarios) |
| `agent/scenarios.py` | -- | Scenario-specific plan builders |

### Research Agents (684 LOC)

| File | LOC | Purpose |
|---|---|---|
| `agent/research/base.py` | 122 | ResearchAgent protocol + AgentResult |
| `agent/research/norm_agent_v2.py` | 146 | Normativa research |
| `agent/research/juris_agent.py` | 144 | Jurisprudence research |
| `agent/research/doctrine_agent_v2.py` | -- | Doctrine research |
| `agent/research/vigenza_agent.py` | 71 | Vigenza verification |
| `agent/research/gap_analyst.py` | -- | Gap remediation |
| `agent/research/serendipity_agent.py` | 97 | Cross-domain discovery |


---

*Last updated: 2026-03-19. Review quarterly or after any architectural change to the orchestration or pipeline layers.*
