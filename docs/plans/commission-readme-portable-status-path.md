---
id: 5aqx95ck26bvj6dafmsa4rns
title: "Commissioned README should not reference machine-specific paths or status usage"
status: backlog
source: "GitHub issue #172 (filed by Jared Scott / gcko, 2026-04-30)"
started:
completed:
verdict:
score: 0.7
worktree:
issue: "#172"
pr:
mod-block:
---

## Problem (as filed by reporter)

`commission` generates READMEs containing absolute per-machine status invocations:

```
/Users/<user>/.claude/plugins/cache/spacedock/spacedock/0.11.0/skills/commission/bin/status --workflow-dir <dir>
```

Three sources of per-machine drift: username, plugin version directory, cache prefix. Single-operator workflows are unaffected. Team-shared workflows break silently for every operator other than the original commissioner — `command not found` with no in-README hint.

## Captain-directed scope (2026-04-30)

A standalone `spacedock` CLI on PATH is the systemic fix (gives plugin CLIs the portability property agents already have via `{plugin}:{agent}` identifiers), but it's a bigger change. For now, the scope is to fix the commissioned README directly along three constraints:

1. **No machine-specific paths.** The commissioned README must not embed `~/.claude/plugins/cache/...` (or any other per-machine absolute path). The `{spacedock_plugin_dir}` placeholder must not be resolved into the generated README.

2. **No status-usage prose.** The commissioned README must not document `status` invocation at all. Status usage is encapsulated in the first-officer skill — that's where the runtime knows how to find and use it. Captains who want to inspect workflow state run the FO; the FO knows how. Captains who want raw `status` access read the FO skill prose; the README doesn't need to teach them.

3. **Refer to the first-officer skill.** The commissioned README's runtime-entrypoint section becomes: "to operate this workflow, run `claude --agent spacedock:first-officer`." That's it. The first-officer agent identifier is portable because the plugin loader resolves it the same way on every machine.

4. **Refit checks the constraints.** When `spacedock:refit` runs against an existing workflow, it verifies the README does not contain machine-specific paths and does not document status usage. If it finds either, it flags the drift to the captain and offers to regenerate the relevant README sections to the new shape.

## Concrete edits in `skills/commission/SKILL.md`

The interpolation sites identified in the original intake (lines 401, 409, 415, 662) all need to be removed or replaced with FO-skill references. Same for setup-time prose at lines 503 and 634 (those are setup instructions, not generated README content — keep those, since setup is captain-machine-local by definition).

The "Workflow State" section in the generated README (currently shows `status --next` example invocations) becomes either dropped entirely or replaced with one line: "Workflow state is read by the first officer at boot. Run `claude --agent spacedock:first-officer` to view current state."

## Acceptance hints (for ideation)

- AC: generated README contains zero `{spacedock_plugin_dir}` placeholders or absolute cache paths (grep guard).
- AC: generated README contains zero `bin/status` references (grep guard).
- AC: generated README runtime-entrypoint section is the canonical FO-invocation line.
- AC: `spacedock:refit` detects either machine-specific paths or status-usage prose in an existing README and surfaces the drift to the captain.
- AC: regression — commissioning a fresh workflow produces a README that passes the above grep guards.

## Out of scope

- Standalone `spacedock` CLI wrapper on PATH (the systemic fix; captain has it in mind for a separate larger task — flagged here for cross-reference).
- Migration helper for existing already-commissioned READMEs (refit's check + offer-to-regenerate covers the upgrade path).
- Other absolute paths in commissioned files (mod source paths in setup prose, etc.) unless they fall into the same anti-pattern at commission time.

## Cross-references

- **#221** (commission templates + Trait Detection, just shipped) — the proximate cause: trait detection now confidently produces multi-operator workflows for team-flavored missions, exposing the per-machine path as a plurality rather than an edge case.
- **GH #172** original framing offered three fix shapes (CLI wrapper, portable resolution snippet, doc note). The captain chose a fourth: encapsulate status in the FO and remove status from the README.
- Deferred follow-up: standalone `spacedock` CLI (captain-future).
