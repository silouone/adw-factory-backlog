---
id: adw-fe-09-node-drawer
type: feat
status: done
priority: 2
created: 2026-09-12
depends: [adw-fe-06-gantt, adw-fe-03-prompt-persistence]
attempts: [{"runId":"adw-fe-09-node-drawer-1789339650557","branch":"adw/adw-fe-09-node-drawer","workspace":"/Users/silouane/adw-factory/runs/adw-fe-09-node-drawer-1789339650557/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/23","provider":"claude","model":"sonnet"}]
---
# The per-node drawer — owner, kind, attempt, gates, and the prompt

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

Selecting a block opens its detail. All the metadata already exists in the
journal; the prompt bodies arrive with `adw-fe-03`.

## Requirements

- [ ] Drawer shows the node's **owner**, **kind** (from the `node-end` detail)
      and **attempt n/m** (from the `round` event).
- [ ] **Each gate's individual result** — which check failed, not merely that
      gates failed.
- [ ] The node's compiled prompt, read from its sidecar via the journal's index
      record. Length and hash are verified on read; a mismatch is shown as
      such rather than rendered silently.
- [ ] A node with no persisted prompt (a run predating `adw-fe-03`) says so.
- [ ] **A slot for the system prompt**, rendering today's honest value:
      *none (empty override)*. Choosing a different value is `adw-sysprompt-01`,
      deliberately not this ticket.
- [ ] Codex nodes render structurally, with configuration labelled **"not
      captured on this provider"** rather than empty — absence of capture is not
      absence of configuration.

## Verify

- [ ] Owner / kind / attempt / per-gate results render from a fixture journal.
- [ ] A hash mismatch on a sidecar is surfaced, not swallowed.
- [ ] A pre-persistence run renders the drawer without a prompt and says why.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
