---
id: amrd1r63jfkjq14tvwnkfges
title: "status --set should validate stage-name values against the workflow's stages.states[].name list"
status: ideation
source: GitHub issue #189 (clkao/spacedock)
started: 2026-05-06T08:39:40Z
completed:
verdict:
score: 0.55
worktree:
issue: "#189"
pr:
mod-block:
---

`status --set {slug} status={value}` accepts arbitrary string values without checking against the workflow README's `stages.states[].name` list. Workers (especially LLM-driven ensigns) can silently write semantically-invalid values like `status: review` when the workflow defines `verify → ship → done`. The result is a quietly broken state machine: subsequent dispatches based on `status --next` skip the entity, downstream gates never fire, and recovery depends on human review.

Real-world hit at carlove: an ensign-class subagent wrote `status: review` (not a defined stage) via direct file edit during a ship-verify dispatch; the FO recovered by rolling status back, but only because the captain noticed during gate review.

Issue #189 proposes:

1. `status --set status={value}` (and any path mutating `status:`) reads the README's `stages.states[].name` list at invocation time and rejects unknown values with an explicit error naming the known stages.
2. Optional `--force` bypass for legitimate schema-evolution cases (mid-flight rename of a stage), with a captain-visible warning.
3. Complementary tightening of the dispatch prompt template so workers MUST NOT write `status:` in frontmatter — closing the bypass route at the source.

Ideation should decide whether to scope this entity to the `status` binary validation only, or to also cover the worker-prompt tightening. Test plan must cover: (a) valid stage names accepted; (b) typos like `desgin` rejected with the stages list in the error; (c) `--force` bypass; (d) the validator reads from `--workflow-dir`'s README, not a cached schema.
