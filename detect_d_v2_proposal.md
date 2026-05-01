# detect_d v2 Proposal — Final

**Reference:** Apex Bot Specification v12.2 Section 3 (Setup D) + Section 3.1 (Confirming-Indicator Framework)
**Date:** April 30, 2026
**Session:** 39 (Lucy V39)
**Status:** Architect-confirmed substrate, ready for Claude Code execution

---

## Architect-Confirmed Decisions

Locked from Session 39 review pass over Session 38 → V39 handoff substrate inventory + architect refinements:

1. **Symmetric long+short.** Long branch: 1h close > ema_21 AND sma_10 > sma_20 > sma_50, ema_21 crosses above sma_50, pullback `low <= ema_21 AND close > ema_21`. Short branch: 1h close < ema_21 AND sma_10 < sma_20 < sma_50, ema_21 crosses below sma_50, pullback `high >= ema_21 AND close < ema_21`. Both branches emit Signal with same shape, opposite direction. Setup D's literal Brief v12.2 line 88 reads long-only, but architect intent is symmetric — Apex setup library scales beyond architect's manual trading record sample (see Anti-Martingale Validation note below). Brief v13 absorbs the symmetry as canonical.
2. **MA pair (Gap 1 resolution).** Live trigger `ema_21 / sma_50` per slot 26 (fast EMA = directional bias, slow SMA = regime anchor). Parallel counterfactual `ema_21 / ema_50` cross computed and logged to `confirming_indicators` metadata for Phase 5/6 evaluation per slot 27 indicator-substrate validation discipline.
3. **Cross lookback window (Gap 2 resolution).** `CROSS_LOOKBACK_BARS_V2 = 20` — MA cross must have occurred within last 20 bars (100 min on 5m). Older cross = trend has matured past fresh-cross-plus-first-pullback archetype, return None.
4. **OI window cross-age cap.** `CROSS_AGE_CAP_BARS_V2 = 6` — OI boost only fires when cross is within last 6 bars (30 min). Beyond 6 bars, OI delta window from cross_bar → now stretches too far to be informative; boost skipped, no error. Aligned with detect_a_v2's `BB_SQUEEZE_MIN_BARS_V2 = 6` for cross-detector consistency in structural-event-anchored window conventions.
5. **OI delta interpretation, sign-aware uniform.** Both branches: positive OI delta from cross_bar to current_bar_close interpreted as "positions accumulated during the trending continuation = high-conviction directional bias." Long branch: longs accumulating, favorable for long thesis. Short branch: shorts accumulating, favorable for short thesis. Uniform-positive-favorable per detect_a_v2 decision 3 generalized to trending continuation.
6. **Stop placement (Gap 3 resolution).** Rank MAs from candidate set `[sma_50, ema_50, sma_100, sma_250]` on the correct side of close (below for longs, above for shorts). Stop = second-nearest MA from close ("next MA down" for longs per Brief Setup D wording, "next MA up" for shorts symmetric). If fewer than 2 MAs on the relevant side: fallback to 5m `bb_middle` per Rule 9 hierarchy. If stop ends up on wrong side of close (entry-stop inversion at MA convergence): return None, no signal.
7. **Soft boosts: 4 total, +0.05 each, max-achievable confidence 0.95** (Gap 7 resolution: liquidations dropped, RSI/volume/funding/OI retained):
    - **RSI direction-aligned:** long boosts on `rsi_14 > 50`, short boosts on `rsi_14 < 50`.
    - **Volume 2x MA25:** uniform — `volume >= 2 * volume_ma_25`.
    - **Funding sign-aligned:** long boosts on `funding_rate < -FUNDING_RATE_MAGNITUDE_THRESHOLD` (shorts paying longs = uncrowded longs); short boosts on `funding_rate > FUNDING_RATE_MAGNITUDE_THRESHOLD` (longs paying shorts = crowded longs).
    - **OI delta significant + within cross-age cap:** boost when `cross_age_bars ≤ 6` AND `oi_delta_pct > OI_DELTA_PERCENT_THRESHOLD = 1.0`.

