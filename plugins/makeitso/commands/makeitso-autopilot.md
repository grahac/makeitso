---
description: "Drive a single phase to completion: execute the phase's plans, run /makeitso-review, fix HIGH-severity findings, re-review. Capped at 3 fix iterations. Invoked by makeitso-flow; can be run directly."
disable-model-invocation: false
---

# /makeitso-autopilot

You are driving one phase of work through makeitso's autonomous loop:

```
execute  →  review  →  (if HIGH findings)  fix  →  re-review  →  …  →  done
```

You handle one phase only. The caller (typically makeitso-flow) is responsible for sequencing across phases and honoring `confirm_transition` gates.

## Step 1 — Identify the phase

If the user passed a phase number as an argument (`/makeitso-autopilot 2`), use it. Otherwise read `.planning/STATE.md` and use the current in-flight phase. If neither is available, stop and tell the user: "I don't know which phase to run. Tell me a phase number, or run `/gsd-progress` first to set up state."

Record:

- `PHASE_NUMBER` — e.g. `2`
- `PHASE_NAME` — from STATE.md if available
- `BASE_REF` — the merge-base with the repo's default branch, captured **before** any execution work happens. Reviews always diff against this same base.

```bash
BASE_REF=$(git merge-base HEAD "$(git symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null | sed 's@^origin/@@' || echo main)")
```

## Step 2 — Execute the phase

Invoke GSD's phase-execution command:

```
/gsd-execute-phase ${PHASE_NUMBER}
```

That runs the plans in this phase (in parallel waves, per GSD's design). Wait for it to complete. If it fails or aborts, stop the autopilot and report the failure plainly — do not loop on execution failures.

## Step 3 — Review the work

Invoke `/makeitso-review ${BASE_REF}`. This fans out six reviewer subagents and produces a markdown report ending with a `RECOMMENDATION:` line.

Parse the final recommendation:

- `RECOMMENDATION: ship` → phase is done. Go to Step 6.
- `RECOMMENDATION: fix-then-ship` → there are HIGH findings with clear fixes. Go to Step 4.
- `RECOMMENDATION: rework` → the approach itself needs rethinking. Stop the autopilot, surface the report to the user, and return without advancing. Don't try to fix-loop your way out of a rework verdict — that's a human decision.

If the report has no recognizable `RECOMMENDATION:` line, treat it as `ship` if there are zero HIGH findings, otherwise as `fix-then-ship`.

## Step 4 — Fix HIGH-severity findings

Spawn the `mi-fixer` subagent with:

- The full markdown review report from Step 3
- The `BASE_REF`
- The phase number and name
- Instruction: HIGH-severity findings only, no scope creep into MEDIUM/LOW, no opportunistic refactoring

Wait for it to return its summary of fixes applied. If `mi-fixer` reports that it couldn't apply any fixes (e.g. findings were too vague, or the suggested fix conflicts with other code it couldn't reconcile), stop the autopilot and surface that to the user — looping won't help.

## Step 5 — Re-review and iterate

Go back to Step 3 and run `/makeitso-review ${BASE_REF}` again. Track the iteration count.

**Iteration cap: 3.** That means: execute → review (1) → fix → review (2) → fix → review (3) → fix → review (4 = cap hit).

If the cap is hit and review still says `fix-then-ship`, stop the autopilot. Tell the user, in plain English:

> "I tried three rounds of fixes on phase ${PHASE_NUMBER} but the review still flags issues. The latest report is above. I think a human needs to look at this before we keep going."

Surface the latest review report. Do not advance to "phase done".

## Step 6 — Phase done

Report in plain English:

> "Phase ${PHASE_NUMBER} (${PHASE_NAME}) is done. [N] review rounds. [M] fixes applied along the way."

Then stop. The caller decides whether to move on to the next phase — that's where `gates.confirm_transition` from `.planning/config.json` applies, and it's the caller's job to honor it.

## Operating rules

1. **One phase only.** Do not advance to the next phase yourself. The caller sequences phases.
2. **Don't fix-loop a `rework` verdict.** That's a human call.
3. **HIGH only.** mi-fixer is instructed to ignore MEDIUM and LOW. Don't second-guess that.
4. **Cap at 3 fix rounds.** If you're still failing, surface to human — looping more wastes tokens and rarely converges.
5. **Plain English when you talk to the user.** Match makeitso-flow's voice: non-developers may be reading.
6. **Don't narrate every step.** Brief progress lines at major transitions ("Executing phase 2…", "Review found 4 high-priority items — fixing…", "Re-reviewing…") are enough.
