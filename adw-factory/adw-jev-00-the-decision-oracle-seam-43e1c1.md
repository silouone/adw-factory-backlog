---
id: adw-jev-00-the-decision-oracle-seam-43e1c1
type: feat
status: in-progress
priority: 2
created: 2026-09-28
caps: {minutes: 180, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# The decision-oracle seam: a typed, journaled, vendor-agnostic port that no node consults yet

> First of the `adw-jev-*` group (`ai_docs/2026-09-28-jev-report-2-factory-coupling.md`
> §5, phase 0; verdict in `…-report-3-assessment.md`). Zero live calls in the
> factory's own runs. Reviewed: the port's shape is judgement-shaped work, so
> `review:` stays on.

## Why this exists

Jev (TypeSafe AI, `jev-1.13.0`) is a decision model: a state plus typed
questions in, a probability distribution per question out, ~300 ms, output
free, no text. Independent tests make it a strong *judge on captured
artefacts* (500/500 vs human pass/fail labels, 92–913× less variance than an
LLM judge) and a usable *yes/no diff gate* (ROC-AUC 0.89–0.99, but per-repo
thresholds). They also show it is flipped by prose in its state, cannot
abstain, and mis-calibrates Score questions out of distribution.

Every later use in the factory (the acceptance audit, the review↻fix
convergence probe, the failure-class reader) needs the same four things
before any of them may run in shadow: one typed port, one fake for tests,
one byte-exact record of every call, and a vendor that is a config line.
This ticket builds exactly those four and stops. **No node calls the port
in this ticket**, so Art. III is untouched: no decision moves from code or
agent to the oracle here. The amendment that would let one do so
(Art. III-bis, report 2 §6) is the operator's to approve and is not this
ticket's work.

The only existing call seam is `AgentQuery` (`build.ts:407`): a streamed
agent session. A single-shot typed decision does not fit it, nor the profile
registry (`profiles.ts`), so the port is new, injected the way `query` and
`codexQuery` are (`LaneDeps`, `shared.ts:77`; `cli.ts:379-386`).

## Requirements

- [ ] **R1** `src/decision/decision-query.ts` exports the port and its
      types, pure data, strict:
      - `DecisionQuestion` = `{ type: "noul", instructions }` |
        `{ type: "choice", instructions, options: Readonly<Record<string,string>> }` |
        `{ type: "score", instructions, levels: readonly string[] }`.
      - `DecisionRequest` = `{ state: DecisionState, questions: Readonly<Record<string, DecisionQuestion>> }`,
        where `DecisionState` is `Readonly<Record<string, string | number | boolean | readonly string[]>>`
        (flat, so a caller cannot smuggle a transcript object in by accident).
      - `DecisionAnswer` per type: noul → `{ p: number }`; choice →
        `{ pick: string, probabilities: Readonly<Record<string, number>> }`;
        score → `{ level: string, probabilities: Readonly<Record<string, number>>, expected: number }`.
        Probabilities are validated in [0, 1]; choice/score keys must equal
        the request's option/level keys, or the adapter throws.
      - `DecisionQuery = (req: DecisionRequest, ctx: { ticketId: string; node: string }) => Promise<DecisionResponse>`
        with `DecisionResponse = { model: string, answers: Readonly<Record<string, DecisionAnswer>>, latencyMs: number, inputTokens?: number }`.
- [ ] **R2** Two pure helpers beside the types. `canonicalRequest(req)`
      returns the request with question keys and choice/score option keys in
      sorted order (option order moves Jev's numbers; sorting pins it).
      `wordingHash(req)` is the sha256 of the canonical questions **only**
      (no state), the same primitive as `hashPrompt` (`prompt-sink.ts:42`), so
      two runs asking the same questions share a hash regardless of state.
- [ ] **R3** `makeTypeSafeDecisionQuery({ apiKey, model, fetch })` in
      `src/decision/typesafe.ts` maps a `DecisionRequest` to
      `POST https://api.typesafe.ai/v1/systemone` (bearer auth, `model`
      pinned to the literal passed in; the default the CLI passes is
      `jev-1.13.0`, never an alias). The request and response shapes are
      pinned by a recorded fixture, `test/fixtures/decision/typesafe-v1.json`,
      captured once by the operator from one real call (see Verify). A non-2xx
      status, a missing answer, an unknown option key or an out-of-range
      probability throws an `Error` whose message carries `ticketId`, `node`,
      the wording hash and the HTTP status (Art. IX, fail fast).
- [ ] **R4** `makeFakeDecisionQuery(fixtures)` in
      `test/fake-decision-query.ts`: keyed by wording hash, returns the
      fixture's answers, records every call. This is the binding every test
      in later `adw-jev-*` tickets uses; it must not touch the network.
- [ ] **R5** Every call is recorded byte-exact, in two places, by a
      `recordedDecisionQuery(inner, sink)` wrapper (the same middleware shape
      as the prompt sink, `prompt-sink.ts`): the journal gains an optional
      event `{ type: "decision", node, wordingHash, model, questions: readonly { key, type }[], answers, latencyMs, inputTokens?, sidecar }`,
      and `runs/<runId>/decisions/<seq>-<node>.json` holds the canonical
      request (state included) and the raw vendor response, owner-only like
      `prompts/`. Journal events are optional on `JournalEvent` so every
      banked journal still parses.
- [ ] **R6** `LaneDeps` gains `decide?: DecisionQuery` next to `query`
      (`shared.ts:84`). `cli.ts` constructs the TypeSafe adapter, wrapped by
      R5, **only when** `TYPESAFE_API_KEY` is set, and leaves `decide`
      undefined otherwise. With the variable unset every node, prompt and
      journal line is byte-identical to today (pin with a test).
<<<<<<< HEAD
- [ ] **R7** `TYPESAFE_API_KEY` is added to the credential list
      `sanitizeAgentEnv` strips (`test/live-query.test.ts` names the
      contract), so the key never reaches an agent child process.
=======
- [ ] **R7** `TYPESAFE_API_KEY` never reaches an agent child process.
      `SENSITIVE_ENV_KEY` (`src/env-policy.ts:12`) already strips any
      `_KEY`-suffixed name, so this is a pinning test on `sanitizeAgentEnv`
      naming that variable, not a list edit; the CLI reads the key at its own
      edge (R6) and hands the adapter a closure, never the env.
>>>>>>> 48f68a0 (operator: mint adw-jev-00 (decision-oracle seam, phase 0 of the Jev plan))
- [ ] **R8** No node reads `deps.decide` in this ticket. A test asserts that
      a full fake lane run with `decide` set makes zero calls on the fake.

## Verify

- [ ] Red first (Art. I).
- [ ] `canonicalRequest` on a request with shuffled option keys equals the
      sorted one; `wordingHash` is unchanged when only `state` changes and
      changes when one question's wording changes.
- [ ] The TypeSafe adapter, fed the recorded fixture through a fake `fetch`,
      returns typed answers; fed a 500, an unknown option key and a
      probability of 1.2, it throws with `ticketId`, `node`, hash and status
      in the message.
- [ ] Operator-run, once, off the factory path (about $0.00002):
      `bun run scripts/decision-probe.ts` with `TYPESAFE_API_KEY` set sends
      one three-question request (one noul, one choice, one score) on a
      fixed toy state and writes `test/fixtures/decision/typesafe-v1.json`.
      The file is committed; the script is not run by `bun run test`.
- [ ] A fake lane run with `decide` set: zero calls recorded (R8). The same
      run with `TYPESAFE_API_KEY` unset: journal identical to a run before
      this ticket, modulo timestamps.
- [ ] `runs/<runId>/decisions/` is created with mode 0700 and one file per
      call when the recorded wrapper is exercised directly in a test.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Out of scope

Any node consulting the oracle (adw-jev-01 acceptance audit, adw-jev-02
backfill, adw-jev-03 convergence probe, adw-jev-04 failure reader). Per-target
calibration files. The Art. III-bis amendment text (operator). A local
logit-readout adapter: the port is shaped so one can be added as a second
`make*DecisionQuery` without touching callers, and that is all this ticket
promises about it.
