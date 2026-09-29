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
01 GET /cqc/config (feat) ─────┬─ 02 authorizer ──┼─┬─ 04 list ─────────┐
                                │                  │ └─ 05 history+report ── 06 artefact ─┴─ 10 openapi (chore)
                                └─ 03 projector ───┼─── 07 import-run (also 00)
                                                   ├─── 08 go1 enrichment
                                                   └─── 09 refresher
11 deploy dev + verify (MANUAL) waits on 02, 04-09 and now 12-14

Found by the first browser against the dev stack, 2026-09-29 (evidence:
adw-factory ai_docs/2026-09-29-cqc-dev-integration-findings.md):

12 cors-preflight-on-every-cqc-route      (bug)  4 of 5 routes 403 the preflight
13 artefact-names-are-paths               (bug)  no artefact is fetchable
14 the-content-summary-the-cmc-renders    (feat) carries spec amendment BE-38
```

All three are independent of each other and of 00-10, so they can run in parallel.
`14` changes `spec-cqc-backend-release-1.md`, `decisions-2026-09-26.md` and
`openapi.yaml` alongside the code - that is deliberate, per the amendment rule.

01a (hand-built scaffold, PR #13: dependencies + lockfile + tooling) gates 01. Agents have no
network, so every dependency lives in that lockfile; tickets must not `npm install`.

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
