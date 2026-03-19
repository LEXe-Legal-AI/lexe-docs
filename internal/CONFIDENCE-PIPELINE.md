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

# Confidence Pipeline -- Scoring, Verification, and Quality Reference

> Complete reference for the LEXE confidence scoring algorithm, verification
> pipeline, authority weighting, gap detection, and integration points.
> Based on B4 analysis findings and validated against `agent/scoring.py` source code.

---

## Scope

This document covers:

- The confidence scoring formula and all 5 weighted components.
- Component scoring functions: how each sub-score is computed.
- Hard penalties, caps, and floor logic.
- The 3-band confidence classification (VERIFIED / CAUTION / LOW_CONFIDENCE).
- Authority weights for Italian judicial sources.
- The verification pipeline: structural, vigenza, cross-reference, LLM coherence, self-correction.
- Gap detection and auto-remediation.
- Post-verification calibration rules.
- Integration points: Langfuse, Prometheus, turn envelope, frontend badges.
- Design decisions and rationale.

## Non-Scope

- Pipeline orchestration and depth routing (see `PIPELINE-REFERENCE.md`).
- Tool definitions and capabilities (see `TOOL-CATALOG.md`).
- Frontend badge rendering (see lexe-webchat CLAUDE.md).
- GDPR implications of quality metrics (see `DATA-FLOWS-GDPR.md`).

---

## 1. Scoring Formula

**Source file:** `lexe-core/src/lexe_core/agent/scoring.py`

```python
raw_score = (
    source_reliability * 0.30
    + vigenza_match    * 0.25
    + link_verification * 0.20
    + cross_source     * 0.15
    + text_quality     * 0.10
)

overall = round(raw_score * 100)
```

Each component function returns a float in [0.0, 1.0]. The weighted sum produces a raw score in [0.0, 1.0], which is then scaled to an integer in [0, 100].

After the raw computation, hard penalties, caps, and floor are applied (see section 3).

### Weight Rationale

| Component | Weight | Why |
|-----------|--------|-----|
| **source_reliability** | 0.30 | Most important: primary sources (Normattiva, EUR-Lex, KB) are authoritative; secondary sources (web scraping) are unreliable |
| **vigenza_match** | 0.25 | Critical for legal accuracy: a norm that has been abrogated or modified can lead to wrong advice |
| **link_verification** | 0.20 | URL quality reflects whether the user can verify the citation independently |
| **cross_source** | 0.15 | Multiple sources agreeing increases reliability; authority-weighted for jurisprudence |
| **text_quality** | 0.10 | Having actual text excerpts (vs. just titles) allows verification; lowest weight because absence is often a tool limitation, not a quality signal |

---

## 2. Component Functions

### 2.1 Source Reliability (`_score_source_reliability`)

Measures the proportion of evidence items coming from primary vs secondary sources.

**Scoring:**
```
score = primary_count / (primary_count + secondary_count)
```

**Source classification:**

| Evidence Type | Primary Sources | Secondary Sources |
|--------------|----------------|-------------------|
| Norms | `normattiva`, `eurlex` | Everything else |
| Case Law | `kb_search`, `infolex`, `kb_lexe` | Everything else |
| Massime | `kb_search` | Everything else |

**Edge case:** If no evidence items exist, returns `0.0`.

### 2.2 Vigenza Match (`_score_vigenza_match`)

Measures how many norms have verified vigenza (currency) status.

**Per-norm scoring:**

| Condition | Score |
|-----------|-------|
| `norm.vigenza.verified == True` (API-confirmed) | 1.0 |
| `norm.source == "kb_normativa_search"` (KB, presumed current) | 0.7 |
| All other sources | 0.3 |

**Aggregate:** Average of per-norm scores.

**Edge case:** If no norms in evidence pack, returns `0.5` (neutral -- vigenza not applicable to pure jurisprudence queries).

**Special case:** If `vigenza_skipped=True` (early exit), the function is bypassed and the component is set to `0.5`.

### 2.3 Link Verification (`_score_link_verification`)

Measures URL quality across all evidence types (norms, case law, massime).

**URL tier weights:**

