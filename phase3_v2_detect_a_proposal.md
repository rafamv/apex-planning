# detect_a v2 Proposal — Final

**Reference:** Apex Bot Specification v12.2 Section 3 (Setup A) + Section 3.1 (Confirming-Indicator Framework)
**Date:** April 29, 2026
**Session:** 35 (Lucy V35)
**Status:** Approved by architect, ready for Claude Code execution

---

## Architect-Approved Decisions

Locked from session 35 review pass:

1. **Routing:** detect_a_v2 = pure function over DataFrames + indicators dict. Caller (engine.py) pre-fetches WebSocket-derived signals and passes as additional `indicators` dict keys. Caller owns Signal storage.
2. **Funding sign interpretation:** long breakouts favor negative funding (shorts paying longs = uncrowded longs); short breakouts favor positive funding (longs paying shorts = crowded longs).
3. **OI delta direction:** uniform positive-favorable for both long and short breakouts (positions accumulated during squeeze indicate high-conviction breakout regardless of direction).
4. **TP placement:** dynamic structural — Signal dataclass extended with `tp_strategy: Optional[str]` and `tp: Optional[float]`. detect_a_v2 emits `tp=None`, `tp_strategy='structural_5m_bb'`. Phase 4b position manager (queued, not in scope here) consumes `tp_strategy` to execute trailing-TP logic per V33 Part 22.3.1 TCH framework.
5. **Soft-boost magnitude:** uniform +0.05 across all five boosts. Recalibration deferred to post-Phase-6 empirical data.
6. **Test coverage:** 25 tests across 2 files, scope as listed.
7. **Branch strategy:** `phase3-v2-detect-a` off main, merge after Claude Code-side test pass + chat-Claude diff review.

---

## Scope

**Ships in this proposal:**
- `Signal` dataclass extension in `apex/signals/models.py`: `tp` becomes `Optional[float]`, new field `tp_strategy: Optional[str] = None`
- `detect_a_v2()` function in `apex/signals/setups.py` alongside existing `detect_a()`
- New `signals` table in `apex/data/store.py` with minimal schema
- New `insert_signal()` method on `Store`
- Unit tests: 20 in `tests/test_detect_a_v2.py`, 5 in `tests/test_store_signals.py`

**Does NOT ship:**
- Engine.py / scheduler integration (separate session — engine consumes detect_a_v2 output and calls insert_signal)
- 50-signal manual review tooling (gated on storage existing — this proposal ships the storage)
- Outcome tracking columns on `signals` table (filled, exit_price, pnl) — added when Phase 4b position manager exists
- Phase 4b position manager (TCH, trailing-stop logic) — separate phase entirely
- detect_b_v2 through detect_g_v2 — separate proposals per setup
- Retirement of v1 detect_a — happens after v2 validates in paper trading

---

## File Modifications

`apex/signals/models.py`:
- Existing `Signal` dataclass field `tp: float` → `tp: Optional[float]`
- New field added: `tp_strategy: Optional[str] = None`
- Default value preserves backward compatibility — existing v1 setups (detect_a, detect_f, detect_b) construct Signal without `tp_strategy` and continue to work.

`apex/signals/setups.py`:
- Adds v2 constants block at module level
- Adds `detect_a_v2()` function
- Existing v1 functions unchanged

`apex/data/store.py`:
- Adds `signals` entry to `_MARKET_DATA_TABLES`
- Adds `idx_signals_bar_time` index in `init_db()`
- Adds `insert_signal(signal)` method

`tests/test_detect_a_v2.py`: new file
`tests/test_store_signals.py`: new file

---

## Constants

