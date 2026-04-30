---
id: h4bhyezqbehpatscwydsp8rz
title: "Codex checklist E2E parser over-collects post-checklist protocol bullets"
status: backlog
source: "PR #176 and PR #177 codex-live CI failures on 2026-04-30; both failed tests/test_checklist_e2e.py::test_checklist_e2e_codex after the worker produced a substantively valid stage report, because the test parser treated post-checklist protocol bullets as checklist items."
started:
completed:
verdict:
score: 0.56
worktree:
issue:
pr:
mod-block:
---

## Why this matters

PR #176 (`commission-readme-portable-status-path`) and PR #177 (`debrief-tolerate-missing-workflow-status`) both failed `codex-live` on the same ambient test-harness issue:

- PR #176: run `25193091046`, job `73867530125`, artifact `spacedock-test-ubxs6i_m`, `test_checklist_e2e_codex` failed 6 of 20 checks.
- PR #177: run `25193155461`, job `73867732326`, artifact `spacedock-test-qkt0gdtz`, `test_checklist_e2e_codex` failed 7 of 21 checks.

In both artifacts, the Codex worker created `checklist-pipeline/output.txt`, recorded evidence, appended `## Stage Report: work`, used `DONE:` markers, and included `### Summary`. The failure came from `tests/test_checklist_e2e.py::_extract_checklist_items()`: after seeing `### Completion checklist`, it continued collecting later bullets under headings like `Required worker behavior:` or `Required completion protocol:`. The harness then incorrectly required the stage report to account for FO/worker protocol bullets such as "Commit all stage work..." and anchors like `### Completion checklist` / `work:`.

This is not a product regression from PR #176 or PR #177. It is a parser-boundary bug in the checklist E2E harness.

## Proposed approach

Patch `_extract_checklist_items()` so it treats the completion checklist as a bounded section:

- Start collecting only after the completion-checklist heading.
- Collect numbered or bulleted checklist items.
- Once at least one checklist item has been collected, stop at a non-list section label such as `Required worker behavior:` or `Required completion protocol:`.
- Keep the existing explicit stop-heading compatibility for older prompt variants.

Prefer section-boundary behavior over adding only the two observed heading strings, so future protocol sections after the checklist do not become false checklist items.

## Acceptance criteria

**AC-1 - Parser stops before post-checklist protocol sections.**
Verified by: a unit test feeds a prompt shaped like PR #176, with three numbered checklist items followed by `Required completion protocol:` and protocol bullets. `_extract_checklist_items()` returns only the three real checklist items.

**AC-2 - Parser stops before "Required worker behavior" sections.**
Verified by: a unit test feeds a prompt shaped like PR #177, with two numbered checklist items followed by `Required worker behavior:` and protocol bullets. `_extract_checklist_items()` returns only the two real checklist items.

**AC-3 - Existing checklist variants remain supported.**
Verified by: existing `Completion checklist:`, `Completion checklist (linchpins):`, and `### Completion checklist` parser behavior still passes for numbered and bulleted checklist forms.

**AC-4 - Static verification is enough.**
Verified by: the targeted parser/unit tests and `make test-static` pass. No live Codex rerun is required unless needed for branch protection, because the failure is reproducible from captured log shapes and the fix is pure parser logic.

## Test plan

- Add narrow unit coverage in or near `tests/test_checklist_e2e.py` for the two observed Codex prompt shapes.
- Run the targeted test module or parser tests.
- Run `make test-static`.

## Out of scope

- Changing Codex FO dispatch wording.
- Changing ensign stage-report requirements.
- Reworking `test_checklist_e2e_codex` live orchestration.
- Treating PR #176 or PR #177 implementation work as suspect based on this failure alone.
