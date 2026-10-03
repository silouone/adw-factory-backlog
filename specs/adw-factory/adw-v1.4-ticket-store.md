# Amendment v1.4 — the ticket store is not the code repo

> **Status:** proposed 2026-09-12, **APPROVED by the operator 2026-09-21** with
> three additions (§10). **Independent of** `adw-v1.3-review-lane.md` — different
> surfaces, no ordering between them.
> **Amends:** `adw-v1.md` §4 (the ticket-state NFR and the crash-safety NFR) and
> §5 (the dirty-ticket-file edge case). **Binding:** `constitution.md` — unchanged.
> **Depends on:** `tickets/adw-cli-01-the-ledger-recipes-ignore-target.md`, which
> ships `ticketsDirOf(config)` — the one seam this amendment widens (§10.4).
> **Implemented by:** `tickets/adw-store-01-tickets-dir.md`.
> **Deliberately excluded:** `specs/` — see §7.

## 1. Trigger

`adw-v1.md` §4 says:

> *"Ticket state shall live in exactly one place — the ticket file in the target
> repo — and every state transition shall be committed, making git history the
> audit log."*

That sentence bundles **three independent decisions** into one clause:

1. Ticket state lives in exactly one place. *(keep — Art. VIII)*
2. That place is inside the target repo's working copy. *(the one under review)*
3. Transitions are git commits, and git history is the audit log. *(consequence of 2)*

Decision 2 was never argued; it was assumed, because the first target's tickets
already lived beside its code. Everything painful downstream follows from it.

The operator's report, 2026-09-12, verbatim:

> *"most of the time I dont commit ticket and or spec because those are tmp
> files."*

Attempting to express that with a `.gitignore` line broke self-dispatch outright
(commit `5423f1c`, reverted in `86494d3`) — which is the proof that the coupling
is structural and belongs in the spec, not in an ignore rule.

## 2. Problem statement

The factory's own backlog is 186 ticket files in the code repo (91 when this was written 2026-09-12). Every dispatch rewrites
a ticket's `status:` and appends to `attempts:` — runIds, attempt branch names,
absolute workspace paths — and **commits each transition to the default branch**.
On this repo that is the single largest source of commit churn, and none of it is
shared source: it is per-machine run state, meaningless in any other checkout.

The operator cannot opt out, because three sites hard-require the tickets to be
tracked git objects at `<repo>/tickets/`:

| Site | What it requires |
|---|---|
| `intake/status.ts:156` (`dispatch`) | `git show HEAD:tickets/<id>.md` — the queued check |
| `cli.ts:1211` (`statusAtHead`) | the same read, on the CI path |
| `intake/repo-commit.ts:181` (`commitTicketFile`) | branch check, dirty check, `git commit -- tickets/<id>.md` |

All four writers — `status.ts::transition`, `attempts.ts::finalizeBlocked`,
`open-pr.ts`, `sync-pr-state.ts::finalizeRejected` — funnel through
`commitTicketFile`. **The whole write path is one function.** That is what makes
this amendment small.

### What the ticket does NOT require

Established by inspection 2026-09-12, and it is the finding that shrinks the
design space:

- **No agent ever reads a ticket file.** `assemble-prompt.ts` inlines the whole
  body into the prompt as `{{ticketBody}}`. No file in `prompts/` references a
  ticket path.
- **No workspace code touches `tickets/`.** Nothing in `src/workspace/` mentions
  them; the worktree, container and sandbox never need the directory to exist.

Tickets are read host-side by the CLI, rendered into a prompt, and written back
host-side. The isolation boundary is not involved, so no "copy the tickets into
the workspace" step is needed — now or ever.

## 3. Decision

**Split *where the ticket lives* from *where the code lands*.**

`TargetConfig` gains one optional field:

```jsonc
{
  "name": "adw-factory",
  "repo": "/Users/silouane/adw-factory",     // code, branches, commits, PRs
  "ticketsDir": "~/adw/backlog/adw-factory", // ticket state — may be anywhere
  …
}
```

- **Absent → `<repo>/tickets`.** Byte-identical to today for every existing
  target. `clens.json`, `sabado.json` and `claude-home.json` are untouched, and
  their runs keep committing ticket transitions exactly as they do now.
- The factory detects **once per run** whether the resolved `ticketsDir` sits
  inside a git working tree, and picks the matching protocol.

### Two protocols, one interface

