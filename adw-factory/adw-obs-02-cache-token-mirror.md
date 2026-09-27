---
id: adw-obs-02-cache-token-mirror
type: chore
status: done
priority: 2
created: 2026-09-13
depends: []
attempts: [{"runId":"adw-obs-02-cache-token-mirror-1789309935236","branch":"adw/adw-obs-02-cache-token-mirror","workspace":"/Users/silouane/adw-factory/runs/adw-obs-02-cache-token-mirror-1789309935236/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/12","provider":"claude","model":"sonnet"}]
---
# The journal cannot price a prompt change, because it mirrors only two usage fields

> Minted 2026-09-13. Blocks any honest cost comparison of the
> `systemPrompt: "preset"` flip landed the same day.

## Evidence

`SdkMessage`'s result usage mirror (`src/pipeline/nodes/build.ts`) carries only:

```ts
readonly usage: {
  readonly input_tokens: number;
  readonly output_tokens: number;
};
```

The SDK's own result usage carries `cache_read_input_tokens` and
`cache_creation_input_tokens` (`sdk.d.ts:3072-3073`). Neither reaches the
journal, so `just usage <run-id>` reports a number that omits most of what a
long agent session actually costs.

This is not academic. A one-turn probe measured the preset's delta as
**+7 768 tokens landing in `cache_creation`** — a cached prefix, not fresh
input per turn. The previous handoff's "+7 836 tokens **every turn**" framing,
and the rate-limit argument built on it, could neither be confirmed nor refuted
from any of the 55 banked journals.

## Requirements

- [ ] Widen the `SdkMessage` result-usage mirror to carry
      `cache_read_input_tokens` and `cache_creation_input_tokens`.
- [ ] Thread them onto the node-end `usage` the engine journals, beside
      `tokens`, `turns`, `sessionId` and `systemPrompt`.
- [ ] `just usage` reports them.
- [ ] Absent fields (a provider that does not report cache usage — codex) must
      not break the mirror or the journal schema.

## Verify

- A unit test over `consumeAgentStream` pins the new fields through to the
  returned `usage`.
- `just usage <run-id>` on a fresh run shows non-zero cache reads.

---

## Superseded — 2026-09-13, the same day it was minted

Retired as **rejected: superseded**, not as done.

This ticket was written after measuring that no banked journal could price the
`systemPrompt: "preset"` flip. That observation was correct; minting a ticket
for it was not — `adw-fe-01-journal-schema` already owned the same seam and had
been decomposed from `specs/adw-v1.2-live-view.md` a day earlier. Its
requirement 3 reads:

> `AgentUsage` carries the usage **components** (input, output, cache read,
> cache write) — not only their sum.

That is a superset of this ticket's two flat fields. The FE chain
(`fe-02`, `fe-03`, `fe-10`) is written against fe-01's `breakdown` shape.

It was dispatched and merged as PR #12 before the collision was noticed. PR #13
(the fe-01 salvage) removes `cacheReadInputTokens`/`cacheCreationInputTokens`
and keeps `breakdown.{input, output, cacheRead, cacheWrite}` — one
representation (Art. VIII). **No data is lost:** every number PR #12 journaled
is still journaled, under `breakdown`, alongside input/output as components
rather than only their sum.

The lesson is the cheap one: read the open chain before minting into it.
`just next` showed `adw-fe-01` as `in-progress`, which read as "someone is on
it" rather than "stranded, and it already covers this".
