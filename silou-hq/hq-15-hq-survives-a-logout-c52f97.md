---
id: hq-15-hq-survives-a-logout-c52f97
type: manual
status: queued
priority: 2
created: 2026-10-02
depends: []
attempts: []
---
# HQ comes back on its own after a logout

Operator-executed. It closes the two hq-08 and hq-04 acceptance criteria that were never checked.

Done 2026-10-02:
- `bun run hq:install` ran. `com.silou.hq` is loaded (PID 43993, last exit 0), listening on 127.0.0.1:4327.
- Load times measured warm, under launchd: `/` 1.2 ms (1.8 KB), `/graph.json` 2.5 ms (418 KB), `/bundle.js` 0.8 ms (131 KB). Well under the 1 s target. The Google Fonts request in `index.html` is not counted.

## Steps left

- [ ] Log out and log back in, then run `bun run hq:status`: it reports "answering" with no manual start.
- [ ] Open `http://127.0.0.1:4327` from a cold browser and note the time to first paint of the rings (DevTools → Performance). Record it here.

## Blocked by

- (nothing)
