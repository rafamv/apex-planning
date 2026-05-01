# detect_d_v2 Step 3 Ship Report — Session 40

**Branch:** `phase3-v2-detect-d`
**Step:** 3 of 3 (final, per revised proposal §7 Sequencing — consolidated 5→3 commits at Session 40)
**Date:** 2026-05-01

---

## Status

Step 3 shipped + tests green. **detect_d_v2 functionally complete.** 29/29 detect_d_v2 tests pass (12 Step 1 + 12 Step 2 + 5 Step 3); 97/97 full suite pass (68 prior + 29 new).

Branch ready for chat-Claude diff review across all 3 commits, then merge to main per per-detector branch convention.

---

## Commits

apex-planning (proposal patches first, separate commits):
```
4b6b5f5 detect_d_v2 proposal §6+§9: align OI column to detect_a_v2 (open_interest, not open_interest_value)
992d1ad detect_d_v2 proposal §7: consolidate 5-commit sequence to 3-commit (Session 40 architect decision)
aaa822c detect_d_v2 Pre-Write findings (Session 40)
```

apex (3-commit detector chain on `phase3-v2-detect-d`):
```
a189340 phase3-v2 step 3: detect_d_v2 soft confidence boosts
6b7de69 phase3-v2 step 2: detect_d_v2 functional core (stop + emission + counterfactual)
c19372a phase3-v2 step 1: detect_d_v2 hard gates skeleton (long+short)
```

(Brief v12.3 deploy at `c34178a` is the prior commit on branch, pre-detect_d_v2 work.)

---

## Files touched

| Path | Lines (+/-) | Description |
|------|-------------|-------------|
| `<REPO_ROOT>apex/signals/setups.py` | +60 / -8 | Added Stage 6 (4 soft boosts) between Stage 4 (counterfactual) and Stage 7 (emission). Updated Stage 7 to use boost-accumulated `confidence` (not just `CONFIDENCE_BASE`). Removed `# noqa: F841` from `cross_bar_timestamp` (now consumed by Boost 4). Updated docstring to reflect Stage 6 active. |
| `<REPO_ROOT>tests/test_detect_d_v2.py` | +118 / -1 | Added `CONFIDENCE_BOOST` import + `_build_oi_history(timestamps, values)` helper. 5 new tests (25-29) for Boost 1 (RSI), Boost 2 (volume 2x), Boost 3 (funding sign-aligned with magnitude threshold), Boost 4 (OI delta cross-aligned), and Boost 4 circuit breaker (cross_age > cap). Module docstring extended for Step 3 scope. |

Total this step: +178 / -9.

---

## Implementation notes

### Boost 3 funding semantic resolution

Architect's literal Step 3 spec had a sign-flipped rule:
> long: funding_rate > 0 → boost fires
> short: funding_rate < 0 → boost fires

This contradicted detect_a_v2 (shipped) AND Brief v12.3 §3.1 ("short setup with positive funding = crowded longs paying = room to run") AND the architect's own parenthetical reasoning ("shorts paying longs = market positioned for rise" describes NEGATIVE funding, not positive).

I stopped before writing Boost 3 and surfaced the contradiction in chat. Architect resolution received:
> long: funding_rate < -FUNDING_RATE_MAGNITUDE_THRESHOLD → boost fires
> short: funding_rate > FUNDING_RATE_MAGNITUDE_THRESHOLD → boost fires

Implemented per the corrected semantic. Magnitude-gated (uses `FUNDING_RATE_MAGNITUDE_THRESHOLD = 0.0001` constant from detect_a_v2). Single direction-agnostic label `'funding_directional_alignment'` per architect's instruction (differs from detect_a_v2's split `'funding_favorable_long'` / `'funding_favorable_short'` — Brief v13 may canonicalize cross-detector labeling).

### Boost 4 OI column resolution applied + proposal patched

Per Pre-Write Item 6 architect resolution, Boost 4 reads `oi_history.iloc[...]['open_interest']` (contracts), NOT `'open_interest_value'` (USD). Reasoning: USD-OI conflates real position changes with price drift in the trend direction — for a trending-continuation detector, contract-OI is the cleaner accumulation signal.

Proposal patch shipped to apex-planning at `4b6b5f5` BEFORE the apex code commit per architect's pre-work instruction. Changed `open_interest_value` → `open_interest` in:
- §6 Stage 6 Boost 4 (Read line)
- §9 Pre-Write Verification Checklist Item 6 (verification line)
- Added one-line "Resolution (Session 40)" note at end of §9 Item 6

### Boost label format

Architect's spec for Boost 4 specified data-embedded label `f'oi_delta_pct:{oi_delta_pct:.2f}'` (carries the actual computed delta percentage as part of the string). Diverges from detect_a_v2's static label `'oi_delta_significant'`. Implemented per architect's spec; surfacing as cross-detector label-format consistency question.

Other Boost labels per architect's spec:
- Boost 1: `'rsi_directional_alignment'` (single, direction-agnostic)
- Boost 2: `'volume_2x_above_ma25'`
- Boost 3: `'funding_directional_alignment'` (single, direction-agnostic per Step 3 prompt)
- Boost 4: `f'oi_delta_pct:{value:.2f}'` (data-embedded)