| Tier | Weight | Description |
|------|--------|-------------|
| 1 | 1.0 | Direct authoritative URL (normattiva.it, eur-lex.europa.eu) |
| 2 | 0.7 | Reliable secondary URL (dejure.it, brocardi.it, altalex.com) |
| 3 | 0.4 | Search-derived URL (SearXNG result) |
| 0 | 0.0 | No URL available (internal source only, "Fonte interna") |

**Aggregate:** Average of tier weights across all items.

**Edge case:** If no items, returns `0.5`.

**Note on massime:** Since the massimario multi-format search was deployed (2026-03-15), URLs are resolved via 3-strategy cascade per massima. When no authoritative URL is found, the URL is empty (tier 0) rather than a fabricated fallback. Negative caching (24h TTL) prevents repeated SearXNG queries.

### 2.4 Cross-Source (`_score_cross_source`)

Combines source diversity with authority-weighted jurisprudence scoring.

**Source diversity base:**

| Unique Sources | Score |
|---------------|-------|
| 3+ | 1.0 |
| 2 | 0.7 |
| 1 | 0.4 |
| 0 | 0.0 |

**Authority weighting:** For massime with `sezione` or `authority` metadata, compute the average authority weight (see section 5).

**Blending formula:**
```
blended = diversity_score * 0.70 + avg_authority * 0.30
```

**Conformity bonus:** If 3 or more massime have authority weight >= 0.5, add `CONFORMITY_BONUS = 0.15` (capped at 1.0).

**Edge case:** If no massime have authority metadata, returns pure diversity score.

### 2.5 Text Quality (`_score_text_quality`)

Measures whether norm evidence includes substantive text excerpts.

**Scoring:**
```
score = count(norms with text_excerpt > 50 chars) / total_norms
```

**Edge case:** If no norms, returns `0.5`.

**Rationale:** Text excerpts longer than 50 characters indicate the tool successfully retrieved article content, not just metadata. This is a proxy for retrieval quality.

---

## 3. Hard Penalties, Caps, and Floor

Applied **after** the raw score computation, in this order:

| Rule | Condition | Effect | Rationale |
|------|-----------|--------|-----------|
| **Existence fail cap** | `any_existence_failed=True` | `overall = min(overall, 60)` | If a primary citation (norm) failed deterministic existence check on Normattiva, the response may cite a non-existent law |
| **Vigenza skip cap** | `vigenza_skipped=True` | `overall = min(overall, 55)` | If vigenza verification was skipped (early exit/timeout), we cannot guarantee the cited norms are current |
| **No evidence floor** | No norms, no case_law, no massime | `overall = 0` | No evidence = no confidence |
| **Minimum floor** | Evidence exists AND `overall < 40` | `overall = 40` | UX hotfix: avoids misleadingly low scores when evidence exists but scoring components are individually low |
| **Clamp** | Always | `overall = max(0, min(100, overall))` | Safety bound |

### Application Order

```
1. Compute raw_score (weighted sum)
2. Scale: overall = round(raw_score * 100)
3. Apply existence_fail cap (60)
4. Apply vigenza_skip cap (55)
5. Clamp [0, 100]
6. Check no-evidence → 0
7. Apply floor (40) if evidence exists
```

---

## 4. 3-Band Thresholds

| Band | Range | Label | Color (Frontend) | Meaning |
|------|-------|-------|-----------------|---------|
| **VERIFIED** | >= 45 | `VERIFIED` | Green | Evidence is well-sourced, vigenza confirmed, citations verifiable |
| **CAUTION** | 25 -- 44 | `CAUTION` | Amber/Yellow | Some verification gaps: missing vigenza, secondary sources, or incomplete URLs |
| **LOW_CONFIDENCE** | < 25 | `LOW_CONFIDENCE` | Red | Significant gaps: existence failures, no verifiable sources, or very sparse evidence |

**Note:** With the floor at 40, a `LOW_CONFIDENCE` score in practice only occurs when:
- There is zero evidence (score = 0), OR
- The raw score before floor was < 25 AND the floor was not applied (which cannot happen -- floor always applies when evidence exists).

In practice, the effective bands with evidence present are: VERIFIED (>= 45) and CAUTION (40 -- 44). LOW_CONFIDENCE (0) means "no results found".

