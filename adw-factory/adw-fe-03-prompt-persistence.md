---
id: adw-fe-03-prompt-persistence
type: feat
status: done
priority: 1
created: 2026-09-12
depends: [adw-fe-02-agent-config-capture]
caps: {minutes: 120, turns: 600}
attempts: []
---
# Persist every prompt actually handed to an agent

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

No prompt the factory composes is ever written to disk. They survive only
*incidentally*, inside the cLens capture — and **18 of 69 captured sessions
have none at all** (measured 2026-09-11), so that is not a guarantee.

There are **four** sites, not one. Three are built inline and never even placed
on run context: `repair` (which carries the full diff so far), `ci-repair`,
and `revise`.

There is also a live defect here: the `feat` lane compiles **three** distinct
prompts and every one overwrites the same run-context key, so only the last
survives to run end. Persisting at the query boundary — where each prompt is
actually used — is what makes all three recoverable.

## Requirements

- [ ] Every prompt handed to an agent is written, at the query boundary, to a
      sidecar under the run directory. Four sites.
- [ ] The journal records an index entry naming the node, the sidecar path, the
      byte length and a content hash — so a truncated or externally edited body
      is detectable. **The journal stays the index** (Art. VIII); bodies are
      sidecars because they run 4–30 KB and a feat run compiles three.
- [ ] A failed sidecar write is journaled and the run **continues**. Same
      tolerance stance as tracing and capture.
- [ ] Sidecars inherit the 0700 owner-only posture; no redaction (the boundary
      is the guard, not laundering the contents).

## Verify

- [ ] A `feat` run yields **three distinct prompt bodies**, not one — the
      regression test for the overwritten-key defect.
- [ ] Each of the four sites writes its body and journals a matching index
      record; the recorded hash and length match the file.
- [ ] A failing write journals the failure and the run reaches its normal
      outcome.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
