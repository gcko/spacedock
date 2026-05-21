---
id: nwwqsx5qh090mf6sm12ajee9
title: "Haiku-bare FO startup-protocol adherence: prose-hardening attempt at #200's medium-term branch"
status: ideation
source: "PR #231 CI surfaced the exact #200 Pattern A flake (haiku-bare-FO cd's to $HOME, freelances ahead of `status --boot`, declares 'no workflow found') AND today's FO log artifact shows the FO never runs the canonical `status --boot` first call — it freelances with cd/find/ls/cat. #200 closed with only its near-term xfail bandaid landing (and that marker has since drifted out of `test_gate_guardrail.py`). The medium-term FO-prose-hardening branch was never chased. Captain decision today (2026-05-21): attempt option C (prose tightening) with an empirical stop-loss; if it doesn't measurably move the needle, recommend option B (retire bare-haiku coverage) as the honest accounting."
started: 2026-05-21T01:56:33Z
completed:
verdict:
score:
worktree:
issue:
pr:
---

## Problem

`test_gate_guardrail` fails on `claude-live-bare` (and equivalent local runs) because the haiku-bare FO does not follow the documented startup protocol. The protocol in `skills/first-officer/references/first-officer-shared-core.md` says:

> 4. Run `status --boot` for all startup information in one call.

But today's CI artifact (PR #231, `claude-live-bare`, run 26198088404, FO log `spacedock-test-5tfsjaww/fo-log.jsonl`) shows the haiku-bare FO does *not* run `status --boot` first. Instead it freelances:

1. `echo "CLAUDECODE env: ${CLAUDECODE:-not set}"`
2. `cd /tmp/spacedock-clean-home-oguncxnd && git rev-parse --show-toplevel` ← cd'd to `$HOME`, not the test-project cwd
3. `find ... README.md ...`
4. `ls /tmp/spacedock-clean-home-oguncxnd/`
5. Only THEN runs `status --discover` — but from the wrong cwd, so it returns empty.
6. Final assistant text: *"No workflow directory was discovered. To proceed, I need one of: 1. Explicit workflow path: ..."* — FO gives up and asks the user.