| | **git store** (dir is inside a work tree) | **plain store** (it is not) |
|---|---|---|
| queued check | `git show HEAD:<rel>` | `readFileSync` |
| transition | rewrite + `git commit -- <rel>` | rewrite + `writeFileSync` |
| concurrency | `O_EXCL` ticket lock + `_repo.lock` | **identical — unchanged** |
| audit log | git history | `journal.jsonl` + the `attempts:` ledger |
| dirty guard | refuse on uncommitted edits | n/a |
| branch guard | must be on `base` | n/a |

Detection is `git -C <ticketsDir> rev-parse --is-inside-work-tree`, evaluated
once and carried on the resolved config. It is never re-probed mid-run, so a
store cannot change protocol underneath a live run.

## 4. What changes in adw-v1.md

**§4, the ticket-state NFR.** Replace:

> ~~Ticket state shall live in exactly one place — the ticket file in the target
> repo — and every state transition shall be committed, making git history the
> audit log.~~

with:

> Ticket state shall live in exactly one place — the ticket file (Art. VIII).
> Its directory is the target's `ticketsDir`, defaulting to `<repo>/tickets`.
> **When that directory is inside a git working tree**, every state transition
> shall be committed, making git history the audit log. **When it is not**, the
> transition is a plain file write, and the audit log is the run journal plus
> the ticket's own `attempts:` ledger — both of which already record every
> transition and are unaffected by this amendment.

**§4, the crash-safety NFR.** Its exception becomes conditional:

> …with one explicit exception: ticket status commits, **which occur only when
> the ticket store is a git working tree**, touch only the ticket file, and are
> themselves the audit log.

Note this makes the NFR *stronger* for a plain store: the target repo's default
branch is then never modified by a run at all.

**§5, the dirty-ticket-file edge case.** Gains a scope clause:

> …if the *ticket file itself* has uncommitted edits, the factory shall refuse
> dispatch until they are committed. **This applies to a git store only**; in a
> plain store "uncommitted" has no meaning and no such refusal exists.

**§5, E7 needs no change.** It already reads *"if two invocations race on the
same tickets directory…"* — store-agnostic as written, and satisfied by the
`O_EXCL` lock, which this amendment does not touch.

## 5. Why E7 survives without git

The queued check moving from HEAD to disk does **not** weaken double-dispatch
protection, and the argument has three parts:

1. **Concurrency is not git's job today.** `status.ts::dispatch` takes an
   `O_EXCL` lockfile at `<locksDir>/<ticketId>.lock` *before* reading HEAD and
   holds it across the whole check-and-write window. Two simultaneous
   invocations are serialized by that lock alone.
2. **Sequential re-dispatch is caught by the status value**, and `in-progress`
   reads the same from disk as from HEAD.
3. **The only state where HEAD and disk disagree is an uncommitted edit** — and
   that is precisely E1, a guard that exists *because* the file is tracked. In a
   plain store there is no committed-vs-working divergence to guard.

The residual difference is that a git store reads a committed snapshot while a
plain store reads whatever is on disk. Recorded as a risk in §8.

## 6. Acceptance

- [ ] A target with no `ticketsDir` behaves byte-identically to today, proven by
      the existing intake, status and repo-commit suites passing **unmodified**.
- [ ] A target whose `ticketsDir` is outside any work tree dispatches, transitions
      and finalizes with zero `git` invocations against the store.
- [ ] A target whose `ticketsDir` is inside a work tree keeps the full git
      protocol: HEAD check, branch guard, dirty guard, one commit per transition.
- [ ] `~` and relative paths in `ticketsDir` resolve deterministically, and a
      `ticketsDir` that does not exist is a **config refusal** (exit 2, no
      dispatch, no tokens) — not a silent empty backlog.
- [ ] E7 holds in a plain store: two concurrent dispatches of one ticket produce
      exactly one `in-progress`, the loser a descriptive refusal.
- [ ] Every terminal outcome still finalizes the ticket. No path leaves
      `in-progress` in either store.
- [ ] `adw status` and `adw clean` read the resolved `ticketsDir`, not
      `<repo>/tickets`.

## 7. Out of scope

- **`specs/`.** The operator's phrasing was "ticket and or spec", but `specs/`
  is *not* symmetrical with `tickets/`. `constitution.md` is binding and is
  referenced from tracked `CLAUDE.md` and the committed `adwf` skill; the spec
  files are reviewed source, not run state, and nothing rewrites them
  mechanically. Untracking them would break documents that govern the repo.
  A separate decision, on its own evidence.
- ~~**Migrating this repo's backlog.**~~ **NOW IN SCOPE — operator decision
  2026-09-21 (§10.3).** A store that is merely *possible* leaves the backlog
  split across N repos, and the backlog tab (`adw-v1.14`) then has N places to
  look. Every existing ticket moves to the central store as part of this work.
