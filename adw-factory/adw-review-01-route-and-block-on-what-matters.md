---
id: adw-review-01-route-and-block-on-what-matters
type: bug
status: done
priority: 1
review: false
created: 2026-09-18
depends: []
attempts: [{"runId":"adw-review-01-route-and-block-on-what-matters-1789756040551","branch":"adw/adw-review-01-route-and-block-on-what-matters","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-review-01-route-and-block-on-what-matters-1789756040551/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/80","provider":"claude","model":"sonnet"}]
---
# A reported bug never reaches the builder, and an absent test kills the run

> `review: false` — the lane being fixed is the lane that would judge the fix
> (same reason `adw-bug-12` and `adw-bug-13` carried it).
>
> **Spec authority:** `specs/adw-v1.12-review-routing.md` §3. Read it first —
> it answers `adw-v1.3` D3, which was listed as an operator decision required
> *before any code* and was never made.

## The defect

`src/pipeline/review/verdict.ts`:

```ts
export function isBlocking(finding: Finding): boolean {
  if (finding.axis === "standards") return finding.severity === "hard";
  return finding.kind === "missing";      // "wrong" and "unasked" -> false
}
```

`src/pipeline/review/fix-loop.ts` renders the fix agent's `{{findings}}` from
`unionBlockingFindings`, which is `…filter(isBlocking)`. So a spec finding of
kind **`wrong`** — the reviewer's own vocabulary for *"this is implemented and
it does not match what the ticket asked"*, i.e. an actual bug — is raised,
journaled, shown on the run screen, and **dropped before the prompt is built**.

The build agent that runs immediately afterwards is never told. It is not
ignoring the finding; it cannot see it.

### Proof, from `adw-fe-24`'s own prompt files

`runs/adw-fe-24-…-1789738858464/prompts/`:

| prompt | findings handed to the fix agent | the `hard`/`wrong` defect present? |
|---|---|---|
| `0006-review-fix.txt` (round 1) | 1 standards + 4 `missing` | **no** |
| `0009-review-fix.txt` (round 2) | 1 standards + 2 `missing` | **no** |

The reviewer named that defect in all three rounds. It shipped. It was found
and fixed by hand during salvage (PR #79).

### Measured across every banked run

| spec finding kind | raised | reached a fix agent |
|---|---|---|
| `missing` | 30 | yes |
| `wrong` | 12 (4 `hard`) | **never** |
| `unasked` | 2 | **never** |

**14 of 44 (32%) discarded, including every report of a real bug.** No `wrong`
finding has ever reached a fix agent in this factory's history.

### The asymmetry

- `missing` (overwhelmingly *"a test is absent"*) — always blocking, and drives
  the round counter to exhaustion.
- `wrong` (*"the code is incorrect"*) — never routed, never fixed.

The factory blocks runs over absent tests while silently discarding reports of
actual defects. Review-lane record to date: **1 green, 9 blocked**.

## Separate the two notions

They are conflated in one predicate today. Split them (v1.12 §3):

- **Routing** — does this finding reach `{{findings}}`?
- **Blocking** — may the run end `blocked` because it survived?

| finding | routes | may block |
|---|---|---|
| `standards`, `hard` | yes | **yes** |
| `spec`, `wrong`, `hard` | yes | **yes** |
| `spec`, `missing` (any severity) | yes | no |
| `spec`, `wrong`, `judgement` | yes | no |
| `spec`, `unasked` | no | no |
| `standards`, `judgement` | no | no |

**The round counter advances only while a blocking finding is outstanding.** A
round whose remaining findings are all advisory ends the loop, and the run
proceeds to `open-pr`.

## Requirements

- [ ] Replace `isBlocking` with two pure predicates — a routing predicate and a
      blocking predicate — matching the table above. Keep them pure (Art. III):
      no I/O, no clock, no randomness.
- [ ] `unionBlockingFindings` (or its successor) feeds the fix agent from the
      **routing** predicate, so a `hard`/`wrong` finding reaches `{{findings}}`.
- [ ] `review-spec`'s retry decision uses the **blocking** predicate. Zero
      blocking findings ⇒ `next`, even when advisory findings remain.
- [ ] Exhausting the rounds is reachable **only** with a blocking finding
      outstanding.
- [ ] `renderFindings` distinguishes blocking from advisory in the prompt, so
      the fix agent knows which objections can end the run.

## Verify

- [ ] Red test: a spec verdict carrying only `{kind:"wrong", severity:"hard"}`
      routes that finding into the fix agent's rendered prompt. **RED today** —
      it is filtered out.
- [ ] Red test: a spec verdict carrying only `{kind:"missing"}` findings does
      **not** block — the node returns `next` and the lane reaches `open-pr`.
      RED today.
- [ ] Red test: `unasked` and `standards`/`judgement` never route and never
      block.
- [ ] Replay: the four salvaged runs' real verdicts (quoted in
      `adw-v1.12` §3) produce the predicted outcomes — `adw-fe-24` blocks;
      `adw-fe-15`, `adw-pr-01` and `adw-fe-22` proceed.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

- Surfacing unresolved findings in the PR body — `adw-review-02`.
- Giving the reviewer memory of its previous round — `adw-review-03`.
- `adw-v1.3` D2 (a reviewer for `chore`) and D4 (reviewer model).
