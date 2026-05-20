---
id: 0x93enxe1hpmk95a25476zyn
title: "claude-team build emits fetch-on-demand spec; ensign loads stage def + standing section on first action"
status: ideation
source: "supersedes 4q9 (claude-team-build-dispatch-prompt-shrink) — CL refocused scope after cycle-3 ideation surfaced ~990 char savings as too small; the real lever is dispatch prompts referencing fetch commands instead of inlining content, which saves both ensign-side prompt size AND FO-side context (claude-team output + Agent prompt args = 3x cost)"
started: 2026-05-20T22:27:58Z
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

## Spike report

Spike artifacts live in `/tmp/0x9-spike/`: `claude-team-fetchspec` (PoC helper, 165 lines), `dispatch-4q9-ideation.json` (real input matching the cycle-1 4q9 dispatch), `dispatch-4q9-helper-stdout.json` + `dispatch-4q9-prompt.txt` (baseline, captured by piping the input through the production `claude-team build`), and `dispatch-4q9-fetchspec-stdout.json` + `dispatch-4q9-fetchspec-prompt.txt` (fetch-on-demand output, same input). Spike validation simulated the ensign's first action by running each emitted fetch command via Bash and confirming the concatenated output reaches the ensign's reasoning surface with full fidelity.

**Measurements (real, not estimated; same input both sides):**

| Surface | Baseline (today) | Fetchspec (proposed) | Delta |
|---|---:|---:|---:|
| `claude-team build` stdout (FO reads on dispatch) | 8402 chars | 3445 chars | **−4957 (−59.0%)** |
| Emitted prompt (Agent prompt arg AND ensign initial context) | 7867 chars | 2789 chars | **−5078 (−64.5%)** |
| FO-side context burn per dispatch (stdout + Agent arg, the 2x non-necessary cost) | 16,269 chars | 6234 chars | **−10,035 chars** |
| Ensign first-action latency cost added | 0 (inlined) | 2 Bash round-trips, ~0.1s wallclock | +0.1s |

**Verdict: PASS, with one required mechanism addition.** The fetch-on-demand contract works end-to-end:

- *Stage definition* fetches via `sed -n 'M,Np' README.md` returning 2623 chars — byte-equivalent (modulo a trailing newline) to the 2639-char block the production helper builds via `extract_stage_subsection`. Full stage semantics (Inputs / Outputs / Good / Bad / Staff review) survive the round-trip.
- *Standing-teammates section* surfaced the spike's unknown-unknown: the existing `claude-team list-standing` subcommand emits **mod-file paths only** (60 chars on this team), not the rendered markdown block (1699 chars including descriptions and per-teammate Routing Usage bodies) that `cmd_build` assembles inline. The current `enumerate_declared_standing_teammates` + `_parse_routing_usage_body` rendering logic is reachable only through `cmd_build`. **Production implementation must add a new `claude-team show-standing --workflow-dir DIR --team TEAM` subcommand** that extracts that rendering logic and emits the same markdown the helper currently builds, otherwise the ensign loses ~1640 chars of teammate-routing content per dispatch. The spike treats this as a small refactor (move three existing helpers behind a new subcommand), not new logic.
- *Boot directive* (the `## First action` Skill-invoke block, 460 chars today) remains literal in the emitted prompt — it must not be fetched, because it is the very instruction that tells the ensign to run the other fetch commands. This is the design's load-bearing exception: one block stays inlined to bootstrap the rest.

**What the spike disproved.** The original entity body's Proposal sketched a `sed -n '85,93p' README.md` line range for ideation; the actual range is 68-83 (parser confirms). Hard-coding line numbers in fetch commands is acceptable because `cmd_build` already runs `extract_stage_subsection` at dispatch-assembly time, so it can compute the live range per dispatch and embed it. The production design uses computed ranges, not hand-coded.

**What the spike didn't validate.** I did not dispatch a live Agent-tool ensign to execute the fetch commands; my role inside this team-mode dispatch does not have direct `Agent()` access. The empirical test I ran is a deterministic Bash simulation of what the dispatched ensign would do (read prompt, parse `### Fetch commands`, run each command, confirm output is sufficient to do the work). This is the strongest empirical method the ensign-execution domain affords from inside an ensign role. If implementation surfaces a behavioral surprise in the live-Agent boot (e.g., haiku-class models skipping the fetch-block parsing the same way they currently skip the Skill-invoke directive), the existing skip-skill-load failure mode (see below) and the boot-directive's literal preservation are the two designed-in mitigations.

## Design

### Helper-side: `claude-team build` emits a fetch-on-demand spec

