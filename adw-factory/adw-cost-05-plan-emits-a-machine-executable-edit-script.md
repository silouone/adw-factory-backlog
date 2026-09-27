---
id: adw-cost-05-plan-emits-a-machine-executable-edit-script
type: feat
status: queued
priority: 3
review: true
created: 2026-09-27
caps: {minutes: 240, turns: 1200}
depends: [adw-cost-04-take-the-agent-out-of-the-loop]
attempts: []
---
# Graduated from `adw-cost-04` R3/R3a — mechanical edits skip the agent

> **Evidence:** `adw-cost-04-take-the-agent-out-of-the-loop`'s own
> `.adw/artifacts/build.md`, R3/R3a section. `build` is **1.562B = 52% of
> the bill** (`ai_docs/2026-09-21-token-audit-FULL.md`). Some fraction of it
> is unambiguous mechanical application of decisions `plan` already made.

## What the spike found

Ceiling (5 real merged PRs, hunks classified by inspection, host `gh pr
diff`): **36/51 ≈ 70.6% of edits were anchorable** — a unique, stable-context
literal insert/replace, not a judgment call. Prototype (one real mechanical
PR, `#25`'s sysprompt-02 commit, replayed as a derived `(file, anchor,
replace)` edit list against a read-only scratch clone, run through a
committed pure applier `scripts/apply-plan-edits.ts`): **19/21 applied
(90.5%)**; both failures correctly surfaced as `anchor-not-unique`
(near-duplicate boilerplate needing wider context) rather than misapplying —
verified byte-for-byte against the real post-commit blob on the 8 files
touched.

The schema decided by the spike: `{file, anchor, replace}` — `anchor` an
exact substring of `file`'s **current** content occurring exactly once (the
SDK's own `Edit` tool's uniqueness contract, reused rather than invented),
`replace` the literal new text, `intent` an optional audit-only label never
consulted by the applier.

## Requirements

- [ ] **R1 — `plan` emits an edit-script artifact alongside its prose.**
      `.adw/artifacts/plan.md` gains a sibling (e.g.
      `.adw/artifacts/plan-edits.json`): an array of `{file, anchor, replace,
      intent?}` per the schema above, for the subset of the plan's changes
      `plan` itself judges mechanical. Not every plan step needs an entry —
      generative work (new tests, prose) has none.
- [ ] **R2 — `build` applies the edit script deterministically before its own
      agent turn.** Reuse `scripts/apply-plan-edits.ts`'s `applyPlanEdits`
      (already committed, tests green) as the applier; do not reimplement
      the uniqueness check. Applied edits are committed to the workspace
      tree with no agent call. `Edit`'s own uniqueness contract errors
      (`anchor-not-found`, `anchor-not-unique`) are the two failure
      reasons — no third invented here.
- [ ] **R3 — a failed edit falls back to an agent turn, scoped to just that
      edit.** `build`'s agent session receives the plan's remaining prose
      plus the specific failed edits (file, anchor, reason, intent) rather
      than the whole plan from scratch — the deterministic pass has already
      done the mechanical fraction's work, so the agent's job is only what
      it couldn't apply plus whatever the plan never treated as mechanical.
- [ ] **R4 — the digest carries feedback either way.** Journal/build report
      records `{applied: n, failed: n, reasons}` for observability
      (`run-autopsy.ts`-visible), matching the spike's own reporting
      convention.

## Verify

- [ ] Red test first (Art. I): `plan`'s node emits a `plan-edits.json`
      matching the schema for a fixture ticket with at least one mechanical
      and one non-mechanical change; confirmed failing before
      implementation.
- [ ] Red test: `build` applies a fixture edit script's clean edits without
      an agent call (mock/stub the agent dispatch and assert it is not
      invoked for the applied subset).
- [ ] Red test: a fixture edit with an `anchor-not-unique` failure reaches
      the agent turn with that specific failure surfaced, not silently
      dropped and not silently retried against the whole plan.
- [ ] **Measured:** applied/failed counts + reasons for at least one real
      dispatched ticket, quoted in the PR body — the spike's own 90.5% was
      one prototype off a replayed commit, not a live `plan`→`build` run.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- Natural-language `intent` driving any deterministic behavior — it is
  audit-only, per the spike's own schema note.
- Widening the schema beyond `{file, anchor, replace}` (e.g. multi-anchor,
  regex anchors) without new evidence that the literal-substring form is
  insufficient.
- Any node other than `plan`/`build` — `build-fix`/`repair` are not scoped
  here; if this graduates cleanly, a follow-on ticket can propose extending
  it, with its own evidence.
