# CQC backend tickets (ADW factory)

Work orders for the ADW factory (`~/personal_project/adw-factory`, target
`content-quality-checker`, `ticketsDir: ~/adw/backlog/cqc`). They live in the central
backlog, never in the CoorpAcademy repository. The factory rewrites `status:` and
`attempts:` itself; never edit `attempts:` by hand.

## Backend release 1, read API (`cqc-be-*`)

Source spec: `docs/backend/spec-cqc-backend-release-1.md`, decisions `BE-n` in
`docs/backend/decisions-2026-09-26.md`, deferred work in `docs/backend/todo.md`, coding
rules in `.agents/skills/cqc-guideline/` (all in the content-quality-checker repo).

Dependency graph (`depends:` is enforced; a blocker must be `done`, that is merged):

```
00 redaction-scan (chore) ─────────────────────────┐
01 skeleton + /config (MANUAL) ─┬─ 02 authorizer ──┼─┬─ 04 list ─────────┐
                                │                  │ └─ 05 history+report ── 06 artefact ─┴─ 10 openapi (chore)
                                └─ 03 projector ───┼─── 07 import-run (also 00)
                                                   ├─── 08 go1 enrichment
                                                   └─── 09 refresher
11 deploy dev + verify (MANUAL) waits on 02, 04–09
```

Frontier at start: 00 (factory) and 01 (built by hand in session).

## Before the first dispatch (operator)

1. **Land the backend docs on `main`.** `docs/backend/`, `.agents/skills/cqc-guideline/` and
   the `AGENTS.md` change are untracked or uncommitted locally. The factory cuts every
   worktree from `origin/main`, so an agent cannot see them, and every ticket cites them.
2. **Build 01 by hand.** It creates the `backend/` scripts (`typecheck`, `lint`, `test`) that
   the target's gates need. Until then the gates are the two existing selftests only, which
   is enough for 00 and nothing else. Then add the backend gates (and a `setup`) to
   `targets/content-quality-checker.json`, and flip 01 to `done`.
3. Operator-only items (DevOps grant, portal token, secret rotation) are in `docs/backend/todo.md`
   "Owned by Silou"; they block prod, not the build.

## Running

```bash
cd ~/personal_project/adw-factory
TARGET=content-quality-checker just next
TARGET=content-quality-checker just run cqc-be-00-redaction-scan-without-side-effects-4de841
```
