---
id: 4q934ttakvtv5ngt662pez34
title: "claude-team build dispatch prompts are ~7000 chars; could be ~500"
status: backlog
source: "GitHub issue #229 — CL observed dispatch prompts inline boilerplate that duplicates `spacedock:ensign` skill body"
started:
completed:
verdict:
score:
worktree:
issue: "#229"
pr:
---

## Problem

`claude-team build`'s emitted dispatch prompt inlines, on every dispatch:

- Full stage definition (the same block on every entity at the same stage)
- Read-entity instructions
- Completion signal protocol (full breakglass note)
- Worktree + branch declarations
- Stage report appending instructions

Ensigns load the `spacedock:ensign` skill on their first action anyway, so most of this content is redundant duplication of the skill body.

## Proposal

Shrink the dispatch prompt to roughly:

- Entity path
- Stage name
- Checklist items (the stage-specific work)
- Scope notes (worktree, branch)
- Completion signal target

Let the ensign skill carry the rest. Reduces ~7000 chars per dispatch to ~500.

## Why this matters

- Faster reads for the human watching the team
- Less context burned at dispatch time
- Easier to spot differences between dispatches (most of the content is currently identical boilerplate)

## Scale context

- Spacedock version: 0.11.2
- GitHub issue: https://github.com/clkao/spacedock/issues/229
