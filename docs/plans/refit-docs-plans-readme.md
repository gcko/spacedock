---
id: 06z0dycs40qr0a9b35waxaxf
title: "Refit docs/plans/README.md to match the post-#176 commission template"
status: ideation
source: "Captain direction 2026-04-30 immediately after PR #176/#177 landed: 'refit our docs/plans readme with what the new commission skill would create'"
started: 2026-04-30T23:35:00Z
completed:
verdict:
score: 0.55
worktree:
issue:
pr:
mod-block:
---

## Problem

PR #176 (just merged) updated the commission template so that newly-commissioned workflow READMEs:

1. Contain no machine-specific paths (no `{spacedock_plugin_dir}/.../bin/status` examples)
2. Contain no `bin/status` invocation prose
3. Have a `## Workflow State` section that delegates to `claude --agent spacedock:first-officer`

We dogfood spacedock with `docs/plans/`. Our own `docs/plans/README.md` still carries the pre-#176 shape — it has the three `{spacedock_plugin_dir}/.../bin/status` example blocks and the `grep -l "status:"` snippet that the new template removes. The README also still claims `commissioned-by: spacedock@0.11.0`, which is fine for the version field but doesn't tell us anything about post-#176 template alignment.

This is a one-shot dogfood update: make our own README look like what fresh commission would emit today, while preserving all our captain-customized content (sd-b32 id-style, the specific stage definitions for our 5-stage pipeline, the `## Testing Resources` section, the `## Commit Discipline` section, the `## File Naming` and `## Schema` sections).

## Captain direction (2026-04-30)

> refit our docs/plans readme with what the new commission skill would create

Treat as ideation-gate spec; advance straight to implementation.

## Proposed approach

Run `spacedock:refit` against `docs/plans/`. Refit's Phase 3b is the prose-and-diff Show-Diff pattern: it generates what current commission would produce for this workflow and presents the diff against the existing README; the captain (or implementation worker) accepts the changes that align with intent and rejects accidental customizations.

The expected diff is narrow:
- Delete the three example status-invocation snippets in the existing `## Workflow State` section
- Delete the `grep -l "status: {stage_name}" {dir}/*.md` snippet
- Replace with the canonical `claude --agent spacedock:first-officer` paragraph

All captain-customized sections (`## File Naming`, `## Schema`, the per-stage definitions in `## Stages`, `## Testing Resources`, `## Commit Discipline`, ID-style frontmatter, etc.) are out of scope and must be preserved.

## Acceptance criteria

**AC-1 — `docs/plans/README.md` contains no machine-specific path interpolations.**
Verified by: `grep -nE '\{spacedock_plugin_dir\}|\.claude/plugins/cache' docs/plans/README.md` returns zero matches.

**AC-2 — `docs/plans/README.md` contains no `bin/status` invocation prose.**
Verified by: `grep -n 'bin/status' docs/plans/README.md` returns zero matches.

**AC-3 — `docs/plans/README.md` `## Workflow State` section delegates to `claude --agent spacedock:first-officer`.**
Verified by: the `## Workflow State` section's body mentions `claude --agent spacedock:first-officer` and contains no other invocation examples.

**AC-4 — All captain-customized sections survive the refit unchanged.**
Verified by: explicit diff inspection — only the `## Workflow State` section changes. The frontmatter (`id-style: sd-b32`, `stages.defaults`, `stages.states`), `## File Naming`, `## Schema`, the five per-stage definitions, `## Workflow State` *heading* (kept; only its body changes), `## Task Template`, `## Testing Resources`, and `## Commit Discipline` sections all remain byte-identical.

**AC-5 — `commissioned-by:` field is bumped to the current `plugin.json` version.**
Verified by: `grep '^commissioned-by:' docs/plans/README.md` matches the version in `plugin.json` at the time of merge.

## Test plan

1. Implementation worker runs `spacedock:refit` against `docs/plans/` to generate the diff.
2. Worker applies only the changes described in the proposed-approach section above (Workflow State body + commissioned-by bump). All other diff hunks are rejected.
3. After the edit, run the AC-1..AC-5 grep / diff checks above as evidence in the stage report.
4. `make test-static` to confirm no test regressions (none expected — README is documentation).

## Out of scope

- Other workflow READMEs in `tests/fixtures/*/README.md` (#176 already audited those; they're clean).
- Captain-customized prose updates outside the `## Workflow State` section (those are intentional drift from the template; not refit's job to revert).
- Any commission-template changes — that landed in #176 and is now upstream.

## Cross-references

- Filed alongside `yv8kbe048ad4y1mb0dtqe5dj` (pr-merge-audit-link-short-sd-b32) which the captain ordered in the same turn.
- Builds on PR #176 (5a — commission template update).
