---
id: cqc-fe-22-case-actions-follow-the-config-1bb8a8
type: feat
status: in-progress
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-22-case-actions-follow-the-config-1bb8a8-1790795241911","branch":"adw/cqc-fe-22-case-actions-follow-the-config-1bb8a8","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-22-case-actions-follow-the-config-1bb8a8-1790795241911/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# Resolve, Validate and Reopen are disabled, with the reason, when the backend says they don't exist

> **Reset to `queued` 2026-10-01.** The first attempt blocked on
> `ContentQuality > renders manual resolution outcome, person, date-time and note in the
> drawer`, which this ticket did not introduce: `cqc/release-1` was briefly red and the
> repair loop spent all three rounds on someone else's failure. Nothing was committed and
> no branch was pushed. The branch is green again (1028 tests, 123 suites), so start from a
> clean read of the ticket — do not try to work around that test.

> 2026-09-30: dispatched early and blocked on a false-green baseline; nothing to salvage. Requeue only after cqc-be-28 is done AND deployed to dev (PR #39 merged, but dev still runs 3dc471e, so not deployed).

Sources: CMC spec "Unavailable actions are disabled and explain why"
(`spec-cqc-fe-release-1.md:381-387`); the `cqc-fe-16` decision (a), taken by the operator on
2026-09-30. Backend: `cqc-be-28` adds `actions: { resolve, reopen }` to `GET /cqc/config`,
each `{ enabled, disabled_reason? }`.

The `cqc-be-*` blocker below lives in the CQC ticket store (`~/adw/backlog/cqc`), so it is **not** in `depends:`, because `just next` only resolves ids in the same store. Check that it is `done` (merged **and deployed to dev**) before dispatching.

## Today

- `CaseActions` gets no config; its props are `{api, apiBaseUrl, content, jwt}`
  (`common/CaseActions.tsx:29-34`).
- It renders live and calls a route that doesn't exist yet.
- Its mount points already have config in scope: `CheckedContentList/index.tsx:85-86` and
  `RunDrawerFooter.tsx:19-33, 97`.
- Reuse the disabled-with-reason pattern from `RunCheckAction.tsx:38, 62, 133-160`.

## Red first (Art. I)

`CaseActions.test.tsx`: with `actions.resolve.enabled = false`, the Resolve and Validate
buttons are disabled and the reason is visible. The same holds for Reopen.

## Acceptance criteria

- [ ] `ContentQualityConfig` gains `actions` (`ContentQuality.service.ts:299-317`).
- [ ] If `actions` is **absent** (an older backend), the actions are disabled, not enabled.
- [ ] Both mount points pass the config.
- [ ] No request is sent while an action is disabled.

## Blocked by

- cqc-be-28-config-says-which-case-actions-exist-40185e (CQC store)
