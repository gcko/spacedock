---
id: k9s1zdwzfmdzdayjdvjjpj6f
title: "FO-owned `### Feedback Cycles` section should live on the worktree branch, not main"
status: done
source: "Captain observation 2026-04-30 during PR #176 + PR #177 rebase: feedback cycles entries on main collide with worktree-branch stage reports, producing painful merge conflicts in the entity body when the PR lands"
started: 2026-05-01T00:48:15Z
completed: 2026-05-01T04:04:49Z
verdict: REJECTED
score: 0.65
worktree:
issue:
pr:
mod-block:
archived: 2026-05-01T04:04:49Z
---

The FO Write Scope (shared-core) currently gives the first officer write rights to the `### Feedback Cycles` section in entity bodies on main:

> **`### Feedback Cycles` section** — in entity bodies, tracking rejection rounds

In practice, during worktree-backed stages (implementation / validation), the worktree branch appends stage reports to the same entity file. The FO appends Feedback Cycles entries to the same file on main when a gate is rejected. Both regions live in the bottom of the entity body, sequentially. When the PR lands, `git merge` or `git rebase` has to hand-resolve the overlap because each side advanced different lines in the same trailing region.

PR #176 and PR #177 (2026-04-30) both went DIRTY/CONFLICTING for this exact reason. The captain's `/clear` session had to manually resolve the merge in the worktree because the cycle entries on main and the cycle-2 stage reports on the branch were adjacent and the merge driver couldn't tell which order they should interleave. The work is mechanical but slow and error-prone — it costs a captain turn every PR merge, and it scales linearly with the number of feedback cycles a PR went through.

## Recommendation: fold into entity 21 (`stage-worktree-stickiness`, issue #104)

