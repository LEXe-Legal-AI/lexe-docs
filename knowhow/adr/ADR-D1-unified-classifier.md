# ADR-D1: Unified Intent Classifier

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-02-21 |
| **Deciders** | LEXE Core Team |
| **Scope** | lexe-core / agent pipeline |

## Context

The original LEXE pipeline used two separate LLM calls for query classification:

1. **LEXORC Complexity Classifier** (`classifier.py`): Classifies query complexity into FLASH/STANDARD/DEEP/STRATEGIC using a fast LLM call (~1.5s).
2. **LEGIS Intent Detector** (`intent_detector.py`): Classifies the query into PipelineLevel (DIRECT/SIMPLE/STANDARD/COMPLEX) + ScenarioType (ricerca/contratto/parere/etc.) with another LLM call (~1.5s).

This double-classifier pattern introduced:
- **~2.5-3s additional latency** before any tool execution begins
- **Redundant LLM inference**: both classifiers analyze the same query text
- **Inconsistency risk**: the two classifiers could disagree (e.g., LEXORC says FLASH but Intent Detector says COMPLEX)
- **Cost duplication**: two LLM calls per query, doubling classification cost

## Decision

Unify the LEXORC Complexity Classifier and the LEGIS Intent Detector into a **single LLM call** (`_pre_intent_route` in `customer_router.py`) that:

1. Determines the pipeline route: `toolloop`, `legis`, or `lexorc`
2. Classifies complexity level when routed to legis/lexorc
3. Extracts legal references and scenario type in the same JSON response

The unified classifier uses a single system prompt that combines the classification criteria from both original classifiers.

## Consequences

### Positive
- **Latency reduction**: From ~3s (2 sequential calls) to ~1.5s (1 call)
- **Consistency**: Single source of truth for routing decisions
- **Cost savings**: 50% reduction in classification LLM calls
- **Simpler debugging**: One classifier output to inspect instead of two

### Negative
- **Larger prompt**: The unified prompt is more complex (~50% larger)
- **Single point of failure**: If the classifier fails, no secondary classification available (mitigated by fallback to STANDARD/ricerca)
- **Testing complexity**: Must validate against both routing accuracy AND level/scenario accuracy simultaneously

### Risks
- The unified classifier may be slightly less accurate on edge cases where the two specialized classifiers would have disagreed productively
- Prompt engineering requires more careful iteration since changes affect both routing and classification

## Alternatives Considered

### Keep 2 LLM calls (rejected)
- Pro: Each classifier is focused and easier to maintain
- Con: 2.5s latency is unacceptable for user experience (target: <1s classification)

### Rule-based pre-filter + 1 LLM call (partially adopted)
- Short non-legal messages (<30 chars without legal hints) are routed to toolloop with 0ms latency via keyword matching in `_pre_intent_route`
- This handles ~30% of queries without any LLM call
- The remaining 70% go through the unified LLM classifier

## Related
- `lexe-core/src/lexe_core/gateway/customer_router.py` (`_pre_intent_route`)
- `lexe-core/src/lexe_core/agent/intent_detector.py`
- `lexe-core/src/lexe_core/agent/classifier.py`
