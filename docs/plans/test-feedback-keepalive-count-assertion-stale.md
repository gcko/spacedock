---
id: na9ydw9h5cvey0qmqeb96te8
title: test_feedback_keepalive dispatch-count assertion is stale under fresh validation
status: backlog
source: captain (CL) — observed on PR #208 cycle-3 live-CI (claude-live-opus job)
started:
completed:
verdict:
score:
worktree:
issue: "#209"
---

`tests/test_feedback_keepalive.py::test_feedback_keepalive` asserts the FO emits *exactly two* ensign `Agent()` dispatches under the feedback-keepalive flow. The current FO contract emits three because validation has `fresh: true`, forcing a fresh validation `Agent()` after the kept-alive implementer's fix lands.

Observed on PR #208 `claude-live-opus` run: 6 of 7 checks pass; the failing one is:

```
FAIL: FO emitted exactly two ensign Agent() dispatches (impl + validation; feedback via SendMessage)
  dispatch records: [('Implementation: greeting.txt', 30.2), ('Validation', 22.6), ('Validation cycle 2', 0.0)]
```

The other six checks — including "feedback routed via SendMessage to kept-alive implementation ensign" and the three static-template greps for shared-core feedback-flow rules — all pass. Only the dispatch-count assertion is stale.

Full GitHub issue: #209.

## What the test was designed to catch

The original bug: instead of `SendMessage`-routing the fixup to the kept-alive implementation ensign, the FO fresh-dispatches a *new* implementation `Agent()`. Under that bug the dispatch count would be four (impl, validation, *new impl*, re-validation) — distinct from both the legitimate-today count (three) and the obsolete-expected count (two).

## What the contract emits today

1. `Agent()` #1 — implementation (kept alive for `feedback-to: implementation`)
2. `Agent()` #2 — validation (rejects, recommends feedback)
3. SendMessage → kept-alive #1 with fixup; #1 commits the fix
4. `Agent()` #3 — re-validation, fresh because `fresh: true`

## Resolution directions (decide in ideation)

- **Option A: per-stage breakdown.** Replace the single dispatch-count assertion with three sub-assertions: exactly 1 `Agent()` whose name suffix is `implementation`, exactly 2 `Agent()`s whose name suffix is `validation`, and the SendMessage-to-kept-alive assertion (already separately checked). Preserves discrimination against the original bug (under which impl-count would be 2 and validation-count would still be 2 — fails the impl-count check).
- **Option B: rebrand to a fresh-aware total.** Assert `total Agent() dispatches == 3` and `count(name-ending-in-implementation) == 1`. Tighter but still catches the buggy scenario (impl-count = 2). Simpler than A but loses the separate validation-count signal if a future contract change adds a third validation cycle for a legitimate reason.
- **Option C: assert behavior, not counts.** Drop the dispatch-count check entirely; rely on the SendMessage-to-kept-alive positive assertion plus a NO-Agent()-with-implementation-name-after-validation negative assertion. Most robust to future contract drift but harder to read.

## Out of scope

- The other failing job on PR #208 (`claude-live` — `test_standing_teammate_spawn` step timeout: TeamCreate not emitted within 60s) is a separate concern, likely a CI flake. Track separately if it persists.
- Changing the FO contract itself (e.g., trying to make re-validation reuse instead of fresh-dispatch). The `fresh: true` on validation is intentional and not in scope here; this task only updates the test to reflect the contract as written.
