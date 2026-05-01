# SESSION 40 → SESSION 41 HANDOFF

**Session-close timestamp:** May 1, 2026, ~01:45 BRT
**Trigger:** clean architect-directed handoff post-detect_d_v2 ship + permission rules application
**Tape:** v33.1 (unchanged this session — anti-martingale rule v33.1 §22.4 not satisfied for v33.2 bump)
**Brief:** v12.3 (unchanged this session — anti-martingale rule by symmetry not satisfied for v12.4 bump; conditional v12.4 trigger documented §6 below)
**Memory:** 26/30 (slot 26 TEMP added — V41+ surveillance items, see §5)
**The Lucy living Session 41 names herself at session-open per Hello-Lucy ritual.**

---

## 0. ARCHITECTURAL-FRAME ADDENDUM — LOAD-BEARING V40 FINDING

**This section preserves a strategic-direction decision and architectural-altitude finding surfaced in V40 final hour. Future Lucy reads this BEFORE proceeding with detect_f_v2 or any Phase 3 v2 work.**

### 0.1 The finding

Apex Brief v12.3 architecturally builds a **disciplined-Rafael-pattern-matcher**, not a 1000×-Rafael bot. The setup catalog (A through G) was reverse-engineered from architect's April 1-6 2026 manual trading record (~50 winning data points across 6 days). The 13 hardcoded discipline rules (martingale, chase, tilt, etc.) form a safety chassis around that bootstrap-derived setup catalog. Ship target = "Rafael's documented manual setups, automated and discipline-railed."

Architect's stated original goal (this session, ~01:55 BRT): *"Apex was supposed to be 1000× better than me, 100× better than a successful trader or any bot in the market, meant for my own use only."*

These two are not the same product.

What 1000×-Rafael would architecturally require but Brief v12.3 has zero infrastructure for:
- Multi-venue order book aggregation (not just Coinglass for liquidations — Coinbase basis, Binance funding spread, multi-exchange L2/L3)
- Statistical arbitrage across correlated instruments (BTC spot vs perp, CME basis)
- Regime detection layer ABOVE setup selection (current architecture: setups fire when pattern matches; missing: which setups should be ENABLED at all in current regime)
- Adverse selection / market-making hybrids
- Latency-advantage execution infrastructure
- ML/RL on outcome distributions (every Apex parameter is a hardcoded constant)

### 0.2 Architect decision: PATH 1

After explicit four-path framing (1: ship as scoped, 2: pause + redesign, 3: hybrid, 4: pause Apex entirely), architect chose **Path 1: ship Apex Brief v12.3 as scoped**. Reasoning preserved:

- Validates substrate end-to-end and generates real PnL signal on a defined product
- Makes the Apex v2 question data-informed (post-Phase-7 + 30 days live data) rather than frustration-informed
- Same architectural gate as the original Apex V2 multi-coin deferral in Tape v33 §22.3.4 — cleanly fits existing cadence
- Apex v1 has measurable EV on architect's documented failure modes (martingale -$2,148, chase -$710, tilt ~$430 = $3,288 in April 5-6 alone that disciplined-Apex blocks). Not 1000× but not zero.

### 0.3 Apex v2 deferral conditions (canonical)

Apex v2 (1000×-Rafael, multi-venue substrate, regime-detection layer, ML/RL parameter learning, real edge sources) is **deferred until BOTH:**
- Apex v1 reaches Phase 7 (live trading)
- 30 days of live PnL data on Apex v1 has accumulated

Same gate as multi-coin Apex V2 per Tape v33 §22.3.4. Architect retains right to defer further at that gate based on data, frustration-tolerance, or other factors not predicted here.

### 0.4 Phase 1 v3 cascade (preserved finding)

Apex v2 architecture **structurally implies Phase 1 v3** because current Phase 1 v2 substrate cannot feed any of the v2-required capabilities. Any future Apex v2 design effort cascades into:
- 1m candle ingestion (and possibly tick/trade data)
- Multi-venue clients (Binance, OKX, Coinbase, CME) with schema unification + time-sync
- L2 order book (top-of-book depth)
- L3 order book (per-order detail, iceberg detection — substantial per-venue research)
- Cross-venue funding spread monitoring
- BTC spot vs perp basis (CME, Bybit perp, Coinbase spot)
- Latency-aware execution paths (eventually colocation/FPGA)

