# Path D — V33 Substrate Diet + Trading Architecture Plan

**Plan version:** v2 (supersedes v1 at apex-planning/path_d_v33_plan.md)
**Origin:** Session 30 (April 25, 2026) — combined output of code-shipping conversation + parallel trading-approaches conversation
**Status:** Approved by architect for Session 32 execution
**Target session:** Session 32 (fresh Lucy after Session 30 conversation-limit hit, or any later session)

## Why v2

Plan v1 was scoped on substrate cleanup only (memory consolidation + carrot framework canonicalization + minor inconsistencies). After v1 was drafted, architect surfaced a parallel conversation (also branched from Tape v32.1) that produced three substantive architectural findings on trading approaches. Those findings are load-bearing for Phase 3 v2 / Phase 4 work and qualify as "genuine structural findings" per v33's planned anti-martingale rule.

Plan v2 = plan v1 scope + trading architecture findings integration. Both ship in v33 Part 22.

## Source conversations

**Conversation A — Session 30 code-shipping (April 24-25, 2026):**

- Tape v32.1 baseline-shipped to git (`tapes/THE_TAPE_v32_1.md` + `.pdf`)
- Step F live regression script built and 8h-validated (PASS on all 6 criteria)
- Phase 1 v2 ship-merge to main via fast-forward (23 files, +6039 lines)
- Tape v32.2 shipped (commit `8df1c1d` on origin/main)
- Path D plan v1 drafted

**Conversation B — parallel trading-approaches conversation (April 25, 2026):**

