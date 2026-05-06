---
id: yj8zqrsgettqf8q3dgwtf2yg
title: "dispatch prompt template should forbid workers from writing `status:` in entity frontmatter"
status: backlog
source: "GitHub issue #189 (clkao/spacedock), split-off from `status-set-should-validate-stage-name-values` ideation gate"
started:
completed:
verdict:
score: 0.4
worktree:
issue: "#189"
pr:
mod-block:
---

Complementary defense-in-depth for issue #189: tighten the dispatch prompt template so workers explicitly know they MUST NOT write the `status:` field in entity frontmatter via direct file edit. The FO Write Scope contract already reserves frontmatter mutation to `status --set`, but mechanically nothing prevents a worker from `Edit`-ing the file in their worktree.

Real-world hit at carlove (referenced in issue #189): an ensign-class subagent wrote `status: review` (not a defined stage) via direct file edit during a ship-verify dispatch; the FO recovered only because the captain noticed during gate review.

This task is the **prompt-template** half of issue #189; the **CLI validation** half is being implemented as `status-set-should-validate-stage-name-values` (entity `am`, currently in implementation). Splitting was decided at that ideation gate because the two changes ship at different layers (prompt template vs. CLI), need different test surfaces (transcript fixture vs. subprocess), and ship independently.

Ideation should:

- Identify the canonical dispatch-prompt template surface (likely `claude-team build` prompt assembly in `skills/commission/bin/claude-team`, possibly also the ensign skill / agent definition).
- Decide the wording: a positive contract ("frontmatter belongs to the FO via `status --set`; do not edit YAML keys") plus a specific carve-out for body content the worker IS allowed to edit (stage report, design notes, AC).
- Specify whether the constraint applies to all frontmatter fields or only `status:` (and whether `worktree:`, `mod-block:`, `pr:`, `verdict:` should also be on the forbidden list).
- Test plan: transcript fixtures that send a worker into a worktree-stage dispatch and assert the prompt contains the forbidden-fields contract; regression test that the existing dispatched-ensign behavior on a real entity does not emit a `status:` line in any commit it makes to the worktree.
- Confirm this does NOT break any current ensign behavior — workers should already not be writing status, so the prompt change should be additive (clearer language, no behavioral surprise).
