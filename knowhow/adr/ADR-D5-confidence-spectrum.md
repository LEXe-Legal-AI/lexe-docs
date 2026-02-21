# ADR-D5: Three-Level Confidence Spectrum

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-02-21 |
| **Deciders** | LEXE Core Team |
| **Scope** | lexe-core / agent / synthesizer + verifier |

## Context

Legal AI outputs require clear confidence signaling to prevent lawyers from relying on unverified information. The initial LEXE implementation used a numeric confidence score (0-100) which was:

- **Hard to interpret**: What does "72% confidence" mean for a legal opinion?
- **False precision**: Suggesting exactness that the system cannot guarantee
- **No actionable guidance**: The user does not know what to do differently at 65% vs. 75%

Legal practitioners need clear, actionable signals about when they can trust the output and when they need to verify independently.

## Decision

Replace the numeric confidence score with a **three-level confidence spectrum**:

| Level | Badge Color | Criteria | User Guidance |
|-------|-------------|----------|---------------|
| **VERIFIED** | Green | Norm text confirmed via official API (Normattiva/EUR-Lex), jurisprudence from certified sources (KB massimario), cross-source coherence check passed | "This information has been verified against official sources" |
| **CAUTION** | Yellow | Some sources verified but not all, minor inconsistencies detected, temporal scope unclear, or single-source only | "This information should be independently verified before use" |
| **LOW_CONFIDENCE** | Red | No official source confirmation, potential conflicts detected, verifier flagged issues, or fallback/timeout during research | "This information requires independent verification - treat as preliminary" |

## Rules

1. **VERIFIED** requires at least one official source (Normattiva or EUR-Lex) AND no verifier issues with severity "error"
2. **CAUTION** is the default when official sources are present but verifier found warnings
3. **LOW_CONFIDENCE** is assigned when: no official sources, verifier failed, or tools timed out
4. The confidence level applies to the **entire response**, not individual citations (individual citations have their own vigenza status)

## Consequences

### Positive
- **Actionable**: Lawyers know exactly what "CAUTION" means and what to do
- **Legal compliance**: Aligns with Italian Bar Association guidelines on AI-assisted research
- **Trust calibration**: Users learn to trust green badges and verify yellow/red ones
- **Simple UX**: Three colors are immediately understandable

### Negative
- **Information loss**: The granularity of 0-100 scores is reduced to 3 levels
- **Edge cases**: Borderline cases between CAUTION and LOW_CONFIDENCE require careful threshold tuning
- **Potential over-caution**: System may default to CAUTION too often, reducing user trust in the badge system

## Related
- `lexe-core/src/lexe_core/agent/scoring.py`
- `lexe-core/src/lexe_core/agent/verifier.py`
- `lexe-core/src/lexe_core/agent/synthesizer.py`
- `lexe-webchat/src/components/` (confidence badge rendering)