---

## 5. Authority Weights (Intervento D)

**Source file:** `lexe-core/src/lexe_core/agent/scoring.py` -- `classify_authority()` and `_AUTHORITY_WEIGHTS`

| Classification | Weight | Judicial Body |
|---------------|--------|--------------|
| `sezioni_unite` | 1.0 | Corte di Cassazione a Sezioni Unite -- highest judicial authority, binding precedent |
| `cassazione` | 0.8 | Corte di Cassazione (single section) -- authoritative precedent |
| `corte_appello` | 0.5 | Corte d'Appello -- intermediate appellate court |
| `tribunale` | 0.3 | Tribunale -- first instance court |
| `isolato` | 0.2 | Default -- unclassified source, isolated ruling |

### Classification Logic

`classify_authority(metadata)` examines two fields from massima metadata:

1. **`sezione`** field (primary):
   - Contains "unite" or equals "su" -> `sezioni_unite`
   - Is a number (1-6) or Roman numeral (i-vi) -> `cassazione`

2. **`authority`** field (fallback):
   - Contains "sezioni unite" or "sez. un." -> `sezioni_unite`
   - Contains "cassazione" or "cass." -> `cassazione`
   - Contains "appello" or "corte d'appello" -> `corte_appello`
   - Contains "tribunale" -> `tribunale`

3. **Default:** `isolato`

### Conformity Bonus

When 3 or more massime from authoritative sources (weight >= 0.5, i.e., `corte_appello` or higher) agree on a legal point, a `CONFORMITY_BONUS = 0.15` is added to the cross_source component. This rewards concordant jurisprudence from significant courts.

---

## 6. Verification Pipeline

**Source file:** `lexe-core/src/lexe_core/verification/runner.py`

The verification pipeline runs 5 steps in sequence. Steps 4 and 5 are conditional on depth level.

### Step 1: Structural Checks (always)

**Source:** `verification/structural.py`

Regex-based checks on evidence items. No LLM, no network calls.

| Check | What It Detects | Severity |
|-------|----------------|----------|
| URN format | Malformed Normattiva URN (missing date, invalid authority) | error |
| Article reference | "Art. X" without valid norm identifier | warning |
| Date consistency | Norm date in the future or before 1861 | error |
| Duplicate detection | Same norm cited multiple times with different text | warning |

**Output:** List of `VerificationIssue` with `severity` (error/warning) and `message_it` (Italian description).

### Step 2: Vigenza Consistency (always)

Checks that vigenza status is internally consistent across evidence items.

| Check | Condition | Severity |
|-------|-----------|----------|
| Abrogated norm cited as current | `vigenza.verified=True` but `vigenza.status=abrogated` | error |
| Conflicting vigenza across sources | Same norm shows different vigenza status from different tools | warning |

### Step 3: Cross-Reference Check (always)

Verifies that norms referenced in massime text are present in the evidence pack.

| Check | Condition | Severity |
|-------|-----------|----------|
| Referenced norm missing | Massima cites "Art. 2043 c.c." but no matching norm in `evidence_pack.norms` | warning |

### Step 4: LLM Coherence (conditional)

**Runs only at:** STANDARD and DEEP depth levels.

Uses `lexe-verifier` model (Gemini 3 Flash, reasoning=low) to check:
- Whether the synthesized response is consistent with the evidence pack.
- Whether legal citations are correctly attributed.
- Whether the conclusion follows from the cited norms.

**Budget:** Single LLM call, max 8192 tokens.

### Step 5: Self-Correction (conditional)

**Runs only at:** DEEP depth level.

**Budget:** 15 seconds, maximum 2 attempts.

If the verifier identifies issues (status = `warn` or `fail`):
1. Feeds the verification issues back to the synthesizer.
2. Synthesizer produces a corrected response.
3. Re-runs verification on the corrected response.
4. If the second verification is worse, keeps the original.

**Source:** ADR-D6 (self-correction max 2 attempts -- more attempts showed diminishing returns in benchmark).

### Verdict Status