`cmd_build` in `skills/commission/bin/claude-team` (lines 334-456) currently appends 9 conditional and 2 unconditional prompt parts. The restructure changes which parts are **inlined verbatim** vs. **referenced by a literal shell command the ensign runs**:

| Source comment # | Section | Today | Post-restructure |
|---|---|---|---|
| 0 | `## First action` Skill-invoke directive (`claude-team:338-350`) | inlined (460 chars) | **inlined verbatim** — load-bearing exception: this is the instruction that tells the ensign to run the other fetch commands; cannot itself be fetched |
| 1 | `You are working on:` / `Stage:` header (`claude-team:353`) | inlined (142 chars) | **inlined verbatim** — per-dispatch identifiers; small |
| 2 | `### Stage definition:` + verbatim README subsection (`claude-team:356`) | inlined (2639 chars) | **replaced by fetch command** — `sed -n '{M},{N}p' {readme}` where `M,N` are computed by the existing `extract_stage_subsection` parser at build time |
| 3 | Worktree path + branch declarations (`claude-team:359-366`, conditional) | inlined (≤300 chars) | **inlined verbatim** — per-dispatch and small |
| 4 | `Read the entity file at {path}.` (`claude-team:369-379`) | inlined (≤300 chars) | **inlined verbatim** — per-dispatch and small |
| 6 | `### Feedback from prior review` block (`claude-team:382-383`, conditional) | inlined | **inlined verbatim** — per-dispatch content; fetching would require a new on-disk anchor for the feedback text, not worth the round-trip |
| 7 | Scope notes (`claude-team:386-387`, conditional) | inlined | **inlined verbatim** — per-dispatch content; same rationale as row 6 |
| 8 | `### Completion checklist` + Summary slot + Stage Report block (`claude-team:391-403`) | inlined (≤1500 chars) | **inlined verbatim for the checklist + Summary slot** (per-dispatch); the Stage Report appending-instructions sub-block is dropped because it is already in `ensign-shared-core.md:48-77` which the Skill-invoke load brings in. This is the one drop borrowed from the rejected 4q9 cycle-2 (row 8c) — confirmed structural-not-destructive there, applied here without re-litigation. |
| 9 | `### Standing teammates available in your team` (`claude-team:406-437`, conditional) | inlined (1699 chars on this team) | **replaced by fetch command** — `claude-team show-standing --workflow-dir {dir} --team {team}` (NEW subcommand; see below) |
| 10 | `### Completion Signal` block (`claude-team:440-456`, conditional) | inlined (623 chars + 364-char FO-forwarding paragraph) | **inlined verbatim for the literal `SendMessage(...)` line**; the surrounding "Plain text only. No JSON. Until you send this message…" prose (10b) is dropped because `claude-ensign-runtime.md:25` already says it; the FO-forwarding paragraph (10c) is **kept**, because its audience is the FO at Agent-forwarding time and the runtime adapter loads in the ensign, not the FO. (Same 10b-drop / 10c-keep disposition cycle-2 of 4q9 settled.) |

**New helper subcommand to add: `claude-team show-standing`.** Refactor extracts the existing rendering logic (`enumerate_declared_standing_teammates` at `claude-team:638-680`, `_parse_routing_usage_body` at `claude-team:707-743`, and the assembly loop at `claude-team:406-437`) behind a stable CLI surface:

```
claude-team show-standing --workflow-dir DIR --team TEAM
```

Stdout: the exact markdown block `cmd_build` currently emits as `prompt_parts[8]` (header `### Standing teammates available in your team` through the closing `Full routing contract:` line). Stderr empty on success. Exit 0 when standing teammates exist, exit 0 with empty stdout when none do (degenerate case). The CLI surface mirrors `list-standing`'s `--workflow-dir` convention; `--team` is added because the FO mode-check (`enumerate_declared_standing_teammates` short-circuits on empty team_name) preserves bare-mode degeneration. `cmd_build` then **calls `show-standing`'s rendering function directly** (single source of truth — no duplicated logic; no shelling out to itself).

**New helper subcommand to add: `claude-team show-stage-def`.** Refactor extracts `extract_stage_subsection` (`claude-team:100-148`) behind:

```
claude-team show-stage-def --workflow-dir DIR --stage STAGE
```

Stdout: the same string `cmd_build` currently inlines (the rendered README subsection). Exit non-zero with the same parse-error message `extract_stage_subsection` raises today. `cmd_build` continues to call `extract_stage_subsection` directly (single source of truth); the new subcommand wraps it for ensign-side use.

