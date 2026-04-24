# Phase 1 v2 Step F — Live Regression Script Proposal

## Goal
Build `scripts/run_step_f.py` — 8h regression test of WebSocket pipeline against Bybit mainnet. Closes Phase 1 v2 fully.

## Rule 6 verifications (already executed by Claude Code, findings approved)

1. `run_ws()` accepts Store via constructor pattern — confirmed reusable for `data/step_f.db`
2. `store.init_db("BTCUSDT", ["5m", "15m", "1h", "4h", "1d"])` creates candle tables (overhead, harmless) + WS shadow tables (load-bearing). Use as-is.
3. httpx absent. Add via `uv add httpx`.
4. `TickerHandler.reset()` does not log. Add `import logging` + `logger = logging.getLogger(__name__)` at module top, plus `logger.info("ticker handler reset on reconnect")` in reset() body.
5. No singletons in ws_client.py. Dual-subscriber safe (apex:0 against `data/apex.db`, apex:4 against `data/step_f.db`).

## Work scope

A. `uv add httpx`
B. Add logger to `TickerHandler.reset()`
C. Create `scripts/run_step_f.py` per spec below
D. Run 5-min smoke pre-flight, report results
E. STOP — await approval before 8h run

## scripts/run_step_f.py spec

### Async architecture
- Reads `MAX_RUNTIME_SECONDS` env var, default 28800 (8h). Override for smoke: 300 (5min).
- Constructs Store pointing at `data/step_f.db`, calls `store.init_db("BTCUSDT", ["5m", "15m", "1h", "4h", "1d"])` to mirror schema.
- Constructs `BybitWsClient` mirroring `run_ws()` pattern in `apex/data/ws_client.py`.
- Subscribes to `tickers.BTCUSDT` + `allLiquidation.BTCUSDT`.
- Registers `TickerHandler()` + `handle_all_liquidation` via `set_handler`.
- Runs ws_client until `MAX_RUNTIME_SECONDS` elapsed. Use `asyncio.wait_for` or task-cancel timeout pattern.

### Periodic logging — every 5 minutes
Log row counts of: `funding_rate_ws`, `open_interest_ws`, `liquidations_ws`, `ws_reconnect_gaps`. Stdout + tee'd log file.

### Reconciliation pass — every 30 minutes
- **OI:** `GET https://api.bybit.com/v5/market/open-interest?category=linear&symbol=BTCUSDT&intervalTime=5min&limit=1` → compare `openInterest` value to most recent `open_interest_ws.open_interest` row. Tolerance: **±0.5%**. REST 5-min snapshot vs WS continuous push will differ slightly within this band.
- **Funding:** `GET https://api.bybit.com/v5/market/funding/history?category=linear&symbol=BTCUSDT&limit=1` → compare `fundingRate` to most recent `funding_rate_ws.funding_rate` row matching same `cycle_end_ms` (REST returns settled rate keyed by fundingRateTimestamp ms). Tolerance: **1e-06**. Step D captures at cycle-advance pre-merge per v32 Part 19.1, expect tight match unlike Step B's pre-settlement T-60s capture.
- **Liquidations:** SKIP. Bybit liquidation REST is saturation-prone per v26 Part 12.2 (Phase 1 v1 limit=1000 every 30s = chronic 1000-cap-hit). The whole reason we moved to WebSocket. Log row count growth as the only signal.
- **Failure semantics:** outside tolerance → WARN logged. Run continues. Documented in final report. Does NOT abort.
- Reconciliation results: logged to stdout + appended to `/tmp/step_f_run_<timestamp>.log`.

### Graceful shutdown
On `MAX_RUNTIME_SECONDS` reached: cancel ws_client task, close store, write final report to `/tmp/step_f_report_<timestamp>.txt`.

### Final report — pass criteria
1. Zero unhandled exceptions during run.
2. OI handler wrote ≥ 60 rows/hr averaged over the run.
3. Funding handler wrote ≥ 1 row (cycle advance occurred during 8h window — cycle period is 8h on Bybit so smoke-5min may produce 0).
4. Liquidations handler wrote ≥ 1 row/hr averaged.
5. If `ws_reconnect_gaps` rows present: `reset_handlers` log line present in run log for each gap.
6. All reconciliation passes within tolerance OR all WARN'd ones documented in report.

Exit 0 on all pass. Exit 1 on any fail.

### Exception handling
All exceptions during run: log full traceback. Set `unhandled_exception` flag. Continue if recoverable. Exit 1 in final report.

## Deferred (do not include)
- Induced disconnect — natural disconnects only per architect decision.
- Liquidation REST reconciliation — saturation-prone.
- Production `data/apex.db` touch — sandboxed to `data/step_f.db`.

## Tests
None. Operational script, not library. Validation is smoke + 8h run itself.

## Commit
Message: `Phase 1 v2 Step F: live regression script (5-min smoke + 8h run scaffold)`. Commit only after smoke passes. Branch: `phase1-v2-websocket`. Per CLAUDE.md: no Co-Authored-By, attributed to Rafael.

## Smoke pre-flight criteria (before 8h run)
- All three handlers wrote at least once
- Reconciliation ran at least once
- No exceptions
- If pass: kill smoke, start 8h run
- If fail: diagnose before 8h

## 8h run command
```bash
tmux new-window -t apex -n step_f
# in new window:
cd /root/code/apex
MAX_RUNTIME_SECONDS=28800 uv run python scripts/run_step_f.py 2>&1 | tee /tmp/step_f_run_$(date +%Y%m%d_%H%M).log
```
