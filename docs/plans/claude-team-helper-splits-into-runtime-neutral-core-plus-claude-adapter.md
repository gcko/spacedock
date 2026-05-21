---
id: mtc76hh8swg9gfh1gshsteej
title: "claude-team helper splits into spacedock-dispatch-core (runtime-neutral) + claude-team (Claude adapter)"
status: backlog
source: "Promised by 0x9 (claude-team-build-fetch-on-demand-dispatch-spec, PR #231 merged 2026-05-21) `## Design` → `### Runtime-boundary disposition`. The 0x9 implementation annotated runtime-neutral functions in `skills/commission/bin/claude-team` with `# RUNTIME-NEUTRAL` markers to make this future split mechanical. The split is deferred to keep 0x9's scope bounded; this entity executes the split as a separate, mechanical refactor."
started:
completed:
verdict:
score:
worktree:
issue:
pr:
---

## Problem

`skills/commission/bin/claude-team` is named Claude-specific but contains a mix of runtime-neutral and Claude-specific code:

| Category | Examples |
|---|---|
| **Runtime-neutral** (annotated `# RUNTIME-NEUTRAL` by 0x9) | `extract_stage_subsection`, `cmd_show_stage_def`, `enumerate_declared_standing_teammates`, `_parse_routing_usage_body`, the `### Fetch commands` block format, the `## Fetch-on-Demand Bootstrap` shared-core section, stage-name-to-line-range computation |
| **Claude-specific** | `## First action` Skill-invoke directive, `### Completion Signal` SendMessage body, the standing-teammates routing-prose body, the FO-forwarding `Agent(prompt=...)` warning, `cmd_spawn_standing`, `cmd_context_budget` (reads `.claude/teams/`) |

This conflation has two costs:

1. **Codex adapter parity is harder.** When a future entity ships a Codex equivalent (`codex-team`), it has to either copy-paste-modify the runtime-neutral pieces or import them across a CLI boundary — both ugly. With a clean split, both adapters import a shared `spacedock-dispatch-core` library.

2. **Spacedock's existing idiom isn't followed.** The FO and ensign skills already use `{x}-shared-core.md` + `claude-{x}-runtime.md` + `codex-{x}-runtime.md`. The helper hasn't yet adopted this shape; this entity brings it in line.

## Why this matters

- 0x9 added `# RUNTIME-NEUTRAL` markers on five functions specifically so this future split is mechanical (a grep tells the implementer exactly what to move). The work is bounded.
- Future entities that ship Codex adapters or other runtime-specific dispatchers benefit from the clean import boundary.
- Even if no other runtime ships, the split makes the helper easier to reason about — its name no longer lies.

## Proposed approach

1. **Create new module file** `skills/commission/bin/team_helper_core.py` (name chosen to avoid baking "claude" into the runtime-neutral filename — `claude_team_core.py` would itself violate the boundary).
2. **Move every `# RUNTIME-NEUTRAL`-marked function** from `claude-team` to the new module. Verify the marker grep enumerates the move-candidates correctly.
3. **`claude-team` imports from the new module.** No duplicated logic; single source of truth.
4. **Update every caller of `claude-team`** that depends on the moved functions — runtime adapters (`claude-first-officer-runtime.md`, `claude-ensign-runtime.md`), tests under `tests/test_claude_team*.py`, any `_mods/` consumers.
5. **`cmd_show_standing` straddles the boundary** — its scaffolding (composition of runtime-neutral helpers) is runtime-neutral but its output body (SendMessage routing prose) is Claude-specific. The split should put `cmd_show_standing`'s scaffolding in the core module and parameterize the rendering on runtime. The 0x9 ideation surfaced this; the split makes the parameterization concrete.

## Out of scope

- Adding a Codex adapter (`codex-team`). That's a separate downstream entity. This entity only does the split; it does not introduce a second adapter.
- Changing the `claude-team build` CLI surface. The split is internal; the CLI behavior is unchanged. Existing tests should continue to pass.
- Renaming `claude-team` itself. The Claude-specific portions stay in a Claude-named module. Only the runtime-neutral extraction moves.

## Scale context

- Spacedock version: 0.11.2+
- Supersedes/follows: archived 0x9 (`claude-team-build-fetch-on-demand-dispatch-spec`, PR #231) which staged `# RUNTIME-NEUTRAL` markers for this split.
- Estimated complexity: moderate. ~300 lines moved, ~50 lines of import wiring, every caller of `claude-team` updated. Mechanical once the markers are followed.
- No new behavior; structural refactor only. Existing tests should continue to pass after the move; new tests for the imports may be added.
