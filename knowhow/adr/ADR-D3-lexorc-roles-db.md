# ADR-D3: LEXORC Roles Managed via Database

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | 2026-02-21 |
| **Deciders** | LEXE Core Team |
| **Scope** | lexe-core / agent / LEXORC orchestrator |

## Context

The LEXORC (LEXe ORChestrator) advanced pipeline uses multiple specialized agents:
- **NormAgent**: Searches normative sources (Normattiva, EUR-Lex)
- **CaseLawAgent**: Searches jurisprudence (KB massimario, InfoLex)
- **DoctrineAgent**: Searches doctrine and commentary
- **LexorcAuditor**: Verifies and audits evidence quality
- **LexorcSynthesizer**: Produces the final structured response

Each agent needs a specific LLM model configuration (model name, temperature, max_tokens, system prompt version). Initially, these were hardcoded in Python configuration files.

## Decision

Store LEXORC agent role configurations in the database via a `model_role_defaults` table (or equivalent settings mechanism), allowing:

1. **Runtime configuration**: Change agent models without redeployment
2. **Per-tenant customization**: Different tenants can have different agent configurations
3. **A/B testing**: Route a percentage of requests to alternative model configurations
4. **Audit trail**: Track which model configuration was used for each session

Configuration structure per role:
```json
{
  "role": "lexorc_synthesizer",
  "model": "google/gemini-2.5-flash",
  "temperature": 0.3,
  "max_tokens": 4096,
  "system_prompt_version": "v1",
  "enabled": true
}
```

## Consequences

### Positive
- **No-downtime model changes**: Switch from GPT-4 to Gemini for a specific agent without deploy
- **Experimentation**: Test new models on specific agents while keeping others stable
- **Tenant isolation**: Legal firms with different needs can have tailored configurations
- **Observability**: Every LEXORC session records which model/config was used

### Negative
- **Database dependency**: Agent configuration requires DB access at startup
- **Cache invalidation**: Config changes require cache refresh (mitigated by TTL)
- **Complexity**: More moving parts than simple Python config

## Fallback Strategy

If database configuration is unavailable:
1. Use `settings.lexorc_*_model` from environment variables
2. If env vars are also missing, use hardcoded defaults in code

This ensures the system degrades gracefully without DB access.

## Related
- `lexe-core/src/lexe_core/agent/advanced_orchestrator.py`
- `lexe-core/src/lexe_core/agent/agents/lexorc_synthesizer.py`
- `lexe-core/src/lexe_core/agent/agents/auditor.py`
- `lexe-core/src/lexe_core/agent/config_utils.py`
- DB migration: `lexe-core/migrations/017_rename_manus_to_lexorc.sql`
