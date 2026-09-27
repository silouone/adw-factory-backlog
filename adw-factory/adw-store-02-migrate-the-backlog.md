---
id: adw-store-02-migrate-the-backlog
type: manual
status: done
priority: 1
created: 2026-09-21
depends: [adw-store-01-tickets-dir]
attempts: []
---
# Move the 186 ticket files into the central backlog store

> **Executed 2026-09-27 by hand** (the runbook below): sabado 16 → `sabado/`, clens 13 → `clens/`,
> adw-factory 202 → `adw-factory/`; all three targets carry `ticketsDir`. claude-home has no
> tickets dir. The last file, `adw-skills-01-…-eca3a0.md`, moved after its run went green (17:10Z). `type` restored to `manual`
> (a wip commit had set `chore`; the "Why this is not a factory run" section still holds).

> **`type: manual` — operator-executed.** `RESERVED_OPERATOR_TYPES = ["manual"]`
> (`src/intake/ticket.ts:94`), so `parseTicket` refuses it by design with
> *"operator-executed, not agent-executed — a human discharges it directly"*.
> It will NOT appear in `just next` or `just tickets`; that is correct, not a bug.
> Spec: `specs/adw-v1.4-ticket-store.md` §10.3.

## Why this is not a factory run

A factory run produces one PR in one repo. This is a cross-repo move: 186
deletions here, 186 additions in `~/adw/backlog/`. A PR can show only the
deletions, and the destination population is a host-side effect no PR contains.

Worse, tickets are read and written **host-side** during a run. A run that moved
its own ticket file out from under `open-pr`/`finalizeBlocked` would strand
itself. `adw-store-01` ships the capability; a human moves the files.

## Precondition

`adw-store-01-tickets-dir` is **merged**, so `ticketsDirOf` resolves
`targets/*.json`'s `ticketsDir` and every reader goes through it.

## Runbook

```sh
STORE=~/adw/backlog
mkdir -p $STORE && git -C $STORE init -q 2>/dev/null || true
mkdir -p $STORE/adw-factory

cd ~/personal_project/adw-factory

# 1. the 186 ticket files ONLY — README.md and BACKLOG.md stay put
ls tickets/*.md | grep -vE '/(README|BACKLOG)\.md$' | wc -l      # expect 186
ls tickets/*.md | grep -vE '/(README|BACKLOG)\.md$' \
  | xargs -I{} cp {} $STORE/adw-factory/

# 2. history lives in the DESTINATION repo, not by rewriting this one
git -C $STORE add adw-factory && \
  git -C $STORE commit -qm "adw-factory: import 186 tickets from the code repo"

# 3. only now remove them here
ls tickets/*.md | grep -vE '/(README|BACKLOG)\.md$' | xargs git rm -q
git commit -qm "adw: move the backlog to the central ticket store (adw-store-02)"
```

Then point the target at it:

```jsonc
// targets/adw-factory.json
"ticketsDir": "~/adw/backlog/adw-factory"
```

## What must NOT move

`tickets/README.md` (the pickup protocol) and `tickets/BACKLOG.md`. They are
**not tickets** — they are reviewed source, referenced from `CLAUDE.md:5`,
`README.md:362`, `README.md:603` and `.claude/skills/adwf/SKILL.md:70`. Moving
them breaks four documents that govern this repo.

## Verify

- [ ] `just next` lists the same runnable tickets as before the move.
- [ ] `just status` and `just in-progress` resolve through the new dir.
- [ ] `ls tickets/` shows exactly `README.md` and `BACKLOG.md`.
- [ ] `git -C ~/adw/backlog log --oneline` shows the import commit.
- [ ] A dispatch against `adw-factory` commits its status transition **in the
      backlog repo**, and `git -C ~/personal_project/adw-factory status
      --porcelain` stays clean.
- [ ] `grep -rn "tickets/README\|tickets/BACKLOG" README.md CLAUDE.md .claude/`
      still resolves — those paths are unchanged.

## Then do the same per target

Every other target with a `tickets/` dir gets `<STORE>/<target-name>/`.
Per-target subdirectories are **mandatory**: ticket ids are not globally unique,
and the MinIO sweep used the id `minio-quay-registry-move` in seven repos at
once. A flat store collapses them into one file.
