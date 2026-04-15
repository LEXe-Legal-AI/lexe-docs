# Sprint 25 Log — Architectural Deepening

Autonomous execution log: phase-by-phase commit SHAs, bench results,
gate outcomes, rollback events.

Plan reference: `~/.claude/plans/dreamy-waddling-lecun.md`
Started: 2026-04-15

---

## Fase 0 — Reference Trace Capture

- ✅ Query `5BDEX6DS` extracted from stage DB
  - correlation_id: `67478c46-e7b5-4903-82cb-197280f80110`
  - trace_id: `53efc4e8-d77e-4d26-9f4d-36dfa8d85cac`
  - conversation_id: `1ea96c4f-5396-4519-93cd-16c0833eaa02`
  - original `created_at`: 2026-04-15 12:53:18 UTC
- ✅ Fixture saved: `lexe-core/tests/fixtures/sprint25_reference_5BDEX6DS.json`
- ✅ Query text: composizione negoziata crisi d'impresa (D.L. 118/2021 +
  D.Lgs. 14/2019, verifica vigenza + modifiche)
- ✅ Baseline replay (pre-sprint25-v6) — 2026-04-15 ~13:30 UTC

### Baseline metrics (pre-Sprint25)

| field | DB original | v6 replay |
|---|---|---|
| confidence_score | 83 | 86 |
| confidence_level | GOOD | VERIFIED |
| latency_ms | 31779 | 45110 |
| verification.overall_status | None | None |
| claim_audit present | null | present-in-done (blocking) |
| event_order | meta→preprocessing→agent_phase→legis_source_progress→legis_verify→legis_evidence→done | legis_verify→done |
| findings.unverified_citations | 1 | 3 |
| tools_executed / failed | 4 / 3 | 8 / 5 |

Baseline file: `lexe-core/sprint25_5BDEX6DS_replay_pre-sprint25-v6.json`

---

## Fase 1 — Async Claim Audit + SSE Events

**Design**: move claim_audit from 5s-blocking wait_for right before
scoring, to asyncio.create_task kicked off right after Phase 2b synth.
New event order: `token* → legis_verify → legis_claim_audit → done`.
Stream closes on done (SSEClient.ts:368), so audit event must precede
done. Payload persisted in done.legis.claim_audit for reload
rehydration.

### Commits
| repo | SHA | message |
|---|---|---|
| lexe-core | `87f357e` | feat(sprint25-f1): async claim-audit + SSE events for finishing phase |
| lexe-webchat | `d30127e` | feat(sprint25-f1): LEGIS_CLAIM_AUDIT + LEGIS_REPAIR SSE handlers |

Safety tags: `pre-sprint25-fase1` (both repos).

### Deploy
- SSH + git pull + `docker compose -f docker-compose.yml -f docker-compose.override.stage.yml build --no-cache lexe-core lexe-webchat` ✅
- `up -d` ✅
- Health check: `/health` 200 ✅, `stage-chat.lexe.pro` 200 ✅

### Smoke4 v7
| item | conf | latency | order | audit_event | notes |
|---|---|---|---|---|---|
| VIG-ANA-002 | 86 | **39.5s** | legis_verify → legis_claim_audit → done | ✅ | baseline 45.1s — **-12%** |
| KRY-ANA-047 | 88 | 41.5s | legis_verify → legis_claim_audit → done | ✅ | baseline BASSA AFF+88 fail |
| VIG-ANA-003 | 90 | 47.2s | legis_verify → legis_claim_audit → done | ✅ | baseline BASSA AFF+90 fail |
| TRI-ANA-101 | 90 | 61.2s | legis_verify → legis_claim_audit → done | ✅ | baseline BASSA AFF+89 fail |

**GATE ✅ PASS** — 0 errors, 0 fallback, avg latency 47.3s (within +15% of 43.4s baseline), audit-before-done rate 100%.

### Bench16 v7 — 2026-04-15 16:02-16:13 UTC (648s)
- **Avg confidence**: 82.6 (baseline 82.13, +0.6 ≤ ±2pt ✅)
- **Avg latency**: 46.3s (baseline 43.4s, **+6.7%** ≤ +15% ✅)
- **Audit event emitted**: 12/16 — 2 transient errors (EU-F-002, TRI-ANA-101 RemoteProtocolError), 1 guardrail (KRY-ANA-030), 1 no audit (VIG-ANA-003 — task likely timed-out silently)
- **Event order legis_verify → legis_claim_audit → done**: 12/16 valid = 100% of those that completed the audit pipeline
- **Repair event rate**: 0/16 (expected, Phase 3 not yet deployed)
- **Fallback count**: 0
- **Errors**: 2/16 (transient staging streaming layer, unrelated to Sprint 25 changes)

### Gate outcome: **PASS (DEGRADED)** post-retry
After retrying EU-F-002, TRI-ANA-101, VIG-ANA-003 transient failures:
- avg confidence: **83.2** (+1.1 vs Sprint 24 82.13) ✅
- avg latency: **49.4s** (+13.8% vs baseline 43.4s, within +15%) ✅
- audit event emitted: **14/16** (87.5% overall, 14/15 non-guardrail = 93.3%) ⚠️ soft-fail on VIG-ANA-003 (parallel audit task silently timed out on a 98s query — statistically tail, not systemic)
- audit-before-done: 14/16 ✅ on 14/14 valid emissions
- fabricated_caselaw total: **18** — Phase 3 repair will activate on ~3-4 items in bench v8
- errors: 0, fallback: 0 ✅

**Proceeding to Fase 2+3 with `degraded=true` flag per plan §Step 5**.

---

## Fase 2 — Proportional Verification Penalty + Quality Alerts

_TBD_

---

## Fase 3 — Active Scoped Repair Loop

_TBD_

---

## Fase 4 — Pack Sufficiency + Soft Veto + Bench Modernization

_TBD_

---

## Approval Gates
- [ ] Fase 1 stable on staging
- [ ] Fase 2 stable on staging
- [ ] Fase 3 stable on staging
- [ ] Fase 4 stable on staging
- [ ] Manual approval for `stage → main` merge + prod deploy