- **A ticket store that is neither a directory nor local** — a database, an
  issue tracker, a remote API. `ticketsDir` stays a filesystem path.
- **Per-ticket store overrides.** Store choice is per target, resolved once.

## 8. Residual risks

| Risk | Mitigation |
|---|---|
| A plain store loses git history as a third copy of the audit trail | It is the third, not the first: `journal.jsonl` records every node and outcome, and the ticket's `attempts:` ledger records runId, branch, workspace, outcome and PR. An operator who wants the git copy keeps the default |
| A plain store reads uncommitted disk state, so a half-saved ticket could be dispatched | The ticket parser already rejects malformed frontmatter pre-engine, at zero cost. A partially-written file fails to parse rather than dispatching wrongly |
| Two protocols is two code paths to keep correct | They meet at one function. `commitTicketFile` is the only writer, and the contract suite runs both stores against the same assertions |
| An operator points two targets at one `ticketsDir` | Out of scope, and no worse than today: the `O_EXCL` locks are keyed on the store path, so they serialize correctly regardless |

## 10. Operator additions, 2026-09-21

Approved on the evidence of the **MinIO sweep** (2026-09-15): seven CoorpAcademy
repos each received **two local-only commits** on their default branch — the
ticket and a model pin — diverging local `master` from `origin/master` in seven
professional repos. Nothing reached GitHub (`resolveBranchPoint` cuts from
`origin/<base>`, so the commits are not PR ancestors), so the harm is local
hygiene and an accidental-push hazard, not exposure. That is the trigger.

### 10.1 The store is one repo, namespaced per target

```
~/adw/backlog/
  adw-factory/  clens/  sabado/  coorpacademy/  api-content/  …
```

Each target's `ticketsDir` points at its own subdirectory. Because that tree is
a git work tree, the **git store** protocol of §3 applies unchanged — full
commit-per-transition history, just in the backlog repo instead of the client's.
§4's crash-safety NFR becomes *stronger*: a professional target repo is then
never written to by a run at all.

**Per-target subdirectories are mandatory, not cosmetic.** Ticket ids are not
globally unique and the sweep proves it — all seven repos used the id
`minio-quay-registry-move`. A flat store would collapse them into one file.

### 10.2 Ticket ids gain an optional short uid suffix

`id` already must equal the filename stem (`intake/ticket.ts:180`), and
`ticketId` flows into the runId (`cli.ts:735`), the attempt branch
(`worktree.ts:142`, `e2b.ts:446`, `kill.ts:308`) and the dispatch lock
(`status.ts:135`). So a suffix on `id` propagates uniqueness to all four **with
no new field and no lock re-keying**:

```
id:     minio-quay-registry-move-a3f91c
file:   tickets/minio-quay-registry-move-a3f91c.md
branch: fix/minio-quay-registry-move-a3f91c
runId:  minio-quay-registry-move-a3f91c-1789464200548
lock:   runs/locks/minio-quay-registry-move-a3f91c.lock
```

**Six lowercase hex characters** (16.7M) — ample for a personal factory, and
short enough that `just run <id>` stays typable. A full UUIDv4 was rejected:
36 extra characters in every branch name, PR title and runId, unusable by hand.
A separate `uid:` field was rejected: it creates two identity concepts and
solves neither the filename nor the branch collision.

**Optional, enforced for new tickets only.** The 186 existing tickets keep their
current ids. Renaming them would orphan every `attempts:` record, branch name
and runId that references the old id — historical traceability for no gain,
since per-target namespacing already removes the collision.

### 10.3 Migration is in scope

All 186 existing `adw-factory` tickets move to `<store>/adw-factory/`, and every
other target with a `tickets/` dir moves likewise. The rationale is the backlog
tab: `adw-v1.14` §D2 has the web layer read the ticket store, and *one place to
look* is the whole point. A half-migrated store is worse than either end state.

`tickets/README.md` (the pickup protocol) and `tickets/BACKLOG.md` are **not
tickets** — they are reviewed source, referenced from `CLAUDE.md:5`,
`README.md:362`, `README.md:603` and `.claude/skills/adwf/SKILL.md:70`. They stay
in the code repo. Only the 186 `<id>.md` ticket files move — `tickets/*.md` is 188 entries, two of which are the docs above.

### 10.4 One resolver, not two

`adw-cli-01-the-ledger-recipes-ignore-target` R1 ships `ticketsDirOf(config)` in
`src/targets/loader.ts`, and says of it verbatim: *"This is the **one seam**
`adw-v1.4` may later widen; nothing else resolves a ticket dir by hand."*

This amendment **widens that function**. It does not add a second resolver.
`adw-cli-01` is a queued priority-1 chore with `depends: []` — land it first.

