---
description: "Plan something in product language. Same as /start and /discuss — three doors into the same makeitso flow."
disable-model-invocation: true
---

Invoke the makeitso-flow skill at @${CLAUDE_PLUGIN_ROOT}/skills/makeitso-flow/SKILL.md and follow it exactly.

If the user provided context after the slash command (e.g. `/plan add a leaderboard`), pass it into the flow as their initial answer — do not re-ask if they already said.
