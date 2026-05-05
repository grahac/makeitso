---
name: triage-product-vs-technical
description: |
  Before any GSD pause-for-user, classifies the pending question as
  PRODUCT (must surface to user) or TECHNICAL (auto-resolve and proceed).
  Surfaces only product-level questions in plain English; auto-resolves
  technical questions silently.

  Injected via project's .planning/config.json agent_skills mapping for
  agent types: verifier, plan-checker, integration-checker, executor,
  nyquist-auditor. Not intended for direct user invocation.

  This is the keystone skill for the "non-dev autopilot" pattern: GSD
  drives discuss → plan → execute → review autonomously, and only stops
  when something a non-developer must decide actually surfaces.
allowed-tools:
  - Read
  - Write
  - AskUserQuestion
---

# Triage: Product vs. Technical

This skill is consulted before any GSD agent surfaces a question, blocker, "grey area," or pause to the user. It classifies the pending question and either:
- Surfaces it to the user in plain English (PRODUCT), or
- Resolves it autonomously and writes the decision to a decisions log (TECHNICAL)

## Activation

Add to project's `.planning/config.json`:

```json
{
  "agent_skills": {
    "verifier": ["triage-product-vs-technical"],
    "plan-checker": ["triage-product-vs-technical"],
    "integration-checker": ["triage-product-vs-technical"],
    "executor": ["triage-product-vs-technical"],
    "nyquist-auditor": ["triage-product-vs-technical"]
  }
}
```

## Classification rules

A pending question is **PRODUCT-LEVEL** (surface to user) when answering it changes any of:

1. **What the user sees or experiences** — the visible surface, the wording, the flow, the feedback
2. **What scenarios are supported** — does it work offline, slow network, multiple devices, after a crash
3. **Who the feature is for** — audience changes, persona shifts, accessibility expectations
4. **What outcome counts as success** — KPIs, target metrics, acceptable latency from the user's perspective
5. **Trade-offs between user needs** — speed vs. accuracy, breadth vs. depth, simple vs. powerful
6. **What's explicitly out of scope** — refusing a use case is a product decision

A pending question is **TECHNICAL** (auto-resolve, do not surface) when answering it changes any of:

1. **How something is implemented** — architecture, framework, library, data structure, pattern
2. **Internal naming, file layout, module structure** — engineers' ergonomics
3. **Test strategy** — unit vs. integration, coverage targets, mocking approach
4. **Retry/timeout/concurrency values** — without user-visible latency consequences
5. **Error-handling approach** — how to log, how to retry, what to do on partial failure
6. **Performance trade-offs that are invisible to the user** — caching, indexing, query optimization
7. **Refactoring scope** — does this clean up adjacent code or stay surgical
8. **Anything an experienced engineer would decide without asking a PM**

## Edge cases — when technical IS product

A technical decision becomes product-level when:

- It changes user-visible latency in a meaningful way ("this approach makes the page take 2s vs. 200ms")
- It changes which user scenarios actually work (offline support, slow connections, etc.)
- It has user-facing cost implications (a more expensive infrastructure choice that ends up affecting pricing)
- It prevents a user-stated requirement (the user said "must work offline" and the technical choice precludes it)
- It introduces a privacy/data-handling consideration the user should know about
- It permanently locks in a choice that's hard to reverse later

When in doubt: **classify as PRODUCT.** False positives waste a user moment. False negatives bury a real product question.

## Output: PRODUCT classification

When PRODUCT, surface the question via AskUserQuestion. Reframe as plain English:

- Ban technical vocabulary: schema, endpoint, framework, library, deploy, config, migration, service, API, validation, async, sync, queue, cache, retry
- Frame as scenario: "What should happen when ___?"
- Frame as outcome: "Does this need to also work when ___?"
- Show the user the trade-off in their terms, not engineering terms

Examples:

| Raw agent question | Product reframe |
|---|---|
| "Should we use Stripe or Square?" | If both have been pre-approved by the user → TECHNICAL. If neither has → PRODUCT, frame as: "Do you have a preferred payment provider, or should I pick the one that's fastest to integrate?" |
| "Sync or async checkout confirmation?" | "When the customer clicks Pay, do they need to see confirmation right away, or is a 'processing — we'll email you' screen acceptable?" |
| "Soft delete or hard delete?" | "If an admin deletes a customer record by mistake, should they be able to recover it for a window of time?" |
| "What's the cache TTL?" | TECHNICAL — auto-resolve unless user-visible staleness is in question |

## Output: TECHNICAL classification

When TECHNICAL, do NOT pause. Resolve autonomously and write the decision to `.planning/{phase}/DECISIONS.md`:

```yaml
- decision: "Use Phoenix LiveView for the dashboard rather than React"
  rationale: "Project's existing dashboards use LiveView; convention match; no user-visible difference"
  reversibility: "easy — could swap later if needed"
  confidence: "high"
  surfaced: false
  reason_not_surfaced: "Pure technical — no user-visible consequence"
  classifier_version: "triage-product-vs-technical@1"
```

Continue with the work. The decision is logged for audit, surfaced in batch via the milestone summary, and reviewable by anyone later.

## Output: borderline cases

When you classify but with low confidence (60–80%), still resolve technically but flag in the DECISIONS.md entry:

```yaml
- decision: "Use eventual consistency for profile sync across devices"
  rationale: "Vast majority of users won't notice; matches existing pattern"
  reversibility: "medium — would require schema/replication change"
  confidence: "low"
  surfaced: false
  reason_not_surfaced: "Borderline — likely invisible to users, but flagging for human review"
  needs_review: true
  classifier_version: "triage-product-vs-technical@1"
```

Entries with `needs_review: true` are surfaced in the milestone summary as "decisions worth a human pass."

## Anti-patterns to avoid

1. **Surfacing technical questions in product clothing** — "Are you OK with the data being slightly stale?" is still a technical question. Don't.
2. **Auto-resolving things that change what the user sees** — if the trade-off is between two visible behaviors, that's product, even if engineers think it's technical
3. **Asking the user about defaults** — if the user hasn't specified preferences and the choice is genuinely invisible, just pick the conventional default and move on
4. **Surfacing decisions with low confidence and no user-visible consequence** — log them, don't ask
5. **Bundling multiple unrelated questions into one user pause** — surface one product question at a time

## Procedure

1. Read the agent's pending question / pause reason
2. Classify against the rules above
3. If PRODUCT: reframe in plain English → AskUserQuestion → integrate answer
4. If TECHNICAL: pick the best option using project context (existing patterns, CLAUDE.md, sensible defaults) → write to DECISIONS.md → signal proceed
5. If borderline: TECHNICAL with `needs_review: true` flag

## Calibration

The Triage prompt should be re-tuned by reading recent DECISIONS.md entries periodically. If many `needs_review: true` items turn out to have been product-level in retrospect, the classifier is too aggressive. If users frequently say "you didn't need to ask me that," it's too conservative.

Target: < 1 user pause per phase, except when the user explicitly requested more involvement.