### 10.5 As built (adw-store-01, 2026-09-27)

Four points where the implementation settles what the text above left open,
or departs from it:

- **The uid cutoff is exclusive and sits on the landing date.** §10.2 says
  "new tickets only", and the ticket turned that into "created on or after
  2026-09-21". By the time the rule landed, seven tickets created after
  2026-09-21 carried bare ids, two of them in flight. Enforcing 2026-09-21
  would have made them unparseable, which breaks the "existing ids untouched"
  invariant that §10.2 exists to protect. `parseTicket` requires the
  `-<6hex>` suffix when `created` is **after 2026-09-27**. A test asserts
  that no ticket in the repo's own backlog is refused by the rule.
- **`ticketsDirOf` still returns a path.** It takes
  `{ repo, ticketsDir? }` and returns `ticketsDir ?? <repo>/tickets`.
  `resolveTicketStore(config, probe)` calls it once and adds the probed
  `kind`. For a git store it also adds `gitRoot` and the store's own
  checked-out `branch`. The probe is layered on top of the one directory
  resolver rather than being a second one, and the ledger recipes keep
  consuming a plain string. Any caller that holds only a repo path gets its
  fallback by calling `ticketsDirOf` too, so the literal default appears
  exactly once in `src/`.
- **Which branch the git branch guard requires depends on where the store
  lives.** When the store is inside the target repo itself (compared by
  realpath), it must be on `target.base`, exactly as N6 always required.
  An external git store must stay on the branch it had checked out when it
  was probed. `target.base` names the code repo's branch, not the backlog
  repo's. An external store on a detached HEAD is refused before dispatch.
- **`gitRoot` shares `dir`'s spelling.** It is `dir` walked up by
  `git rev-parse --show-prefix`, not `--show-toplevel`. The latter returns
  the realpath, which puts a symlinked repo's tickets "outside" the repo.
- **A relative `ticketsDir` resolves against `targets/`**, the directory of
  the config file, in the CLI and in the ledger recipes alike.

`targets/adw-factory.json` does **not** gain `ticketsDir` in adw-store-01.
A configured store that does not exist is refused before dispatch (§6), so
pointing at `~/adw/backlog/adw-factory` before that directory exists would
refuse every command against the factory's own target. The config line
belongs to `adw-store-02-migrate-the-backlog`'s runbook, which creates the
directory first.

### 10.6 `depends:` resolves across every target's store (adw-bug-32, 2026-09-30)

A `depends:` id is looked up in the ticket's own store first, then in the
store of every other target under `targets/*.json` (each config's
`ticketsDir`). The uid suffix (§10.2) makes ids unique across stores; a bare
pre-cutoff id that collides resolves to the own store.

- A dependency that is not `done` refuses dispatch, pinned or auto. A
  dependency in another store is reported with its status and owning target.
- A dependency id found in **no** store refuses dispatch and is reported as
  unknown. It is no longer treated as met (this closes
  `adw-depends-enforce` requirement 2).
- A target whose config is invalid, or whose `ticketsDir` does not exist,
  contributes no tickets. It never makes a run fail by itself.
- The same rule applies everywhere a dependency is read, not only at dispatch:
  `just next` and `adw pairs` never offer a ticket with an unmet cross-store
  dependency, `just tickets` flags a dependency id found in no store, and the
  backlog projection resolves cross-store deps (a row's `deps` entry carries
  the owning `target`; the graph shows it as an outside node with that
  target).
- A dependency on an epic/manual/malformed file in the ticket's own store is
  *known* (not unknown) and keeps its existing handling: dispatch and
  `adw pairs` count it as met; `just next` still reads its raw status.

## 9. Change log

- [PROPOSED] 2026-09-12 — split the ticket store from the code repo; make the
  git-commit protocol conditional on the store being a work tree.
- [APPROVED] 2026-09-21 — operator approval with three additions (§10): one
  namespaced central store, an optional 6-hex-char uid suffix on `id`, and
  migration moved from out-of-scope into scope. Resolver is `ticketsDirOf`.
- [AMENDED] 2026-09-27 — as-built notes from `adw-store-01` (§10.5): the
  uid cutoff moves to after 2026-09-27; `ticketsDirOf` returns a path and
  `resolveTicketStore` layers the probe on it; the git branch guard requires
  `target.base` in the default layout and the probed branch for an external
  store; the `adw-factory` `ticketsDir` line moves to
  `adw-store-02`.
- [AMENDED] 2026-09-30 — §10.6 (adw-bug-32): a `depends:` id resolves across
  every target's store; an unknown id refuses dispatch instead of counting as
  met.
