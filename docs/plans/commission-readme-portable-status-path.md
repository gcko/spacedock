---
id: 5aqx95ck26bvj6dafmsa4rns
title: "Status-viewer paths in commission-generated READMEs are per-machine, breaking team-shared workflows"
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

The `commission` skill template at `skills/commission/SKILL.md` lines 401, 409, 415, 662 generates README invocations using the `{spacedock_plugin_dir}` placeholder, which gets resolved at commission time into the captain's absolute per-machine path:

```
/Users/<user>/.claude/plugins/cache/spacedock/spacedock/0.11.0/skills/commission/bin/status --workflow-dir <dir>
```

Three sources of per-machine drift in that path:

- **Username** — different on every operator's machine
- **Plugin version directory** (`0.11.0`) — drifts on plugin upgrades, even on the same machine
- **Cache prefix** — `~/.claude/plugins/cache/...` is the Claude Code default; differs across runtimes (Codex, Gemini)

Single-operator workflows are unaffected (the path works for the commissioner). Team-shared workflows break silently for every operator other than the original commissioner — they hit `command not found` with no in-README hint that the path needs per-machine resolution.

## Suggested fix shapes (Jared listed three; for ideation)

**A — CLI wrapper on PATH.** Provide `spacedock status …` (or similar) that operators install once via the plugin's setup. README uses the portable form. Highest effort; cleanest end-state.

**B — Portable resolution pattern in template.** Generate a small inline shell snippet that resolves the plugin directory at runtime (e.g., `claude plugin path spacedock`, an env var, or a launcher script the plugin can drop into `~/.local/bin/spacedock-status`). Medium effort; preserves the "no extra install" property.

**C — Documentation note in generated README.** Add a per-machine-path call-out explaining the path is captain-specific and how teammates resolve it. Cheapest; preserves silent-break for teammates who don't read prose.

## Adjacent context

- Reporter is the third external contributor to file an issue against this repo (after Kent Chen and CL). Multi-operator use is now an exposed surface.
- The `commission-suggest-common-workflows` design (#221, just shipped) introduced the `development` template explicitly recommending sd-b32 IDs for collaborative workflows — the framing already assumes multi-operator workflows are first-class. This task closes the per-machine-path gap that surfaced once captains actually tried that.
- Current behavior is captured at SKILL.md:401, 409, 415, 503, 634, 662 — six interpolation sites total.