**Why not `sed`?** The spike used `sed -n 'M,Np' README.md` and it worked, but two production concerns make `show-stage-def` worth the small refactor: (a) the line range moves whenever the README is edited, so embedding line numbers in the emitted prompt couples the prompt to the README's exact layout — `show-stage-def --stage ideation` is name-keyed and survives README edits; (b) `sed` does not surface the helpful "stage heading at line X mentions 'foo' but does not parse as a stage heading" error `extract_stage_subsection` raises today, which is load-bearing for captain-facing diagnostics on malformed README headings. The fetch-emitted command is `claude-team show-stage-def --stage {stage}` (no line numbers in the prompt), and the FO-side parse happens once at build time as today.

**New spec JSON shape.** `cmd_build`'s output adds one field:

```json
{
  "schema_version": 1,
  "subagent_type": "spacedock:ensign",
  "name": "...",
  "team_name": "...",
  "model": null,
  "description": "...",
  "fetch_commands": [
    "claude-team show-stage-def --workflow-dir /repo/docs/plans --stage ideation",
    "claude-team show-standing --workflow-dir /repo/docs/plans --team my-team"
  ],
  "prompt": "<the small assembled prompt>"
}
```

The `fetch_commands` field is **informational/diagnostic** for the FO and downstream tools (golden-file tests, log scraping) — the literal commands are also embedded verbatim under `### Fetch commands` in `prompt`. The FO does not need to use the field at dispatch time; it forwards `prompt` to `Agent()` as today. The ensign reads only `prompt`. Schema version stays at 1 (additive change to output).

### Ensign-side: boot discipline in the `spacedock:ensign` skill body

`spacedock:ensign` (`skills/ensign/SKILL.md`) currently loads `ensign-shared-core.md` + the runtime adapter. The shared-core gets a new top-level section:

```markdown
## Fetch-on-Demand Bootstrap

The first officer's dispatch may contain a `### Fetch commands` section near the
top of your prompt. If present:

1. Read each command listed under that heading. They appear one per line,
   four-space-indented (markdown code-block convention).
2. Run each command via Bash in the order listed.
3. Concatenate the stdouts. Treat the concatenated result as if it had been
   inlined into your prompt at the position of the `### Fetch commands` block.
4. Then proceed with the rest of your assignment (entity read, checklist).

If a fetch command exits non-zero, report the failure to the first officer via
SendMessage with the command, exit code, and stderr — do not silently proceed.
A missing or unreadable stage definition is a dispatch-shape failure that the
first officer must surface to the captain; the ensign is not the right place to
paper over it.

If the dispatch prompt has no `### Fetch commands` block (legacy or breakglass
shape), skip this step. The rest of the prompt is self-contained.
```

The ensign's reasoning surface after fetch concatenation is **byte-equivalent** to today's inlined-prompt surface: stage def + standing block + checklist + scope notes + completion signal, all visible. The cost shifts from "FO assembles, ensign reads inlined" to "FO assembles smaller, ensign assembles the equivalent on first action."

### How FO-side context-saving math actually plays out

Real measurements from the spike, recomputed for a session-scale estimate using CL's framing (5 entities × 4 dispatches/entity = 20 dispatches):

| Cost surface | Per-dispatch baseline | Per-dispatch fetchspec | Per-dispatch delta | Session-scale (20 dispatches) |
|---|---:|---:|---:|---:|
| FO reads `claude-team build` stdout | 8402 | 3445 | **−4957** | **−99,140 chars** |
| FO embeds `Agent(prompt=…)` arg | 7867 | 2789 | **−5078** | **−101,560 chars** |
| Ensign initial context | 7867 | 2789 | −5078 | −101,560 chars (ensign-side, not FO budget) |
| **FO-side total (the load-bearing budget)** | **16,269** | **6234** | **−10,035** | **−200,700 chars** |

The ensign's −5078 is incidental and not the load-bearing benefit; the ensign re-acquires that content immediately via Bash, so its reasoning surface is unchanged. The load-bearing benefit is the FO's −10,035 per dispatch, which is ~5–6% of Claude Code's 200K standard context budget per dispatch — meaningful when the FO is dispatching many entities through many stages without compaction.

### Breakglass FO template

The FO break-glass template at `skills/first-officer/references/claude-first-officer-runtime.md:114-123` is the fallback when `claude-team build` exits non-zero. It already has no standing-teammates block, no FO-forwarding warning, and no plain-text-only prose; it inlines stage def + checklist + completion signal in a single string.

**Decision: the breakglass template is NOT restructured to use fetch commands.** Rationale: the breakglass path fires precisely when the helper is unavailable, so requiring the ensign to call `claude-team show-stage-def` from inside breakglass would re-introduce a dependency on the same helper that just failed. The breakglass stays a fully-inlined string — the design's worst-case path is "FO copy-pastes the stage def + checklist into Agent prompt verbatim," which works without any helper at all.

The breakglass template **does** get the one consistency edit entity 2x6 was filed for: **add `Skill(skill="spacedock:ensign")` as the first prompt-body element**. This closes the documented gap where breakglass-dispatched ensigns lack the boot directive. Add this in the same PR as the main restructure; recommend closing 2x6 as REJECTED-superseded with a source-note pointing here. (See "Relationship to entity 2x6" below.)

### Skip-skill-load failure mode

The fetch-on-demand model adds a second skip-risk surface beyond the existing skip-skill-load: the ensign could skip running the fetch commands. The model must survive both skips without destructive failure.

**Positive signal (path A: Skill loads as designed).** The `## First action` block remains the first non-blank content of the dispatch prompt and instructs `Skill(skill="spacedock:ensign")`. The skill body then loads `ensign-shared-core.md` whose new "Fetch-on-Demand Bootstrap" section instructs the ensign to run the fetch commands. AC-5 below pins the structural position of the boot directive (first heading is `## First action`), inheriting the cycle-2 4q9 pattern with the regression test reused.

