---
id: cqc-fe-14-one-contract-types-generated-from-openapi-9d0a63
type: feat
status: done
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-14-one-contract-types-generated-from-openapi-9d0a63-1790689265727","branch":"adw/cqc-fe-14-one-contract-types-generated-from-openapi-9d0a63","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-14-one-contract-types-generated-from-openapi-9d0a63-1790689265727/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# The CMC's CQC types are generated from the backend's OpenAPI, not hand-written from a prose doc

## ⚠️ Do not dispatch until `cqc-be-14` is merged

`cqc-be-14-the-content-summary-the-cmc-renders` rewrites `backend/openapi.yaml` (amendment
BE-38). Generating against the current file would bake in the shape that is being replaced.
**This blocker cannot be expressed in `depends:`** — that field only resolves inside this
target's own backlog, and a cross-target id is silently treated as met (`unmetDependsRefusal`
filters dangling ids). The guard is this heading and the operator.

## Why

On 2026-09-29 the page met the real API for the first time and React unmounted the whole tree:

```
Uncaught TypeError: Cannot read properties of undefined (reading 'Fail-block')
  at CheckedContentSummary
```

The CMC's `ContentSummary` was hand-written from `docs/cqc/backend-contract.md` line 117; the
backend implemented `spec-cqc-backend-release-1.md`. Both were written 2026-09-26, both were
implemented faithfully, and the two prose documents disagreed. A field-by-field diff of every
live payload showed `summary` was the **only** breaking divergence — this time. Nothing stops
the next one, because the CMC's types and the backend's contract are two documents maintained
by hand in two repositories.

`cqc-be-10` published `backend/openapi.yaml` precisely so there could be one contract. Every
`cqc-fe-*` ticket says the contract doc "becomes `backend/openapi.yaml` later". This is later.

## Scope

- Bring `openapi.yaml` into this repo and generate TypeScript from it, replacing the
  hand-written interfaces in `src/services/ContentQuality.service.ts`
  (`ContentRow`, `RunSummary`, `ContentSummary`, `ContentFacets`, `ContentListResponse`,
  `ContentRunsResponse`, `CheckDetailResponse`, `ArtefactSummary`, `RunMetadata`,
  `PlayabilityReport`, `ContentQualityConfig`, the trigger request/response types).
- Retire `docs/cqc/backend-contract.md`: replace its body with a pointer to the generated
  types and to `openapi.yaml` in the content-quality-checker repo.
- Keep the service class, its methods and its error normalisation as they are. This ticket
  changes where the *types* come from, not how the client behaves.

## The design choice, to make and state in the PR

How the spec reaches this repo. **Vendoring a copy** (commit `openapi.yaml` under e.g.
`src/services/cqc/openapi.yaml`, regenerate by hand when it changes) is the option that works
with this repo's constraints: the build agents have no network, node is pinned at 14.21.3, and
the two repos have no shared CI. Fetching at build time does not. Take vendoring unless you
find a concrete reason it cannot work — and if you do, stop and say so rather than inventing a
third path.

Whatever you choose, the PR must say how a future backend change reaches these types, because
that is the whole point of the ticket.

## Constraints (read first)

- **Node 14.21.3, TypeScript 3.9.10.** Any generator must emit types this compiler accepts —
  no `satisfies`, no template-literal types, no `const` type parameters. Check the generator's
  output against `tsc` before committing to it.
- **No network in the sandbox.** A generator that is not already in `package.json` cannot be
  installed. If none of the installed tooling can do the job, **do not add a dependency** —
  stop and say so in the final message, and propose what to add. The operator will land it.
  A hand-transcribed `types.ts` that is *derived from and checked against* `openapi.yaml` is an
  acceptable fallback if it comes with the check in the next bullet.
- The generated or transcribed types must be **verifiable**: a test that fails if
  `openapi.yaml` and the types drift. Without that, this ticket has changed nothing.

## Acceptance criteria

- [ ] Every CQC response type used by the page comes from `openapi.yaml`, not from a
      hand-written interface authored against prose.
- [ ] A test fails if the vendored `openapi.yaml` and the types disagree.
- [ ] `docs/cqc/backend-contract.md` no longer describes response shapes; it points at the
      contract.
- [ ] The three shapes that differ additively today still work: the API sends
      `exists_in_go1` on `ContentRow`, `case_state` on `facets`, `verdict_scope` on config,
      and extra fields on `RunMetadata` and `PlayabilityReport`. Extra fields must not break
      the page.
- [ ] tslint, the 952-test suite and `npm run build` stay green.
- [ ] The PR states how a backend contract change reaches these types from now on.

## Base

**Base branch: `cqc/release-1`**, not `master`.
