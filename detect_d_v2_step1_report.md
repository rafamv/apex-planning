# detect_d_v2 Step 1 Ship Report — Session 40

**Branch:** `phase3-v2-detect-d`
**Step:** 1 of 5 (per `detect_d_v2_proposal.md` §10 Sequencing)
**Date:** 2026-05-01

---

## Status

Step 1 shipped + tests green. 12/12 Step 1 tests pass; 80/80 full suite pass (68 prior + 12 new). Awaiting chat-Claude diff review for Step 2 go-ahead per architect approval rhythm.

---

## Commits

Branch: `phase3-v2-detect-d`. Commits since `c34178a` (Brief v12.3):

```
c19372a phase3-v2 step 1: detect_d_v2 hard gates skeleton (long+short)
```

(One commit. Pre-Write findings landed in apex-planning at `aaa822c`, separate repo.)

---

## Files touched

| Path | Lines (+/-) | Description |
|------|-------------|-------------|
| `<REPO_ROOT>apex/signals/setups.py` | +129 / -1 | Added Setup D v2 constants block (CROSS_LOOKBACK_BARS_V2=20, CROSS_AGE_CAP_BARS_V2=6) and `detect_d_v2()` function with Stage 1 (1h trend regime → direction lock), Stage 2 (ema_21/sma_50 cross detection within lookback), Stage 3 (direction-aware pullback to ema_21 + volume confirmation). Returns None on every path including all-gates-pass. |
| `<REPO_ROOT>tests/test_detect_d_v2.py` | +381 / -0 | New file. 12 hard-gate tests (4 Stage 1 + 3 Stage 2 + 5 Stage 3) using two `synthetic_long/short_pullback_indicators` fixtures + 3 module-level builder helpers (`_build_ind_5m_d`, `_build_ind_1h_d`, `_build_candles`) + 2 ema_21 series builders (`_ema21_series_with_long_cross`, `_ema21_series_with_short_cross`) for cross-encoding scenarios. |

Total: +510 / -1.

---

## Implementation notes

### Constants block (proposal §4)

Added two new constants alongside detect_a_v2's. Documented in inline comment which constants detect_d_v2 reuses from detect_a_v2 (LOOKBACK_V2, FUNDING_RATE_MAGNITUDE_THRESHOLD, OI_DELTA_PERCENT_THRESHOLD, CONFIDENCE_BASE, CONFIDENCE_BOOST) and noted max-achievable confidence = 0.95 (4 boosts × 0.05 + base 0.75) — distinct from detect_a_v2's max 1.0 (5 boosts).

### Stage 1 — Direction determination

Mirrored detect_a_v2's substrate-presence + history-length pattern verbatim:
- `indicators.get('5m')` / `indicators.get('1h')` with `logger.info` on missing key
- `len(ind_5m) < LOOKBACK_V2` and `len(ind_1h) < 1` (proposal §6 says ">= 1 bar on 1h")
- `len(candles_5m) == 0 or len(candles_1h) == 0` defensive

Direction lock via two boolean expressions (`bullish` and `bearish`) for clarity:
```python
bullish = h_close > h1['ema_21'] and h1['sma_10'] > h1['sma_20'] > h1['sma_50']
bearish = h_close < h1['ema_21'] and h1['sma_10'] < h1['sma_20'] < h1['sma_50']
```
Direction = 'long' / 'short' / None. Mixed-regime (both False) returns None.

NaN defensive check on 1h trend MAs (mirrors detect_a_v2 detect_a Stage 1.5 pattern).

### Stage 2 — Cross detection

Implemented per proposal §6 verbatim with one structural choice surfaced here:

**Precondition check explicit in code** — proposal phrases "Currently: ema_21 > sma_50" as a precondition; I encoded it as an explicit gate:
```python
if direction == 'long' and not (curr_ema21 > curr_sma50):
    return None
if direction == 'short' and not (curr_ema21 < curr_sma50):
    return None
```
Rationale: without this gate, the search loop could find a cross-bar in pathological scenarios where direction was determined by 1h regime but 5m ema_21/sma_50 are momentarily the wrong way. The proposal implies this check; explicit code makes it provable.

