---
id: "214"
title: "Stage worktree stickiness — once in a worktree, stay until terminal"
status: backlog
source: GitHub issue #104 (filed by CL). Originally surfaced via the spacedock-prompt/experiments workflow (template-simplification variants for FO, variant 015 tighten-hedging). Today's session (2026-04-30) extended scope: PR #176, #177, #180 each hit entity-body merge conflicts because the FO writes `### Feedback Cycles` to main while the worktree branch writes stage reports to the same trailing region — folded entity `k9s` (`feedback-cycles-on-worktree-not-main`, archived) into this task as a sub-scope on captain direction.
started:
completed:
verdict:
score: 0.65
worktree:
issue: "#104"
pr:
mod-block:
---

# Clarify stage worktree semantics: post-worktree stickiness

## Problem statement

The workflow stage model lets each stage declare `worktree: true|false` (with a per-workflow default in `stages.defaults`). When a stage with `worktree: true` is followed by a stage that inherits `worktree: false`, the downstream stage strands its prior artifacts:

- Run-stage outputs (files written under the worktree, branches on a submodule's `experiment/{slug}` branch, uncommitted WIP) are only visible from within the worktree.
- A downstream `worktree: false` stage would nominally dispatch against main — where those artifacts do not exist until the terminal merge.

**Concrete example** (from the experiments workflow that surfaced this):

```yaml
stages:
  defaults:
    worktree: false
  states:
    - name: hypothesis
    - name: run
      worktree: true
    - name: analyze        # inherits worktree: false
      feedback-to: run
    - name: done
      terminal: true
```

- `run` writes `_results/{slug}.json` and commits template diffs on branch `spacedock-ensign/{slug}` with a matching `experiment/{slug}` submodule branch.
- `analyze` per README takes `_results/{slug}.json` as input and produces an adopt/reject/combine recommendation.
- But `analyze` inherits `worktree: false`, so the FO would dispatch it against main — where the results JSON and the submodule branch do not exist yet.
- The FO has to override the config to keep `analyze` inside the `run` worktree for it to work at all.
- If `analyze` rejects and `feedback-to: run` routes back, the feedback fix obviously needs to land in the same worktree — reinforcing that once an entity enters a worktree stage, it should not leave it until the terminal merge.

## FO contract intersection

The shared-core `## Completion and Gates` reuse conditions currently include:

> **Reuse conditions** (all must hold — if any fails, dispatch fresh):
> 3. Next stage has the same `worktree` mode as the completed stage

(`skills/first-officer/references/first-officer-shared-core.md:112`)

With stickiness, this condition becomes meaningless or needs rewording — once an entity is in a worktree, the downstream stage's declared `worktree:` value should not gate reuse or dispatch location.

The "worktree mode" field on a per-stage basis also needs reinterpretation: it becomes "if no worktree exists yet, create one at this stage" rather than "run this stage in a worktree or on main."

## Alternatives (from issue body)

### A — Stickiness by default

Once an entity has been dispatched to any stage with `worktree: true`, all subsequent non-terminal stages for that entity operate in the same worktree, regardless of their own `worktree` field. The `worktree:` field per stage then means "create a worktree if none exists yet" rather than "run in a worktree or on main." Terminal (`done`) stage retains existing merge-and-cleanup semantics.

Simplest. Matches current de-facto operational need. FO keeps `worktree:` on entity frontmatter across run → analyze, clears on terminal merge.

### B — Explicit per-stage opt-out

Introduce `worktree: inherit-from-previous` or similar. More verbose, lets a workflow opt-out of stickiness if needed. Probably YAGNI — can't think of a real reason to un-stick.

### C — Require downstream stages to redeclare `worktree: true`

Less surprising syntactically but fragile — any author forgetting to redeclare trips the same trap.

**CL's lean:** A. Captured here for ideation to confirm or push back with evidence.

## Design questions for ideation

1. **Interaction with `fresh: true`.** Stickiness is worktree-lifecycle; `fresh` is agent-lifecycle. Under stickiness, if stage B has `fresh: true` and entity is in a worktree from stage A (`worktree: true`), the correct behavior is: fresh agent dispatched into the *same* worktree. Worth calling out explicitly in the contract so workflow authors don't conflate the two.

2. **Interaction with `feedback-to`.** Already implicit: feedback rejection flow already keeps the fix agent in the same worktree. Stickiness makes this uniform across all advancement paths, not just rejection-routing.

3. **What does the per-stage `worktree:` field mean post-change?** Probably: "create a worktree at this stage if one does not exist for this entity yet." So a workflow with `worktree: true` only on a middle stage means "entity runs on main until reaching this stage; from here on, in a worktree until terminal." A workflow with `worktree: true` on the initial stage means "always in a worktree." A workflow with `worktree: true` nowhere means "always on main."

4. **Documentation surfaces.** README stages section in every self-hosted workflow documents stage semantics. The commission skill's README template should explain stickiness. The FO shared-core and runtime adapter prose need the updated rules.

5. **Migration for existing workflows.** All existing in-repo workflows use `worktree: true` on implementation + validation (and nowhere else) or everywhere or nowhere. Stickiness doesn't change their behavior — the run of worktree-true stages is contiguous. Only external workflows with a non-contiguous pattern (like the experiments one) are affected, and they're affected positively. No migration needed.

## Acceptance criteria (draft; ideation to refine)

**AC-1 — Shared-core reuse condition #3 is reworded or removed.**
Verified by: `skills/first-officer/references/first-officer-shared-core.md` no longer has the exact phrase "Next stage has the same `worktree` mode as the completed stage" at or near line 112. Replacement prose makes stickiness explicit. Grep assertion in `tests/test_agent_content.py` or similar.

**AC-2 — FO stamps worktree on entity frontmatter at first worktree-true stage and preserves it across subsequent advancements.**
Verified by: a new unit/integration test in `tests/test_status_script.py` (or a new file) simulates an entity advancing through a `worktree: true` → `worktree: false` (inherited) stage pair and asserts the `worktree:` frontmatter field stays set. `status --set {slug} status={next}` without an explicit `worktree=` argument preserves the existing value; currently this is already the case for `status --set` semantics, but the assertion gates future regression.

**AC-3 — Terminal-stage advancement still clears worktree.**
Verified by: existing archive flow (`status --archive`) clears `worktree:` when moving to terminal; covered by current tests — just confirm no regression.

**AC-4 — Commission README template documents stickiness.**
Verified by: `skills/commission/SKILL.md` and/or the generated README template includes a line like "Once an entity is dispatched to a stage with `worktree: true`, all subsequent non-terminal stages operate in the same worktree" in the stage-semantics section.

**AC-5 — `claude-team build` prompt assembly respects stickiness.**
Verified by: integration test or prompt-replay — when an entity is in a worktree and the next stage has `worktree: false` inherited, `cmd_build` still emits worktree-aware dispatch (prompt points at the worktree, not main). If `cmd_build` reads the entity's `worktree:` frontmatter field rather than re-computing from stage config, this may already work by construction; ideation confirms or refutes.

## Out of scope

- Runtime support for git submodules inside worktrees (the experiment workflow's pattern). This entity is about FO contract semantics, not submodule plumbing.
- Changing the archive flow's worktree-cleanup behavior.
- Introducing new `worktree:` field values beyond `true|false`. Alternative B in the issue body is rejected as YAGNI.
- Migrating existing in-repo workflows (none have the non-contiguous pattern).

## Test plan

Three surfaces:
1. **Shared-core grep / contract assertion** — `tests/test_agent_content.py` or sibling. Assert the reworded rule is present; assert the old rule is gone.
2. **Status script integration test** — `tests/test_status_script.py`. Simulate an entity frontmatter with `worktree: .worktrees/...` + advance through `status --set status={next_non_terminal}` without passing `worktree=`. Assert the field is preserved. Then simulate terminal advancement and assert it clears.
3. **FO dispatch behavior test** — likely in `tests/test_dispatch_names.py` or a new `tests/test_worktree_stickiness.py`. A minimal 3-stage fixture (`initial` → `middle worktree: true` → `terminal`), dispatch initial (no worktree), advance to middle (worktree created + stamped), advance to terminal (worktree cleared + merged). Can be offline / bare-mode.

Estimated cost: low-to-medium. Shared-core edit + status script preservation check + one test file. 30-60 min implementation. No live CI matrix needed.

## Cross-references

- GitHub issue #104
- `skills/first-officer/references/first-officer-shared-core.md:112` — reuse condition #3 (the rule that needs rewording)
- `skills/commission/bin/claude-team::cmd_build` — dispatch prompt assembly; check whether it reads entity `worktree:` field or re-derives from stage config
- `skills/commission/bin/status::_set` + `_archive` — frontmatter field preservation logic
- External workflow `spacedock-prompt/experiments` — originating use case (not in this repo)
- **Folded sub-scope (entity `k9s` archived 2026-04-30):** `### Feedback Cycles` routing under stickiness — see `## Folded sub-scope: Feedback Cycles routing` below.

## Folded sub-scope: Feedback Cycles routing

Folded from entity `k9s` (`feedback-cycles-on-worktree-not-main`) on 2026-04-30 per captain direction after the cycle-2 ideation gate. PR #176 / #177 / #180 in this session each hit a merge conflict in the entity body because the FO writes `### Feedback Cycles` to main while the worktree branch writes stage reports to the same trailing region.

Under the stickiness invariant from this entity, the cycle-routing rule becomes a one-line corollary: cycle entries are entity-body content, entity-body content during in-flight stages lives where stage reports live, stage reports live in the worktree under stickiness — therefore cycle entries live in the worktree while `worktree:` is set, and on main only before the first worktree-creating dispatch. The terminal merge brings the worktree-side entries onto main.

Concrete shared-core changes folded in (in addition to the reuse-condition #3 reword named in `## FO contract intersection`):

- **`## FO Write Scope`** (`first-officer-shared-core.md:218`) — replace the unconditional `### Feedback Cycles` bullet with a sub-clause naming both states: `When \`worktree:\` is set, the FO writes the cycle entry to the worktree copy and commits on the worktree branch; when \`worktree:\` is empty, the FO writes to main.` Anchor phrase: `When \`worktree:\` is set`.
- **`## Feedback Rejection Flow`** (`first-officer-shared-core.md:178`) — replace `keeps it on the main branch` with `Routing follows FO Write Scope: worktree-side when \`worktree:\` is set, main-side otherwise.`
- **`## Worktree Ownership`** (`first-officer-shared-core.md:206`) — add `### Feedback Cycles` to the list of body content that stays in the worktree during in-flight stages.
- **`## FO Write Scope` "Entity body content beyond `### Feedback Cycles`"** (`first-officer-shared-core.md:228`) — clarify the carve-out applies in the appropriate view.
- **`claude-first-officer-runtime.md:155`** (cooperative-shutdown sweep / cycle-detection reader) — add a one-line view-clarification parenthetical so the reader operates on the worktree copy when `worktree:` is set.

Folded ACs (renumber as part of `21`'s final AC list during `21`'s ideation):

- **AC (Feedback Cycles, worktree-side write):** When `worktree:` is set on an entity, the FO writes new `### Feedback Cycles` entries inside the worktree copy of the entity file and commits on the worktree branch; the main copy of the entity file is untouched until PR merge. Verified by extending `tests/test_rejection_flow.py` with two assertions after the FO routes feedback back to implementation: `git -C {worktree_path} show HEAD:{workflow_dir}/{slug}.md` contains the new cycle entry; `git show main:{workflow_dir}/{slug}.md` does not contain it.
- **AC (Feedback Cycles, merge-clean with control case):** Two consecutive feedback cycles on a worktree-backed entity merge into main with zero conflicts in the entity file, and a control case proves the test exercises the actual PR #176/#177 conflict shape. Verified by a new offline test `tests/test_feedback_cycles_merge_clean.py` plus a fixture under `tests/fixtures/feedback-cycles-merge/`. Treatment case (worktree-side cycle entry placed line-adjacent to stage-report region) merges clean (`git merge` exit 0, zero `<<<<<<<` markers). Control case (cycle entry on main, old rule) reproduces the conflict — proving the test is not passing because of benign whitespace separation.
- **AC (Feedback Cycles, shared-core wording with structural anchors):** `first-officer-shared-core.md` describes the conditional routing rule with anchors readers and tests can rely on. Verified by extending `tests/test_repo_edit_guardrail.py` static-content phase with two literal-substring assertions: `When \`worktree:\` is set` inside the FO Write Scope `### Feedback Cycles` bullet's surrounding paragraph; `worktree-side when \`worktree:\` is set, main-side otherwise` in the Feedback Rejection Flow section.

The standalone-shipping form of `k9s` (with additional ACs for empty-worktree, bare-mode equivalence, no-migration policy) is preserved in the archived entity body for reference; under stickiness those three ACs collapse and are not folded in.

The full cycle-2 design rationale (architectural answers, findings-under-stickiness, test plan) lives in `docs/plans/_archive/feedback-cycles-on-worktree-not-main.md`. The ideation worker on `21` should use it as load-bearing input for the Feedback Cycles portion of `21`'s eventual design.

## Summary

Today the FO has to work around a gap: once an entity runs a `worktree: true` stage, the next inherited-default stage lies to the operator by claiming it should run on main, even though its inputs are in the worktree. The proposed fix (Option A, stickiness by default) makes the implicit contract explicit, simplifies the reuse-condition logic, and matches CL's operational workaround. Needs an ideation pass to lock in exact contract wording + test coverage, then a surgical implementation across shared-core prose + README template + one or two test files.
