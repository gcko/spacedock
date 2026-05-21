---
id: 2ydg3csfyx22kt1qhgcphjm8
title: "Raise supported-model floor for bare-mode FO CI from haiku to sonnet"
status: backlog
source: "Captain pivot at the cycle-5 gate of `nwwqsx5q` (haiku-bare-fo-startup-protocol-adherence, REJECTED, archived). Cycle-4 spike (14 runs, real API, full data) showed: baseline production prose passes haiku-bare 5/5 locally (#200 Pattern A doesn't reproduce off-CI), candidate prose tightening regresses haiku-bare pass rate from 5/5 to 3/5 (introduces over-approve + early-terminate failure modes), cross-check shows opus-bare 2/2 PASS and haiku-teams 2/2 PASS (candidate doesn't regress working modes). Honest conclusion: prose tightening alone can't lift haiku's protocol-adherence floor; sonnet sits between haiku and opus on capability and is the cheapest model empirically plausible to clear the bar. Captain (CL) chose option D — raise the model floor — over option B (retire bare-haiku coverage) on 2026-05-21."
started:
completed:
verdict:
score:
worktree:
issue:
pr:
---

## Problem

Bare-mode FO CI runs `tests/test_gate_guardrail.py` against `--model haiku`. The haiku-bare combination is below the protocol-adherence bar (per archived `nwwqsx5q` cycle-4 empirical evidence and `_archive/haiku-bare-fo-guardrail-weaknesses.md` #200 history); the test flakes intermittently on CI even though it passes locally with production prose. Each CI failure costs captain attention (artifact inspection, regression-vs-flake call) and blocks PRs unintentionally.

The honest read of the cycle-4 empirical evidence: haiku-low's bare-mode FO is below the bar. Prose hardening can't lift it without introducing new failure modes. The cheapest fix that preserves test coverage and clears the bar is **switching the bare-mode CI model from haiku to sonnet**, and declaring "Spacedock targets sonnet-or-above for bare-mode FO" as the supported-model contract.

## Why this matters

- **Eliminates a recurring CI flake source** that has produced false-alarm investigations on at least PRs #210 and #231 in this session alone.
- **Sets a clear capability contract.** Right now Spacedock implicitly supports any model the user passes, then fails inconsistently when the model is below the protocol bar. Declaring sonnet-or-above gives users a clear floor and gives Spacedock a clear support boundary.
- **Smaller than retiring bare-haiku coverage outright.** Coverage is preserved (sonnet exercises the same test surface); the matrix just shifts up one model tier.

## Proposed approach

Three concrete deliverables in one entity:

### 1. Empirical spike (gates the design)

Run `tests/test_gate_guardrail.py --runtime claude --model sonnet --team-mode bare --effort low` **5 times** locally. PASS criterion: ≥4/5 pass with no protocol-adherence flakes (no `cd $HOME` freelance; first Bash call contains `status`). Cost estimate ~$10-15 wall-clock (sonnet 3-4x haiku per call). If PASS, proceed to steps 2-3. If FAIL, surface findings and either escalate to opus floor or reconsider option B.

The spike harness from `nwwqsx5q` at `/tmp/nwq-spike/run-spike.sh` is reusable — change `MODEL=haiku` → `MODEL=sonnet` in the `baseline` invocation. Artifacts: `/tmp/2yd-spike/baseline/run{1..5}/` mirroring the prior naming convention.

### 2. CI matrix update

If spike PASSES: edit `.github/workflows/runtime-live-e2e.yml` `claude-live-bare` job. Change the model parameter from haiku to sonnet for the bare-mode invocation. Verify the `make test-live-claude-bare` Makefile target also defaults to (or accepts) sonnet; update if hardcoded to haiku.

The change is minimal — likely one or two YAML lines in the workflow file plus a Makefile variable update.

### 3. Supported-model floor documentation

Add a "Supported models" section to a top-level reference. Candidates (in preference order):
- `README.md` top-level — most visible to users
- `references/code-project-guardrails.md` — closer to where existing capability discipline lives
- The commissioning skill's reference material — discoverable at commission time

Content: declare "Spacedock targets sonnet-or-above for the first-officer role in bare mode. Haiku is below the protocol-adherence bar (cwd-drift, gate-self-approve, early-terminate failure modes observed empirically; see archived entity `nwwqsx5q`). Teams-mode FO and ensigns may still run on haiku for cost; bare-mode FO requires sonnet minimum."

This is the explicit capability contract the entity establishes.

## Acceptance criteria

End-state properties of the finished entity. Each verifiable by a future reader.

1. **The spike report in the entity body documents 5 sonnet bare-mode runs of `test_gate_guardrail`** with the three observables (first-Bash-is-status, test pass/fail, FO cwd) per run, plus a named verdict (PASS / INCONCLUSIVE / FAIL). PASS criterion: ≥4/5 pass with no protocol-adherence flakes.
   - **Test:** entity body's `### Spike report` subsection contains the table; reviewer cross-checks against `/tmp/2yd-spike/baseline/run*/spacedock-test-*/fo-log.jsonl` artifacts.

2. **`.github/workflows/runtime-live-e2e.yml` `claude-live-bare` job runs `--model sonnet` (not `--model haiku`)** for the bare-mode FO invocation. (Only fires if AC-1 verdict is PASS.)
   - **Test:** static lint — grep the workflow file for the bare-mode job's model parameter. If `haiku` appears in that job's model arg, fail.

3. **`make test-live-claude-bare` target accepts `--model sonnet` and defaults to it.** Either the Makefile variable defaults to sonnet for the bare suite, OR the bare suite's pytest invocation passes `--model sonnet` explicitly. The target should NOT default to haiku.
   - **Test:** static lint — `grep -E "MODEL.*=.*haiku|model haiku" Makefile` returns no matches in the bare-mode target.

4. **A supported-models section exists at a top-level reference** (README.md or equivalent) declaring sonnet-or-above for bare-mode FO. The section names haiku as below-bar and points at the archived `nwwqsx5q` for empirical evidence.
   - **Test:** static-content test asserts the supported-models section exists with the key substrings ("sonnet-or-above", "bare-mode", reference to archived empirical evidence).

5. **`make test-static` passes with the changes applied** — no regressions from the CI workflow + Makefile + documentation updates.
   - **Test:** `make test-static` exit code 0.

6. **`make test-live-claude-bare` runs the gate-guardrail test on sonnet locally before PR open** — empirical confirmation the CI matrix change works end-to-end. Pass result documented in the Stage Report.
   - **Test:** `make test-live-claude-bare TEST=test_gate_guardrail` exit code 0 with the bare-mode model verified as sonnet in fo-log.jsonl.

## Test plan

- **Spike (cost-controlled):** 5 runs of `test_gate_guardrail` with `--model sonnet --team-mode bare`, ~$10-15. Records observables, computes verdict. Artifacts under `/tmp/2yd-spike/`.
- **`tests/test_agent_content.py`** (extend): static lint for the supported-models section in whichever top-level reference receives it. ~5 lines added.
- **`tests/test_ci_workflow.py`** (new, OR extension of `test_agent_content.py`): static lint asserting `runtime-live-e2e.yml`'s bare-mode model is sonnet. ~10 lines.
- **`make test-static`**: confirm offline suite green after changes.
- **`make test-live-claude-bare`**: implementation gate; run the gate-guardrail test on sonnet locally before PR open. (This is the captain's cross-cycle live-test requirement.)

**No new modules, no schema changes.** Net change is ~5 lines of YAML + ~5 lines of Makefile + ~30 lines of documentation + ~15 lines of tests.

## Out of scope

- **Cascade verification on other bare-mode live tests** (`test_feedback_keepalive`, `test_merge_hook_guardrail`, etc.). Captain explicitly deferred this to CI: "once we PR we'll know the rest." If a non-gate-guardrail bare-mode test regresses on sonnet, address as a separate small entity.
- **Teams-mode model floor.** Teams-mode FO works on haiku per the cycle-4 cross-check (haiku-teams 2/2 PASS). The supported-models declaration is scoped to bare-mode FO only.
- **Ensign model floor.** Ensigns run downstream of the FO and have different capability requirements; not scoped here.
- **Retroactive xfail of test_gate_guardrail's haiku entry.** Not needed — the entity drops haiku from the bare-mode matrix entirely, so the entry no longer exists.

## Scale context

- Spacedock version: 0.11.2+
- Supersedes/pivots from: `_archive/haiku-bare-fo-startup-protocol-adherence.md` (`nwwqsx5q`, REJECTED, cycle-4 FAIL verdict + captain pivot to option D).
- Related: `_archive/haiku-bare-fo-guardrail-weaknesses.md` (#200, original Pattern A documentation), GitHub issue #219 (Bash wedge — different surface).
- Cost: spike ~$10-15, implementation review ~$1, validation ~$5. Total ~$20.
- Wall time: spike ~25 min, implementation ~30 min, validation ~25 min.
