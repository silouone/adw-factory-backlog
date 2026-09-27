---
id: adw-fe-01-journal-schema
type: feat
status: done
priority: 1
created: 2026-09-12
depends: [adw-sysprompt-01-own-the-system-prompt]
attempts: []
---
# Journal schema: `target` on run-start, model + usage breakdown on AgentUsage

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

Three pieces of data the factory already has and throws away, all at the same
seam. Doing them as one change keeps `journal.ts`'s event union edited once.

1. **`target` is not journaled.** `runs/` is one flat directory and nothing
   records which target repo a run was against, so factory self-runs and cLens
   product runs are indistinguishable.
2. **`AgentUsage` has no `model`.** `provider`/`model` reach only the ticket's
   `attempts[]` at finalize — per run, not per role, and not live.
3. **The usage breakdown is summed away.** `build.ts:472` adds
   `input_tokens + output_tokens` and discards the rest — including **cache
   reads, which on a cached run are the large majority of the real prompt**.

## Requirements

- [ ] `run-start` carries `target` (the target config's `name`).
- [ ] `AgentUsage` carries the resolved `model` for that node.
- [ ] `AgentUsage` carries the usage **components** (input, output, cache read,
      cache write) — not only their sum. Keep a summed accessor if callers want
      one, but the components must survive to the journal.
- [ ] Existing consumers (`status.ts`, the PR-body stats) keep working against
      journals **written before** this change. Additive only.

## Verify

- [ ] A run's `run-start` names its target; a journal without it still reads.
- [ ] An agent `node-end` carries model + the four components; the components
      reconcile to the previously-reported total.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Anything that renders this. Any other journal event type (later tickets add
`agent-config`, `prompt`, `heartbeat` — deliberately sequenced, not parallel,
because they all edit the same union).

---

## Resolution — 2026-09-13 (salvaged by hand)

The run `adw-fe-01-journal-schema-1789213958548` died inside the `test` node —
no `node-end`, no `run-end`, ticket left `in-progress`. The worktree was intact
and the work complete: 11 files, +350/−6, every `consumeAgentStream` call site
threaded (build, repair, revise-test-only, build-fix, ci-repair) and `target`
journaled from both `runLane` callers (`cli.ts`, `ci-round.ts`).

Rebased onto `origin/main` (`e9dd79d`) — applied clean, no conflicts — and
verified there:

| | |
|---|---|
| `bun run lint` | clean |
| `bunx tsc --noEmit` | clean |
| `bun test` | **1048 pass, 4 skip, 0 fail**, 335 s |
| peak concurrent containers | 1 |
| docker skips | **0** — every container test really ran |

Supersedes `adw-obs-02-cache-token-mirror` (PR #12), which was minted
2026-09-13 without noticing this ticket already owned the same seam.
Requirement 3 here — the four-component `breakdown` — is a superset of that
PR's two flat cache fields, and it is the shape `specs/adw-v1.2-live-view.md`
decomposed for `fe-02`/`fe-03`/`fe-10` to build on. Only one of the two may
land: they edit the same lines of `journal.ts` and `build.ts`.