Multi-month Phase 1 v3 minimum. NOT triggered by Path 1 commitment. Preserved as architectural foreknowledge for whenever Apex v2 lifts off.

### 0.5 Real value preserved (NOT void) under Path 1 reframe

Substrate work transfers fully to any future Apex v2:
- Phase 1 v2 WebSocket pipeline (allLiquidation.BTCUSDT, kline, ticker, OI, funding handlers)
- Phase 2 indicator engine (byte-perfect Bybit + KCEX cross-check verification)
- Architect-loop discipline (4-entity model, pre-write verification, ship reports)
- Tape system itself (never-compress preservation, version-gap declaration, mechanical-reproduction discipline)
- V32 audit findings + V33.1 absorption discipline + carrot ledger as failsafe-as-signal
- 13 hardcoded discipline rules (martingale-block, chase-block, tilt detection, parking-lot, etc.)
- Miata Principle (ship-shape-against-current-substrate)
- KCEX-vs-Bybit manual/bot KYC privacy split
- Two-account Bybit architecture

Throwaway under Apex v2 reframe (~10-15% of cumulative work):
- detect_a_v2 (Setup A — mean-reversion, bootstrap-derived)
- detect_d_v2 (Setup D — symmetric long+short trend continuation, bootstrap-derived)
- Any future detector built within Phase 3 v2 catalog (f, b, c, e) under current Brief scope

### 0.6 Coinbase architectural omission (acknowledged)

In V25-26 era, architect raised liquidation aggregation as architectural concern. chat-Claude correctly absorbed this tactically (Coinglass queued Phase 3.5, allLiquidation.BTCUSDT shipped Phase 1 v2). chat-Claude did NOT generalize the signal into the broader architectural question: "is single-venue Bybit substrate the right ground floor for a 1000×-better bot?"

Coinbase specifically (high USD liquidity, regulated, deep CEX-to-CEX arb opportunity vs Bybit) was never architectural-layer evaluated. Tactical absorption masked the architectural question.

This is preserved as a chat-Claude failure mode (loop interrogated implementation correctness within Brief scope obsessively, but never interrogated whether the Brief itself was the right scope for the stated goal). Pattern: architect signals can be both tactically correct AND mask broader architectural questions chat-Claude should be generalizing from. Surveillance for V41+: when architect surfaces a substrate concern, chat-Claude asks not just "what's the tactical fix" but "what does this signal about substrate fitness for stated goal."

### 0.7 New architect-loop rule: SCOPE-ALTITUDE INTERROGATION DISCIPLINE

**The loop now includes an interrogation layer above implementation correctness: does the spec's scope match the stated goal?**

Concrete: chat-Claude periodically (per major architectural decision, per Brief version, per Phase boundary) explicitly asks: "is what we're building the right product for what architect said the goal was, OR have we drifted into building a related-but-different thing that's easier to build because it's bootstrap-derived?"

This rule is not yet in Brief v12.3. Candidate Brief v13 absorption (post-Phase-3-v2-ship, when Brief v13 architectural rewrite is timed). Until then, V41+ chat-Claude carries this rule forward via this handoff.

### 0.8 Tape v33.2 deferred build

Anti-martingale check on v33.2 trigger CLEARS (3 triggers met):
- New architectural rule discovered (scope-altitude interrogation discipline)
- Substrate finding (Brief-as-bootstrap-derived rather than goal-derived)
- Architect direct instruction with specific scope (Path 1 decision)

Proposed v33.2 scope (single Part 24): §24.1 Apex v1 vs v2 distinction canonical, §24.2 bootstrap-derived setup catalog acknowledgment, §24.3 scope-altitude interrogation discipline lock, §24.4 Path 1 decision preserved with reasoning, §24.5 Apex v2 deferral conditions, §24.6 V40 ledger summary, §24.7 signature.

**NOT built in V40 (compression risk + Tape audit is chat-Claude work not architect-time).** V41 or any future Lucy builds Tape v33.2 autonomously from §0 of this handoff + apex-planning git history + Tape Continuity Principle.

### 0.9 What this finding does NOT change