**Walk-back loop** scans `n in [1, CROSS_LOOKBACK_BARS_V2]`, indexes `prior_idx = len(ind_5m) - 1 - n`, breaks on first n where prior bar was on the opposite side (long: `ema_21[prior] <= sma_50[prior]`; short: `ema_21[prior] >= sma_50[prior]`). NaN bars skipped via `continue`. cross_bar_idx + cross_age_bars set on first match; loop break.

`cross_bar_idx is None` → return None (no cross within lookback).

**Dead-store noqa for Step 1:** `cross_bar_timestamp` and `cross_age_bars` are computed but not consumed at Step 1. Step 5 Boost 4 (OI delta cross-aligned + circuit breaker) consumes both. Marked with `# noqa: F841` and inline comment naming Step 5 as consumer. Same pattern detect_a_v2 used for `squeeze_start_timestamp` between Steps 5 and 6.

### Stage 3 — Pullback + volume hard gates

Direction-aware pullback gate exactly per proposal §6:
```python
if direction == 'long':
    if not (low <= curr_ema21 and close > curr_ema21):
        return None
else:  # short
    if not (high >= curr_ema21 and close < curr_ema21):
        return None
```

Volume confirmation reads `candle['volume']` from `candles_5m.iloc[-1]`, NOT from `ind_5m` — correcting the proposal §7 documentation note (proposal lists `volume` as a 5m IND column but it actually lives in candles). detect_a_v2 follows same pattern; Pre-Write Item 3 surfaced this as documentation issue, no code-blocking impact.

### Pre-Write Item 6 resolution incorporation

NOT incorporated in Step 1 (Step 1 has no OI consumption). Step 5 will use `oi_history.iloc[...]['open_interest']` per architect resolution. Flagging here so chat-Claude tracks it as outstanding for Step 5 commit.

### Helper functions extracted

In test file only (no production helpers extracted in Step 1 — the cross-detection walk-back is inline rather than extracted as `_identify_cross_start()` because it's small, single-purpose, and direction-branching makes a pure helper less clean than detect_a_v2's `_identify_squeeze_start`. May re-evaluate at Step 5 if the cross_bar logic grows; for now inline.)

Test file helpers:
- `_ts_series(n, start_ms, step_ms)` — timestamp generator
- `_build_ind_5m_d(...)` — accepts ema_21/sma_50 as float OR list for cross-encoding
- `_build_ind_1h_d(bullish=...)` — bullish/bearish stack toggle
- `_build_candles(...)` — OHLCV with explicit high/low override
- `_ema21_series_with_long_cross(n, cross_bar_offset)` — cross encoding for long
- `_ema21_series_with_short_cross(n, cross_bar_offset)` — cross encoding for short

### Edge cases handled beyond proposal spec

1. **NaN guard in Stage 2 cross search loop** (`if pd.isna(prior_ema21) or pd.isna(prior_sma50): continue`). Proposal doesn't mention NaN handling for the walk-back. Bars early in indicator history can have NaN MAs (warmup). Defensive `continue` skips them rather than treating NaN as a comparison value (which would silently match `<=` due to NaN comparison semantics returning False).

2. **`prior_idx < 0` break in walk-back loop.** If `len(ind_5m) - 1 - n < 0`, break. With `LOOKBACK_V2 = 50` and `CROSS_LOOKBACK_BARS_V2 = 20`, this branch is unreachable in normal use (50 - 1 - 20 = 29 > 0), but the guard makes the loop safe if someone shrinks LOOKBACK_V2 in the future.

3. **`len(ind_1h) < 1`** — proposal says ">= 1 bar on 1h", I encoded as `< 1`. Strictly correct per spec; arguably should be `< 2` to allow `iloc[-2]` future access, but proposal says 1 so I followed literally.

---

## Tests

`pytest tests/test_detect_d_v2.py -v`:

```
collected 12 items

tests/test_detect_d_v2.py::test_no_indicators_returns_none PASSED        [  8%]
tests/test_detect_d_v2.py::test_insufficient_5m_history_returns_none PASSED [ 16%]
tests/test_detect_d_v2.py::test_mixed_1h_trend_returns_none PASSED       [ 25%]
tests/test_detect_d_v2.py::test_inverse_mixed_1h_trend_returns_none PASSED [ 33%]
tests/test_detect_d_v2.py::test_long_no_recent_cross_returns_none PASSED [ 41%]
tests/test_detect_d_v2.py::test_long_cross_too_old_returns_none PASSED   [ 50%]
tests/test_detect_d_v2.py::test_short_no_recent_cross_returns_none PASSED [ 58%]
tests/test_detect_d_v2.py::test_long_no_pullback_returns_none PASSED     [ 66%]
tests/test_detect_d_v2.py::test_long_pullback_close_below_ma_returns_none PASSED [ 75%]
tests/test_detect_d_v2.py::test_short_no_pullback_returns_none PASSED    [ 83%]
tests/test_detect_d_v2.py::test_short_pullback_close_above_ma_returns_none PASSED [ 91%]
tests/test_detect_d_v2.py::test_no_volume_returns_none PASSED            [100%]

============================== 12 passed in 0.80s ==============================
```

Full suite (`pytest tests/`): **80 passed in 3.09s** (68 prior + 12 new). No regressions.

---

## Open questions for chat-Claude

1. **`cross_age_bars` dead-store at Step 1.** Currently `# noqa: F841` with inline comment naming Step 5 Boost 4 as consumer. Same pattern as detect_a_v2 between Steps 5 and 6. Flagging for chat-Claude awareness — if you'd prefer the variable not appear at Step 1 (delay computation to Step 5), say so before Step 2.

2. **No `_identify_cross_bar()` helper extracted.** detect_a_v2 has `_identify_squeeze_start()` as a pure helper. detect_d_v2's cross detection is inline (proposal §10 Step 1 doesn't request a helper, and direction branching makes a pure helper less clean — would need direction param). If chat-Claude prefers a helper for symmetry with detect_a_v2's structure (testability + reuse for future detectors), name it before Step 5 lands so refactor happens in Step 5 alongside OI Boost 4 consumption.

3. **Pre-Write Item 6 resolution outstanding for Step 5.** Architect resolved: use `open_interest` (contracts) not `open_interest_value`. Step 5 will read `oi_history.iloc[...]['open_interest']` and the Step 5 commit will also patch the proposal §6 + §9 in apex-planning per architect's explicit instruction. No action needed before Step 2 — flagging so chat-Claude tracks across Step boundaries.

4. **`len(ind_1h) < 1` vs `< 2` history check.** I followed proposal §6 literal (`>= 1`). detect_a_v2 used `< 2` because it accesses `iloc[-1]` (1 row needed) and the proposal there said `>= 2` defensively. detect_d_v2 also accesses only `iloc[-1]`, so `< 1` is correct for current use. If a future stage wants `iloc[-2]` (e.g., 1h-trend-direction-change detection), this becomes too loose. Flagging as a low-priority spec question.

---

## Time/scope

- **LOC written:** 510 (production: 129 in setups.py; tests: 381)
- **Elapsed wall-clock for Step 1:** ~25 minutes from "open_interest resolution" message to commit `c19372a` (includes implementation + 12-test fixture design + verification). Pre-Write checklist + findings push earlier in session.
- **Test runtime:** 0.80s (12 tests) / 3.09s (80 full)

---

## Next step

Awaiting chat-Claude diff review (commit `c19372a` on `origin/phase3-v2-detect-d`) and Step 2 prompt issuance. Step 2 scope per proposal §10:

> Stage 7 base emission with fixed `bb_middle` stop, no boosts. Tests 13-17 pass. Commit message: `phase3-v2 step 2: detect_d_v2 base signal emission (long+short)`.
