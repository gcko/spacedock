---
id: rdtbc72j3wszp1skfjxvp3m5
title: "Dispatch prompt as file-pointer: shrink FO per-dispatch context from ~14000 to ~380 chars"
status: backlog
source: "Captain question in 2026-05-21 session about Agent-spawn-phase expansion. Research doc at docs/research/2026-05-21-agent-spawn-prompt-expansion.md walks the three spawn-phase expansion mechanisms (skills frontmatter, @-include in skill body, Skill args) and empirically confirms @-include does NOT resolve in Agent() prompt args. The proposed mechanism for FO context compression — `DISPATCH_FILE: {path}` convention with ensign-side Read on first action — composes with PR #231's existing fetch-on-demand pattern at one layer up. Predicted ~97% FO per-dispatch context reduction."
started:
completed:
verdict:
score:
worktree:
issue:
pr:
---

## Problem

After PR #231 (0x9, fetch-on-demand dispatch spec), the FO's per-dispatch context cost is still **~14,000 chars** even though the ensign-side prompt was shrunk significantly:

- ~7,000 chars from `claude-team build`'s stdout JSON (the FO reads this to dispatch)
- ~7,000 chars from `Agent(prompt=...)` (the FO emits this verbatim to spawn the ensign)

The prompt enters the FO's context twice per dispatch. PR #231 only fixed the ensign side (`### Fetch commands` block makes the ensign load stage def + standing teammates JIT). The FO still has to carry the full prompt through its own reasoning surface.

A captain-session of 20 dispatches accumulates ~280,000 chars of FO-side prompt-passing. That's a significant fraction of the FO's context budget that the FO never reasons about — it's purely pass-through cost.

## Why this matters

- **Long-session FO context budget.** The FO's context budget compounds across all dispatches. Reducing per-dispatch cost is the most direct win for session-length viability.
- **Codex parity.** The Codex runtime adapter will face the same issue if it follows the helper output → forward-verbatim shape. Solving this at the helper-protocol layer benefits both adapters.
- **Composes cleanly with 0x9.** PR #231's `### Fetch commands` block is the *ensign-side* fetch-on-demand pattern. This entity applies the *same pattern* at the FO-side: the FO emits a tiny pointer, the ensign Reads the per-dispatch content from a file on first action. Different beneficiaries, same architectural idea.

## Proposed approach

The FO's `Agent(prompt=...)` arg becomes a tiny `DISPATCH_FILE:` pointer (~80 chars). The helper writes the per-dispatch content to a stable path. The ensign Reads the file on first action and treats its content as the assignment.

### Address the four design dimensions

#### 1. What the FO actually assembles (the irreducible per-dispatch context)

The FO MUST author and pass through these per-dispatch elements; they cannot move out of the FO's context:

| Element | Source | Approx. size | Why FO-authored |
|---|---|---|---|
| `checklist` | FO judgment per stage | 800-1500 chars | FO decides what this dispatch must accomplish |
| `scope_notes` | FO judgment per dispatch | 0-2000 chars | Conditional context (corrections, prior-cycle issues, captain feedback to relay) |
| `feedback_context` | Reviewer findings (when re-dispatching) | 0-3000 chars | Per-dispatch input from validation/staff-review |
| `bare_mode` | Live team state | 1 byte | FO observes runtime state |
| `is_feedback_reflow` | Stage transition logic | 1 byte | FO knows when routing rejection back |
| Entity identity | Entity slug + stage + worktree path | ~200 chars | FO decides which entity advances when |
| Team context | `team_name` (when applicable) | ~60 chars | FO's team-create or active-team awareness |

**Estimate: ~1500-7000 chars of genuinely FO-authored per-dispatch input.** This is what the FO needs in its own context to assemble. It already does today via the input JSON it pipes to `claude-team build`. Unchanged by this entity.

What is NOT FO-authored and currently still passes through FO context: stage definition prose, standing-teammates routing instructions, the boot directive, the completion-signal prose, the FO-forwarding warning. **These ~5000+ chars are the target — they should not appear in the FO's prompt context.**

#### 2. How (1) flows into the `claude-team build` helper

Unchanged from today's contract. The FO assembles its inputs as a JSON object and pipes to the helper:

```json
{
  "schema_version": 2,
  "entity_path": "/abs/path/to/entity.md",
  "workflow_dir": "/abs/path/to/workflow",
  "stage": "implementation",
  "checklist": ["1. ...", "2. ..."],
  "scope_notes": "...",
  "feedback_context": null,
  "team_name": "...",
  "bare_mode": false,
  "is_feedback_reflow": false
}
```

