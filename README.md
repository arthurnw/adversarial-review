# adversarial-review

Multi-agent adversarial review. Specialized reviewers run in parallel, then
adversarial challengers pressure-test their findings. The result is a curated,
high-signal list of recommendations presented one at a time.

Two skills ship in this package:

- **`/adversarial-review`** — reviews **production code** in a diff. Three
  reviewers (quality, modularity, security), three challengers.
- **`/adversarial-test-review`** — pressure-tests **tests** (especially
  LLM-generated ones). Three reviewers across distinct test-smell lenses
  (validity; behavior-vs-implementation; robustness & coverage), three
  challengers, then per-test findings with concrete fixes.

Both work in **Claude Code** (as a plugin) and **pi** (as a package). Single
canonical workflow per skill; dispatch differs per harness.

## Install

### Claude Code

Install from a local checkout:

```bash
claude mcp add --local adversarial-review /path/to/adversarial-review
```

Or add the plugin via Claude Code's plugin settings, pointing at this directory.
After install, `/adversarial-review` and `/adversarial-test-review` appear in
autocomplete.

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

### Code review

```
/adversarial-review                          # diff against origin/main
/adversarial-review origin/feature-base      # stacked branch
/adversarial-review --no-migration-cost      # greenfield / large refactor
```

1. Get the diff against the chosen base ref.
2. Auto-detect greenfield/large-refactor; may suggest `--no-migration-cost`.
3. Dispatch three reviewers in parallel: quality & standards, cleanliness &
   modularity, security.
4. Dispatch three challengers in parallel — one per reviewer.
5. Synthesize survivors: deduplicate, rank by severity, group by theme.
6. Present findings one at a time, waiting for your reaction before moving on.

### Test review

```
/adversarial-test-review                     # staged tests, else latest commit touching tests
/adversarial-test-review HEAD~3..HEAD         # tests changed in a range
/adversarial-test-review spec/models/user_spec.rb   # specific test files, reviewed in full
```

1. Resolve a test scope: staged test changes, else the latest commit that touched
   tests, else the explicit ref/range/paths you pass.
2. Dispatch three reviewers in parallel across three smell lenses; each also reads
   the production code under test.
3. Dispatch three challengers in parallel — one per reviewer — to defend tests
   against false positives and catch missed smells.
4. Synthesize survivors: dedupe by test, rank by severity.
5. Present per-test findings one at a time, each with a recommended action
   (delete / rewrite / strengthen / move) and a concrete fix. Say "fix it" on any
   finding to have the edit applied.

It hunts for: trivial/no-op, circular/tautological, testing-the-library,
over-mocked, implementation-detail, snapshot-only, happy-path-only, duplicate, and
flaky/nondeterministic tests.

For both skills, reviewers and challengers run with fresh context (no shared parent
history). The adversarial pass exists to cut false positives — for test review in
particular, that means not recommending you delete a test that quietly guards a
real regression.

Pi also supports `/skill:adversarial-review …` and `/skill:adversarial-test-review …`
(same effect).

## Why two harnesses, one repo?

The workflow is identical across harnesses; only the subagent-launch syntax
differs. Each skill's `SKILL.md` holds the workflow; its `dispatch-claude-code.md`
and `dispatch-pi.md` hold the harness-specific tool calls. Adding a third harness
later means adding one more dispatch file per skill.

## Limitations

- **Pi:** relies on the bundled `pi-subagents` extension. The skills will refuse to
  run if the `subagent(...)` tool is not available (e.g. extension load failure).
  Run `pi reinstall npm:adversarial-review` if that happens.
- **Claude Code:** uses `general-purpose` subagents by default. Custom domain
  subagents are not required.
