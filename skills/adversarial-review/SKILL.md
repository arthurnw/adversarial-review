---
name: adversarial-review
description: >
  Multi-agent adversarial code review. Only invoked manually via /adversarial-review
  (Claude Code, pi prompt template) or /skill:adversarial-review (pi). Do NOT invoke
  this skill automatically — wait for the user to call it explicitly. Dispatches three
  parallel reviewer agents (quality, modularity, security), then three adversarial
  challengers, then synthesizes findings. Accepts optional base ref and
  --no-migration-cost flag.
---

# Adversarial Code Review

Six-agent review process: three specialized reviewers run in parallel, then three
adversarial challengers pressure-test their findings. The result is a curated,
high-signal list of recommendations presented one at a time for your reaction.

## Arguments

- `[base-ref]` — Git ref to diff against (default: `origin/main`). Use this for stacked
  branches, e.g. `/adversarial-review origin/feature-base`.
- `--no-migration-cost` — Tell all reviewers to ignore backwards compatibility, migration
  cost, and "is this worth the diff?" concerns. Useful for greenfield work, pre-production
  code, or large refactors where migration cost is already sunk.

## Workflow

### 0. Pick the dispatch adapter

This skill is harness-agnostic. The actual subagent-launch calls live in a sibling
reference file — load the right one now, before proceeding:

- If a `subagent` tool is available (pi with `pi-subagents` installed), read
  [`dispatch-pi.md`](dispatch-pi.md) and follow its dispatch patterns. The preflight
  check in that file runs **before** step 1 — it may tell you to bail out early if
  the extension failed to load.
- Otherwise (Claude Code), read [`dispatch-claude-code.md`](dispatch-claude-code.md)
  and follow its `Task`-tool dispatch patterns.

If the environment is ambiguous, ask the user once before continuing.

### 1. Get the diff

```bash
BASE_REF="${arg_or_default:-origin/main}"
git diff "$BASE_REF"..HEAD --stat
git diff "$BASE_REF"..HEAD
```

Read the diff stat and full diff. This is the material every reviewer and challenger
will work from.

### 2. Auto-detect greenfield / large-refactor

Check the diff stat for signals that migration-cost concerns are likely irrelevant:

- More than 60% of changed files are newly created
- Net line additions exceed 500 with few modifications to existing files
- New top-level directories or packages introduced

If these signals are present AND `--no-migration-cost` was not explicitly passed, ask
the user:

> "This looks like greenfield or a large refactor — want me to tell reviewers to ignore
> migration cost and backwards compatibility? (y/n)"

If they confirm, treat it as if `--no-migration-cost` was passed.

### 3. Dispatch three reviewer agents in parallel

Use the adapter's parallel-dispatch pattern. Each reviewer gets the full diff (via
the commands above), one of the three role mandates below, and — when active — the
migration-cost-exclusion block appended verbatim.

**Reviewer prompt template (assembled by the adapter):**

```
You are reviewing code changes on a feature branch.

## Your focus: {DOMAIN}

{DOMAIN_INSTRUCTIONS}

## Diff to review

Run these commands to get the diff:
  git diff {BASE_REF}..HEAD --stat
  git diff {BASE_REF}..HEAD

## Output format

Return a structured list of findings. For each finding:
- **ID**: Sequential number (R1, R2, R3...)
- **Severity**: Critical / Important / Minor
- **File:line**: Specific location
- **Finding**: What you found
- **Why it matters**: The actual risk or cost
- **Suggested fix**: Concrete action (not vague advice)

End with a one-line overall assessment.

{MIGRATION_COST_EXCLUSION_IF_ACTIVE}
```

#### Reviewer A — Quality & Standards

Focus: general code quality, design patterns, error handling, test coverage, edge
cases, performance implications, API design.

Look for: bugs, logic errors, missing error handling, untested paths, bad
abstractions, naming that misleads, over-engineering, under-engineering.

#### Reviewer B — Cleanliness & Modularity

Focus: code organization, single responsibility, DRY, coupling, cohesion,
readability, naming consistency, function/module boundaries.

Look for: god functions, tangled dependencies, copy-paste duplication, inconsistent
patterns within the changeset, abstractions at the wrong level, files that do too
many things.

