---
id: adw-cli-01-the-ledger-recipes-ignore-target
type: chore
status: done
priority: 1
created: 2026-09-19
review: false
depends: []
attempts: [{"runId":"adw-cli-01-the-ledger-recipes-ignore-target-1790003645070","branch":"adw/adw-cli-01-the-ledger-recipes-ignore-target","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-cli-01-the-ledger-recipes-ignore-target-1790003645070/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/107","provider":"claude","model":"sonnet"}]
---
# `just next` ignores `TARGET` — four ledger recipes read only this repo's backlog

> **Spec authority:** `specs/adw-v1.14-backlog-tab.md` §1 (trigger 2) and
> §3 **D12**. `chore` lane, `review: false`: every acceptance bullet below is
> a command whose output is checked against a known value.

## The defect

`justfile` recipes `next`, `tickets`, `ticket` and `in-progress` hardcode
`tickets/`; only `status` and `sync` pass `--target {{TARGET}}`. Measured
2026-09-19:

```
$ TARGET=clens just next
adw-review-03-the-reviewer-must-see-its-last-round bug    P2     ← adw-factory's ticket
```

clens has three `queued` tickets (`clens-010`, `-012`, `-013`) that no
recipe can list. This is the CLI half of the forgetting the amendment fixes.

## Requirements

- [ ] **R1 — resolve the dir from the target.** Add `ticketsDirOf(config)`
      to `src/targets/loader.ts` returning `<repo>/tickets`. This is the
      **one seam** `adw-v1.4` may later widen; nothing else resolves a ticket
      dir by hand.
- [ ] **R2 — the four recipes read that dir.** `next`, `tickets`, `ticket`,
      `in-progress` resolve `{{TARGET}}` through `ticketsDirOf` (a `bun -e`
      one-liner over `loadTarget`, same shape the recipes already use for
      `parseTicket`). `TARGET` unset keeps today's default (`adw-factory`).
- [ ] **R3 — the recipes still run the real `parseTicket`.** Not a grep — the
      README documents why.
- [ ] **R4 — a missing store is loud.** A target whose resolved dir does not
      exist prints `no ticket store at <path>` and exits 2 — never an empty
      list that reads as "nothing to do".

## Verify

- [ ] `TARGET=clens just next` lists only `clens-*` ids; `just next` lists
      only `adw-*` ids.
- [ ] `TARGET=clens just tickets | wc -l` equals the number of clens files
      `parseTicket` accepts.
- [ ] `TARGET=sabado just next` prints `no ticket store at …` and exits 2.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

- Any web change. The tab (`adw-backlog-01/02`) consumes `ticketsDirOf`.
- `ticketsDir` in target config (`adw-v1.4`, still proposed).