After cycle-1 captain feedback, this task should be **folded into entity `21` (`stage-worktree-stickiness.md`, issue #104) as a sub-scope** rather than shipped standalone. Rationale:

- Under stickiness (issue #104 Option A — captain's lean), `worktree:` becomes a stable invariant: set when the entity first enters a worktree-creating stage, cleared at terminal merge. The cycle-routing rule then collapses to a corollary of that invariant: "cycle entries follow the stickiness invariant — worktree-side while `worktree:` is set, main-side otherwise (which only happens before the first worktree-creating stage)."
- All five staff-review findings either collapse or simplify under stickiness (see `### Findings under stickiness` below). The standalone version of the design has to defend against edge cases (orphan worktrees, reads-of-cycle-counter splitting across two views) that simply do not exist once stickiness is the contract.
- Entity `21` already names this exact scenario in its design: *"If `analyze` rejects and `feedback-to: run` routes back, the feedback fix obviously needs to land in the same worktree — reinforcing that once an entity enters a worktree stage, it should not leave it until the terminal merge"* (`stage-worktree-stickiness.md` line 45). The PR #176/#177 merge-conflict pain is the empirical evidence motivating stickiness, and the rule is part of the same FO contract section (Feedback Rejection Flow + FO Write Scope + Worktree Ownership).

**Stickiness is a hard precondition** for shipping the cycle-routing rule. If the cycle-routing rule landed first (without stickiness), the edge cases the staff review flagged at cycle-1 — stale-worktree fallback, cycle-counter read-both-views, mid-flight pre-existing main-side entries — re-enter as live concerns and the implementation has to grow shims to handle them. With stickiness as the invariant, those concerns disappear by construction.

### Folded sub-scope inside entity 21

When entity `21` advances, fold the following into its scope:

1. Add a new `## Sub-scope: Feedback Cycles routing` section in `stage-worktree-stickiness.md` naming the corollary rule and citing PR #176/#177 as motivation.
2. Add the Feedback Cycles ACs (renumbered as part of `21`'s AC list) covering: worktree-side writes during the in-flight life of a worktree-set entity (AC-1 below), merge-clean proof in an offline fixture (AC-2 below), and shared-core wording (AC-3 below). Drop AC-4 (empty-worktree-but-feedback-to case) and AC-5 (bare-mode equivalence) and AC-6 (no-migration policy) — those collapse under stickiness, see `### Findings under stickiness`.
3. Update entity `21`'s test plan to include the offline merge-clean test (`tests/test_feedback_cycles_merge_clean.py` + fixture under `tests/fixtures/feedback-cycles-merge/`) as one of its surfaces.
4. Update entity `21`'s "FO contract intersection" section to call out the Feedback Rejection Flow + FO Write Scope wording changes alongside the reuse-condition #3 reword it already names.

This entity (`feedback-cycles-on-worktree-not-main.md`) should be marked `verdict: REJECTED` (or moved to archive) at the captain's discretion, with a pointer in the archive note to entity `21`. The cycle-1 ideation work is preserved inline below as the design rationale that informs the folded sub-scope.

If the captain instead wants this task to ship *before* `21` lands, the design below is the standalone form — but the captain must accept the edge-case shims and the explicit precondition that stickiness has not yet shipped (so cycle-counter reads and orphan handling remain live concerns).

## Proposed approach (under stickiness, as folded sub-scope)

### The rule

Under the stickiness invariant from entity `21`, an entity with `worktree:` set keeps that worktree as canonical for all entity work until terminal merge. The Feedback Cycles routing rule becomes a one-line corollary:

> **Feedback Cycles entries are written to the same view as the entity's other in-flight body content.** While `worktree:` is set, that view is the worktree copy; before the first worktree-creating dispatch, that view is main. The terminal merge brings the worktree-side entries onto main.

This is not a separate conditional in the FO. It is a clarification of an existing invariant: cycle entries are entity-body content, entity-body content during in-flight stages lives where stage reports live, stage reports live in the worktree under stickiness. The FO Write Scope and Feedback Rejection Flow sections of shared-core are updated to *name* this consequence rather than to introduce a new rule.

### Source files that change (under stickiness)

1. `skills/first-officer/references/first-officer-shared-core.md`
   - **`## FO Write Scope`** (line 218): replace the unconditional `**\`### Feedback Cycles\` section** — in entity bodies, tracking rejection rounds` bullet with a sub-clause that names both states explicitly. Recommended wording (sub-clause heading anchor: "when `worktree:` is set"):
     > **`### Feedback Cycles` section** — in entity bodies, tracking rejection rounds. **When `worktree:` is set on the entity, the FO writes the cycle entry to the worktree copy of the entity file and commits on the worktree branch (the cycle entry then rides the next stage-report commit into the merge). When `worktree:` is empty, the FO writes to main.** Under stage-worktree stickiness (entity `21`), `worktree:` is empty only before the first worktree-creating dispatch.
   - **`## Feedback Rejection Flow`** (line 178): replace `The first officer owns the \`### Feedback Cycles\` section and keeps it on the main branch.` with `The first officer owns the \`### Feedback Cycles\` section. Routing follows FO Write Scope: worktree-side when \`worktree:\` is set, main-side otherwise.` Step 2 of the flow gains the same routing reminder.
   - **`## Worktree Ownership`** (line 206): add `### Feedback Cycles` to the list of body content that stays in the worktree during in-flight stages, alongside the existing "active stage/status/report/body state" wording.
   - **`## FO Write Scope` "Entity body content beyond `### Feedback Cycles`"** (line 228): clarify that the carve-out for Feedback Cycles still applies — the FO can write that subsection in the appropriate view (worktree or main), other body content remains worker-only.

2. `skills/first-officer/references/claude-first-officer-runtime.md`
   - **Line 155** (cooperative-shutdown sweep exemption that reads `### Feedback Cycles` to detect active feedback state): under stickiness this reader operates against the worktree copy when `worktree:` is set. The text needs a parenthetical: "(reads from the worktree copy when `worktree:` is set; otherwise main)." This is a one-line cite, not a behavioral change — the reader was already doing the right git-show against the right ref by virtue of stickiness; stating it makes the contract auditable.
   - The 3-cycle-limit cycle-counter read in `## Feedback Rejection Flow` step 3 reads from the same single source under stickiness — no read-both-views shim needed.

3. `skills/first-officer/references/codex-first-officer-runtime.md`
   - Audit for any prose that says "on main" or "main branch" near Feedback Cycles; update only if a contradiction with the new rule exists.

### Findings under stickiness — captain's cycle-1 review items, addressed

- **(a) Stale-worktree handling.** Under stickiness, the invariant is `worktree:` set ↔ live worktree on disk. A `worktree:` field that points at a deleted path is an *orphan* state, handled at boot (the existing orphan-detection in `status --boot`). The Feedback Cycles routing rule does not need its own fallback because it operates inside the invariant, not against it; stale-worktree recovery is `21`'s job (and arguably `status --boot`'s job, where it already lives). **Resolved by stickiness.** A defensive guard ("if write fails, fall back to main and log") is fine to add as a one-liner in shared-core, but it is not load-bearing for the design.

- **(b) Source-file list incomplete.** Added two missed sites: `first-officer-shared-core.md:228` (Entity body content carve-out — clarified) and `claude-first-officer-runtime.md:155` (agent-reuse / shutdown-sweep exemption that reads `### Feedback Cycles` — gets a one-line view-clarification parenthetical). Both are now in the source-file list above.

- **(c) AC-3 grep too loose.** The AC-3 verification now anchors on a literal sub-clause phrase (`when \`worktree:\` is set`) inside the FO Write Scope block, plus the literal sentence in the Feedback Rejection Flow section. See AC-3 below for the exact strings.

- **(d) AC-6 fixture risks passing by luck.** Removed AC-6 entirely under stickiness (no main-side legacy entries on a worktree-set entity to migrate). Where the offline merge-clean fixture (AC-2) needs to prove the right conflict shape, the fixture explicitly places the worktree-side cycle-2 stage report immediately adjacent to (within one blank line of) the previous cycle's stage report region, so the control case (cycle entry written on main) lands in the same trailing-region overlap that PR #176/#177 hit. AC-2's wording below makes this explicit.

- **(e) Cycle-counting / session-resume.** Resolved by stickiness — `### Feedback Cycles` lives in exactly one view per entity at any given time, so the cycle-counter read targets that view directly. The shutdown-sweep reader and the 3-cycle-limit counter both rely on this single-source-of-truth property. No read-both-views requirement is introduced; if `21` were not in scope, this would have to become an explicit AC.

### Where the rule fires

Same as cycle-1 — the FO's gate-rejection path (shared-core `## Feedback Rejection Flow` step 2) is the only place the FO writes `### Feedback Cycles`. Under stickiness, the FO reads `worktree:` from frontmatter (already loaded for other dispatch logic) and routes the write target accordingly. One conditional, one spec section, no new helper.

### Migration story (under stickiness)

- **Entities that have already had cycle entries written on main during a worktree-backed stage** (PR #176/#177 era): the existing entries stay on main; future cycle entries on those same entities follow the new rule. Because stickiness keeps the worktree alive across the entity's life, the worktree branch and main do not both append to the trailing region simultaneously, so even mid-flight entities stop hitting the conflict shape going forward.
- No frontmatter migration, no entity-file rewrite, no script.

## Acceptance criteria

**AC-1 — When `worktree:` is set on an entity, the FO writes new `### Feedback Cycles` entries inside the worktree copy of the entity file and commits on the worktree branch; the main copy of the entity file is untouched until PR merge.**
Verified by: extending `tests/test_rejection_flow.py` (the existing live validation-rejection test) with two assertions after the FO routes feedback back to implementation: (1) `git -C {worktree_path} show HEAD:{workflow_dir}/{slug}.md` contains a `### Feedback Cycles` block with the new cycle entry; (2) `git show main:{workflow_dir}/{slug}.md` does *not* contain the new cycle entry. Test shape mirrors the existing FO log + frontmatter assertions in that file. Use the live-claude tier already wired for that test.

**AC-2 — Two consecutive feedback cycles on the same worktree-backed entity merge into main with zero conflicts in the entity file, and a control case proves the test exercises the actual PR #176/#177 conflict shape.**
Verified by: a new offline test `tests/test_feedback_cycles_merge_clean.py` plus a fixture under `tests/fixtures/feedback-cycles-merge/` containing:
- a workflow `README.md` with the standard four-stage spacedock layout
- one entity file `task.md` with frontmatter `status: validation`, `worktree: .worktrees/task`, and a body that includes one `## Stage Report: implementation` section followed immediately by one `## Stage Report: validation` section (representing cycle-1)

The test scripts the conflict scenario directly (no live FO required):

1. **Treatment case (new rule):** create a worktree branch off main; on the worktree branch, append a second `## Stage Report: implementation` (cycle-2), append a `### Feedback Cycles` entry placed *immediately after* the cycle-1 validation report (no intervening blank-line buffer beyond the standard single blank line — same trailing region as the stage reports), then append a `## Stage Report: validation (cycle 2)` section. Run `git merge --no-ff {branch}` from main. **Assertion:** `git merge` exits 0; `git diff --check` reports zero conflict markers in `task.md`; `grep -c '<<<<<<<' task.md` returns 0; the merged file contains exactly one `### Feedback Cycles` block with the cycle entry.

2. **Control case (old rule, expected to conflict):** reset main, create a fresh worktree branch off main, on the worktree branch append the same cycle-2 stage reports as above; *separately on main* append a `### Feedback Cycles` entry immediately after the cycle-1 validation report. Run `git merge --no-ff {branch}` from main. **Assertion:** the merge fails or `task.md` contains `<<<<<<<` markers — proving the test exercises the actual failure mode and is not passing because of benign whitespace separation.

The fixture's stage-report regions are written line-adjacent to the trailing region where the FO would append the Feedback Cycles entry under the old rule, so the control case reproduces PR #176/#177's conflict shape, not a benign whitespace-only diff.

**AC-3 — `skills/first-officer/references/first-officer-shared-core.md` describes the conditional routing rule with structural anchors readers and tests can rely on.**
Verified by: extending `tests/test_repo_edit_guardrail.py`'s static-content phase (which already greps the assembled FO content for FO Write Scope items) with two literal-substring assertions on the assembled agent content:
- the assembled text contains the literal substring `When \`worktree:\` is set` *inside* the FO Write Scope `### Feedback Cycles` bullet (verified by extracting the bullet's surrounding paragraph by anchored regex and asserting the substring appears within it, not anywhere in the file);
- the Feedback Rejection Flow section's ownership sentence contains the literal substring `worktree-side when \`worktree:\` is set, main-side otherwise`.

These are structural anchors, not proximity windows — false positives require the literal phrase to leak into unrelated prose, which is much narrower than the cycle-1 200-character window.

## Test plan

| AC | Test | Tier | Approx cost |
|----|------|------|-------------|
| AC-1 | Extend `tests/test_rejection_flow.py` with worktree-vs-main split assertions on the cycle entry | live-claude (existing tier) | adds ~0 budget — same FO run, two extra assertions |
| AC-2 | New offline test `tests/test_feedback_cycles_merge_clean.py` + fixture under `tests/fixtures/feedback-cycles-merge/` (treatment + control cases) | offline (`make test-static`) | seconds; no live model |
| AC-3 | Extend `tests/test_repo_edit_guardrail.py` static-content phase with literal-substring greps inside extracted blocks | offline (assembled-agent content check) | seconds; no live model |

The offline merge-clean fixture (AC-2) is the load-bearing proof — it directly exercises the merge property the change targets, including a control case that fails the old way. AC-1 is the live confirmation that the FO actually writes to the worktree path. AC-3 is the spec-wording check.

E2E coverage beyond the existing rejection-flow tests is not needed: the conditional is a one-line corollary of stickiness, firing off a frontmatter field already read elsewhere. Static checks alone would not have caught PR #176/#177 (the pain is mechanical line-overlap), so the offline merge fixture is the exact-abstraction proof. Bare-mode equivalence (cycle-1's AC-5) collapses under stickiness — the routing rule is independent of teams availability and the fixture-level proof covers it without an extra runtime tier.

If the captain prefers to ship this *before* `21` lands (against the recommendation above), add back AC-4 (empty-worktree case, sibling fixture under `tests/fixtures/feedback-cycles-no-worktree/`), AC-5 (codex-bare runtime confirmation), and AC-6 (no-migration policy with a line-adjacent legacy-entry fixture). Those are the shims the staff review flagged; the recommendation is to avoid them by waiting for stickiness.

## Stage Report: ideation

- DONE: Resolve the architectural questions the captain flagged at intake: (a) when is the worktree FIRST available for FO writes — only after the first dispatch into a worktree-backed stage, so feedback rejection at *backlog* or *ideation* (which precede the first worktree-creating stage) must keep writing on main; (b) bare-mode: in single-entity / -p mode the FO still has filesystem access to the worktree path even without a live team — does the worktree-write rule still apply, or does bare mode keep main-writes for feedback?; (c) backward compat: the FO Write Scope clause currently authorizes ONLY `### Feedback Cycles` and frontmatter on main — the new rule has to express the conditional cleanly so future readers don't lose the audit trail of allowed main-writes.
  Answered in `## Proposed approach` → `### Architectural answers`. (a) routing fires off frontmatter `worktree:`, not stage name; pre-worktree rejections write main. (b) bare mode is orthogonal; rule fires off `worktree:` uniformly. (c) FO Write Scope sub-clause names both branches of the conditional plus rationale.
- DONE: Replace the `## Sketch of the fix` section with a concrete `## Proposed approach` covering: which FO source files change (shared-core + claude-runtime adapter wording), where in the FO's gate-rejection flow the conditional fires, and what the migration story is for entities mid-cycle when this lands.
  `## Proposed approach` now names `skills/first-officer/references/first-officer-shared-core.md` line 218 (FO Write Scope), line 178 (Feedback Rejection Flow ownership statement), step 2 of the rejection flow, and the worktree-ownership cross-reference; flags the runtime adapters for audit; and gives a no-migration policy for mid-flight entities.
- DONE: Refine the AC list: each AC must be an end-state property with a `Verified by:` clause that names a specific test file + test shape. Tighten AC-2's repro fixture spec — name the fixture path under `tests/fixtures/`, the entity body shape it constructs, and which assertion proves no conflict (e.g., `git merge` exit 0 + zero conflict markers). Add an AC if architectural question (a)/(b)/(c) reveals one — captain prefers ACs over hand-waving.
  AC-1 names extension to `tests/test_rejection_flow.py`. AC-2 names fixture `tests/fixtures/feedback-cycles-merge/`, entity-body shape (frontmatter + cycle-1 implementation/validation reports), and assertions (`git merge` exit 0 + `git diff --check` zero markers + control case proves the failure mode). AC-3 names `tests/test_repo_edit_guardrail.py` extension. AC-4 covers the empty-worktree case via a sibling fixture. AC-5 (new) covers bare-mode equivalence — answers (b). AC-6 (new) covers the no-migration policy — answers (c). All AC-N items are end-state properties with `Verified by:` clauses naming a specific test file + shape.

### Summary

Hardened the spec into a single-conditional design: the FO routes `### Feedback Cycles` writes by reading frontmatter `worktree:` at write time, with no bare-mode exception and no migration needed. Six entity-level ACs cover (a) the worktree-set case in live testing, (b) merge-clean proof in an offline fixture that includes a control case to prove the test exercises the actual failure mode, (c) shared-core wording verified via the existing FO write-scope guardrail test, plus the empty-worktree, bare-mode, and no-migration cases. Test plan is mostly offline: the merge property is the load-bearing claim, and a fixture-level `git merge` + conflict-marker check proves it directly without burning live-model budget.

### Feedback Cycles

- **Cycle 1 → 2 (2026-04-30):** Captain rejected the cycle-1 ideation gate. Staff review surfaced 5 findings: (a) stale-worktree handling, (b) incomplete source-file list (missed `first-officer-shared-core.md:228` Entity body content carve-out and `claude-first-officer-runtime.md:155` cycle-detection reader), (c) AC-3 200-char proximity grep too loose, (d) AC-6 fixture risks passing by luck without line-adjacent placement, (e) cycle-counter read-both-views requirement once entries split. Captain reframed the design under stage-worktree stickiness (entity `21` / issue #104). Cycle-2 response: recommend folding into entity `21` as a sub-scope (stickiness collapses findings (a) and (e), simplifies the rule to a corollary of the stickiness invariant); rewrote `## Proposed approach` under stickiness; added the two missed source files (b); rewrote AC-3 with literal sub-clause anchors instead of proximity windows (c); rewrote AC-2 to require line-adjacent fixture placement plus a control case that reproduces PR #176/#177's conflict shape (d); dropped AC-4/AC-5/AC-6 from cycle-1 (they collapse under stickiness, with a fallback path described if the captain instead wants to ship standalone before `21` lands).

## Stage Report: ideation (cycle 2)

- DONE: Address staff-review finding (a) — stale-worktree handling.
  Under stickiness, `worktree:` set ↔ live worktree on disk is the invariant; orphan handling stays at boot (existing `status --boot` orphan detection). Documented in `### Findings under stickiness` finding (a). Defensive fallback noted as optional one-liner, not load-bearing.
- DONE: Address staff-review finding (b) — source-file list incomplete.
  Added `skills/first-officer/references/first-officer-shared-core.md:228` (Entity body content carve-out) and `skills/first-officer/references/claude-first-officer-runtime.md:155` (shutdown-sweep / feedback-cycle reader) to the source-file list in `### Source files that change (under stickiness)`.
- DONE: Address staff-review finding (c) — AC-3 grep too loose.
  AC-3 now anchors on two literal substrings (`When \`worktree:\` is set` inside the FO Write Scope `### Feedback Cycles` bullet, and `worktree-side when \`worktree:\` is set, main-side otherwise` in the Feedback Rejection Flow section), extracted from the surrounding paragraph by anchored regex. No proximity-window check.
- DONE: Address staff-review finding (d) — AC-6 fixture risks passing by luck.
  Cycle-1's AC-6 dropped under stickiness. AC-2 rewritten to require: (1) the fixture's cycle-1 stage reports placed line-adjacent to where the cycle-2 entries land, so the trailing region is genuinely overlapping; (2) a treatment case (worktree-side cycle entry) that must merge cleanly; (3) a control case (main-side cycle entry, old rule) that must reproduce the conflict — proving the test exercises the actual failure shape.
- DONE: Address staff-review finding (e) — cycle-counting / session-resume.
  Under stickiness, `### Feedback Cycles` lives in one view per entity at any time. Cycle-counter and shutdown-sweep readers target that single view via the same `worktree:` field they already read for other purposes. No read-both-views shim needed. Documented in `### Findings under stickiness` finding (e). If the design ships standalone (against the recommendation), this becomes an explicit AC.
- DONE: Major reframe — stickiness semantics; decide standalone vs fold-in.
  Recommendation: **fold into entity `21` as a sub-scope.** Rationale in `## Recommendation: fold into entity 21`: stickiness is the invariant that makes the cycle-routing rule sound; without it, the staff-review edge cases come back as live concerns; the merge-conflict pain is empirical motivation for `21` itself. Standalone form is preserved as a fallback in case the captain prefers to ship before `21` lands, with the explicit precondition that stickiness has not yet shipped (so the AC-4/AC-5/AC-6 shims must be added back).
- DONE: Append a `### Feedback Cycles` entry on this entity (eat our own dog food).
  Added `### Feedback Cycles` section above this report with the cycle-1 → cycle-2 entry naming the rejection findings and the cycle-2 response. Note: this entry is being written on `main` because this entity has `worktree:` empty (it has never been dispatched to a worktree-backed stage, since ideation is its first non-backlog stage). The same entry would have landed on the worktree branch under stickiness if the entity had ever been dispatched to implementation. This is the architecturally correct location under both old and new rules — confirming the design at the same time as exercising it.

### Summary

Cycle-2 reframes the design under stage-worktree stickiness (entity `21` / issue #104) and recommends folding this task into `21` as a sub-scope. Under stickiness, three of the six cycle-1 ACs collapse (empty-worktree case, bare-mode equivalence, no-migration policy), and the routing rule becomes a one-line corollary of the stickiness invariant rather than a standalone conditional. The remaining three ACs (worktree-side write, offline merge-clean with control case, shared-core wording with literal anchors) address the staff-review findings directly. If the captain prefers to ship before `21` lands, the standalone form's shims are documented as a fallback path with their preconditions called out.