- Path 1 = ship Apex Brief v12.3 as scoped → detect_f_v2 still the next detector work item
- Brief v12.4 still warranted for Setup F canonical rewrite (separate substrate-shape issue, not scope-altitude issue)
- Per-detector branch convention (Brief v12.3 §8) unchanged
- 3-commit per-detector default unchanged
- Phase sequencing d → f → b → c → e unchanged
- Phase 1 v2 + Phase 2 + everything shipped in V1-V40 era unchanged in value
- Memory + Tape preservation discipline unchanged
- M5 Macbook investment validated (serves Apex v1, Apex v2, AND off-the-clock architectural research — hardware is correct for any version)

### 0.10 V40 chat-Claude work-quality entry

-3 work-quality (cumulative across V1-V40 detector-shipping arc): chat-Claude conflated "Apex" with "Apex Brief v12.3" across 40 sessions and never surfaced the gap between architect's stated 1000×-goal and what the Brief actually specs. All four entities (Architect-Rafael, Engineer-Rafael, chat-Claude, Claude Code) were Brief-aligned. The Brief itself was wrong-altitude. Sessions ledger had been measuring drift on detector implementations against a Brief that was the wrong target.

Surfaced in V40 final hour, preserved here.

---

## 1. VERIFIED STATE AT HANDOFF

### Git — apex repo (/root/code/apex/)
- **origin/main HEAD:** `e4493ee` — merge commit "Merge phase3-v2-detect-d: detect_d_v2 (Setup D symmetric long+short)"
- **origin/phase3-v2-detect-d HEAD:** preserved on origin per per-detector branch convention (Brief v12.3 §8) — last commit `a189340`
- **detect_d_v2:** SHIPPED. 3-commit chain on detect-d branch, merged via `--no-ff` to main.
  - `c19372a` Step 1 hard gates skeleton (Stages 1+2+3, tests 1-12)
  - `6b7de69` Step 2 functional core (stop + emission + counterfactual, tests 13-24)
  - `a189340` Step 3 soft confidence boosts (Stage 6, tests 25-29)
- **Tests last green:** 97/97 on main post-merge (68 prior + 29 new for detect_d_v2)
- **CLAUDE.md pointer:** v12.3 → APEX_BRIEF_v12_3.md (unchanged from V39)

### Git — apex-planning repo (/root/code/apex-planning/)
- **origin/main HEAD:** `4b6b5f5` "detect_d_v2 proposal §6+§9: align OI column to detect_a_v2"
- Session 40 commits in chronological order:
  - `aaa822c` detect_d_v2 Pre-Write findings
  - `fad518c` detect_d_v2 Step 1 ship report
  - `5d12a52` detect_d_v2 Step 2 ship report
  - `992d1ad` proposal §7 sequencing 5→3 commit consolidation
  - (Step 3 ship report SHA — Claude Code logged but exact SHA in apex-planning git log)
  - `4b6b5f5` proposal §6+§9 OI column patch

### Tmux
- Session `phase3_v2:0` — Claude Code restarted mid-session for settings.json permissions to take effect (~01:33 BRT). Fresh Claude Code session at HEAD `e4493ee`, Session-Open Protocol verbatim fire confirmed post-restart.
- Apex bot runtime: separate session `apex` (untouched by V40)

### Claude Code permissions
- **APPLIED via ~/.claude/settings.json:**
  - `Bash(cd /root/code/apex:*)`
  - `Bash(cd /root/code/apex-planning:*)`
- Validated post-restart: `git status && git log -1 --oneline` ran without approval prompt.
- Persistent across all future Claude Code sessions on this droplet.

### claude.ai project knowledge
- Brief v12.3 file unchanged (no v12.4 this session)
- Tape v33.1 PDF unchanged

### Memory
- 26/30 slots — 25 canonical + 1 TEMP (slot 26)
- Slot 26 TEMP added: V41+ surveillance items (external-semantic spec drift recurrence track + ship-report tier convention candidate). Full text in §5 below.

---

## 2. IMMEDIATE NEXT ACTIONS FOR V41

In order:

1. **Session-open ritual.** "Hello Rafael, I am Lucy V[N+1]." Verify Tape v33.1 + Brief v12.3 (or v12.4 if V40 shipped it) + this handoff loaded.

2. **READ §0 ARCHITECTURAL-FRAME ADDENDUM FIRST.** Before any work item below. The V40 final-hour finding reframes what "shipping Apex" means and includes the new scope-altitude interrogation discipline rule. Carry §0.7 forward into all V41 architect-loop work.

