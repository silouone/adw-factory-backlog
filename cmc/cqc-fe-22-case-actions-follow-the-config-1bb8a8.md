---
id: cqc-fe-22-case-actions-follow-the-config-1bb8a8
type: feat
status: queued
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: []
---
# Resolve, Validate and Reopen are disabled, with the reason, when the backend says they don't exist

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
