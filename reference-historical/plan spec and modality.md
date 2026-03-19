# LEXE Hardening Execution Spec for Parallel Claude Code Teams

## Summary
Obiettivo: implementare S1 -> S5 senza reinterpretazioni architetturali, massimizzando il lavoro parallelo e minimizzando i conflitti sui file critici. La regola operativa è questa: un solo team possiede i file di integrazione cross-cutting, mentre gli altri team lavorano in autonomia su moduli, servizi e test con contratti congelati upfront. Nessun workstream può ampliare tool access, allargare payload user-facing o introdurre fallback silenziosi non tracciati.

## Shared Contracts
Questi contratti vanno congelati prima del coding. Sono la fonte di verità per tutti i team.

- `TurnEnvelope` interno contiene: `trace_id`, `tenant_id`, `contact_id`, `conversation_id`, `intended_pipeline`, `actual_pipeline`, `fallback_chain`, `degradations`, `model_calls`, `tool_calls`, `wall_clock_ms`, `memory_source`, `memory_chunks_retrieved`, `final_confidence_level`, `confidence_reason`.
- `done` SSE espone solo: `pipeline`, `fallback_count`, `degraded`, `confidence_level`, `memory_source`, `trace_id`.
- `degradations.reason` usa codici machine-readable, non stack trace. Codici iniziali: `intent_detector_exception`, `super_tool_fallback`, `lexorc_fallback`, `policy_fail_closed_default`, `memory_v1_fallback`, `self_correction_degraded`, `budget_exhausted`.
- `BudgetController` è per-turn, esplicito, passed-by-reference, con API minima: `check_budget()`, `record_model_call()`, `record_tool_call(dedup_key)`, `get_cached_result(dedup_key)`, `cache_result(dedup_key, result)`, `remaining_wall_clock`, `remaining_tool_calls`.
- Auth interna: `X-Forwarded-By=lexe-core`, `X-Gateway-Ts`, `X-Gateway-Sig`; firma HMAC-SHA256 su `tenant_id + timestamp + method + path`.
- `DEFAULT_SAFE_TOOL_SET` iniziale: `kb_search`, `normattiva_search`, `vigenza_fast`. Va validato a startup contro [definitions.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\tools\definitions.py).
- Config/env da introdurre o standardizzare: `LEXE_TURN_MAX_WALL_CLOCK_S`, `LEXE_TURN_MAX_MODEL_CALLS`, `LEXE_TURN_MAX_TOOL_CALLS`, `LEXE_LEGIS_MAX_WALL_CLOCK_S`, `LEXE_INTERNAL_HMAC_KEY`, `LEXE_INTERNAL_HMAC_TOLERANCE_S`, `LEXE_DEFAULT_SAFE_TOOLS`.
- Migrazione DB: usare `027_turn_envelopes.sql` in [migrations](C:\PROJECTS\lexe-genesis\lexe-core\migrations).

## Ownership Model
Per evitare conflitti, questi file sono **integration-owned** e modificabili solo dal Team Integrator: [customer_router.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\gateway\customer_router.py), [config.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\config.py), [docker-compose.override.stage.yml](C:\PROJECTS\lexe-genesis\lexe-infra\docker-compose.override.stage.yml), [docker-compose.override.prod.yml](C:\PROJECTS\lexe-genesis\lexe-infra\docker-compose.override.prod.yml).

Tutti gli altri team lavorano su moduli autonomi con test propri. Il Team Integrator fa solo wiring finale, config finale, e merge validation.

## Parallel Workstreams

### Team 0: Integrator and Contract Owner
- Congela i contratti sopra in un breve `HARDENING-CONTRACT.md` nel repo docs operativo o nella PR description di coordinamento.
- Introduce i nuovi env in [config.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\config.py) senza ancora cambiare comportamento.
- Fa il wiring finale in [customer_router.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\gateway\customer_router.py): creazione envelope, creazione budget, uso del safe tool set, iniezione signing memory client, emissione `done` summary, completion hook di persistenza.
- Aggiorna gli override infra stage/prod con env OTEL/HMAC e feature flags.
- Acceptance: build verde con tutti i moduli collegati, zero conflitti aperti sui file integration-owned.

