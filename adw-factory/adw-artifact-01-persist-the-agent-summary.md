---
id: adw-artifact-01-persist-the-agent-summary
type: feat
status: done
priority: 1
created: 2026-09-17
depends: []
attempts: [{"runId":"adw-artifact-01-persist-the-agent-summary-1789598816035","branch":"adw/adw-artifact-01-persist-the-agent-summary","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-artifact-01-persist-the-agent-summary-1789598816035/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/62","provider":"claude","model":"sonnet"}]
---
# The agent's final summary exists only in a terminal that has scrolled away

> Found 2026-09-16 while prototyping the run panel (`?summary=F`). The web app
> cannot show a run's report because there is nothing durable to show.

## 1. What happens today

`cli.ts` prints the agent's summary on every completion and embeds it in the
PR body:

```ts
const summary = outcome.data.agentSummary;
if (typeof summary === "string" && summary !== "") {
  out(`☰ agent summary: ${summary}`);
}
```

Then drops it. It is **never journaled** and **never written under
`runs/<id>/`**. The only on-disk copy is `<workspace>/.adw/artifacts/*.md`,
and the workspace is reclaimed by `just clean` (Art. VII keeps it only until
then).

Confirmed against a real run — `runs/<id>/` contains `journal.jsonl`,
`prompts`, `spans`, `workspace`. No `artifacts`.

## 2. Why it matters

- The run screen cannot render the run's own report (`adw-fe-20` blocks on
  this).
- A `blocked` run never opens a PR, so for exactly the runs whose summary
  matters most, the console is the *only* copy. `cli.ts`'s own comment records
  why it prints on blocked runs at all: *"adw-m2-01, run obs-001: an agent
  correctly surfaced its blocker and the CLI discarded it."* The same discard
  is still happening one layer down.
- The §6 metrics are recomputed from journals. A report that never reaches the
  journal cannot be quoted.

## 3. Decision

**Persist the artifacts, mirroring `prompts/`.** At run end, copy
`<workspace>/.adw/artifacts/*.md` into `runs/<id>/artifacts/` and append one
journal record per file carrying `path`, `bytes` and `hash` — the exact shape
the existing `prompt` record already uses (`adw-fe-03-prompt-persistence` is
the precedent to copy).

Rejected alternatives:

- **Journal the summary inline** on `run-end.details`. One record, no new
  files — but it inlines an unbounded blob (a measured run's was ~4.6 KB) into
  a journal that is read on every grid render.
- **Read it back from the PR body via `gh`.** No new storage, but needs
  network, fails offline, and fails entirely for `blocked` runs — which is the
  case that motivated this ticket.

## 4. Build protocol

TDD, Article I — no source before a reviewed, red test.

1. Write tests at the persistence seam: given a finished run whose workspace
   holds artifacts, assert each is copied under `runs/<id>/artifacts/` and one
   journal record per file is appended with a correct byte count and hash.
   Assert the run still completes when the directory is absent (a lane that
   writes no artifacts must not fail), and that a reclaimed workspace after
   the fact does not invalidate what was copied.
2. Confirm red. Present for operator review.
3. Implement to green.
4. `bun run lint && bunx tsc --noEmit && bun test` all green.

## 5. Out of scope

- Rendering the artifacts. That is `adw-fe-20-the-run-panel`.
- Back-filling artifacts for runs already banked — they are gone with their
  workspaces, and the panel's "not available" state exists for them.
