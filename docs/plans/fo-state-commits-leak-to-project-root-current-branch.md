---
id: 60mv0j3qa2hk947rkkm7s18w
title: "first-officer state commits leak to project-root current branch instead of main (multi-session contamination)"
status: backlog
source: GitHub issue #187 (clkao/spacedock)
started:
completed:
verdict:
score: 0.75
worktree:
issue: "#187"
pr:
mod-block:
---

The FO writes state-transition commits (`dispatch:`, `advance:`, `mod-block:`, `pr:`, `auto-finalize:`) to the project root via `git -C {project_root} commit ...` without verifying HEAD is on `main`. When the captain has another concurrent session checked out to an unrelated branch (or moves the project root to a feature branch between FO ticks), every FO state commit silently lands on the captured branch. Worker stage commits are unaffected — only FO state commits leak.

Real-world contamination at carlove (PR #657, two waves of state commits silently committed to `chore/slim-claude-md` and `fix/supabase-push-dockerless-gen-types`); captain noticed and reset both times before pushing.

Issue #187 contains the full reproduction, root-cause analysis (shared-core L66-69 vs L186-189 contradiction), and a worked-out preferred fix (B4: dedicated `.worktrees/_fo-state` worktree pinned to `origin/main`; project root becomes captain-owned and untouchable by FO). Smaller alternatives (B3 `--require-branch=main`, B2 commit helper, B5 frontmatter `branch:` field) considered and ranked.

Ideation should:
- Resolve the L66-69 vs L186-189 contradiction in shared-core.
- Decide between B4 (dedicated FO worktree) and a smaller alternative; the issue recommends B4+B1 with B2/B3/B5 deferred.
- Specify which writes (if any) still need to mirror to `main` (initial worktree creation, final `pr:` mirror, merge-time advance) versus which live on the FO worktree branch.
- Test plan must include the multi-session repro from the issue (a captain-checked-out unrelated branch must not be polluted by FO state commits).