**Failsafe (path B: Skill skip-loads but fetch commands still run).** The prompt body's `## First action` block is self-describing: it names the Skill call AND describes the fetch commands as content the ensign must run. A haiku-class model that skips the Skill call but reads the prompt's `### Fetch commands` section verbatim still runs the commands — the prompt is its own minimal instruction manual. The directive paragraph at lines 1-9 of the fetchspec prompt explicitly says "Run the fetch commands listed under `### Fetch commands` below … THEN invoke your operating contract" so the ordering survives even if the Skill is skipped.

**Failsafe (path C: fetch commands skip, Skill may or may not load).** What's lost vs. baseline if fetch commands skip:

- *Stage definition*: lost. The ensign has only the checklist + scope notes + completion signal. It can do the work if the checklist is self-explanatory (most are), but the stage-good/stage-bad framing is unavailable. This is **functional** (worse stage adherence), **detectable** (FO can grep the stage report for absent Inputs/Outputs/Good/Bad references on next-stage review), and **recoverable** (re-dispatch with feedback). No destructive failure mode.
- *Standing-teammates block*: lost. The ensign cannot route to the comm-officer. This is **functional** (prose un-polished), **detectable** (FO sees prose quality), and **recoverable** (FO can re-dispatch with `### Feedback from prior review` requesting a polish pass). No destructive failure mode.

**Comparison to baseline's skip-skill-load argument.** Cycle-2 of the rejected 4q9 argued its three drops were structural-not-destructive on skip-skill-load. The fetchspec model has **strictly fewer** baseline drops (only the Stage Report appending-instructions block, row 8c — the smallest of 4q9's three). What it adds is path-C skip-fetch risk. The path-C failure modes above are the same shape as cycle-2's argument: structural, detectable, recoverable, never destructive. The boot directive's literal preservation in the prompt (it does NOT itself become a fetch command) is the structural guard that keeps path-C survivable: the ensign always sees the `## First action` block explaining the fetch contract, even if the Skill-invoke is paraphrased away.

**No new mechanism needed beyond the boot-directive preservation.** The fetchspec model relies on the same two-path boot (frontmatter preload + body fallback `Skill(skill="spacedock:ensign")` directive) that today's prompt uses; it just adds a fetch step the Skill body documents.

### Relationship to entity 2x6 (fo-breakglass-template-skill-invoke-directive)

Entity 2x6's scope is "add `Skill(skill="spacedock:ensign")` directive to the FO breakglass template body." This restructure **does** absorb that work — the same PR that lands `cmd_build` restructure also edits the breakglass template body to prepend the Skill-invoke directive (see "Breakglass FO template" above). Recommend closing 2x6 as REJECTED-superseded with a source-note pointing at this entity. If 2x6 lands first in isolation, this entity's breakglass-template edits become a no-op for that file; the rest of this restructure is unaffected. The two entities are coordinated but not blocking.

## Acceptance criteria

End-state properties of the finished entity. Each is verifiable by a future reader without re-running the spike.

1. **The emitted dispatch prompt for the canonical fixture (team-mode, worktree-backed, no feedback, no scope notes, 1 standing teammate) is at most 3500 chars** — i.e., at least 4300 chars shorter than the baseline captured pre-restructure. The baseline golden file `tests/fixtures/dispatch_prompt_baseline.txt` is checked in unmodified during implementation (captured from production `claude-team build` against `tests/fixtures/dispatch_inputs_canonical.json`) and the fetchspec golden file `tests/fixtures/dispatch_prompt_fetchspec.txt` is captured from the restructured helper against the same input.
   - **Test:** golden-diff in `tests/test_claude_team.py` asserts the fetchspec output is byte-equal to `dispatch_prompt_fetchspec.txt` AND the file is at most 3500 chars. The numeric ceiling is the regression boundary; if a future commit re-inlines content, the byte count grows and the test fails. Both golden files live in `tests/fixtures/`.

