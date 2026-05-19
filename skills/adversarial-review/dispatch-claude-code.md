# Dispatch adapter: Claude Code

Use this adapter when you're running inside Claude Code. The parent agent has the
`Task` tool available and can launch multiple Task calls in a single assistant turn
for true parallel execution.

## Step 3 dispatch — reviewers in parallel

In ONE assistant turn, emit three `Task` tool calls. Use `general-purpose` as the
subagent type unless the user has installed domain-specific subagents. Pass each
reviewer the assembled prompt from SKILL.md §3, substituting `{BASE_REF}`,
`{DOMAIN}`, `{DOMAIN_INSTRUCTIONS}`, and (if active) the migration-cost-exclusion
block verbatim.

```
Task(
  subagent_type="general-purpose",
  description="Adversarial review: quality & standards",
  prompt="<assembled Reviewer A prompt>"
)
Task(
  subagent_type="general-purpose",
  description="Adversarial review: cleanliness & modularity",
  prompt="<assembled Reviewer B prompt>"
)
Task(
  subagent_type="general-purpose",
  description="Adversarial review: security",
  prompt="<assembled Reviewer C prompt>"
)
```

Do NOT serialize these into separate turns. Emit all three in one turn so they run
concurrently. Wait for all three to complete before moving to step 4.

Capture each reviewer's full output verbatim, labelled clearly:

```
## Reviewer A — Quality & Standards

{full output}

## Reviewer B — Cleanliness & Modularity

{full output}

## Reviewer C — Security

{full output}
```

## Step 4 dispatch — challengers in parallel

Once all three reviewer outputs are in hand, emit three more `Task` calls in a
single turn. Each challenger gets exactly one reviewer's output substituted into
`{REVIEWER_OUTPUT}` in the challenger prompt template from SKILL.md §4. Do not
let challenger A see reviewer B's output, and so on.

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

Wait for all three to complete before moving to step 5 (synthesis).

## Notes specific to Claude Code

- **Parallel is mandatory.** The three reviewer Task calls must appear in a single
  assistant turn. Same for the three challenger calls. Serializing them into separate
  turns defeats the independent-perspectives design.
- **Subagent type.** Use `general-purpose` unless the user has provisioned domain
  subagents (e.g. a dedicated `security-reviewer`). If uncertain, stick with
  `general-purpose`; do not probe `/agents` unless the user explicitly mentions
  custom subagents.
- **Tool access.** Task subagents inherit standard tool access; `git diff` calls
  inside the subagent work without extra wiring.
- **Migration-cost block.** When `--no-migration-cost` is active, append the
  exclusion block from SKILL.md §3 to each reviewer prompt AND each challenger
  prompt. Keep the text byte-identical to SKILL.md.
- **Do not name the Tool.** The reviewer/challenger prompts in SKILL.md are
  tool-agnostic; do not mention "Task tool" or "Claude Code" inside those prompts.
