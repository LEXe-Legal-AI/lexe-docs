# Orchestration V2 — Test Report (Staging Deploy 2026-03-12)

## Environment
- **Server**: staging (91.99.229.111)
- **Branch**: `stage`
- **Commits**: `085f1d1` (Phase 2d) → `d5b73f0` (3 bridge hotfixes)
- **FF**: `LEXE_FF_ORCHESTRATION_V2_TENANTS=d35385b4-695a-423d-b079-f470687349b6`
- **FF global**: `ff_orchestration_v2=False` (default)

## Test Tenant
| Campo | Valore |
|-------|--------|
| Tenant ID | `d35385b4-695a-423d-b079-f470687349b6` |
| Slug | `test-orch-v2` |
| Logto Org | `xadtn9478j4s` |
| Logto User | `testorchv2` / `TestOrchV2x2026` (id: `a61exulk29qt`) |
| LiteLLM Team | `tenant-d35385b4-695a-423d-b079-f470687349b6` |
| LiteLLM Key | `sk-L7-HXqzayV_6ZnC5lmnYRw` |
| Logto App (curl) | `wpr7krjrebqg4en3i0h1i` (Native, PKCE) |

---

## Test 1 — Container Health
**Method**: `docker ps` + `curl /health`
**Result**: PASS
```
lexe-core    Up (healthy)
{"status":"healthy","service":"lexe-core"}
```

## Test 2 — Feature Flag Config (in-container Python)
**Method**: Python script inside lexe-core container
**Assertions**:
- `is_orchestration_v2_enabled_for_tenant(d35385b4-...)` → `True`
- `is_orchestration_v2_enabled_for_tenant(81d9b68c-...)` (prova2) → `False`
- `ff_orchestration_v2` (global) → `False`
- `ff_orchestration_v2_tenants` → `d35385b4-695a-423d-b079-f470687349b6`

**Result**: PASS

## Test 3 — Module Imports (in-container Python)
**Method**: Import all 11 v2 modules inside container
**Modules verified**:
1. `orchestration.contracts` (TurnContext, IntentResult, ModelRoleSet, TraceContext)
2. `orchestration.events` (TokenEvent, ErrorEvent, RawSSEEvent)
3. `orchestration.orchestrator` (Orchestrator)
4. `orchestration.strategies.toolloop` (ToolloopStrategy)
5. `orchestration.strategies.legis` (LegisStrategy)
6. `orchestration.strategies.super_tool` (SuperToolStrategy)
7. `orchestration.strategies.lexorc` (LexorcStrategy)
8. `gateway.adapters` (BudgetHandleAdapter, HttpCapabilityClient)
9. `gateway.sse_adapter` (to_sse)

**Result**: PASS

## Test 4 — Dataclass Instantiation (in-container Python)
**Method**: Create instances of all core dataclasses
**Verified**:
- `TraceContext(correlation_id, trace_id)` → OK
- `ModelRoleSet(fast, primary, complex)` → OK (also has `embedding`, `configs`)
- `IntentResult(level, scenario, confidence)` → OK
- `TokenEvent(text=...)` → OK
- `ErrorEvent(message=..., code=...)` → OK
- `RawSSEEvent(sse_string=...)` → OK

**Result**: PASS

## Test 5 — Strategy Instantiation (in-container Python)
**Method**: `cls(bridge_fn=None)` for all 4 strategies
**Result**: PASS — all 4 strategies instantiate without error

## Test 6 — SSE Adapter (in-container Python)
**Method**: `to_sse(event, correlation_id="test")` for TokenEvent, ErrorEvent, RawSSEEvent
**Result**: PASS
- TokenEvent → 45 chars SSE
- ErrorEvent → 111 chars SSE
- RawSSEEvent → 12 chars SSE (passthrough)

## Test 7 — Source Code Verification (in-container Python)
**Method**: `inspect.getsource(stream_to_orchestrator)` → check for v2 markers
**Assertions**:
- `"is_orchestration_v2_enabled_for_tenant"` in source → True
- `"[ORCH_V2]"` in source → True

**Result**: PASS

## Test 8 — E2E CHAT via curl (org-scoped JWT)
**Method**: Full OIDC PKCE flow → org-scoped JWT → POST `/api/v1/gateway/customer/stream`
**Message**: `"Ciao, come stai?"`
**Token flow**: Native App → PKCE auth → experience API → password verify → identification → submit → interaction consent (org) → auth code → refresh token → org-scoped JWT

### SSE Response
```
data: {"type": "conversation_id", "id": "3a6ccfd4-..."}
data: {"type": "meta", ..., "pipeline": "auto", "conversation_mode": "studio"}
data: {"type": "preprocessing", "step": "resolving_mode", "detail": "Modalità STUDIO"}
data: {"type": "token", "content": "Ciao! Sto bene e sono pronto..."}
data: {"type": "token", "content": "...supporto strategico..."}
data: {"type": "done", ..., "pipeline": "toolloop", "trace_hash": "M6SES9W9"}
```

### Server Logs (v2 path)
```
[ORCH_V2] Orchestration v2 enabled for tenant=d35385b4-...
[orchestrator] executing strategy=toolloop (intended=toolloop)
[toolloop] executing via bridge
[orchestrator] strategy=toolloop completed in 9039ms
```

**Result**: PASS — Full v2 pipeline, no fallback to legacy

## Test 9 — Fallback Safety (pre-fix)
**Method**: Same E2E test before bridge hotfixes
**Observed behavior**: v2 pipeline fails (signature mismatches), automatically falls through to legacy v1 pipeline, user gets response normally.
**Log evidence**:
```
[ORCH_V2] ErrorEvent code=ALL_STRATEGIES_FAILED: ...
[ORCH_V2] No response from v2, falling through to legacy pipeline
[PRE_INTENT] CHAT conf=1.00 ... → toolloop
```

**Result**: PASS — Fallback mechanism works correctly, zero user impact on v2 failure

## Test 10 — Tenant Isolation
**Method**: Verified `prova2` tenant NOT affected when only `test-orch-v2` in allowlist
**Evidence**: `is_orchestration_v2_enabled_for_tenant(prova2_uuid)` → `False`

**Result**: PASS

---

## Bugs Found & Fixed During Deploy

| Bug | Commit | Root Cause |
|-----|--------|------------|
| `_apply_model_config() missing 'defaults'` | `233f2c7` | Bridge called with 2 args, signature needs 3 |
| `done_event() missing 'conversation_id'` | `a1ba85c` | 3rd positional arg required |
| `error_event()` wrong kwargs | `a1ba85c` | Used `code=`/`message=` instead of `error_code=`/`error_message=`, raw string instead of `ErrorCode` enum |
| `token_event()` wrong arity | `d5b73f0` | Called with `(cid, tid, content)` but signature is `(content)` only |

## Summary

| Category | Count | Status |
|----------|-------|--------|
| Config tests | 2 | PASS |
| Import tests | 1 | PASS |
| Dataclass tests | 1 | PASS |
| Strategy tests | 1 | PASS |
| SSE adapter tests | 1 | PASS |
| Source verification | 1 | PASS |
| E2E CHAT (v2 path) | 1 | PASS |
| Fallback safety | 1 | PASS |
| Tenant isolation | 1 | PASS |
| **Total** | **10** | **10/10 PASS** |
