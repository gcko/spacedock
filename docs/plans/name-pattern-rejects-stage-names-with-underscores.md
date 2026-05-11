---
id: bwck187yrng4rgxeyar1kfzz
title: NAME_PATTERN rejects stage names with underscores
status: ideation
source: captain (CL)
started: 2026-05-11T06:39:07Z
completed:
verdict:
score:
worktree:
---

`claude-team build` derives `Agent()` names as `{worker_key}-{slug}-{stage}` and validates them against `NAME_PATTERN = ^[a-z0-9][a-z0-9-]*[a-z0-9]$` (`skills/commission/bin/claude-team:37`, used at line 317). Workflows whose `stages.states[].name` contains underscores cannot be dispatched — the derived name carries the underscore through and fails the regex with `derived name '...' contains invalid characters`.

Observed examples of stage names that trip this: `in_progress`, `live_full_approved`, `live_probation`.

The README's stage-name field is operator-authored prose. Spacedock's own README uses hyphens by convention, but nothing in commission or status enforces that convention, so workflows authored with snake_case stage names look valid at commission time and then break only at the first dispatch.

Possible resolution directions (decide in ideation):
- Allow underscores in `NAME_PATTERN` (and verify Claude Code's own agent-name validation accepts them).
- Reject underscore stage names at commission / `status --validate` time with a clear error pointing at the offending stage.
- Normalize underscores to hyphens when deriving the agent name (risks collisions across stages that differ only by `_` vs `-`).

Whichever path we pick, the failure must move from "first dispatch surprises the captain" to either "stage name is accepted everywhere" or "commission/validate refuses the stage name with an actionable message".

## Proposed approach — reject at `status --validate`

Take direction 2: surface the failure at workflow-authoring / validation time, not at first dispatch. Concretely:

1. In `skills/commission/bin/status`, extend `validate_workflow()` (around line 699) to walk each workflow's `parse_stages_block()` output and emit one error per stage whose `name` does not match the dispatch-name character class `^[a-z0-9][a-z0-9-]*[a-z0-9]$`. The error message names the offending stage and points at the convention: e.g. `workflow '<dir>': stage name 'in_progress' must be lowercase kebab-case ([a-z0-9-]); rename to 'in-progress' or similar`.
2. In `skills/commission/bin/claude-team` (line 317), refine the existing `derived name '...' contains invalid characters` error to call out the most common cause — an underscored stage name — and point operators at `status --validate`. This is the safety net for workflows that skipped validation.
3. Optionally surface the convention once in `skills/commission/SKILL.md` near the workflow template (line ~291) so future commissions are authored correctly. One-line note, not a new section.

### Why this over the other two seed directions

- **Relax `NAME_PATTERN` to allow `_`** (direction 1) — rejected.
  - Spacedock's own naming convention is hyphen-only and pervasive: `FO-R-006` mandates hyphen-separated team names (`docs/plans/fo-behavior-spec-and-coverage-matrix.md:306`), entity slugs are documented as "lowercase, hyphens, no spaces" (`docs/plans/README.md:34`, `skills/commission/SKILL.md:439`), and every existing stage name in the codebase uses hyphens. Allowing `_` everywhere splits the convention and invites a workflow with both `in-progress` and `in_progress` to look fine to the regex while being a debugging hazard.
  - Claude Code's published agent-name rules are not part of the Spacedock contract; we should stay conservative on the conservative side of whatever Claude Code accepts rather than discover the limit at dispatch time. The current regex was deliberately chosen to be a safe subset.
  - No upside vs. direction 2 except sparing the captain a rename when a workflow already uses underscores — but renames in unstarted README stage names are cheap, and once entities exist with that `status:` value the rename touches every entity's frontmatter anyway. The captain pays that cost once, then is done.
- **Normalize `_` → `-` at derive time** (direction 3) — rejected.
  - Captain's seed already flags the collision risk: a workflow with sibling stages `live_full_approved` and `live-full-approved` (or, more realistically, an author who switches conventions mid-workflow) produces two distinct stages with one derived agent name. That is silent, downstream, and corrupts the shutdown-sweep guarantees that depend on agent name uniqueness.
  - Normalization breaks the grep-from-stage-name-to-agent-name correspondence. An operator who sees `derived name 'spacedock-ensign-foo-in-progress'` in a log cannot grep the README and find a stage named `in-progress` because the README says `in_progress`. The mapping is invisible and unguessable.
  - Silent rewriting violates the broader Spacedock principle that workflow text is authoritative — what's in the README is what runs.

### Why move enforcement out of dispatch and into validate

The current failure mode is "workflow looks fine at commission and at `--validate`; first `Agent()` call rejects with a low-context message". Direction 2 inverts that: `status --validate` becomes the single source of truth for "is this workflow shaped correctly?", which matches how the rest of the validate surface already works (entity id uniqueness, slug uniqueness, sd-b32 id well-formedness — all caught at `validate_workflow` rather than at first read). Adding stage-name regex enforcement there is a one-line addition to an existing list, not a new validation pass.

### Boundaries

- The dispatch-time regex stays unchanged. It is the load-bearing safety check; we are adding a friendlier earlier failure point, not replacing the late one.
- The validation applies to the workflow README's `stages.states[].name`, not to entity slugs (slugs already obey hyphen convention via the slugify path).
- The validation is purely static: it reads the README and reports errors. No runtime behavior, no dispatch, no live Claude.

## Acceptance Criteria

1. `status --validate` reports one error per stage whose `name` field violates the dispatch-name character class, naming the workflow and stage and including a kebab-case suggestion.
   Verified by: a unit test that constructs a workflow README with a `name: in_progress` stage and asserts `validate_workflow()` returns an error string containing `stage name 'in_progress'` and the suggested kebab form. Test file: `tests/test_status_validate.py` (new test function `test_validate_rejects_underscored_stage_name`).
2. The dispatch-time check in `claude-team build` (currently `skills/commission/bin/claude-team:317`) produces an error message that explicitly mentions stage-name convention and points at `status --validate`, rather than the generic `contains invalid characters` text.
   Verified by: a unit test that constructs a workflow whose stage `name` contains an underscore, calls `build` for that stage, and asserts stderr matches a regex covering both the offending derived name AND the words `kebab-case` / `--validate`. Extends `tests/test_claude_team.py` with `test_build_underscore_stage_error_message`.
3. A clean workflow (all stage names already kebab-case) still passes `status --validate` with `VALID`.
   Verified by: a regression unit test using the existing fixture workflow under `tests/fixtures/` (the standard fixture used by `test_claude_team.py::_make_workflow_fixture` already uses hyphenated stage names — assert `status --validate` exits 0 on it).
4. The commission SKILL template documents the stage-name convention in one line near the workflow template.
   Verified by: `grep -n "kebab\|hyphen.*stage\|stage.*hyphen" skills/commission/SKILL.md` returns at least one hit in the stages section.

## Test Plan

- **Mechanism check first** (per Rule: validate smallest end-to-end exercise of the riskiest path FIRST): write the failing unit test against `validate_workflow()` for AC1 and confirm it fails with the *current* code before changing anything. This proves the test wires up to the right entry point and confirms today's behavior is "no error for underscored stage name".
- **Unit tests** for AC1, AC2, AC3 — all live in the existing pytest suite, no E2E. Each test is a single Python function (~20-30 lines) using `tmp_path` and the established `_make_workflow_fixture` pattern from `tests/test_claude_team.py`. Cost: ~5 minutes wallclock per test, milliseconds runtime.
- **Static check** for AC4 — `grep` assertion in a test or manual verification. Either is fine; the AC is durable doc structure.
- **No E2E tests needed**. The claim is "static validation of workflow README config", not "runtime dispatch behavior". A live-Claude test would be paying a 10+ minute round-trip to verify a regex check that a unit test verifies in milliseconds. Per the ideation guide: "Choose proof at the same abstraction level as the claim: static checks for durable doc/contract structure".
- **No fixture changes expected**. Existing kebab-case fixtures already exercise the happy path; the new tests construct underscored stage names inline in the test file.

## Out of scope

- Migrating existing in-the-wild workflows with snake_case stage names. The new validation will surface them; renaming is a per-workflow operator task, not part of this change.
- Changing `NAME_MAX_LEN`, `MODEL_ENUM`, or any other dispatch-name validation knob.
- Adding stage-name validation to commission's interactive prompts (a separate, larger ergonomics change).
- Touching `parse_stages_block` parsing — the raw string capture stays as-is; validation runs against the parsed result.

## Stage Report: ideation

- DONE: Acceptance criteria are entity-level end-state properties with concrete 'Verified by' citations (grep / test name / file path / command). No imperative-verb AC items.
  Four ACs, each stated as an end-state ("`status --validate` reports …", "error message … mentions …", "clean workflow … still passes", "template documents …") with named test functions, file paths, and a grep command as the verifier.
- DONE: Proposed approach picks among the three seed directions (relax NAME_PATTERN, reject underscores at commission/validate, or normalize underscores to hyphens) with explicit reasoning about tradeoffs — including the collision risk for normalization and whether Claude Code's own agent-name rules accept underscores.
  Selected direction 2 (reject at `status --validate`). Direction 1 rejected because it splits the pervasive hyphen-only convention (cited FO-R-006, slug docs) and treats Claude Code's external rules as load-bearing when they shouldn't be. Direction 3 rejected on the seed's own collision argument plus loss of grep-from-README-to-agent-name correspondence.
- DONE: Test plan picks proof at the right abstraction level for the chosen approach — unit test against NAME_PATTERN or commission validation for static checks, with E2E only if real-runtime dispatch behavior is the claim.
  Three unit tests against `validate_workflow()` and `claude-team build`, no E2E. Mechanism check (failing test before code change) included as the first step per the validate-the-riskiest-path-first rule.

### Summary

Fleshed the task body around direction 2 (reject underscored stage names at `status --validate`, with a friendlier dispatch-time error as a safety net). The collision risk and the codebase-wide hyphen convention rule out the other two seed directions. Test plan stays at unit-test level — this is static workflow-config validation, not runtime dispatch behavior, so an E2E would be over-instrumentation. Implementation surface is one added loop in `validate_workflow()` plus a more specific error string in `claude-team build`.
