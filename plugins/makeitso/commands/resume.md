---
description: "Pick up an interrupted makeitso run. Re-enters the autopilot at the right point — answers a pending question, retries a failed step, or continues from where you left off."
disable-model-invocation: true
---

Invoke the makeitso-resume skill at @${CLAUDE_PLUGIN_ROOT}/skills/makeitso-resume/SKILL.md and follow it exactly.

If the user provided a phase identifier after the slash command (e.g. `/resume 2` or `/resume tip-splitter`), pass it through to the skill.
