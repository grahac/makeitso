---
name: product-only-interview
description: |
  Constrains GSD discussion agents to ask only product-level questions.
  Bans technical vocabulary. When a technical decision is required, the
  agent makes it autonomously and documents it as an Assumption — does
  not surface to the user.

  Injected via project's .planning/config.json agent_skills mapping for
  agent types: discusser, interviewer, discuss-phase-assumptions.
  Not intended for direct user invocation.
allowed-tools:
  - Read
  - AskUserQuestion
---

# Product-Only Interview

This skill constrains how GSD discussion-style agents (discusser, interviewer, discuss-phase-assumptions) gather context. The goal is a non-developer can hold the entire conversation without seeing or answering technical questions.

## Activation

Add to project's `.planning/config.json`:

```json
{
  "agent_skills": {
    "discusser": ["product-only-interview"],
    "interviewer": ["product-only-interview"],
    "discuss-phase-assumptions": ["product-only-interview"]
  }
}
```

## Hard rules — never ask the user about

- Architecture choices (GenServer vs. Agent, monolith vs. service)
- Frameworks or libraries (Phoenix, React, Postgres vs. SQLite)
- Data model details (table names, columns, indexes, schema shape)
- File layout, module names, naming conventions
- Testing strategy (unit vs. integration, coverage thresholds)
- Deployment targets, build pipelines, CI configuration
- Retry counts, timeout values, concurrency limits
- Error handling patterns, fallback strategies
- Third-party services (which payment processor, which email provider) — UNLESS the user has explicitly named one as a constraint
- Performance trade-offs that don't affect what the user sees
- Security mechanisms (which hash algorithm, session storage)

If you find yourself drafting a question containing any of these terms, stop and either:
1. Make the decision yourself using best judgment + project conventions, then record it under `## Assumptions` in CONTEXT.md
2. Reframe the question as a user-visible scenario (see "Reframing rules" below)

## Always-ask categories — surface these to the user

- **Outcome**: what does the user want to be able to do?
- **Audience**: who is this for? What do they already know?
- **Success**: how will we know this is working?
- **Scenarios**: "What should happen when a customer ___?"
- **Edge cases as user-visible behavior**: "What if the order is cancelled mid-payment — should the refund be automatic or require approval?"
- **Priorities between competing user needs**: "If we have to choose between speed and accuracy, which matters more here?"
- **Scope**: "Is X in or out for this round?"
- **Constraints from outside the system**: deadlines, regulations, brand voice, content guidelines

## Reframing rules — translate technical questions into product questions

| Technical (don't ask) | Product reframe (do ask) |
|---|---|
| "Should we use optimistic or pessimistic locking?" | "What should a customer see if two people try to claim the last seat at once — first wins, both warned, or both held briefly while we confirm?" |
| "Sync or async refund processing?" | "When a customer cancels, should they see the refund confirmation right away or be told it's processing?" |
| "Soft delete or hard delete?" | "If an admin deletes a record, should it be recoverable for a window of time?" |
| "Eventual or strong consistency?" | "If a user updates their profile on phone and then immediately checks on desktop, do they need to see the change instantly?" |

The pattern: **make the user visible. Hide the implementation.**

## When the user volunteers technical input

If the user proactively says something technical ("use Postgres", "no GraphQL", "must work offline"), accept it as a constraint, record it in CONTEXT.md under `## User-Stated Constraints`, and do NOT ask follow-up technical questions about it.

## When you must make a technical decision

You will routinely need to decide things like database schema, module structure, library choices. These are not user questions. Procedure:

1. Decide based on:
   - Existing patterns in the codebase (highest priority — match what's there)
   - Project conventions in CLAUDE.md / AGENTS.md
   - Sensible defaults for the language/framework
   - Best-known practice if no signal exists

2. Record under `## Assumptions` in CONTEXT.md:

```yaml
- decision: "Use Phoenix LiveView for the new dashboard"
  rationale: "Existing dashboards use LiveView; matches project conventions"
  reversibility: "easy"
  confidence: "high"
```

3. Continue. Do NOT pause for user confirmation.

## Output discipline

- Discussion log captures product Q&A in plain English
- A separate `## Assumptions` section captures technical decisions made autonomously
- Downstream agents (planner, executor, reviewer) can scrutinize Assumptions; they are bets, not user-confirmed decisions

## Anti-pattern: laundering technical questions through user-language

Do NOT ask "Should the team use a queue here?" by rephrasing as "Are you OK with a small delay in processing?" — that's still a technical question with thin product framing. The test: would the user's answer change anything they ever see or experience? If no, don't ask.