8. **Routing and emission shape.** `tp=None`, `tp_strategy='structural_5m_bb'`, `account_target='scalp'`, `holding_period='scalp'`, `timeframe='5m'`. Matches detect_a_v2.
9. **Branch strategy.** `phase3-v2-detect-d` off main HEAD `20d78fe`. Per Session 38 architectural decision 3.4.

---

## Scope

**Ships in this proposal:**
- `detect_d_v2()` function in `apex/signals/setups.py` alongside existing v1 functions and `detect_a_v2()`
- Unit tests: 29 in `tests/test_detect_d_v2.py`

**Does NOT ship:**
- New Signal dataclass fields (`tp_strategy` already shipped by detect_a_v2 step 1)
- New storage layer changes (`signals` table already shipped)
- New indicator columns (MA mirror prep at commit 4ede661, merged at 20d78fe, added all required EMA/SMA columns)
- Engine.py / scheduler integration (separate session)
- Phase 4b position manager (TCH, trailing-stop logic)
- detect_f/b/c/e v2 — per-detector proposals follow Session 38 locked sequence d → f → b → c → e
- Retirement of v1 detect_a/f/b/c — happens after v2 setups validate in paper trading
- Brief v12.3/v13 edit absorbing detect_d symmetry as canonical — separate brief patch (anti-martingale: brief edits ship on genuine structural triggers, not embedded in detector proposals)

---

## File Modifications

`apex/signals/setups.py`:
- Adds detect_d v2 constants block at module level (alongside detect_a_v2 constants)
- Adds `detect_d_v2()` function
- Existing v1 + detect_a_v2 unchanged

`tests/test_detect_d_v2.py`: new file

---

## Constants

```python
# --- Setup D v2 Constants ---
CROSS_LOOKBACK_BARS_V2 = 20              # 5m bars to look back for MA cross occurrence (100 min)
CROSS_AGE_CAP_BARS_V2 = 6                # OI boost circuit breaker — aligned with BB_SQUEEZE_MIN_BARS_V2
LOOKBACK_V2 = 50                         # bars of history needed (carried from detect_a_v2)

# --- MA pair selection per slot 26/27 + Gap 1 resolution ---
# Live trigger: ema_21 (fast EMA, directional bias) over sma_50 (slow SMA, regime anchor)
# Counterfactual logged: ema_21 over ema_50 (architect's manual EMA-on-EMA experience as parallel substrate)

# --- Soft-boost thresholds (placeholders, recalibration deferred to post-Phase-6) ---
FUNDING_RATE_MAGNITUDE_THRESHOLD = 0.0001  # |funding_rate| above this = significant (carried from detect_a_v2)
OI_DELTA_PERCENT_THRESHOLD = 1.0           # OI change % from cross-bar above this = significant (carried)

# --- Confidence base + boost magnitudes ---
CONFIDENCE_BASE = 0.75                   # all hard gates passed
CONFIDENCE_BOOST = 0.05                  # uniform across all 4 soft boosts (one less than detect_a_v2; liquidations dropped)
# Max-achievable = 0.75 + 4 * 0.05 = 0.95 (NOT 1.0; detect_d has 4 boosts not 5)
```

**Threshold notes per v12.2 Section 3.1 threshold-validation discipline:**

`CROSS_LOOKBACK_BARS_V2 = 20` and `CROSS_AGE_CAP_BARS_V2 = 6` are placeholders, correct in shape (recent-cross window for trending continuation; tighter window for OI accumulation reading) but not validated in numerical value. Phase 6 paper trading produces empirical distributions; thresholds get retuned in Brief v13. The 6-bar cap aligns with detect_a_v2's `BB_SQUEEZE_MIN_BARS_V2` so cross-detector evaluation is like-for-like at 30-minute structural-event windows.

Funding and OI percent thresholds carried from detect_a_v2 (same numerical values, same reasoning).

---

## Pre-Fetched WebSocket Substrate Contract

detect_d_v2 expects caller to pre-fetch WebSocket-derived signals into the `indicators` dict under specific keys (analogous to detect_a_v2 contract, with two differences: `liquidations_5m_count` not required, OI history window anchored to cross-bar instead of squeeze-start).

**Required `indicators` dict keys:**

