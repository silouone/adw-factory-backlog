---
id: adw-bug-37-reviewers-block-on-what-they-cannot-see-3b5500
type: bug
status: queued
priority: 1
created: 2026-10-02
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# Reviewers open fix rounds over things they cannot see: gates the engine already ran, and how the builder worked

## The bug

On the cmc target a `review-fix` round costs **10–20 minutes**: a fresh build session plus
the in-process docker gates. At least three of the rounds measured below were opened by
findings that are **not about the diff**:

| Run | Finding (verbatim summary) | Severity | What it really is |
|---|---|---|---|
| `cqc-fe-19-…-1790782788355`, review-spec round 2 | "The available report shows Jest passing, but the recorded Docker lint and build commands could not access Docker, and the dev server could not bind. Those required gates remain unverified." | spec / missing / **hard** | The reviewer's **own sandbox** has no Docker. The engine's `gates` node had already run lint, test and build **green** before review started. |
| `cqc-fe-26-…` (green run), review-fix standards re-check, rounds 1 and 2 | "The new Go1d View and Text composition was implemented without inspecting the matching Go1d source revision. The fix agent states that direct access was unavailable…" | standards / **hard** | A finding about the **builder's process** (what it read), lifted from the fix agent's own message. Nothing in the diff is cited. |
| same | "The SCO count … use String(length), so large counts lack the required thousands separators." | standards / hard | A real diff finding: kept as a control, it must still route. |

fe-19 then spent its second round fixing the non-defect and blocked when that round's
gates flaked (127 min total, `blocked`). fe-26 went green after 2 rounds (69 min); round 2
was opened again by the same process finding.

## Why it happens (read before fixing)

- `prompts/review-spec.md` has **no** rule against duplicating tooling.
  `prompts/review-standards.md:82-84` has one ("Do not duplicate what tooling already
  enforces"), but only says *gates exist*. Neither prompt receives the **outcome** of the
  run's `gates` node, so a reviewer that cannot run docker concludes "unverified".
- Neither prompt says that the builder's research process, or the reviewer's own tool
  limits, are out of scope. `priorReviewContext` (adw-review-03) feeds the fix agent's
  verbatim message to the next reviewer, so a remark like "direct access was unavailable"
  becomes a citeable "violation".
- `isBlocking` (`src/pipeline/review/verdict.ts`) routes any `hard` finding, and the retry
  decision unions both axes (`unionBlockingFindings`, `fix-loop.ts:114`). One stray hard
  finding on either axis buys a full round.

## The fix

1. **Give both reviewers the gate outcome.** Render the most recent `gates` result of this
   run (name, command, pass/fail per gate) into the review prompts through a new slot,
   e.g. `{{gateResults}}`, with an explicit line: "These gates were run by the engine, outside
   your sandbox, on this exact diff. Treat them as verified." Round 1 and every fix round use
   the latest result. When no gates ran, the slot says so explicitly (no silently empty slot,
   same N5 rule as `renderFindings`).
2. **One shared out-of-scope rule in both review prompts:**
   - Do not report a gate, test, lint, build or dev-server requirement as unverified when
     `{{gateResults}}` shows it ran; and never report your own inability to run a command.
   - Review the **diff**, not the builder's process: what files it read, which sources it
     checked, what it says it could not access. A finding must cite a location in the diff
     (`file`, and `line` when applicable).
3. Do **not** change `isBlocking`, severities or the round ceiling here. Routing is
   `specs/adw-v1.12-review-routing.md`'s and changing it needs an amendment.

## Acceptance criteria

- [ ] Red tests first:
  - assembling the review-spec and review-standards prompts with a green `gates` result in
    `ctx` renders every gate name and `pass` into the prompt; with no gates result it renders
    the explicit "no gates have run" line;
  - both committed prompt templates contain the out-of-scope rule (assert on the template
    text, as the existing prompt-contract tests do);
  - a review-fix round passes the **latest** gates result (the in-process re-check's), not
    the pre-review one.
- [ ] The fe-26 control finding ("thousands separators") is unaffected: routing logic untouched.
- [ ] `just verify` green.

## Verify (operator)

On the next cmc run that reaches review, `jq` the review-spec / review-fix findings:
no finding mentions Docker access, the dev server, or what the builder read.
