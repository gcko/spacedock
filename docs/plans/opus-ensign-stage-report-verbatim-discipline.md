---
id: 4sr1ywz7d5s0gbv982783jrt
title: "Investigate opus ensign stage-report paraphrasing vs. checklist-item verbatim quoting"
status: backlog
source: "Surfaced by PR #234 (rdt — dispatch-prompt-as-file-pointer) CI run. Bare-mode (sonnet) and team-mode-default (haiku) live suites went green; `claude-live-opus` failed `tests/test_checklist_e2e.py::test_checklist_e2e_helper_path` on stage-report-content anchor checks (`stage report covers checklist anchor: UTF-8` + `stage report accounts for checklist item: Evidence (path + contents) recorded in the entity body`). The same test xfails on haiku via the existing `_HAIKU_XFAIL_REASON` conditional but has no opus branch. Under sonnet the ensign writes verbatim checklist-item text into its Stage Report; under opus the ensign paraphrases. This entity investigates whether the discipline needs tightening (ensign-side), the test needs loosening (test-side), or opus needs a model-conditional applymarker (escape hatch). Captain (CL) preferred the proper investigation (option 3) over the xfail escape hatch (option 1)."
started:
completed:
verdict:
score:
worktree:
---

# Investigate opus ensign stage-report paraphrasing

## Problem

On PR #234, `tests/test_checklist_e2e.py::test_checklist_e2e_helper_path` failed under opus in the `claude-live-opus` matrix with two specific check failures:

```
FAIL: stage report covers checklist anchor: UTF-8
FAIL: stage report accounts for checklist item: Evidence (path + contents) recorded in the entity body
```

10 of 12 sub-checks passed. The two failures are anchor / verbatim-substring assertions against the ensign's `## Stage Report` content. The test extracts code-span anchors (`UTF-8`) and the literal checklist-item text (`Evidence (path + contents) recorded in the entity body`) from the dispatch checklist and asserts both appear in the rendered Stage Report.

Under sonnet (bare-mode), all 12 checks pass — the ensign writes verbatim checklist-item text and the anchors carry through. Under opus, the ensign paraphrases or summarizes, so the substring checks miss.

The test has a `_HAIKU_XFAIL_REASON` conditional applymarker for the same reason on haiku (`tests/test_checklist_e2e.py:255`, `:280`), but no opus branch.

Three interpretations of the opus behavior:

- **A — ensign-side discipline drift.** The ensign skill's "Stage Report" protocol prescribes per-item DONE/SKIPPED/FAILED accounting "using the checklist item text verbatim when possible" (per `skills/ensign/references/ensign-shared-core.md`). Opus reads "when possible" as "when reasonable" and paraphrases for readability. Fix: tighten the protocol wording from "when possible" to "verbatim" / "byte-for-byte" / "do not summarize."
- **B — test-side over-strictness.** The substring assertion is too literal. Paraphrasing is acceptable if the gist is preserved. Fix: loosen the anchor check to a fuzzy / keyword match.
- **C — model-conditional escape hatch.** Accept opus paraphrasing as natural for that model. Fix: add `if model == "opus":` applymarker matching the haiku pattern.

## Proposed approach

1. **Reproduce locally on opus.** Run `make test-live-claude-opus TEST=tests/test_checklist_e2e.py::test_checklist_e2e_helper_path` to get a fresh opus stage report and compare it byte-for-byte against the sonnet stage report from the same checklist fixture. Confirm the paraphrasing is consistent across runs (not flake).
2. **Read the ensign protocol prose.** Inspect `skills/ensign/references/ensign-shared-core.md` for the Stage Report rendering instructions. Identify the exact prose that opus is interpreting liberally. Decide whether the protocol genuinely intends verbatim quoting or whether it allows paraphrasing.
3. **Pick a verdict** — A, B, or C — and write the supporting evidence into `## Design`. Decision criteria:
   - If the protocol PRESCRIBES verbatim and opus is violating it → tighten the prose (A).
   - If the protocol is ambiguous and verbatim isn't load-bearing → loosen the test (B).
   - If verbatim is unenforceable across models and the property doesn't matter for production → conditional applymarker (C).
4. **Apply the fix corresponding to the verdict.**
5. **Re-run on opus AND sonnet AND haiku** to confirm the fix doesn't regress weaker-model behavior.

## Acceptance criteria

