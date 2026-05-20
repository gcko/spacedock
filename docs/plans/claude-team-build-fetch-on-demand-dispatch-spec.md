---
id: 0x93enxe1hpmk95a25476zyn
title: "claude-team build emits fetch-on-demand spec; ensign loads stage def + standing section on first action"
status: backlog
source: "supersedes 4q9 (claude-team-build-dispatch-prompt-shrink) — CL refocused scope after cycle-3 ideation surfaced ~990 char savings as too small; the real lever is dispatch prompts referencing fetch commands instead of inlining content, which saves both ensign-side prompt size AND FO-side context (claude-team output + Agent prompt args = 3x cost)"
started:
completed:
verdict:
score:
worktree:
issue: "#229"
pr:
---

## Problem

`claude-team build`'s emitted dispatch prompt inlines large content blocks the ensign already has access to via the filesystem. Measured on a real cycle-1 dispatch (entity `4q9` ideation):

- Stage definition: **2639 chars (30.8% of the prompt)** — slice of `docs/plans/README.md` that varies per stage but is fixed per workflow.
- Standing teammates section: **1699 chars (19.9%)** — assembled from `_mods/*.md` files; varies per team membership.
- First-action / boot directive: **460 chars (5.4%)** — static text per workflow.

Total inlined-but-fetchable content: **~4800 chars (~56% of the prompt)**.

These same chars are paid **three times in context-window cost** per dispatch:

1. **`claude-team build` stdout** — the helper returns the assembled prompt embedded in JSON; the FO reads ~9000 chars of helper output to dispatch one ensign.
2. **FO's `Agent()` tool-call args** — the full prompt sits in the FO's tool-call args; ~8500 chars.
3. **Ensign's initial system context** — the full prompt is the ensign's starting context; ~8500 chars.

The ensign's #3 cost is necessary (the ensign needs the stage def to do the work). Costs #1 and #2 are not — the FO never reads the inlined stage def or standing section for its own reasoning, just forwards them. Per-dispatch FO context burn: **~17,500 chars**. Of that, **~12,700 chars carries content the FO will never use**.

## Proposal

Restructure the dispatch contract so that **shared, fetchable content is referenced by command, not inlined**. The dispatch prompt becomes a small spec the ensign expands on first action.

### Concrete shape

The emitted dispatch prompt (today's ~8500 chars per fresh dispatch) collapses to roughly:

```
## First action

Run these fetch commands and load their output into your context:

  sed -n '85,93p' /Users/clkao/git/spacedock/docs/plans/README.md   # stage definition
  claude-team show-standing --team {team} --members comm-officer     # standing section

Then invoke Skill(skill="spacedock:ensign") to load the operating contract.

You are working on: {entity title}
Stage: {stage}

Read the entity file at {entity_path}.

### Completion checklist
{checklist}

### Scope notes
{scope_notes if any}

### Feedback context
{feedback_context if any}

### Completion Signal
SendMessage(to="team-lead", message="Done: {title} completed {stage}. Report written to {path}.")
```

Estimated post-restructure size: **~3300 chars** for the same fresh dispatch — about **60% reduction**.

### What changes for `claude-team build`

The helper emits a spec where shared content is replaced by literal fetch commands:

```json
{
  "subagent_type": "spacedock:ensign",
  "name": "...",
  "team_name": "...",
  "fetch_commands": [
    "sed -n '85,93p' /Users/clkao/git/spacedock/docs/plans/README.md",
    "claude-team show-standing --team {team} --members comm-officer"
  ],
  "entity_path": "...",
  "checklist": ["..."],
  "scope_notes": "...",
  "feedback_context": null,
  "completion_signal": "SendMessage(to=\"team-lead\", message=\"Done: ...\")",
  "prompt": "<the small assembled prompt above>"
}
```

The helper's stdout drops from ~9000 to ~1500 chars. The FO's `Agent()` prompt arg drops from ~8500 to ~3300. **FO context burn per dispatch drops from ~17,500 to ~4800 — saves ~12,700 chars per dispatch.**

### What changes for the ensign

A new boot discipline lives in `spacedock:ensign`'s skill body (loaded via `Skill(skill="spacedock:ensign")` after the fetch round-trip):

> First action on dispatch: read your initial prompt's `## First action` block. Run each listed fetch command via Bash, concatenate the output, and treat the result as if it had been inlined in your initial prompt. Then proceed with the checklist.

The ensign's actual context after loading still contains the stage def + standing section — fetched JIT instead of pre-loaded — so the ensign's reasoning surface is unchanged. The cost shifts from "prompt assembly time" to "first-action time" (one Bash round-trip, ~1 second).

## Why this matters

Three reasons, in order of leverage:

1. **FO context budget is the expensive one.** It compounds across an entire session of dispatching multiple entities through multiple stages each. A session with 5 entities × 4 dispatches per entity × ~13,000 chars FO-side savings = ~260,000 chars freed in the FO's context. That's meaningful headroom for long sessions.
2. **Always-fresh content.** If the workflow README is edited mid-session, the next dispatch sees the new content; no stale inlined copy.
3. **The helper's API gets simpler.** `claude-team build` no longer has to maintain README-extraction and standing-section-formatting logic in its output assembly — those become separately-callable `claude-team show-stage-def` / `claude-team show-standing` commands the ensign invokes directly.

## Scale context

- Spacedock version: 0.11.2
- GitHub issue: https://github.com/clkao/spacedock/issues/229
- Supersedes: `_archive/claude-team-build-dispatch-prompt-shrink.md` (entity 4q9, rejected at ideation 2026-05-20 with ~990 chars saved as too small a target)
- Related: `fo-breakglass-template-skill-invoke-directive` (entity 2x6) — small standalone fix for adding the Skill-invoke directive to the FO breakglass template; may be absorbed by this entity's restructure
