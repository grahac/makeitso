---
name: makeitso-resume
description: |
  Pick up an interrupted makeitso run. Re-enters the autonomous flow at
  the right point: re-run failed task, answer pending question, continue
  from where the last session left off. Invoked by /resume.
allowed-tools:
  - Read
  - Bash
  - Glob
---

# makeitso resume

Continue an in-flight run that was interrupted (compaction, crash, manual stop, or just "I'll come back to this tomorrow").

## Procedure

1. Confirm we're in a makeitso project:

   ```bash
   test -f .planning/config.json && echo "yes" || echo "no"
   ```

   - **No:** tell the user "There's nothing to resume here. Use `/start` to begin." Stop.

2. Determine which phase to resume:

   - If the user passed an argument (e.g. `/resume 2` or `/resume tip-splitter`), use it.
   - If not, find the most recently modified phase directory:

     ```bash
     ls -dt .planning/*/ 2>/dev/null | grep -v -E '/(intel|research|inbox|advisor|users)/$' | head -1
     ```

   - If still ambiguous (multiple recent phases, no argument), ask the user in product language: "There are a few stories in flight — which one do you want to keep working on?" and list them with phase numbers and product-language descriptions.

3. Read the phase's state to figure out where to re-enter:

   - **Pending user question** — look for `BLOCKERS.md`, `pending-questions.md`, or any file with an unresolved question marker. Apply the triage skill (`@${CLAUDE_PLUGIN_ROOT}/skills/triage-product-vs-technical/SKILL.md`). If product, surface in plain English. If technical, resolve it yourself, log to `DECISIONS.md`, and continue.
   - **Failed task** — look for `errors.log`, failed checkpoints, or `gsd-debug` artifacts. If the failure looks recoverable (e.g., tests failed, retry plausible), run `/gsd-debug` or re-invoke `/gsd-autonomous` for that phase. If not, surface to the user: "Something failed I can't recover from on my own — [plain summary]. Want me to take a different approach, or stop?"
   - **No checkpoint, just stopped** — re-invoke `/gsd-autonomous {phase-number}` to pick up.

4. Before resuming, **summarize where we are** to the user in two sentences:

   > "Picking up [phase name in product language]. Last thing that happened: [one-sentence plain summary]. Resuming now."

5. Then proceed exactly as the makeitso-flow skill would from the relevant step:
   - Apply product-only-interview discipline if any user input is needed.
   - Apply triage on every potential pause.
   - When done, summarize in product language per the makeitso-flow's "Step 8 — Report" pattern.

## Anti-patterns

- Do not re-run the product-only interview unless the user explicitly says "let's start over" or the pending question requires re-scoping.
- Do not show the user raw GSD state files. If you need to ask a clarifying question, frame it in product terms.
- Do not silently choose a phase when there's genuine ambiguity — ask.
- Do not assume `/resume` always means "continue immediately." If the last action was a failure, surface the failure first; let the user choose to retry, change tack, or stop.

## When the project is brand-new and `/resume` was called by mistake

If only `.planning/config.json` exists with no phase directories, tell the user: "There's nothing to resume yet — only setup is done. Use `/start` to describe what you want to build." Do not try to auto-create a phase.
