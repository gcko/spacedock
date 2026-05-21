## Finding: Spawn-phase context expansion is real and underused by the current FO

**Date:** 2026-05-21
**Context:** Surfaced while investigating per-dispatch FO context cost during the `0x9` (claude-team-build fetch-on-demand) and `2y` (sonnet model-floor) work. Question from captain: *"is there somehow expansion in the spawn phase (not constructed in the parent's context)?"*

### Status

Research-only — no spike executed, no production change. The empirical verification (a tiny `Agent()` call with a minimal prompt that reads its real assignment from a file on first action) is deferred to a future entity if the captain wants to act on these findings.

### Question being researched

When the FO calls `Agent(subagent_type=..., prompt=...)` to dispatch an ensign, the `prompt` string enters the FO's context **twice** per dispatch under the current contract:

1. As the helper's stdout JSON the FO reads after `claude-team build` (~7000 chars)
2. As the `prompt=` argument to `Agent()` (~7000 chars)

Combined per-dispatch FO context cost: ~14,000 chars even after PR #231's fetch-on-demand shrunk the *ensign-side* prompt. PR #231 fixed the ensign side (`### Fetch commands` block, no inlined stage def). The FO side is unaddressed.

The question: is there a mechanism by which the per-dispatch content can be expanded *at spawn time on the ensign side*, with the FO emitting only a tiny pointer/identifier — so the per-dispatch content enters the FO context **zero or one** times instead of two?

### What the references say

Three mechanisms exist in Claude Code for spawn-phase expansion, with different load-bearing properties.

