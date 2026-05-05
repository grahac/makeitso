# makeitso

Three doors. One flow. The system asks what you want, in plain English. Then builds it.

```
/start
/plan
/discuss
```

Any of those will trigger the same autopilot. The system asks 3–5 product questions ("who's it for?", "what should happen when X?", "what's success?"), then runs the discuss → plan → execute → review loop autonomously, only stopping when something a non-developer must decide actually surfaces.

It does **not** ask you about architecture, frameworks, libraries, schemas, file layout, retry counts, or any other technical detail. It makes those decisions itself and writes them to a decisions log you can audit later.

## Commands

| Command | What it does |
|---|---|
| `/start` | Begin a new makeitso run. (Aliases: `/plan`, `/discuss` — same thing.) |
| `/status` | Plain-English summary of where each in-flight story is, decisions made on your behalf, and what's waiting on you. |
| `/resume` | Pick up an interrupted run. Re-enters the autopilot at the right point. |

## How it works

`makeitso` is a thin wrapper over [GSD](https://github.com/dsifry/get-shit-done). It adds:

1. **`product-only-interview`** skill — applied during the discussion phase so questions are always framed in user-visible terms, never technical ones.
2. **`triage-product-vs-technical`** skill — injected into GSD subagents (`gsd-verifier`, `gsd-plan-checker`, `gsd-integration-checker`, `gsd-executor`, `gsd-nyquist-auditor`, `gsd-assumptions-analyzer`). Before any pause-for-user, the agent classifies the pending question. Product questions surface in plain English. Technical questions are auto-resolved and logged to `.planning/<phase>/DECISIONS.md`.
3. **A default `.planning/config.json` template** with the right `agent_skills` injection, `code_review_command: "/ce-code-review"` (routes review to the compound-engineering reviewer spectrum), and sensible parallelization defaults.
4. **A default permissions allowlist** so non-dev users don't get prompted constantly during autonomous runs.

## Install

This plugin requires [GSD](https://github.com/dsifry/get-shit-done) to be installed.

```
/plugin marketplace add second-coffee/makeitso
/plugin install makeitso@makeitso-marketplace
```

## Usage

In any project directory:

```
/start
```

It will:
1. Set up `.planning/config.json` if missing
2. Ask what you want built (one prompt, plain English)
3. Ask 3–5 follow-up questions, **all in product language**
4. Hand to GSD's autonomous loop with the right discipline applied
5. Surface only product-level questions during the run
6. Notify you when the work is ready

To see where things are:

```
/status
```

To pick up an interrupted run:

```
/resume          # picks up the most recent in-flight story
/resume 2        # specific phase by number
/resume payments # specific phase by name
```

## Status

**v0.1 — alpha.** The triage classifier's accuracy is the primary unknown. Expect prompt iteration over the first few uses.
