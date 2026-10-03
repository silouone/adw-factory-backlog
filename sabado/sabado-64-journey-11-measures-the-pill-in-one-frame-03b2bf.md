---
id: sabado-64-journey-11-measures-the-pill-in-one-frame-03b2bf
type: bug
status: queued
priority: 1
created: 2026-10-03
depends: []
attempts: []
---
# test(e2e): journey 11 measures the pill and the title once they have settled, so it stops failing PRs that never touched the calendar

## The bug

Journey 11 (`frontend/e2e/journeys/11-calendar-provenance.spec.ts:49`, step « an imported event
carries its provenance pill with the title ») is a **flake**. The assertion
`Math.abs(pillBox.x - title.x) <= 2` fails by 2–5 px:

```
the pill (left 918.236) is left-aligned with the title (left 920.863)
the pill (left 924.197) is left-aligned with the title (left 928.339)
the pill (left 928.338) is left-aligned with the title (left 933.436)
```

**Proof that it is a flake:** on PR #1360, the same tree (`1d68a146` and `4e9513ea` have an empty
diff) went green at 12:30 and red at 12:51 on 2026-10-03. It went red seven times between
2026-10-02 and 2026-10-03, on PRs with no calendar change: sabado-42, -46, -61, -44, and
fix/an-all-day-import.

**The earlier fix did not hold.** The spec already waits for `html.is-relaying` to clear, a fix for
an earlier version of this same 3.3 px drift. It still fails. `ds/EventPanel/EventPanel.css` has no
transition or animation. So whatever settles late is somewhere else: the panel's anchoring or
positioning after the month → 3-day navigation (#1302), or a font swap.

The two `boundingBox()` calls in `Promise.all` are separate round-trips. If the panel is still
moving, they measure two different frames.

## Requirements

- [ ] **R1** Find what is still moving when the boxes are read: record a trace on a failing run, or
      log `getAnimations()` and the panel's rect over a few frames. Write the cause in the PR body.
      Do not guess.
- [ ] **R2** Measure the title and the pill in **one** `page.evaluate` (the same frame), after the
      cause from R1 has settled. Alternatively, wrap the alignment assertion in
      `expect(async () => …).toPass({ timeout: 5_000 })`. Keep the **2 px tolerance**: the claim
      "flush with the title" stays as strict as it is.
- [ ] **R3** If R1 shows a real layout bug (the pill really is offset at rest), fix the layout.
      Do not loosen the test.

## Verify

- [ ] `npx playwright test --project=journeys 11-calendar-provenance --repeat-each=30` passes
      30/30 against a local backend (per ci.yml's `e2e` job: alembic upgrade, then uvicorn on
      :8010). Paste the count in the PR body.
- [ ] Journeys 10 and 14 are still green.
