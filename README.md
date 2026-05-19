# adversarial-review

Multi-agent adversarial code review. Three specialized reviewers (quality,
modularity, security) run in parallel, then three adversarial challengers
pressure-test their findings. The result is a curated, high-signal list of
recommendations presented one at a time.

Works in **Claude Code** (as a plugin) and **pi** (as a package). Single
canonical workflow; dispatch differs per harness.

## Install

### Claude Code

Install from a local checkout:

```bash
claude mcp add --local adversarial-review /path/to/adversarial-review
```

Or add the plugin via Claude Code's plugin settings, pointing at this directory.
After install, `/adversarial-review` appears in autocomplete.

### pi

```bash
pi install npm:adversarial-review
```

Or from a local checkout:

```bash
pi install ./path/to/adversarial-review
```

To install into project scope rather than user scope, add `-l`.

This package bundles `pi-subagents` so the `subagent(...)` tool it provides is
available automatically — no separate install needed. If you already have
`pi-subagents` installed at user/project scope, both copies will be present;
pi loads them with separate module roots so they don't collide.

## Usage

Either harness:

```
/adversarial-review                          # diff against origin/main
/adversarial-review origin/feature-base      # stacked branch
/adversarial-review --no-migration-cost      # greenfield / large refactor
```

Pi also supports `/skill:adversarial-review …` (same effect).

## How it works

1. Get the diff against the chosen base ref.
2. Auto-detect greenfield/large-refactor; may suggest `--no-migration-cost`.
3. Dispatch three reviewers in parallel: quality & standards, cleanliness &
   modularity, security.
4. Dispatch three challengers in parallel — one per reviewer. Each challenger
   filters noise and adds missed findings.
5. Synthesize survivors: deduplicate, rank by severity, group by theme.
6. Present findings one at a time, waiting for your reaction before moving on.

Reviewers and challengers run with fresh context (no shared parent history). The
adversarial pass typically filters 30–50% of initial findings.

## Why two harnesses, one repo?

The workflow is identical; only the subagent-launch syntax differs. `SKILL.md`
holds the workflow; `dispatch-claude-code.md` and `dispatch-pi.md` hold the
harness-specific tool calls. Adding a third harness later means adding one more
dispatch file.

## Limitations

- **Pi:** relies on the bundled `pi-subagents` extension. The skill will refuse to
  run if the `subagent(...)` tool is not available (e.g. extension load failure).
  Run `pi reinstall npm:adversarial-review` if that happens.
- **Claude Code:** uses `general-purpose` subagents by default. Custom domain
  subagents are not required.
