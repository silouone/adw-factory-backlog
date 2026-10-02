---
id: hq-13-m1-routines-report-their-failures-too-eb305c
type: feat
status: queued
priority: 3
created: 2026-10-02
depends: [hq-12-a-failed-routine-is-flagged-the-day-it-fails-cb38c0, hq-11-an-m1-scan-survives-a-missing-cache-933bf9]
attempts: []
---
# M1 routines report their failures too

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

hq-12 flags failed routines on this Mac only. On the M1, the scanner reads plists but not their run state.

## Spec amendment first (Amendment rule)

In Implementation Decisions → **The M1**, extend "It reads only:" with:

> - the PID and last exit status of the launchd labels HQ owns (`launchctl list`, filtered to the same prefixes).

And extend hq-12's *Routine health* paragraph to the M1. An M1 that is offline, or a snapshot older than `2 × scanEveryMin`, gives **unknown**, never failed.

## Red first

- A fixture M1 snapshot with `{label, pid: null, lastExit: 1}` gives `meta.failed` on the `m1` routine.
- A stale snapshot (older than 2 × `scanEveryMin`) gives unknown, not failed.
- A snapshot from the old scanner (no health fields) still parses, and every routine is unknown.

## Acceptance criteria

- [ ] `m1-scan.py` emits health only for owned labels, and still reads nothing else new.
- [ ] Dial, Routines and In progress treat M1 failures exactly like local ones, with the machine shown.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-12-a-failed-routine-is-flagged-the-day-it-fails-cb38c0
- hq-11-an-m1-scan-survives-a-missing-cache-933bf9