All differ from detect_a_v2's labeling style — flagging for future Brief v13 cross-detector canonicalization.

### Confidence accumulation

Implemented per architect spec:
```python
confidence = CONFIDENCE_BASE             # 0.75 baseline
# Each boost: confidence += CONFIDENCE_BOOST  # +0.05
# Max with all 4 boosts firing: 0.75 + 4*0.05 = 0.95 (no min() cap needed since 0.95 < 1.0)
```

Stage 7 now passes `confidence=confidence` (accumulated) instead of literal `CONFIDENCE_BASE`. The Step 2 test `test_confidence_equals_base_no_boosts_at_step_2` continues to pass because the default fixture has zero boost-firing substrate (rsi=50, volume=1001, funding=0, oi_history empty) — confidence stays at 0.75.

### `cross_bar_timestamp` noqa removed

Step 1 + Step 2 had `cross_bar_timestamp = int(...)  # noqa: F841` because the variable was dead-stored awaiting Step 3 consumption. Boost 4 now reads it (`oi_history[oi_history['ts'] <= cross_bar_timestamp]`). Removed noqa marker; updated inline comment.

### Defensive guards in Boost 4

OI lookup is wrapped in nested `if` checks to handle:
- `oi_history` key missing from indicators → skip silently (substrate gap, no error)
- `oi_history` empty DataFrame → skip silently
- No rows with `ts <= cross_bar_timestamp` (oi_history doesn't extend that far back) → skip silently
- No rows with `ts <= breakout_ts` (oi_history doesn't include current bar) → skip silently
- `oi_at_cross == 0` or `pd.isna(oi_at_cross)` → skip silently (avoid div-by-zero)

Same defensive pattern as detect_a_v2. All "skip" paths leave confidence and confirming_indicators unchanged.

### Test 28 + 29 oi_history fixture construction

Constructed via `_build_oi_history(timestamps, values)` helper. For test 28 (boost fires):
- Timestamps: `[1777013800000, 1777014700000]` — at cross_bar_timestamp (idx 46) and at current candle ts (idx 49)
- Values: `[100.0, 102.0]` → delta = 2.0% > 1.0% threshold → boost fires

For test 29 (circuit breaker fires before lookup):
- Override ema_21 series so cross is at age 10 (still within `CROSS_LOOKBACK_BARS_V2` = 20, so Stage 2 passes; outside `CROSS_AGE_CAP_BARS_V2` = 6, so Boost 4 short-circuits)
- oi_history with 10% delta provided to prove the circuit breaker fires BEFORE the OI lookup (boost wouldn't fire even if lookup ran, but the test verifies the SKIP path)

---

## Tests

`pytest tests/test_detect_d_v2.py -v` (29 tests):

```
collected 29 items

[Step 1 tests 1-12 — all PASSED]
test_no_indicators_returns_none PASSED
test_insufficient_5m_history_returns_none PASSED
test_mixed_1h_trend_returns_none PASSED
test_inverse_mixed_1h_trend_returns_none PASSED
test_long_no_recent_cross_returns_none PASSED
test_long_cross_too_old_returns_none PASSED
test_short_no_recent_cross_returns_none PASSED
test_long_no_pullback_returns_none PASSED
test_long_pullback_close_below_ma_returns_none PASSED
test_short_no_pullback_returns_none PASSED
test_short_pullback_close_above_ma_returns_none PASSED
test_no_volume_returns_none PASSED

[Step 2 tests 13-24 — all PASSED]
test_long_stop_at_sma_50_when_immediately_below PASSED
test_long_stop_at_ema_50_when_ema_50_nearer_below_close PASSED
test_short_stop_at_sma_50_when_immediately_above PASSED
test_long_stop_falls_back_to_bb_middle_when_only_one_ma_below PASSED
test_long_emits_signal_with_correct_fields PASSED
test_short_emits_signal_with_correct_fields PASSED
test_confidence_equals_base_no_boosts_at_step_2 PASSED
test_account_target_is_scalp PASSED
test_counterfactual_logs_cross_age_when_present PASSED
test_counterfactual_logs_None_when_no_cross PASSED
test_counterfactual_does_not_affect_signal_emission PASSED
test_counterfactual_uses_independent_walkback_from_live_substrate PASSED

[Step 3 tests 25-29 — all PASSED]
test_rsi_boost_fires_long_when_rsi_above_50 PASSED
test_volume_2x_boost_fires_when_volume_above_threshold PASSED
test_funding_boost_fires_long_when_funding_negative_below_threshold PASSED
test_oi_delta_boost_fires_when_within_cross_age_cap_and_positive_delta PASSED
test_oi_delta_boost_skipped_when_cross_age_exceeds_cap PASSED

============================== 29 passed in 0.99s ==============================
```

Full suite (`pytest tests/`): **97 passed in 3.71s** (68 prior + 29 new). No regressions in detect_a_v2, store_signals, Phase 1 v2 handlers, MA mirror prep tests.

---

## Open questions for chat-Claude

1. **Cross-detector label-format consistency.** detect_d_v2 uses different label formats than detect_a_v2:

   | Boost | detect_a_v2 (shipped) | detect_d_v2 (this commit) |
   |-------|----------------------|---------------------------|
   | RSI | `rsi_confirms_long` / `rsi_confirms_short` | `rsi_directional_alignment` |
   | Volume 2x | `volume_2x_ma25` | `volume_2x_above_ma25` |
   | Funding | `funding_favorable_long` / `funding_favorable_short` | `funding_directional_alignment` |
   | OI delta | `oi_delta_significant` | `f'oi_delta_pct:{value:.2f}'` |
   | Counterfactual | (none in detect_a_v2) | `f'counterfactual_ema21_ema50_cross_age_bars:{value}'` |

   detect_d_v2 follows architect's explicit Step 2 + Step 3 instructions for these labels. Brief v13 may canonicalize cross-detector. Not load-bearing for current work; flagging for tracking.

2. **Boost 3 magnitude threshold inheritance.** detect_d_v2 reuses `FUNDING_RATE_MAGNITUDE_THRESHOLD` (= 0.0001) from detect_a_v2. Per Brief v12.3 §3.1 threshold-validation discipline, this value is a placeholder pending Phase 6 paper-trading data. Same constant means same threshold — appropriate for cross-detector evaluation, but if detect_d's pullback-continuation semantic warrants different funding-magnitude sensitivity than detect_a's compression-breakout, that's a Phase 6 retune target.

3. **`_build_oi_history` helper duplication.** Both `tests/test_detect_a_v2.py` and `tests/test_detect_d_v2.py` define `_build_oi_history(timestamps, values)` identically (2-column DataFrame with `ts` + `open_interest`). Single duplicated helper across 2 test files — could lift to `tests/conftest.py` or `tests/_helpers.py`. Not blocking; flag for cross-detector test consolidation later (b/c/e/f detectors will likely also need this helper).

4. **Boost 4 OI delta sign interpretation.** Currently fires only on `oi_delta_pct > OI_DELTA_PERCENT_THRESHOLD` (positive delta). Per substrate decision 5, positive OI delta = positions accumulating during continuation = favorable for both directions. But: a SHORT setup with NEGATIVE OI delta would mean positions UNwinding during the continuation — which arguably shows ANOTHER kind of conviction (longs forced out as price drops). Currently no boost for that case. Surfacing as a possible Phase 6 evaluation question, not blocking.

5. **Tests 14 + 16 semantic substitution from Step 2 — still standing.** Reminder: I substituted reachable test variants for two tests where the architect's literal test name described structurally-unreachable Stage 5 states (close > ema_21 > sma_50 invariant). Step 2 ship report flagged this; not re-raising in Step 3 unless chat-Claude wants the original literal names back via xfail.

---

## Time/scope

- **LOC written this step:** 178 (production: 60; tests: 118)
- **Cumulative LOC for detect_d_v2 across Steps 1-3:** ~1,033 (production: 332; tests: 701)
- **Elapsed wall-clock for Step 3:** ~25 minutes from "open_interest resolution + funding correction" message to commit `a189340` (includes proposal patch in apex-planning, mid-implementation funding-sign STOP+REPORT cycle, code implementation of Stage 6, 5 new tests, full regression).
- **Test runtime:** 0.99s (29 tests) / 3.71s (97 full)

---

## Cumulative branch state — detect_d_v2 complete

`phase3-v2-detect-d` branch chain off main HEAD `20d78fe`:

```
a189340 phase3-v2 step 3: detect_d_v2 soft confidence boosts
6b7de69 phase3-v2 step 2: detect_d_v2 functional core (stop + emission + counterfactual)
c19372a phase3-v2 step 1: detect_d_v2 hard gates skeleton (long+short)
c34178a Brief v12.3: Setup D symmetric, MA-pair principle, substrate validation discipline, cross-age cap convention
20d78fe Merge phase3-v2-prep-ma-mirror: ...  ← main
```

apex-planning supporting documents:
- `detect_d_v2_proposal.md` (canonical spec, Session 39 + Session 40 sequencing/OI patches)
- `detect_d_v2_prewrite_findings.md` (Session 40, commit `aaa822c`)
- `detect_d_v2_step1_report.md` (Session 40, commit `fad518c`)
- `detect_d_v2_step2_report.md` (Session 40, commit `5d12a52`)
- `detect_d_v2_step3_report.md` (Session 40, this report)

---

## Next step

Awaiting chat-Claude diff review (commits `c19372a` + `6b7de69` + `a189340` on `origin/phase3-v2-detect-d`) and architect approval to merge to main per per-detector branch convention (Brief v12.3 Section 8: `--no-ff` to preserve detector boundaries in main history).

After detect_d_v2 merges to main, next sequenced detector per Brief v12.3 Section 8 is **detect_f_v2** (Setup F — Capitulation Candle at 1h Bollinger Extreme), branched off the new main HEAD.