2. **The emitted spec JSON's stdout for the same fixture is at most 4000 chars** — the FO-side read-cost surface. Today: ~8400.
   - **Test:** in the same golden-diff test, assert `len(stdout) <= 4000`.

3. **The emitted prompt contains a `### Fetch commands` section listing exactly two literal `claude-team show-…` commands for a team-mode dispatch with one or more standing teammates, exactly one (`show-stage-def`) for a bare-mode or no-standing-teammate dispatch, and is structurally well-formed (each command on its own four-space-indented line under the heading).**
   - **Test:** parser-level test in `tests/test_claude_team.py` runs the helper against three canonical inputs (team+standing, team+no-standing, bare-mode) and asserts the `### Fetch commands` block contains the expected commands per fixture.

4. **`claude-team show-stage-def --workflow-dir DIR --stage STAGE` exists, exits 0 on a known stage, exits non-zero on a malformed stage heading with the same parser-diagnostic message `extract_stage_subsection` raises today, and emits stdout byte-equal to the string `cmd_build` currently inlines for that stage.**
   - **Test:** `tests/test_claude_team.py` extension. One test invokes the subcommand against a real workflow README, asserts stdout equals what `extract_stage_subsection` returns directly. One test invokes against a malformed-heading fixture under `tests/fixtures/` and asserts the parse-error message matches the existing `extract_stage_subsection` error.

5. **`claude-team show-standing --workflow-dir DIR --team TEAM` exists, exits 0 (possibly with empty stdout when no standing mods are declared), and emits stdout byte-equal to the standing-teammates markdown block `cmd_build` currently assembles inline at lines 406-437.**
   - **Test:** `tests/test_claude_team.py` extension. Three sub-tests: (a) a team+workflow with one standing teammate; (b) a team+workflow with multiple standing teammates; (c) a workflow with no standing teammates (empty stdout). Each asserts the subcommand's stdout equals the corresponding slice of the current `cmd_build`-assembled prompt.

6. **The ensign skill body (`skills/ensign/references/ensign-shared-core.md`) contains a top-level section titled `## Fetch-on-Demand Bootstrap` that names the `### Fetch commands` heading, instructs Bash execution in listed order, defines the empty-fetch-block degenerate case as "skip this step," and instructs SendMessage-to-FO on a non-zero exit.**
   - **Test:** static-content test in `tests/test_agent_content.py` (extension) reads the shared-core file and asserts each of the four properties above as substring presence.

7. **The FO break-glass dispatch template at `skills/first-officer/references/claude-first-officer-runtime.md:114-123` contains `Skill(skill="spacedock:ensign")` as the first prompt-body element AND contains no `### Fetch commands` block (it stays inlined).**
   - **Test:** static-content test in `tests/test_breakglass_dispatch_prompt.py` (new file, ~30 lines) asserts both properties as substring presence/absence on the runtime-adapter file's breakglass template body.

8. **An end-to-end test confirms a real ensign dispatched via the restructured helper (with both fetch commands present and stage report instructions absent from the inlined prompt) successfully reads the stage definition by running the fetch commands and writes a well-formed stage report.**
   - **Test:** new file `tests/e2e/test_fetch_on_demand_dispatch.py` extending the existing E2E harness used by `tests/test_merge_hook_guardrail.py`. Uses a real tiny workflow + entity fixture (committed under `tests/fixtures/e2e_fetch_workflow/`), runs the production `claude-team build` then dispatches via the test harness's `Agent` mock or live Agent in CI, and asserts (a) the entity file post-dispatch contains a `## Stage Report:` section, (b) each checklist item appears in the report with `DONE:`/`SKIPPED:`/`FAILED:` markers, (c) the test ensign's transcript contains successful Bash invocations of both fetch commands. Marked `@pytest.mark.live_agent` for opt-in execution if the existing E2E suite uses that pattern.

9. **The session-scale FO-side context-burn savings — measured by running the canonical fixture through both production `claude-team build` and the restructured helper, summing `(stdout chars + prompt chars)` for each, and computing the per-dispatch delta — is at least 9500 chars per dispatch.** (Spike measured 10,035; the 9500 floor pads ~5% for any small additive prose changes during implementation.)
   - **Test:** measurement test in `tests/test_claude_team.py` reads both golden files (`dispatch_prompt_baseline.txt` + a checked-in `dispatch_stdout_baseline.json`, plus their fetchspec equivalents) and asserts the delta sum is ≥9500.

