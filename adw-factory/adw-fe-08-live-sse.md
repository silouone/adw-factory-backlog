---
id: adw-fe-08-live-sse
type: feat
status: done
priority: 2
created: 2026-09-12
depends: [adw-fe-06-gantt, adw-fe-04-heartbeat]
attempts: [{"runId":"adw-fe-08-live-sse-1789345184581","branch":"adw/adw-fe-08-live-sse","workspace":"/Users/silouane/adw-factory/runs/adw-fe-08-live-sse-1789345184581/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-08-live-sse-1789347220177","branch":"adw/adw-fe-08-live-sse-2","workspace":"/Users/silouane/adw-factory/runs/adw-fe-08-live-sse-1789347220177/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/33","provider":"claude","model":"sonnet"}]
---
# Live — SSE tailing and the three render states

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

The point of the whole view. The journal is `appendFileSync` per event and the
lane process runs **host-side for all three isolation kinds** — only the agent
runs in the sandbox. So a tailing reader streams mid-run with no new plumbing,
for worktree, container and remote alike. Append-only JSONL **is** the cursor;
no database, no write-ahead log.

## Requirements

- [ ] Byte-offset journal tailer emitting only complete lines and resuming
      correctly across appends.
- [ ] SSE push to the client; the view updates without a refresh.
- [ ] **Three render states — `running` / `finished` / `unknown`.** Stale
      heartbeat (> 60 s) ⇒ `unknown`, **never `running`**.
- [ ] A `watchdog` event stalls **one lane** while the run keeps rendering
      healthy — process death and agent silence are different failures and must
      look different.
- [ ] Live runs sort first in the grid.
- [ ] The view must not imply it can detect an **alive-but-looping** run. It
      cannot, in v1; say so rather than let the card read as authoritative.

## Verify

- [ ] Fresh heartbeat → `running`; stale → `unknown`; `run-end` present →
      `finished` regardless of heartbeat.
- [ ] A journal with **no** heartbeats (pre-`adw-fe-04`) renders without
      crashing and **without claiming liveness**.
- [ ] A watchdog trip stalls one lane and leaves the run healthy.
- [ ] Tailing resumes correctly across appends.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
