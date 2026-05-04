---
id: m4ywfn5tbtsaxebf6btxrhww
title: "status --set silently injects frontmatter into files whose first --- is a body separator"
status: ideation
source: GitHub issue #186 (clkao/spacedock)
started: 2026-05-04T16:22:12Z
completed:
verdict:
score: 0.6
worktree:
issue: "#186"
pr:
mod-block:
---

`skills/commission/bin/status` treats the first `---` it encounters anywhere in a markdown file as the opening of YAML frontmatter, rather than requiring `---` on line 1. Files with a body horizontal rule (a common markdown idiom) are misidentified as entities; `status --set` then mutates them, splicing keys into the middle of a user-authored document with exit 0 and no warning.

The entity-discovery glob compounds this: every `*.md` in the workflow directory is a candidate, so research artifacts, drafts, or other docs colocated with entities are eligible for corruption.

See issue #186 for full reproduction, root-cause analysis (`parse_frontmatter` line 96 and `update_frontmatter` line 1438 share the flaw; `parse_stages_block` line 188 has the same shape but is read-only), and suggested fixes:

1. Strict opening fence: require the first non-empty, non-BOM line to be exactly `---`.
2. Defensive write check in `update_frontmatter`: refuse to write if the file does not start with a valid fence.
3. Entity discovery should require valid frontmatter; files lacking a proper opening fence should be skipped during scan.
4. `status --set` should resolve the slug through entity discovery before opening the file, so a typo or non-entity slug fails fast rather than mutating the matching path.

Ideation should decide between fix-everything (1+2+3+4) and a minimal subset, weighing blast radius against scope.
