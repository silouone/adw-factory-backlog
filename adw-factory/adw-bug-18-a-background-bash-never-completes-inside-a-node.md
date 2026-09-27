---
id: adw-bug-18-a-background-bash-never-completes-inside-a-node
type: bug
status: done
priority: 1
created: 2026-09-20
review: false
caps: {minutes: 120, turns: 500}
depends: []
attempts: [{"runId":"adw-bug-18-a-background-bash-never-completes-inside-a-node-1789908523295","branch":"adw/adw-bug-18-a-background-bash-never-completes-inside-a-node","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-18-a-background-bash-never-completes-inside-a-node-1789908523295/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/100","provider":"claude","model":"sonnet"}]
---
# A test command longer than the Bash tool's 600 s cap is silently backgrounded, and the prompt then tells the agent to wait for a notification the factory never sends

> Measured 2026-09-20 on sabado wave 2 — the first five runs with
> `adw-tools-02`'s allowlist, so `pytest` finally executed. Three of five
> blocked within the hour on one shape; the captures below are the evidence.

## What happens today

sabado's backend suite takes **~725 s** (`just test backend`, 2 118 tests).
The SDK's `Bash` tool caps `timeout` at **600 000 ms**. Any longer command
is not failed: it is **silently converted to a background task** and the
tool returns `{"stdout":"","stderr":"","backgroundTaskId":"…"}`. In Claude
Code the harness later re-invokes the model with a completion notification;
inside a factory node there is no later — the query is one turn, and the
node ends when the model stops. Every path from that empty result loses:

| run | what the agent did after the empty result | how it died |
|---|---|---|
| `sabado-18-…-1789903682714` (`test` stage) | ended its turn: *"I'll stop issuing commands now and wait for the background test run's completion notification"* | no `.adw/artifacts/test.md` → `blocked` |
| `sabado-23-…-1789905465452` (`test` stage) | same sentence, same turn end | same |
| `sabado-22-…-1789905462370` (`build` stage) | polled `Bash("true", "Check for background completion notification")` and `until [ -f /private/tmp/claude-502/…/tasks/<id>.output ]` | no-progress detector (adw-auto-02) → `blocked`, with a complete diff and a written `build.md` |
| `sabado-21-…-1789905459432` (`build`) | wrote its own `tests/_run_isolated.py` and `_ensure_isolated_db.py` into the target to run subsets | improvised helpers in the diff (survived, so far) |

None of the three set `run_in_background` themselves (22 did once; 18 and
23 never). They followed `prompts/feature-test.md` §"Iterating cheaply" to
the letter: *"Run any test command in the FOREGROUND with `timeout: 600000`"*
— sized for **this repo's** 300-480 s suite, not the target's — and then its
last bullet, *"If you do background something, wait for its completion
notification — do not poll it"*, which is exactly the instruction that ends
a node with nothing written. The suite guard (adw-perf-04) would have
stopped a bare `bun test`; it knows nothing about `pytest` or `just test`,
so the 12-minute call was allowed in the first place.

## Requirements

- [ ] **R1 — the timeout follows the target.** `targets/<name>.json` may
      declare `bashTimeoutMs`; the factory passes it to the SDK as
      `BASH_MAX_TIMEOUT_MS` (and `BASH_DEFAULT_TIMEOUT_MS`) in the agent's
      env, through `agentSdkEnv`. Absent → the SDK default. `sabado.json`
      declares `1200000`, comfortably over its 725 s suite and under the
      ticket's 25-minute stall watchdog.
- [ ] **R2 — the prompt stops lying.** The "Iterating cheaply" section of
      `feature-build.md` / `feature-test.md` (and the bug/chore prompts if
      they carry it) loses this repo's timings and the "wait for its
      completion notification" bullet. In their place, one paragraph that is
      true for every target: a background task never completes inside a
      stage; an empty result with a `backgroundTaskId` means the command was
      too slow for the cap — re-run a subset in the foreground; the `gates`
      node is the verification of record.
- [ ] **R3 — the suite guard follows the target.** `targets/<name>.json` may
      declare `suiteCommands` (sabado: `["just test", "just test backend",
      "pytest", "pytest -q"]`); `suite-guard.ts` denies a bare one exactly as
      it denies `bun test`, with the same "gates re-runs the whole suite"
      message. `AGENT_ALLOWED_TOOLS`'s bun rules stay.
- [ ] **R4 — journaled.** A guard denial and an SDK auto-background (a
      `PostToolUse` whose response carries `backgroundTaskId`) each land as
      a detail on the agent node, so `just fails` shows them.

## Files

`src/targets/loader.ts` (two fields + validation) · `src/live-query.ts`
(`agentSdkEnv`) · `src/pipeline/suite-guard.ts` ·
`src/observability/suite-guard-hook.ts` · `prompts/feature-build.md` ·
`prompts/feature-test.md` · `prompts/bug-build-fix.md` (if it carries the
section) · `targets/sabado.json` · `.claude/skills/adwf/references/targets.md`.

## Verify

- [ ] Red test (`test/targets/loader.test.ts`): `bashTimeoutMs` must be a
      positive integer; `suiteCommands` a non-empty string array. RED today.
- [ ] Red test (`test/live-query…`): with `bashTimeoutMs: 1200000` the env
      handed to the SDK carries `BASH_MAX_TIMEOUT_MS=1200000`; absent, it
      carries neither key. RED today.
- [ ] Red test (`test/pipeline/suite-guard.test.ts`): with sabado's
      `suiteCommands`, bare `pytest -q` and `just test backend` are denied
      and `pytest tests/test_x.py` is allowed; with no field, `pytest` is
      allowed and `bun test` is still denied. RED today.
- [ ] Red test: no assembled prompt contains "completion notification" or
      "timeout: 600000".
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] **Operator, after merge:** re-fire one sabado `feat` ticket; its
      capture shows no `backgroundTaskId` response and every stage writes
      its artifact.

## Out of scope

Delivering the notification into a node (a multi-turn node is a different
design); speeding sabado's suite (`sabado-23` owns the ratchets); the
`.claude/` write refusal (its own ticket: sabado-11 R3 and sabado-23 R5 both
had to be applied by hand).
