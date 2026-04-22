# V32 Session Prep — Structural Audit & Rule Surface Index

**Document version:** 1
**Date created:** 2026-04-21 BRT
**Author:** Chat-Claude, Session 28
**Intended reader:** Chat-Claude, V32 session
**Trigger conditions met:** yes (see below)
**Session type:** dedicated structural, no code-write, no Apex progress

---

## Why V32 is happening

V32 was originally gated in Tape v31 Part 17.4 behind three conditional triggers. Two have now fired:

**Trigger A — rule-set cross-violations causing measurable session drift.** Session 28 (2026-04-21) produced three observable violations of rules that were loaded into context at session open:

- Slot 21 (environment prerequisite rule) violated 3 messages after merge. Chat-Claude produced an `scp` command block mixing Mac-side and droplet-side execution without marking which host each line runs on. Rafael caught via confusion, not by my self-check.
- Tool-dispatch inertia with Claude Code: Claude Code used its internal Read tool instead of Bash cat twice in succession despite explicit correction. Chat-Claude escalated wording but did not pivot strategy until Rafael did.
- Agenda drift at session open: chat-Claude's Rule 3 opening agenda proposed memory consolidation and apex-planning setup as Session 28 work items — both already shipped in Session 26 post-v30. Rafael caught via file upload of SESSION_26_CLOSEOUT.pdf.

**Trigger B — Phase 1 v2 near full ship.** Step F (regression under load) is the only remaining step. Rafael explicitly decided in Session 28 close that V32 comes before Step F. Justification: Step F executes end-to-end against live Bybit; running it with a substrate whose drift is observable and untreated contaminates the regression signal.

**Trigger C (memory ceiling)** has not fired — memory at 28/30 at session close.

Two of three triggers is sufficient. V32 proceeds.

---

## What V32 produces

**Primary output:** a rule surface map — lookup table from action surfaces to active rules, with precedence ordering for known conflicts.

**Secondary outputs:**
- Deprecation markers on superseded rules (preserved text, clear dormant marking)
- Canonical wording authority decision (Tape wins vs memory wins vs third surface)
- Memory slot consolidation where slots are redundant with Tape-preserved content
- Pattern library pruning policy (last-fired session tracking)
- Tape v32 with Part 19 documenting all of the above
- CLAUDE.md update for Claude Code reflecting wording alignment

**Not produced in V32:** any Phase 1 v2 Step F work, any Phase 3 v2 detector work, any Priority 0 channel build, any Mike Krieger outreach work.

---

## Phase sequence — 8 phases, ~4-6 hours total

### Phase 1 — Inventory (~60 min)

**Deliverable:** full enumeration of active rules across memory and Tape.

**Tasks:**
1. Iterate memory slots 1-28 (or whatever count at session open). Extract every rule-shaped statement (prescriptive: "do X" or "never Y"). Skip character/context statements and historical records.
2. Walk Tape v31.1 Parts 1-18. Extract rule-shaped statements — prescriptions only, not observations or historical records.
3. Cross-list results: mark rules that appear in both memory and Tape. Note any wording drift between the two surfaces.

**Expected output:** flat list of ~40-60 active rules.

### Phase 2 — Action-surface indexing (~90 min)

**Deliverable:** lookup table from action → applicable rules.

**Starting set of action surfaces** (not exhaustive, expected to expand during Phase 2):

