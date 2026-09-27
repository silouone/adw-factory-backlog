---
id: adw-store-01-tickets-dir
type: feat
status: done
priority: 1
created: 2026-09-12
caps: {minutes: 180, turns: 600}
depends: [adw-cli-01-the-ledger-recipes-ignore-target]
attempts: [{"runId":"adw-store-01-tickets-dir-1789345227206","branch":"adw/adw-store-01-tickets-dir","workspace":"/Users/silouane/adw-factory/runs/adw-store-01-tickets-dir-1789345227206/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-store-01-tickets-dir-1790448708493","branch":"adw/adw-store-01-tickets-dir","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-store-01-tickets-dir-1790448708493/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# Split the ticket store from the code repo (`ticketsDir`)

> **APPROVED 2026-09-21.** `specs/adw-v1.4-ticket-store.md` carries the operator's
> approval and three additions in its §10 — a namespaced central store, an optional
> uid suffix on `id`, and migration moved into scope. Read §10 before starting; the
> requirements below (R6–R8) implement it.
>
> **Land `adw-cli-01-the-ledger-recipes-ignore-target` first.** It ships
> `ticketsDirOf(config)`, the one seam this ticket widens (R2). It is a queued
> priority-1 chore with no dependencies. `depends:` is advisory — the factory
> parses it but does not gate dispatch on it — so this ticket *can* be triggered
> first; it would just have to create `ticketsDirOf` itself and risk a conflicting
> second resolver.

## Why

The factory's backlog is **186 ticket files** (counted 2026-09-21) in its own code repo, and every dispatch
commits a `status:` rewrite plus an `attempts:` append to the default branch.
That is per-machine run state — runIds, attempt branches, absolute workspace
paths — and it is the largest source of commit churn here.

The operator cannot opt out today. Expressing "tickets are tmp files" as a
`.gitignore` line broke self-dispatch outright (`5423f1c`, reverted in
`86494d3`): three sites hard-require the tickets to be tracked git objects.

Full argument, and the exact spec deltas, in `specs/adw-v1.4-ticket-store.md`.

## The finding that bounds the scope

**No agent ever reads a ticket file**, and **no workspace code touches
`tickets/`**. `assemble-prompt.ts` inlines the body as `{{ticketBody}}`; nothing
in `prompts/` names a ticket path; `src/workspace/` never mentions the directory.

Tickets are read host-side, rendered into a prompt, written back host-side. The
isolation boundary is not involved. **Do not add a step that copies tickets into
a workspace** — it would be dead code.

## Scope — one field, one seam

`TargetConfig` gains optional `ticketsDir`. Absent → `<repo>/tickets`, which
must be byte-identical to today. The factory resolves the directory once, probes
once whether it is inside a git working tree, and picks the protocol:

- **git store** — today's protocol, unchanged: `git show HEAD:<rel>` queued
  check, branch guard, dirty guard, one commit per transition.
- **plain store** — `readFileSync` queued check, `writeFileSync` transition, no
  branch guard, no dirty guard. The `O_EXCL` locks are **unchanged in both**.

## Requirements

### R1 — config (`src/targets/loader.ts`)

- [x] `ticketsDir?: string` on `TargetConfig`; added to `KNOWN_FIELDS` so it is
      not rejected as an unknown field (N4).
- [x] Validated as a non-empty string when present; wrong type joins the same
      single-pass error list as every other field.
- [x] `~` expands to the home directory; a relative path resolves against the
      **config file's** directory, not the cwd — `cli.ts` is already anchored to
      the factory root and must stay cwd-independent.
- [x] The resolved absolute path is exposed on the parsed config. Callers never
      re-derive `join(repo, "tickets")`.

### R2 — one resolver, one probe (**widen `ticketsDirOf`, do not add a second**)

- [x] **`adw-cli-01` R1 ships `ticketsDirOf(config)` in `src/targets/loader.ts`,
      returning `<repo>/tickets`, and names itself "the one seam `adw-v1.4` may
      later widen".** Widen *that* function to return the configured
      `ticketsDir`. If `adw-cli-01` has not landed, create `ticketsDirOf` with
      the same name and signature — never a parallel resolver under a new name.
- [x] It resolves `(target) → { dir, kind: "git" | "plain" }`.
      `kind` is decided by `git -C <dir> rev-parse --is-inside-work-tree`.
- [x] Probed **once** per CLI invocation and carried; never re-probed mid-run,
      so a store cannot switch protocol under a live run.
- [x] A `ticketsDir` that does not exist is a **refusal before dispatch**
      (`EXIT_REFUSED`, no workspace, no tokens), naming the resolved path. An
      empty-but-present directory is legal — that is just an empty backlog.
- [x] Pure and injectable: the probe is a parameter, so tests drive both kinds
      with no git (Art. I, Art. IX).

### R3 — the write path (`src/intake/repo-commit.ts`)

- [ ] `commitTicketFile` takes the resolved store, not `repoPath`. The relative
      path stops being the literal `tickets/<id>.md`.
- [x] **git store:** behaviour byte-identical to today — acquire `_repo.lock`,
      branch check, dirty check, then per step rewrite + pathspec-limited commit.
- [x] **GAP (found 2026-09-21) — the branch guard must be decoupled from
      `target.base`.** `repo-commit.ts:188` reads
      `if (branch !== deps.defaultBranch) throw deps.onWrongBranch(branch)`, and
      `defaultBranch` is `target.base` — the **code** repo's base, which varies
      per target (`master` ×5, `main`, `develop` across the CoorpAcademy targets
      alone). A central backlog repo sitting on `main` while serving a
      `master`-based target would trip `onWrongBranch` **spuriously on every
      transition**. The guard must compare against the *store's* own branch
      identity, not the code repo's. "Byte-identical to today" holds only in the
      default case, where store and code repo are the same repo — say so
      explicitly rather than leaving it implied.
- [x] A test proves it: a git store on branch `main` serving a target whose
      `base` is `master` transitions successfully.
- [x] **plain store:** acquire the **same** `_repo.lock`, skip both guards, then
      per step rewrite + write. Multi-step callers (open-pr) still run every step
      under one acquisition.
- [x] Lock ordering is untouched: per-ticket lock OUTER, `_repo.lock` INNER. The
      reentrancy corollary in that module's header still holds — do not
      reintroduce a nested `commitTicketFile` call.
- [x] All four writers keep their own verbatim refusal wording via the existing
      `onWrongBranch` / `onDirtyTicket` factories. In a plain store neither
      factory is ever invoked.

### R4 — the read path

- [x] `src/intake/status.ts::dispatch` — the queued check reads HEAD in a git
      store, disk in a plain one. The `O_EXCL` ticket lock is acquired **before**
      the read in both, exactly as now (E7).
- [x] `src/cli.ts::statusAtHead` — same split. Rename it; it no longer always
      reads HEAD.
- [x] Every `join(repoPath, "tickets")` site takes the resolved dir. Verified
      2026-09-21: `cli.ts:1227` (`readTickets`), `status.ts:50`, `kill.ts:247`,
      `clean.ts:479`, `pairs.ts:151`, `sync-pr-state.ts:326`.
- [x] **GAP (found 2026-09-21) — five sites build the path as a TEMPLATE STRING
      and the old checklist grep misses every one of them:**

      | site | construction |
      |---|---|
      | `cli.ts:1566` | `` `HEAD:tickets/${ticketId}.md` `` — the CI-path queued check |
      | `intake/repo-commit.ts:181` | `` `tickets/${deps.ticketId}.md` `` — the commit pathspec |
      | `intake/status.ts:186` | `` `tickets/${ticketId}.md` `` — `ticketRelPath`, the intended resolver |
      | `pipeline/nodes/open-pr.ts:718` | `` `tickets/${ctx.ticketId}.md` `` |
      | `pipeline/nodes/ci-round.ts:920` | `` `tickets/${ticketId}.md` `` |

- [x] **GAP (found 2026-09-21) — `src/web/server.ts:97` hardcodes**
      `TICKETS_ROOT = join(import.meta.dir, "..", "..", "tickets")` — the
      **factory's own** backlog, regardless of target. The run panel's
      initial-ticket lookup therefore misses for every non-`adw-factory` run
      today (all 23 `sabado` runs, and the 7 CoorpAcademy runs). It must read the
      resolved store. This is a live bug the central store fixes; do not leave it.
- [x] **The verification grep is replaced**, because `grep -rn '"tickets"' src/`
      cannot match a template string and would pass with all five sites intact:

      ```
      grep -rnE '"tickets"|`[^`]*tickets/\$\{|HEAD:tickets/' src/
      ```

      must return **only** the default inside `ticketsDirOf` and `ticketRelPath`.

### R5 — behaviour

- [x] E7 in a plain store: two concurrent dispatches of one ticket yield exactly
      one `in-progress`; the loser gets a descriptive refusal and does **not**
      remove the lock it does not own.
- [x] Every terminal outcome finalizes the ticket in both stores. No path leaves
      `in-progress`.
- [x] A plain store performs **zero** `git` invocations against the store —
      asserted, not assumed, by a fake that throws on any git call.
- [x] Target-repo git (branch, worktree, push, PR) is untouched by this ticket.

### R6 — the uid suffix on `id` (spec §10.2)

`id` must already equal the filename stem (`intake/ticket.ts:180`), and
`ticketId` flows into the runId (`cli.ts:735`), the attempt branch
(`worktree.ts:142`, `e2b.ts:446`, `kill.ts:308`) and the dispatch lock
(`status.ts:135`). A suffix on `id` therefore propagates to all four **with no
new field and no lock re-keying** — that is why this shape was chosen.

- [x] **Six lowercase hex characters**, as a trailing `-<6hex>` segment on `id`.
      `minio-quay-registry-move-a3f91c`. Not a full UUIDv4 (36 chars would make
      every branch name and PR title unusable by hand); not a separate `uid:`
      field (two identity concepts, and it solves neither the filename nor the
      branch collision).
- [ ] **Enforced by an applicable rule, not by convention.** "New tickets only"
      needs a predicate the validator can evaluate, and there is no
      `adw new-ticket` command to hang it on. Use the field that already exists:
      **`parseTicket` REQUIRES the `-<6hex>` suffix when `created:` is on or
      after `2026-09-21`, and accepts a bare id when it is earlier.** All 186
      existing tickets predate the cutoff and keep their ids untouched —
      renaming them would orphan every `attempts:` record, branch name and runId
      pointing at the old id.
- [ ] A suffix-less id with `created: 2026-09-22` is a **parse error**, named
      like every other field error, refused pre-engine at zero cost.
- [x] **This is the part that actually closes the collision.** Without an
      enforceable rule the suffix is decoration: whoever authors the next
      seven-repo wave omits it, and the `minio-quay-registry-move` lock
      collision returns. Do not ship R6 as "the validator accepts both shapes".
- [x] The stem-match rule is **unchanged**: whatever `id` says, the filename must
      match it. The suffix is part of the id, not an addition to it.
- [x] Nothing in the pipeline parses the suffix back out. It is opaque — an
      identity, not a field to read.

**Why it is needed even with per-target namespacing:** `<locksDir>/<ticketId>.lock`
is **factory-global** (`status.ts:135`, `locksDir = runs/locks`). Namespaced
subdirectories fix the *file* collision; they do not fix the *lock* collision.
The MinIO wave had seven runs sharing the id `minio-quay-registry-move` and
therefore one lockfile — it only worked because the lock releases in `finally`
immediately after the check-and-write (`status.ts:177`), and because the operator
staggered the launches. A uid suffix removes the hazard instead of timing around it.

### R7 — make the migration possible; do NOT perform it here

**The move itself cannot be a factory PR, and attempting it strands this run.**

A factory run produces **one** PR in **one** target repo. Moving 186 files from
`adw-factory` to `~/adw/backlog/adw-factory/` is cross-repo: the PR can only
show the deletions, and populating the destination is an out-of-band host-side
effect no PR can contain. If the PR is not merged, the files exist in both
places. Worse — tickets are read and written **host-side** (the scope finding
above), so `open-pr` and `finalizeBlocked` write
`tickets/adw-store-01-tickets-dir.md` in the real repo *during this very run*.
Move the host files and the write target is gone; move them only inside the
worktree and the end state is not achieved. Either way the run strands.

So this ticket ships the **capability**:

- [ ] `targets/adw-factory.json` gains a `ticketsDir` pointing at the central
      store, and every reader resolves through `ticketsDirOf` (R2/R4) — so the
      relocated store *works* the moment the files are there.
- [x] `just next`, `tickets`, `ticket`, `in-progress`, `status` and `adw clean`
      all read the resolved dir. Proven against a fixture store under `/tmp`,
      not against the real backlog.
- [x] **Do not move, delete or `git mv` a single real ticket file.** Any diff
      under `tickets/` other than this ticket's own status transitions is a
      failed run, not a partial success.

The 186-file move is **`tickets/adw-store-02-migrate-the-backlog.md`**, a
`type: manual` operator-executed ticket — the mechanism the repo already has
(`adw-ticket-kind-operator-executed`, `RESERVED_OPERATOR_TYPES = ["manual"]`).
It carries the runbook and is discharged by a human after this lands.

### R8 — the backlog tab reads one store

- [x] `src/web/server.ts:97`'s `TICKETS_ROOT` is replaced by the resolved store
      (R4). With a central store that means the board finally has **one** root
      for every target, which is what `adw-v1.14` §D2/§D3 assume.
- [x] Out of scope here: building the tab itself (`adw-backlog-01`/`02`). This
      ticket only makes the path resolvable.

## Tests first (Art. I — red, reviewed, then green)

- [x] **Default-is-identical:** existing `test/intake/`, `test/targets/` and
      `test/cli.test.ts` pass **unmodified**. Any diff to them is a regression,
      not a test update.
- [x] **Contract suite over both stores.** One assertion set, parameterized by
      kind — mirrors `test/workspace/contract.ts`, which already proves one
      interface across three implementations.
- [x] Resolution table: absent · absolute · `~`-prefixed · relative · missing dir.
- [ ] Probe seam faked both ways; no test shells out to git to decide a kind.
- [x] Plain-store E7 race test.
- [x] Plain-store "no git" test (git fake throws on any invocation).
- [x] Git-store regression: branch guard, dirty guard and commit message
      (`adw: <id> → <status>`, U+2192, no trailer) all still fire.
- [x] **Branch-guard decoupling (R3 gap):** a git store on `main` serving a
      target whose `base` is `master` transitions without tripping
      `onWrongBranch`.
- [x] **uid validator (R6):** accepts `slug-a3f91c` and bare `slug`; still
      rejects any id that does not match its filename stem.
- [x] **No hardcoded path survives (R4 gap):** a test asserts the replacement
      grep returns only the two resolver sites.

## Verify

```
bun run lint && bunx tsc --noEmit && bun test
```

Plus, against a real relocated store:

```
mkdir -p /tmp/adw-store-probe && cp tickets/adw-m4-08-ssh-push-support.md /tmp/adw-store-probe/
# a target config with ticketsDir: /tmp/adw-store-probe
just status                       # lists the relocated ticket
git -C . status --porcelain       # UNCHANGED after a dispatch attempt
```

- [x] `just test-fast` green.
- [x] A dispatch against a plain store leaves `git status` on the code repo
      clean — no status commit, no modified default branch.
- [x] A dispatch against a git store still produces exactly one commit per
      transition, on `base`, touching only the ticket file.
- [x] **End state, proven on a FIXTURE target** (not a real CoorpAcademy clone):
      a full run against a target whose `ticketsDir` is outside its repo leaves
      `git -C <target-repo> status --porcelain` clean and its local default
      branch identical to `origin/<base>` — the exact failure the MinIO sweep
      produced seven times.
- [x] `git status --porcelain tickets/` shows **only** this ticket's own status
      transitions. Any other file touched under `tickets/` fails the run.

## Self-hosting hazard — read before dispatching this at itself

This ticket **rewrites the dispatch protocol that is dispatching it.** A run
against `targets/adw-factory.json` will commit its own `in-progress` through the
very code it is editing, and a half-applied change can strand the ticket
mid-flight. Its previous attempt is already `blocked`
(`adw-store-01-tickets-dir-1789345227206`).

Run it in a worktree (`just run` — the default) and treat a mid-run protocol
break as expected, not surprising. If the lane blocks partway, the ticket may
need its status reset by hand. Dispatching it against a **disposable fixture
target** first is the safer order.

**R7 is what keeps this merely hazardous rather than impossible.** If this run
moved the real ticket files, `open-pr` and `finalizeBlocked` would write to a
path that no longer exists and the run could not finalize at all. The file move
is deliberately a separate `type: manual` ticket
(`adw-store-02-migrate-the-backlog`) for exactly that reason.

## Out of scope

- `specs/` — not symmetrical with `tickets/`; §7 of the amendment says why.
- **Actually moving the 186 files.** Operator decision 2026-09-21 put migration
  in scope *for the project*, but it cannot be a factory PR (R7) — it is
  `adw-store-02-migrate-the-backlog`, `type: manual`.
- Building the backlog tab (`adw-backlog-01`/`02`). R8 only makes the store path
  resolvable for it.
- Renaming the 186 existing tickets to carry a uid. R6 is new-tickets-only by
  decision; a mass rename would orphan every historical `attempts:` record.
- A non-filesystem store (database, issue tracker, remote API).
- Per-ticket store overrides. Store choice is per target, resolved once.
- Any change to the `O_EXCL` locking scheme or the lock-ordering proof.

## Run log

_(empty — not yet dispatched)_