```python
# --- Setup A v2 Constants ---
BB_SQUEEZE_THRESHOLD_V2 = 1.5            # bb_width % below this = squeeze (carried from v1)
BB_SQUEEZE_MIN_BARS_V2 = 6               # consecutive bars in squeeze before valid (carried from v1)
LOOKBACK_V2 = 50                         # bars of history needed (carried from v1)

# --- Soft-boost thresholds (placeholders, recalibration deferred to post-Phase-6) ---
LIQUIDATIONS_EVENT_RATE_THRESHOLD = 5    # events in 5m window above this = elevated
FUNDING_RATE_MAGNITUDE_THRESHOLD = 0.0001  # |funding_rate| above this = significant
OI_DELTA_PERCENT_THRESHOLD = 1.0         # OI change % from squeeze-start above this = significant

# --- Confidence base + boost magnitudes ---
CONFIDENCE_BASE = 0.75                   # all hard gates passed
CONFIDENCE_BOOST = 0.05                  # uniform across all 5 soft boosts
```

**Threshold notes per v12.2 Section 3.1 threshold-validation discipline:**

All three new soft-boost thresholds are placeholders — correct in shape (right field, right computation) but not validated in value. Phase 6 paper trading produces empirical distributions; thresholds get retuned in Brief v13.

Initial values reasoning:
- Liquidations: 5 events per 5m on BTCUSDT was elevated regime in Rafael's manual trading record (typical quiet regime: 0-2 events per 5m). Conservative.
- Funding: 0.0001 (= 0.01%) is roughly 12% of Bybit's typical 8h funding settlement magnitude. Below this, sign carries little information.
- OI delta percent: 1% change in OI over a 6-bar (30-minute typical) squeeze is meaningful — typical quiet regime OI fluctuates < 0.3% over similar windows.

---

## Pre-Fetched WebSocket Substrate Contract

detect_a_v2 expects caller to pre-fetch WebSocket-derived signals into the `indicators` dict under specific keys. Caller (engine.py, future) is responsible for:

- Querying `liquidations_ws` for events in `[breakout_candle_close_ts - 5min, breakout_candle_close_ts]` and computing count
- Querying `funding_rate_ws` for most recent row with `received_ts <= breakout_candle_close_ts` and extracting `funding_rate`
- Querying `open_interest_ws` for latest row with `ts <= breakout_candle_close_ts` and `ts <= squeeze_start_timestamp`, computing delta percent

**Required `indicators` dict keys:**

| Key | Type | Source | Purpose |
|-----|------|--------|---------|
| `'5m'` | `pd.DataFrame` | indicator tables | Existing v1 contract |
| `'1h'` | `pd.DataFrame` | indicator tables | Existing v1 contract |
| `'liquidations_5m_count'` | `int` | `liquidations_ws` query | Soft boost 3 |
| `'funding_rate_latest'` | `float` | `funding_rate_ws` query | Soft boost 4 |
| `'oi_squeeze_delta_pct'` | `Optional[float]` | `open_interest_ws` two-point query | Soft boost 5 |

**Key `'oi_squeeze_delta_pct'` is `Optional[float]`** because squeeze-start identification happens *inside* detect_a_v2 (Stage 2). Caller cannot pre-compute the delta until detect_a_v2 has determined when the squeeze started.

This breaks the pure pre-fetch contract for OI specifically. Two resolution options:

**Resolution A:** Caller passes `'oi_history'` as a `pd.DataFrame` containing `open_interest_ws` rows for the last LOOKBACK_V2 bars. detect_a_v2 computes the squeeze-aligned delta internally after Stage 2 squeeze-start identification.

**Resolution B:** detect_a_v2 takes an extra explicit parameter `oi_history: pd.DataFrame` outside the indicators dict for clarity.

**Decision:** Resolution A. Reasoning: keeps function signature stable (no extra parameters), extends `indicators` dict pattern naturally. Caller queries `open_interest_ws` for last 50 5m-bar-aligned rows (matched to LOOKBACK_V2), passes as DataFrame. detect_a_v2 performs the squeeze-aligned lookup against the DataFrame.

**Updated required keys:**

| Key | Type | Source | Purpose |
|-----|------|--------|---------|
| `'5m'` | `pd.DataFrame` | indicator tables | Existing v1 contract |
| `'1h'` | `pd.DataFrame` | indicator tables | Existing v1 contract |
| `'liquidations_5m_count'` | `int` | `liquidations_ws` query | Soft boost 3 |
| `'funding_rate_latest'` | `float` | `funding_rate_ws` query | Soft boost 4 |
| `'oi_history'` | `pd.DataFrame` | `open_interest_ws` last LOOKBACK_V2 rows | Soft boost 5 |

