---
id: adw-learn-02-a-run-names-the-factory-that-produced-it-3ceb3e
type: feat
status: in-review
priority: 2
created: 2026-09-28
review: false
caps: {minutes: 120, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-learn-02-a-run-names-the-factory-that-produced-it-3ceb3e-1790578748243","branch":"adw/adw-learn-02-a-run-names-the-factory-that-produced-it-3ceb3e","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-learn-02-a-run-names-the-factory-that-produced-it-3ceb3e-1790578748243/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/silouone/adw-factory/pull/149"}]
---
# A run names the factory version and the prompt templates that produced it

> `adw-learn-*` group (`ai_docs/2026-09-28-prompt-training-readiness.md` §3.2).
> Zero-token plumbing. `review: false`: hard-gated.

## The defect this closes

`run-start` carries `isolation`, `lane`, `target`, `type`, nothing else
(measured over 141 journals: 133 with those four keys, 8 older with three).
Nothing in a run record says **which factory** ran it: which commit of
`src/`, which revision of each file under `prompts/`.

The rendered prompt is hashed per stage (`prompt` events, `prompt-sink.ts`),
but that hash covers the ticket body and the plan artifact too, so it is
unique per run and cannot be grouped. Today the only way to attribute a
change in outcomes to a change in `prompts/bug-build-test.md` is to diff the
rendered prompt against `git log -- prompts/` by timestamp. 24 commits have
touched `prompts/`; none of them can be credited or blamed from the run
record.

## Requirements

- [ ] **R1** `run-start` gains `factory: { sha: string, dirty: boolean }`:
      the factory checkout's `HEAD` sha and whether its working tree had
      uncommitted changes under `src/` or `prompts/` at dispatch. Both read
      once, at the CLI edge, before the engine starts (Art. IX: the engine
      takes them as data, it does not shell out).
- [ ] **R2** `run-start` gains `templates: Record<string, string>`: for every
      `*.md` under `prompts/`, its filename → sha256 of its bytes, computed
      with `hashPrompt` (`prompt-sink.ts:43`). Byte-identical templates hash
      identical across runs, so a `group_by(templates["bug-build-test.md"])`
      is the attribution key.
- [ ] **R3** Both fields are optional on the `JournalEvent` type so every
      banked journal still parses; both real `runLane` callers (`cli.ts`'s
      main run and `ci-round.ts`'s mini-lane) always set them.
- [ ] **R4** `scripts/run-metrics.ts` and `scripts/run-autopsy.ts` gain a
      `--by-template <file>` grouping: outcomes and per-stage turns bucketed by
      that template's hash, oldest bucket first, with the first and last run
      timestamp per bucket. Runs lacking the field land in one `(unstamped)`
      bucket, named, never dropped silently.
- [ ] **R5** The web run header shows the short sha and a `dirty` badge when
      set. No other UI change.

## Verify

- [ ] Red first (Art. I).
- [ ] A run against a clean checkout journals `dirty:false` and a 12-entry
      `templates` map whose values match `shasum -a 256 prompts/*.md`.
- [ ] Editing one template between two fake runs changes exactly one entry.
- [ ] `--by-template bug-build-test.md` over the current `runs/` yields one
      `(unstamped)` bucket and no error.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Out of scope

Stamping the target repo's sha: `baseline.sha` already does that. Stamping the
SDK or CLI version: `agentConfig.claude_code_version` already rides on the
agent node-end.