### Team 1: Turn Envelope, Persistence, Metrics
- Crea [turn_envelope.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\gateway\turn_envelope.py).
- Crea [027_turn_envelopes.sql](C:\PROJECTS\lexe-genesis\lexe-core\migrations\027_turn_envelopes.sql).
- Implementa helper di persistenza envelope a fine turno, separato da [event_sink.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\gateway\event_sink.py); `event_sink` non deve diventare blocking.
- Aggiunge contatori/istogrammi Prometheus derivati dall’envelope.
- Estende [pipeline.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\agent\pipeline.py) per scrivere `final_confidence_level`, `confidence_reason`, e degradazioni di verifier/self-correction.
- Riuso Langfuse in [langfuse_integration.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\gateway\langfuse_integration.py) per trace start, fallback spans e final scores.
- Acceptance: unit test su envelope serialization, migration applicabile, persistence best-effort non blocca il turn, metrics emesse su fallback/degradation.

### Team 2: Budget Controller and Amplification Control
- Crea [budget.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\gateway\budget.py).
- Modifica [pipeline.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\agent\pipeline.py), [researcher.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\agent\researcher.py), [super_tool.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\agent\super_tool.py), [executor.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\tools\executor.py) per accettare un `budget` opzionale e rispettarlo.
- `executor.py` deve interrompere retry quando il budget è esaurito e deve usare dedup cache intra-turn.
- `researcher.py` deve trattare `budget.remaining_tool_calls` come hard cap globale.
- `super_tool.py` mantiene i limiti locali, ma il cap autoritativo è il budget globale.
- Acceptance: con budget basso, LEGIS e SUPER_TOOL si fermano entro il limite senza retry extra; duplicate tool calls nello stesso turno non chiamano il backend due volte.

### Team 3: Memory Integrity and Memory-Side Trust
- Modifica [memory_client.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\gateway\memory_client.py) per propagare sempre `tenant_id` nella fallback v1 e firmare tutte le chiamate interne verso memory.
- Modifica [routes.py](C:\PROJECTS\lexe-genesis\lexe-memory\src\lexe_memory\api\routes.py), [delta.py](C:\PROJECTS\lexe-genesis\lexe-memory\src\lexe_memory\api\delta.py), [service.py](C:\PROJECTS\lexe-genesis\lexe-memory\src\lexe_memory\service.py), [l3_semantic.py](C:\PROJECTS\lexe-genesis\lexe-memory\src\lexe_memory\layers\l3_semantic.py) per rendere esplicito il contratto `tenant_id`/`contact_id`.
- Estende [main.py](C:\PROJECTS\lexe-genesis\lexe-memory\src\lexe_memory\main.py) riusando il gateway middleware esistente per validare HMAC e timestamp, non per creare un secondo enforcement path.
- Wire OTEL in `lexe-memory` usando il pacchetto telemetry già presente; non introdurre un secondo stack parallelo.
- Non toccare il key schema L0/L1 in questo sprint.
- Acceptance: `lexe-memory` rifiuta richieste senza tenant o con firma invalida; v1 fallback include tenant; behavior L3 senza contact è esplicito e testato.

### Team 4: Policy Fail-Closed and Tools Boundary
- Modifica [policy.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\tools\policy.py) solo per distinguere chiaramente “persona assente” da “errore di risoluzione”.
- Modifica [executor.py](C:\PROJECTS\lexe-genesis\lexe-core\src\lexe_core\tools\executor.py) per firmare le chiamate a `lexe-tools-it`.
- In `lexe-tools-it`, estende [main.py](C:\PROJECTS\lexe-genesis\lexe-tools-it\src\lexe_tools_it\main.py) o middleware dedicato per validare HMAC; non creare un nuovo `telemetry.py`, perché esiste già.
- Verifica che `DEFAULT_SAFE_TOOL_SET` contenga solo tool realmente registrati e che il fallback su errore policy non usi mai toggles client-side.
- Acceptance: errore policy in core produce safe set o tools-disabled; request verso tools-it senza firma valida ricevono `403`; tool catalog validation fallisce a startup se safe set è invalido.

