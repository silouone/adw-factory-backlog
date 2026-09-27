---
id: adw-bug-08-a-fail-reason-never-reaches-the-journal
type: bug
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-bug-08-a-fail-reason-never-reaches-the-journal-1789485491793","branch":"adw/adw-bug-08-a-fail-reason-never-reaches-the-journal","workspace":"/Users/silouane/adw-factory/runs/adw-bug-08-a-fail-reason-never-reaches-the-journal-1789485491793/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/54","provider":"claude","model":"sonnet"}]
---
# A node's `fail()` reason never reaches the journal — "why did it stop" is answerable only from a terminal that has scrolled away

> Reproduced 2026-09-15 by `minio-quay-registry-move-1789464269318` (target
> `api-content`): a chore run reached gate-green and died at `push`. The
> journal records **nothing** about why, and the work had to be salvaged by
> hand.

## 1. What happened

```
$ just fails minio-quay-registry-move-1789464269318
{"type":"node-end","node":"push","outcome":"fail"}
{"type":"run-end","outcome":"blocked"}
```

That is the entire durable record of the failure. No reason, no stderr, no
classification. The README's own promise — *"each one journals its own trip,
so 'why did it stop' is always answerable from `runs/<runId>/journal.jsonl`"*
— does not hold for the single most common failure shape: a node returning
`fail()`.

The reason **is** produced. `push.ts:216` builds it, carrying the real git
stderr:

```
push failed after 3 attempts (<lastError>); local branch "<branch>" left
intact for autopsy and retry (E6)
```

and `engine.ts:607` threads it into `blocked(...)`, which surfaces it on
**stderr** and nowhere else. The run that produced the line above was one of
seven launched by hand in separate terminals; by the time the failure was
investigated the line was gone. The root cause of that push failure is, as of
this ticket, **permanently unknowable**.

## 2. Root cause

`engine.ts:498` appends `node-end` with `type`, `node`, `outcome` and an
optional `details` bag (review / agent / gates). `result.reason` is read at
line 607 for the run outcome and **never written to the journal**. The same
gap exists on `run-end` (`engine.ts:302`), which journals `outcome` and
`durationMs` only.

So the journal — the factory's only durable evidence base, and a gitignored
per-machine one at that — records *that* a node failed and never *why*.

## 3. Why this is priority 1

It is not a cosmetic logging gap. It is the difference between a blocked run
being a **diagnosable event** and being a shrug. Three concrete costs already
paid:

- `adw-bug-09` (the sibling ticket) cannot state its own root cause, because
  the stderr that would name it was never written down.
- A blocked run's post-mortem currently requires the operator to have kept the
  launching terminal open. That is not a property of a factory.
- `just fails` — the recipe whose entire job is "failures, retries and every
  bound-trip in one pass" — returns an outcome with no cause.

## 4. Scope

- `node-end` carries `reason` whenever the node returned `fail` (and the
  engine-wrapped message whenever it threw and journalled `outcome:"error"`).
- `run-end` carries the terminal `reason` for a `blocked` outcome.
- `just fails` / `just outcome` surface it without hand-rolled `jq`.

Out of scope: changing any node's reason *wording*, changing retry behaviour
(that is `adw-bug-09`), and streaming node stdout/stderr into the journal
wholesale — this ticket is the one already-composed reason string, nothing
more.

## 5. Red test first (Art. I)

1. A lane whose node returns `fail("...")` must journal a `node-end` whose
   `reason` is that string verbatim. Red today: the field is absent.
2. A lane whose node **throws** must journal `outcome:"error"` with the
   engine-wrapped `ticket "<id>" node "<name>": <message>` as its reason.
3. The terminal `run-end` for a blocked run must carry the same reason the
   CLI prints, so the journal and the terminal can never disagree.
4. A green run's `node-end`/`run-end` must be byte-identical to today — no
   `reason` key appears on a success path.

## Verify

- `bun run lint && bunx tsc --noEmit && bun test` green.
- `just fails <blocked-run-id>` names the cause on a real blocked run.
- Replaying `minio-quay-registry-move-1789464269318`'s shape through the
  engine test harness yields a journal that says why push failed.