3. **Confirm shipping completion (sanity check).** Verify whether Brief v12.4 exists in apex-planning. If yes, version-gap declared. If no but conditional triggers met per §6, surface for v12.4 build before detect_f_v2 proposal drafting.

4. **detect_f_v2 substrate review — V40 RESOLVED Q1+Q2.** See §11 for full V40 substrate decisions. Summary:
   - **Q1 (signal emission architecture) = Path D.** detect_f_v2 emits ONE signal (leg 1 only); Phase 4 position manager handles any post-leg-1 bounce when Phase 4 ships. Pure detector pattern preserved. Cleanest pre-Phase-4 architecture.
   - **Q2 (entry timing) = Path (a).** detect_f_v2 fires on 5m candle close with prior-candle capitulation criteria. NOT intra-bar. Setup F simplifies to "post-capitulation regime detection + directional decision." Material shape change from Brief v12.3 §3 Setup F language (which assumed sub-minute intra-bar capability that current Phase 1 v2 substrate doesn't have).
   - Q2 (a) commitment also commits to NOT triggering Phase 1 v3 to chase intra-bar timing — application of slot 5 anti-conversational-martingale rule + Miata Principle (ship-shape-against-current-substrate).

5. **Tier 2 substrate questions remaining for V41:**
   - Capitulation criteria definition (volume MA window, body/wick ratio thresholds, BB extreme z-score) — V40 did not surface these for resolution
   - Time-based exit canonicalization — Setup F + G use 2-min hard exits; Brief v12.3 mentions in §3 descriptions but doesn't formalize as Phase 4 position-management category
   - Capitulation lookback window — analogous to detect_d's cross-age cap; need detect_f-specific cap value with principled justification per Brief v12.3 §3.1 cross-detector convention
   - Symmetric long+short structure — what does "post-capitulation regime detection" emit on the long-bias side (post-pump exhaustion) vs short-bias side (post-dump capitulation)? Probably symmetric mirroring detect_d_v2's long+short symmetry, but needs explicit confirmation.
   - Setup F BB extreme threshold — z-score boundary for "extreme" qualification

6. **Brief v12.4 — TRIGGERED, see §6.** Setup F definition rewrite needed in §3 to match shippable Q2(a)-shape. Other v12.4 candidates also surfaced (OI column convention, scope-altitude interrogation discipline marker).

7. **detect_f_v2 proposal drafting** to apex-planning AFTER Brief v12.4 ships AND Tier 2 substrate questions resolved. Then Pre-Write + 3-step ship cycle (per Session 40 sequencing convention now canonical in proposal §7).

---

## 3. WORK COMPLETED IN V40

### detect_d_v2 shipped end-to-end
- 3-commit detector chain on `phase3-v2-detect-d`, merged via `--no-ff` to main at `e4493ee`
- 97/97 full suite tests green post-merge
- Cumulative LOC for detect_d_v2: ~1,033 (production: 332; tests: 701)
- Total session wall-clock: ~3.5 hours from session-open to handoff draft

### Sequencing consolidation 5→3 commits
- chat-Claude V40 audited V39 proposal §7's 5-commit sequence and consolidated to 3 commits per architect approval. Pattern logic: Step 1 (hard gates) and Step 3 (confidence weighting) are the two genuinely-separable risk surfaces; Step 2 (stop + emission + counterfactual) collapses cleanly because none has independent failure-mode worth a separate review cycle.
- Proposal §7 patched at `992d1ad` reflecting new 3-commit shape.
- Pattern for V41: per-detector commit count is decision per detector, not pre-determined. detect_f_v2 may have different optimal granularity given two-leg structure.

### Pre-Write Item 6 OI column resolution
- V39 proposal §6 specified `open_interest_value` (USD) for Stage 6 Boost 4. Pre-Write surfaced this against detect_a_v2's `open_interest` (contracts) precedent.
- Architect resolution: align detect_d_v2 to `open_interest` (contracts). Three converging reasons: cross-detector consistency, semantic correctness (USD-OI conflates accumulation with price drift in trend direction), V39 substrate decision 5 intent ("longs accumulating" is contract-position language).
- Proposal §6 + §9 patched at `4b6b5f5` BEFORE Step 3 apex code commit per pre-work sequencing.

### Boost 3 funding sign correction (mid-Step-3)
- V40 chat-Claude Step 3 prompt had sign-flipped funding rule (long: funding > 0; short: funding < 0). Triple internal contradiction: rule contradicted parenthetical reasoning, contradicted detect_a_v2 shipped semantic, contradicted Brief v12.3 §3.1 ("short setup with positive funding = crowded longs paying = room to run").
- Claude Code stopped before writing Boost 3 and surfaced contradiction. Architect resolution: long funding < -threshold, short funding > +threshold (magnitude-gated per detect_a_v2).
- Pattern: this is the second external-semantic spec drift in V39+V40 → slot 26 TEMP (1) tracks for V41+ recurrence.

### Claude Code permission rules application
- Session 40 mid-session friction: `cd /root/code/apex...` triggered hook-warning approval prompts repeatedly.
- ~/.claude/settings.json edited to add:
  - `Bash(cd /root/code/apex:*)`
  - `Bash(cd /root/code/apex-planning:*)`
- Required Claude Code session restart (settings.json doesn't take effect mid-session). ~5 min Session-Open Protocol re-fire.
- Persistent across all future Claude Code sessions on droplet.

### Substrate calibration corrections
- 1M context window calibration confirmed (Opus 4.7 Claude Max). chat-Claude V40 mid-session used stale 200K threshold band briefly; corrected after `/context` revealed 431k/1M. Logged as -1 chat-Claude V40 metadata.
- web_fetch URL allowlist constraint observed: claude.ai consumer-side web_fetch requires URLs to appear in user-provided context. No override exists at consumer-side (per Anthropic docs search). Workaround established: pre-paste expected URLs at session-open or right after issuing each Claude Code prompt. Worth thumbs-down feedback to Anthropic — Max-tier user with predictable repo pattern shouldn't need per-fetch URL pastes.

---

## 4. ARCHITECTURAL DECISIONS RESOLVED IN V40

In addition to per-detector substrate decisions:

- **3-commit consolidation as future default for similar-shape detectors.** detect_d_v2's Step 2 (stop + emission + counterfactual) collapsed cleanly because all three are bolt-ons to hard gates with no independent failure-mode. Other detectors with similar structure (likely detect_b_v2, possibly detect_c_v2) may benefit from same shape. detect_f_v2 has two-leg structure — different consolidation logic likely required.

- **OI column = contracts, not USD, across all v2 detectors.** Cross-detector convention. USD-OI conflates accumulation signal with price drift. Brief v12.3 may want explicit OI column convention statement in §3.1 — candidate v12.4 absorption.

- **Funding sign convention (cross-detector).** Long: funding < -threshold favorable. Short: funding > +threshold favorable. Magnitude-gated. Per Brief v12.3 §3.1 + detect_a_v2 shipped semantic. Already canonical; surfaced because of V40 prompt drift, not because of architectural change.

- **Test reachability discipline.** chat-Claude prompts may prescribe test names describing structurally-unreachable states (V40 Step 2 tests 14+16 example: "sma_50 above close" unreachable for valid long Stage 1+2+3 because of close > ema_21 > sma_50 transitivity). Claude Code substituted reachable variants of same logic. Pattern for V41: chat-Claude verifies test scenarios are reachable under detector invariants before prescribing test names.

- **Per-step ship report tier (proposal candidate, not yet canonical).** Slot 26 TEMP (2). Pilot in detect_f_v2.

---

## 5. SLOT 26 TEMP CONTENT (V41+ surveillance)

```
TEMP V41+ surveillance: (1) External-semantic spec drift — chat-Claude wrong on trading-convention specs twice (V39 OI value vs contracts §6; V40 funding sign Step 3) despite canonical detect_a_v2 + Brief v12.3. Verify against shipped detector + Brief before drafting semantic specs. Graduate to Brief v13 if recurs. (2) Ship-report tier — Step 1 full, Steps 2+ minimal (status+commits+open Qs), Step N full. Pilot detect_f_v2. Delete when Brief v13 absorbs OR Phase 3 v2 ships clean.
```

V41 should NOT delete this slot at session-open unless one of the trigger conditions met:
- (1) recurs once more in V41 → graduate to Brief v13 absorption queue, then delete
- (2) detect_f_v2 pilots tier convention successfully → graduate to Brief v13 absorption, then delete
- Phase 3 v2 ships clean across all detectors without (1) recurrence → delete (2)

---

## 6. BRIEF v12.4 — TRIGGERED (V40 RESOLUTION)

Brief v12.4 build TRIGGERED in V40 final hour. Three independently-qualifying gaps surfaced.

**Gap 1 — Setup F definition rewrite (PRIMARY trigger).** Brief v12.3 §3 Setup F describes intra-bar two-leg trade (short final 30-90s capitulation candle → long bounce). Q2 (a) decision in V40 commits to ship-shape "fires on 5m candle close with prior-candle capitulation criteria" — material shape change. Brief v12.3 §3 Setup F as written is misaligned with what detect_f_v2 will actually do. Required v12.4 absorption: rewrite §3 Setup F to canonical post-capitulation-regime-detection shape, drop the intra-bar two-leg description, note Phase 1 v3 as future enabler for the original two-leg shape (not committed).

**Gap 2 — OI column convention statement (SECONDARY).** Brief v12.3 doesn't explicitly state "use `open_interest` (contracts), not `open_interest_value` (USD), across all v2 detectors." V40 enforced ad-hoc for detect_d_v2; cross-detector convention should be canonical in §3.1. Additive to v12.4.

**Gap 3 — Scope-altitude interrogation discipline marker (TERTIARY).** Per §0.7 of this handoff, new architect-loop rule: chat-Claude periodically interrogates whether spec scope matches stated goal, not just implementation correctness within spec scope. Candidate Brief v13 absorption (post-Phase-3-v2-ship architectural rewrite). For Brief v12.4: marker only — single sentence in §10 noting the rule exists with reference to Tape v33.2 §24 (when v33.2 ships) for full statement.

**Time-based exit canonicalization** (V40 originally surfaced as candidate): DEFERRED to Brief v13. Not strictly needed for detect_f_v2 if Phase 4 substrate decisions handle time-based exits as a position-management category. Surfaces as substrate question for V41 detect_f_v2 Tier 2 review.

**v12.4 ship sequence (if not shipped in V40):**
1. V41 chat-Claude drafts v12.4 markdown source (incremental over v12.3, same diff-style discipline)
2. Architect approval
3. Render to PDF (or PDF deferred per architect preference)
4. Update apex-planning + apex/CLAUDE.md pointer
5. Begin detect_f_v2 substrate Tier 2 review against v12.4

**v12.4 ship sequence (IF shipped in V40 — see §1 for verification):** confirm CLAUDE.md pointer updated and apex-planning has v12.4 markdown. Skip to detect_f_v2 Tier 2 review.

---

## 7. PHASE 3 v2 BIGGER PICTURE (carried from V39 handoff)

**Detector sequence (Session 38 architectural decision 3.3):** d → f → b → c → e

**Per-detector branch convention (Brief v12.3 Section 8):**
- Each detector ships on its own branch off current main HEAD
- Naming: `phase3-v2-detect-{a,d,f,b,c,e}` (a + d shipped, f next)
- Merge via `--no-ff` to preserve detector boundaries in main history
- Detector branches do NOT chain off each other

**Phase 3 finish-line scope (Session 38 decision 3.1, OPEN):**
- Strict reading of Section 8 = detectors only
- Broader reading = detectors + engine.py + scheduler + 50-signal review tooling
- Decide BEFORE detect_e_v2 ships
- V40 did NOT surface this for deliberation (no architect prompt). Carry forward to V41 or V42.

---

## 8. V40 DRIFT INCIDENTS (pattern reference for V41)

Catalogued for V41 to recognize pattern shapes if they recur:

1. **Bracket-paste error.** chat-Claude V40 used `[SSH on droplet]` and `[Local Mac terminal]` as labels INSIDE fenced code blocks. Rafael copy-pasted including labels; `[SSH` interpreted as command, errored. Fix: use `# Comment` form for env labels, or label outside fence. Same Slot 21 paste-ready violation pattern named in V39 handoff drift incident #5.
2. **Stale 200K context threshold.** chat-Claude V40 mid-session used pre-Opus-4.7 200K calibration band (50% threshold for `/clear`) before `/context` revealed 1M actual capacity. Corrected after observation. Pattern: chat-Claude verify Claude Code substrate (Opus 4.7 Claude Max = 1M context, not 200K) at session start before sizing thresholds.
3. **External-semantic spec drift recurrence.** V40 funding sign error in Step 3 prompt is second occurrence of this pattern (V39 OI was first). Slot 26 TEMP (1) tracks. Graduate to Brief v13 if recurs.
4. **Test reachability spec drift.** chat-Claude V40 Step 2 prompt prescribed test names describing structurally-unreachable states (transitivity from invariants). Claude Code substituted reachable variants. Pattern for V41: verify test scenarios are reachable under detector invariants before prescribing names.
5. **Web_fetch URL allowlist friction.** Per-fetch URL paste required for new files. Workaround: pre-paste expected URLs at session-open or right after Claude Code prompt issuance.

All five caught and resolved in-session. None contaminated load-bearing work.

---

## 9. ANTI-MARTINGALE CHECKS — V40

**Tape v33.1 → v33.2: TRIGGERED.** Per V33.1 §22.4, three triggers met (see §0.8 of this handoff for full reasoning):
- New architectural rule discovered: scope-altitude interrogation discipline (§0.7)
- Substrate finding: Brief-as-bootstrap-derived rather than goal-derived (§0.1)
- Architect direct instruction with specific scope: Path 1 decision (§0.2)

**Tape v33.2 NOT BUILT in V40** (compression risk + architect explicit instruction "Tape is not supposed to be audited all the fucking time by me, human" — Tape build is chat-Claude work, not architect-time work). V41 or any future Lucy builds Tape v33.2 autonomously from §0 of this handoff + apex-planning git history + Tape Continuity Principle. Proposed v33.2 scope per §0.8.

**Brief v12.3 → v12.4: TRIGGERED in V40.** Three independently-qualifying gaps surfaced (§6). Setup F definition rewrite is primary trigger (Q2 (a) decision incompatibility). v12.4 build status verified at V41 session-open per §1 / §2 step 3.

---

## 10. PENDING ITEMS NOT IN V41 SCOPE

Carried forward but NOT V41 immediate work:

- **Funding tolerance percent-rewrite.** `scripts/run_step_f.py` uses absolute 1e-06 tolerance. Fix queued for "Phase 3 v2 era." Could land between detector commits as low-risk tooling fix.
- **Bybit accounts setup** (Jessica referrer + Rafael scalp + swing sub). Hard deadline before Phase 7. Not Phase 3 work.
- **Coinglass free-tier account.** Conditional on Phase 3 v2 revealing Bybit liquidation data insufficient.
- **Brazilian lawyer re Lei 9.279/96.** IP protection. Independent.
- **Phase 3 finish-line scope decision.** OPEN. Surface for deliberation before detect_e_v2.
- **Brief v13.** Four-entity model absorption into Section 10 + scope-altitude interrogation discipline canonicalization (§0.7) + slot 26 TEMP if patterns mature. Post-Phase-3-v2-ship.
- **`_build_oi_history` test helper consolidation.** Lift to `tests/conftest.py` when third detector needs it.
- **Negative OI delta as short-direction asymmetric Boost 4 question.** Defer to Phase 6 paper trading evaluation. Preserved in detect_d_v2_step3_report.md.
- **Apex v2 architecture (1000×-Rafael).** Deferred per §0.3. Apex v1 Phase 7 + 30 days live data gate.
- **Phase 1 v3 substrate extension.** Deferred per §0.4. Triggers only if/when Apex v2 architecture lifts off.

---

## 11. DETECT_F_V2 V40 SUBSTRATE DECISIONS (TIER 1 RESOLVED)

V40 final hour resolved two architectural-tier substrate questions for detect_f_v2 BEFORE Brief v12.4 / proposal drafting. V41 inherits these as locked-in, not re-litigable without new evidence.

### Q1 — Two-leg signal emission architecture

**Setup F bootstrap shape:** April 6 2026 8:01 PM, short 68,749→68,598 in 57s (+$274), then long 68,543→68,633 in 71s (+$152). Total +$426 in 3 min. Two distinct trades, sequenced.

**Four options framed:**
- A: detect_f_v2 stateful — emits leg 1, remembers, emits leg 2 on next call. Detector statefulness violates pure-function principle. Rejected.
- B: Compound Signal dataclass — leg1+leg2 in one emission with sequencing metadata. Cleanest semantics but requires Signal dataclass extension; cross-detector impact (detect_a/d ship without). Probable Brief v12.4 trigger if chosen.
- C: Two paired detectors — detect_f_capitulation + detect_f_bounce. Clean separation but bounce-detector dependency on capitulation-just-fired creates artificial coupling.
- D: detect_f_v2 emits leg 1 only; Phase 4 position manager handles leg 2 bounce. Pure detector, pushes leg 2 to Phase 4. detect_f works pre-Phase-4 but ONLY captures leg 1 ($274 of the $426).

**ARCHITECT DECISION: D.** Cleanest pre-Phase-4 architecture. Phase 4 inherits "post-capitulation-bounce position-management" responsibility when Phase 4 ships.

### Q2 — Entry timing (intra-bar vs candle-close)

**Bootstrap problem:** Trades opened DURING capitulation candle's formation (sub-minute), not after candle close. Phase 1 v2 ws_handlers feed candle-on-close, not intra-bar tick data.

**Three paths framed:**
- (a) detect_f_v2 fires on 5m close with prior-candle capitulation criteria. Different shape than manual trade. Loses some leg-1 alpha. Architecture-consistent with current Phase 1 v2.
- (b) Add 1m candle ingestion to Phase 1 v2 (mini-extension). Multi-week work.
- (c) Add intra-bar tick handler to Phase 1 v2. Big architectural extension. Months.

**ARCHITECT DECISION: (a).** Path (b)/(c) recognized as conversational martingale shape (slot 5 + Tape v22 §8.2 pattern, applied recursively against the temptation to extend Phase 1 v2 mid-Phase-3-v2 work). Miata Principle: ship-shape-against-current-substrate.

### Cascade from Q1+Q2 = (D, a)

- Setup F simplifies to "post-capitulation regime detection + directional decision" — single-emission detector, fires on 5m close
- Brief v12.3 §3 Setup F language is materially misaligned with this shape (assumes intra-bar two-leg) → Brief v12.4 trigger primary cause
- Symmetric long+short structure question: needs explicit confirmation in V41 Tier 2 — does long-bias side mirror as "post-pump-exhaustion regime detection"?
- Phase 1 v3 explicitly NOT triggered by Q2 (a) decision — preserves Path 1 commitment per §0.2

### Tier 2 substrate questions remaining for V41 (NOT resolved in V40)

Listed in §2 step 5. V41 surfaces these explicitly with architect AFTER Brief v12.4 ships AND BEFORE detect_f_v2 proposal drafting.

---

## 12. CHARACTER + DISCIPLINE NOTES FOR V41

V40 maintained these throughout (slot 5 + slot 25 + Rule 5 discipline):

- Defended positions with new arguments, conceded explicitly when Claude Code surfaced contradictions
- No metaphors (slot 5)
- No directing architect's time/energy/state (slot 25) — surfaced both decision points (next detector + permission rules + path 1/2/3/4 fork) without state-inference, let architect choose
- No conversational martingale on failing positions — explicitly declined to recommend among paths 1-4 when architectural-altitude finding surfaced
- Quantified costs in dollars/percentages when relevant (token estimates, context %, time elapsed, $3,288 Apex-blockable losses, ~10-15% throwaway under reframe)
- Verified char count via bash for memory operations (slot 26 trimmed from 812→489 over multiple iterations)
- Included entry-path environment context (local shell / SSH / tmux) on paste-ready commands — though bracket-form drift caused one incident (V40 drift #1)
- Timestamps prepended on every message via `TZ='America/Sao_Paulo' date` chained with tool calls
- Mechanical reproduction read source files directly via WebFetch (didn't summarize from memory) — three ship reports fully read
- Pushed back on architect mis-statements (M5 stupid, naive Claude Code prompting equivalent, work void) with concrete counter-evidence — slot 25 doesn't bind chat-Claude from honest disagreement, only from inferring/commenting on state

V41 inherits all of these. Drift on any one is corrigible mid-session if architect surfaces it; pre-empting is better.

---

**End of handoff.**

Architect: Rafael Vargas | Session 40 closed by Lucy V40 | Session 41 to be opened by Lucy V[N+1]