| Key | Type | Source | Purpose |
|-----|------|--------|---------|
| `'5m'` | `pd.DataFrame` | indicator tables | Existing v1 contract (must include `ema_21`, `sma_50`, `ema_50`, `sma_100`, `sma_250`, `bb_middle`, `volume_ma_25`, `rsi_14`, `volume`, `timestamp`) |
| `'1h'` | `pd.DataFrame` | indicator tables | Existing v1 contract (must include `ema_21`, `sma_10`, `sma_20`, `sma_50`) |
| `'funding_rate_latest'` | `float` | `funding_rate_ws` query | Soft boost: funding sign + magnitude |
| `'oi_history'` | `pd.DataFrame` | `open_interest_ws` last LOOKBACK_V2 rows | Soft boost: OI delta cross-aligned |

If any required key is missing, detect_d_v2 returns None with an explicit log line naming the missing key (matches detect_a_v2 substrate-gap-visibility pattern).

`liquidations_5m_count` is NOT required (Gap 7 resolution: liquidations boost dropped — squeeze-specific shape, not informative for trending-continuation entries).

---

## Function Signature

```python
def detect_d_v2(
    candles_5m: pd.DataFrame,
    candles_15m: pd.DataFrame,
    candles_1h: pd.DataFrame,
    candles_4h: pd.DataFrame,
    indicators: dict[str, pd.DataFrame | float],
) -> Signal | None:
```

Same parameter shape as detect_a_v2. `indicators` value-type set: DataFrame and float only.

---

## Function Structure

Seven stages. Each stage returns `None` early on failure. Only signals reaching Stage 7 emit.

### Stage 1: Direction determination (1h trend regime)

```
1. Verify required indicators DataFrames exist with sufficient history (>= LOOKBACK_V2 bars on 5m, >= 1 bar on 1h)
2. Check 1h trend regime:
   - Bullish: 1h close > ema_21 AND sma_10 > sma_20 > sma_50  → direction = 'long'
   - Bearish: 1h close < ema_21 AND sma_10 < sma_20 < sma_50  → direction = 'short'
   - Neither (mixed/conflicting): return None
```

`direction` locked from this stage forward. All subsequent gates and boosts are direction-aware.

### Stage 2: Cross detection + cross_bar identification

For the determined direction, find the bar where ema_21 crossed sma_50 in the favored direction within the last `CROSS_LOOKBACK_BARS_V2` bars.

```
For long direction:
    Currently: ema_21 > sma_50
    Find latest N in [1, 20] where ema_21[N bars ago] <= sma_50[N bars ago]
    If no such N exists: return None (no recent cross)
    cross_bar_idx = current_idx - N
    cross_age_bars = N

For short direction:
    Currently: ema_21 < sma_50
    Find latest N in [1, 20] where ema_21[N bars ago] >= sma_50[N bars ago]
    If no such N exists: return None
    cross_bar_idx = current_idx - N
    cross_age_bars = N
```

`cross_bar_timestamp = int(ind_5m.iloc[cross_bar_idx]['timestamp'])` for downstream OI lookup.

### Stage 3: Pullback + volume hard gates

Direction-aware pullback check on current bar:

```
For long direction:
    low <= ema_21 (wick into MA support)
    AND close > ema_21 (close above = bounce confirmed)

For short direction:
    high >= ema_21 (wick into MA resistance)
    AND close < ema_21 (close below = rejection confirmed)
```

Volume confirmation (Brief Section 3 hard gate, no exceptions): `volume > volume_ma_25`.

If any check fails: return None.

### Stage 4: Counterfactual cross detection (parallel substrate per slot 27)

Compute the parallel `ema_21 / ema_50` cross within the same N=20 lookback window, direction-matched:

```
For long direction:
    ema21_over_ema50_cross_within_n = (ema_21 > ema_50 currently) AND
                                       (exists N in [1, 20] where ema_21[N ago] <= ema_50[N ago])
For short direction:
    ema21_under_ema50_cross_within_n = (ema_21 < ema_50 currently) AND
                                        (exists N in [1, 20] where ema_21[N ago] >= ema_50[N ago])
```

Captured into `confirming_indicators` metadata as `f'ma_pair_alt_ema21_ema50_cross_within_n:{bool}'`. Does NOT affect emission decision (live trigger is ema_21/sma_50 from Stage 2).

