# ADR-D2: Differentiated Semaphores for Tool Rate Limiting

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-02-21 |
| **Deciders** | LEXE Core Team |
| **Scope** | lexe-core / tools / executor |

## Context

The LEGIS pipeline executes multiple tools in parallel during the Research phase (normattiva_search, infolex_search, kb_search, eurlex_search, lex_search, etc.). Without rate limiting, this leads to:

- **Upstream overload**: Normattiva and EUR-Lex APIs have strict rate limits
- **Resource exhaustion**: Too many concurrent HTTP connections per request
- **Cascading failures**: One tool timeout causing others to fail
- **Unfair scheduling**: Web search tools (slow, expensive) competing with fast KB lookups

## Decision

Implement **differentiated asyncio semaphores** by tool category:

| Category | Semaphore Limit | Tools | Rationale |
|----------|----------------|-------|-----------|
| **official** | 6 | normattiva_search, eurlex_search | Government APIs: reliable but rate-limited |
| **certified** | 4 | infolex_search, kb_search, kb_normativa_search | Internal KB + Brocardi: fast, CPU-bound |
| **web** | 3 | lex_search, lex_search_enriched | External web search: slow, expensive, rate-limited |

Each semaphore is per-request (not global), so concurrent user requests each get their own set of semaphores.

## Consequences

### Positive
- **Upstream protection**: Government APIs never see more than 6 concurrent requests per user request
- **Fair scheduling**: Fast KB tools are not blocked by slow web searches
- **Predictable behavior**: Maximum concurrent connections are bounded and documented
- **Graceful degradation**: If web search semaphore is full, other tools continue executing

### Negative
- **Configuration complexity**: Three semaphore values to tune instead of one
- **Potential underutilization**: If only web tools are needed, the official/certified capacity is wasted

## Implementation

```python
# In tool executor
_TOOL_CATEGORIES = {
    "normattiva_search": "official",
    "eurlex_search": "official",
    "infolex_search": "certified",
    "kb_search": "certified",
    "kb_normativa_search": "certified",
    "lex_search": "web",
    "lex_search_enriched": "web",
}

_SEMAPHORE_LIMITS = {
    "official": 6,
    "certified": 4,
    "web": 3,
}
```

## Alternatives Considered

### Single global semaphore (rejected)
- Pro: Simple to implement
- Con: A burst of slow web searches blocks fast KB lookups

### Per-tool semaphore (rejected)
- Pro: Maximum granularity
- Con: Too many semaphores (7+), hard to reason about combined behavior

### Token bucket rate limiter (deferred)
- Pro: Smoother rate limiting over time windows
- Con: More complex, unnecessary for current scale (< 50 concurrent users)

## Related
- `lexe-core/src/lexe_core/tools/executor.py`
- `lexe-core/src/lexe_core/tools/policy.py`