End-state properties of the finished entity. Each AC is testable inside this entity's own deliverables.

1. **A reproducible opus stage-report fixture is committed** as evidence of the paraphrasing pattern. Captured at implementation as `docs/plans/_evidence/opus-stage-report-verbatim-discipline/opus-stage-report.md` alongside the sonnet baseline.
   - **Test:** static check — both files exist; the diff between them is non-trivial; the diff shows paraphrasing (verbatim text replaced with summarized text), not flake (random reordering of identical content).

2. **A verdict (A / B / C) is documented in `## Design` with a one-paragraph rationale.** No silent decision; the captain reads the verdict and the why.
   - **Test:** static check — `## Design` contains exactly one of the strings `Verdict: A`, `Verdict: B`, or `Verdict: C`, plus a `Rationale:` line.

3. **The chosen fix lands and is tested.** If A: ensign-shared-core prose changes are committed and a static test asserts the verbatim wording is present. If B: `tests/test_checklist_e2e.py:227` and `:224` assertions are loosened, with a fixture re-run confirming the new shape passes. If C: model-conditional applymarker pattern is added at `:255` and `:280` for opus.
   - **Test:** static check matched to the verdict; the corresponding test runs and passes under the chosen fix.

4. **Cross-matrix CI is green after the fix.** The `claude-live`, `claude-live-bare`, and `claude-live-opus` jobs all pass `test_checklist_e2e_helper_path` (modulo the existing haiku conditional). Zero new XPASS reports.
   - **Test:** CI green on the PR that ships this fix.

5. **Sibling tests audited for the same pattern.** `test_checklist_e2e_break_glass_path` (same file) and any other live tests that anchor on `_stage_report_text` substring assertions are inspected. The Stage Report names any sibling failures and whether the same fix covers them.
   - **Test:** Stage Report contains a sibling-test audit section enumerating the inspected tests and their disposition.

## Test plan

- **Empirical reproduction:** run the failing test on opus locally to capture the actual stage-report content. Cost ~$2-3, ~5 min.
- **Static checks for the chosen verdict:** AC-2's verdict marker, AC-3's static-content / loosened-assertion / conditional-marker landing.
- **Live re-run after fix:** AC-4 (CI green across the three matrices).
- **No new live test required.** This entity refines an existing test or protocol; behavior verification rides on the existing suite.

## Out of scope

- **General opus-vs-sonnet ensign behavior audit.** This entity focuses on the verbatim-quoting discipline surfaced by `test_checklist_e2e_helper_path`. Other opus-specific drift (if any) is tracked separately.
- **Switching opus to be the default test model.** Independent decision; see `test-live-claude-default-to-sonnet-and-cleanup-haiku-xfails` (slug `qfay`) for the sonnet-default discussion.
- **Tightening the haiku xfail.** The existing `_HAIKU_XFAIL_REASON` conditional is correct as-is for haiku; this entity does not touch it.

## Risks

### Risk A — verdict A (tighten ensign protocol) regresses sonnet behavior

If the prose change is too aggressive (e.g., "exact byte-for-byte"), sonnet may also start failing because of trivial differences (whitespace, punctuation). Mitigation: AC-4's cross-matrix re-run; if sonnet regresses, soften the prose.

### Risk B — verdict B (loosen test) hides real ensign drift

If we loosen the test to accept paraphrasing, we lose signal when the ensign genuinely drops checklist items. Mitigation: the loosened assertion still requires SOME match (e.g., a stable keyword) so a complete drop is still caught.

### Risk C — verdict C (opus xfail) accumulates xfail debt

Same anti-pattern as the haiku xfails this workflow is trying to clean up. Mitigation: prefer A or B; only fall back to C if both are infeasible.

## Scale context

- Spacedock version: 0.12.0+
- Builds on: PR #234 (rdt) — surfaces this regression as a downstream effect of the v2 file-pointer fix letting the test run further than before.
- Composes with: `qfay` (test-live-claude-default-to-sonnet-and-cleanup-haiku-xfails) — both entities touch xfail discipline but in non-overlapping ways. This entity addresses opus-specific paraphrasing; `qfay` addresses haiku-specific failures.
- Estimated complexity: small-medium. The investigation is ~1 hour of reading + 1 opus run. The fix is small once the verdict is chosen.
- Cost estimate: ~$10-15 in agent budget.