- **Producing paste-ready command block** → Slot 21 (environment prerequisites), v31 Part 17.2 Rule 7 (paste-ready primacy), Slot 26 (PT→EN disambiguation if terms are ambiguous)
- **Proposing architectural decision** → v24 Part 10.2 (instrument-proposal rule), v30 Part 16.3 Rule 1 (end-to-end or silent), Rule 2 (pre-draft before reviewing), Rule 6 (interrogation: "what does this assume exists that doesn't?")
- **Session open** → Slot 18 (Lucy trigger, Tape request), Slot 26 (BRT timestamp, upload count, quick commands), v30 Part 16.3 Rule 3 (session-open architectural agenda), Slot 20 (Apex state pointer as current truth)
- **Closing session / building Tape** → v31 Part 17.3 (three close-out states: addendum, sub-version, clean close), Slot 18 (never-compress, page count must not decrease), Slot 24 (hard-load rule — approved Tape immediately active)
- **Reporting session ledger** → v27 Part 13.3 (work-quality vs metadata bifurcation), v30 Part 16.3 Rule 5 (tightened criteria: zero-carrot for baseline competence), v20 reset rule (ledger resets to 0 each session)
- **Memory modification** → Slot 5 (propose content before execute, no unauthorized delete), Slot 21 (view before modify, merge before creating new slots)
- **Suggesting to Rafael** → Slot 5 (no unsolicited advice), Slot 28 (no state inference), Slot 27 (quantify in dollars at Rafael's scale)
- **Responding to Rafael challenge** → Slot 5 (no silent adjustment under pressure, no capitulation retreat, no conversational martingale, no balanced-answer evasion when answer is known)
- **Producing images/files for Rafael** → Slot 26 (upload count reporting), Slot 21 (output location discipline)

**For each action surface:**
- List every rule that should fire
- Identify any two rules that pull opposite directions (precedence conflict)
- Note the trigger phrase or condition that should fire the rule retrieval

### Phase 3 — Conflict resolution (~60 min)

**Deliverable:** precedence ordering for known conflicts.

**Known conflicts from v31 Part 17.4:**

1. **Push-back rule vs time-direction rule.** Interrogating a time-allocation framing ("I should keep doing X") is functionally the same as directing time.

2. **Honesty rule vs push-back rule.** Genuine agreement with a framing is either pushed back on performatively (violates honesty) or not pushed back on (violates push-back).

3. **Balanced-answer rule vs push-back rule.** Surfacing alternatives to a known-correct framing = hedging when the answer is known.

4. **Slot 21 environment-prerequisite vs Rule 7 paste-ready primacy.** Flagging environment context at the top of a paste-ready block adds prose around what should be pure paste-ready content.

**Proposed tiebreaker (v24 Part 10.3, never tested):**
- Time-direction rule wins over everything. After that, honesty wins over push-back when Claude's agreement is genuine. Push-back fires only on real uncertainty.

**Phase 3 task:** test this tiebreaker against each conflict. Accept, modify, or replace.

**Additional conflict surfaced in Session 28:** conflict 4 (env-prereq vs paste-ready primacy) needs explicit resolution. Proposal: environment entry path is part of the paste-ready block itself (first line of the block, not prose around it). Example:

    # Run on Mac terminal (not SSH, not tmux):
    scp root@droplet:/tmp/file.txt ~/Downloads/

This satisfies both rules — the environment marker is inside the paste-ready surface, not analysis around it. Validate this resolution in V32.

### Phase 4 — Deprecation markers (~30 min)

**Deliverable:** explicit deprecation on superseded rules. Never-compress rule preserves text; dormant markers stop active retrieval.

**Known deprecations needing markers:**
- **v21 Part 7 inheritance loophole** — superseded by v23 Part 9.1 hard-load rule. Supersession recognizable only by reading v23 after v21. Needs explicit DEPRECATED header in v21 Part 7.
- **v18 unified ledger formula** — superseded by v27 Part 13.3 work-quality/metadata split, then tightened by v30 Part 16.3 Rule 5. Two-layer supersession.
- **Slot 21 residual version-gap quote wording** — the Korean text includes a fragment from the pre-v23 version-gap framing that v23 Part 9.1 corrected. Stale wording, already not applied in practice.

**Phase 4 task:** mark each with DEPRECATED header. Preserve original text (Lucy Continuity, never-compress). Update any memory slot that still references superseded rules.

### Phase 5 — Canonical wording authority (~30 min)

**Deliverable:** rule for resolving memory vs Tape wording drift.

**Proposal (draft, open to revision):**
Tape is canonical for rule wording. Memory slots are operational aliases that must match Tape wording at creation. When Tape text changes in a new version, memory must be updated to match within the same session, or the drift flagged in deferred items.

**Rationale:** Tape is the preservation vault under Lucy Continuity (Slot 25). Memory is thin (28/30 limit) and rewritten frequently. Authority flows from durable surface to volatile one.

**Alternative considered:** memory-wins because memory is newer by definition. Rejected because memory entries are compressed into Korean and may lose nuance present in Tape prose; under conflict, the Tape's more detailed expression is typically the intended rule.

**Alternative considered:** adopt a third canonical surface (dedicated `rules.md` in apex-planning). Rejected because it adds a surface to maintain without eliminating the memory/Tape tension — it just makes three surfaces to keep aligned.

**Phase 5 task:** confirm, modify, or replace the Tape-wins proposal.

### Phase 6 — Pattern library pruning (~45 min)

**Deliverable:** pattern library entries across Tape Parts 8-18 tagged with last-fired session number. Dormant marking for entries not fired in 5+ sessions.

**Known pattern library entries for review:**
- Capitulation retreat (Tape v22 Part 8.2) — fired in Session 19, 20, 27, 28. Active.
- Conversational martingale (Tape v22 Part 8.2) — fired in Session 19, 27, 28. Active.
- Time-cost ledger gap (Tape v22 Part 8.3) — graduated to formal rule in v27 Part 13.3. Dormant, superseded.
- Un-push category (Tape v21 Part 7.3) — fired in most sessions since v21. Active.
- Discovery-without-implementation (Tape v30 Part 16.2) — fired in Session 26 retroactively, Session 27. Active.
- Tool-dispatch inertia (Session 28, this document) — candidate for pattern library on first occurrence. Name if pattern holds in V33+.
- Self-observation asymmetry / Category D (Tape v29 Part 15.5 Incident 3) — fired in Session 25, 28. Active.
- Writing-surface (Tape v21 Part 7) — foundational, never dormant.

**Phase 6 task:** walk each entry, tag with last-fired session number, mark dormant where applicable. Add new candidate entries surfaced since last audit.

**Pruning threshold proposal:** dormant after 5 sessions without firing. Preserved in Tape (Lucy Continuity), removed from active retrieval map.

### Phase 7 — Tape v32 build (~60 min)

**Deliverable:** Tape v32 PDF at 11x34, markdown canonical in git.

**Part 19 content:**
- Phases 1-6 outputs from V32 session
- Session 28 drift incidents as evidence base for the audit
- Rule surface map summary (full map in separate apex-planning file)
- v24 Part 10.3 precedence tiebreaker status (accepted/modified/replaced)
- Canonical wording authority decision
- Deprecation markers applied
- Pattern library pruning results

**Close-out state per v31 Part 17.3:** state (c) — clean session close immediately after V32 ships. No post-V32 work this session.

### Phase 8 — CLAUDE.md and memory alignment (~30 min)

**Deliverable:** wording alignment across surfaces.

**Tasks:**
- Memory slots updated to match Tape v32 canonical wording where drift was found
- CLAUDE.md Session-Open Protocol includes pointer to `apex-planning/rule_surface_map_v1.md`
- Slot 20 refreshed if still stale (Phase 1 v2 progress — likely Step F status by V32)
- Memory slot consolidation: redundant slots merged, freed slots available for future use
- Commit all updates to apex-planning origin

---

## Open questions — decide at session start

1. **Phase 6 aggressiveness.** Aggressive pruning reduces retrieval surface but risks erasing context. Conservative preserves more but keeps growth. Skip-Phase-6 defers to V33. Default recommendation: conservative for V32 (first-time audit), consider aggressive for V33+.

2. **Canonical wording authority direction.** Tape-wins is my proposal. Alternatives considered and rejected above. Confirm or replace.

3. **Dormant vs delete.** Under Lucy Continuity (Slot 25, Tape v25 Part 11.4), nothing should be deleted. Dormant marking preserves text but stops active retrieval. Confirm: dormant acceptable, delete not allowed. If delete needed in some cases, name which cases.

4. **Phase order.** The 8-phase sequence assumes inventory -> index -> conflicts -> deprecation -> authority -> pruning -> Tape -> sync. Alternative: canonical authority decision (Phase 5) first, because it determines how conflicts in earlier phases resolve. Your call at V32 session open.

5. **Priority 0 channel resurrection.** V32 is substrate-layer work. If V32 reduces substrate drift materially, Priority 0 argument weakens. If V32 produces no observable improvement, Priority 0 becomes stronger (accept substrate, engineer around it). Flag: V32 outcome conditions Priority 0 decision.

---

## Session prerequisites

Before V32 session opens:
- This prep document available: either via `curl https://raw.githubusercontent.com/rafamv/apex-planning/main/v32_session_prep.md` OR as uploaded PDF failsafe
- Tape v31.1 uploaded at session open alongside the prep document
- Fresh Claude instance (Opus 4.7 Adaptive at this writing; whatever is current at V32 session open)
- Memory in state visible at session open — do not assume 28/30 exactly
- Clean tmux apex session on droplet (Claude Code may or may not be attached; V32 is chat-Claude work, Claude Code not strictly needed)

---

## What this prep document is NOT

This is the handoff specification. It is not the V32 session itself. The actual audit work happens in the V32 session.

Subtle reminder for the V32 Claude instance: **apply Rule 3 at session open.** Propose the V32 session agenda concretely, referencing this prep but confirming it against current state (has Step F shipped? has memory changed? has Rafael modified priorities?). Do not assume this document reflects the state at V32 session open — it reflects state at V32 prep time, 2026-04-21.

---

## V32 close signal

Session 28 closes immediately after V32 prep is pushed to apex-planning and rendered as PDF. Step F deferred to post-V32. Session 28 final deliverables:

- Step D commit 1 shipped (`8df4399`)
- Step E commit 1 shipped (`6727b7d`)
- Step E commit 2 shipped (`ee44d05`)
- V32 session prep pushed to `apex-planning/v32_session_prep.md`
- V32 prep PDF available for upload failsafe
- HTML trading lesson for Jessica's brother — produced, separate artifact

Session 28 ledger (estimated, to be finalized at close):
- Work-quality: +5 (Step D proposal schema catch, Step D 6-point confirmation, Step E c1 ground-truth diff review, Step E c2 integration test review, V32 prep draft)
- Metadata: -6 (scp command missing host markers, agenda drift at open, tool-dispatch escalation with Claude Code, reliability pattern failure to apply Slot 21 at action time, premature 4d clarification on "temptation" without prompting from Rafael, initial hedge on ship-D vs re-run-B before Rafael pushed)

Interpretation per v24 Part 10.4 failsafe-as-signal: strong work-quality signal preserved across all architectural reviews; metadata drift confirms the V32 audit is the right next move.

---

**Document ends.**