| Status | Condition | Downstream Effect |
|--------|-----------|-------------------|
| `pass` | 0 errors, 0-1 warnings | No impact on confidence |
| `warn` | 1 error OR 2+ warnings | Triggers self-correction at DEEP |
| `fail` | 2+ errors | Triggers self-correction at DEEP; may cap confidence |

---

## 7. Gap Detection

**Source file:** `lexe-core/src/lexe_core/agent/gap_reporter.py`

### `compute_research_gaps()`

Compares the evidence pack against the research plan to identify what was not found.

**Gap types:**

| Gap Type | Detection Method | Example |
|----------|-----------------|---------|
| **Missing tool** | Plan requested a tool that was never called (timeout/error) | "Ricerca Normattiva non eseguita (timeout)" |
| **Empty result** | Tool was called but returned zero results | "Nessuna massima trovata nel Massimario" |
| **Referenced but unfetched** | Massima text cites a norm not present in evidence | "Art. 2043 c.c. citato in giurisprudenza ma non verificato" |
| **Partial coverage** | Plan had N subtasks, only M < N produced evidence | "3 su 5 fonti consultate" |
| **Depth cutoff** | Budget exhausted before all planned research complete | "Ricerca interrotta per limite budget (MINIMAL)" |

### `extract_missing_norms()`

Extracts norm citations from massime text that are not present in the evidence pack.

Uses regex `_NORM_CITE_RE` to find patterns like "art. 2043 c.c.", "art. 575 c.p.", "art. 24 Cost." and cross-checks against `evidence_pack.norms`.

Output is used by the remediation loop (gap_analyst agent in multi-agent research) to auto-fetch missing norms.

---

## 8. Post-Verification Calibration

After verification and scoring, calibration rules adjust the confidence band for specific scenarios.

| Rule | Condition | Adjustment | Rationale |
|------|-----------|------------|-----------|
| **CHAT intent** | Intent is CHAT (non-legal) | Force VERIFIED | Chat responses don't need legal verification |
| **Caselaw absence + user asked** | User explicitly asked about jurisprudence but none found | Downgrade to CAUTION if currently VERIFIED | User expectation gap |
| **Self-correction downgrade** | Self-correction produced a worse verification score | Preserve the downgrade, do not re-calibrate up | Conservative: if the model struggled, flag it |

### What Is NOT a Penalty (BK-006 Fix C)

**Jurisprudence absence is NOT a penalty.** This was a critical fix (BK-006 Fix C, Sprint 9):

- **Before:** Missing case law reduced confidence, which penalized purely normative queries (e.g., "What does Art. 2043 c.c. say?").
- **After:** Confidence reflects norm verification (vigenza + existence) only. Jurisprudence is informational enrichment.
- **`coverage_gap`** from the LLM verifier is informational -- it is logged but does NOT trigger a hard cap.

---

## 9. Integration Points

### 9.1 Langfuse

**Source:** `lexe-improve-lab/src/lexe_improve/collector/langfuse.py`

After each conversation turn, the confidence score is reported to Langfuse:

| Score Value | Mapping |
|-------------|---------|
| 1.0 | VERIFIED (>= 45) |
| 0.5 | CAUTION (25-44) |
| 0.0 | LOW_CONFIDENCE (< 25) |

Used for: LLM observability, prompt quality tracking, model comparison.

### 9.2 Prometheus

**Source:** `lexe-core/src/lexe_core/gateway/quality_metrics.py`

| Metric | Type | Description |
|--------|------|-------------|
| `lexe_confidence_score` | Histogram | Overall confidence score distribution |
| `lexe_evidence_norms_count` | Histogram | Norms per evidence pack |
| `lexe_evidence_massime_count` | Histogram | Massime per evidence pack |
| `lexe_evidence_caselaw_count` | Histogram | Case law refs per evidence pack |
| `lexe_synth_output_chars` | Histogram | Synthesizer output length |
| `lexe_pipeline_duration_seconds` | Histogram | End-to-end pipeline time |
| `lexe_tool_calls_total` | Counter | Tool invocations by tool name |
| `lexe_verification_status_total` | Counter | Verification verdicts (pass/warn/fail) |

### 9.3 Turn Envelope

**Source:** `lexe-core/src/lexe_core/gateway/turn_envelope.py`

