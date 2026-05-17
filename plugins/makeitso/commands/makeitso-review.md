---
description: "Multi-perspective code review of the current branch. Runs five reviewer subagents in parallel (correctness, testing, maintainability, simplicity, security) and merges their findings into a single report grouped by severity."
disable-model-invocation: true
---

# /makeitso-review

You are running makeitso's bundled code review. This is the default reviewer invoked by GSD's `code_review_command`. It's lightweight on purpose — five focused reviewer subagents in parallel, no PR-platform integration, no auto-fix. Power users who want a richer review can point `code_review_command` at `/ce-code-review` (compound-engineering) instead.

## Step 1 — Detect scope

Figure out what to review. In order of preference:

1. If the user passed a base ref as an argument (e.g. `/makeitso-review main`), diff against that.
2. Otherwise, determine the merge-base with the repo's default branch:

   ```bash
   git symbolic-ref refs/remotes/origin/HEAD --short 2>/dev/null | sed 's@^origin/@@' || echo "main"
   ```

   Use that as the base and diff `HEAD` against it.

3. If the working tree has uncommitted changes, include them in the review (treat them as additions on top of HEAD).

Capture two things for the reviewers:

- **Base ref** — what you're diffing against.
- **Changed files** — `git diff --name-only <base>...HEAD` plus `git status --porcelain` for uncommitted work.

If the file list is empty, say: "Nothing to review — no changes between this branch and the base." Stop.

## Step 2 — Run reviewers in parallel

Spawn all five reviewer subagents in a single message using the Task tool. Each call uses `subagent_type` set to the reviewer's name, and a prompt that gives:

- The base ref
- The list of changed files
- Instructions to read the files at HEAD and use `git diff <base>...HEAD -- <file>` to see what changed
- A reminder to return findings in the format specified by their agent definition

Reviewers to invoke:

- `mi-correctness-reviewer`
- `mi-testing-reviewer`
- `mi-maintainability-reviewer`
- `mi-simplicity-reviewer`
- `mi-security-reviewer`

Suggested prompt template for each subagent:

> Review the diff between `<base>` and `HEAD` in this repository. Changed files:
>
> ```
> <file list>
> ```
>
> Use `git diff <base>...HEAD -- <file>` to see each change, read files at HEAD with the Read tool when you need surrounding context, and follow the output format in your agent definition exactly. Be specific — every finding must cite file and line.

Run all five **in parallel** (one message, five Task tool calls). Wait for all to return.

## Step 3 — Merge findings

Collect each reviewer's markdown output. Build a single report with this structure:

```
# Code review

**Base:** <base ref>
**Files reviewed:** <count> file(s)
**Reviewers:** correctness, testing, maintainability, simplicity, security

## Summary

<one paragraph in plain English: how many high/medium findings across reviewers, the dominant theme if any (e.g. "mostly missing tests"), and an overall recommendation (ship / fix-then-ship / rework)>

## High-priority findings

<every HIGH finding from any reviewer, deduplicated. If two reviewers flagged the same line, merge them into one entry citing both perspectives.>

## Medium-priority findings

<every MEDIUM finding, deduplicated>

## Notes from individual reviewers

<paste each reviewer's full markdown output verbatim under a `### <reviewer name>` heading, so a curious reader can see the raw analysis>
```

When merging, dedupe by `file:line + finding class`. If two reviewers disagree about severity, take the higher one and note both reviewers.

## Step 4 — Plain-English summary for non-dev users

The makeitso audience is often non-technical. At the very top of the report (above the `# Code review` heading), include a 2–3 sentence plain-English summary:

- "I checked the new work from five angles. Found [N] things worth your attention — [M] high priority, [K] medium."
- If everything is clean: "I checked the new work from five angles and didn't find anything blocking. It looks ready to use."
- If there are HIGH findings: end with a one-line recommendation ("I'd fix the high-priority ones before considering this done.").

## Step 5 — Exit code semantics

If invoked from GSD's autonomous loop, GSD will read the report for go/no-go signal. Make the recommendation explicit at the end of the report on its own line:

- `RECOMMENDATION: ship` — no HIGH findings.
- `RECOMMENDATION: fix-then-ship` — HIGH findings exist but all have clear fixes.
- `RECOMMENDATION: rework` — multiple HIGH findings in correctness or security indicating the approach needs rethinking.

That's the whole review.
