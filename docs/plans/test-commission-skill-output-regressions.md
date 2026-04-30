---
id: 197
title: "test_commission: commission skill produces leaked templates, absolute paths, unwanted _mods/pr-merge.md"
status: ideation
source: "PR #131 CI (#154 cycle-1 pre-merge) — after #154 lifted the content-drift xfail and swapped test_commission's static content reads to `assembled_agent_content`, 60/63 inner checks pass; 3/63 remain FAIL on all three claude variants (claude-live, claude-live-bare, claude-live-opus)"
started: 2026-04-30T22:10:38Z
completed:
verdict:
score: 0.7
worktree:
issue:
pr:
mod-block:
---

## Problem

`test_commission` is decorated `@pytest.mark.xfail(strict=False, reason="pending #197 …")`. A live re-run against the current 5a worktree (`spacedock-ensign-commission-readme-portable-status-path` HEAD `8a19c7e6`) shows **2 inner failures out of 68 checks** — a different shape than the PR #131-era body (3/63). The two current failures are:

### Failure 1: `[File Existence] workflow-local pr-merge mod is not generated`
Test code: `t.check("workflow-local pr-merge mod is not generated", not (workflow_dir / "_mods" / "pr-merge.md").exists())`
The test asserts the file MUST NOT exist. Commission produces it: the generated `_mods/pr-merge.md` is byte-identical to the plugin-shipped `mods/pr-merge.md`. Source: `skills/commission/SKILL.md:482-483` instructs the LLM to `cp "{spacedock_plugin_dir}/mods/pr-merge.md" {dir}/_mods/pr-merge.md`.

### Failure 2: `[No Leaked Template Variables] no leaked template variables`
Test code (lines 315-330): scans every `*.md` under `workflow_dir` for `\{[a-z_]+\}` excluding lines containing `${` or `slug`. Five leaks reported; **four of the five live inside `_mods/pr-merge.md`** (a verbatim copy of the plugin file, which legitimately contains doc/template variables `{number}`, `{branch}`, `{constructed body}`):

- `_mods/pr-merge.md`: `gh pr view {number} --json state --jq '.state'`
- `_mods/pr-merge.md`: `**Branch:** {branch} -> main`
- `_mods/pr-merge.md`: `git push origin {branch}`
- `_mods/pr-merge.md`: `gh pr create --base main --head {branch} --title "{entity title}" --body "{constructed body}"`

The fifth lives in `refit-command.md` (a generated entity body, not a template):
- `refit-command.md`: `Verified by: the README frontmatter \`commissioned-by\` field reads \`spacedock@{current_version}\` after refit completes.` — the LLM wrote `{current_version}` as English prose describing a refit AC. It is not a leaked template slot; it is prose that *mentions* a template variable.

