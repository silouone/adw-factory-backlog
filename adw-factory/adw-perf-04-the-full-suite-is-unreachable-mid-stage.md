---
id: adw-perf-04-the-full-suite-is-unreachable-mid-stage
type: feat
status: done
priority: 1
created: 2026-09-18
review: false
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-perf-04-the-full-suite-is-unreachable-mid-stage-1789764674449","branch":"adw/adw-perf-04-the-full-suite-is-unreachable-mid-stage","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-04-the-full-suite-is-unreachable-mid-stage-1789764674449/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/82","provider":"claude","model":"sonnet"}]
---
# One command is 80% of all tool time, and the prompt telling the agent not to run it does not hold

> **Spec authority:** `specs/adw-v1.13-run-economics.md` §3 **S3**, §5 **D3**.

## Evidence

Across 50 banked runs, tool calls timed from `PreToolUse`/`PostToolUse` deltas
and filtered to journal-attributed sessions:

| category | calls | total | avg | % tool |
|---|---:|---:|---:|---:|
| **FULL SUITE** | **144** | **268.2m** | **111.7s** | **80.3%** |
| targeted test | 469 | 46.2m | 5.9s | 13.8% |
| `tsc --noEmit` | 242 | 9.7m | 2.4s | 2.9% |
| Read + Edit + Grep | 3387 | **2.4m** | 0.0s | 0.8% |

Mean **5.8 full-suite calls per run**. Worst: 12 in one run.

`prompts/feature-build.md` already says — correctly, specifically, with
timings — to use `bun run test:unit` (~21s), and explains that the `gates` node
re-runs the full suite unconditionally afterwards so the agent's own run is
never the verification of record. `adw-bug-06` shipped against this same
behaviour and cut it from ~15/run to 5.8. **It stopped there.** The agent
re-derives "verify my work" every turn; across 400 turns the prompt decays.

Move the constraint out of prose and into the engine.

## Requirements

- [ ] **R1 — a pure classifier.** A new leaf module exporting one function:
      Bash command string → *allow* | *deny with text*. No I/O, no clock, no SDK
      types, no imports from `pipeline/nodes/`.

      | command shape | decision |
      |---|---|
      | `bun test` / `bun run test`, **no path argument** | **deny** |
      | `bun test <path>` / `bun run test <path>` | allow |
      | `bun run test:unit` | allow |
      | anything else | allow |

      A **compound** command containing a bare full-suite invocation
      (`bun run lint && bunx tsc --noEmit && bun test`) must **deny** — that
      exact string appears in the banked captures.
- [ ] **R2 — the deny text teaches.** It names `bun run test:unit`
      (439 tests, ~21s) and `bun test <file>`, and states that `gates` re-runs
      the full suite after the turn ends regardless.
- [ ] **R3 — compose, do not invent.** The agent node receives an **opaque**
      hooks bag it neither constructs nor inspects. `makeStreamWatchdog` already
      *wraps* that bag to add behaviour — use the same precedent: wrap, add one
      `PreToolUse` matcher, forward every other event untouched.
- [ ] **R4 — cLens capture is unaffected.** Every call, including denied ones,
      still reaches the capture sink. A denied call must be visible in the
      transcript.
- [ ] **R5 — agent stages only.** `gates`, `red-check` and `baseline` do not
      route Bash through the agent and must be unaffected. The `gates` node
      still runs the real full suite — **confirm this rather than assuming it.**

## Order of work — Article I

1. Write the classifier's table-driven test against the not-yet-existing module,
   including the compound-command case. Confirm **red**.
2. Write the composition test: wrapping preserves every other event's
   forwarding. Confirm **red**.
3. Implement.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] Red output for both tests pasted into this ticket (Art. I).
- [ ] Classifier cases cover every row of R1 **plus** these exact strings seen
      in banked captures: `bun test 2>&1 | tail -20`,
      `bun run lint && bunx tsc --noEmit && bun test`,
      `bun test test/intake/select.test.ts`, `bun run test:unit`.
- [ ] A test proves the `gates` node still executes the full suite.
- [ ] After this lands, one real run's scorecard shows **suite calls ≤ 1**:
      `bun scripts/run-scorecard.ts <runId>` — pasted into this ticket.

## Out of scope

- Making the suite itself faster.
- Any other command. This is not Bash-subcommand whack-a-mole —
  `adw-tools-01` explicitly ruled that out. One classifier, one command family.
- Rewriting the prompt. The prompt is already correct.
