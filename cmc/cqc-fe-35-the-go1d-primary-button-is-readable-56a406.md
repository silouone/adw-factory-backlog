---
id: cqc-fe-35-the-go1d-primary-button-is-readable-56a406
type: manual
status: queued
priority: 2
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: []
---
# The Go1d primary button is readable

**Operator-executed (design-system work, CMC-wide).** Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Item 2.

## The problem

Primary `Button` text is white on `rgb(49,184,218)`: about **2.3:1**, failing WCAG AA (4.5:1 for
normal text, 3:1 for large). This is Go1d's own `accent`, used across the whole CMC, not a CQC
choice. `.agents/design-system/accessibility.md` documents no compensation for it. A local override
inside `src/components/ContentQuality/` would break CMC-RX-14.

## Steps

- Confirm the contrast on the go1d 0.9.95 theme source (accent and contrast colour).
- Decide with the CMC/CLB owners: a documented compensation (e.g. dark text on accent, or a
  darker accent shade from the theme) in `.agents/design-system/accessibility.md` and
  `known-drift.md`, or accept the gap and record it.
- `.agents/design-system/` is operator-owned (overlaid, never committed by the factory), so the
  doc change is yours. If a code compensation is chosen, file a follow-up FE ticket.

## Done when

- [ ] The decision is recorded in `accessibility.md` / `known-drift.md`.
- [ ] A follow-up ticket exists if code must change.
