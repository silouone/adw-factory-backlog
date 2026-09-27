---
id: adw-render-05-parity-sweep
type: chore
status: done
priority: 2
created: 2026-09-18
caps: {minutes: 120, turns: 600}
depends: [adw-render-04-the-run-screen-becomes-components]
attempts: [{"runId":"adw-render-05-parity-sweep-1789928045839","branch":"adw/adw-render-05-parity-sweep","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-render-05-parity-sweep-1789928045839/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/101","provider":"claude","model":"sonnet"}]
---
# P4 — delete the retired render layer, and prove the view contracts still hold

> **Spec authority:** `specs/adw-v1.11-render-architecture.md` §4 **P4** and
> §9 (success criteria). `chore` lane: this is mechanical removal plus
> verification, with the judgment already spent in P0–P3.

## Requirements

- [ ] **R1 — delete the retired modules.** `src/web/render.ts`,
      `src/web/render-run.ts`, and `src/web/queue.prototype.ts` (plus
      `QUEUE-PROTOTYPE-NOTES.md`) if P2 has not already removed them.
- [ ] **R2 — delete the retired tests.** `test/web/render.test.ts`,
      `test/web/render-run.test.ts`, `test/web/usage-parity.test.ts`, and the
      HTML-asserting portions of `test/web/server.test.ts`. **Do not delete
      `server.test.ts`'s no-mutating-route tests** — those are a standing
      guarantee, not markup assertions.
- [ ] **R3 — the surviving layer is untouched.** Confirm by diff that
      `projection` `board` `card` `gantt` `timeline` `drawer` `usage`
      `metrics` `capture` `chain` `run-view` `runs` `tailer` `pricing`
      `run-summary` and their ~7,602 lines of tests were **not modified**
      across P0–P4. If any were, name each and say why in the PR body — that
      is a spec deviation worth seeing.
- [ ] **R4 — no orphans.** No dead import, no unreferenced CSS rule, no
      exported symbol with zero consumers. `biome`'s `noUnusedImports` /
      `noUnusedVariables` are already `error`, so the gate mostly enforces
      this; check exported-but-unused by hand.
- [ ] **R5 — re-queue `adw-fe-23`.** Flip
      `adw-fe-23-zoom-and-scroll-the-run-timeline` from `blocked` back to
      `queued`, widen its `depends:` to `adw-render-04-…`, and **strike its
      R4** (the `innerHTML` save/restore band-aid) — that requirement is
      satisfied structurally and re-implementing it would be dead code.

## Verify — the success criteria from §9, measured

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green, **with no second
      toolchain added** to `package.json` beyond P0's.
- [ ] **An idle live run pushes 0 bytes over 10 s.** The direct inverse of the
      11-pushes/0-bytes measurement that triggered this work. Record the
      number in the PR body.
- [ ] A user-collapsed section stays collapsed, and inner scroll, text
      selection and keyboard focus survive, across ≥10 live ticks.
- [ ] The v1.7 and v1.10 view contracts hold: the panel, the rail, the
      one-entry-per-box rule, the metric tri-state, the linear axis.
- [ ] `grep -rn "innerHTML" src/web/` returns nothing.
- [ ] `queue.prototype.ts` and the `?queue=` branch are gone.

## Out of scope

- Any redesign (§8). If the screens look different at the end of this, the
  migration failed its own success criterion.
- `adw-v1.5` D3 / `adw-console-01` — still gated on a separate operator
  decision about reading the ticket store, untouched by this work.
