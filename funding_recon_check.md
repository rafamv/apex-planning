# Step F funding reconciliation logic check

Read scripts/run_step_f.py funding reconciliation logic. Specifically: how does it match the REST response (which returns fundingRate + fundingRateTimestamp = settled cycle boundary, e.g. 16:00:00 UTC) against funding_rate_ws table rows (which are written at cycle-advance, with cycle_end_ms = the previous cycle's nextFundingTime — i.e. the boundary that just settled)?

## Question

Are those timestamps semantically the same value at the same moment, or are they offset by one cycle?

Read TickerHandler funding write logic in apex/data/ws_handlers/tickers.py for how cycle_end_ms is set. Read the reconciliation match logic in scripts/run_step_f.py for how it queries funding_rate_ws.

## Trace evidence from smoke

The 5-min smoke fired reconcile_funding with rest_ts=1777046400000 (= 2026-04-24 16:00:00 UTC). The previous funding cycle advance was at 16:00 UTC today. Would TickerHandler have written cycle_end_ms=1777046400000 at that advance, or would it have written some different value?

## Decision rule

- If timestamps are semantically aligned (REST fundingRateTimestamp === WS cycle_end_ms for the same settled cycle): confirm with one trace example. Then we proceed to cycle-boundary smoke at 00:00 UTC.

- If timestamps are NOT semantically aligned: this is a reconciliation bug that would silently fail the 8h run too. Surface the bug. Propose a fix. Do NOT modify code yet — await architect approval.

Do not approve any next run until this analysis comes back.
