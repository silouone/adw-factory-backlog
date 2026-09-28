---
id: adw-graph-02-the-dependency-model-in-the-projection-e146f2
type: feat
status: queued
priority: 1
created: 2026-09-28
review: false
caps: {minutes: 150, turns: 600, stallMinutes: 25}
depends: [adw-graph-01-manual-tickets-keep-their-deps-c6db32]
attempts: []
---
# The backlog projection carries each project's dependency graph: waves, critical path, held-by

> **Spec:** `specs/adw-v1.14-backlog-tab.md` §10, 2026-09-28, **G1–G6**.
> Decided by the operator over three prototype rounds on real data.
>
> **Visual reference only:** `/Users/silouane/personal_project/adw-factory/.proto/backlog-graph/model.ts`
> (throwaway and never committed, so it does not exist inside a run worktree).
> - Do not copy it. Re-derive under Art. I.
> - Where it and this ticket disagree, this ticket wins.
>
> This ticket is **data only**. No UI; ship the model on the wire.
> Derivation lives in the projection (v1.14 R7). Components only read it.

## The wire shape (add to `src/web/backlog.ts`)

```ts
export interface BacklogGraphNode {
  readonly id: string;
  /** The id's sequence part minus the project's common id prefix:
   *  cqc-be-04-list-… → "04"; adw-backlog-03-… → "backlog-03".
   *  An outside node keeps its full sequence ("clens-007"). */
  readonly label: string;
  readonly title: string;          // row title; outside: the dep id
  readonly status: string;         // verbatim
  readonly state?: BacklogState;   // absent for closed / non-ticket / outside
  readonly closed: boolean;
  readonly open: boolean;          // G1: !closed; outside: !met
  readonly outside: boolean;       // a dep that is not a row of this project
  readonly kind: BacklogRow["kind"] | "outside";
  readonly priority?: 1 | 2 | 3;
  readonly wave: number;           // G1
  readonly critical: boolean;      // G4
  readonly heldBy?: string;        // G5: nearest blocked ancestor's label
  readonly openDeps: number;       // G3 drift: closed node with n open deps
}
export interface BacklogGraphEdge {
  readonly from: string;           // the dep
  readonly to: string;             // the ticket that needs it
  readonly met: boolean;
  readonly critical: boolean;      // G4, edge by edge
}
export interface BacklogGraph {
  readonly nodes: readonly BacklogGraphNode[];
  readonly edges: readonly BacklogGraphEdge[];   // G3 edges only
  readonly maxWave: number;
  readonly criticalPath: readonly string[];      // ids, wave-ascending
}
// BacklogProject gains:  readonly graph: BacklogGraph;
```

## Requirements

- [ ] **R1 — red first, one test per rule.** Put them in a new
      `test/web/backlog-graph.test.ts`, over hand-built rows, no filesystem.
      Watch each fail before implementing.
- [ ] **R2 — G1 waves.**
      - A closed node is wave 0.
      - An open ticket is `1 + max(wave of its open deps)`, so an open ticket
        whose deps are all done is wave 1.
      - An outside node is wave 0.
      - A cycle `a → b → a` terminates and gives both a finite wave.

      Fixture: the cqc backend plan (`00` done; `01` ← `00`; `02`,`03` ← `01`;
      `04`,`05` ← `02`+`03`; `07` ← `00`+`03`; `08`,`09` ← `03`; `06` ← `05`;
      `10` ← `04`+`05`+`06`).
      - Expected waves: 01=1, 02/03=2, 04/05/07/08/09=3, 06=4, 10=5.
      - Expected `maxWave` = 5.
- [ ] **R3 — G2 pruning.** A closed row is a node only if an open row of the
      project lists it in `deps`.
      - Test: done `x` ← done `y` ← open `z`. Nodes are `y` and `z`; `x` is
        absent.
- [ ] **R4 — G3 edges.** An edge is emitted only when its `to` is open.
      - Test: done→done yields no edge.
      - Test: a closed ticket with an open dep yields no edge and has
        `openDeps: 1`.
- [ ] **R5 — G4 critical path, edge by edge.**
      - Take the longest open chain. Ties go to priority, then id.
      - Walk back from its tail and mark exactly the edges walked.
      - On the R2 fixture, `criticalPath` = `[01, 02, 05, 06, 10]` and exactly
        4 edges are `critical`.
      - Regression: `05→10` is **not** critical, even though both ends are on
        the path.
- [ ] **R6 — G5 held.**
      - With `01` blocked in the R2 fixture, every open descendant of `01`
        has `heldBy: "01"`.
      - A blocked node has no `heldBy`.
      - With two blocks, the nearest one (fewest hops) wins.
- [ ] **R7 — labels.**
      - The common prefix is cut back to its last `-`. So `cqc-be-00`…
        `cqc-be-11` → `00`…`11`, and a mixed `adw-m4-08` / `adw-backlog-03`
        project → `m4-08` / `backlog-03`.
      - Outside nodes keep their full sequence.
- [ ] **R8 — on the wire.**
      - `loadBacklog` sets `graph` on every project. A `hasStore:false`
        project gets an empty graph.
      - Malformed and epic rows are not nodes.
      - Manual rows are nodes (they have deps since `adw-graph-01`).
- [ ] **R9 — purity.**
      - A pure module with zero I/O and no clock, so the backlog SSE still
        pushes 0 bytes when idle (v1.14 §8 criterion 4).
      - It is either a new `src/web/backlog-graph.ts` called by
        `loadBacklog`, or inside `backlog.ts`. Pick one and say why in the
        header comment.
      - Existing backlog tests stay green, unmodified.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web` →
`/backlog.json`: for content-quality-checker,
`graph.criticalPath` reads 01→02→05→06→10, and every waiting row's node has
a wave ≥ 2.