If any key is missing, detect_a_v2 returns None with an explicit log line naming the missing key. This makes substrate gaps visible at signal time.

---

## Function Signature

```python
def detect_a_v2(
    candles_5m: pd.DataFrame,
    candles_15m: pd.DataFrame,
    candles_1h: pd.DataFrame,
    candles_4h: pd.DataFrame,
    indicators: dict[str, pd.DataFrame | int | float],
) -> Signal | None:
```

Same parameter shape as v1 — only the `indicators` dict's value-type set has expanded to accept int and float values for the new substrate keys.

---

## Function Structure

Four stages. Each stage returns `None` early on failure. Only signals reaching Stage 4 emit.

### Stage 1: Hard gates (structural pattern)

```
1. Verify indicators['5m'] and indicators['1h'] DataFrames exist with sufficient history
2. Verify candles_5m has current bar
3. BB squeeze: last 6 bars (excluding current) all have bb_width < BB_SQUEEZE_THRESHOLD_V2
4. Breakout direction: current candle close > bb_upper (long) or < bb_lower (short)
5. Volume confirmation: current candle volume > volume_ma_25
6. 1h trend alignment matching breakout direction:
   - Long: 1h close > ema_21 AND sma_10 > sma_20 > sma_50
   - Short: 1h close < ema_21 AND sma_10 < sma_20 < sma_50
```

If any check fails: return None.

Squeeze check uses `iloc[-(BB_SQUEEZE_MIN_BARS_V2 + 1):-1]` (bars before current) — squeeze ends, breakout fires. v1 convention preserved.

Breakout direction check: close-only ("close outside band"). v1 convention preserved.

### Stage 2: Squeeze-start identification

For OI-delta soft boost. Walk backward from current bar through `ind_5m['bb_width']` until first bar where `bb_width >= BB_SQUEEZE_THRESHOLD_V2`. The first squeeze-bar is the next index forward.

```python
squeeze_start_idx = len(ind_5m) - 2  # bar before current (current is breakout, not squeeze)
lookback_floor = max(0, len(ind_5m) - LOOKBACK_V2)
while squeeze_start_idx > lookback_floor:
    if ind_5m.iloc[squeeze_start_idx - 1]['bb_width'] < BB_SQUEEZE_THRESHOLD_V2:
        squeeze_start_idx -= 1
    else:
        break

squeeze_start_timestamp = int(ind_5m.iloc[squeeze_start_idx]['timestamp'])
```

Squeeze capped at LOOKBACK_V2=50 bars (4 hours of 5m data). Squeezes longer than that produce different setup characteristics and are out of scope for v2.

### Stage 3: Soft-boost computation

Five additive boosts at +0.05 each, evaluated independently. Confidence accumulates from CONFIDENCE_BASE (0.75) and caps at 1.0.

**Boost 1: RSI direction-aligned (carried from v1)**
- Long: `curr['rsi_14'] > 50` → +0.05, append `'rsi_confirms_long'`
- Short: `curr['rsi_14'] < 50` → +0.05, append `'rsi_confirms_short'`

**Boost 2: Volume 2x MA25 (carried from v1)**
- `volume >= 2 * curr['volume_ma_25']` → +0.05, append `'volume_2x_ma25'`

**Boost 3: Liquidations event-rate elevated (new)**
- Read `indicators['liquidations_5m_count']` (int)
- If `>= LIQUIDATIONS_EVENT_RATE_THRESHOLD` (5) → +0.05, append `'liquidations_elevated'`

**Boost 4: Funding sign + magnitude (new)**
- Read `indicators['funding_rate_latest']` (float)
- For `direction == 'long'`: boost if `funding_rate < -FUNDING_RATE_MAGNITUDE_THRESHOLD` → +0.05, append `'funding_favorable_long'`
- For `direction == 'short'`: boost if `funding_rate > FUNDING_RATE_MAGNITUDE_THRESHOLD` → +0.05, append `'funding_favorable_short'`

