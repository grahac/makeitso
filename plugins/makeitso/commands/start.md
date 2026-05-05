---
description: "Start a new makeitso run. Asks what you want built in plain English, then runs discuss → plan → execute → review autonomously, only stopping for product-level questions."
disable-model-invocation: true
---

Invoke the makeitso-flow skill at @${CLAUDE_PLUGIN_ROOT}/skills/makeitso-flow/SKILL.md and follow it exactly.

If the user provided context after the slash command (e.g. `/start build me a tip calculator`), pass it into the flow as their initial answer to "What would you like to build?" — do not re-ask if they already said.
