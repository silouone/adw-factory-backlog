---
id: adw-graph-05-the-why-cell-speaks-waves-5b894e
type: feat
status: done
priority: 1
created: 2026-09-28
review: true
caps: {minutes: 180, turns: 350, stallMinutes: 20}
depends: [adw-graph-02-the-dependency-model-in-the-projection-e146f2, adw-graph-04-the-graph-drawer-661279]
attempts: [{"runId":"adw-graph-05-the-why-cell-speaks-waves-5b894e-1790642868715","branch":"adw/adw-graph-05-the-why-cell-speaks-waves-5b894e","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-graph-05-the-why-cell-speaks-waves-5b894e-1790642868715/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/160","provider":"claude","model":"claude-sonnet-5-5"}]
---
# A waiting row says `W3 need 02 & 03`, and each dep opens the graph

> **Spec:** `specs/adw-v1.14-backlog-tab.md` §10, 2026-09-28, **G9**.
>
> Four table variants were prototyped. The operator:
> > "the table to elect is table 1, dependencies written as ' need x & y'"
>
> Today's why-cell reads
> `waits on cqc-be-01-backend-skeleton-and-config-end…`: one long id,
> truncated, and only the first unmet dep.

## Requirements

- [ ] **R1 — red first, a pure helper** in `backlog-groups.ts` (next to
      `whyLineOf`). It maps a row plus its project's `graph` to
      `{wave, deps: {id, label, state}[]}` for a `waiting` row, and to
      `undefined` for anything else.
      - Unmet deps only, in `deps` order.
      - Labels come from the graph nodes (`adw-graph-02` R7).
- [ ] **R2 — the text.**
      - The why-cell is `W<wave>`, as a small badge, then `need`, then the
        labels:
        - one dep: `need 02`;
        - two deps: `need 02 & 03`;
        - three or more: `need 04, 05 & 06`, with commas and a final `&`.
      - Each label is coloured by its dep's state (the `s-<section>` palette).
      - The row tooltip lists each dep's full id and status.
      - Test the joiner for 1, 2 and 3+ deps.
- [ ] **R3 — each label is a chip.**
      - It is a `<button data-dep-id>` that opens the graph focused on that
        dep, through `adw-graph-04`'s trigger.
      - Its click does **not** select the row (stop propagation).
- [ ] **R4 — unchanged elsewhere.**
      - Blocked, ready, running, in-flight, drift, manual and epic rows keep
        today's why-lines, byte for byte.
      - The state-major sections are unchanged.
      - The existing `whyLineOf` tests stay green.
- [ ] **R5 — DOM test:**
      - a waiting row renders `W3 need 02 & 03`;
      - clicking `03` opens the graph focused on `03`, and `selectedRowId` is
        unchanged.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web` →
Backlog, `content-quality-checker`:
- `cqc-be-10` reads `W5 need 04, 05 & 06`;
- clicking `05` opens the graph on 05.