**Boost 5: OI delta squeeze-aligned (new)**
- Read `indicators['oi_history']` (DataFrame with `ts` and `open_interest_value` columns)
- Find latest row with `ts <= breakout_candle_close_ts` → `oi_now`
- Find latest row with `ts <= squeeze_start_timestamp` → `oi_squeeze_start`
- If either query returns no row: skip boost (substrate gap, no boost applied, no error)
- Compute `oi_delta_pct = (oi_now - oi_squeeze_start) / oi_squeeze_start * 100`
- If `oi_delta_pct > OI_DELTA_PERCENT_THRESHOLD` (1.0) → +0.05, append `'oi_delta_significant'`
- Uniform direction interpretation: positive OI delta favorable for both long and short (positions accumulated during squeeze indicate high-conviction breakout regardless of direction).

### Stage 4: Signal emission

Build Signal dataclass:

```python
return Signal(
    setup_type='A',
    direction=direction,
    account_target='scalp',          # v12.2 Section 7 routing rule
    holding_period='scalp',
    entry=close,
    stop=bb_middle,                  # 5m middle band per Rule 9
    tp=None,                         # dynamic — Phase 4b consumes tp_strategy
    tp_strategy='structural_5m_bb',  # Phase 4b reads this for trailing logic
    confidence=min(confidence, 1.0),
    confirming_indicators=confirming,
    timeframe='5m',
    bar_time=bar_time,
)
```

Caller (engine.py, future) receives Signal and is responsible for calling `store.insert_signal(signal)`. detect_a_v2 itself does not touch storage.

---

## Storage Layer

### `signals` table DDL

```python
"signals": """
    CREATE TABLE IF NOT EXISTS signals (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        bar_time INTEGER NOT NULL,
        emitted_at INTEGER NOT NULL,
        setup_type TEXT NOT NULL,
        direction TEXT NOT NULL,
        account_target TEXT NOT NULL,
        holding_period TEXT NOT NULL,
        entry REAL NOT NULL,
        stop REAL NOT NULL,
        tp REAL,
        tp_strategy TEXT,
        confidence REAL NOT NULL,
        confirming_indicators TEXT NOT NULL,
        timeframe TEXT NOT NULL,
        symbol TEXT NOT NULL DEFAULT 'BTCUSDT'
    )
"""
```

`bar_time` and `emitted_at` are separate columns. Same bar may have multiple signal rows if engine.py re-runs the detector during the bar — preserves all emissions for 50-signal review. Deduplication is engine.py concern, not storage concern.

`tp` is nullable to support `tp_strategy='structural_5m_bb'` and other dynamic-TP strategies that future setups may introduce.

`idx_signals_bar_time` on `bar_time` for time-range queries.

### `insert_signal(signal)` method

```python
def insert_signal(self, signal) -> int:
    """Insert one signal record. Returns inserted row id.
    
    signal is an apex.signals.models.Signal instance. tp may be None
    when tp_strategy is set (dynamic TP via Phase 4b position manager).
    """
    import json
    import time
    self._conn.execute(
        "INSERT INTO signals "
        "(bar_time, emitted_at, setup_type, direction, account_target, "
        "holding_period, entry, stop, tp, tp_strategy, confidence, "
        "confirming_indicators, timeframe, symbol) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
        (
            int(signal.bar_time.timestamp() * 1000),
            int(time.time() * 1000),
            signal.setup_type,
            signal.direction,
            signal.account_target,
            signal.holding_period,
            signal.entry,
            signal.stop,
            signal.tp,
            signal.tp_strategy,
            signal.confidence,
            json.dumps(signal.confirming_indicators),
            signal.timeframe,
            getattr(signal, 'symbol', 'BTCUSDT'),
        ),
    )
    self._conn.commit()
    cur = self._conn.execute("SELECT last_insert_rowid()")
    return cur.fetchone()[0]
```

---

## Test Plan

### `tests/test_detect_a_v2.py` (20 tests)

