---
name: orchestrate-implementation
description: Use whenever a reviewed plan exists and implementation work is about to start. Triggers on phrases like "implement this plan", "execute the plan", "build the feature", "start implementation", "let's code this up", "dispatch implementation", "go ahead and build it", or any variant of moving from design/plan to code. This skill is the single authority for execution protocol — how to dispatch Codex with goal-only prompts, inject .claude/rules invariants, enforce verification gates, run reviews, fix gaps, and verify GO/NO-GO. Use BEFORE writing your own Codex prompt — this skill defines the prompt structure and the lifecycle.
---

# Orchestrate Implementation

**This is the single authority for execution protocol.** CLAUDE.md / AGENTS.md define the lifecycle; this skill defines how to execute it. When you have a reviewed plan ready to implement, this is your playbook.

## Choose the execution owner before applying this playbook

Apply the active AGENTS.md scope and review level first. This skill cannot turn a
focused change into a mandatory delegation or independent-review workflow. When
the current executor is already Codex and can own the complete change, implement
in that session; do not spawn another Codex merely to satisfy the examples below.
The “one plan = one invocation” rule applies when implementation is delegated:
keep one owner accountable for the whole result. Independent tests and release
preflight may run concurrently when their state is isolated. Required review
still inspects the complete stable diff and its evidence before release.

Why: the September 9, 2026 résumé comment retrospective found that a Codex parent
delegated the entire job to another Codex, mostly supervised it, and then ran a
serial review. Ownership continuity does not require that extra agent boundary.
Source: `tejasdc/resume`, `docs/2026-09-09-comment-redesign-retrospective.md`.

## The Bitter Lesson — How to Talk to Codex

Codex is smarter than you at codebase-wide analysis. When you narrow its scope — prescribing files, diagnosing root causes, listing steps — you replace its intelligence with yours. Its intelligence is why you're using it.

**Every Codex prompt follows this structure:**
1. **GOAL** — what the outcome should be (behavior, not implementation)
2. **CONTEXT** — pointers to original user requirements/amendments and files (plans, rules, docs). NOT your summary of those files
3. **KNOWN CONCERNS** — things we've already identified as risks. Framed as "these are the things we know to check, but there may be things we don't know — use your judgment to find issues beyond this list"
4. **INVARIANTS** — battle-tested rules from `.claude/rules/` + "Do NOT send changes to any remote repository. Commit locally only."
5. **VERIFICATION GATES** — see below. Non-negotiable. Codex cannot declare done until all gates pass.
6. **MARKER** — `[GOALS-ONLY]` to confirm you followed this structure

## Verification Gates (Non-Negotiable)

**Every Codex implementation/fix prompt MUST end with this exact verification block.** Codex does not get to declare "done" until every gate passes locally. Runtime tests passing is NOT sufficient — TypeScript can fail while runtime tests pass; lint can fail while everything else passes; the pre-push hook will then block the push and you'll iterate again.

```
VERIFICATION GATES (Codex cannot declare done until ALL pass)

Before claiming the task is complete, you MUST run AND pass each gate that exists in the affected package(s). Do not skip a gate because it "should be fine." Run it.

For TypeScript packages (backend/, frontend/, cloudcli/server/, packages/*):
1. Typecheck — run the package's typecheck script. Common shapes:
   - `cd <pkg> && npx tsc --noEmit`
   - `cd <pkg> && npm run typecheck`
   Whichever the package uses. Find it in package.json. Exit 0 required.
2. Lint — run the package's lint script:
   - `cd <pkg> && npx eslint <files-you-touched>`
   - or `npm run lint` if it covers the same surface.
   Exit 0 required.
3. Tests — run the test file(s) covering your change. Exit 0 required, pass count must include any new tests you added.

For other languages, run the equivalent: typechecker, linter, test runner.

If any gate fails, FIX IT in this same invocation. Do NOT commit a state where any gate fails. Do NOT report "tests pass" while typecheck fails — that is a false success.

Report each gate's result in your final summary, with the exact command you ran and the exit code.
```

