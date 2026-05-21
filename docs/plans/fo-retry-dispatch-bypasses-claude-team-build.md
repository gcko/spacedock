---
id: y15kpj107feq06542eh1fsn2
title: "FO retry-path freelances when claude-team build's first dispatch fails (opus-low under 529 outage)"
status: backlog
source: "PR #231 CI run 2 (26199920245, claude-live-opus job) — under an Anthropic 529 outage cascade, the FO correctly used `claude-team build` for the first two Agent() dispatches (FETCH-on-demand shape), then on the THIRD dispatch freelanced and manual-assembled an INLINED-MANUAL prompt. Evidence in artifact `spacedock-test-rt_kxghh/fo-log.jsonl`: dispatch #1 = FETCH 1083 chars, dispatch #2 = FETCH 840 chars, dispatch #3 = INLINED 1001 chars after two prior 529s and a `SendMessage` recovery attempt. The FO's runtime-adapter contract (`do NOT assemble Agent() prompts manually`) was followed on the first two attempts and silently bypassed on the third under repeated upstream failures."
started:
completed:
verdict:
score:
worktree:
issue:
pr:
---

## Problem

When `claude-team build`-emitted dispatches produce repeated upstream failures (in the observed case: HTTP 529 Overloaded errors that immediately fail the spawned ensign), the opus-low FO eventually freelances on a subsequent retry and manually assembles an Agent() prompt instead of going through the helper. This bypasses the contract enforced in `skills/first-officer/references/claude-first-officer-runtime.md` `## Dispatch Adapter` ("Do NOT assemble Agent() prompts manually... Manual assembly is a protocol violation except in the documented break-glass fallback below").

The break-glass fallback exists for the case where `claude-team build` ITSELF exits non-zero. In the observed case, the helper exited cleanly and emitted a correct fetch-on-demand spec — the FO used it twice. The failure was downstream (ensign-spawn-time 529 error), not in the helper. The FO's recovery path doesn't currently distinguish "helper failed" from "ensign failed during spawn" and the latter case triggers freelance behavior that isn't justified by the contract.

## Evidence

Reproduced once, in PR #231 CI run 2 (`spacedock-test-rt_kxghh/fo-log.jsonl`), opus-low + teams-mode dispatch on `test_fetch_on_demand_dispatch_runs_fetch_commands_and_writes_stage_report`:

| Dispatch # | Event idx | Shape | Prompt length | Outcome |
|---|---|---|---|---|
| 1 | 28 | FETCH (`### Fetch commands` present) | 1083 chars | ❌ 529 Overloaded |
| 2 | 37 | FETCH | 840 chars | ❌ 529 Overloaded |
| 3 | 51 | **INLINED-MANUAL** (verbatim stage-def inlined; no fetch block) | 1001 chars | ❌ 529 Overloaded |

The third prompt is qualitatively different from the first two. Where the FETCH-shaped prompts read like helper output, the INLINED-MANUAL prompt reads like the FO authored from a mental model: `Stage definition (from README):` followed by the verbatim README subsection, `Commits should be prefixed 'work:'`, etc. These are phrasings that the helper does not emit.

The test (AC-8) checks the LAST dispatch's prompt (via `LogParser.agent_prompt()`, see `scripts/test_lib.py:1802-1807`), so it caught the bypass. The contract violation was real even though it manifested under an upstream outage — the FO's response to repeated failures shouldn't be to abandon the helper.

## Why this matters

- **Contract integrity.** The "no manual assembly" rule is one of two structural guarantees the runtime adapter makes about dispatch (the other is shape consistency via `claude-team build`). If the FO freelances on retries, neither guarantee survives stress.
- **Cross-runtime consistency.** As the helper splits into a runtime-neutral core (separate entity `mtc76hh8`) and runtime adapters, manual assembly diverges per-FO. Codex FO and Claude FO would produce different-shaped manual prompts under load.
- **Test confidence.** The bug was caught by a test that explicitly asserts fetch-on-demand shape. Without AC-8 from the 0x9 PR, this would have shipped silently.

