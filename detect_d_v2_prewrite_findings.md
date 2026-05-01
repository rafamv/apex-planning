# detect_d_v2 Pre-Write Verification Findings — Session 40

**Branch:** `phase3-v2-detect-d` at commit `c34178a` (Brief v12.3)
**Proposal reference:** `detect_d_v2_proposal.md` §9 (Pre-Write Verification Checklist)
**Date:** 2026-05-01

---

## Item 1: detect_a_v2 constants at module level

Status: PASS

Evidence:
```
$ grep -n "^# --- Setup A v2 Constants\|^BB_SQUEEZE_THRESHOLD_V2\|^CONFIDENCE_BASE\|^CONFIDENCE_BOOST\|^def detect_a_v2\|^def _identify_squeeze_start" <REPO_ROOT>apex/signals/setups.py
16:# --- Setup A v2 Constants ---
17:BB_SQUEEZE_THRESHOLD_V2 = 1.5            # bb_width % below this = squeeze (carried from v1)
27:CONFIDENCE_BASE = 0.75                   # all hard gates passed
28:CONFIDENCE_BOOST = 0.05                  # uniform across all 5 soft boosts
140:def _identify_squeeze_start(ind_5m: pd.DataFrame) -> int:
156:def detect_a_v2(
```

`BB_SQUEEZE_THRESHOLD_V2`, `CONFIDENCE_BASE`, `CONFIDENCE_BOOST` and the rest of detect_a_v2's constants block live at module level (lines 16-28). Helper `_identify_squeeze_start` and main function `detect_a_v2` also at module level. detect_d_v2's constants block can sit alongside (e.g., immediately after detect_a_v2's constants block, around line 30) without any import gymnastics or namespace conflict.

Resolution needed: none.

---

## Item 2: Signal import chain with tp_strategy field

Status: PASS

Evidence:
```
$ grep -n "tp_strategy\|tp:" <REPO_ROOT>apex/signals/models.py
21:    tp:                    Optional[float]
31:    tp_strategy:           Optional[str] = None  # e.g. 'structural_5m_bb' for dynamic TP via Phase 4b position manager
52:        if self.setup_type not in ('F', 'G') and self.tp_strategy is None and self.rr_ratio < 1.5:
```

`tp: Optional[float]` (line 21) and `tp_strategy: Optional[str] = None` (line 31) both present on the Signal dataclass. Validator at line 52 correctly skips the R:R 1.5 check when `tp_strategy is not None` — meaning detect_d_v2 can emit `tp=None, tp_strategy='structural_5m_bb'` without triggering SignalValidationError. Import chain `from apex.signals.models import Signal` reusable verbatim from detect_a_v2.

Resolution needed: none.

---

## Item 3: 5m indicator table columns

Status: PASS (with documentation note)

Evidence:
```
$ uv run python -c "import sqlite3; conn = sqlite3.connect('<REPO_ROOT>data/apex.db'); ..."
Item 3a 5m IND missing: none — all present
Item 3b: volume column lives in: candles_5m=True, ind_5m=False
```

Required indicator columns on `indicators_BTCUSDTUSDT_5m` (per proposal §7): ema_21, sma_50, ema_50, sma_100, sma_250, bb_middle, volume_ma_25, rsi_14, timestamp. All 9 present.

Documentation note: proposal §7 lists `volume` under the 5m IND column requirements, but `volume` actually lives in `candles_5m`, not `indicators_BTCUSDTUSDT_5m`. detect_a_v2 reads volume from candles via `candle = candles_5m.iloc[-1]; volume = candle['volume']` (setups.py:201-203), NOT from the indicators DataFrame. detect_d_v2 will follow the same precedent. This is a minor proposal-text error; no code-blocking impact.

Resolution needed: none. detect_d_v2 reads volume from candles_5m, mirroring detect_a_v2.

---

## Item 4: 1h indicator table columns

Status: PASS

Evidence:
```
Item 4 1h IND missing: none — all present
```

Required indicator columns on `indicators_BTCUSDTUSDT_1h` (per proposal §7): ema_21, sma_10, sma_20, sma_50. All 4 present (existing pre-merge from prior schema).

Resolution needed: none.

---

## Item 5: pytest fixture conventions

Status: PASS

Evidence:
```
$ grep -n "^@pytest.fixture\|^def synthetic_\|^def _build_" <REPO_ROOT>tests/test_detect_a_v2.py
38:def _build_ind_5m(
71:def _build_ind_1h(
102:def _build_candles(n: int, *, close: float, volume: float, ...
116:@pytest.fixture
117:def synthetic_long_breakout_indicators():
147:@pytest.fixture
148:def synthetic_short_breakout_indicators():
342:def _build_oi_history(timestamps: list[int], values: list[float]) -> pd.DataFrame:
```

detect_a_v2 test file establishes the convention:
- Module-level builder helpers: `_build_ind_5m`, `_build_ind_1h`, `_build_candles`, `_build_oi_history`
- `@pytest.fixture` decorators on `synthetic_<scenario>_indicators` returning a dict with keys `candles_5m`, `candles_15m`, `candles_1h`, `candles_4h`, `indicators`
- Test bodies mutate fixture via `.copy()` + `.loc[]` for negative-test cases

detect_d_v2 will reuse this pattern with `synthetic_long_pullback_indicators` / `synthetic_short_pullback_indicators` / `synthetic_long_pullback_old_cross_indicators` per proposal §8 Test Fixtures section.

Resolution needed: none.

---

## Item 6: oi_history column names — DEVIATION

Status: DEVIATION

Evidence:

Production schema (store.py):
```sql
"open_interest_ws": """
    CREATE TABLE IF NOT EXISTS open_interest_ws (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        ts INTEGER NOT NULL,
        received_at_ms INTEGER NOT NULL,
        symbol TEXT NOT NULL,
        open_interest REAL NOT NULL,           -- contract count
        open_interest_value REAL NOT NULL,     -- USD value (= open_interest * price)
        cs INTEGER
    )
""",
```

detect_a_v2 reads `open_interest` (contract count):
```
$ grep -n "open_interest\|oi_history\|oi_now\|oi_squeeze" <REPO_ROOT>apex/signals/setups.py
280:    oi_history = indicators.get('oi_history')
281:    if oi_history is not None and len(oi_history) > 0:
283:        oi_now_rows = oi_history[oi_history['ts'] <= breakout_ts]
284:        oi_start_rows = oi_history[oi_history['ts'] <= squeeze_start_timestamp]
285:        if len(oi_now_rows) > 0 and len(oi_start_rows) > 0:
286:            oi_now = oi_now_rows.iloc[-1]['open_interest']
287:            oi_squeeze_start = oi_start_rows.iloc[-1]['open_interest']
```

detect_d_v2 proposal §6 Stage 6 Boost 4 specifies `open_interest_value`:
> Read `indicators['oi_history']` (DataFrame with `ts` and `open_interest_value` columns)

And proposal §9 checklist item 6:
> confirm `ts` and `open_interest_value` are the exact column names produced by `open_interest_ws` query helper. If column names differ, surface deviation; do not silently rename.

The production schema has both columns. The proposal asks me to verify against the schema; both are present. The deviation is between detectors:

- detect_a_v2 (shipped): reads `open_interest` (contracts)
- detect_d_v2 (proposed): reads `open_interest_value` (USD)

Resolution needed: architect call.

Semantic impact:
- `open_interest` (contracts) gives clean accumulation signal — OI growth = real positions added.
- `open_interest_value` (USD) conflates price moves with position changes: `open_interest_value = open_interest × price`. During a trending continuation (price moves in the trend direction), USD-OI rises even when contract count is flat. The boost would fire on price-move-only OI inflation, not pure-accumulation, which is the opposite of what "high-conviction directional bias" should mean per proposal §1 decision 5.

Recommendation: `open_interest` (contracts).
- Cleaner accumulation semantic
- Matches detect_a_v2 precedent (cross-detector consistency)
- Avoids the conflation issue above

If proposal authors had a deliberate reason for `open_interest_value` (e.g., USD-denominated conviction is the intended signal), they should confirm — otherwise the recommendation is to align detect_d_v2 with detect_a_v2's `open_interest` and update proposal §6 + §9 item 6 in the next proposal revision.

---

## Summary

PASS: 5 (items 1, 2, 3, 4, 5)
DEVIATION: 1 (item 6 — oi_history column choice)

Awaiting chat-Claude resolution on item 6 before Step 1 code write.
