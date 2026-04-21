# Unit 2 Step A — Shadow Table DDL Proposal

Based on payload probe output in `planning/unit2_step_a_probe_output.md` (commit `bfa77be`) and approvals on `planning/unit2_step_a_prep.md` (commit `28be284`).

Non-destructive — all DDL is `CREATE TABLE IF NOT EXISTS`. Added to `_MARKET_DATA_TABLES` in `apex/data/store.py`. Written against observed WS payload shapes where possible.

---

## Findings that deviate from Brief v12.1 Section 5.5

Surfaced from the probe, not absorbed silently. These affect the DDL and the handlers.

### 1. Subscribe-ack frame DOES exist

Brief 5.5 says: "Bybit sends NO explicit subscribe-ack frame. Subscription is confirmed implicitly via first topic data frame within 10 seconds of the subscribe message."

Probe observed (line 107 of output):

```
{"success": true, "ret_msg": "", "conn_id": "d6ki0d7qlk1a2463ccmg-a9uuo",
 "req_id": "", "op": "subscribe"}
```

It arrived AFTER the first six ticker frames, not before — which is why Unit 1's "wait for first topic data" approach works. The ack is real but unreliably ordered. Unit 1 correctly ignores it. The brief's statement is wrong on the "no explicit ack" part but the practical consequence (can't rely on it for confirmation) is right.

**Action:** brief correction to Section 5.5 needed on a future version bump. No code change in Unit 2 — Unit 1's confirmation logic stays as-is. Step F regression test must tolerate the ack frame as a normal control message (treat like pong, skip past it).

### 2. `fundingRateTimestamp` field does not exist

Brief 5 §1 says: "storage logic deduplicates on `fundingRateTimestamp` change."

Probe observed funding-related fields in tickers snapshot:

- `fundingRate` ("-0.00005148")
- `nextFundingTime` ("1776758400000", ms epoch)
- `fundingIntervalHour` ("8")
- `fundingCap` ("0.005")

No `fundingRateTimestamp`. The field identifying a funding cycle is `nextFundingTime` — it advances by 8h when a cycle settles.

**Proposed dedup key:** `nextFundingTime`. One row per funding cycle, written when the cycle advances. Matches canonical REST semantics (REST returns one settled value per cycle) and enables apples-to-apples row-count comparison during the 7-day validation.

Alternative considered: dedup on `fundingRate` change (captures intra-cycle estimate drift). Rejected — it's higher volume, and comparison to canonical REST becomes noisy since canonical only stores the settled value.

**Interpretation of "last observed":** the `funding_rate` stored in the shadow row is the final WS value before the cycle advances — this is what should match the REST settled rate. Requires handler to track pending cycle in state and commit on cycle boundary.

### 3. Tickers frame type is explicit

Probe shows `"type": "snapshot"` on frame #0 and `"type": "delta"` on frames #1-5. Brief describes snapshot vs delta semantics correctly; the probe confirms the field is explicit. Handler uses `data["type"]` directly, doesn't need to infer from field coverage.

### 4. OI fields can move independently in deltas

Frame #1 delta contained `openInterestValue` but NOT `openInterest`. The handler must track the two fields in state independently — a delta with one field does not imply the other is absent-by-change.

### 5. `cs` field — topic sequence number

Probe shows a `cs` field (top-level of each frame) that increments monotonically per topic. Not mentioned in the brief. Useful for intra-connection gap detection. **Proposing** we capture it in `open_interest_ws` and `liquidations_ws` for diagnostic value during the 7-day validation, but not make it authoritative (Bybit may reset it per connection — unverified).

### 6. Liquidation payload shape NOT observed

Zero `allLiquidation.BTCUSDT` frames in the 60s probe window. Matches brief's warning that liquidation topics can be silent for minutes in quiet markets.

**Two options for Step A:**

**Option L1 — ship DDL now with best-effort columns + `raw_json` fallback.** Columns match canonical shape (`timestamp`, `symbol`, `side`, `price`, `quantity`). Add a `raw_json TEXT NOT NULL` column capturing the full event payload so we can re-parse if field names are wrong. Handler parsing gets validated against live data during the 7-day window and fixed in Step C if needed, without DB migration.

**Option L2 — block Step A until a liquidation frame is observed.** Extend probe to multi-hour run, or wait for a volatile market session. Resolves the shape question before DDL, but blocks everything else on event timing we don't control.

**Recommendation: L1.** The `raw_json` column is cheap insurance; 7-day validation is already a trust-but-verify window; blocking on external event timing is a schedule risk for no architectural gain.

---

## Proposed DDL

### `funding_rate_ws`

```sql
CREATE TABLE IF NOT EXISTS funding_rate_ws (
    next_funding_time INTEGER PRIMARY KEY,
    symbol TEXT NOT NULL,
    funding_rate REAL NOT NULL,
    funding_interval_hour INTEGER,
    funding_cap REAL,
    received_ts INTEGER NOT NULL
)
```

- `next_funding_time`: ms epoch from `nextFundingTime`. Dedup key. One row per cycle.
- `symbol`: CCXT form `"BTC/USDT:USDT"`.
- `funding_rate`: final observed `fundingRate` before the cycle advanced.
- `funding_interval_hour`, `funding_cap`: contextual metadata, stored explicitly for forensic value.
- `received_ts`: Bybit payload `ts` at the moment the row was committed (cycle-advance moment).

Validation query shape (7-day window):

```sql
SELECT
    fr.timestamp AS rest_ts,
    fr.funding_rate AS rest_rate,
    fw.next_funding_time AS ws_cycle,
    fw.funding_rate AS ws_rate,
    ABS(fr.funding_rate - fw.funding_rate) AS delta
FROM funding_rate fr
JOIN funding_rate_ws fw
    ON fr.timestamp = fw.next_funding_time
WHERE fr.symbol = fw.symbol
ORDER BY fr.timestamp;
```

Open assumption: CCXT's canonical `funding_rate.timestamp` aligns with Bybit's `nextFundingTime` (the settlement epoch). The Step B funding REST verification test confirms this; if the alignment is off by a cycle, the JOIN changes to `fr.timestamp = fw.next_funding_time - 8*3600*1000`. Flagged for Step B to resolve.

### `open_interest_ws`

```sql
CREATE TABLE IF NOT EXISTS open_interest_ws (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ts INTEGER NOT NULL,
    received_at_ms INTEGER NOT NULL,
    symbol TEXT NOT NULL,
    open_interest REAL NOT NULL,
    open_interest_value REAL NOT NULL,
    cs INTEGER
)
```

- `ts`: Bybit payload `ts` (ms).
- `received_at_ms`: server-local receipt time (`time.time() * 1000`). Separate from `ts` per brief approval — lets validation measure WS push frequency and end-to-end latency.
- `open_interest`, `open_interest_value`: ALWAYS non-NULL after the first snapshot — merged from handler's last-known state so every row is a complete record.
- `cs`: topic sequence number for intra-connection gap detection. Nullable (diagnostic, not authoritative).

Write semantics — decision point:

Brief 5 §2: "Stored at every-push resolution — delta-merged against last-known state, one row per update frame." Two readings:

- **R1 (every OI-carrying frame):** write a row only when the delta carries `openInterest` or `openInterestValue`. Skip frames that only update bid/ask/mark. ~1 row per few-seconds at steady-state.
- **R2 (every ticker frame):** write a row for every ticker push regardless of whether OI changed. Much higher volume; 99% of rows are redundant.

**Recommendation: R1.** "Every-push resolution" is in opposition to "every 4h" (the old REST cadence) — it means "whenever the value changes" not "whenever any frame arrives." R2 would bloat the table with duplicates of the last-known values.

Confirm or override — this is load-bearing for data volume.

Index:

```sql
CREATE INDEX IF NOT EXISTS idx_open_interest_ws_ts ON open_interest_ws(ts);
```

### `liquidations_ws`

```sql
CREATE TABLE IF NOT EXISTS liquidations_ws (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp INTEGER NOT NULL,
    symbol TEXT NOT NULL,
    side TEXT NOT NULL,
    price REAL NOT NULL,
    quantity REAL NOT NULL,
    received_at_ms INTEGER NOT NULL,
    cs INTEGER,
    raw_json TEXT NOT NULL
)
```

- Column set mirrors canonical `liquidations` for direct validation JOINs.
- `side` stored lowercased to match canonical (`liquidation_poller.py` does `t["side"].lower()`).
- `raw_json`: full event JSON preserved as-captured. Insurance against field-name guesses in Step C. Can be dropped post-validation once parsing is trusted.
- `cs`: topic sequence number, diagnostic.

Index:

```sql
CREATE INDEX IF NOT EXISTS idx_liquidations_ws_ts ON liquidations_ws(timestamp);
```

Step C will propose the exact parser from the observed allLiquidation payload once a frame has been captured. The `raw_json` column means the table schema does NOT change when the parser lands.

### `ws_reconnect_gaps`

```sql
CREATE TABLE IF NOT EXISTS ws_reconnect_gaps (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    start_ts INTEGER NOT NULL,
    end_ts INTEGER NOT NULL,
    duration_ms INTEGER NOT NULL,
    reason TEXT
)
```

- `start_ts`: ms epoch of the last message received before the disconnect (from `_last_message_ts` max across topics at disconnect time). NOT `time.monotonic()` — this is a wall-clock epoch used for joining against data.
- `end_ts`: ms epoch of the first message received after reconnect (Bybit payload `ts` of that frame).
- `duration_ms`: `end_ts - start_ts`.
- `reason`: disconnect exception class + short message (e.g. "ConnectionError: No pong within 10s of ping").

One row per reconnect. Written by `ws_client.py` after a successful reconnect's subscription is confirmed.

---

## Proposed Store method signatures

Added to `apex/data/store.py` alongside existing `insert_liquidations` / `upsert_funding_rates` / `upsert_open_interest`. Singular (one record) because WS frames arrive one at a time and there is no meaningful batching window at the handler layer.

```python
def upsert_funding_rate_ws(self, record: dict) -> None:
    """Insert or replace one funding-rate cycle row.

    record keys: next_funding_time, symbol, funding_rate,
    funding_interval_hour, funding_cap, received_ts
    """

def insert_open_interest_ws(self, record: dict) -> None:
    """Insert one OI row (every-OI-push).

    record keys: ts, received_at_ms, symbol,
    open_interest, open_interest_value, cs
    """

def insert_liquidations_ws(self, records: list[dict]) -> None:
    """Insert N liquidation events from one WS frame.

    A single allLiquidation frame can carry multiple events;
    batch by frame.

    record keys: timestamp, symbol, side, price, quantity,
    received_at_ms, cs, raw_json
    """

def insert_reconnect_gap(self, record: dict) -> None:
    """Insert one reconnect-gap marker row.

    record keys: start_ts, end_ts, duration_ms, reason
    """
```

Note: `insert_liquidations_ws` takes a list (batch per frame) because one frame can carry many events. The other three take singular because one frame → one row.

---

## Init wiring

DDL entries added to `_MARKET_DATA_TABLES` in `apex/data/store.py`. `init_db` already iterates over that dict (`store.py:78-79`), so no change to `init_db` logic itself. Indexes added alongside the existing `idx_liquidations_ts` creation (`store.py:80-83`).

---

## Decision points needing explicit confirmation

1. **Liquidation shape option L1 vs L2.** Recommendation: L1 (ship now, `raw_json` fallback).
2. **OI write semantics R1 vs R2.** Recommendation: R1 (every OI-carrying frame, not every ticker frame).
3. **Funding cycle alignment assumption** between CCXT `funding_rate.timestamp` and Bybit `nextFundingTime`. Flagged for Step B to verify empirically.

After these three confirm, I'll write the DDL and Store methods into `store.py` as a single commit.

---

## Things NOT in this step

- Handler implementations (Steps C and D)
- ws_client.py changes (Step E)
- Test code (Steps B and F)
- Migration of existing production DB (data accumulates forward from first WS run; no migration needed — tables appear empty until handlers ship)

---

## File-level summary of the proposed store.py diff

Non-destructive, append-only additions:

- Four entries added to `_MARKET_DATA_TABLES` dict
- Two `CREATE INDEX IF NOT EXISTS` statements added in `init_db`
- Four new methods appended to `Store`

No changes to existing DDL, existing methods, or existing index definitions.