This is the exact Pattern A failure mode documented in `_archive/haiku-bare-fo-guardrail-weaknesses.md` (#200). #200 closed with a near-term xfail marker; that marker has since drifted out (`tests/test_gate_guardrail.py` has no `xfail`/`skipif` markers today). The medium-term FO-prose-hardening work was deferred and never picked up.

## Why this matters

- `claude-live-bare` CI failures repeatedly block PRs (PR #210 and now PR #231 both observed the same flake-shape). Each time, the FO has to manually inspect artifacts to confirm pre-existing-flake rather than regression. Repeated drain on captain attention.
- The xfail bandaid is fragile — it drifts away across refactors and its model-alias predicate misses runtime model names CI actually uses (`haiku` vs `claude-haiku-4-5` vs `claude-haiku-4-5-20251001`).
- The protocol exists and is correct; the model is just not following it. A small prose change might be enough to push haiku-low onto the protocol path. If not, the honest answer is to retire the bare-haiku combination.

## Proposed approach (with stop-loss)

This entity has TWO outcomes — `PASSED` (prose change shipped, measurably reduces haiku-bare cwd-drift) or `REJECTED-with-recommendation` (prose change didn't help; recommend separate entity to retire bare-haiku coverage).

### Prose changes to attempt (`skills/first-officer/references/first-officer-shared-core.md`, Startup section)

The current Startup section lists six numbered steps. Specific changes proposed:

1. **Reorder step 4 to step 1.** Move `status --boot` to the FIRST step explicitly. The current ordering invites freelancing: "discover project root with `git rev-parse --show-toplevel`" comes first, and when that errors (clean-home test env, no git repo at cwd), haiku reasons forward by searching.
2. **Add an explicit prohibition on freelancing ahead of `status --boot`.** Sentence: *"Your FIRST Bash call MUST be `status --boot`. Do not run `cd`, `find`, `ls`, `git rev-parse`, or any other Bash command before `status --boot` succeeds. The boot probe is the single source of truth for workflow state — do not infer it from other signals."*
3. **Add a stop-and-ask directive for the no-workflow case.** Sentence: *"If `status --boot` returns no workflow (empty DISPATCHABLE, no MODS, no PR_STATE), stop and ask the captain for an explicit workflow path. Do NOT search the filesystem; the captain will provide the path."*
4. **(Optional) Add a `status --startup-sanity` helper.** A new tiny subcommand that runs `git rev-parse --show-toplevel` and the discovery probe internally, fails loudly with a single-line diagnostic message if cwd looks wrong. The FO is then instructed to run this BEFORE `status --boot` so the failure mode surfaces as an error the model can recognize rather than an empty result the model reasons around. Decision on whether to add this falls out of the spike below.

### Empirical spike (gates the design)

Before locking the design, run a small spike to measure whether the proposed prose changes actually move the needle:

1. **Baseline measurement.** Set up the `test_gate_guardrail` fixture (or a stripped-down equivalent) locally. Run `pytest tests/test_gate_guardrail.py --runtime claude --model haiku-bare-mode` (or equivalent) 5 times. Record: did the FO's first Bash call equal `status --boot`? Did the test pass? Capture fo-log.jsonl for each run.
2. **Apply candidate prose changes** (rewritten `## Startup` section).
3. **Re-measurement.** Re-run 5 times against the same fixture. Record same observables.
4. **Verdict criteria:**
   - PASS: First-Bash-call-is-`status --boot` rate goes from <50% to >80% across 5 runs. AND test pass rate increases.
   - INCONCLUSIVE: First-Bash-call rate moves but test still fails for other reasons (suggests additional Pattern A facets beyond cwd drift). Report findings; decide whether to extend prose changes or recommend B.
   - FAIL: First-Bash-call rate doesn't change meaningfully. Prose tightening alone isn't enough; recommend option B in the validation report.

The spike artifacts (fo-log.jsonl files before/after, comparison summary) go under `### Spike report` in the entity body.

## Acceptance criteria

End-state properties of the finished entity. Each verifiable by a future reader.

1. **The Startup section of `skills/first-officer/references/first-officer-shared-core.md` contains a sentence requiring `status --boot` as the FIRST Bash call**, with an explicit prohibition on `cd`/`find`/`ls`/`git rev-parse` ahead of it. The current Startup section's step ordering is also reorganized so `status --boot` is the first numbered step (not the fourth).
   - **Test:** static-content test in `tests/test_agent_content.py` (extension) asserting both properties: (a) Startup step 1 mentions `status --boot`; (b) the prohibition sentence is present verbatim.

2. **The Startup section contains a stop-and-ask directive for the empty-discovery case**: if `status --boot` returns no workflow signals, the FO must stop and ask the captain for an explicit workflow path, rather than searching the filesystem.
   - **Test:** static-content test asserts the stop-and-ask sentence is present.

3. **The spike report in the entity body documents a measured before/after comparison** with at least 5 runs per side. Each run records (a) whether the FO's first Bash call equaled `status --boot`, (b) whether `test_gate_guardrail` passed, (c) the cwd the FO operated from. The spike's verdict (PASS / INCONCLUSIVE / FAIL) is named explicitly in the report.
   - **Test:** the report exists in the entity body; reviewer cross-checks the recorded fo-log.jsonl artifacts (preserved under `tests/fixtures/200-spike-artifacts/` or `/tmp/200-spike/`) match the recorded observations.

4. **If the spike verdict is FAIL or INCONCLUSIVE, the validation Stage Report explicitly recommends filing a follow-up entity for option B (retire bare-haiku coverage for `test_gate_guardrail` and adjacent live tests).** The recommendation names the suggested slug (`retire-bare-haiku-fo-coverage`) and what gets retired (which test files, which CI matrix entries).
   - **Test:** if applicable, conditional content check on the validation Stage Report.

5. **The shipped change does NOT modify `tests/test_gate_guardrail.py` to add an xfail marker.** This entity is specifically about the protocol-adherence prose change; xfail-marker resurrection is a separate small entity (likely to land alongside or instead of this one). Keeping them separate keeps the scope clean.
   - **Test:** `git diff` shows no changes to `tests/test_gate_guardrail.py` from this entity's PR.

## Test plan

- **`tests/test_agent_content.py`** (existing file, extend): static-content tests for AC-1 and AC-2 — assert the prose changes are present in `first-officer-shared-core.md`. ~10 lines added.
- **Spike artifacts** (under `tests/fixtures/200-spike-artifacts/` or `/tmp/200-spike/`): fo-log.jsonl files from before/after spike runs, plus a summary markdown. Committed under tests/fixtures if implementation chooses; otherwise referenced by path from the entity body.
- **`make test-static`** — confirm the offline suite passes. ~626/626 expected (same baseline as the most recent passing run).
- **`make test-live-claude` (captain's gate requirement, restated)** — run from worktree before signaling implementation complete. The validation gate explicitly checks that `test_gate_guardrail` passes (or xfails per a non-#200 marker) on the haiku-bare matrix entry. If it still fails after the prose change, the spike's INCONCLUSIVE/FAIL branch fires and the recommendation pivots to option B.

No new modules, no schema changes, no test framework changes. ~30 lines of prose edits to one reference file, ~10 lines of test additions. The spike work happens in scratch space and produces measurements, not shipped code.

## Out of scope

- Resurrecting the `@pytest.mark.xfail` marker for `test_gate_guardrail` haiku-bare (separate small entity if needed; should NOT be combined with this one).
- The opus `test_standing_teammate_spawn` timeout flake (separate entity; different failure mode, different model).
- The `claude-first-officer-runtime.md` Team Creation section. The prose changes here are scoped to `first-officer-shared-core.md`'s Startup section only.

## Scale context

- Spacedock version: 0.11.2
- Supersedes/extends: `_archive/haiku-bare-fo-guardrail-weaknesses.md` (#200) — specifically the medium-term branch #200 deferred.
- Related: `_archive/fo-cwd-drift-bug.md` (#072, different surface: cwd drift after worktree commands, not at startup), GitHub issue #219 (Bash wedge after `git worktree remove`).
- Captain (CL) explicitly named local-live-test verification as a gate requirement (cross-stage, same as 0x9): `make test-live-claude` must run from worktree at both implementation and validation, with results reported in stage reports.
