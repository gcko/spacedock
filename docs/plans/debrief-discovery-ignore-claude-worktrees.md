---
id: s68tqg0gqqyy8hpc2py48gq9
title: "debrief discovery should ignore .claude/worktrees workflow copies"
status: ideation
source: "GitHub issue #174 (filed by Kent Chen / iamcxa, 2026-04-30)"
started: 2026-04-30T19:47:24Z
completed:
verdict:
score: 0.55
worktree:
issue: "#174"
pr:
mod-block:
---

`skills/debrief/SKILL.md:22` runs the workflow-discovery grep with `--exclude-dir=.worktrees` (among others) but does NOT exclude `.claude/worktrees`. In repos where agent-created git worktrees live under `.claude/worktrees/...`, every worktree carries a copy of the workflow README, so debrief discovery returns the primary workflow + N duplicate copies. The captain has to disambiguate even when there's a single intended workflow in the primary checkout.

## Suggested fix

Add `.claude/worktrees` (or equivalent path filtering) to the discovery exclusion list at `skills/debrief/SKILL.md:22`. Two implementation shapes worth ideation:

- **Add another `--exclude-dir=` arg** — `grep`'s `--exclude-dir` matches directory NAME, so `--exclude-dir=worktrees` (without leading dot) would catch `.claude/worktrees/` along with any other dir literally named `worktrees`. Risk: may exclude legitimate user-level `worktrees/` dirs.
- **Post-filter the grep output** — pipe through `grep -v '/\.claude/worktrees/'` or equivalent. More surgical; preserves directory names captains might intentionally use.

Reporter (Kent Chen) flagged this for `spacedock@0.10.2`; the same code path is in current `0.11.0`. No regression test for discovery exclusion behavior currently exists.
