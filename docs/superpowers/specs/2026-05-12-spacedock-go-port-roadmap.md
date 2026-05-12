# Spacedock Go Port — Roadmap

**Status:** sub-projects defined; sub-project #1 begins brainstorming next.
**Date:** 2026-05-12.
**Companion doc:** `2026-05-12-spacedock-go-port-design.md` (foundational decisions).

The Go port is a program of work, not a single project. This doc decomposes it into sub-projects, each shippable through the normal spec → plan → implementation cycle.

## Sub-projects

| # | Sub-project | Depends on | Output | Notes |
|---|---|---|---|---|
| 1 | **Frontmatter contract spec** | — | Doc artifact: versioned spec of README + entity frontmatter | No code. Pure documentation, prerequisite for ports. |
| 2 | **`spacedock` binary skeleton + launcher** | — | Tiny Go module; `spacedock claude` and `spacedock codex` exec the runtimes | ~500 LOC. Validates the multi-runtime sibling-subcommand pattern. |
| 3 | **`status` port to Go** | 1 (schema) | Replacement for `skills/commission/bin/status` (Go subcommand) | Equivalence oracle: existing pytest suite. |
| 4 | **`claude-team` port to Go** | 3 (shared frontmatter parsing) | Replacement for `skills/commission/bin/claude-team` (Go subcommand) | Shares the frontmatter parser with #3. |
| 5 | **Pluggable entity-state backend** | 3 | Refactored store layer + concrete backend #2 (SQLite candidate) | Don't design the interface with one implementation. |
| 6 | **Safehouse integration** | 2 | Subprocess wrapper in launcher (`sandbox-exec` on macOS, no-op elsewhere) | Optional captain-protection feature. |
| 7 | **Distribution pipeline** | 2, 3, 4 | Goreleaser config + pip wheel build (manylinux + macos-arm64 + macos-x86_64) | The `ruff` model. |
| 8 | **Plugin coexistence + migration** | 7 | Shim implementation (Shape A non-fatal; Shape B pre-cutover); sunset plan for Python plugin | Includes hardcoded "Go-ready" subcommand list per shim release. |

## Dependency graph (informal)

```
        ┌──────────────┐
        │ #1 frontmatter│
        │   contract    │
        └───────┬───────┘
                │
        ┌───────┴───────┐
        ▼               │
  ┌──────────┐          │
  │ #3 status│          │
  │   port   │          │
  └────┬─────┘          │
       │                │
       ├──────────┐     │
       ▼          ▼     │
┌──────────┐ ┌─────────┐│  ┌──────────────┐
│ #4 claude│ │ #5 plug-││  │ #2 launcher  │
│  -team   │ │  gable  ││  │   skeleton   │
│   port   │ │ backend ││  │              │
└────┬─────┘ └─────────┘│  └──────┬───────┘
     │                  │         │
     │                  │         ▼
     │                  │   ┌──────────────┐
     │                  │   │ #6 safehouse │
     │                  │   │ integration  │
     │                  │   └──────────────┘
     │                  │         │
     └──────────────────┴─────────┤
                                  ▼
                          ┌──────────────┐
                          │ #7 distribu- │
                          │ tion pipeline│
                          └──────┬───────┘
                                 │
                                 ▼
                          ┌──────────────┐
                          │ #8 plugin co-│
                          │ existence +  │
                          │  migration   │
                          └──────────────┘
```

Critical path: 1 → 3 → 4 → 7 → 8. Parallel-able: 2 + 6 can ship alongside 1+3 since they don't depend on the helper ports.

## Recommended sequencing

**Phase 1 (specs + skeleton, no user-visible binary yet):**
- Sub-project #1: frontmatter contract spec. Pure documentation, prerequisite for ports.
- Sub-project #2: launcher skeleton (parallel). Tiny Go binary that execs `claude` and `codex`. Captain can install + test end-to-end without any helper ports.

**Phase 2 (helpers ported, binary becomes useful):**
- Sub-project #3: `status` port. Equivalence-tested against existing pytest suite.
- Sub-project #4: `claude-team` port. Shares the frontmatter parser.

**Phase 3 (backend pluggability + sandboxing):**
- Sub-project #5: pluggable backend interface + SQLite implementation (concurrent with helper ports if engineering bandwidth allows).
- Sub-project #6: safehouse `sandbox-exec` wrap.

**Phase 4 (distribution + migration):**
- Sub-project #7: pip wheel + goreleaser.
- Sub-project #8: Shape A shim deployment; Python plugin sunset plan + Shape B bootstrap.

## What gets brainstormed next

Sub-project #1 — frontmatter contract spec. Starts immediately after this doc commits.

## Open coordination points (between sub-projects)

- **Frontmatter parser code-sharing.** Sub-projects #3 and #4 both need to read/write entity files. Decide once in #3 whether parsing is its own internal package, an exported library, or duplicated.
- **Capability manifest format.** If subcommand routing in #8 outgrows the hardcoded-list approach, the format (JSON shape, where the binary publishes it) gets specced in #7 alongside the distribution work.
- **Version stamp threading.** Sub-projects #2, #3, #4, #7 all need to know the binary version (for `commissioned-by` stamps, `--capabilities` output, etc.). Use `runtime/debug.BuildInfo` + a single `internal/version` package; agree on the interface in #2 and reuse downstream.

## Out of scope for this roadmap

- Adding new functionality to `spacedock` beyond what the current plugin does. The port is a packaging change, not a feature change. New features land after the port stabilizes.
- Telegram / web-UI / non-CLI surfaces. The binary is CLI-first.
- Multi-user / hosted runs. spacedock stays single-captain.
