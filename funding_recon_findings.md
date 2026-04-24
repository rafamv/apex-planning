# Step F Funding Reconciliation Findings

**Date:** 2026-04-24
**Branch:** phase1-v2-websocket
**Files inspected:**
- `apex/data/ws_handlers/tickers.py` (WS write path)
- `scripts/run_step_f.py` (REST-vs-WS match path)
- Smoke run output (2026-04-24 22:54–22:59 UTC)

## Verdict

**Timestamps are semantically aligned.** REST `fundingRateTimestamp` and WS `funding_rate_ws.next_funding_time` hold the same value in milliseconds for the same settled cycle. No offset, no bug. The `no_match` result in smoke is expected behaviour for a 5-min window that does not contain a cycle boundary.

**Decision per spec rule:** proceed to cycle-boundary smoke at 00:00 UTC.

## Trace

### Bybit V5 ticker field semantics

`nextFundingTime` on `tickers.BTCUSDT` is the ms-epoch of the upcoming settlement. During an active cycle (e.g. between 08:00 and 16:00 UTC), it carries the NEXT boundary (16:00). At the instant of settlement, Bybit flips it to the FOLLOWING boundary (24:00). The `fundingRate` field behaves in parallel: it carries the rate that WILL apply at the next settlement during the active cycle, then flips to the new cycle's rate at the boundary.

### WS write path — `apex/data/ws_handlers/tickers.py:64-103`

Handler retains the previous merged state in `self._last_ticker`. On every incoming frame:

1. It reads pre-merge values:
   ```python
   prev_next_funding_time = self._last_ticker.get("nextFundingTime")  # :68
   prev_funding_rate = self._last_ticker.get("fundingRate")           # :69
   ```

2. Detects cycle advance when the incoming `nextFundingTime` is strictly greater than the prior one:
   ```python
   cycle_advanced = (
       new_next_ft_raw is not None
       and prev_next_funding_time is not None
       and int(new_next_ft_raw) > int(prev_next_funding_time)
   )   # :75-79
   ```

3. Merges the frame (snapshot replace OR delta update) — line 81-84.

4. On advance, writes the row keyed by the PRE-merge `nextFundingTime`:
   ```python
   funding_record = {
       "next_funding_time": int(prev_next_funding_time),  # :88
       "funding_rate": float(prev_funding_rate),          # :90
       ...
   }
   store.upsert_funding_rate_ws(funding_record)
   ```

**Therefore the WS row's PK `next_funding_time` equals the boundary that JUST SETTLED, not the next upcoming one.** The column name is past-tense in practice despite its future-tense-sounding label.

### REST read path — Bybit V5 `/v5/market/funding/history`

With `limit=1`, returns the most recently settled cycle:
- `fundingRateTimestamp` = ms-epoch of the settlement boundary
- `fundingRate` = rate applied at that settlement

### Match logic — `scripts/run_step_f.py:127-163`

```python
rest_ts = int(rest_list[0]["fundingRateTimestamp"])      # :144
...
cur = store._conn.execute(
    "SELECT funding_rate FROM funding_rate_ws WHERE next_funding_time = ?",
    (rest_ts,),
)  # :146-149
```

The WHERE clause compares REST `fundingRateTimestamp` against the WS `next_funding_time` column. Both are ms-int, both are the settlement boundary for the same cycle. This is the correct key.

### Worked example at a 16:00 UTC boundary

Assume WS connection was live before and during 16:00 UTC on a given day.

| Wall clock | Ticker frame carries | Handler state (pre-advance) | Action |
|---|---|---|---|
| 15:59:59 | `nextFundingTime=16:00:00`, `fundingRate=r₁` | `_last_ticker.nextFundingTime=16:00:00` | merge |
| 16:00:00 (settlement) | `nextFundingTime=24:00:00`, `fundingRate=r₂` | `prev=16:00:00`, `new=24:00:00` → `cycle_advanced=True` | write row: `next_funding_time=16:00:00` ms, `funding_rate=r₁` |

Any subsequent `/v5/market/funding/history?limit=1` call returns `fundingRateTimestamp=16:00:00` ms, `fundingRate=r₁`. The reconciliation query `WHERE next_funding_time = 16:00:00_ms` finds the row written above. Delta = |r₁ - r₁| = 0. PASS within 1e-06 tolerance.

## Smoke evidence (corroborates alignment, does not contradict)

The 5-min smoke ran 22:54–22:59 UTC on 2026-04-24. REST query returned:

```
rest_ts=1777046400000  →  2026-04-24 16:00:00 UTC
rest_fr=-4.07e-05
```

The 16:00 UTC settlement occurred ~6h 54min BEFORE the smoke's WS connection opened. The handler could only have captured a cycle advance if the WS connection had been live AT the 16:00 boundary — it was not. Therefore `funding_rate_ws` was (correctly) empty at smoke end.

Match attempt:
```
SELECT funding_rate FROM funding_rate_ws WHERE next_funding_time = 1777046400000
→ 0 rows
→ status: no_match
```

This is the informational branch for "REST knows about a cycle we haven't observed yet," not a bug path. Had the handler captured the 16:00 advance, the WHERE clause would have hit and we'd see `status: compared, pass: true`.

## Precision / type notes

- Bybit V5 sends most numeric fields as JSON strings. Handler coerces to `int()` before comparison and storage. REST handler same (`int(rest_list[0]["fundingRateTimestamp"])`).
- Schema: `funding_rate_ws.next_funding_time INTEGER PRIMARY KEY`. WHERE clause binds an `int` param. Exact match.
- No float funny business on the key — only on the value (`funding_rate`), and only for the tolerance comparison.

## Naming nit (non-blocking)

The column `funding_rate_ws.next_funding_time` stores "the boundary that just settled," not "when the next funding will happen." The name echoes Bybit's field name at the moment of capture, which is defensible, but a future reader might misread. Not a reconciliation bug. Leave alone unless Architect wants a rename pass later.

## 8h run prediction

If the 8h run starts at ~23:00 UTC 2026-04-24, it covers boundaries at 00:00 and 08:00 UTC 2026-04-25:

- **First cycle advance** captured at ~00:00 UTC → writes row `next_funding_time=1777075200000`, `funding_rate=<settled_rate>`.
- **First reconciliation after 00:00** (next 30-min tick) → REST returns `fundingRateTimestamp=1777075200000`. WHERE clause hits. Expected `delta ≤ 1e-06` → PASS.
- **Second cycle advance** at 08:00 UTC → writes row `next_funding_time=1777104000000`.
- **Reconciliation after 08:00** → PASS.

Final-report criterion 3 (`funding ≥ 1 row`) becomes satisfied at the first cycle advance, well within the 8h window.

## Recommendation

No code change. Proceed per spec decision rule: cycle-boundary smoke at 00:00 UTC. A 6-minute smoke spanning 23:57–00:03 UTC would guarantee the handler captures one cycle advance AND one reconciliation run fires after the boundary — full proof of the funding path before 8h.
