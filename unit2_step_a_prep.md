# Unit 2 Step A — Preparation

Prerequisite questions and payload-probe proposal before drafting shadow-table DDL.

Context: Phase 1 v2 Unit 2 — WebSocket topic handlers. Branch `phase1-v2-websocket`. Unit 1 shipped (`apex/data/ws_client.py`, 245 lines, commits `7ca01dc` + `88d1d6f`). Architectural decisions locked per session prompt (shadow tables, accept-gap with RECONNECT_GAP log, funding REST verification first, `apex/data/handlers/` subdir).

Sources read:

- `CLAUDE.md`
- `APEX_BRIEF_v12.1.md` (especially Section 5 and Section 5.5)
- `apex/data/ws_client.py`
- `apex/data/store.py`
- `apex/data/liquidation_poller.py`
- `apex/data/market_data.py`
- `apex/data/exchange.py`
- `tests/test_ws_unit1_recovery.py`

---

## Assumptions and open questions

### A. Payload shapes — not observed yet

Brief v12.1 Section 5.5 names the ticker fields (`fundingRate`, `nextFundingTime`, `openInterest`, `openInterestValue`) but does NOT give the liquidation event record's field names. Writing DDL without seeing a real frame means guessing Bybit's V5 WebSocket field naming convention. The rule is no guessed field names.

**Proposal:** run a bounded one-shot probe script (≤60s) against the live public stream that logs one full `tickers.BTCUSDT` snapshot frame, several delta frames, and any `allLiquidation.BTCUSDT` frames that arrive. Raw output dumped to stdout/file, not DB. Read-only, no credentials, no DB writes. See "Payload probe script" below.

Open: do we run the probe as a prerequisite to Step A, or is there a captured frame dump in the tapes (v27 / v28 / v29) or a log file that already has live frames we can read?

### B. REST shape vs WebSocket shape — known deltas

From reading the code:

- Canonical `liquidations` table (`store.py:38`) has `(timestamp, symbol, side, price, quantity)` — populated by the REST poller from `/v5/market/recent-trade` with `execType=BustTrade` (`liquidation_poller.py:42`). WebSocket `allLiquidation.BTCUSDT` almost certainly has the same economic fields but with Bybit's WS naming convention (likely abbreviated single-letter keys per Bybit V5 WS style). Shadow table columns should be chosen from the WS payload, not cloned from REST.
- Canonical `funding_rate` has `(timestamp, symbol, funding_rate)` — no `nextFundingTime` column. Brief v12.1 explicitly carries `nextFundingTime` in the WS frame. Shadow table needs an extra column.
- Canonical `open_interest` has `(timestamp, symbol, open_interest)` — one field. Brief v12.1 says WS carries both `openInterest` AND `openInterestValue` (notional). Shadow needs both.

### C. Deduplication semantics

Brief Section 5 §1 says "storage logic deduplicates on `fundingRateTimestamp` change" for funding. Brief Section 5 §2 says "every-push resolution, one row per update frame" for OI. Those are different semantics and should be reflected in shadow table design.

**Proposed split:**

- `funding_rate_ws` dedups on `fundingRateTimestamp` change — matches canonical semantics so the 7-day validation can do apples-to-apples row-count and value comparison.
- `open_interest_ws` every-push — per brief. Add a `received_at_ms` column separate from payload `ts` so validation can also measure WS push frequency over the 7-day window.

Confirm or override.

### D. Symbol representation in shadow tables

Canonical tables store `symbol` as `"BTC/USDT:USDT"` (CCXT unified form — see `liquidation_poller.py` where `symbol` is passed through unchanged). WebSocket topic is `BTCUSDT` (Bybit native). Storing CCXT form (`BTC/USDT:USDT`) in shadow tables lets validation queries join cleanly on `symbol` against canonical. Topic name is already implicit. Proposing CCXT form.

### E. RECONNECT_GAP — dedicated table

Decision #2 says "log a RECONNECT_GAP record with start/end timestamps and duration." Three options:

1. Column in each shadow table flagging the row as a gap marker — mixes data and metadata.
2. Dedicated `ws_reconnect_gaps` table: `(start_ts, end_ts, duration_ms, reason)`.
3. Log-only, no DB persistence.

**Proposing option 2.** Phase 4 signal generation needs to query gaps programmatically; log-only doesn't survive a restart and isn't queryable. Single table covers all three topics.

### F. Store methods — new, not reused

`Store` currently has `insert_liquidations(records)`, `upsert_funding_rates(records)`, `upsert_open_interest(records)`. Shadow tables need their own methods because column sets differ. **Proposing** `insert_liquidations_ws`, `upsert_funding_rate_ws`, `insert_open_interest_ws`, and `insert_reconnect_gap`. Not reusing the existing methods.

### G. `init_db` wiring

Shadow table DDL should be added to `_MARKET_DATA_TABLES` in `store.py` so `init_db` creates them on first run. Non-destructive — `CREATE TABLE IF NOT EXISTS` is safe against existing deployments. Confirm this is the right place vs. a separate `init_ws_shadow_tables` method.