**Mechanism 1: agent definition `skills:` frontmatter.** Documented in `_archive/agent-boot-skill-preload.md` (#085). When `Agent(subagent_type=X)` spawns an agent of type X, Claude Code loads the skills listed in X's frontmatter as user messages prepended to the agent's conversation **before** the `prompt=` arg is delivered. This is the canonical spawn-phase mechanism.

```yaml
# agents/ensign.md
---
name: ensign
description: Executes workflow stage work
skills: ["spacedock:ensign"]
---
```

When an agent of type `ensign` spawns, `spacedock:ensign`'s SKILL.md content is in its context before the `prompt=` arg arrives.

**Status:** broken for plugin agents per upstream `claude-code#25834`. The body-fallback `Skill(skill="spacedock:ensign")` instruction in the dispatch prompt is the current workaround (see archived #085 for the fallback mechanism). Both mechanisms exist; the frontmatter preload is the intended path.

**Mechanism 2: `@`-include syntax in skill bodies.** Spacedock's `skills/ensign/SKILL.md` uses:

```markdown
## Operating contract

@references/ensign-shared-core.md
@../first-officer/references/code-project-guardrails.md
```

These `@` references are resolved by Claude Code at skill-load time. The referenced file contents are inlined into the model's context as part of the skill body. This means: a skill can carry the *equivalent* of multiple files of context, and the FO doesn't have to construct any of that content per-dispatch — the ensign's skill body picks it up automatically when the skill loads.

**Status:** working and in production use today. Verified in `skills/ensign/SKILL.md:8-9` and `skills/first-officer/SKILL.md:18-19`. The `@`-include is processed by Claude Code, not by a tool call from the agent.

**Mechanism 3: `Skill(skill=X, args=Y)` runtime invocation.** The `Skill` tool takes an optional `args` string parameter (verified in the tool's JSON Schema). The skill body can in principle interpolate `args` into its loaded content. The args field is small (a path, an identifier) but the loaded skill body can be large.

**Status:** schema-confirmed; spacedock skills today do NOT use `args` interpolation. Unverified whether the args parameter actually substitutes into skill body content the way one might hope, or whether it's only available to the model reading the skill (model needs to interpret the args, not the platform substituting them). Empirical spike required to settle this.

### Current FO dispatch shape (per PR #231, fetch-on-demand)

```python
Agent(
    subagent_type="spacedock:ensign",
    name="spacedock-ensign-{slug}-{stage}",
    team_name="...",
    prompt=output.prompt  # ~7000 chars including stage def reference, checklist, scope notes, completion signal
)
```

The `output.prompt` contains:
- `## First action: Skill(spacedock:ensign)` directive (~600 chars — body-fallback for the broken Mechanism 1)
- `### Stage definition` reference via `### Fetch commands` block (~300 chars in the block; the actual content is fetched by the ensign's first Bash call)
- `### Completion checklist` (~800-1500 chars — per-dispatch content the FO authors)
- `### Scope notes` (~300-1000 chars — per-dispatch context)
- `### Standing teammates` block (~1500 chars — full per-team body)
- `### Completion Signal` block with literal SendMessage line (~900 chars)

Of these, the **checklist + scope notes** are the only segments that *fundamentally must be authored by the FO per-dispatch*. Everything else is either static (boilerplate) or fetchable (stage def via `claude-team show-stage-def`, standing teammates via `claude-team show-standing`).

The post-PR-#231 contract has the ensign Read stage def + standing on first action via `### Fetch commands`. But the prompt string itself still carries everything inline — the FO emits 7000 chars to Agent() even though most of it is fetch-pointers, not content.

### Proposed minimum-context dispatch shape (untested)

Apply 0x9's fetch-on-demand pattern at one layer up: the dispatch prompt ITSELF is a fetch-pointer.

```python
# Helper writes per-dispatch content to a file:
# /tmp/spacedock-dispatch/{name}.md  ←  full prompt content (today's ~7000 chars)

Agent(
    subagent_type="spacedock:ensign",
    name="spacedock-ensign-{slug}-{stage}",
    team_name="...",
    prompt="DISPATCH_FILE: /tmp/spacedock-dispatch/{name}.md"  # ~80 chars
)
```

Plus a new clause in `skills/ensign/references/ensign-shared-core.md` `## First action` section:

> If your initial prompt is `DISPATCH_FILE: {path}`, your first action is `Read({path})` and treat the file's content as if it had been your inline prompt. Then proceed with the operating contract.

**Predicted savings:**

| Cost surface | Today | Proposed |
|---|---|---|
| Helper stdout (FO reads) | ~7000 chars | ~300 chars (small JSON with metadata + DISPATCH_FILE path) |
| FO `Agent(prompt=...)` arg | ~7000 chars | ~80 chars |
| **FO per-dispatch context cost** | **~14,000 chars** | **~380 chars** |
| Ensign initial context | ~7000 chars (in prompt) | ~7000 chars (read from file on first action) |

Net: **~97% FO context reduction per dispatch**, ensign reasoning surface unchanged.

This composes with PR #231's fetch-on-demand savings, not duplicates them. PR #231 shrunk what enters the ensign's reasoning surface. This proposal shrinks what enters the FO's reasoning surface. Different beneficiaries.

### What the spike would need to validate

This research stops short of empirical verification. A minimal spike would:

1. Build a tiny test fixture (single entity, single stage) where the ensign's checklist is `Read /tmp/test-dispatch.md and report its first three lines.`
2. Write the per-dispatch content to `/tmp/test-dispatch.md` (whatever shape the spike chooses)
3. Spawn `Agent(subagent_type="spacedock:ensign", prompt="DISPATCH_FILE: /tmp/test-dispatch.md")`
4. Verify the ensign's first Bash/Read call resolves the file and reports its contents

Cost estimate: ~$1-3, ~5 min wall-clock. Single dispatch is sufficient evidence the mechanism works; reliability spike (5+ runs) would be optional.

If the spike PASSES, a small follow-up entity would:
- Update `claude-team build` to write per-dispatch content to `/tmp/spacedock-dispatch/{name}.md` and emit only a tiny spec referencing the file
- Update `ensign-shared-core.md` `## First action` with the DISPATCH_FILE clause
- Add the FO runtime adapter discipline: "If a fetch-file path appears in helper output, write the per-dispatch content there before dispatch; Agent() prompt becomes the tiny pointer"
- Static-content tests for the new contract

If the spike FAILS (e.g., the ensign needs more context in the prompt to know what `DISPATCH_FILE` means before the skill loads), the fallback is to bake the `DISPATCH_FILE:` instruction into the agent definition (`agents/ensign.md`) so it's part of the spawn-phase context rather than requiring the model to interpret it from the prompt alone.

### Empirical addendum (2026-05-21): `@`-include does NOT resolve in `Agent()` prompt args

Captain asked: does the `@`-include syntax that works in skill bodies (Mechanism 2 above) also work when used in the `Agent()` tool's `prompt=` arg? If yes, the platform itself would handle the inlining and the proposed `DISPATCH_FILE:` convention below would be unnecessary — the FO could just emit `prompt="@/path/to/dispatch.md"` and Claude Code would substitute the file contents at spawn time.

**Result: it does not work.** Empirical probe (single throwaway `Agent(subagent_type="general-purpose", prompt="@/tmp/at-include-probe/test-include.md...")` spawn): the spawned agent received the **literal string** `@/tmp/at-include-probe/test-include.md` with no expansion. The fixture file containing `MAGIC_INCLUDE_TOKEN_2026052100` was not visible to the agent's context.

This means:

- Claude Code's `@`-include feature is scoped to skill bodies (verified in production via Mechanism 2 above) and likely CLAUDE.md / interactive prompts (not tested here), but **NOT** `Agent()` tool prompt args.
- The proposed `DISPATCH_FILE:` convention below remains necessary — there is no platform-level shortcut that lets the FO emit a tiny `@`-pointer and have the platform inline the content. The ensign must Read the file itself on first action.
- Mechanism 1 (skills frontmatter preload) and Mechanism 2 (`@`-include in skill body) both ARE expansion-at-load. They're spawn-time platform features, but only apply to skill content, not arbitrary `Agent()` prompts.

The proposal below is unchanged; this addendum just records the empirical answer to the captain's `@`-shortcut question.

### What this research does NOT settle

- Whether Mechanism 3 (`Skill(skill=X, args=Y)`) can substitute `args` into the skill body at expansion time, or whether `args` is just a string the model reads after expansion. Read of the tool schema is consistent with both interpretations.
- Whether Claude Code's plugin-agent path supports `--system-prompt-file` style mechanics for sub-agents. CLI-level flags exist (`--system-prompt-file`, `--append-system-prompt-file`) but their applicability to `Agent()`-spawned subagents is undocumented.
- Whether the FO's `Agent()` tool result echoes the prompt back (it does NOT today, per inspection of past tool-result messages in the FO log — `"Spawned successfully"` is the only echo). If a future Claude Code change adds prompt echoing in Agent results, the FO-side savings would degrade.
- Cross-runtime: whether Codex has an equivalent spawn-phase expansion. Spacedock's runtime adapter pattern (`claude-ensign-runtime.md` + `codex-ensign-runtime.md`) suggests the per-runtime adapter could handle this differently; not investigated.

### Recommendation

File a small follow-up entity if the captain wants to pursue the ~97% FO context savings:

- **Slug suggestion:** `dispatch-prompt-as-file-pointer`
- **Scope:** spike (above) + helper change + ensign-shared-core clause + tests
- **Cost:** small. ~3-5 lines of helper code change, ~10 lines of ensign-shared-core prose, ~30 lines of tests, ~$3 spike.
- **Risk:** low. The mechanism reuses 0x9's fetch-on-demand pattern at a different layer. If the spike fails, the fallback shape is well-understood.

This is the natural next move after PR #231; it composes cleanly with what's already shipped. Whether to file it now or batch with `mtc76hh8` (runtime-boundary split) or `y15kpj10` (FO retry-path bypass) is a sequencing call.
