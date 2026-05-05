---
name: makeitso-status
description: |
  Product-language status check for an in-flight makeitso run.
  Shows where each story is in the pipeline using user-visible terms,
  not engineering terms. Invoked by /status.
allowed-tools:
  - Read
  - Bash
  - Glob
---

# makeitso status

Report what's happening in the current project, in plain English the user can read.

## Procedure

1. Look for `.planning/` in the current working directory:

   ```bash
   test -d .planning && echo "exists" || echo "missing"
   ```

   - **Missing:** tell the user "There's no makeitso work going on in this directory. Use `/start` to begin." Stop.
   - **Exists:** continue.

2. Discover phases:

   ```bash
   ls -d .planning/*/ 2>/dev/null | grep -v -E '/(intel|research|inbox|advisor|users)/$' | sort
   ```

3. For each phase directory, read these files if they exist (do not error if they don't):

   - `CONTEXT.md` — what the user said they wanted, in product terms
   - `ROADMAP.md` (top-level, not per-phase) — overall plan
   - `PLAN.md` — the technical plan (you'll *read* this but not show it raw)
   - `VERIFICATION.md` — whether the goal is met
   - `STATUS.md` or any `*-status.md` — explicit status if GSD wrote one
   - `DECISIONS.md` — auto-resolved technical decisions made on the user's behalf

4. Summarize each phase in **two parts**:

   **What this is (product language):**
   One sentence drawn from CONTEXT.md or the phase title — what does this *do* for the user?

   **Where it is right now:**
   Pick the highest-confidence label that matches what the files show:
   - `Just describing it` — CONTEXT.md exists but no PLAN.md
   - `Figuring out how` — PLAN.md is being drafted
   - `Building` — PLAN.md exists, code is being written, no VERIFICATION yet
   - `Checking it works` — code written, tests/review running
   - `Ready` — VERIFICATION.md says goal met
   - `Stuck — needs your input` — there's a pending question or escalation
   - `Stuck — something failed` — repeated failures with no clear path forward

   If you genuinely cannot tell, say "Status unclear — check `.planning/<phase>/` directly."

5. Surface decisions count and any pending questions:

   - Read all `DECISIONS.md` files and count entries — report as "X decisions made on your behalf so far."
   - If `needs_review: true` appears anywhere, list those entries (one bullet each) under "Decisions worth a quick look."
   - If any phase has a `BLOCKERS.md` or pending-question file, list it under "Waiting on you for:" — in plain English, with the product reframe applied.

6. Present in this shape:

   ```
   Project: <directory name>

   Stories in flight:
     1. <Phase name in product language> — <status label>
     2. ...

   Decisions made on your behalf: <N>
   Worth a quick look: <count>, with the highest-impact one shown
   Waiting on you for: <list, or "nothing">

   Want to dig in? Say `/resume <phase number>` to keep going,
   or ask me to explain any decision in plain English.
   ```

## Anti-patterns

- Do not list raw filenames or paths beyond what's needed for navigation.
- Do not surface internal status names (e.g., "phase active", "verification pending") unless they're the only useful signal.
- Do not summarize code changes — the user wants product status, not changelog.
- Do not invent confidence — if a phase is genuinely ambiguous, say so plainly.