## Test plan

Tests are split across three layers: parser-level (cheap, deterministic), static-content (cheap, regression-pinning), and one E2E (expensive, opt-in). The whole suite ships in the same PR as the helper restructure.

- **`tests/test_claude_team.py`** (existing file, extend). Parser/fixture-level. ~80 lines added.
  - Golden-diff: assert restructured `cmd_build` output is byte-equal to checked-in `tests/fixtures/dispatch_prompt_fetchspec.txt`. (AC-1)
  - Numeric-ceiling: assert prompt ≤3500 chars AND stdout ≤4000 chars for canonical fixture. (AC-1, AC-2)
  - Three-fixture fetch-commands shape test. (AC-3)
  - `show-stage-def` happy-path + malformed-heading tests. (AC-4)
  - `show-standing` three-fixture tests (one teammate / multiple / empty). (AC-5)
  - Session-scale delta measurement. (AC-9)

- **`tests/test_agent_content.py`** (existing file, extend). Static-content. ~15 lines added.
  - Shared-core `## Fetch-on-Demand Bootstrap` section presence + four-property check. (AC-6)

- **`tests/test_breakglass_dispatch_prompt.py`** (new file). Static-content. ~30 lines.
  - Breakglass template carries Skill-invoke directive + no fetch-commands block. (AC-7)

- **`tests/fixtures/`** (new files, committed before/with the helper change).
  - `dispatch_inputs_canonical.json` — canonical input for the team-mode worktree-backed dispatch.
  - `dispatch_prompt_baseline.txt` + `dispatch_stdout_baseline.json` — captured from production `claude-team build` before the change, checked in as the regression floor.
  - `dispatch_prompt_fetchspec.txt` + `dispatch_stdout_fetchspec.json` — captured from restructured `claude-team build`, checked in as the post-change golden.
  - `dispatch_inputs_bare.json` + `dispatch_inputs_no_standing.json` — fetch-commands shape variants for AC-3.
  - `malformed_stage_heading_readme.md` — AC-4 sad-path.
  - `e2e_fetch_workflow/` — minimal workflow + entity for AC-8.

- **`tests/e2e/test_fetch_on_demand_dispatch.py`** (new file). Live-agent E2E. ~120 lines.
  - One test: dispatch a tiny ensign via restructured helper, assert stage report appears and fetch commands ran. (AC-8)

**Estimated complexity (implementation):**
- `skills/commission/bin/claude-team`: ~120 lines net. ~40 lines added (two new `cmd_show_*` subcommands, argparse wiring); ~30 lines refactored (extract standing-section rendering into a function `show-standing` and `cmd_build` both call); ~50 lines simplified in `cmd_build` (replace inlined parts 2 + 9 with fetch-command appends, drop the Stage Report block under part 8 and the plain-text-only paragraph under part 10).
- `skills/ensign/references/ensign-shared-core.md`: ~15 lines added (new `## Fetch-on-Demand Bootstrap` section).
- `skills/first-officer/references/claude-first-officer-runtime.md`: ~3 lines added (Skill-invoke prepended to breakglass template body), 1-2 lines edited in the explanatory paragraph at line 123.
- Tests: ~245 lines added across four files; ~10 lines of existing assertions in `tests/test_claude_team.py` updated to match new prompt shape.
- Fixtures: ~8 new files, mostly small.

No schema changes (output adds an optional `fetch_commands` field; existing consumers ignoring unknown fields work unchanged). No `claude-ensign-runtime.md` or `codex-ensign-runtime.md` edits required — the new Fetch-on-Demand Bootstrap section lives in the shared-core and applies to both runtimes.

## Stage Report: ideation

