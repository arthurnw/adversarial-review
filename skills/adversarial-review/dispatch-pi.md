# Dispatch adapter: pi (`pi-subagents`)

Use this adapter when running inside pi.

## Preflight: `subagent` tool availability check

`pi-subagents` ships bundled inside this package, so under normal installs the
`subagent(...)` tool is registered at startup. Still verify it's actually present
before dispatching anything — the extension may have failed to load (jiti error,
version mismatch with the pi core, conflicting user-scope install).

Check whether `subagent` is listed in your tool inventory. If you can't determine
this directly, attempt a no-op probe:

```typescript
subagent({ action: "list" })
```

If the tool is missing or errors as unknown, stop and tell the user:

> The `adversarial-review` skill requires the `subagent(...)` tool from the
> bundled `pi-subagents` extension. It does not appear to be loaded in this
> session.
>
> Try:
>
>     pi reinstall npm:adversarial-review
>
> If you also have `pi-subagents` installed at user/project scope, that copy
> may be shadowing the bundled one — check `pi extensions list` and resolve
> the conflict, then restart pi.

Do not try to fall back to single-agent review — the value of this skill is the
six-agent adversarial process. A degraded single-agent path would be misleading.

Run the preflight **before** step 1 (diff capture) — no point gathering the diff
if dispatch is unavailable.

## Step 3 dispatch — reviewers in parallel

Launch all three reviewers in a single `subagent` call. Use the builtin `reviewer`
agent; `context: "fresh"` ensures independent perspectives with no parent history
leaking in; `async: true` keeps the main chat responsive while reviewers work;
`output: false` returns findings inline rather than saving to disk.

Assemble each task prompt from SKILL.md §3, substituting `{BASE_REF}`, `{DOMAIN}`,
`{DOMAIN_INSTRUCTIONS}`, and (when active) the migration-cost-exclusion block.

```typescript
const reviewerRun = subagent({
  tasks: [
    {
      agent: "reviewer",
      task: "<assembled Reviewer A prompt — Quality & Standards>",
      output: false
    },
    {
      agent: "reviewer",
      task: "<assembled Reviewer B prompt — Cleanliness & Modularity>",
      output: false
    },
    {
      agent: "reviewer",
      task: "<assembled Reviewer C prompt — Security>",
      output: false
    }
  ],
  concurrency: 3,
  context: "fresh",
  async: true
})
```

While the parallel run is in flight you may do your own local diff inspection, but
do NOT edit files. Use `subagent({ action: "status", id: reviewerRun.id })` to
poll if needed. Wait for all three to complete before moving to step 4.

## Step 4 dispatch — challengers in parallel

Once all three reviewer outputs are in hand, launch three challengers with the same
flags. Each challenger gets exactly one reviewer's output substituted into
`{REVIEWER_OUTPUT}` from the challenger prompt template in SKILL.md §4. Do not let
challenger A see reviewer B's output, and so on.

```typescript
const challengerRun = subagent({
  tasks: [
    {
      agent: "reviewer",
      task: "<assembled Challenger prompt — Reviewer A output substituted>",
      output: false
    },
    {
      agent: "reviewer",
      task: "<assembled Challenger prompt — Reviewer B output substituted>",
      output: false
    },
    {
      agent: "reviewer",
      task: "<assembled Challenger prompt — Reviewer C output substituted>",
      output: false
    }
  ],
  concurrency: 3,
  context: "fresh",
  async: true
})
```

Wait for all three to complete before moving to step 5 (synthesis).

## Notes specific to pi

- **Use `reviewer`, not `delegate` or `general-purpose`.** `reviewer` is the pi
  builtin review specialist; it inherits pi's review defaults and is purpose-built
  for advisory findings. `delegate` has a heuristic that marks runs as failed when
  no files are edited — adversarial review produces prose, not edits, so it would
  trigger false failures.
- **`context: "fresh"` is required.** Some builtins default to forked context;
  fresh context is what gives each reviewer an independent perspective. Do not omit
  it.
- **`async: true` is the pi default.** Keep it; the parent remains responsive and
  intercom notifications arrive on completion.
- **`output: false` keeps findings inline.** If a user requests review artifacts on
  disk, switch to `output: "review/reviewer-a.md"` etc. with `outputMode:
  "file-only"` — but the default is inline.
- **`needs_attention` events.** If `subagent` returns a `needs_attention` event for
  any child, do NOT interrupt by default — reviewers can be quiet while thinking.
  Only interrupt if the user asks or the child has been silent past
  `needsAttentionAfterMs * 2`.
- **Migration-cost block.** When `--no-migration-cost` is active, append the
  exclusion block from SKILL.md §3 to each reviewer prompt AND each challenger
  prompt. Keep the text byte-identical to SKILL.md.
- **This is not `/parallel-review`.** The pi-subagents `/parallel-review` prompt
  template covers a similar three-reviewer workflow but omits the challenger step.
  This skill is the six-agent adversarial superset; do not substitute one for the
  other.