## Proposed approach

This entity proposes a small FO-runtime-adapter prose tightening, with empirical validation.

**Prose change candidate** in `skills/first-officer/references/claude-first-officer-runtime.md` `## Dispatch Adapter`:

> "On Agent dispatch failure (529, 503, generic capacity errors), the FO MUST NOT manually assemble a replacement prompt. The retry path is: (a) wait via `claude-team context-budget` or a short sleep; (b) re-invoke `claude-team build` from the same input JSON to get a fresh helper-emitted spec; (c) re-dispatch via `Agent()` with the helper's output verbatim. The break-glass manual template is reserved for cases where `claude-team build` ITSELF exits non-zero, not cases where the helper succeeded and the dispatched ensign failed."

This is a sub-bullet under the existing "MANDATORY — Dispatch assembly via `claude-team build`" section, scoped to the retry case.

**Empirical validation:** the spike harness from the haiku-bare-fo-startup-protocol-adherence entity (`/tmp/nwq-spike/run-spike.sh`) can be adapted — substitute the bare-haiku fixture with a 529-injection harness (mock the Anthropic API to return 529 for the first N Agent calls) and measure whether the FO retries with FETCH shape vs INLINED-MANUAL shape across 5 runs before/after the prose change.

If the prose change measurably reduces INLINED-MANUAL fallback rate (>80% FETCH-shape on retries), land it. If not, consider whether a more structural fix is needed (e.g., a `claude-team build --retry-mode` flag that bakes "this is a retry; helper succeeded last time" into the FO's mental model).

## Acceptance criteria

End-state properties of the finished entity:

1. **The `## Dispatch Adapter` section in `claude-first-officer-runtime.md` contains an explicit sub-bullet for the retry case**, distinguishing "helper failed → break-glass" from "ensign failed after helper succeeded → re-invoke helper". (Verified by static-content grep.)

2. **The break-glass manual-assembly template's prose explicitly names that it is NOT for the ensign-spawn-failure retry case.** (Verified by static-content grep.)

3. **An empirical validation has been recorded in the entity body** measuring INLINED-MANUAL fallback rate before vs after the prose change under simulated 529 failures, with at least 5 runs per side. Verdict named explicitly: PASS / INCONCLUSIVE / FAIL.

4. **`make test-static` passes with the prose change applied** — no regressions from the contract update.

5. **`tests/test_fetch_on_demand_dispatch.py` (the AC-8 test from 0x9) is NOT modified by this entity.** This entity addresses the FO behavior the test caught; the test itself stays as the assertion surface.

## Out of scope

- The Anthropic 529 outage itself. That's an upstream environmental issue, not Spacedock-fixable.
- Adding retry semantics to `claude-team build` itself. The helper is stateless and shouldn't track retry context — that's the FO's job.
- The `_archive/haiku-bare-fo-guardrail-weaknesses.md` (#200) class of bare-haiku-FO startup failures (different entity: `haiku-bare-fo-startup-protocol-adherence`, currently in-flight).

## Scale context

- Spacedock version: 0.11.2+ (after PR #231 fetch-on-demand contract)
- Surfaced by: PR #231 CI run 2 artifact, `spacedock-test-rt_kxghh`. Run 1 of the same source did NOT manifest the bug because that run's dispatches succeeded on first try (no 529 cascade).
- Estimated complexity: small. ~10 lines of prose edits to one reference file, ~50 lines of test additions if empirical validation requires a 529-injection harness. The empirical validation may be skippable if the prose change is judged self-evidently correct on review.
- Related: archived 0x9 (`claude-team-build-fetch-on-demand-dispatch-spec`) introduced the fetch-on-demand contract this bug interacts with. Related: in-flight `haiku-bare-fo-startup-protocol-adherence` (`nwwqsx5q`), a sibling entity also about FO prose tightening.
