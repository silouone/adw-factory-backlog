---
id: sabado-18-a-deploy-or-a-dead-model-no-longer-voids-an-ingestion-silently
type: feat
status: done
priority: 2
created: 2026-09-19
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side, sabado-13-models-and-migrations-agree]
attempts: [{"runId":"sabado-18-a-deploy-or-a-dead-model-no-longer-voids-an-ingestion-silently-1789903682714","branch":"adw/sabado-18-a-deploy-or-a-dead-model-no-longer-voids-an-ingestion-silently","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-18-a-deploy-or-a-dead-model-no-longer-voids-an-ingestion-silently-1789903682714/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1227"}]
---
# fix(ingestion): a promotion waits for the live run, an interrupted run re-queues once, and a dead model fails loudly

> **Audit:** **P1-15** (axes M, B) + **P1-25** (axis O) — Lane G.
> Depends on `sabado-13` only because both add a migration: this one's
> `down_revision` must be `0068`, never a second branch off `0067` (that is
> the two-heads incident `sabado-14` exists to prevent).

## What happens today

**P1-15.** `deploy/production-deploy.sh:255` runs `compose up -d` with no
check for a live run. `backend/app/extraction/worker.py:258-300`
(`_settle_claimed_jobs_as_interrupted`) settles the jobs this process owns
as `status="error", error=_INTERRUPTED_MESSAGE` on SIGTERM; no compose file
sets `stop_grace_period` (Docker default 10 s); `enrichment_jobs` /
`mail_scan_jobs` have no `attempts` or `interrupted` state
(`models/enrichment.py:58-92`). Runs last 20–56 min (`worker.py:29`);
promotions land at 07:12, 13:59, 17:23, 19:24 UTC; Monday's release
targets six concurrent runs. The user sees « Analyse interrompue… Relance »
and pays the run again. On staging, every merge does the same.

**P1-25.** Model names default to `-latest` aliases (`config.py:68`,
`deploy/secrets.env.example:44-59`). `model_call.py:120-131` swallows
non-429 exceptions **without logging**; `facts.py:1030-1036` marks
`run.failed`; `household_read.py:520-525` still writes `status="done"`;
`enrich.py:3615` `job.status = "done"`; Réglages shows `done`;
`ingestion-alarm.sh:112-123` alerts on `status='error'` only;
`mails_failed` is consumed by no front component. A retired or mistyped
model name fails every extraction silently as "done".

## Requirements

- [ ] **R1 — the promotion waits.** In `production-deploy.sh`, **before
      step 4**: a bounded wait (30 min, polling every 30 s) while
      `SELECT count(*) FROM enrichment_jobs WHERE status='running' AND
      COALESCE(last_progress_at, started_at) > now() - interval '5 minutes'`
      is > 0, through the compose `exec postgres` idiom the script already
      uses. `FORCE=1` skips the wait and writes `promotion-over-live-run`
      to `promotions.log`. `sabado-16` owns the post-health step of the
      same file; this ticket owns only the pre-step-4 region.
- [ ] **R2 — an interrupted run re-queues once.** `attempts` (integer, not
      null, default 0) on `enrichment_jobs` and `mail_scan_jobs`, one
      migration (`0069`, `down_revision` = `0068`).
      `_settle_claimed_jobs_as_interrupted` sets rows with `attempts < 2`
      back to `pending` with `attempts + 1` and clears the owner; rows at
      `attempts >= 2` keep today's `error` + `_INTERRUPTED_MESSAGE`.
- [ ] **R3 — a refusal is logged.** `model_call.py` `analyze_with_retry`:
      `logger.error` (Sentry-visible) with the model name and the exception
      class on any non-rate-limited refusal. Not a warning.
- [ ] **R4 — a fully failed read is an error.** `household_read.perform_read`:
      when `mails_read == 0 and mails_failed > 0`, set the row's error and
      raise `HouseholdReadFailed` so the job ends `error` and
      `ingestion-alarm.sh` fires. Touch `enrich.py:3615` only if the outcome
      cannot be surfaced from `household_read` alone, and then only that
      site — `enrich.py` is Lane K's file.
- [ ] **R5 — the worker refuses to boot on a dead model.** At worker
      start, when a real provider is configured (never for the fake or
      recording providers, never when `ENVIRONMENT=development` without
      `SABADO_MODEL_PROBE=1`), a 1-token completion against each configured
      tier model; failure exits non-zero with the model name in the log.
      The probe is injectable so the test uses a fake.
- [ ] **R6** No compose file is touched (`sabado-22` owns
      `stop_grace_period: 60s` and the caps).

## Files

`deploy/production-deploy.sh` (pre-step-4 region) ·
`backend/app/extraction/worker.py` · `backend/app/models/enrichment.py`
(+ the mail-scan job model) · one new migration ·
`backend/app/extraction/model_call.py` ·
`backend/app/extraction/household_read.py` ·
`backend/tests/test_enrichment_worker.py` ·
`backend/tests/test_household_read.py` · `backend/tests/test_model_call.py`
(or the existing test file for that module).

## Verify

- [ ] Red test, next to `test_reap_once_*`: an owned running job with
      `attempts=0` is `pending` with `attempts=1` after
      `_settle_claimed_jobs_as_interrupted`; with `attempts=1` it is
      `error` with `_INTERRUPTED_MESSAGE`. RED today (no column).
- [ ] Red test: a read with `mails_read == 0 and mails_failed == 3` leaves
      the job `error`, never `done`. RED today.
- [ ] Red test: a non-429 provider exception in `analyze_with_retry`
      produces one `ERROR` log record naming the model (caplog). RED today.
- [ ] Red test: the boot probe with a fake provider that refuses one tier
      model → the worker's start function raises/exits naming that model;
      with the fake accepting → boots. RED today.
- [ ] `alembic heads` → one head, `0069_…`; the chain test from
      `sabado-13` stays green.
- [ ] `bash -n deploy/production-deploy.sh` and a dry read of the new block:
      the SQL runs through the same `docker compose exec -T postgres psql`
      path as the dump-floor step.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

`stop_grace_period` and every compose file (`sabado-22`); the `enrich.py`
split (Lane K); a dead-letter queue; surfacing `mails_failed` in the front;
the eval suite (Lane L).