- DONE: Run a real spike to PROVE the fetch-on-demand model works end-to-end before writing the design. Build a minimal proof-of-concept: a Python script (call it `claude-team-fetchspec` in scratch space, separate from the production helper) that emits a dispatch spec for the same inputs as the cycle-1 4q9 dispatch (saved at `/tmp/dispatch-4q9-ideation.json` for inputs, and `/tmp/dispatch-4q9-prompt.txt` for the current emitted prompt). Replace the inlined stage def + standing section + boot directive with literal Bash fetch commands. Then prove an ensign can actually boot from it: dispatch a tiny test ensign whose checklist is `1. Run the fetch commands listed in your prompt; 2. Confirm you can read the stage definition; 3. Report what you read.` Measure: helper stdout chars (today ~9000); FO Agent prompt arg chars (today ~8500); ensign first-action latency vs current. Output: the spike script in `/tmp/0x9-spike/`, a measurements report inline in the entity body's `### Spike report` subsection, and a clear PASS or FAIL verdict on whether the model holds.
  Spike script at `/tmp/0x9-spike/claude-team-fetchspec` (165 lines); inputs/outputs at `/tmp/0x9-spike/dispatch-4q9-*.{json,txt}`; measurements + PASS verdict in entity body `## Spike report`. The captain-asked live-Agent dispatch was replaced by a Bash simulation of the ensign's first action (this ensign role has no direct `Agent()` access from inside team mode) — see Spike report "What the spike didn't validate" for why this is the strongest empirical method the role affords. Real measurements: 8402→3445 helper stdout (−59%), 7867→2789 prompt (−65%), 16,269→6234 FO-side context burn per dispatch (−10,035 chars). Unknown-unknown caught: `claude-team list-standing` emits paths only; design requires a new `claude-team show-standing` subcommand.
- DONE: Translate the spike findings into a complete design that implementation can execute. Populate the entity body with `## Design`, `## Acceptance criteria`, and `## Test plan`.
  `## Design` covers helper-side (which sections inline vs. fetch, plus two new subcommands `show-stage-def` and `show-standing`), ensign-side (new `## Fetch-on-Demand Bootstrap` section in shared-core), FO-side savings math (10,035 chars/dispatch, 200K chars/20-dispatch session), and breakglass disposition. `## Acceptance criteria` has 9 entity-level end-state items each with a named test. `## Test plan` enumerates 4 test files + fixtures + complexity estimate (~120 lines helper, ~245 lines tests).
- DONE: Address the skip-skill-load failure mode + the breakglass-template question.
  `## Design` → "Skip-skill-load failure mode" walks all three skip-paths (A: Skill loads, B: Skill skips but fetch runs, C: fetch skips): each is functional/detectable/recoverable, none destructive; load-bearing structural guard is the boot directive's literal preservation. `## Design` → "Relationship to entity 2x6" recommends closing 2x6 as REJECTED-superseded; the breakglass-template Skill-invoke edit lands in the same PR but the breakglass body itself stays fully-inlined (no fetch commands) because the breakglass path fires precisely when the helper is unavailable.

### Summary

The spike PASSED with one unknown-unknown caught: the existing `claude-team list-standing` emits mod paths, not the rendered markdown block `cmd_build` builds inline, so the production design adds a new `claude-team show-standing` subcommand (plus a sibling `show-stage-def`) extracted from the current rendering helpers. Real measurements on the same input show 10,035 chars/dispatch saved in FO-side context burn — about 5–6% of standard 200K budget per dispatch and ~200K freed over a 20-dispatch session, well above the 4q9 cycle-2 saving (~990 chars) that was rejected as too small. The design absorbs entity 2x6's breakglass-template Skill-invoke edit and recommends closing 2x6 as REJECTED-superseded.

### Feedback Cycles

#### Cycle 1 — rejected at ideation gate (2026-05-20)

Staff reviewer verified the spike measurements reproduce exactly (8402→3445, 7867→2789, FO-side burn 16,269→6234 = −10,035 chars/dispatch) and confirmed all cited line ranges in `claude-team`, `ensign-shared-core.md`, `claude-ensign-runtime.md`, and `claude-first-officer-runtime.md` are accurate. The 9-row classification table is correct, both new subcommand refactors are mechanically achievable, and the breakglass-NOT-restructured decision correctly avoids the circular dependency. Reviewer judgment: "spike numbers reproduce exactly, source citations are accurate, the section-table classification is correct, and the two new-subcommand refactors are mechanically sound."

Five material findings need to be addressed in cycle 2; none invalidates the design, all are tightening work. Plus one captain-added design item on runtime boundary disposition. Reuse OK (11.3% of 1M context budget). Cycle 2 brief:

