---
id: hq-12-a-failed-routine-is-flagged-the-day-it-fails-cb38c0
type: feat
status: blocked
priority: 1
created: 2026-10-02
depends: [hq-10-routine-schedules-mean-what-launchd-means-2232d3]
attempts: [{"runId":"hq-12-a-failed-routine-is-flagged-the-day-it-fails-cb38c0-1790968302849","branch":"adw/hq-12-a-failed-routine-is-flagged-the-day-it-fails-cb38c0","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-12-a-failed-routine-is-flagged-the-day-it-fails-cb38c0-1790968302849/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# A routine that failed its last run is flagged the day it fails (this Mac)

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

The Problem Statement's own example: a brain-hub routine had been failing every day and nobody noticed. Today `com.silouane.brainhub.d1-channels`, `d2-redshift` and `d3-snapshot` all show last exit **127** in `launchctl list`, and HQ shows nothing. The spec already names "a broken routine" as a detector computed on every refresh (Implementation Decisions, *Read-only*). Its **input** is missing.

## Spec amendment first (Amendment rule)

Add to Implementation Decisions → **Data**, after "HQ owns no data":

> - **Routine health (this Mac):** on each local refresh HQ runs `launchctl list` (read-only) and keeps, per owned label, the PID and the last exit status. A routine is **failed** when it has no PID and its last exit status is non-zero. It is **not** failed when the status is `-` or `0`, or when it currently has a PID (a KeepAlive job that was restarted, such as `com.user.brain-hub-mcp`, running with a last status of 143). M1 routine health is out of scope for this amendment.

And in **Read-only**, change "a broken routine" to "a broken routine (failed, per *Routine health*)".

## Red first

- Pure parse of a fixture `launchctl list` output (`PID\tStatus\tLabel`), where `-` is a PID or status, gives `{pid?, lastExit?}` per label.
- With a fixture of `-  127  com.silouane.brainhub.d2-redshift`, the routine gets `meta.failed` with reason `exit 127`.
- With `55970  143  com.user.brain-hub-mcp`, it is **not** failed (it has a PID).
- `-  0` and `-  -` are not failed.
- An M1 routine is never marked failed (unknown is not failed).

## Acceptance criteria

- [ ] The spec amendment is in the same PR, above the code.
- [ ] The `launchctl` read is an injected reader, with a timeout, and never blocks the landing.
- [ ] The failed routine is marked on the dial station, its Routines row says `failed · exit N` with its output one click away, and In progress lists it under "needs you".
- [ ] Only owned labels are read (the same `com.silou*|com.user.*|io.sabado*` filter as the plists).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-10-routine-schedules-mean-what-launchd-means-2232d3
