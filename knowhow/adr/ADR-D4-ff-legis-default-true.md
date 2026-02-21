# ADR-D4: Feature Flag ff_legis_agent Enabled by Default

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-02-21 |
| **Deciders** | LEXE Core Team |
| **Scope** | lexe-core / gateway / feature flags |

## Context

The LEGIS Agent is LEXE's core legal research pipeline (Planner-Researcher-Verifier-Synthesizer). It was initially introduced behind a feature flag `ff_legis_agent` set to `False` to allow gradual rollout and testing.

After Sprint 8:
- LEGIS routing accuracy on the gold set exceeded 85%
- LEGIS pipeline latency (P95) was under 15 seconds for STANDARD queries
- All critical SSE events (meta, tool_call, tool_end, done) were verified complete
- Zero data loss incidents in staging over 2 weeks of testing

## Decision

Set `ff_legis_agent = True` as the **default value** in the application configuration.

This means:
1. All new deployments have LEGIS enabled automatically
2. The pre-intent classifier (`_pre_intent_route`) routes legal queries to LEGIS
3. Generic queries continue to use ToolLoop (no change)
4. The feature flag remains available for emergency rollback

## Consequences

### Positive
- **All users benefit**: Legal queries get structured research with citations, verification, and confidence scoring
- **No opt-in required**: Removes friction for new tenant onboarding
- **Consistent experience**: All LEXE instances behave the same way
- **Simpler configuration**: One less flag to set during deployment

### Negative
- **Higher resource usage**: LEGIS pipeline uses 3-8 tool calls per query vs. 0-1 for ToolLoop
- **Higher LLM cost**: LEGIS uses planner + synthesizer + verifier LLM calls
- **Latency increase for legal queries**: 5-15s vs. 2-5s for ToolLoop (but quality is much higher)

## Rollback Plan

If LEGIS causes issues in production:

1. **Immediate**: Set `LEXE_FF_LEGIS_AGENT=false` in environment and restart container
2. **Graceful**: All in-flight LEGIS requests will complete; new requests fall back to ToolLoop
3. **Zero downtime**: The flag is checked per-request, not at startup

## Related
- `lexe-core/src/lexe_core/config.py` (`ff_legis_agent`)
- `lexe-core/src/lexe_core/gateway/customer_router.py` (routing logic)
- `lexe-core/src/lexe_core/agent/pipeline.py` (LEGIS pipeline)