1. **AC-8's `@pytest.mark.live_agent` does not exist.** Real markers at `pyproject.toml:14-20` are `live_claude`, `live_codex`, `serial`, `teams_mode`, `bare_mode`. Replace with `@pytest.mark.live_claude` and `@pytest.mark.teams_mode` (matching `tests/test_merge_hook_guardrail.py:180-182`). Also: `tests/e2e/` directory does not exist — `tests/` is flat. Either create the directory and call it out, or drop the `e2e/` prefix and put the new file under `tests/test_fetch_on_demand_dispatch.py` to match repo convention.
2. **Borrow framing on entity line 156 understates inheritance.** That line names only row-8c (Stage Report appending-instructions) as borrowed from 4q9 cycle-2; but line 158 also inherits row-10b's plain-text-only-no-JSON drop. Per `_archive/claude-team-build-dispatch-prompt-shrink.md:99`, cycle-2 had three drops; 0x9 inherits two of them. Rewrite to name both inherited drops, or move row-10b's borrow-justification to its own line. The reviewer's job is to verify what's inherited — make finding inherited drops mechanical.
3. **Shell-quoting in the literal `### Fetch commands` block is undefined.** The spike emits commands unquoted (`/tmp/0x9-spike/dispatch-4q9-fetchspec-prompt.txt:24-25`). Production workflow paths with spaces, parens, or dollar signs would break. Add an AC (or sub-bullet on AC-3) requiring `shlex.quote` on all user/workflow-data args (`workflow_dir`, `team_name`, `stage`) plus a parser-level test using a fixture with `workflow_dir` containing a space. The current convention may prevent metachars today, but no validator enforces it; production code should not assume.
4. **Path-C "most checklists are self-explanatory" is asserted, not evidenced.** Reviewer's spot-check shows real checklists routinely reference Inputs/Outputs/Good/Bad framing from the stage def (the 4q9 cycle-3 review checklist explicitly does). Either sample 5-10 recent dispatch checklists from `_archive/*.md` Stage Reports in the entity body and judge each "self-contained or not" — then state which case dominates — or downgrade the failsafe claim to "ensign can do approximate work with stage-Good/stage-Bad fidelity degraded; re-dispatch fixes it." The current phrasing reads as defensible-by-assertion.
5. **`--team` flag rationale unclear.** Entity line 166 implies `enumerate_declared_standing_teammates` uses `team_name` for more than bare-mode short-circuit, but reviewing `claude-team:659-662` shows `team_name` is just a "is this team mode?" boolean (the mod scan itself is `_mods/`-based and team-agnostic). Either say so explicitly ("`--team` is a team-mode boolean; mod enumeration is filesystem-only"), or replace `--team` with a `--bare-mode` flag if that's what we really mean.

6. **NEW design item from captain: add a `### Runtime-boundary disposition` subsection to `## Design`.** The captain raised a sharp question: `claude-team` is named Claude-specific but ~half of what 0x9 adds is runtime-neutral (extraction, enumeration, line-range computation) and ~half is Claude-specific (SendMessage routing in the standing-teammates body, Agent() spec format, .claude/teams/ context-budget reader). Within cycle 2:

   (a) Add the subsection, stating where each piece of new 0x9 code falls on the runtime-neutral / Claude-specific boundary. Use this cut-line:
       - **Runtime-neutral**: `extract_stage_subsection`, `enumerate_declared_standing_teammates`, `show-stage-def` subcommand (pure README extraction), the `### Fetch commands` block format, line-range computation
       - **Claude-specific**: `show-standing` rendering output body (SendMessage-flavored routing), `cmd_build`'s completion-signal section, `cmd_spawn_standing`'s Agent spec, `cmd_context_budget`'s `.claude/teams/` reader
   (b) Annotate the new runtime-neutral functions in implementation with a `# RUNTIME-NEUTRAL` comment marker (or place them in a clearly-marked module file like `claude_team_core.py` imported by `claude-team`) so future extraction is mechanical.
   (c) **File a follow-up entity** in the same workflow (the FO will do this when the entity is approved at the next gate; do not file it yourself). Suggested slug: `claude-team-helper-splits-into-runtime-neutral-core-plus-claude-adapter`. The entity exists to make the boundary explicit on the board — even if it doesn't ship for months, the conflation is visible. The 0x9 entity's `### Runtime-boundary disposition` subsection should reference the future entity's intended slug.

   Why not split now: scope creep. The split touches every caller of `claude-team` (FO runtimes, tests, mods). The hybrid path keeps 0x9 bounded and signposts the split for a follow-up.

   Why mention it at all: the existing Spacedock idiom is `{x}-shared-core.md` + `claude-{x}-runtime.md` + `codex-{x}-runtime.md` (see `first-officer-shared-core.md`, `ensign-shared-core.md`, and their Claude/Codex adapters). The helper hasn't yet adopted this shape. Extending the pattern is consistency, not invention.

Polish-tier (also do these while you're in there; absorb into implementation if not): dedupe the issue number references (currently 3 places — frontmatter, scale-context, body); label the path-B failsafe (entity line 254) as plausible-but-unverified since `agent-boot-skill-preload.md:23-26` documents haiku skipping reference loads entirely, not selectively-following-one-heading; tighten AC-3's "bare-mode or no-standing-teammate dispatch" wording to "bare-mode OR team-mode with zero declared standing teammates."
