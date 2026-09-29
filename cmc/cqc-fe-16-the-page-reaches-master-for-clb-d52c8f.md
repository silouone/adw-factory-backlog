---
id: cqc-fe-16-the-page-reaches-master-for-clb-d52c8f
type: manual
status: queued
priority: 1
created: 2026-09-29
depends: [cqc-fe-14-one-contract-types-generated-from-openapi-9d0a63, cqc-fe-15-the-page-runs-locally-and-the-gates-see-it-5f8b21]
attempts: []
---
# The Content quality page goes to `master` as one PR for CLB

**Operator-executed.** The CLB team owns the CMC and reviews from another timezone; the
whole page reaches `master` as a single PR, per the operator decision of 2026-09-28.

State on 2026-09-29: `cqc/release-1` is **ahead 12, behind 0** of `master`, 01–13 merged,
and the assembled branch was verified green (tslint, 952 tests in 121 suites, build) on a
clean checkout with a fresh `pnpm install --frozen-lockfile`.

## ⚠️ Decide this first: four of the page's actions have no backend

`cqc-fe-10` and `cqc-fe-12` shipped the client for endpoints that do not exist yet. Probed
against the live dev API on 2026-09-29 — all return a **raw API Gateway `403`**, not the
service's `{error, request_id}` shape, because no route exists:

| button | calls | lands in |
|---|---|---|
| Check / paste-many | `POST /cqc/checks`, `POST /cqc/batches` | **release 2** (`cqc-be-22`) |
| Run check again | `POST /cqc/checks` | **release 2** |
| Resolve / Validate | `POST /cqc/content/{lo}/resolution` | **release 3** |
| Reopen | `DELETE /cqc/content/{lo}/resolution` | **release 3** |

So the page looks finished and four of its actions fail with a raw AWS error. Pick one
before opening the PR:

- **(a) Ship read-only now.** Hide or disable the four actions behind something the backend
  controls — a `/cqc/config` capability flag is the natural home, since the page already
  reads config and already renders `launch_modes[].disabled_reason` the same way. Needs a
  small FE ticket and a small BE one. Content ops get the list, the drawer and the evidence,
  and nothing that lies.
- **(b) Hold the PR until release 2.** Nothing more to build; CLB waits. The trigger is the
  reason the page exists, so a read-only page may not be worth their review cycle.

Option (a) is the smaller risk if there is any appetite to get the page in front of content
ops early. Option (b) is the smaller amount of work. It is a product call, not a technical one.

## Steps

- Bring `cqc/release-1` up to date with `master` (it was behind 0 on 2026-09-29; re-check).
- Re-run the gates on the assembled branch in `node:14.21.3` — lint, jest, build.
- Open one PR `cqc/release-1` → `master`, titled for CLB, with a body that lists the 13
  tickets, the spec, and — explicitly — which actions are live and which are not.

## Done when

- [ ] The gates are green on the assembled branch at the moment the PR opens.
- [ ] The PR body states the decision above and what a reviewer can and cannot click.
- [ ] `docs/cqc/` and `.agents/` land with it, or the PR says why they do not — they are
      overlaid from the operator's checkout today and have never been committed.

## Blocked by

- cqc-fe-14-one-contract-types-generated-from-openapi-9d0a63
- cqc-fe-15-the-page-runs-locally-and-the-gates-see-it-5f8b21
