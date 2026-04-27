---
id: 217
title: "Pluggable id-style with collaboration-friendly generated IDs"
status: ideation
source: "GitHub issue #150 (filed by CL, 2026-04-27)"
started: 2026-04-27T21:32:17Z
completed:
verdict:
score: 0.66
worktree:
issue: "#150"
pr:
mod-block:
---

Spacedock should support pluggable `id-style` strategies and add a generated ID style that is better for collaborative workflows than centrally allocated sequential numbers.

## Problem statement

`id-style: sequential` is readable and works well for single-writer workflows, but it makes entity creation a coordinated write. Today `status --next-id` computes the next numeric ID from active plus archived entities. If two people or agents create different task files from stale branches, worktrees, or offline clones, both can legitimately choose the same next number. Git may merge the files cleanly because their slugs differ, but Spacedock now has a semantic duplicate in the `id` field.

Spacedock already treats the filename slug as the operational path key, while the `id` field is the human-facing compact identity shown in status tables, stage reports, dispatch output, PR bodies, and issue references. This task should make that identity policy explicit and pluggable, preserve existing sequential workflows, cover the `id-style: slug` request from GitHub issue #98, and provide a generated style that is safe for concurrent and offline creation without a central reservation step.

## Prior-art findings

- Shortest-unique-prefix identifiers use a prefix index over the set of known keys. Trie/PATRICIA-style structures are established prefix-retrieval structures: a trie stores strings by common prefixes, and PATRICIA compresses sparse trie branches. For Spacedock this maps to "store a full ID, display the shortest prefix that is unique across this workflow." It is compact, but the displayed prefix is contextual and can grow after another branch imports a colliding prefix.
- Git accepts full object IDs or a unique leading substring within the repository. `git rev-parse --short` shortens to a unique prefix, and `git describe --abbrev` can emit a longer suffix as the repository grows. This is strong prior art for accepting unambiguous prefixes and rejecting ambiguous ones, but it also shows the tradeoff: a prefix that was sufficient yesterday may not be sufficient after new objects arrive.
- RFC 9562 UUIDs are 128-bit identifiers that require no central registration. The RFC explicitly frames UUIDs as an alternative when coordinating auto-increment values across distributed systems is burdensome. UUIDv7 adds time-ordering, but any short-prefix display of UUIDv7 must account for its timestamp-heavy left side.
- ULID stores a 48-bit millisecond timestamp plus 80 bits of randomness and uses a 26-character Crockford Base32 string. KSUID stores a 32-bit timestamp plus 128 random bits and uses a 27-character Base62 string. These are useful full-ID formats, but their leading timestamp component means many IDs created near each other share the same early characters, so shortest-prefix display must often include enough random suffix to become useful.
- Nano ID is closer to the stored-short-ID option: choose an alphabet and length, then budget collision risk with birthday-paradox math. Its default 21-character URL-safe ID targets UUID-v4-like collision probability, but Spacedock can choose a shorter human-friendly generated ID because workflows are expected to have far fewer entities than large databases.

