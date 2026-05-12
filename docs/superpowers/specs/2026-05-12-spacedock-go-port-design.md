# Spacedock Go Port — Design

**Status:** approved foundational decisions; sub-project work tracked separately (see roadmap doc).
**Date:** 2026-05-12.
**Authors:** captain (CL) + first-officer (this session).

## Goal

Replace the current Claude-Code-plugin distribution of Spacedock with a Go binary (`spacedock`) that:

- Acts as the captain-facing entrypoint: `spacedock claude` launches what `claude --agent spacedock:first-officer` does today, `spacedock codex` is its sibling for the Codex runtime, both optionally wrapped in a `sandbox-exec` "safehouse" profile on macOS.
- Bundles helper subcommands: `spacedock status …` and `spacedock claude-team …` replace the Python scripts at `skills/commission/bin/`.
- Keeps the skill-markdown contract (FO contract, ensign contract, mods, agent definitions) intact; the binary stages markdown where the underlying runtime expects.
- Ships via `go install` for Go users and pip wheel (with platform-specific binaries) for everyone else — the `ruff` / `uv` distribution model.

## Foundational decisions

### Architecture path

- **Launcher path.** `spacedock claude` execs the existing `claude` CLI (Claude Code) with the right plugin/agent/env config; spacedock orchestrates around the runtime, it does not own the LLM loop. Same shape for `spacedock codex`. Confirmed against the alternative "SDK driver" path (spacedock owns the LLM loop via Claude Agent SDK) — that path is reserved as a future escape hatch under the `pi` vendor-agnostic harness if something forces it later (e.g., needing tool injection Claude Code can't give us).
- **Multi-runtime as sibling subcommands.** `spacedock claude` and `spacedock codex` share a `spacedock` binary. The runtime-specific markdown adapters (`claude-first-officer-runtime.md`, `codex-first-officer-runtime.md`) continue to live in `skills/first-officer/references/` and are staged by the binary at launch.
- **Skill markdown preserved.** The FO contract, ensign contract, mods, agent definitions remain markdown files. The binary either ships them as `go:embed` assets and stages them at launch, or installs them alongside as a sibling tree under the spacedock install directory. Either way, the markdown remains the captain-readable source of truth.

### Helpers

- **`status` and `claude-team` ported to Go.** Both are mature Python (~2700 LOC combined) with hard-won bug fixes. Port them so the binary is self-contained; preserve test coverage by running the existing pytest suite against the Go binary (test surface unchanged from outside).
- **Pluggable entity-state backend.** Today's markdown-frontmatter store becomes backend #1 behind an interface. The interface design and a concrete second backend (chosen during sub-project #5 — likely SQLite for stress-testing the interface) ship together; never design an interface with one consumer.

### Specs first

- **Frontmatter contract specced before the port.** The README (workflow definition) and entity-file frontmatter shapes get a formal versioned spec. Existing `commissioned-by: spacedock@0.11.1` stamps imply versioned contract; the spec makes that explicit. Spec is sub-project #1 and is a prerequisite for the helper ports — the Go implementation reads/writes against a contract, not against current Python behavior alone.

### Distribution

- **`go install` + pip wheel shipping the binary.** Pip wheel uses platform-specific binary wheels (manylinux, macos-arm64, macos-x86_64, etc.) — same model as `ruff`. `pip install spacedock` puts the binary on PATH. `go install github.com/clkao/spacedock@latest` works for Go users.
- **No bootstrap-at-runtime download in the steady state.** The binary is installed via the package manager, not fetched on first use. (Bootstrap-download is the migration-only Shape B; see migration section below.)

### Safehouse

- **`sandbox-exec` profile, macOS-only, no-op elsewhere.** Same mechanism Claude Code uses internally for its own sandboxing. Linux/Windows users get the binary without safehouse; the feature engages only when the underlying tool is available and the captain opts in.

### Migration

The migration is **incremental**, not big-bang. Today's plugin keeps working while the Go binary ships subcommand by subcommand. Two shapes of shim cover the transition:

- **Shape A — non-fatal shim (port phase).** Today's `skills/commission/bin/status` and `skills/commission/bin/claude-team` become tiny Python shims that detect `spacedock` on PATH:
  - If present AND the invoked subcommand is supported, exec into the Go binary.
  - Otherwise, fall back to the existing bundled Python implementation.
  - **Never exit on missing binary** — the Python fallback is the safety net for the entire port. Users on the current plugin keep working without installing anything.
- **Shape B — bootstrap-download (pre-cutover, before Python sunset).** The shim gains an optional bootstrap mode that downloads the binary from a pinned GitHub release URL (SHA256 + signature verification, atomic install, cache to `~/.cache/spacedock/{version}/`) if `spacedock` isn't on PATH. Engages only when Shape A's fallback path is being sunset, to catch users who haven't installed via pip but still want the binary path.
- **Subcommand routing during the port.** The shim needs to know which subcommands the Go binary supports (it won't implement everything at once). **Don't over-design this** — the hybrid period is intentionally short, so the cheapest viable mechanism wins. Default: hardcoded list of "Go-ready" subcommands in the shim, updated per shim release (shim and binary version together). Upgrade to a `spacedock --capabilities` JSON manifest only if subcommand churn during the port forces it.

### Versioning

- **Single-source version stamp.** Pip wheel version is canonical. Go module version is kept in lockstep at release time. The binary's `commissioned-by: spacedock@X.Y.Z` stamp comes from `runtime/debug.BuildInfo` populated by goreleaser; matches the pip wheel version exactly.

## Out of scope (called out)

- **The FO contract itself.** The Go port is a packaging + helper-port change. The FO contract (shared core + runtime adapters + ensign discipline) remains markdown that the binary stages. Contract changes ship as markdown PRs, unchanged.
- **Plugin auto-uninstall.** When Spacedock is sunset as a Claude Code plugin, users uninstall the plugin manually. The binary doesn't manipulate `~/.claude/plugins/` on their behalf.
- **Cross-platform safehouse.** `sandbox-exec` is Apple-only and stays that way. Linux equivalents (bwrap/firejail/namespaces) and Windows equivalents are not in scope for this design; they can be added later under the same safehouse abstraction if needed.

## Architectural risks

- **Capability-detection drift between shim and binary.** If subcommand support diverges (shim thinks Go binary supports `status --where` but actual binary doesn't), the routing breaks silently. Mitigation: lockstep shim+binary releases; the shim is bundled with the spacedock plugin/wheel that matches a known binary version.
- **Backend interface premature ossification.** Designing the pluggable backend interface with only one implementation (markdown frontmatter) is the trap. Mitigation: design + ship a second concrete backend (SQLite) in the same sub-project to stress-test the interface before it ossifies.
- **gf-spec gaps.** The automated reverse-engineering captured most behavior but likely missed edge cases the Python catches via tests. Mitigation: keep the existing pytest suite as the equivalence oracle during the port — every ported subcommand passes the same tests the Python implementation does.
- **Skill-markdown staging timing.** If the Go binary stages markdown files at launch (writing to a temp dir), interrupted launches could leave partial state. Mitigation: idempotent staging + version-based content-addressed paths.

## What this enables (downstream)

- Single-binary distribution: `pip install spacedock` puts everything on PATH; no `~/.claude/plugins/` configuration step.
- Cross-runtime parity in one binary: `spacedock claude` and `spacedock codex` are siblings, not separate installs.
- Type-safe state mutations on the entity store (Go), reducing the surface area for the kind of frontmatter parsing bugs the Python has historically had.
- A formal frontmatter contract that's versioned and auditable, instead of "whatever the current Python script does."