### Team 5: Orchestrator Guardrail and Runtime Drift Cleanup
- Modifica [routes.py](C:\PROJECTS\lexe-genesis\lexe-orchestrator\src\lexe_orchestrator\api\routes.py) per rimuovere riferimenti non sicuri a workflow mancanti o trasformarli in startup validation esplicita.
- Aggiunge guardrail di config per `ff_orchestrator_enabled` con errore/deprecation chiari.
- Documenta nello strato config/runtime che l’orchestrazione attiva è in core, non in ORCHIDEA.
- Acceptance: enablement non valido di ORCHIDEA fallisce in modo esplicito e osservabile; nessun import morto resta nel path attivo.

## Integration Sequence
1. Team 0 congela contratti e env names.
2. Team 1, 2, 3, 4, 5 lavorano in parallelo.
3. Team 0 integra nell’ordine: envelope wiring, budget wiring, memory signing/wiring, policy safe-set wiring, infra env wiring.
4. Merge order raccomandato: Team 1 -> Team 2 -> Team 3 -> Team 4 -> Team 5 -> Team 0 integration PR finale.
5. Nessun team diverso da Team 0 apre PR che modifica `customer_router.py` o `config.py`.

## Test Plan
- `S1 fallback coverage`: forzare errore intent detector, `SUPER_TOOL`, `LEXORC`, verifier degradation; atteso record in `core.turn_envelopes`, metrics incrementate, `done` summary sanificato.
- `S2 budget hard-stop`: con `LEXE_TURN_MAX_TOOL_CALLS=5`, query legale complessa deve fermarsi entro 5 tool calls aggregate; nessun retry dopo exhaustion.
- `S2 dedup`: due tool call identiche nello stesso turno devono riusare cache intra-turn e non colpire il backend due volte.
- `S3 tenant/contact integrity`: request memory senza `X-Tenant-ID` o con signature errata devono fallire; fallback v1 deve propagare tenant.
- `S3 L3 contract`: identity query senza `contact_id` deve essere bloccata; non-identity tenant-only query deve essere esplicita e loggata.
- `S4 policy safety`: errore DB/policy resolution deve produrre `DEFAULT_SAFE_TOOL_SET`; assenza persona non deve essere trattata come errore.
- `S4 tools-it trust`: chiamate interne non firmate verso tools-it devono ricevere `403`.
- `S5 orchestrator guardrail`: `ff_orchestrator_enabled=True` senza runtime valido deve produrre errore o disable esplicito.
- `observability`: in staging devono apparire trace distribuite `lexe-core -> lexe-memory` e `lexe-core -> lexe-tools-it`; Langfuse deve mostrare route/fallback/confidence spans.

## Delivery Criteria
- Ogni team consegna una PR isolata con test locali e una breve nota “public contract impact”.
- Team 0 consegna una PR finale di integrazione con smoke test end-to-end.
- Il rollout è pronto quando esistono: DB record queryabili, fallback/degradation visibili in telemetry, budget enforcement attivo, HMAC enforcement attivo, policy fail-open eliminato, orchestrator drift reso esplicito.

## Assumptions
- OneContext non era disponibile in questa sessione; la spec è basata su repo e doc correnti.
- `done` SSE resta compatibile lato client perché aggiunge solo campi sanificati.
- La write del turn record è best-effort e bounded; non deve bloccare la risposta utente.
- OTEL in `lexe-tools-it` esiste già; in `lexe-memory` va solo completato il wiring runtime.
- Nessun cambio di schema L0/L1 keying entra in questo sprint.
