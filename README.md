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

`makeitso` is a thin wrapper over [GSD](https://github.com/gsd-build/get-shit-done). It adds:

1. **`product-only-interview`** skill — applied during the discussion phase so questions are always framed in user-visible terms, never technical ones.
2. **`triage-product-vs-technical`** skill — injected into GSD subagents (`gsd-verifier`, `gsd-plan-checker`, `gsd-integration-checker`, `gsd-executor`, `gsd-nyquist-auditor`, `gsd-assumptions-analyzer`). Before any pause-for-user, the agent classifies the pending question. Product questions surface in plain English. Technical questions are auto-resolved and logged to `.planning/<phase>/DECISIONS.md`.
3. **A default `.planning/config.json` template** with the right `agent_skills` injection, `code_review_command: "/makeitso-review"` (routes review to the bundled reviewer spectrum), and sensible parallelization defaults.
4. **`/makeitso-review`** — a bundled code-review orchestrator that fans out to six reviewer subagents in parallel (correctness, testing, maintainability, simplicity, security, performance) and merges findings into a single report. Inspired by [compound-engineering](https://github.com/EveryInc/compound-engineering-plugin)'s reviewer spectrum but trimmed and self-contained, so makeitso has no required runtime plugin dependency beyond GSD.
5. **A default permissions allowlist** so non-dev users don't get prompted constantly during autonomous runs.

## Install

### Prerequisites

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)** — the CLI this plugin runs inside.
- **Node.js 18+** — needed to install GSD via `npx`. Verify with `node --version`. If missing, install from [nodejs.org](https://nodejs.org/) or via `brew install node` on macOS.

makeitso depends on GSD. Install both in order:

### 1. Install GSD

makeitso wraps GSD's autonomous discuss → plan → execute → review loop. From your project directory:

```bash
npx get-shit-done-cc@latest
```

Follow the prompts. Full docs: **https://github.com/gsd-build/get-shit-done**.

### 2. Install makeitso

Inside Claude Code:

```
/plugin marketplace add grahac/makeitso-marketplace
/plugin install makeitso@makeitso-marketplace
```

That's it — the first time you run `/start`, makeitso copies its two helper skills into `~/.claude/skills/` so GSD can find them via the `global:` prefix. Subsequent runs are instant.

### Optional: use compound-engineering for code review instead

makeitso ships with `/makeitso-review` (six reviewer subagents) as the default. If you want the richer compound-engineering reviewer spectrum (~20 reviewers, including stack-specific ones for Rails / Python / TypeScript / Swift, plus adversarial and reliability lenses), install it separately and point the config at it:

```
/plugin marketplace add EveryInc/compound-engineering-plugin
/plugin install compound-engineering@compound-engineering-plugin
```

Then edit `.planning/config.json` and change `code_review_command` from `"/makeitso-review"` to `"/ce-code-review"`.

### Verify the install

In any project directory, run:

```
/start
```

If Claude Code prompts you to install missing GSD commands (like `/gsd-new-project`), complete step 1 above.

## Usage

In any project directory:

```
/start
```

It will:
1. Bootstrap helper skills into `~/.claude/skills/` (first run only)
2. Set up `.planning/config.json` if missing
3. Ask what you want built (one prompt, plain English)
4. Ask 3–5 follow-up questions, **all in product language**
5. Hand to GSD's autonomous loop with the right discipline applied
6. Surface only product-level questions during the run
7. Notify you when the work is ready

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