**Hard gate tests (return None):**
1. `test_no_indicators_returns_none` — `indicators={}`
2. `test_insufficient_5m_history_returns_none` — len(ind_5m) < LOOKBACK_V2
3. `test_no_squeeze_returns_none` — last 6 bars all bb_width >= threshold
4. `test_squeeze_but_no_breakout_returns_none` — close inside bands
5. `test_breakout_no_volume_returns_none` — volume <= volume_ma_25
6. `test_long_breakout_no_1h_trend_returns_none` — 1h close < ema_21 OR sma stack not bullish
7. `test_short_breakout_no_1h_trend_returns_none` — 1h close > ema_21 OR sma stack not bearish

**Hard gate pass tests:**
8. `test_long_breakout_emits_signal` — full long path, base confidence 0.75
9. `test_short_breakout_emits_signal` — full short path, base confidence 0.75
10. `test_signal_routes_to_scalp_account` — verify `account_target == 'scalp'` (regression vs v1's `'swing'`)
11. `test_signal_emits_tp_strategy_marker` — verify `tp is None` and `tp_strategy == 'structural_5m_bb'`

**Soft boost tests:**
12. `test_rsi_aligned_long_boosts_confidence` — RSI > 50 on long signal → 0.80
13. `test_rsi_aligned_short_boosts_confidence` — RSI < 50 on short signal → 0.80
14. `test_volume_2x_boosts_confidence` — volume >= 2 * MA25 → 0.80
15. `test_liquidations_elevated_boosts_confidence` — count >= threshold → 0.80
16. `test_funding_sign_aligned_long_boosts_confidence` — funding < -threshold on long → 0.80
17. `test_funding_sign_aligned_short_boosts_confidence` — funding > threshold on short → 0.80
18. `test_oi_delta_significant_boosts_confidence` — OI delta % > threshold → 0.80
19. `test_all_boosts_active_caps_at_1_0` — all 5 boosts add to 0.75 + 0.25 = 1.0 (no 1.05)

**Squeeze-start identification:**
20. `test_squeeze_start_walks_back_correctly_and_caps_at_lookback` — both behaviors covered in single test (find first sub-threshold bar; cap at LOOKBACK_V2 boundary)

### `tests/test_store_signals.py` (5 tests)

21. `test_init_db_creates_signals_table` — verify table exists post-init
22. `test_insert_signal_returns_row_id` — basic round-trip
23. `test_insert_signal_serializes_confirming_indicators_as_json` — verify JSON encoding
24. `test_insert_signal_handles_optional_tp_with_strategy` — `tp=None, tp_strategy='structural_5m_bb'` round-trips correctly
25. `test_signals_table_allows_duplicates_per_bar` — same `(bar_time, setup_type, direction)` two rows OK

**Total: 25 tests across 2 files.**

### Test Fixtures

Module-level fixture in `tests/conftest.py` (or `tests/test_detect_a_v2.py` if not shared):

```python
@pytest.fixture
def synthetic_long_breakout_indicators():
    """Build minimal indicators DataFrames + WebSocket data dict
    that triggers a clean long-breakout signal at base confidence 0.75."""
    # ind_5m: 50 bars, last 6 in squeeze, current bar close > bb_upper, 
    #         volume > MA25, RSI = 50 (no boost), volume_ma_25 set
    # ind_1h: bullish trend stack (close > ema_21, sma_10 > sma_20 > sma_50)
    # candles_5m: current bar matching breakout
    # candles_1h: close > ema_21
    # liquidations_5m_count: 0 (no boost)
    # funding_rate_latest: 0 (no boost)
    # oi_history: flat (no delta, no boost)
    return {
        'candles_5m': ...,
        'candles_15m': ...,
        'candles_1h': ...,
        'candles_4h': ...,
        'indicators': {
            '5m': ...,
            '1h': ...,
            'liquidations_5m_count': 0,
            'funding_rate_latest': 0.0,
            'oi_history': ...,
        },
    }
```

Inverse fixture for short.

Each soft-boost test uses the base fixture and modifies the single relevant input.

---

## Sequencing for Claude Code

Six commits on branch `phase3-v2-detect-a` (off main). Each commit independently passes its scoped tests. Architect reviews diff per commit per Brief v12.2 Section 10 standing instruction "Surface deviations, do not absorb them."

**Step 1 — Signal dataclass extension.**
- File: `apex/signals/models.py`
- Change: `tp: float` → `tp: Optional[float] = None`. Add `tp_strategy: Optional[str] = None`.
- Verify: existing v1 tests still pass (regression — v1 detect_a, detect_f, detect_b construct Signal without `tp_strategy`).
- Commit: `phase3-v2 step 1: Signal dataclass extension for dynamic TP support`

**Step 2 — Storage layer.**
- File: `apex/data/store.py`
- Change: add `signals` DDL to `_MARKET_DATA_TABLES`, add `idx_signals_bar_time` index in `init_db()`, add `insert_signal()` method.
- Verify: tests 21-25 pass.
- Commit: `phase3-v2 step 2: signals table DDL + insert_signal method`

**Step 3 — Constants + function skeleton (hard gates only, returns sentinel).**
- File: `apex/signals/setups.py`
- Change: add v2 constants block + `detect_a_v2()` with Stage 1 hard gates, returns None on every path (no Signal emission yet).
- Verify: tests 1-7 pass (all hard gate tests expect None).
- Commit: `phase3-v2 step 3: detect_a_v2 hard gates skeleton`

**Step 4 — Stage 1 emission (base confidence, no boosts, no squeeze-start).**
- File: `apex/signals/setups.py`
- Change: complete Stage 1 path → Signal emission with base confidence 0.75, `tp=None`, `tp_strategy='structural_5m_bb'`, `account_target='scalp'`, no soft boosts computed.
- Verify: tests 8-11 pass.
- Commit: `phase3-v2 step 4: detect_a_v2 base signal emission`

**Step 5 — Stage 2 squeeze-start identification.**
- File: `apex/signals/setups.py`
- Change: add Stage 2 walk-back logic.
- Verify: test 20 passes.
- Commit: `phase3-v2 step 5: detect_a_v2 squeeze-start identification`

**Step 6 — Stage 3 soft boosts (all 5).**
- File: `apex/signals/setups.py`
- Change: add five boost computations consuming `indicators` dict keys.
- Verify: tests 12-19 pass. Full regression: tests 1-25 all pass on single run.
- Commit: `phase3-v2 step 6: detect_a_v2 soft confidence boosts`

After Step 6: branch ready for chat-Claude diff review across all 6 commits, then merge to main.

---

## Pre-Write Verification Checklist for Claude Code

Per Brief v12.2 Section 10 standing instruction "Re-read before deciding," Claude Code must verify these assumptions before any code change:

1. `apex/signals/models.py` `Signal` dataclass current shape — confirm fields and verify Optional/default-value mechanics work for the proposed extension.
2. `apex/data/store.py` `_MARKET_DATA_TABLES` dict structure — confirm new entry pattern matches existing entries.
3. `apex/data/store.py` `init_db()` index creation pattern — confirm new index addition matches existing pattern.
4. `apex/signals/setups.py` v1 import chain — verify `from apex.signals.models import Signal` is the existing import.
5. `tests/` directory structure — confirm pytest discovery matches existing test files; confirm fixture conventions in use.
6. `pyproject.toml` (or equivalent) — confirm pytest is dependency; no new dependencies needed.

Surface any deviation found. Do not absorb silently.

---

## Anti-Martingale Validation

- Brief v12.2 ships first ✓ (shipped this session)
- detect_a v2 proposal references v12.2 canonical brief ✓
- Proposal scope contained: detect_a only, not detect_b/c/d/e/f/g ✓
- Phase 4b TCH framework referenced but explicitly out of scope ✓
- Storage layer ships to satisfy 50-signal review gate ✓
- Implementation deferred to Claude Code session (proposal does not ship code) ✓
- Six small commits with per-commit test passes (not one large commit) ✓

End of proposal. Ready for Claude Code execution next session.
