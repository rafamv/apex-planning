# V60 → Chat-Claude Handoff: Brief v14.1 Amendment Proposal

**Priority:** HIGH
**Origin:** V60 W3 Step 1 end-of-step retrospective (2026-05-22)
**Ship branch:** `w3-step1-hot-reload` — pushed to origin, **NOT YET merged to main** at handoff time
**Affected commits on `w3-step1-hot-reload` (branch-only, not in main yet):**
- `90a1fec` — CLAUDE.md operational form for the proposed rule
- `09bc93b` — CLAUDE.md brief pointer bump v13.3 → v14 (companion item)
- `e66e48a` — W3 Step 1 V60 revision (split watcher to `apex/config_watcher.py`) — concrete evidence of the discipline gap

> ⚠️ **Sequencing dependency for W3 Step 2:** W3 Step 2 (supervised-task lifecycle — Primitive 2) cannot start a clean pre-write contract until `w3-step1-hot-reload` merges to `main`. See "Sequencing" section near the end of this doc for details. Chat-Claude review of THIS amendment proposal does NOT depend on that merge; W3 Step 2 kickoff DOES.

---

## Context

W3 Step 1 (V60) end-of-step retrospective surfaced an unspecified discipline gap in the architect loop: **Code-Claude has no defined obligation to architecturally read the pre-write contract before executing.**

Existing §10 Theme 4 candidates cover:

- **Candidate 7 (Code's contract-binding on execution).** Code obeys the contract during execution; deviations require contract-violation interrupt to architect.
- **Candidate 9 (chat-Claude's deviation skepticism at end-of-step).** chat-Claude reviews Code's execution-time deviations against pre-write, not against post-hoc rationale.

Both fire **after** pre-write. Neither covers what Code should do **at pre-write read time** when the contract itself has an architectural concern.

---

## V60 Trigger

V59 pre-write decision 1: "Watcher lives in `apex/config.py` (NOT a separate `config_watcher.py`) — high cohesion, Brief-aligned." Code-Claude acknowledged at session-open and executed against the contract verbatim. End-of-step review by the architect identified the decision as misaligned with §10 Theme 2 (modularity construction):

- **Settings** (data definition, high churn — grows every Phase as new fields land)
- **ConfigWatcher** (mechanism, near-zero churn — once watchdog wiring works, doesn't change)

…have **inverse change cadences**. The cohesion argument inverts; the correct shape splits them into separate module homes (`apex/config.py` + `apex/config_watcher.py`). V60 commit `e66e48a` is the concrete revision implementing this.

**Cost incurred at V60:** 4 commits landed under the V59 contract, end-of-step review caught the issue, W3 Step 1 required revision before merge. Had a Code pre-write architectural read been live, the concern would have surfaced at session-open in 30 seconds — architect breaks the tie before any code lands.

---

## Proposed Canonical Rule (new §10 Theme 4 candidate)

**Code-Claude Pre-Write Architectural Read.**

After reading the pre-write contract at session-open and BEFORE writing any code, Code-Claude reads the contract against Brief §10 themes and surfaces findings to the architect in plain English. Two modes:

### Mode A — clean

Contract aligns with §10 themes. One-line surface:

> Pre-write read: contract aligns with §10 themes. Proceeding.

### Mode B — flag

One block per concern (cap 3 per Step):

```
### Concern N — [short label]

What the contract says:   [plain-English sentence]
What I'd push back on:    [plain-English sentence]
Why it matters:           [§10 theme reference + Phase 7+ consequence]
Options:
  A — As contracted:      [one-line outcome]
  B — Alternative:        [one-line outcome]
My recommendation:        [A or B] because [architect-readable reason]
```

Closer: "Your call. I'll proceed with whichever you confirm."

### Body discipline (pins architect-readability)

- NO Python syntax, NO diff snippets, NO LoC counts in concern blocks
- NO file paths in "What I'd push back on" / "Options" (OK in "What the contract says")
- §10 themes referenced as labels ("§10 Theme 2 — modularity"), not quoted text
- Wait for architect tie-break BEFORE touching any file

### Calibration target

Mode B fires ~1 in 3 Steps. Mechanical Steps stay in Mode A. If Mode B fires every Step, Code is over-flagging; if it never fires, Code is rubber-stamping.

### Operational reference

Already live in `CLAUDE.md` "Architect Loop / Pre-Write Architectural Read" subsection (V60 commit `90a1fec`, shipped on `w3-step1-hot-reload`). Takes effect at next session-open on any worktree.

---

## Sibling Pairing — Completes the Three-Boundary Set

| Boundary | Code-Claude obligation | chat-Claude obligation |
|---|---|---|
| Pre-write read | **NEW candidate (this proposal)** | (chat-Claude pre-write drafting, existing) |
| Execution time | Candidate 7 (contract-binding) | — |
| End-of-step review | — | Candidate 9 (deviation skepticism) |

The new candidate fills the empty Code-Claude / pre-write-read cell. Three candidates together cover the three architect-loop boundaries where architectural integrity can be checked.

**Suggested positional spot:** between Candidate 7 (Code's execution-time contract-binding) and Candidate 9 (chat-Claude's end-of-step skepticism), matching the pre-write → execution → review temporal order.

---

## Priority — HIGH

The discipline gap is responsible for end-of-step revision cycles that consume architect attention. V60 cost: ~30 minutes of architect end-of-step review + a revision cycle for W3 Step 1.

Across the remaining Phase-7-entry surface — W3 Steps 2-4, Phase 3 v2 detect_0/_f/_b/_c/_e, Phase 4a/4b, Phase 5/5.5/6/7 — easily 20+ pre-write opportunities. Discipline in place from W3 Step 2 forward avoids cumulative drift across all of them.

**Lower-priority companion concern (worth pondering, NOT V60 trigger):** the same drift class might exist for chat-Claude at pre-write drafting (chat-Claude writes a pre-write contract without independently testing it against §10 themes). chat-Claude self-review at pre-write drafting is a sibling rule worth surfacing at v14.1 turn but is not the V60 trigger and not load-bearing for this amendment.

---

## V52 Anti-Pattern Guard Compliance

The V60 W3 Step 1 revision (commit `e66e48a`) is framed explicitly as a **pre-write re-decision at end-of-step**, NOT as "evolution" of the V59 contract. Per §10 Theme 3 V52 anti-pattern guard: drift validated as evolution is the failure mode the discipline prevents.

The V60 commit message documents the re-decision rationale with the original V59 contract surface preserved in git history (commit `00b0071`). chat-Claude review of this handoff should evaluate the proposed rule against the V60 cost (one revision cycle), NOT against post-hoc rationalization of why the V59 contract was acceptable.

---

## Brief v14.1 Patch Surface

- **§10 Theme 4 (Agent supervision)** — add new candidate at suggested positional spot above.
- **§8 Phase status table** — no impact.
- **Change log v14 → v14.1 entry** — Patch 1 documenting V60 surfacing + candidate absorption + sibling-pairing rationale.
- **CLAUDE.md "current brief pointer"** — bump v14 → v14.1 at v14.1 ship time (per Brief section bump rule). Note: pointer is currently at v14 (V60 commit `09bc93b` fixed the v13.3 → v14 lapse that was outstanding from Brief v14 ship at SHA `81a9d49`).

---

## Architect Decision Request to chat-Claude

1. **Approve** promotion to canonical §10 Theme 4 in Brief v14.1.
2. **Suggest refinements** to the operational form (template structure, body discipline, calibration target).
3. **Reject** with reasoning.

Body discipline + Mode A/B template should be verbatim from the CLAUDE.md operational form (commit `90a1fec`) unless chat-Claude proposes specific wording changes. Wholesale replacement of the operational form during canonical absorption is fine; surface-level rewording for canonical voice is expected.

---

## Supporting Material in Repo

- **Operational form:** `CLAUDE.md` "Architect Loop / Pre-Write Architectural Read" subsection (V60 commit `90a1fec`).
- **V60 W3 Step 1 substantive ship:** `w3-step1-hot-reload` branch, 5 commits — `bbce4b0`, `00b0071`, `9d98b03`, `0cb3137`, `e66e48a`.
- **V60 trigger evidence (the actual revision the rule would have prevented):** commit `e66e48a` — moves ConfigWatcher to `apex/config_watcher.py`, adds `replace_settings()` chokepoint in `apex/config.py`, drops `stop_config_watcher` call from `apex/main.py` finally block.

---

## Sequencing — what must happen before W3 Step 2 starts

There are two independent thread-lines downstream of this handoff:

**Thread A — Brief v14.1 amendment absorption (chat-Claude review).** Evaluating, approving, and canonicalizing the proposed §10 Theme 4 candidate into Brief v14.1. Does **NOT** require any merge to complete first — chat-Claude can review this proposal against the operational form (which is already on the `w3-step1-hot-reload` branch and is content-frozen there).

**Thread B — W3 Step 2 kickoff (supervised-task lifecycle, v14 Primitive 2).** **DOES** require `w3-step1-hot-reload` to land on `main` first. Step 2's pre-write contract will reference W3 Step 1's substrate:

- The `config_hot_reload_*` Settings fields (read by the supervisor's flag-polling loop at call-time)
- The `apex.config.replace_settings()` chokepoint (future hook for supervisor-initiated swaps)
- The `apex.config_watcher` module (provides the hot-reload mechanism that supervisor's call-time reads depend on)

If W3 Step 2 branches off main before Step 1 lands, Step 2's branch is missing the substrate Step 2 builds on, and Step 2's pre-write contract references would point at non-existent code. This dependency is mechanical (Step 2's substrate references at the code level), not policy.

### Required actions before W3 Step 2 pre-write drafting

1. Merge `w3-step1-hot-reload` to `main` — either via GitHub PR UI at https://github.com/rafamv/apex/pull/new/w3-step1-hot-reload, or via local `--no-ff` merge from primary apex repo.
2. Push `main` to `origin`.
3. (Optional) Droplet sync per existing project pattern.
4. (Optional, this Brief v14.1 amendment landing) — chat-Claude approves the new §10 Theme 4 candidate; CLAUDE.md operational form stays as-is unless chat-Claude proposes wording changes; Brief v14.1 ships with the new candidate.

Steps 1-3 are blockers for W3 Step 2 substrate availability. Step 4 (Brief v14.1 absorption) is independent and can land in any order relative to W3 Step 2 — but if it lands BEFORE W3 Step 2 starts, W3 Step 2's own pre-write read will benefit from the canonicalized rule.

---

End of handoff.
