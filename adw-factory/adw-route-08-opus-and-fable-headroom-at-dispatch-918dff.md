---
id: adw-route-08-opus-and-fable-headroom-at-dispatch-918dff
type: feat
status: queued
priority: 3
created: 2026-10-03
depends: [adw-route-01-a-gear-ladder-and-honest-prices-476b13]
attempts: []
---
# The operator sees Opus and Fable weekly usage before dispatching a routed run

> Spec: `specs/adw-v1.16-model-routing.md` (Solution: headroom); extends `specs/adw-v1.7-subscription-headroom.md`.

## Context

Routing moves plan and review onto Opus, and extreme tickets onto Fable. Both draw on separate per-model weekly buckets on the Claude subscription. Today the usage panel shows the session and weekly windows, but not which model bucket a routed run will draw on.

## Requirements

- [ ] **R1:** the usage panel shows the per-model weekly figures (Opus, Fable, when the account reports them) beside the existing windows.
- [ ] **R2:** a dispatch whose route plan includes G3–G5 shows which buckets it will draw on, next to the dispatch action.
- [ ] **R3:** a missing per-model figure renders as unknown, never as zero.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
