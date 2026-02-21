# ADR-D6: Self-Correction Max 2 Retries with 15s Timeout

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-02-21 |
| **Deciders** | LEXE Core Team |
| **Scope** | lexe-core / agent / pipeline |

## Context

The LEGIS pipeline includes a self-correction mechanism: when the Verifier phase detects issues in the evidence pack (broken citations, vigenza mismatches, coherence issues), it can trigger a re-research cycle to fix the problems.

Without limits, this self-correction loop could:
- **Run indefinitely**: Some issues are unfixable (e.g., article genuinely does not exist)
- **Increase latency dramatically**: Each retry adds 5-10 seconds
- **Waste LLM tokens**: Repeated planner + researcher + verifier calls
- **Create user frustration**: Long wait times with no visible progress

## Decision

Implement a **maximum of 2 self-correction retries** with a **15-second timeout per retry**:

```
Attempt 1 (initial): Planner -> Researcher -> Verifier
  If verifier finds issues:
Attempt 2 (retry 1): Re-research specific items -> Verifier
  If still issues:
Attempt 3 (retry 2): Re-research specific items -> Verifier
  Regardless: Proceed to Synthesizer with best available evidence
```

### Timeout Behavior
- Each retry has a 15-second wall-clock timeout
- If the timeout fires, the retry is abandoned and the pipeline proceeds with the evidence gathered so far
- Total maximum self-correction time: 30 seconds (2 x 15s)
- Total maximum pipeline time (with retries): ~45 seconds for COMPLEX queries

### What Happens After Max Retries
- The Synthesizer receives the evidence pack as-is, including any unresolved issues
- Unresolved verifier issues are surfaced to the user via the confidence badge (CAUTION or LOW_CONFIDENCE)
- The response includes a transparency note: "Alcune fonti non sono state verificate completamente"

## Consequences

### Positive
- **Bounded latency**: Users never wait more than ~45s even for complex queries with verification issues
- **Progressive improvement**: Most issues are fixed on the first retry (observed: 80%+ fix rate on retry 1)
- **Transparency**: Unresolved issues are visible to the user via confidence badges
- **Resource protection**: LLM token usage is bounded

### Negative
- **Incomplete verification**: Some issues genuinely need 3+ retries to resolve (rare: <5% of queries)
- **Quality trade-off**: Proceeding with unverified evidence is a conscious quality compromise

## Metrics

Based on staging data:
- 70% of queries require 0 retries (verifier passes on first attempt)
- 25% require 1 retry (fixes most issues)
- 4% require 2 retries
- 1% would benefit from 3+ retries (accepted as LOW_CONFIDENCE)

## Related
- `lexe-core/src/lexe_core/agent/pipeline.py` (retry loop)
- `lexe-core/src/lexe_core/agent/verifier.py` (issue detection)
- `lexe-core/src/lexe_core/agent/researcher.py` (re-research)
- ADR-D5 (confidence spectrum for unresolved issues)