The confidence score, breakdown, and verification verdict are persisted in the turn envelope, which is stored in `core.conversation_events` for each pipeline execution.

Fields persisted:
- `confidence.overall` (int 0-100)
- `confidence.breakdown_note` (Italian text)
- `confidence.normativa`, `confidence.giurisprudenza`, `confidence.vigenza`, `confidence.coverage`, `confidence.cross_source_coherence` (float 0.0-1.0)
- `verification.status` (pass/warn/fail)
- `verification.checks_run`, `verification.checks_passed`
- `verification.issues[]` (list of issue descriptions)

### 9.4 Frontend 3-Band Badges

**Source:** `lexe-webchat/src/components/chat/CitationBadge.tsx`

| Band | Badge Color | Icon | Tooltip |
|------|------------|------|---------|
| VERIFIED | Green ring | Checkmark | "Fonti verificate" |
| CAUTION | Amber ring | Warning triangle | "Verifica parziale" |
| LOW_CONFIDENCE | Red ring | X mark | "Confidenza bassa" |

Interactive `[N]` citation badges are rendered inline in AI responses. Clicking a badge:
1. Highlights the corresponding evidence item in the side panel.
2. Auto-scrolls to the item.
3. Shows a 2.5-second ring effect.

The trace hash (8-char hex) is extracted from the SSE `done` event and stored in `ChatMessage.traceHash` for linking to the admin trace view.

---

## 10. Design Notes

### Why 5 Components (Not 3 or 7)?

The 5-component model was chosen after benchmark evaluation (ADR-D5). Fewer components collapsed distinct quality signals (e.g., merging vigenza into source_reliability lost the ability to detect "correct source, abrogated norm"). More components added noise without improving discrimination in the benchmark.

### Why Floor at 40?

Without the floor, a query with valid evidence but low URL tier and single-source results could score 20-30, which users perceived as "broken AI". The floor at 40 ensures that when evidence exists, the score stays in CAUTION range at minimum, which is more honest than LOW_CONFIDENCE for responses that are directionally correct but not fully verified.

### Why Existence Fail Caps at 60 (Not 0)?

An existence failure means one cited norm was not found on Normattiva. This is serious but not catastrophic: the norm might exist under a different URN, or Normattiva might have a transient issue. Capping at 60 (VERIFIED range) rather than zeroing preserves the value of the other evidence while signaling the issue.

### Why Jurisprudence Absence Is Not a Penalty (BK-006)

Italian legal practice does not require jurisprudence for every question. Many queries ("What are the requirements for a DPA under Art. 28 GDPR?") are purely normative. Penalizing the absence of case law for such queries was producing misleading CAUTION scores for correct, well-sourced answers. The fix removed the penalty and reclassified jurisprudence as informational enrichment.

---

## File References

| File | Role |
|------|------|
| `lexe-core/src/lexe_core/agent/scoring.py` | Confidence scoring algorithm + authority weights |
| `lexe-core/src/lexe_core/agent/models.py` | `ConfidenceScore`, `EvidencePack`, `VerificationVerdict` dataclasses |
| `lexe-core/src/lexe_core/verification/runner.py` | Verification pipeline orchestrator |
| `lexe-core/src/lexe_core/verification/structural.py` | Structural checks (regex, no LLM) |
| `lexe-core/src/lexe_core/agent/gap_reporter.py` | Gap detection + missing norm extraction |
| `lexe-core/src/lexe_core/agent/verifier.py` | LLM-based coherence verification |
| `lexe-core/src/lexe_core/gateway/quality_metrics.py` | Prometheus metrics instrumentation |
| `lexe-core/src/lexe_core/gateway/turn_envelope.py` | Turn envelope persistence |
| `lexe-core/src/lexe_core/agent/unified_pipeline.py` | Pipeline entry point (calls scoring) |
| `lexe-docs/knowhow/adr/ADR-D5-confidence-spectrum.md` | Architecture decision record for scoring |
| `lexe-docs/knowhow/adr/ADR-D6-self-correction-max-2.md` | Architecture decision record for self-correction |

---

*Ultimo aggiornamento: 2026-03-19*
