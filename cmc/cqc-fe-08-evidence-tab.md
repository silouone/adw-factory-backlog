---
id: cqc-fe-08-evidence-tab
type: feat
status: queued
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-06-run-drawer-header-and-checks]
attempts: []
---
# The Evidence tab shows every proof behind a verdict, ranked by trust, and explains each gap

The evidence view of the run drawer (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 38–43 and 57; "Domain rules: evidence availability",
"Missing-evidence reasons", "Data fetching: downloads").

## What to build

- **Evidence availability, a pure domain rule tested directly.** For each
  evidence item cited by the report, work out:
  - available or unavailable;
  - the resolved file;
  - the viewer kind: SCORM table, snapshots stepper, image, text or raw;
  - the trust rank, 1–8, most direct first.

  Accessibility snapshots are added as an extra item. The agent's own account
  (the narrative) has the lowest trust and is labelled that way. A narrative
  whose cited file is missing falls back to the agent's last message, with a
  note saying so.
- **Missing-evidence reasons:** one explanation per evidence kind, covering why
  it is missing and what would make it available.
- **Artefacts on demand:** `GET /cqc/checks/{run_id}/artefacts/{name}` returns
  `{ url }`, a fresh presigned URL. It is fetched only when a viewer opens and
  never cached across sessions. Downloads check the HTTP status. A cached
  failure offers Retry, which asks for a fresh URL.
- **Evidence tab**, titled "Evidence (N of M available)":
  - a trust-ordered list with one banner stating the archive gap;
  - viewers:
    - SCORM API trace as a table (calls, writes, last status write), parsed defensively;
    - an accessibility snapshot stepper;
    - the final screenshot (image);
    - text and raw.
- **Unavailable evidence** is greyed out with ⊘ but still clickable. It opens a
  panel with N/A, the cited file, its trust rank, why it is missing and what
  would make it available.
- **Pinned limitations:** on tabs other than Checks, the limitations are
  pinned as a one-line link back to them.
- Checks in the Checks tab link to their evidence inline, reusing the same viewers.

## Acceptance criteria

- [ ] Domain tests cover availability, the resolved file, trust ordering, the added snapshot item and the narrative fallback note.
- [ ] Clicking an unavailable item opens the N/A panel with its kind-specific reason.
- [ ] A SCORM trace fixture renders as a table with the last status write identified. A malformed trace renders a readable error, not a crash.
- [ ] A failed or expired download shows Retry. Retry requests a fresh URL from the service and renders the content.
- [ ] The snapshot stepper steps forward and back with labelled buttons.
- [ ] The tab title shows "N of M available".
- [ ] The limitations link is pinned on the Evidence tab and absent on Checks, which shows them in full.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and the conventions it routes to before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).
- Never cache presigned URLs beyond the session.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-06-run-drawer-header-and-checks