### Failure 3 (historical, currently passing): `no absolute paths in generated files`
The `[No Absolute Paths]` section now passes (5a cycle 2 added the `[README Portability]` section and 5a's portability fix landed in commit `2906988e commission: replace per-machine status snippets with FO-invocation prose`). This failure class is already resolved upstream and out of scope for #197.

## Architectural conflict (must be resolved before fix)

The current pr-merge runtime mechanism only discovers hooks from `{workflow_dir}/_mods/*.md` (`skills/commission/bin/status:863-877`, `skills/first-officer/references/codex-first-officer-runtime.md:187`, `skills/first-officer/references/claude-first-officer-runtime.md:259`). The plugin-shipped `mods/pr-merge.md` is NOT auto-discovered. So the `cp` command is currently the *install mechanism*, not redundant duplication. Simply deleting it would break PR-merge behavior end-to-end (no startup PR-state checks, no merge-hook gate, no mod-block enforcement).

The dispatch framing implies the captain wants to stop the duplicate-copy. Two real paths to that end-state, picked here so the captain can ratify the architectural direction at the gate:

### Option A — Plugin-mods auto-discovery (preferred)
Extend `scan_mods` (`skills/commission/bin/status:863-877`) to ALSO scan `{spacedock_plugin_dir}/mods/*.md` for hook headings, with workflow-local `_mods/*.md` taking precedence on filename collision. Update FO runtime references to document the fallback. Drop the `cp` from `skills/commission/SKILL.md:482-483` and the checklist line at `skills/commission/SKILL.md:495`. Workflow-local `_mods/` becomes opt-in for per-workflow overrides.

Pros: solves both failures in one fix path (no `_mods/pr-merge.md` → no leaks from it); plugin upgrades to pr-merge.md propagate automatically to every workflow without per-workflow refit; aligns with the existing "plugin-shipped status viewer" model that 5a's cycle just enshrined for status.
Cons: changes mechanism, not just commission prose; need to update mod-block enforcement (`status --set` / `status --archive`'s "registered merge hooks" check at `skills/first-officer/references/first-officer-shared-core.md:250`) to consult the plugin scan as well.

### Option B — Keep the local copy; refine the test
Accept that `_mods/pr-merge.md` is the install mechanism. Refine `test_commission`'s leak scan to either (a) skip `_mods/*.md` files entirely (mods are documented templates), or (b) compare `_mods/pr-merge.md` byte-for-byte against `{spacedock_plugin_dir}/mods/pr-merge.md` and fail only on drift. Drop the "workflow-local pr-merge mod is not generated" check — replace with "workflow-local pr-merge mod matches plugin source byte-for-byte".

Pros: smallest mechanism delta; no FO runtime changes.
Cons: leaves the duplicate-copy install model in place; per-workflow plugin upgrades still require manual `cp` refresh; doesn't address the captain's apparent intent to stop redundant generation.

### Recommendation
Option A. The captain's framing in the dispatch ("the file should NOT exist locally — the plugin already ships it") plus the existing "plugin-shipped status viewer" precedent both point at plugin-resident shared assets being first-class. Option B preserves a workflow-pollution pattern the test was built to flag.

The fifth leak (`refit-command.md` `{current_version}` prose) is independent of A/B. Two narrow sub-options:
- **A1**: tighten the test regex to also exclude lines starting with `Verified by:` (the leak-pattern is in plain English describing refit semantics, not a template slot).
- **A2**: tweak commission prose so the seeded refit-command.md AC uses backticks-around-words instead of `{var}` syntax (e.g., "reads the current spacedock version").

Recommendation: A2 (fix at content-source). The test regex was correct to flag braces in generated entity bodies; the LLM should not write `{var}` syntax in entity prose. Adjust the seed-entity scaffolding prose in `skills/commission/SKILL.md` so the refit-command seed example doesn't model bracey prose.

## Proposed approach (assuming Option A + A2)

1. **Extend `scan_mods` in `skills/commission/bin/status`** to scan the plugin `mods/` directory in addition to `{workflow_dir}/_mods/`. Workflow-local files of the same name shadow plugin files. Return mod entries tagged with their source so callers can render them in `--boot` output. Resolve the plugin path the same way the script already resolves itself (`os.path.dirname(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))` walks `bin/ → commission/ → skills/ → plugin_root`).
2. **Update FO runtime references** (`first-officer-shared-core.md`, `claude-first-officer-runtime.md`, `codex-first-officer-runtime.md`) to note that `MODS` in `--boot` output may resolve to plugin-shipped or workflow-local mods, and that workflow-local entries shadow plugin entries by filename.
3. **Update mod-block enforcement** at `status --set` / `status --archive`. The "registered merge hooks" check (currently scans `_mods/*.md` for `## Hook: merge`) must also see plugin-shipped merge hooks — otherwise terminal transitions stop being guarded for workflows that rely on plugin pr-merge.
4. **Drop `cp` from commission**. Remove `skills/commission/SKILL.md:481-484` and the checklist line at `:495`. Update `:520`'s announcement bullet to drop the `_mods/pr-merge.md` mention. Keep the y/n install confirmation prose (still controls the "do you want pr-merge for this workflow" decision; the answer flips a workflow-config flag rather than copying a file — see #5).
5. **Add an opt-out mechanism**. Plugin-mods auto-discovery means EVERY workflow sees the plugin pr-merge by default. Workflows that opt out (captain says "no pr-merge") need a way to suppress it. Simplest: a `_mods/disabled.txt` (or a frontmatter list in README) listing mod names to skip. Commission writes the opt-out file when the captain declines pr-merge install. `scan_mods` honors the suppression list.
6. **Adjust commission prose for refit-command seed entity** (`skills/commission/SKILL.md` — the seed-entity blocks for the refinement template) so the refit-command AC doesn't write `{current_version}` / `{template variable}` syntax in plain English. Replace bracey wording with backticked or natural-language descriptions.

## Acceptance criteria

End-state properties; each AC has a `Verified by:` clause naming the existing extended `test_commission` (no new test infrastructure needed — 5a cycle 2 already extended it to 68 checks).

- **AC1**: `[File Existence] workflow-local pr-merge mod is not generated` PASSES — commission does not produce `{workflow_dir}/_mods/pr-merge.md` for a fresh `make test-live-claude` run.
  Verified by: `tests/test_commission.py:79` runs against the live commission output and `t.check` records PASS.
- **AC2**: `[No Leaked Template Variables] no leaked template variables` PASSES — no `*.md` file under the generated `workflow_dir` contains `{lower_word}` patterns (excluding `${...}` and lines with `slug`).
  Verified by: `tests/test_commission.py:315-330` scan returns an empty `leaked` list.
- **AC3**: PR-merge runtime behavior is preserved end-to-end — startup hook still scans entities for `pr` field and reports merged PRs; merge hook still gates terminal transitions; `mod-block` enforcement still refuses terminal updates when a registered merge hook hasn't run.
  Verified by: `tests/fixtures/merge-hook-pipeline` and `tests/fixtures/push-main-pipeline` continue to pass under existing test runners (`pytest tests/test_status.py -k "merge_hook or pr_merge"`).
- **AC4**: Captain opt-out path exists — when commission's pr-merge install confirmation is declined, the resulting workflow does not run plugin pr-merge hooks.
  Verified by: a single targeted unit test on `scan_mods` that asserts the suppression mechanism (e.g., `_mods/disabled.txt` listing `pr-merge`) returns no pr-merge hooks even when the plugin file is present. Add this test as part of implementation.
- **AC5**: The `refit-command.md` seed-entity prose generated by commission contains no `\{[a-z_]+\}` patterns in entity-body text.
  Verified by: AC2's scan covers this — `refit-command.md` is in `workflow_dir` and is included in the `rglob("*.md")` walk.
- **AC6**: The `xfail(strict=False, reason="pending #197 …")` decorator on `test_commission` is left in place by this task. Removal is a follow-up after 2-3 CI cycles confirm xpass on all three claude variants. The current `strict=False` semantic tolerates xpass without breaking CI.
  Verified by: `tests/test_commission.py` diff shows the decorator unchanged after this task's PR merges; a follow-up issue is filed against #197 to remove it later.

## Test plan

The existing `test_commission` (extended to 68 checks via 5a cycle 2 commit `12f1fa87`) is the authoritative proof. After this task:

1. **Pre-fix baseline (already captured)**: `66 passed, 2 failed (out of 68 checks)` against 5a worktree HEAD `8a19c7e6`.
2. **Post-fix expected**: `68 passed, 0 failed` → pytest reports `XPASS` (tolerated by `strict=False`).
3. **Run command**: `unset CLAUDECODE && uv run pytest tests/test_commission.py -v -s` from the worktree root. Wallclock ~100s for the live commission phase.
4. **Regression coverage**: the existing `tests/test_status.py` (or whichever covers `scan_mods` and `--boot`) must continue passing. Extend it with the `disabled.txt` opt-out test.
5. **No new test files needed** — both AC1 and AC2 are existing inner checks; AC3 leverages existing fixtures; AC4 needs one additional `scan_mods` test in the existing status-test file.

## Out of scope for this task

- **Decorator removal**. The xfail decorator stays. Filing a follow-up issue is acceptable; removal happens after CI confirms a few clean xpass cycles.
- **Other mod types** (silence-watcher, comm-officer). Plugin-mods auto-discovery should be designed extensibly, but only pr-merge is migrated to plugin-resident-only in this task. Other mods follow as a separate refit pass.
- **5a's portability section**. 5a (`commission-readme-portable-status-path`) edits the same `SKILL.md` but in different sections (the README-status-prose generation paths). No coordination needed; merge ordering is captain's call (5a is already PR #176; #197 will rebase on top once 5a merges).
- **Refit propagation**. Existing workflows with `_mods/pr-merge.md` already in their tree continue to work (workflow-local shadows plugin-shipped). A `refit` pass to delete the local copies is a follow-up, not blocking.

## Bundling

Per the dispatch, this task bundles into 5a's PR #176 once ideation gate passes. Implementation will use 5a's existing worktree on branch `spacedock-ensign/commission-readme-portable-status-path`. No new worktree for ideation.

## Stage Report: ideation

- DONE: Pin the actual current failure shape: re-run `unset CLAUDECODE && uv run pytest tests/test_commission.py -v -s` against the current 5a worktree and enumerate every failing inner check.
  Live re-run against 5a worktree HEAD `8a19c7e6` returned `66 passed, 2 failed (out of 68 checks)`. Two failures named verbatim: `[File Existence] workflow-local pr-merge mod is not generated` and `[No Leaked Template Variables] no leaked template variables`. Stale 3/63 framing reconciled — historical Failure 3 (absolute paths) now passes upstream via 5a commit `2906988e`.
- DONE: Concrete-fix plan: for each currently-failing check, name the file and line where commission is leaking the template variable / generating the wrong path / failing to produce the workflow-local pr-merge mod.
  Failure 1 root-caused to `skills/commission/SKILL.md:482-483` (the `cp "{spacedock_plugin_dir}/mods/pr-merge.md"` instruction). Failure 2 root-caused to two sources: 4 of 5 leaks live inside the byte-identical `_mods/pr-merge.md` copy (resolved by fixing Failure 1); the 5th leak (`refit-command.md` `{current_version}`) is LLM-generated entity prose seeded by SKILL.md's refit-command scaffolding. Architectural conflict named: `scan_mods` only sees `_mods/*.md`, so naive deletion breaks pr-merge runtime — captured as Option A vs Option B with recommendation A.
- DONE: AC items as end-state properties with concrete `Verified by:` clauses. Verification is the existing `test_commission`.
  Six ACs filed (AC1–AC6). AC1, AC2, AC5 verified by existing inner checks in `tests/test_commission.py:79` and `:315-330`. AC3 verified by existing pr-merge fixture tests. AC4 requires one targeted unit test on `scan_mods` opt-out. AC6 captures the decorator-removal-as-follow-up requirement called out in the dispatch — `strict=False` tolerates xpass so leaving it in place is safe.

### Summary

Pinned the live 2/68 failure shape, root-caused both failures to `skills/commission/SKILL.md:482-483` plus LLM seed-entity prose, and surfaced the architectural conflict that simply deleting the `cp` would break pr-merge runtime. Recommended Option A (extend `scan_mods` to discover plugin-shipped mods, with workflow-local shadowing and a captain opt-out path) so the fix lands as a one-shot mechanism upgrade rather than mechanism breakage. Captain ratifies direction at the gate; implementation will share 5a's worktree per dispatch.