#### Reviewer C — Security

Focus: input validation, injection (SQL, command, XSS), auth/authz, secrets
exposure, dependency risks, unsafe deserialization, OWASP top 10, data exposure.

Look for: user input flowing to dangerous sinks without sanitization, hardcoded
credentials, overly permissive permissions, missing auth checks, information
leakage in error messages.

#### Migration-cost-exclusion block (verbatim, when `--no-migration-cost` is active)

```
## Ignore migration cost

Do NOT factor in backwards compatibility, migration cost, or "is this change worth
the git diff?" You should evaluate the code purely on its own merits — as if it
were being written from scratch. Do not penalize large diffs, do not suggest
keeping old approaches for compatibility, do not recommend incremental changes to
reduce risk. Judge the end state, not the transition cost.
```

### 4. Dispatch three challenger agents in parallel

Once all three reviewers complete, dispatch three challengers — one per reviewer.
Each gets that reviewer's findings AND the diff. Use the adapter's dispatch pattern
again.

**Challenger prompt template:**

```
You are an adversarial challenger reviewing another agent's code review findings.

Your job is twofold:
1. **Filter noise**: Challenge each finding. Is it valid? Is the severity appropriate,
   or is a nitpick masquerading as Important? Would a senior engineer actually care
   about this, or is it pedantic? If a finding is weak, downgrade or remove it.
2. **Expand coverage**: Read the diff yourself. Did the reviewer miss anything in their
   domain? Add new findings the reviewer overlooked.

Be rigorous but not contrarian — if a finding is solid, say so and move on. Your
goal is signal-to-noise ratio, not disagreement for its own sake.

## Original reviewer findings

{REVIEWER_OUTPUT}

## Diff

Run these commands:
  git diff {BASE_REF}..HEAD --stat
  git diff {BASE_REF}..HEAD

## Output format

For each original finding, return:
- **Original ID**: (e.g. R1)
- **Verdict**: Upheld / Downgraded / Removed
- **Reason**: Why you upheld, downgraded, or removed it
- If downgraded, the new severity

Then list any NEW findings the reviewer missed, using the original reviewer's format
(IDs C1, C2... to distinguish challenger additions).

End with a one-line assessment of the original review's quality.

{MIGRATION_COST_EXCLUSION_IF_ACTIVE}
```

### 5. Synthesize

After all challengers complete:

1. **Collect surviving findings** — anything Upheld or Downgraded (at new severity),
   plus challenger additions (C-prefixed IDs).
2. **Deduplicate** — if multiple reviewers flagged the same issue, merge into one
   entry noting it was caught by multiple reviewers (this is a signal of importance).
3. **Rank** — Critical first, then Important, then Minor. Within a severity, put
   multi-reviewer findings first.
4. **Group by theme** if natural clusters emerge (e.g. "error handling", "test gaps",
   "naming").

### 6. Present

**First**, show the full synthesized list as a summary table:

```
## Review Summary

X findings survived adversarial challenge (Y critical, Z important, W minor)

| # | Severity | Domain | File:line | Finding |
|---|----------|--------|-----------|---------|
| 1 | Critical | Security | auth.py:42 | SQL injection via unsanitized input |
| 2 | Important | Quality | api.py:88 | Missing error handling on network call |
| ... | ... | ... | ... | ... |
```

**Then**, walk through each finding one at a time:
- Show the full finding with context, reasoning, and suggested fix.
- Wait for the user's reaction before moving to the next one.
- Track decisions (accept, reject, defer) but don't require a formal response —
  "next", "skip", "good point", or "disagree" is fine.

If the user wants to stop early ("that's enough", "looks good", "just the
criticals"), respect that and summarize remaining items briefly.

## Notes

- This skill complements but does not replace single-agent review against a plan.
  This skill is for thorough adversarial review of the code itself.
- Reviewers and challengers are independent agents with no shared context. Each gets
  the diff fresh. This is intentional — independent perspectives catch different things.
- The adversarial step typically filters out 30–50% of initial findings. This is
  the point. Better to surface 8 solid findings than 20 mixed with noise.