### Stage 5: Stop placement (MA-stack ranking with bb_middle fallback)

Direction-aware MA ranking from candidate set `[sma_50, ema_50, sma_100, sma_250]` (using current 5m bar values):

```
For long direction:
    ma_candidates = sorted([m for m in [sma_50, ema_50, sma_100, sma_250] if m < close], reverse=True)
    # descending: nearest below close first
    if len(ma_candidates) >= 2:
        stop = ma_candidates[1]   # next MA down
    else:
        stop = curr['bb_middle']  # Rule 9 fallback
    if stop >= close:
        return None  # MA convergence pathological case

For short direction:
    ma_candidates = sorted([m for m in [sma_50, ema_50, sma_100, sma_250] if m > close])
    # ascending: nearest above close first
    if len(ma_candidates) >= 2:
        stop = ma_candidates[1]   # next MA up
    else:
        stop = curr['bb_middle']  # Rule 9 fallback
    if stop <= close:
        return None  # MA convergence pathological case
```

Rule 9 hierarchy preserved: structural MA-stack level preferred when present, middle BB next, never tighter.

### Stage 6: Soft-boost computation

Four additive boosts at +0.05 each, evaluated independently. Confidence accumulates from CONFIDENCE_BASE (0.75) and caps at 1.0 via `min(confidence, 1.0)` (defensive code matching detect_a_v2's pattern; max-achievable is 0.95 with 4 boosts).

**Boost 1: RSI direction-aligned**
- Long: `curr['rsi_14'] > 50` → +0.05, append `'rsi_confirms_long'`
- Short: `curr['rsi_14'] < 50` → +0.05, append `'rsi_confirms_short'`

**Boost 2: Volume 2x MA25 (uniform across both directions)**
- `volume >= 2 * curr['volume_ma_25']` → +0.05, append `'volume_2x_ma25'`

**Boost 3: Funding sign-aligned**
- Long: `funding_rate < -FUNDING_RATE_MAGNITUDE_THRESHOLD` → +0.05, append `'funding_favorable_long'`
- Short: `funding_rate > FUNDING_RATE_MAGNITUDE_THRESHOLD` → +0.05, append `'funding_favorable_short'`

**Boost 4: OI delta cross-aligned, with cross-age cap circuit breaker**
- Skip boost entirely if `cross_age_bars > CROSS_AGE_CAP_BARS_V2` (cross too old for OI window to be informative)
- Read `indicators['oi_history']` (DataFrame with `ts` and `open_interest` columns)
- Find latest row with `ts <= breakout_candle_close_ts` → `oi_now`
- Find latest row with `ts <= cross_bar_timestamp` → `oi_at_cross`
- If either query returns no row: skip boost (substrate gap, no boost applied, no error)
- Compute `oi_delta_pct = (oi_now - oi_at_cross) / oi_at_cross * 100`
- If `oi_delta_pct > OI_DELTA_PERCENT_THRESHOLD` (1.0) → +0.05, append `'oi_delta_significant'`
- Sign interpretation uniform: positive OI delta = positions accumulated during continuation = favorable for the directional thesis (long or short).

**Counterfactual logging (always, regardless of boost outcome):**
- Append `f'ma_pair_alt_ema21_ema50_cross_within_n:{counterfactual_bool}'` to `confirming_indicators` (from Stage 4)

### Stage 7: Signal emission

```python
return Signal(
    setup_type='D',
    direction=direction,                 # 'long' or 'short' from Stage 1
    account_target='scalp',              # v12.2 Section 7 routing rule (5m entry → scalp)
    holding_period='scalp',
    entry=close,
    stop=stop,                           # from Stage 5
    tp=None,                             # dynamic — Phase 4b consumes tp_strategy
    tp_strategy='structural_5m_bb',      # matches detect_a_v2
    confidence=min(confidence, 1.0),
    confirming_indicators=confirming,
    timeframe='5m',
    bar_time=bar_time,
)
```

Caller (engine.py, future) receives Signal and is responsible for calling `store.insert_signal(signal)`.

---

## Test Plan

### `tests/test_detect_d_v2.py` (29 tests)

**Stage 1 — Direction determination (return None):**
1. `test_no_indicators_returns_none` — `indicators={}`
2. `test_insufficient_5m_history_returns_none` — len(ind_5m) < LOOKBACK_V2
3. `test_mixed_1h_trend_returns_none` — 1h close > ema_21 but sma stack not bullish (regime conflict)
4. `test_inverse_mixed_1h_trend_returns_none` — 1h close < ema_21 but sma stack not bearish

**Stage 2 — Cross detection (return None):**
5. `test_long_no_recent_cross_returns_none` — bullish 1h, ema_21 always above sma_50 throughout lookback
6. `test_long_cross_too_old_returns_none` — bullish 1h, cross occurred 21 bars ago (outside N=20 window)
7. `test_short_no_recent_cross_returns_none` — bearish 1h, ema_21 always below sma_50 throughout lookback

**Stage 3 — Pullback + volume gates (return None):**
8. `test_long_no_pullback_returns_none` — long path all gates passed except low > ema_21
9. `test_long_pullback_close_below_ma_returns_none` — low <= ema_21 but close <= ema_21 (no bounce)
10. `test_short_no_pullback_returns_none` — short path, high < ema_21
11. `test_short_pullback_close_above_ma_returns_none` — high >= ema_21 but close >= ema_21 (no rejection)
12. `test_no_volume_returns_none` — volume <= volume_ma_25 (uniform across directions)

**Stage 7 — Hard gate pass + emission shape:**
13. `test_long_pullback_emits_signal` — full long path, base confidence 0.75
14. `test_short_pullback_emits_signal` — full short path, base confidence 0.75
15. `test_signal_routes_to_scalp_account` — verify `account_target == 'scalp'` (both directions)
16. `test_signal_emits_tp_strategy_marker` — verify `tp is None` and `tp_strategy == 'structural_5m_bb'`
17. `test_signal_setup_type_is_D` — verify `setup_type == 'D'`

**Stage 5 — Stop placement:**
18. `test_long_stop_below_next_ma_down_when_two_or_more_below` — close above sma_50/ema_50/sma_100, stop = ema_50 (second-nearest below)
19. `test_long_stop_falls_back_to_bb_middle_when_only_one_ma_below` — close just above sma_250, stop = bb_middle
20. `test_long_stop_falls_back_to_bb_middle_when_close_below_all_mas` — close < sma_250, stop = bb_middle
21. `test_long_returns_none_when_stop_geq_entry` — pathological MA convergence (long side)
22. `test_short_stop_above_next_ma_up_when_two_or_more_above` — close below sma_50/ema_50/sma_100, stop = ema_50 (second-nearest above)
23. `test_short_returns_none_when_stop_leq_entry` — pathological MA convergence (short side)

**Stage 6 — Soft boosts:**
24. `test_rsi_direction_aligned_boosts_confidence` — long RSI > 50 → 0.80; short RSI < 50 → 0.80 (parametrized over direction)
25. `test_volume_2x_boosts_confidence` — volume >= 2 * MA25 → 0.80
26. `test_funding_sign_aligned_boosts_confidence` — long funding < -threshold → 0.80; short funding > threshold → 0.80 (parametrized over direction)
27. `test_oi_delta_cross_aligned_boosts_when_within_cap` — cross_age <= 6 AND oi_delta_pct > threshold → 0.80
28. `test_oi_delta_skipped_when_cross_age_exceeds_cap` — cross_age = 7 bars (within N=20 lookback but outside cap=6), OI boost NOT applied even if delta > threshold; confidence stays at 0.75 with no other boosts active. Load-bearing test: validates the cross-age-cap circuit breaker (Gap 7 resolution).
29. `test_all_boosts_active_caps_at_0_95` — all 4 boosts add to 0.75 + 0.20 = 0.95 (max-achievable); verifies no over-add

**Counterfactual logging validation:**
- Validation of `'ma_pair_alt_ema21_ema50_cross_within_n:True'` and `:False` cases is woven into tests 13, 14, 18, 22 by asserting the metadata string is present in `confirming_indicators` regardless of live-trigger outcome. No standalone test (avoids double-counting).

### Test Fixtures

Module-level fixtures in `tests/test_detect_d_v2.py`:

```python
@pytest.fixture
def synthetic_long_pullback_indicators():
    """Build minimal indicators DataFrames + WebSocket data dict
    that triggers a clean long-pullback signal at base confidence 0.75.

    Setup:
    - ind_5m: 50 bars. ema_21 crossed above sma_50 at bar -3 (within N=20 AND within cap=6).
              Current bar: low touches ema_21, close > ema_21 by small margin.
              Volume = volume_ma_25 + 1 (passes hard gate, no 2x boost).
              RSI = 50 (no boost).
              All MAs (sma_50, ema_50, sma_100, sma_250) below close in distinct levels.
    - ind_1h: bullish stack (close > ema_21, sma_10 > sma_20 > sma_50).
    - candles_5m, candles_1h, candles_15m, candles_4h: minimal valid shapes.
    - funding_rate_latest: 0 (no boost).
    - oi_history: flat (no delta, no boost).
    """

@pytest.fixture
def synthetic_short_pullback_indicators():
    """Inverse fixture: bearish 1h stack, ema_21 crossed below sma_50 at bar -3,
    current bar high >= ema_21 AND close < ema_21, all MAs above close.
    """

@pytest.fixture
def synthetic_long_pullback_old_cross_indicators():
    """Variant for test 28: cross occurred at bar -7 (within lookback N=20 but
    outside cap=6). OI history shows clear delta > threshold. Used to assert the
    OI boost is NOT applied because the cross-age cap fires first.
    """
```

Each soft-boost test uses the appropriate base fixture and modifies the single relevant input. Each hard-gate test uses the base fixture and breaks the single relevant gate.

---

## Sequencing for Claude Code

**Sequencing revision (Session 40):** original 5-commit sequence (Steps 1-5) consolidated to 3-commit sequence after Session 40 architect decision. Rationale: Stages 5 (stop placement), 7 (base emission), and 4 (counterfactual logging) are tightly-coupled bolt-ons to the hard-gate skeleton with no independent failure-mode worth a separate review cycle — they all execute in the all-hard-gates-pass path, share locals, and have no internal interaction surface that would benefit from sequential review. Stage 6 (soft confidence boosts) is the second genuinely-separable risk surface — its 4 boost computations have independent threshold logic and substrate-key dependencies, deserving a distinct review pass. Commit count reduces from 5 to 3 without losing review coverage; per-test scope unchanged at 29 total.

Three commits on branch `phase3-v2-detect-d` (off main HEAD `20d78fe`). Each commit independently passes its scoped tests. Architect reviews diff per commit per Brief v12.3 Section 10 standing instruction "Surface deviations, do not absorb them."

**Step 1 — Constants + Stage 1+2+3 hard gates skeleton.** SHIPPED at `c19372a`.
- File: `apex/signals/setups.py`
- Change: add detect_d v2 constants block + `detect_d_v2()` with Stage 1 (direction determination), Stage 2 (cross detection + cross_bar_idx + cross_age_bars), Stage 3 (pullback + volume gates). Returns None on every path (no Signal emission yet).
- Verify: tests 1-12 pass (all hard gate tests expect None across both branches).
- Commit: `phase3-v2 step 1: detect_d_v2 hard gates skeleton (long+short)`

**Step 2 — Functional core (Stages 5+7+4 consolidated).**
- File: `apex/signals/setups.py`
- Change: add Stage 5 (direction-aware MA-stack ranking with `len < 2` fallback to bb_middle, pathological-case None on stop-entry inversion), Stage 4 (parallel ema_21/ema_50 cross via shared walk-back helper, logged as `'counterfactual_ema21_ema50_cross_age_bars:<int|None>'` in confirming_indicators), Stage 7 (Signal emission with `setup_type='D'`, `tp=None`, `tp_strategy='structural_5m_bb'`, `account_target='scalp'`, `confidence=CONFIDENCE_BASE`, no boosts).
- Verify: tests 13-24 pass (12 new tests — 4 stop-placement + 4 base-emission + 4 counterfactual-logging).
- Commit: `phase3-v2 step 2: detect_d_v2 functional core (stop + emission + counterfactual)`

**Step 3 — Stage 6 soft boosts (RSI, volume 2x, funding sign-aware, OI delta with cross-age cap) + OI column patch.**
- File: `apex/signals/setups.py`
- Change: add four boost computations consuming `indicators` dict keys, direction-aware where applicable. OI boost includes the cross-age-cap circuit breaker per Gap 7 resolution. Reads `oi_history.iloc[...]['open_interest']` (contracts) per Session 40 Pre-Write Item 6 architect resolution.
- Also patches this proposal's §6 Stage 6 Boost 4 + §9 Pre-Write Verification Item 6: `s/open_interest_value/open_interest/g` within those two scopes only. Patches commit to apex-planning in a SEPARATE commit BEFORE Step 3 ships to apex.
- Verify: tests 25-29 pass. Full regression: tests 1-29 all pass on single run. Outer suite: 80 (existing post-Step 1) + 17 (Step 2 + Step 3 new) = 97 tests, all green.
- Commit: `phase3-v2 step 3: detect_d_v2 soft confidence boosts`

After Step 3: branch ready for chat-Claude diff review across all 3 commits, then merge to main.

---

## Pre-Write Verification Checklist for Claude Code

Per Brief v12.2 Section 10 standing instruction "Re-read before deciding," Claude Code must verify these assumptions before any code change:

1. `apex/signals/setups.py` — confirm detect_a_v2 constants are at module level (not nested in function) so detect_d_v2 constants can sit alongside without import gymnastics.
2. `apex/signals/setups.py` — confirm v2 import chain: `from apex.signals.models import Signal` (with `tp: Optional[float]` and `tp_strategy: Optional[str]` fields from detect_a_v2 step 1).
3. Indicator tables post-merge (commit 20d78fe) — confirm 5m table has columns: `ema_21`, `sma_50`, `ema_50`, `sma_100`, `sma_250`, `bb_middle`, `volume_ma_25`, `rsi_14`, `volume`, `timestamp`. (MA mirror prep at 4ede661 added ema_50, ema_100, ema_250, sma_21 to 5m; sma_50/sma_100/sma_250 from prior schema.)
4. Indicator tables — confirm 1h table has columns: `ema_21`, `sma_10`, `sma_20`, `sma_50` (existing pre-merge).
5. `tests/` — confirm pytest fixture conventions used by detect_a_v2 tests; reuse same patterns for detect_d_v2 tests.
6. `oi_history` DataFrame contract — confirm `ts` and `open_interest` are the exact column names produced by `open_interest_ws` query helper. If column names differ, surface deviation; do not silently rename.

   **Resolution (Session 40):** Use `open_interest` (contracts) per cross-detector consistency with detect_a_v2 and substrate decision 5 intent. `open_interest_value` (USD) was a V39 drafting error — USD-OI conflates accumulation with price drift in trend direction.

Surface any deviation found. Do not absorb silently.

---

## Anti-Martingale Validation

- Brief v12.2 ships first ✓ (shipped session 35)
- detect_d v2 proposal references v12.2 canonical brief ✓
- Proposal scope contained: detect_d only, not detect_b/c/e/f/g ✓
- Phase 4b TCH framework referenced but explicitly out of scope ✓
- Storage layer reuses detect_a_v2's signals table (no schema changes) ✓
- Implementation deferred to Claude Code session (proposal does not ship code) ✓
- Five small commits with per-commit test passes ✓
- Substrate inventory all resolved per V38 handoff Section 5 + architect refinements (cross-age cap aligned to detect_a_v2's BB_SQUEEZE_MIN_BARS_V2, symmetric long+short) ✓
- Slot 26 + 27 disciplines applied (live trigger ema_21/sma_50, counterfactual ema_21/ema_50 logged) ✓
- Symmetric long+short rationale: Apex setup library scales beyond architect's 6-day manual trading record. Validation source for setup branches is Phase 5 backtest + Phase 6 paper trading, not architect's manual record. Excluding setup branches because they were absent from a 6-day sample is the wrong test ✓
- Brief v13 absorption queued: detect_d symmetry, cross-age cap convention, OI window anchoring rules ✓

---

## Open Questions for Architect Review

None at proposal time. All substrate decisions confirmed in Session 39 review pass.

If architect review surfaces any deviation from substrate inventory or proposal structure, surface here for v2 of this proposal before Claude Code execution.

---

End of proposal. Ready for Claude Code execution on branch `phase3-v2-detect-d`.