The interface is the same JSON schema. **Schema version bumps to 2** to signal the new helper output contract. The helper accepts both v1 (today's inline-prompt shape) and v2 (file-pointer shape) input during a transition window, OR the entity treats this as a breaking change and bumps in one cut. Decision tracked in design below.

#### 3. The mechanism turning (2) into the Agent() tool call

The helper's behavior changes to produce a file-pointer output:

**Helper internal flow:**
1. Read the input JSON from stdin.
2. Assemble the full dispatch prompt (same content as today's PR #231 output) into a Python string.
3. Compute a stable path: `/tmp/spacedock-dispatch/{name}.md` (where `{name}` is the deterministic agent name the helper already computes).
4. Create `/tmp/spacedock-dispatch/` if missing (parent dir).
5. Write the full prompt content to the path.
6. Emit a NEW stdout JSON:

```json
{
  "schema_version": 2,
  "subagent_type": "spacedock:ensign",
  "name": "spacedock-ensign-{slug}-{stage}",
  "team_name": "...",
  "model": null,
  "description": "{entity title}: {stage}",
  "dispatch_file_path": "/tmp/spacedock-dispatch/spacedock-ensign-{slug}-{stage}.md",
  "prompt": "DISPATCH_FILE: /tmp/spacedock-dispatch/spacedock-ensign-{slug}-{stage}.md"
}
```

The `prompt` field is now ~80 chars. The `dispatch_file_path` field is the same path, exposed for debugging / tooling (the FO doesn't need to read it; it only needs `prompt`).

**FO consumption flow:**

Unchanged in shape from today:

```python
spec = json.loads(helper_stdout)
Agent(
    subagent_type=spec["subagent_type"],
    name=spec["name"],
    team_name=spec["team_name"],
    model=spec["model"],
    prompt=spec["prompt"]  # ~80 chars now, was ~7000
)
```

The FO doesn't have to know about the file. It just forwards the helper's `prompt` verbatim as before. The helper has done the work of writing the file and emitting a tiny pointer.

**Failure modes for step 3:**

- **Helper can't write to `/tmp/spacedock-dispatch/`** (permission, disk full): helper exits non-zero with a clear error. FO falls back to break-glass per existing contract.
- **Two concurrent dispatches collide on the same `{name}.md`**: shouldn't happen because the name is deterministic per (entity, stage, cycle). If it does, the second write overwrites the first, both ensigns Read the same content — same prompt for both. Idempotent.
- **File deleted between helper-write and ensign-Read**: rare (no cleanup process between dispatch and first-action). Ensign Read fails, surfaces to FO via `SendMessage` with the missing-file error.

#### 4. What the spawned ensign's initial prompt is, and how reliable

The ensign sees:

```
DISPATCH_FILE: /tmp/spacedock-dispatch/spacedock-ensign-{slug}-{stage}.md
```

That's the entire `Agent(prompt=...)` arg. ~80 chars.

The ensign's behavior depends on its agent-definition + skill body. Today's `agents/ensign.md` says:

```
You are an ensign executing stage work for a workflow.

## Boot Sequence

If your operating contract was not already loaded via skill preloading, invoke the `spacedock:ensign` skill now to load it.

Then read your assignment and begin work.
```

This entity adds a clause to `skills/ensign/references/ensign-shared-core.md` `## First action` section:

> If your initial prompt matches the pattern `DISPATCH_FILE: {path}` (one line, no other content), your first action is `Read({path})` and you treat the file's content as if it had been your inline prompt. Then proceed with the operating contract.
>
> If the file does not exist or Read fails, send `SendMessage(to="team-lead", message="DISPATCH_FILE_MISSING: {path} - {error}")` and stop. Do NOT proceed with empty context.

**Reliability question:** does the model reliably interpret a 1-line `DISPATCH_FILE:` prompt as "Read this file, then proceed"?

The empirical risk surface is twofold:

(a) **Skill load happens before the prompt is read**, per the spawn-phase Mechanism 1 / 2 (verified in `_archive/agent-boot-skill-preload.md` #085 and the production `@`-include resolution in `skills/ensign/SKILL.md:8-9`). So when the ensign sees the `DISPATCH_FILE:` prompt, the ensign-shared-core (including the new clause above) is already in its context. The model has the discipline-prose before it has the prompt.

(b) **However, the skill-preload path is broken for plugin agents per `claude-code#25834`.** The body-fallback `Skill(skill="spacedock:ensign")` invocation is the workaround. But the body-fallback would be in the *file content* (because the helper still emits the boot directive into the file at /tmp/spacedock-dispatch/). The 80-char Agent() prompt arg has no boot directive. So the question becomes: will the model invoke the skill before Read'ing the file?

**Design fork:** the 80-char prompt could include the boot directive inline:

```
Skill(skill="spacedock:ensign"); then Read /tmp/spacedock-dispatch/{name}.md and treat its content as your assignment.
```

That's ~150 chars instead of 80. Still ~98% smaller than today's ~7000. Adds the boot-fallback explicitly. **This is the design's recommended shape** — slightly larger prompt arg, but it incorporates the skill-load-failure failsafe that 0x9 introduced.

**Empirical risk: ensign freelance behavior.** Sonnet-bare on `test_gate_guardrail` works reliably today (verified by `2y` spike + CI). But the 0x9 cycle-4 spike showed that haiku-bare freelances on minimal prompts. If we're targeting sonnet-or-above for bare-mode FO (per `2y`'s declaration), the same model floor probably applies to ensign reliability with minimal prompts. **The spike for this entity must verify sonnet (the supported floor) reliably interprets the `Skill(...); Read X; ...` shape.**

### Spike that gates the design

Before locking any helper changes, run a 5-run spike that empirically verifies the ensign reliably interprets the minimal `DISPATCH_FILE` prompt shape.

**Spike harness:**

1. Build a tiny test fixture: single entity with checklist "Create `/tmp/dispatch-file-pointer-spike-output.txt` containing `OK`."
2. For each of the 5 runs:
   - Write the full dispatch content (today's PR-#231 fetch-on-demand shape) to `/tmp/spacedock-dispatch/spike-N.md`
   - Call `Agent(subagent_type="spacedock:ensign", prompt='Skill(skill="spacedock:ensign"); then Read /tmp/spacedock-dispatch/spike-N.md and treat its content as your assignment.')`
   - Capture fo-log.jsonl + check the output file exists with the expected content
3. **Verdict:** PASS if ≥4/5 runs successfully Read the file AND complete the checklist work. FAIL if any run freelances (does not Read), errors out without clean SendMessage, or fails to complete the trivial task.
4. **Model:** sonnet (the supported floor per `2y`). The spike is NOT required to work on haiku per the supported-models declaration.

Cost: ~$3-5, ~10 min wall-clock.

If PASS → land helper change + ensign-shared-core clause + tests.
If FAIL → record the failure mode, consider whether a slightly-larger prompt (e.g., ~200 chars with explicit ordering + error-handling instruction) closes the gap. If even that fails on sonnet, the proposal is structurally unworkable; ship a no-code-change Stage Report recommending re-evaluation.

## Acceptance criteria

End-state properties of the finished entity:

1. **`claude-team build` (with `schema_version: 2` input) emits a spec where `prompt` is ≤200 chars** and contains the substrings `Skill(skill="spacedock:ensign")`, `Read`, and `/tmp/spacedock-dispatch/` (the predictable path pattern). The full ~7000-char per-dispatch content is written to a file at `/tmp/spacedock-dispatch/{name}.md`.
   - **Test:** parser-level test in `tests/test_claude_team.py` runs the helper against a canonical v2 input and asserts (a) the emitted `prompt` length is ≤200, (b) the three substrings appear, (c) `dispatch_file_path` field is set and the file exists at that path with the expected content.

2. **The helper preserves v1 schema input** (today's shape) for backwards compatibility OR cleanly errors out with a clear migration message. Decision deferred to implementation; either is acceptable, but the behavior is verified by a test.
   - **Test:** parser-level test pipes a v1 input to the helper and asserts (a) either the helper produces a v1-shape output (back-compat), or the helper exits non-zero with a message containing `schema_version: 2 required`.

3. **`skills/ensign/references/ensign-shared-core.md` `## First action` section contains the `DISPATCH_FILE:` clause** with the two properties: (a) instructs Read on a matching prompt pattern; (b) instructs SendMessage-on-failure with `DISPATCH_FILE_MISSING:` prefix.
   - **Test:** static-content test asserts both substrings appear in the file.

4. **Spike result is documented** in the entity body's `## Empirical findings` section with verdict PASS/INCONCLUSIVE/FAIL and per-run table. Captain-set threshold: ≥4/5 PASS.
   - **Test:** entity body content check.

5. **FO context savings are measured and ≥10,000 chars per dispatch** in a worked example. Comparison shape: today's helper-output-prompt vs. v2-helper-output-prompt against an identical input fixture.
   - **Test:** golden-file test pinning the v2 emitted prompt size at ≤200 chars AND a derived test asserting `helper_stdout_size_v2 < helper_stdout_size_v1 - 10000` for the canonical fixture.

6. **`make test-static` passes with the changes applied** — no regressions from the new behavior.
   - **Test:** `make test-static` exit code 0.

7. **`make test-live-claude` from worktree** validates a real dispatch via the v2 path completes end-to-end without behavioral regression vs. PR #231 baseline. This is the captain's cross-cycle live-test requirement.
   - **Test:** `make test-live-claude TEST=test_dispatch_completion_signal` (the natural test exercising single-dispatch shape) exit code 0 with the new prompt shape verified in fo-log.jsonl.

## Test plan

- **Spike** (cost-controlled, gates design): 5 sonnet runs of the trivial `DISPATCH_FILE:` fixture. ~$3-5, ~10 min. Artifacts at `/tmp/rdt-spike/`.
- **`tests/test_claude_team.py`** (extend): 4 new tests covering AC-1, AC-2, AC-5 (helper output shape + size + back-compat + savings floor).
- **`tests/test_agent_content.py`** (extend): 1 new test for AC-3 (ensign-shared-core `DISPATCH_FILE:` clause).
- **Golden file** at `tests/fixtures/dispatch_prompt_v2.txt` for the v2 helper output, checked in with the implementation commit.
- **No new modules.** ~50 lines of Python in `claude-team build` (file-write logic + schema_version handling), ~10 lines of ensign-shared-core prose, ~50 lines of test additions.

## Out of scope

- **Reuse-path dispatch (SendMessage to a kept-alive ensign).** Today's reuse-advance flow doesn't go through `claude-team build`; it's a direct SendMessage with a few hundred chars. Adding `DISPATCH_FILE:` semantics to reuse would compress reuse-advance content too, but the per-dispatch reuse cost is already small (~1500 chars vs ~14000 for a fresh dispatch). The compression benefit/complexity ratio favors deferring reuse-path changes to a future entity. **In scope here: initial Agent() dispatch only.**
- **Breakglass dispatch.** The break-glass manual-assembly path (used when `claude-team build` itself fails) is reserved for catastrophic failure modes; it stays a fully-inlined string per the current FO runtime adapter. Out of scope.
- **Codex runtime adapter.** The proposal is platform-neutral in concept but this entity only ships the Claude path. Codex parity is a follow-up.
- **Workflow-discovery file location.** `/tmp/spacedock-dispatch/` is chosen for simplicity (tmpfs, no cleanup needed across sessions). An alternative under workflow_dir is possible but adds discovery complexity for the ensign. Out of scope unless the spike surfaces a reason.

## Risks

### Risk A — Schema-version bump breaks downstream consumers

Any tool that reads `claude-team build` output today expects v1 shape. The schema_version=2 output has a different prompt size (much smaller) and a new `dispatch_file_path` field. Mitigation: the helper continues to accept v1 input (back-compat AC-2); the output format change is documented; downstream tests (e.g., `tests/test_claude_team_spawn_standing.py`) get updated alongside the helper change.

### Risk B — File-write-during-dispatch race or cleanup gap

The `/tmp/spacedock-dispatch/` directory accumulates files over a long captain session. Each file is ~7000 chars; 100 dispatches = ~700KB. Not a real disk concern, but worth deciding: does the FO clean up the file after Done: signal? Or does the helper overwrite on every dispatch (idempotent name-based addressing)?

**Recommended:** helper overwrites; no cleanup. Same-named dispatches (re-runs of cycle-N implementation) get a fresh file. Stale files from prior sessions are tmpfs-cleaned on reboot. Implementation can verify with a static test that the file path is deterministic per (entity, stage).

### Risk C — Spike-FAIL fallback

If the empirical spike shows the ensign doesn't reliably interpret the `DISPATCH_FILE:` shape, the fallback is option D below (instead of the proposed primary):

**Option D — bake DISPATCH_FILE handling into the agent definition itself.** Edit `agents/ensign.md` to include the DISPATCH_FILE clause directly in the agent's body (which is in spawn-phase context per Mechanism 1). This makes the discipline part of the agent's identity rather than something it has to read from the skill body. Slightly less idiomatic but more reliable.

If spike on the proposed shape FAILS, run a follow-up spike with Option D before recommending REJECTED.

## Scale context

- Spacedock version: 0.11.2+ (post PR #231 + #233)
- Builds on: PR #231 (0x9, fetch-on-demand dispatch spec — established the ensign-side fetch-on-demand pattern this entity extends to FO-side)
- Composes with: `mtc76hh8` (runtime-boundary split — would split `claude-team build` into runtime-neutral core + Claude adapter; the file-write logic in this entity lands in the runtime-neutral core), `y15kpj10` (FO retry-path bypass — orthogonal but related to FO dispatch discipline)
- Predicted impact: ~97% FO per-dispatch context reduction. For a 20-dispatch captain session, ~260,000 chars of FO budget freed.
- Estimated complexity: small. Helper change ~50 lines Python, ensign-shared-core prose ~10 lines, tests ~50 lines. Spike ~$3-5. Total cost ~$15-20.
- Empirical evidence required at ideation gate: 5-run sonnet spike with verdict PASS/INCONCLUSIVE/FAIL per AC-4.
