---
id: 220
title: "Use Pi plugin subagent tools for dispatch when available"
status: implementation
source: "captain request 2026-04-27"
started: 2026-04-27T00:00:00Z
completed:
verdict:
score:
worktree: .worktrees/spacedock-ensign-pi-plugin-subagent-dispatch
issue:
pr:
mod-block:
---

## Problem Statement

Spacedock's Pi first-officer dispatch path has been built around repo-local Pi worker runtime helpers. Pi now also exposes subagent tooling from a Pi plugin, including background runs (`/run -bg` / subagent async execution). Spacedock should use that plugin-native dispatch surface when it is available, because it better matches Pi's runtime model and avoids maintaining more custom process orchestration than necessary.

The existing helper path is still valuable as a fallback for environments without the Pi subagent extension installed, and users need a clear hint when the plugin-native path is unavailable.

## Scope Boundary

In scope:

- Detect and use Pi plugin subagent tooling for Pi worker dispatch/reuse where the runtime can do so safely.
- Preserve the existing repo-local Pi worker runtime helper fallback.
- Add an operator-facing hint that tells users to install/enable the Pi subagents extension when the plugin-native path is unavailable.
- Keep the behavior compatible with the current Pi runtime baseline work in `.worktrees/spacedock-ensign-pi-runtime-compatibility-baseline`.
- Use `uv` for all Python/test commands.

Out of scope:

- Removing the fallback helper path.
- Requiring the Pi subagents extension for all users.
- Broad runtime parity beyond adapting dispatch logic.

## Acceptance criteria

**AC-1 — Pi dispatch prefers plugin-native subagent tooling when available.**
Verified by: focused tests or fixtures showing the Pi first-officer dispatch logic selects the Pi subagent tool path when the tool/extension is present.

**AC-2 — Existing repo-local Pi worker helper fallback still works when plugin-native subagent tooling is absent.**
Verified by: focused tests covering the unavailable-extension path and by preserving existing Pi runtime helper tests.

**AC-3 — Users receive a clear install/enable hint when the Pi subagent extension is unavailable.**
Verified by: tests or static assertions for the fallback diagnostic/hint text.

**AC-4 — Progressive Pi runtime tests remain meaningful.**
Verified by: relevant `uv run pytest ...` commands for the Pi dispatch selection and progressive watcher coverage.

## Implementation Notes

Start from the existing Pi runtime compatibility baseline branch/worktree and keep the change small. Treat plugin-native subagent dispatch as an adapter choice, not as a rewrite of the whole first-officer runtime. If a runtime call cannot use the plugin-native path, fall back to the current helper and surface a concise hint such as: install/enable the `pi-subagents` extension to use plugin-native `/run -bg` dispatch.
