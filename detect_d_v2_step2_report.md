# detect_d_v2 Step 2 Ship Report — Session 40

**Branch:** `phase3-v2-detect-d`
**Step:** 2 of 3 (per revised proposal §7 Sequencing — consolidated from 5-commit to 3-commit per Session 40 architect decision)
**Date:** 2026-05-01

---

## Status

Step 2 shipped + tests green. 24/24 detect_d_v2 tests pass (12 Step 1 hard-gate tests + 12 Step 2 functional-core tests); 92/92 full suite pass (80 prior + 12 new at Step 2).

Note: a sequencing-revision proposal patch shipped to apex-planning at `992d1ad` BEFORE the apex code commit (per architect's pre-work instruction).

---

## Commits

apex-planning (proposal patch first):
```
992d1ad detect_d_v2 proposal §7: consolidate 5-commit sequence to 3-commit (Session 40 architect decision)
```

apex (code commit):
```
6b7de69 phase3-v2 step 2: detect_d_v2 functional core (stop + emission + counterfactual)
c19372a phase3-v2 step 1: detect_d_v2 hard gates skeleton (long+short)  ← prior
```

---

## Files touched

| Path | Lines (+/-) | Description |
|------|-------------|-------------|
| `<REPO_ROOT>apex/signals/setups.py` | +143 / -46 | Extracted `_find_cross_age_bars()` module-level helper (pure walk-back used by Stages 2 + 4). Refactored Stage 2 to use the helper. Added Stage 5 (MA-stack ranking with bb_middle fallback per `len < 2` condition + pathological-case None on stop-entry inversion). Added Stage 4 (counterfactual ema_21/ema_50 cross via shared helper, logged as `'counterfactual_ema21_ema50_cross_age_bars:<int|None>'` string in confirming_indicators). Added Stage 7 (Signal emission at CONFIDENCE_BASE = 0.75 with setup_type='D', tp=None, tp_strategy='structural_5m_bb', account_target='scalp', holding_period='scalp'). Updated docstring to reflect Steps 5+4+7 active. |
| `<REPO_ROOT>tests/test_detect_d_v2.py` | +203 / -7 | Added `Signal` + `CONFIDENCE_BASE` imports. Updated `synthetic_long_pullback_indicators` fixture: `bb_middle = 69550` (was 69500, distinct from `sma_50` for unambiguous Stage 5 fallback test). Updated `synthetic_short_pullback_indicators`: `bb_middle = 69450`. Added 12 new tests (13-24) covering Stage 5 stop placement (4), Stage 7 base emission (4), Stage 4 counterfactual logging (4). Module docstring extended for Step 2 scope. |

Total: +346 / -53.

---

## Implementation notes

### Helper extraction

`_find_cross_age_bars(ind, fast_col, slow_col, direction, lookback)` extracted as a module-level pure function. Used by:
- Stage 2 (live trigger): `_find_cross_age_bars(ind_5m, 'ema_21', 'sma_50', direction, CROSS_LOOKBACK_BARS_V2)`
- Stage 4 (counterfactual): `_find_cross_age_bars(ind_5m, 'ema_21', 'ema_50', direction, CROSS_LOOKBACK_BARS_V2)`

Refactor was deferred from Step 1 per ship-report open question; Step 2 needed it for Stage 4 anyway. Refactor preserved Step 1 test behavior (12/12 still pass after refactor before adding new tests).

### Stage 5 stop placement — interpretation locked

Architect's instruction had a literal-vs-reachable spec conflict on tests 14 and 16:
- Test 14: `long_stop_at_ema_50_when_sma_50_above_close` — but for valid long Stage 1+2+3, close > ema_21 > sma_50 always (transitivity), so `sma_50 above close` is structurally unreachable.
- Test 16: `long_stop_falls_back_to_bb_middle_when_no_ma_below` — but `sma_50 < close` always for long, so `no_ma_below` is also unreachable.

Resolution: implemented Stage 5 per architect's algorithm (nearest MA below close when ≥2 MAs available, fallback to bb_middle when fewer than 2). For the tests, substituted reachable variants of the same logic:
- Test 14 (revised name `_when_ema_50_nearer_below_close`): override sma_50=69400, ema_50=69500 → mas_below = [69400, 69500, 69300, 69100], max = 69500 = ema_50. Tests "ema_50 is nearest MA below close" via ema_50 > sma_50 within mas_below set.
- Test 16 (revised name `_when_only_one_ma_below`): override ema_50=69800, sma_100=69900, sma_250=70000 (all above close), keep sma_50=69500 (below close). mas_below = [sma_50] only, len < 2 → fallback to bb_middle. Tests the `len < 2` fallback condition with `len == 1` (the reachable boundary).

Implementation algorithm:
```python
mas_below = [m for m in [sma_50, ema_50, sma_100, sma_250] if not pd.isna(m) and m < close]
if len(mas_below) >= 2:
    stop = max(mas_below)        # nearest below close
else:
    stop = bb_middle             # fallback
if pd.isna(stop) or stop >= close:
    return None                  # pathological
```

Mirror for short: `mas_above`, `min(mas_above)`, `stop <= close` pathology check.

### Stage 4 counterfactual — string format

confirming_indicators is `List[str]` per Signal dataclass. Architect's instruction "log result as 'counterfactual_ema21_ema50_cross_age_bars': <int or None>" used dict-syntax — interpreted as string format `f'counterfactual_ema21_ema50_cross_age_bars:{value}'`. This matches detect_a_v2's existing convention (boost labels are strings). Value examples:
- `'counterfactual_ema21_ema50_cross_age_bars:3'` (cross found, age 3 bars)
- `'counterfactual_ema21_ema50_cross_age_bars:None'` (no cross or precondition fail)

Counterfactual is appended to confirming_indicators alongside the 4 baseline strings. Does NOT affect emission, gating, or confidence.

### Baseline confirming_indicators strings for detect_d_v2

Architect didn't specify exact baseline labels for detect_d_v2 (proposal §6 used different strings than what shipped for detect_a_v2). Picked four labels reflecting Stage 1+2+3 hard gates:
- `'1h_trend_alignment'` (Stage 1.2 — matches detect_a_v2 verbatim)
- `'ma_cross_recent'` (Stage 2 cross within lookback)
- `'pullback_to_ma'` (Stage 3 pullback to ema_21)
- `'volume_above_ma25'` (Stage 3 volume gate — matches detect_a_v2 verbatim)

Plus the counterfactual (Stage 4). Total 5 strings on emission. Surfacing for chat-Claude review.

### Pre-Write Item 6 resolution

NOT applied at Step 2 (Step 2 has no OI consumption). Step 3 will:
1. Read `oi_history.iloc[...]['open_interest']` per architect resolution
2. Patch `detect_d_v2_proposal.md §6 + §9` to replace `open_interest_value` → `open_interest` in apex-planning, BEFORE the apex Step 3 commit ships

---

## Tests

`pytest tests/test_detect_d_v2.py -v` (24 tests):

```
collected 24 items

[Step 1 tests 1-12 — all PASSED, expect None]
test_no_indicators_returns_none PASSED                                    [  4%]
test_insufficient_5m_history_returns_none PASSED                          [  8%]
test_mixed_1h_trend_returns_none PASSED                                   [ 12%]
test_inverse_mixed_1h_trend_returns_none PASSED                           [ 16%]
test_long_no_recent_cross_returns_none PASSED                             [ 20%]
test_long_cross_too_old_returns_none PASSED                               [ 25%]
test_short_no_recent_cross_returns_none PASSED                            [ 29%]
test_long_no_pullback_returns_none PASSED                                 [ 33%]
test_long_pullback_close_below_ma_returns_none PASSED                     [ 37%]
test_short_no_pullback_returns_none PASSED                                [ 41%]
test_short_pullback_close_above_ma_returns_none PASSED                    [ 45%]
test_no_volume_returns_none PASSED                                        [ 50%]

[Step 2 Stage 5 stop placement — tests 13-16]
test_long_stop_at_sma_50_when_immediately_below PASSED                    [ 54%]
test_long_stop_at_ema_50_when_ema_50_nearer_below_close PASSED            [ 58%]
test_short_stop_at_sma_50_when_immediately_above PASSED                   [ 62%]
test_long_stop_falls_back_to_bb_middle_when_only_one_ma_below PASSED      [ 66%]

[Step 2 Stage 7 base emission — tests 17-20]
test_long_emits_signal_with_correct_fields PASSED                         [ 70%]
test_short_emits_signal_with_correct_fields PASSED                        [ 75%]
test_confidence_equals_base_no_boosts_at_step_2 PASSED                    [ 79%]
test_account_target_is_scalp PASSED                                       [ 83%]

[Step 2 Stage 4 counterfactual logging — tests 21-24]
test_counterfactual_logs_cross_age_when_present PASSED                    [ 87%]
test_counterfactual_logs_None_when_no_cross PASSED                        [ 91%]
test_counterfactual_does_not_affect_signal_emission PASSED                [ 95%]
test_counterfactual_uses_independent_walkback_from_live_substrate PASSED  [100%]

============================== 24 passed in 0.82s ==============================
```

Full suite (`pytest tests/`): **92 passed in 3.66s** (80 prior + 12 new at Step 2). No regressions in detect_a_v2, store_signals, Phase 1 v2 handlers.

---

## Open questions for chat-Claude

1. **Tests 14 + 16 reachability — semantic substitution applied.** Architect's literal test names (`_when_sma_50_above_close`, `_when_no_ma_below`) describe states that are structurally unreachable for valid long Stage 1+2+3 (transitivity: close > ema_21 > sma_50). I substituted reachable variants of the same logic with revised test names (`_when_ema_50_nearer_below_close`, `_when_only_one_ma_below`). Confirm if this substitution is acceptable; if not, alternative would be `xfail` markers + chat-Claude note for proposal v3 revision.

2. **Baseline confirming_indicators strings for detect_d_v2.** Picked: `['1h_trend_alignment', 'ma_cross_recent', 'pullback_to_ma', 'volume_above_ma25']`. The first and fourth match detect_a_v2 verbatim; the middle two are detect_d-specific. Confirm naming. Brief v13 may relabel canonically.

3. **Counterfactual string format.** Used `f'counterfactual_ema21_ema50_cross_age_bars:{value}'` where value is int or 'None'. Architect's "metadata as 'X': <int or None>" phrasing was dict-like; interpreted as colon-separated string per detect_a_v2's existing label convention. Confirm if format is fine, or if Signal dataclass should grow a separate `metadata: dict` field for counterfactual data structurally (proposal §6 used a different string format `f'ma_pair_alt_ema21_ema50_cross_within_n:{bool}'` — different shape entirely).

4. **Helper extraction reach.** `_find_cross_age_bars()` is module-private (single underscore) per detect_a_v2's `_identify_squeeze_start` precedent. Future detectors (b/c/e/f) may want to reuse. Currently file-local; lift to apex/signals/_helpers.py if cross-file sharing emerges. Flag for chat-Claude tracking, no Step 3 action needed.

5. **Pre-Write Item 6 OI column patch outstanding for Step 3.** Architect resolved: use `open_interest` (contracts), not `open_interest_value` (USD). Step 3 commit will patch the proposal §6 + §9 in apex-planning (separate commit BEFORE Step 3 ships to apex per architect's instruction). Tracking forward.

---

## Time/scope

- **LOC written this step:** 346 (production: 143 in setups.py; tests: 203)
- **Cumulative LOC for detect_d_v2 across Steps 1-2:** ~855 (production: 272; tests: 583)
- **Elapsed wall-clock for Step 2:** ~30 minutes from "open_interest resolution + go" message to commit `6b7de69` (includes proposal patch in apex-planning, code implementation, fixture revisions, 12 new tests, mid-write logic conflict surface + revision on tests 14/16, full regression).
- **Test runtime:** 0.82s (24 tests) / 3.66s (92 full)

---

## Next step

Awaiting chat-Claude diff review (commits `c19372a` + `6b7de69` on `origin/phase3-v2-detect-d`) and Step 3 prompt issuance. Step 3 scope per revised proposal §7:

> Stage 6 soft boosts (RSI, volume 2x, funding sign-aware, OI delta with cross-age cap) + apex-planning OI column patch. Tests 25-29 pass. Full regression: tests 1-29 + 80 outer = 97 tests, all green. Commit message: `phase3-v2 step 3: detect_d_v2 soft confidence boosts`.

Step 3 must also patch `detect_d_v2_proposal.md` `s/open_interest_value/open_interest/g` within §6 + §9 scopes BEFORE the apex code commit.