**Why this exists:** the pre-push hook runs typecheck + lint and BLOCKS the push if either fails. If Codex declares done with passing tests but failing tsc, you will hit the hook, dispatch another Codex iteration, and waste a full cycle. The gates are here to make Codex catch this BEFORE declaring done.

**The test:** If your prompt contains file paths to change, line numbers, function names to fix, or step-by-step instructions — you are prescribing. Rewrite.

**The unknown-unknowns rule:** When listing things to check in a review or implementation prompt, ALWAYS end with: "These are the known concerns. There may be issues we haven't anticipated — use your judgment to find problems beyond this list." Never present a checklist as exhaustive. Codex's value is finding what YOU missed, not confirming what you already know.

---

## Core Principle: The Plan is a Guide, Not an Oracle

For design and implementation reviews, supply the original user request and later
amendments independently of the plan. First map requested outcomes to the artifact;
then assess the proposed solution. An author-added non-goal cannot exclude a user
requirement without evidence that the user changed scope. When the source request
is unavailable, report that alignment is unverified rather than treating the plan's
Goal section as proof. More reviewers do not fix shared, narrowed requirements.

Source: [September 14, 2026 observability retrospective](https://github.com/tejasdc/thinkering/blob/main/docs/plans/2026-09-14-observability-design.md#requirements-and-the-previous-scope-failure).
Both model reviews accepted a diagnosis-only design after the author excluded the
requested logging/metrics/monitoring service comparison from their prompts.

The plan is the best understanding at planning time — but the code is the ground truth. When the implementation agent finds something the plan got wrong (stale line numbers, wrong assumptions, missing edge cases), it must:
1. Fix it in the code (do the right thing, not the planned thing)
2. Document what changed and why: `PLAN DEVIATION: [plan said] -> [did instead] -> [why]`
3. Report it back so deviations can be verified as improvements

---

## Plan Phases Are Commit Boundaries, NOT Dispatch Boundaries

**This is the single most-violated rule.** A plan may have internal staging — Phase 1, Phase 2, Section A, Step 1, etc. Those are for HUMAN UNDERSTANDING and for the implementing agent's COMMIT STRUCTURE as it works.

**They are NOT separate Codex invocations.**

Dispatching "Phase 1" then "Phase 2" then "Phase 3" as three separate Codex runs:
- Destroys the compounding understanding the implementation depends on
- Pays cold-start re-comprehension cost on every restart
- Creates integration seams between phases that no single agent witnessed
- Forces per-phase reviews that miss cross-phase integration bugs

**The rule: one plan = one Codex invocation, regardless of internal staging.** Phase headings are organizing structure — they help the implementing agent think about scope and suggest natural narrative breaks. They are NOT acceptance gates and NOT separate dispatch invocations. The implementing agent decides commit structure as it works.

If a plan says "Phase 1: protocol substrate. Phase 2: lifecycle storage. Phase 3: permission flow" — that means ONE Codex invocation that implements all three. The implementing agent may commit in 1, 4, or 10 commits at whatever narrative breaks make sense to it; that's its judgment, not a prescription from the plan or the orchestrator. The orchestrator runs ONE comprehensive review at the end against the entire feature branch.

If you find yourself drafting "implement Phase N of plan X" in a Codex prompt — STOP. Rewrite to "implement the complete plan at docs/plans/...".

---

## Step 0: Inject Rules into Agent Prompts

**Before dispatching ANY Codex invocation, read relevant `.claude/rules/` files and include them in the prompt.**

Codex does NOT read `.claude/rules/` — only Claude Code sees them. These are battle-tested invariants (security constraints, deployment boundaries, path conventions), not implementation instructions. They tell Codex what NOT to break, not what to build.

**How:**
1. Check which files the plan touches
2. Read matching `.claude/rules/*.md` files (check `paths:` globs)
3. Include rules content in the prompt under an "INVARIANTS" section

This applies to ALL steps below — implementation, review, and fixes.

---

## Step 1: Implement with ONE Codex Invocation

**ONE plan = ONE Codex invocation.** Do not split a plan into multiple Codex batches, phases, or task groups. The Codex agent reads the full plan and implements everything in one pass. Tests are included in the same invocation. Internal phases in the plan are commit boundaries the implementing agent honors, NOT separate dispatch invocations.

Splitting creates partial states, seam gaps, coordination overhead, and false confidence from partial testing. If Codex times out on a large plan, retry with the same full prompt — do not decompose.

Use explicit sandbox and approval flags before `exec`: implementation needs
`-s danger-full-access -a never` when building, fetching dependencies, or running
browsers; reviews use `-s read-only -a never`. Do not use the removed `--full-auto`
alias. Source: résumé comment implementation, 2026-09-09 — Codex CLI 0.153.4
rejected `codex exec --full-auto --help` before starting a session.

```bash
codex -s danger-full-access -a never exec "[GOALS-ONLY] Implement the complete reviewed plan at docs/plans/YYYY-MM-DD-feature.md.

INVARIANTS (from .claude/rules/):
[paste relevant rules content here]

The plan has been reviewed and approved. Follow it, but fix issues you find in the actual code and flag deviations.

Implement ALL tasks AND ALL test updates in one pass. Do NOT divide the implementation. Do NOT skip any tasks. Do NOT leave test updates for a separate pass. If the plan has internal phases, treat them as organizing structure — commit at whatever narrative breaks make sense to you, NOT as stopping points where you return for the orchestrator to dispatch the next phase.

After implementing, check whether your changes make any existing code redundant or superseded. If so, remove the redundant code — one clean path, not two overlapping paths.

Format deviations as: PLAN DEVIATION: [what the plan said] -> [what you did instead] -> [why]

Do NOT send changes to any remote repository. Commit locally only." 2>&1 | tee tmp/reviews/<name>.implement.log
```

Run with `run_in_background: true`. Do other work while waiting.

---

## Step 2: Review — Code Correctness AND Intention

After implementation completes, launch a SEPARATE Codex instance to review. This review checks TWO dimensions:

1. **Code correctness** — Does the code work? Are there bugs, missing imports, type errors?
2. **Intention alignment** — Does the plan and implementation achieve the original user requirements and amendments? Check their coverage before using the plan's tasks to assess the code. Correct implementation of an incorrectly scoped plan is still the wrong solution.

```bash
codex -s read-only -a never exec "Review the implementation against the original user requirements and amendments at tmp/reviews/<name>.requirements.md, then the plan at docs/plans/YYYY-MM-DD-feature.md. The requirements file must contain the source request, not the plan author's paraphrase.

TWO REVIEW DIMENSIONS:

1. CODE CORRECTNESS — For every task and acceptance criterion in the plan:
   - Was it implemented? [DONE], [MISSING], [DEVIATED], or [WRONG]
   - Are there bugs, race conditions, missing imports, type errors?
   - Do plan deviations introduce regressions?
   - Were any plan items skipped entirely?

2. INTENTION ALIGNMENT — Read the original requirements and amendments independently of the plan:
   - Do the plan and implementation cover the requested outcomes? Are exclusions user-authorized?
   - Is this the right architecture for the problem?
   - Could the code pass all task checks but still fail to achieve the intended outcome?
   - Are there architectural decisions that technically work but miss the point?

3. HINDSIGHT — Now that you've seen both the plan and the full implementation:
   - Knowing what you know now, what would you have done differently?
   - Are there simpler approaches that only became obvious after seeing the code?
   - Could a different architecture have prevented issues you found?
   - What would you change if you were writing this plan from scratch today?

These are the known review dimensions. There may be issues we haven't anticipated — use your judgment to find problems beyond this list. Your value is finding what we missed, not just confirming what we already checked.

For every issue found, propose a concrete fix with file paths and pseudo-code.
Classify as BLOCKER (runtime break, data loss, or wrong solution) or ADVISORY (style, naming, optional).
Only BLOCKERs produce a NO-GO verdict.
Hindsight observations go in a separate HINDSIGHT section. Most hindsight informs future work — but if the architecture is fundamentally wrong and should be redone, that IS a BLOCKER.

Verdict: GO or NO-GO with fixes." 2>&1 | tee tmp/reviews/<name>.review.log
```

---

## Step 2b: Run Tests

Run `npm test` (or equivalent) in affected workspaces. If tests fail, fix and re-run. If tests pass, proceed — but passing tests alone don't mean the implementation is correct. The Codex review from Step 2 is the real verification.

---

## Step 3: Fix Gaps with Codex

For any [MISSING], [WRONG], or intention-misaligned items from the review:

```bash
codex -s danger-full-access -a never exec "[GOALS-ONLY] Fix gaps from the implementation review.

Review findings at tmp/reviews/<name>.review.log.
Plan at docs/plans/YYYY-MM-DD-feature.md.

INVARIANTS (from .claude/rules/):
[paste relevant rules content here]

Fix the identified issues. The plan is a guide — use your judgment if the plan's approach won't work for a specific fix. Flag any further deviations.

Include test fixes in this same pass — they must match the actual implementation.

After fixing, check whether your fixes make any existing code redundant. Remove redundant code.

Do NOT send changes to any remote repository. Commit locally only." 2>&1 | tee tmp/reviews/<name>.fix.log
```

**Re-run tests after every fix** — never assume a fix worked.

---

## Step 4: Final Review — GO/NO-GO

One last pass to verify completeness, correctness, AND intention:

```bash
codex -s read-only -a never exec "Final review of the implementation. Original user requirements and amendments at tmp/reviews/<name>.requirements.md; plan at docs/plans/YYYY-MM-DD-feature.md.

Verify:
1. ALL plan items are implemented (check every acceptance criterion)
2. Code compiles and all tests pass
3. No bugs, race conditions, or security issues
4. The plan and implementation satisfy the original requirements and amendments; author-added exclusions do not silently narrow them
5. The architecture is right for the problem — not just technically correct
6. Any plan deviations were improvements, not regressions
7. Tests are testing the right things (not just passing)

After verification, provide a HINDSIGHT section:
- Knowing what you know now from reviewing everything, what would you have done differently?
- Any architectural insights that only became clear after seeing the full implementation?
- Recommendations for future work in this area?

Classify findings as BLOCKER or ADVISORY.
Hindsight observations go in a separate section — most inform future work, but if the architecture is fundamentally wrong and should be redone, that IS a BLOCKER.
Give a GO / NO-GO verdict with reasoning." 2>&1 | tee tmp/reviews/<name>.final.log
```

---

## Anti-Patterns (NEVER Do These)

| Wrong | Right |
|-------|-------|
| Tell Codex which files to change | Give Codex the goal, point to the plan |
| Summarize your analysis in the Codex prompt | Point to files, let Codex read them |
| Prescribe step-by-step fixes | Describe the desired outcome |
| Split the plan into multiple Codex batches | ONE Codex invocation for the entire plan, regardless of internal phase structure |
| Dispatch "Phase N" of a plan as a separate Codex run | One run for the whole plan; phases are organizing structure, implementing agent decides commit structure |
| Write tests in a separate agent from implementation | Tests go in the SAME Codex invocation |
| Run a code review after each phase | One comprehensive review at the end of the full implementation |
| Assume failing tests mean implementation is wrong | Investigate both sides — tests can be wrong too |
| Review only code correctness | Review intention too — does it solve the RIGHT problem? |
| Skip the review because "it's simple" | Every implementation gets reviewed |
| Fix where the error happens instead of preventing it | Think strategically — prevent, don't patch |
| Implement the plan blindly when it's wrong | Fix issues in code, flag deviations |
| Swallow plan deviations silently | Document every deviation with reasoning |
| Ship without running tests | All tests must pass before shipping |
| Leave redundant code after a fix | Check for and remove superseded code paths |

---

## The Trap You Fall Into

> "I was thinking 'fix where the error happens' instead of 'prevent unnecessary work entirely.' Classic case of tactical fixing vs strategic thinking."

Before implementing any fix or feature, ask: **Am I solving the root cause, or am I patching a symptom?** If the plan says to add a guard at point X, but the real issue is that point X shouldn't be reached at all, fix the architecture — don't add a band-aid.
