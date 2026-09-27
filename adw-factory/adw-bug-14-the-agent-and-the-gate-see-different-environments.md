---
id: adw-bug-14-the-agent-and-the-gate-see-different-environments
type: bug
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 90, turns: 450}
depends: []
attempts: []
---
# A gate failure the repair agent cannot reproduce burns all three rounds and blocks, every time

> Observed 2026-09-19 on `adw-render-01-foundation-1789771519506`. Salvaged by
> hand as PR #85 after the run blocked.

## What happened

`gates` failed on 4 tests. `repair` ran three times. Each round the agent ran
the suite, saw **`4 skip, 0 fail`**, investigated thoroughly, and reported —
correctly, for its own process — that it could not reproduce the failure.
Round 3's summary:

> *"No change in conclusion — this is the same failure I already investigated
> and ruled out last round… `E2B_API_KEY` is still unset in this session, and
> `test/workspace/e2b.contract.test.ts` still skips cleanly."*

The agent was not lazy. It drove the real `e2b` SDK through `provisionE2b`
with a fake key and got a clean 401. It was **structurally unable** to see the
defect.

## The defect

**The gate process and the agent process have different environments.**

- `sanitizeAgentEnv` (`src/env-policy.ts`) strips `/(_TOKEN|_SECRET|_KEY|_PASSWORD|_PASSWD)$/i`
  before the agent child inherits. `E2B_API_KEY` matches `_KEY$`.
- The `gates` node runs the target's gate command **without that scrub**, so it
  inherits the operator's `E2B_API_KEY`.
- `test/workspace/e2b-gate.ts` gates the suite on the key's *presence*.

So the same command is a different test run in each process: 4 skips for the
agent, 4 live-network failures for the gate. Nothing in the repair prompt says
so, and the agent has no way to discover it — the variable is invisible to it
**by design**.

This is not specific to E2B. **Any** gate outcome that depends on a stripped
credential is unreproducible by repair, and will always burn all 3 rounds and
land `blocked`.

## Requirements

- [ ] **R1 — name the gap in the repair prompt.** When the gate ran with env
      keys the agent will not receive, the repair prompt must say so and list
      the stripped key **names** (never values). "4 tests failed that you
      cannot run" is actionable; silence is not.
- [ ] **R2 — a pure diff function.** Given the gate's env key set and the
      agent's, return the names present for one and not the other. Pure, no
      I/O, no values.
- [ ] **R3 — journal it.** The divergence is recorded on the gates event, so
      `just fails` can answer "why couldn't the agent see this" from the
      journal alone.
- [ ] **R4 — no values, ever.** Only key names reach the prompt, the journal
      or a span. A test must pin that a value can never be rendered.
- [ ] **R5 — do not widen the agent's env.** Giving the agent the credential
      would "fix" reproducibility by handing a coding agent live cloud keys.
      `adw-m6-03` stripped them deliberately. The agent stays blind; it just
      stops being blind *silently*.

## Order of work — Article I

Write the pure-diff test first (R2), then the prompt-rendering test (R1/R4),
confirm both red, then implement.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] Red output for both tests pasted into this ticket (Art. I).
- [ ] A test proves a stripped key's **value** never appears in prompt, journal
      or span output.
- [ ] A test proves the repair prompt names `E2B_API_KEY` when the gate had it
      and the agent did not.
- [ ] Identical environments render **no** divergence section — no empty
      heading (the N5 honesty rule).

## Out of scope

- Changing `SENSITIVE_ENV_KEY`.
- Making the E2B suite pass. PR #85 addresses its actual cause.
- Reproducing gate failures in the agent's process generally.