Sources checked: [Git rev-parse](https://git-scm.com/docs/git-rev-parse), [Git describe](https://git-scm.com/docs/git-describe), [RFC 9562 UUIDs](https://www.rfc-editor.org/rfc/rfc9562.html), [ULID spec](https://github.com/ulid/spec), [KSUID README](https://github.com/segmentio/ksuid), [Nano ID README](https://github.com/ai/nanoid), [NIST trie definition](https://xlinux.nist.gov/dads/HTML/trie.html), [PATRICIA notes](https://www.allisons.org/ll/AlgDS/Tree/PATRICIA/), [GitHub issue #98](https://github.com/clkao/spacedock/issues/98), and [GitHub issue #150](https://github.com/clkao/spacedock/issues/150).

## Proposed approach

Introduce an `id-style` strategy layer used by the status viewer, commission scaffolding, and first-officer task creation guidance. The initial supported styles should be:

- `sequential`: the current behavior. `id` is required, numeric-looking values continue to sort/display as today, and `status --next-id` returns the next zero-padded number across active plus archived entities.
- `slug`: the issue #98 behavior. `id` is optional; the effective identity is the entity slug. Status output should keep the existing columns for compatibility and render the effective ID as the slug. `status --next-id` should fail with a clear "not applicable for id-style: slug" error because a future slug depends on the entity title, not a central counter.
- `generated`: a new collaboration-friendly style. `id` is required and stores a short generated ID, not a dynamically shortened view of a longer hidden value. `status --next-id` returns a fresh candidate generated from a CSPRNG, checks it against active plus archived entities, and retries on known collisions.

The recommended generated format is a 12-character Crockford Base32 string normalized to lowercase, using the ULID-style alphabet without visually confusing letters. That gives 60 bits of entropy. Birthday-bound collision probability remains negligible for Spacedock-scale workflows: about 4.3e-9 at 100,000 generated IDs and about 4.3e-7 at 1,000,000 generated IDs, before the local collision check and retry path. A deterministic test hook can force collisions so the implementation proves retry and failure behavior without relying on chance.

## Design comparison

**Full stable ID plus shortest unique display prefix.** This is the Git-like design: store a full UUID/ULID/KSUID/NanoID value and display/accept the shortest unique prefix in the workflow. It minimizes real collision risk and lets the display length grow with the corpus. The downside is collaboration ergonomics: a prefix copied into chat, a PR body, or a stage report can become ambiguous after another branch adds an ID with the same prefix. ULID/KSUID/UUIDv7 also have timestamp-leading strings, so short prefixes are less compact for bursts. This is a good future display mode, but it is not the best first collaboration-friendly storage mode.

**Stored short generated IDs with collision handling.** This is the recommended generated style. The visible ID is the canonical stored ID, so references remain stable after merges. Creation needs no central reservation and no shared state beyond a best-effort local collision check. The risk is probabilistic rather than impossible, so validation must include explicit duplicate detection across active, archived, flat, and folder-form entities. The fixed length is a product choice; 12 Crockford Base32 characters are a good default for readability and collision budget.

**Pluggable `id-style` strategy.** This should wrap both existing sequential behavior and new styles behind the same strategy contract. It avoids special-casing `slug` separately from generated IDs and makes the README frontmatter field meaningful. The status script can remain stdlib-only by using a small in-file registry first; if the behavior grows, it can later move to a helper module without changing the workflow schema.

## Behavior details

The strategy contract should answer these questions for a workflow:

- whether `id` is required in entity frontmatter
- how to compute the effective display ID for a scanned entity
- how `status --next-id` behaves
- how references are resolved from exact IDs, ID prefixes, and slugs
- whether active and archived entities participate in uniqueness checks
- what migration checks are required before changing styles

Reference resolution should be explicit and testable. Within a workflow, exact slug always resolves to that slug's canonical entity file. Exact ID resolves to the entity with that effective ID. For `generated`, an ID prefix resolves only when it is unique across active plus archived entities; ambiguous prefixes fail with the matching IDs/slugs. Operational mutations such as `--set` should only mutate active entities, but they should still report when the only match is archived instead of silently ignoring it. Read-only resolution may include archived entities when requested.

Folder-form entities should use the existing discovery rule: `{slug}/index.md` is the canonical entity file, and folder form wins over a same-slug flat file with a warning. Archived folder entities live under `_archive/{slug}/index.md`. Every uniqueness check, `--next-id` check, status display, and resolver should use the same discovery path so flat, folder, active, and archived entities cannot disagree.

Cross-workflow references are scoped to a single `--workflow-dir` by default. A resolver operating above one workflow must require a workflow qualifier, an explicit workflow path, or an unambiguous discovered workflow. If two workflows contain the same slug, ID, or generated prefix, the unqualified reference should fail and list candidate workflow directories. This avoids treating generated IDs as globally meaningful when Spacedock's workflow directory is the real namespace.

Migration should be explicit and non-destructive. Existing workflows with `id-style: sequential` require no data rewrite. New workflows may choose `slug` or `generated` at commission time. A migration helper or documented refit path should support a dry-run first, report duplicate effective IDs, include archived and folder-form entities, and only then rewrite README `id-style` plus entity frontmatter. Sequential-to-generated migration should preserve the old value in `legacy-id` so old numeric references remain searchable during transition. Slug migration should reject workflows whose slugs are not unique under the flat/folder discovery rules.

Backward compatibility matters more than making the new abstraction elegant. Existing status output, `--next`, `--boot`, `--where`, `--set`, `--archive`, and first-officer task creation should continue to work for sequential workflows. The phrase "next ID" can remain the CLI name for compatibility, but the docs and runtime prompts should stop saying it is always sequential once strategies exist.

## Acceptance criteria

**AC-1 - Workflows have a documented `id-style` strategy contract covering `sequential`, `slug`, and `generated`.**
Verified by: static tests over `skills/commission/SKILL.md`, `docs/plans/README.md` template text, and first-officer runtime references confirm the three styles, required/optional `id` semantics, and `--next-id` behavior are documented without calling all IDs sequential.

**AC-2 - Existing sequential workflows keep their current behavior.**
Verified by: existing and updated `tests/test_status_script.py::TestNextIdOption` fixtures show `status --next-id` returns the next zero-padded numeric ID across active and archived entities, and `make test-static` passes without changing current sequential output expectations except where wording is intentionally generalized.

**AC-3 - `id-style: slug` treats the slug as the effective ID and makes `status --next-id` non-applicable.**
Verified by: status-script fixtures with omitted `id` fields show default and `--next` tables render a usable effective ID from the slug, `--where` still works, and `status --next-id` exits non-zero with a clear slug-style message.

**AC-4 - `id-style: generated` stores stable short generated IDs and generates new candidates without central allocation.**
Verified by: status-script tests run `status --next-id` under generated style with a deterministic RNG hook, assert the output matches the configured generated alphabet/length, assert active plus archived collisions are retried, and assert forced retry exhaustion fails loudly.

**AC-5 - Entity reference resolution is deterministic for slugs, exact IDs, and generated ID prefixes.**
Verified by: resolver tests cover exact slug, exact ID, unique generated prefix, ambiguous generated prefix, unknown reference, active-only mutation resolution, and read-only archived resolution, with assertions on the selected canonical path or error candidate list.

**AC-6 - Archived and folder-form entities participate in ID uniqueness and resolution consistently.**
Verified by: fixtures containing active flat files, active `{slug}/index.md`, archived `.md` files, archived `{slug}/index.md`, and flat/folder same-slug conflicts show status display, `--next-id`, resolver behavior, and warnings all use the same folder-preferred discovery rule.

**AC-7 - Cross-workflow ambiguity is rejected rather than guessed.**
Verified by: temp-root fixtures with two discovered workflows containing the same slug, exact ID, or generated prefix show unqualified resolution fails with both workflow paths, while an explicit `--workflow-dir` or workflow qualifier resolves the intended entity.

**AC-8 - Migration behavior is explicit, auditable, and backward compatible.**
Verified by: migration/refit tests cover dry-run duplicate detection, sequential-to-generated mapping with `legacy-id`, slug migration rejection on discovery conflicts, archived/folder inclusion, and no-op behavior for existing sequential workflows.

**AC-9 - Concurrent or offline task creation is safer under generated IDs than under sequential IDs.**
Verified by: a filesystem-level test simulates two isolated creators adding different entity files from the same starting workflow; generated style yields distinct IDs without a shared counter, while a forced generated-ID duplicate is detected by the validation/check command before status output can silently present duplicate effective IDs.

## Test plan

Most proof should be offline unit and fixture tests because the behavior is local file parsing, ID generation, and deterministic reference resolution. Live E2E tests are not required unless implementation also changes first-officer runtime dialogue around filing new tasks.

- Add focused status-script tests in `tests/test_status_script.py` for README `id-style` parsing, `--next-id` delegation, generated collision retry, slug-style non-applicability, resolver behavior, archived entities, and folder-form entities. Estimated cost: medium; the file already has helpers for active, archived, and folder-form workflows.
- Add static content tests in `tests/test_agent_content.py` and commission-template tests so docs and runtime references no longer hard-code "sequential" where they mean strategy-dependent "new ID". Estimated cost: low.
- Add migration/refit tests only if the implementation includes an executable migration helper. If migration is documented but not automated, the test should be a static contract test plus a status validation test that detects duplicate or invalid effective IDs before migration. Estimated cost: low to medium depending on scope.
- Add one concurrency simulation using temporary directories or branches, not live agents. It should copy the same starting workflow into two isolated locations, create entities under generated style, merge/copy the files into one workflow, and run the status validation/resolver checks. Estimated cost: medium.
- Run `uv run pytest tests/test_status_script.py -q` during implementation for the core behavior, then `make test-static` before validation. If first-officer live creation prompts change materially, add a small runtime fixture or transcript-level test before considering expensive live E2E.

## Stage Report: ideation

- DONE: Prior-art research covers shortest-unique-prefix identifiers, Git abbreviated hashes, UUID/ULID/KSUID prefix variants, NanoID-style collision budgeting, and maps them to Spacedock collaboration constraints.
  Evidence: Prior-art section cites trie/PATRICIA, Git `rev-parse`/`describe`, RFC 9562, ULID, KSUID, and Nano ID, then maps each to Spacedock tradeoffs.
- DONE: Proposed design compares full stable ID plus shortest unique display prefix, stored short generated IDs with collision handling, and a pluggable `id-style` strategy that can also cover the `slug` style from GitHub issue #98.
  Evidence: Design comparison and proposed approach recommend stored short generated IDs while defining `sequential`, `slug`, and `generated` strategies.
- DONE: Acceptance criteria and test plan specify concrete behavior for reference resolution, `status --next-id`, archived and folder-form entities, cross-workflow ambiguity, migrations, and safer concurrent/offline task creation.
  Evidence: AC-2 through AC-9 and the test plan cover the requested behaviors with fixture-level verification paths.

### Summary

The ideation output recommends a strategy layer over the existing `id-style` field and a 12-character Crockford Base32 generated style for distributed task creation. It keeps sequential workflows compatible, models issue #98 as the `slug` strategy, and defines resolver, archive, folder-form, cross-workflow, and migration behavior at a testable level.
