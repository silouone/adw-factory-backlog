---
id: sabado-26-eight-journeys-a-user-can-take-are-walked-on-every-slot-deploy
type: feat
status: queued
priority: 3
created: 2026-09-20
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [sabado-30-the-remaining-six-journeys-run-on-every-pr-d8bac6]
attempts: []
---
# test(e2e): the journey suite is walked against a deployed slot, and before every production promotion

> **Rewritten 2026-09-28, narrowed to R4+R6.** As first written this ticket held
> the account lifecycle (R1), the verification-code route (R2), eight journeys
> (R3), trace-on-failure (R5) **and** the VM/workflow plumbing (R4, R6) under one
> 300 min / 1000 turn cap. The operator chose (2026-09-28) to land the journeys
> **in CI on every PR** first, because the suite exists to protect a second
> contributor's iteration *before* the fact and a post-deploy suite cannot do
> that. R1 → `sabado-28` · R2 → `sabado-28`, **amended** · R3 → `sabado-29`
> (journeys 1–2) and `sabado-30` (3–8) · R5 → `sabado-29` R7. What remains here
> is decision **D2**: the same suite, pointed at a real deployed environment.
>
> **The R2 amendment.** D1 asked for `GET /admin/accounts/{id}/email-code` on the
> premise that "the code lives in Redis". Redis holds
> `hmac_sha256(SECRET_KEY, "<user_id>:<code>")`
> (`backend/app/auth/redis_store.py:75-89`) — one-way, so nothing can be read
> back; the mailer logs no code either (`core/mailer.py:42`). The route **mints**
> a code instead. Full reasoning: adw-factory
> `ai_docs/2026-09-28-sabado-26-r2-amendment-email-code.md`.

## Why this tier still exists after the CI tier

The CI job runs against `uvicorn` on `localhost` with service containers. A
deployed slot is a different machine: **nginx in front**, HTTPS cookies with
real `Secure`/`SameSite` behaviour, `alembic upgrade head` running inside the
container at boot, and the slot's own `secrets.slot<n>.env`. Each of those has
broken a deploy that CI called green. On 2026-09-20 staging served a blank page
on every route for half an hour with 3 450 unit tests, the build and **five
deploy proofs** all green.

This tier also guards the one gate CI cannot: **the production promotion.**

## Requirements

- [ ] **R1 — where it runs (decision D2, unchanged).**
      `.github/workflows/e2e.yml` **ssh-runs the suite on the VM**, the way
      `deploy-slot.yml` runs `slot-deploy.sh` — secrets (`ADMIN_TOKEN`) stay in
      `secrets.slot<n>.env` / `secrets.staging.env`, **nothing new enters CI**.
      Triggered by `workflow_run` after `deploy-slot.yml`, against the slot that
      run deployed (`slot<N>.dev.sabado.io`), and as a **required step in
      `deploy-production.yml`** against staging before the promotion.
      `E2E_BASE_URL` carries the target. Budget: 3 minutes.
- [ ] **R2 — the same specs, no fork.** The suite is the one `sabado-29` and
      `sabado-30` wrote, unchanged: only `E2E_BASE_URL` and `E2E_API_TARGET`
      differ. **If a journey needs editing to pass against a slot, that edit
      lands in the shared spec and stays green in CI too** — a slot-only variant
      of a journey is the thing this requirement forbids. Say in the PR body
      which specs needed touching and why.
- [ ] **R3 — the account is always removed, and never the wrong one.** The
      `sabado-28` fixture's `afterAll` runs on the VM too, and the suite
      **refuses to start if `E2E_BASE_URL` resolves to `app.sabado.io`**
      (original R6). On a slot the account is real and persistent until deleted,
      so a leaked account here is a leaked account in a shared database — assert
      the deletion, do not assume it.
- [ ] **R4 — a failure names the journey and keeps the trace.**
      `trace: 'retain-on-failure'` uploaded as a workflow artifact from the VM
      run; the step summary lists each journey with pass/fail (original R5,
      applied to this workflow).
- [ ] **R5 — a red suite stops the promotion.** The `deploy-production.yml` step
      is **required**: a failing journey blocks the promotion rather than
      annotating it. State explicitly what happens on an infrastructure failure
      (the VM unreachable, ssh refused) versus a journey failure — a
      promotion must not sail through because the suite never ran.

## Files

`.github/workflows/e2e.yml` (new) ·
`.github/workflows/deploy-production.yml` (one required step) ·
`deploy/e2e-run.sh` (new, VM side, mirrors `slot-deploy.sh`'s shape).

## Verify

- [ ] Every journey green against slot 1 on `main`; under 3 minutes.
- [ ] `GET /admin/accounts` on that slot no longer lists the account afterwards,
      **including after a run where a journey failed**.
- [ ] A deliberately broken journey shows a red step, names the journey, and
      attaches a trace downloadable from the workflow run.
- [ ] A red suite **blocks** a production promotion — proven by an actual
      blocked promotion, not by reading the workflow.
- [ ] An unreachable VM fails the promotion step rather than skipping it.
- [ ] `git diff` on `frontend/e2e/journeys/` is empty, or every change in it is
      listed in the PR body with its reason (R2).
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `test-e2e-boot` · `just test` — green.

## Out of scope

Everything the journeys already exclude (`sabado-30`) · adding e2e to the
post-merge `main` job · Firefox and WebKit.
