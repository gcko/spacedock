---
id: 0f1smz7rrgm3rdrw5ssqtrhr
title: FO gate presentation buries the lede and flattens reviewer priority
status: ideation
source: captain (CL) — self-critique during ideation gate on `bw / name-pattern-rejects-stage-names-with-underscores`
started: 2026-05-12T03:38:42Z
completed:
verdict:
score:
worktree:
---

The FO's gate-review message — the one assembled per `skills/first-officer/references/claude-first-officer-runtime.md` `## Gate Presentation` and the shared-core gate flow — is captain-hostile in its current form. A fresh-eye captain reading the gate message I produced for `bw` ideation could not, without scrolling, answer "what direction was chosen" or "what's the one thing I actually need to decide". Concrete defects, in roughly the order they cost the captain attention:

1. **Lede is buried.** The actual decision prompt sits at the bottom of a ~60-line message. Chosen direction, FO verdict, and the single decision question should be the first three lines.

2. **Chosen ideation direction never surfaces in the FO's own prose.** The DONE checklist accounting cites "picks among directions with reasoning" but never names the picked direction. The captain has to infer it from reviewer findings or open the entity file.

3. **Stage Report attribution is ambiguous.** The verbatim ensign report is pasted under an `## Stage Report` header without quoting / fencing / preface — looks like FO-generated content. The runtime adapter's template says "paste verbatim" but doesn't specify rendering. The result blurs whose words those are.

4. **Reviewer findings are flat-bulleted with no priority tier.** A material fact-correction (e.g. "your one-line claim is wrong because file X doesn't currently do Y") sits next to wording polish (e.g. "AC2 substring assertion is over-fitted"). The captain weighs them as equals because the rendering treats them as equals.

5. **Recommendation prose duplicates the decision prompt.** A paragraph that says "I recommend #2" is followed by an enumerated "three paths" list that re-explains #2 with slightly different wording. Pick one form.

6. **Recommended bounce-back doesn't show the fixup.** "Address the reviewer's five concrete notes" forces the captain to scroll up and re-derive them. Show the actual fixup ask in one or two lines, or quote the concrete asks; don't reference "the five notes".

7. **Pedantic format drift is presented as load-bearing.** A paragraph called out "ideation used `1./2./3./4.` instead of `**AC-N**`" — either it blocks the gate or it doesn't. If not blocking, cut it.

8. **No worktree heads-up.** Approval of ideation moves the entity into a worktree-backed stage. One sentence noting "approval will open `.worktrees/...`" gives situational awareness for what `approve` actually does.

9. **Signal-to-length ratio is bad.** ~60 lines for a decision that needs maybe 15. The captain just typed `dispatch` 90 seconds prior; they did not opt into a six-section essay.

## Resolution directions (decide in ideation)

- **Option A: tighten the existing template.** The runtime adapter's current gate template is already short:
  ```
  Gate review: {entity title} — {stage}

  {paste the ## Stage Report section from the entity file verbatim}

  Assessment: {N} done, {N} skipped, {N} failed. [Recommend approve / Recommend reject: {reason}]
  ```
  The defects above are mostly FO-improvisation *beyond* this template. Possible fix: add a "Captain-facing assembly rules" subsection — lede-first chosen-direction line, reviewer-finding priority tiers, no parenthetical pedantry, max-15-line target — and call it the discipline rather than expand the template itself.

- **Option B: expand the template.** Add named slots: `Chosen direction`, `Material findings`, `Polish findings`, `Decision prompt`. Force the assembler to fill them. Risk: rigidity for entities where the structure doesn't fit (e.g. simple non-staff-review gates).

- **Option C: hybrid.** Keep the existing terse template as canonical for simple gates; add an "Expanded gate template (when staff review or material reviewer findings are present)" alternate with explicit slots.

Whichever path, the test for "fixed" is: a captain coming in cold on a gate message can name the chosen direction, the FO's verdict, and the decision question without scrolling.

## Out of scope (call out)

- The Stage Report *format itself* (the `- DONE/SKIPPED/FAILED:` accounting block written by the ensign into the entity body) is not the target here. The ensign-side template is fine. This task is about the FO's *gate-presentation* assembly of that report plus reviewer findings plus FO recommendation.
- Changing the gate decision contract (still captain-only, no FO self-approval, etc.).