### H. Funding REST verification script — environment

Brief Section 8 says "testnet or mainnet public stream." Funding rate timing is real-time with an 8h cycle; testnet funding values do not necessarily track mainnet. For verification to mean anything we need the same venue on both sides: mainnet WS vs mainnet REST `/v5/market/funding/history`. The test runs for ~8h in a single shot — this is a long-running manual test, not CI. **Proposing** it runs on the droplet under tmux, documented as "run once pre-Step-C, not CI."

### I. Handler signature — callable vs class

`ws_client.py:60` currently stores handlers as `dict[str, object]` with stub callables `async def _stub_handler(topic, data, ts)`. Two options for Unit 2:

1. Plain async functions in `handlers/*.py` that take `(store, topic, data, ts)`. `ws_client.py` binds `store` via partial or closure when wiring.
2. Handler classes with `__init__(store, symbol)` and `async def handle(topic, data, ts)` — instance attributes carry last-known-state for ticker-delta merge.

Ticker handler needs persistent last-known-state across frames. Option 2 fits cleanly; option 1 needs module-level state or a store closure. **Proposing option 2.** Liquidation handler doesn't need state but using the same pattern is uniform.

### J. ws_client subscription list — no change needed

Both topics Brief v12.1 specifies (`allLiquidation.BTCUSDT`, `tickers.BTCUSDT`) are already subscribed (`ws_client.py:23-26`). No subscription list changes for Step E. Step E is purely replacing `_stub_handler` references in `self._handlers` with real handler-instance callbacks.

### K. Symbol parameter flow

`BybitWsClient.__init__(store, symbol)` receives `symbol` but the subscription list is hardcoded to `BTCUSDT` strings. `self._symbol` is stored but unused in Unit 1. If Unit 2 is BTCUSDT-only that's fine — symbol is a stored param for future multi-symbol support. Handlers take `symbol` via constructor so record writing stays parametric.

---

## Payload probe script (draft)

One-shot, ≤60s, read-only. Not checked in yet — drafted here for review; if approved, land it at `scripts/probe_ws_payload.py` (separate commit from this prep doc).

```python
"""Bounded WebSocket payload probe.

Subscribes to tickers.BTCUSDT + allLiquidation.BTCUSDT on the Bybit V5
public linear stream for a short window and dumps raw frames to stdout.

Purpose: inspect real frame shapes before writing Unit 2 shadow-table DDL.
Read-only. No DB writes. No credentials.

Run:
    uv run python scripts/probe_ws_payload.py --seconds 60
"""

import argparse
import asyncio
import json
import time

import aiohttp

WSS_URL = "wss://stream.bybit.com/v5/public/linear"
TOPICS = ["allLiquidation.BTCUSDT", "tickers.BTCUSDT"]


async def probe(duration_s: int) -> None:
    deadline = time.monotonic() + duration_s
    ticker_frames_logged = 0
    liq_frames_logged = 0
    MAX_TICKER_LOG = 6  # snapshot + a few deltas

    async with aiohttp.ClientSession() as session:
        async with session.ws_connect(WSS_URL) as ws:
            await ws.send_str(json.dumps({"op": "subscribe", "args": TOPICS}))
            print(f"# subscribed to {TOPICS}")

            while time.monotonic() < deadline:
                remaining = deadline - time.monotonic()
                try:
                    msg = await asyncio.wait_for(ws.receive(), timeout=remaining)
                except asyncio.TimeoutError:
                    break

                if msg.type != aiohttp.WSMsgType.TEXT:
                    continue

                data = json.loads(msg.data)
                topic = data.get("topic")

                if topic == "tickers.BTCUSDT" and ticker_frames_logged < MAX_TICKER_LOG:
                    print(f"\n# === tickers frame #{ticker_frames_logged} ===")
                    print(json.dumps(data, indent=2))
                    ticker_frames_logged += 1
                elif topic == "allLiquidation.BTCUSDT":
                    print(f"\n# === allLiquidation frame #{liq_frames_logged} ===")
                    print(json.dumps(data, indent=2))
                    liq_frames_logged += 1
                elif "op" in data:
                    # subscribe ack / pong — log once for reference
                    print(f"# control frame: {json.dumps(data)}")

    print(
        f"\n# done: {ticker_frames_logged} ticker frames, "
        f"{liq_frames_logged} liquidation frames"
    )


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--seconds", type=int, default=60)
    args = parser.parse_args()
    asyncio.run(probe(args.seconds))


if __name__ == "__main__":
    main()
```

Note: this probe does NOT do ping/pong; 60s is well inside the keepalive window so Bybit will not close the connection. It also does NOT handle reconnect — single-shot by design.

---

## Bottom-line

Questions **A** (real payload shapes) and **C** (dedup semantics) are load-bearing for the Step A DDL. Resolve those two — ideally by running the probe script above — before drafting concrete columns. The remaining questions (B, D, E, F, G, H, I, J, K) are confirmable as the proposal takes shape.
