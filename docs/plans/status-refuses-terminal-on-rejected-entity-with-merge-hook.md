---
id: 842ja5phzj5xspyternpww31
title: "status binary refuses terminal advancement on rejected/abandoned entities when merge hook is registered, even when no PR is intended"
status: ideation
source: GitHub issue #188 (clkao/spacedock)
started: 2026-05-06T08:39:39Z
completed:
verdict:
score: 0.55
worktree:
issue: "#188"
pr:
mod-block:
---

`status --set status={terminal} verdict=rejected completed` is refused when the workflow has any registered `## Hook: merge` mod AND `pr` is empty AND `mod-block` is empty. The captain MUST pass `--force` to bypass.

This is correct enforcement when an entity was meant to ship through a PR but skipped the hook. It misfires when the captain explicitly rejected the work — there is no PR to wait for, the merge hook should not run, and forcing `--force` makes the audit history harder to read ("did the captain bypass because they explicitly rejected, or did they fat-finger past a guard?").

Issue #188 lays out three candidate fix shapes:

1. `verdict=rejected` (or any non-PASSED verdict) implicitly bypasses the merge-hook gate; the verdict itself records "this is not a ship".
2. Recognize an explicit `verdict=abandoned` distinct from `rejected` and exempt only that verdict.
3. Document the `--force` requirement on the abandon path and improve the refusal message to suggest `--force` for rejection cases.

Ideation should pick one shape, weighing audit clarity vs. additional verdict surface area. Test plan must cover: (a) rejected-with-no-PR-and-merge-hook flow succeeds without `--force`; (b) ship-flow without `mod-block` still refuses (the original guard still fires); (c) audit-history clarity in both paths.
