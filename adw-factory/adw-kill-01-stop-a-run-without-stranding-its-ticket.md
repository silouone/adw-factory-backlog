---
id: adw-kill-01-stop-a-run-without-stranding-its-ticket
type: feat
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-kill-01-stop-a-run-without-stranding-its-ticket-1789460216127","branch":"adw/adw-kill-01-stop-a-run-without-stranding-its-ticket","workspace":"/Users/silouane/adw-factory/runs/adw-kill-01-stop-a-run-without-stranding-its-ticket-1789460216127/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/50","provider":"claude","model":"sonnet"}]
---
# Ctrl+C strands the ticket — there is no way to stop a run and leave the ledger honest

> Operator, 2026-09-15: *"how to 'kill' a ticket? we don't have anything, I
> had to kill the run."* They did, and it cost a hand-edit.

## 1. What happens today

`Ctrl+C` on `just run <id>` kills the process. Nothing else happens:

- the ticket keeps `status: in-progress`, **which blocks re-dispatch** — the
  operator must hand-edit the file, and did (commit `tickets: adw-m9-04 back
  to queued — the killed run left it stranded`);
- **no attempts entry is written**, so the ledger has no record the attempt
  ever occurred;
- **no `abort` event is journaled**, so the run's own artifacts cannot say why
  it ended — `just outcome <runId>` prints nothing at all;
- the workspace is left behind, which is correct and by design.

This directly violates the engine's own stated invariant: *"no terminal path
leaves a ticket `in-progress`"* (README, Bounded autonomy). Ctrl+C is a
terminal path. It just is not one the engine owns.

`adw clean --ticket <id>` is **not** this. It reclaims the workspace and never
touches status — and reclaiming the workspace is usually the opposite of what
you want after killing a run, because the work is in there (`adw-fe-14`'s
salvage, PR #46, was pulled out of exactly such a workspace).

## 2. The command

```
adw kill --target <name> --ticket <id>     # `just kill <id>`
```

- [ ] Finds the ticket's in-flight run — the newest `runs/<ticketId>-*` with a
      `run-start` and no `run-end`.
- [ ] **Journals an `abort` event** naming the operator as the cause. `abort`
      is already one of the ten event types and the engine already emits it on
      the external-abort path; reuse it (Art. VIII), do not add an eleventh.
- [ ] **Writes a `run-end` with `outcome: "blocked"`** and a reason that says
      it was killed by the operator. `blocked` is the single terminal failure
      outcome; a killed run is not a new category.
- [ ] **Finalizes the ticket via `finalizeBlocked`** (`src/intake/attempts.ts`)
      — the existing function, which already writes the attempts entry and
      commits through the repo lock. **Do not hand-roll a status rewrite:**
      bypassing `commitTicketFile` is what `adw-par-01-repo-commit-lock`
      exists to prevent, and a `sed` on the frontmatter would skip the
      attempts entry entirely.
- [ ] **Leaves the workspace alone.** Killing is not cleaning. Print the
      workspace path so the operator can inspect or salvage it, and tell them
      `just clean-ticket <id>` reclaims it when they are done.
- [ ] **Kills the process if one is still running.** The common case is the
      operator already pressed Ctrl+C, so the command must work fine when
      there is no process left — that is a reconcile, not an error.

## 3. Refusals, before anything is written

- [ ] A ticket that is not `in-progress` → refuse, exit 2, say what its status
      actually is. Nothing to kill.
- [ ] No run directory for that ticket → still finalize the ticket (that is
      the stranded case the operator hit), and say plainly that no run was
      found to journal against.
- [ ] `--dry-run` reports what it would do and mutates nothing, matching
      `adw sync --dry-run`'s precedent.

## 4. Red tests

- [ ] An `in-progress` ticket with a live run directory → after `kill`, the
      ticket is `blocked` with an attempts entry, the journal carries `abort`
      then `run-end{outcome:"blocked"}`, and the workspace still exists.
- [ ] An `in-progress` ticket with **no** run directory → still finalized,
      no throw.
- [ ] A `queued` / `done` ticket → refused, exit 2, nothing written.
- [ ] Two concurrent kills of different tickets do not race the ticket-file
      commit (the `adw-par-01` lock ordering holds).
- [ ] Red before any fix.

## Verify

- [ ] Kill a real run mid-`plan`; `just status` shows `blocked` with an
      attempt, `just outcome <runId>` explains itself, and the ticket
      re-dispatches without a hand-edit.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Pausing or resuming a run. Killing by runId rather than ticket (the ticket is
the operator's unit; add it later if a run without a live ticket ever needs
stopping).

## Note

`adw-finalize-01` fixed `finalizeBlocked` throwing and stranding its ticket —
the same *symptom* from the engine side. This is the operator-initiated path,
which no code owns at all today.
