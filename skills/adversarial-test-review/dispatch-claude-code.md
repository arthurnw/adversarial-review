# Dispatch adapter: Claude Code

Use this adapter when you're running inside Claude Code. The parent agent has the
`Task` tool available and can launch multiple Task calls in a single assistant turn
for true parallel execution.

## Step 2 dispatch — reviewers in parallel

In ONE assistant turn, emit three `Task` tool calls. Use `general-purpose` as the
subagent type unless the user has installed domain-specific subagents. Pass each
reviewer the assembled prompt from SKILL.md §2, substituting `{LENS}`,
`{LENS_INSTRUCTIONS}`, `{SCOPE_INSTRUCTIONS}`, and `{TAXONOMY}` (the shared block
goes into every reviewer).

```
Task(
  subagent_type="general-purpose",
  description="Test review: validity",
  prompt="<assembled Reviewer A prompt — Validity>"
)
Task(
  subagent_type="general-purpose",
  description="Test review: behavior vs implementation",
  prompt="<assembled Reviewer B prompt — Behavior vs. implementation>"
)
Task(
  subagent_type="general-purpose",
  description="Test review: robustness & coverage",
  prompt="<assembled Reviewer C prompt — Robustness & coverage>"
)
```

Do NOT serialize these into separate turns. Emit all three in one turn so they run
concurrently. Wait for all three to complete before moving to step 3.

Capture each reviewer's full output verbatim, labelled clearly:

```
## Reviewer A — Validity

{full output}

## Reviewer B — Behavior vs. implementation

{full output}

## Reviewer C — Robustness & coverage

{full output}
```

## Step 3 dispatch — challengers in parallel

Once all three reviewer outputs are in hand, emit three more `Task` calls in a
single turn. Each challenger gets exactly one reviewer's output substituted into
`{REVIEWER_OUTPUT}` in the challenger prompt template from SKILL.md §3, plus the
same `{SCOPE_INSTRUCTIONS}`. Do not let challenger A see reviewer B's output, and
so on.

```
Task(
  subagent_type="general-purpose",
  description="Challenge reviewer A",
  prompt="<assembled Challenger prompt — Reviewer A output substituted>"
)
Task(
  subagent_type="general-purpose",
  description="Challenge reviewer B",
  prompt="<assembled Challenger prompt — Reviewer B output substituted>"
)
Task(
  subagent_type="general-purpose",
  description="Challenge reviewer C",
  prompt="<assembled Challenger prompt — Reviewer C output substituted>"
)
```

Wait for all three to complete before moving to step 4 (synthesis).

## Notes specific to Claude Code

- **Parallel is mandatory.** The three reviewer Task calls must appear in a single
  assistant turn. Same for the three challenger calls. Serializing them into separate
  turns defeats the independent-perspectives design.
- **Subagent type.** Use `general-purpose` unless the user has provisioned domain
  subagents (e.g. a dedicated test/QA reviewer). If uncertain, stick with
  `general-purpose`; do not probe `/agents` unless the user explicitly mentions
  custom subagents.
- **Tool access.** Task subagents inherit standard tool access; the `git` and file
  reads the reviewers need (test files AND the production code under test) work
  without extra wiring.
- **Scope, not base ref.** This skill resolves a test scope in SKILL.md §1 (staged
  tests, else latest test commit, else explicit ref/range/path). Pass the resolved
  scope into `{SCOPE_INSTRUCTIONS}` — name the exact files and the git command to
  read their content/diff — so each fresh agent can find both the tests and the code
  they exercise.
- **Do not name the tool.** The reviewer/challenger prompts in SKILL.md are
  tool-agnostic; do not mention "Task tool" or "Claude Code" inside those prompts.