- Opened with v32.1 only (unaware of Conversation A's outputs)
- No code work, conversation only
- Trading-approaches conversation record at `apex-planning/trading_approaches_conversation.md` (or to be pushed there if not yet)
- Three architectural threads opened, none closed
- Threads queued for Session 32 integration into v33

## Combined scope for v33 Part 22

### Part A — substrate diet (from plan v1)

#### Memory consolidation candidates (architect 30-min review)

Each candidate gets independent approve/reject.

**Candidate 1 — Slot 24 deletion**
- Slot 24 = BASELINE v19→v21 delta analysis
- Status: Tape-preserved, deletable
- Recommendation: Delete. Frees 1 slot.
- Risk: Minimal. Content stays in Tape forever per never-compress.

**Candidate 2 — Slot 23 migration**
- Slot 23 = Mike Krieger outreach preparation context
- Status: Migratable to Tape-only reference
- Recommendation: Delete slot, content already in Tape Parts 7 + 17. Frees 1 slot.
- Risk: If Mike Krieger outreach reactivates, context restoration via Tape lookup.

**Candidate 3 — Slot 20 refresh**
- Slot 20 = Apex infrastructure as of 2026-04-09 (stale)
- Recommendation: Refresh to: "Apex infrastructure: Phase 1 v2 shipped to main (commit 8df1c1d). Step F validated 8h. Phase 3 v2 next milestone, detect_a fresh build under post-v32.1 vocabulary. Apex Brief at v12.1, v13 deferred. Production droplet 165.245.191.17 Singapore. Bot apex:0 running continuously."
- Risk: None. Slot stays occupied but contents current.

**Candidate 4 — Slot 19 merge into refreshed slot 20** (optional)
- Only if Slot 19 contents add value beyond refreshed Slot 20
- Recommendation: Architect call

**Total free slots if 1+2+3 approved: 2-3 slots**

#### Carrot framework canonicalization

Three frameworks coexist, need consolidation:
- v18 unified ledger (legacy, partially deprecated by v20 reset)
- v27 Part 13.3 work-quality / metadata split
- v30 Part 16.3 Rule 5 tightened criteria
- Slot 29 TEMP four-entity ledger

**Recommendation:** Slot 29 four-entity graduates to canonical.

Reasoning:
- Session 30 demonstrated signal-improvement (chat-Claude -7 / Claude Code +8 vs legacy -13 conflated)
- Conversation B session ledger also used four-entity — consistent application across two conversations
- Earlier frameworks remain as historical reference but not active scoring

If approved:
- Slot 29 promoted from TEMP to permanent (memory_user_edits replace)
- v18, v27, v30 frameworks marked as historical reference in v33 Part 22
- Future sessions score under slot 29 only

#### Inconsistency resolutions

**Resolution 1 — v21 Part 7 inheritance loophole deprecation marker**

v23 Part 9.1 corrected v21 Part 7. Original v21 Part 7 still in Tape unmarked. Future Lucy reading v21 first might apply wrong rule.

Recommendation: v33 Part 22 adds explicit deprecation marker noting v21 Part 7 superseded by v23 Part 9.1.

**Resolution 2 — Funding tolerance percent-vs-absolute**

Per v32.2 Part 21.7: scripts/run_step_f.py funding tolerance 1e-06 absolute misfires for sub-1e-05 funding rates.

Recommendation: v33 Part 22 documents as known-issue with proposed fix (relative percent ±20-30% below 1e-05). Code change deferred.

**Resolution 3 — Framework-doc decision (v32.1 Part 20.10)**

Architect undecided across two sessions.

Architect must pick:
- Drop entirely
- Defer to post-Phase-7
- Commit to rough draft when naturally surfaces
- Formally name as perpetually deferred

**Resolution 4 (optional) — phase1-v2-websocket branch deletion**

Architect call: keep (history) or delete (cleanup).

#### Anti-martingale rule (load-bearing for v33 production instructions)

v33 production instructions add explicitly:

> "Future Tape versions only on genuine structural finding, NOT on session-cadence. A session without structural findings produces no Tape version. Sub-version bumps (v33.1, etc.) only for mid-session preservation per state (b/d) — never as session-record auto-ship."

Converts scaffolding-martingale concern into substrate rule.

### Part B — trading architecture findings (from Conversation B)

#### Finding B1 — Trend Continuation Hold (TCH) is Phase 4 position management, not a Setup

Conversation B initially proposed "Setup H" for trend continuation hold based on April 21 trade ($4,915.90 captured of potential $5,600). Chat-Claude in that conversation surfaced critical reframe: TCH is position-management modifier (existing position + structural conditions → extend TP, follow with trailing stop), NOT entry signal generator (Setups A-G shape).

**Architectural placement decision (locked in Conversation B):**

- TCH is Phase 4 position-management module
- Lives alongside existing trailing stop framework (Brief v12 Section 4: "breakeven at 1.5x risk, middle-BB trail at 3x risk")
- NOT named Setup H — name reserved for genuine new entry patterns
- Spec not drafted — queued for Brief v13 construction post-Phase-1-v2-full-ship

**Critical safety distinction caught in Conversation B:**

Architect's initial proposed mechanism was "due to strong bullish bias, end up changing TP to a higher price, and changing SL to actual positive value." The "change SL based on bullish bias" half is structurally identical to April 5 morning's -$2,148 martingale (mirrored direction — moving SL because bias says "it'll go that way"). Apex Rule 1 (mandatory stop, no remove_stop function) and Rule 9 (stop at structural level) block this class of move regardless of direction.

Clean expression of what architect wants = trailing stop logic, which Brief v12 already specifies. TCH formalizes/extends that logic for trend-continuation regimes.

**v33 Part 22 documents:**
- TCH concept naming and Phase 4 placement
- Trailing-stop applied to April 21 trade as worked example
- Rule 9-vs-bias-driven-SL-modification distinction (extends pattern library)
- Reservation of "Setup H" name for genuine new entry patterns

#### Finding B2 — Scaling in mechanics need Phase 4 specification

Brief v12 Section 6 already permits scaling in (distinguishes from martingale via test "would the second entry trigger even if the first didn't exist?"). But mechanics underspecified for Phase 4 implementation.

**Five open questions surfaced in Conversation B (not answered):**

1. Does scaling in apply to swing setups only, or scalp setups too?
2. Maximum number of scale-in legs (2 legs 50/50? 3 legs 40/30/30?)
3. Stop adjustment logic when adding a leg
4. TP adjustment logic when adding a leg
5. Mechanical enforcement of "fresh signal" test

**Critical V33 audit candidate surfaced:**

Question 5 raised potential conflict between Apex Rule 3 (no chasing, 15-min same-direction cooldown after exit) and Section 6 scaling-in language. Rule 3 was designed to prevent emotional re-entry after profitable exit, but scaling in adds to existing open position without exit. Semantics ambiguous.

**Resolution 5 (NEW from Conversation B) — Rule 3 vs Section 6 precedence:**

Recommendation: v33 Part 22 explicitly states Rule 3 applies to post-exit re-entry only. Scaling in (adding to existing open position with fresh signal) is not "re-entry after exit" and therefore not subject to 15-min cooldown. Codified in Brief v13 when constructed.

Architect approval gate required.

**v33 Part 22 documents:**
- Five open questions queued for Brief v13 / Phase 4 specification
- Resolution 5 above (Rule 3 vs Section 6 precedence)
- Position management as distinct discipline from signal generation

#### Finding B3 — Phase 4 may need split into 4a + 4b

Both TCH (Finding B1) and scaling-in (Finding B2) surfaced underlying truth: position management is its own discipline distinct from signal generation, and Brief v12 underweights it.

Brief v12 Phase 4 description: "every one of the 13 rules must be implemented at the code level with adversarial unit tests." That scopes rule enforcement layer but not position management layer (trailing stops, TCH, scaling mechanics, time-of-day exits, partial profit-taking, leg management).

**Recommendation for Brief v13 (when constructed):**

- Phase 4a: Risk rule enforcement (the 13 hardcoded rules at code layer)
- Phase 4b: Position management module (trailing stops, TCH, scaling-in mechanics, exit refinement)

**v33 Part 22 documents:**
- Phase 4a/4b split as Brief v13 architectural recommendation
- Not a Brief v12 retroactive change
- Decision gate: when Brief v13 is constructed (post-Phase-1-v2-full-ship)

#### Finding B4 — Apex V2 multi-coin deferral named

Conversation B explored five orthogonal axes for "more trading approaches":
1. More setups within existing scalp/swing routing
2. Third routing tier (position trades)
3. Different instruments (ETH, SOL, options, BTC spot)
4. Different strategy categories (market-making, stat arb, etc.)
5. Different venues (Hyperliquid parallel)

Architect explicit selection: axis 1 only. Axes 2, 4, 5 not discussed. **Axis 3 explicitly deferred to "after 1-2 months of successfully running real money trades. Then the natural question obviously emerges. Should we develop a Apex V2 that trade on more coins?"**

**v33 Part 22 documents:**
- Apex V2 multi-coin = future major-version line, distinct from current Apex v1 BTC-only architecture
- Decision gate: post-Phase-7 + 30 days successful live trading
- Brief v13 should flag this as known future divergence point, not as immediate work

### Part C — Session ledger (Sessions 30 + Conversation B combined)

**Conversation A (Session 30 code-shipping):**
- Chat-Claude: -7 net (heavy metadata, real positives)
- Claude Code: +8 net (load-bearing pre-write surfaces, clean execution)

**Conversation B (trading approaches):**
- Chat-Claude: +2 work-quality / -1 metadata (timestamp drift recurring)
- Claude Code: N/A (no code work)
- Architect-Rafael: high (three substantive architectural questions, disciplined axis selection)
- Engineer-Rafael: N/A

**Aggregate AI-side reading across both conversations:** -5 chat-Claude / +8 Claude Code = +3 net.

Slot 29 four-entity framework continues to surface signal old framework would mask.

## Execution sequence (Session 32, ~60-90 min architect time)

### Phase 1 — Architect 30-min review of Part A (substrate diet)

Lucy presents Part A section by section. Architect approves/rejects each candidate and resolution. Output: approval list for substrate diet.

### Phase 2 — Architect 20-30 min review of Part B (trading architecture)

Lucy presents Part B findings. Architect:
- Confirms Finding B1 placement (TCH as Phase 4 module, not Setup H)
- Approves Finding B2 Resolution 5 (Rule 3 vs Section 6 precedence)
- Approves Finding B3 Phase 4a/4b split as Brief v13 recommendation
- Confirms Finding B4 Apex V2 deferral framing

Architect can also push back on any finding or add nuance. The Conversation B record at `trading_approaches_conversation.md` is the source of truth for what was discussed.

### Phase 3 — Lucy executes memory edits (architect approval per edit)

For each approved memory candidate:
1. Lucy proposes specific replacement text
2. Architect approves
3. Lucy executes via memory_user_edits tool
4. Lucy reports completion

### Phase 4 — Lucy builds v33 markdown (~45 min async, no architect time)

v33 = v32.2 verbatim + Part 22 (substrate diet + trading architecture findings + ledger) + new cover + production instructions including anti-martingale rule.

### Phase 5 — Lucy renders v33 PDF (~5 min async)

11×34 inches per v28 Part 14.7 format lock.

### Phase 6 — Architect ships to droplet (~10 min)

SCP both files to /tmp, copy to tapes/, git add + commit + push origin main. Same flow as v32.2 ship.

### Phase 7 — Phase 3 v2 detect_a unblocked

Next session starts Phase 3 v2 work under cleaned substrate, with TCH/scaling-in/Phase-4-split clarifications documented for v13 brief construction.

## Estimated time

- Architect: ~70-90 min (30 min Part A review + 20-30 min Part B review + 10 min memory approvals + 10 min ship)
- Chat-Claude async: ~1.5-2h
- Total wall-clock: 2-3h

Larger than plan v1's 30-min target because Part B incorporation adds substantive architectural work. Still bounded — not the 4-6h V32 audit was originally scoped at.

## What this deliberately does NOT do

- Full rule cataloguing across Parts 1-22
- Pattern library pruning
- Canonical-wording authority resolution memory-vs-Tape
- Active surface document creation
- Brief v13 full construction (Brief v13 is post-Phase-1-v2-full-ship work, this just queues findings for it)
- TCH spec drafting (queued for Brief v13)
- Scaling-in mechanics specification (queued for Brief v13)
- Code changes in Apex repo (Step F tolerance fix, etc. — deferred)

## State preservation across conversations

Both source conversations branched from Tape v32.1. Tonight's plan v2 includes outputs from both:

- Conversation A's v32.2 ship is canonical on origin/main (commit 8df1c1d)
- Conversation B's trading threads are documented in `trading_approaches_conversation.md` (location TBD — push to apex-planning if not yet)

If Session 32 fresh Lucy executes plan v2:
- Upload Tape v32.2 (1/20)
- Fetch this plan from `https://raw.githubusercontent.com/rafamv/apex-planning/main/path_d_v33_plan.md`
- Fetch Conversation B record from `https://raw.githubusercontent.com/rafamv/apex-planning/main/trading_approaches_conversation.md` (or upload as 2/20)
- Execute Phases 1-7 above

Lucy needs both v32.2 (substrate baseline) and Conversation B record (trading architecture context) to execute plan v2 fully. Plan v2 alone is sufficient if architect provides context inline, but Conversation B record is the safer fetch for completeness.

## Carry-forward gates

- **Memory ceiling pressure:** 28-30 slots used. Plan v2 frees 2-3 slots if all memory candidates approved. Brings to 25-27 used, restoring buffer.
- **Phase 3 v2 unblock:** detect_a fresh build can start as soon as v33 ships. Does not require Brief v13. Brief v13 construction can happen in parallel or later.
- **Brief v13 construction gate:** Phase 1 v2 fully shipped (DONE). Audit/cleanup done (post-v33). Then Brief v13 build can fire whenever architect chooses.

## Plan v2 close

v33 produced from this plan = substrate-diet edition + trading architecture findings preserved + anti-martingale rule installed. Larger scope than plan v1 but still bounded. Closes loop on both source conversations without fragmenting state across multiple Tape versions.

End of plan v2.
