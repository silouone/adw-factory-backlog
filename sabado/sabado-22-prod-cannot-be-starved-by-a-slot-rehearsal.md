---
id: sabado-22-prod-cannot-be-starved-by-a-slot-rehearsal
type: feat
status: done
priority: 3
created: 2026-09-19
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side, sabado-14-nothing-red-or-drifted-lives-on-main]
attempts: [{"runId":"sabado-22-prod-cannot-be-starved-by-a-slot-rehearsal-1789905462370","branch":"adw/sabado-22-prod-cannot-be-starved-by-a-slot-rehearsal","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-22-prod-cannot-be-starved-by-a-slot-rehearsal-1789905462370/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1228"}]
---
# fix(infra): the non-prod stacks fit the host, prod has reservations, and /health says degraded when Redis is down

> **Audit:** **P1-10** (axes F, G, K, L) + the health verdict of **P1-14**
> (axes M, D) — Lane H. Depends on `sabado-14` because it adds a line to
> `ci.yml`. Review stays on: the budget numbers are a judgement.

## What happens today

- `docker-compose.staging.yml:168-169` and `docker-compose.slot.yml:181-182`
  each cap the worker at `cpus: "3.0"`, `memory: 8192M` (raised 2026-09-08,
  `2290b5cc`, with a measured justification); staging's other services add
  0.75+0.25+1.0+0.25 vCPU; the VM is **4 vCPU / 16 GiB, no swap**
  (`infra/numspot/variables.tf:102`, `slot.yml:174`). Non-prod ceilings
  sum to ~13 vCPU / 28 GB. `docker-compose.prod.yml:95`: prod sets no
  per-service ceiling and no reservation — the OOM killer's likeliest
  target. The three file headers still claim 2.4 GiB / 1.125 GiB.
- `deploy/monitoring/` holds `disk-alarm` and `ingestion-alarm` only; no
  CPU, steal or memory alarm; `SENTRY_TRACES_SAMPLE_RATE=0.0`, so p95 is
  invisible. Two overlapping ingestion rehearsals (12+ slot deploys a day
  on 2026-09-18) can OOM-kill whichever process the kernel picks.
- `backend/app/main.py:250`: `status = "ok" if checks["db"] == "up"` —
  Redis down still says `ok`. `redis_store.py:42,60` fail open
  (`is_blocklisted → False`, `check_rate_limit → True`): a Redis outage
  silently switches off token revocation and login rate limiting while
  `ingestion-alarm.sh:168` alerts on `status != ok` only.
- No compose file sets `stop_grace_period`; Docker's default 10 s is what
  a worker gets to settle a 50-minute run (`sabado-18` re-queues it; the
  grace period is this ticket's).

## Requirements

- [ ] **R1 — caps that sum to the host.** Worker limits in
      `docker-compose.staging.yml` and `docker-compose.slot.yml` such that
      **staging + 2 × slot** fits under the host minus prod's reservation
      and the OS (starting point from the audit: staging worker 3.0/6G,
      slot workers 1.0/3G each — the script in R3 is the arbiter, not
      these numbers); `stop_grace_period: 60s` on every worker service;
      the header arithmetic in the three files made true.
- [ ] **R2 — prod has a floor and a ceiling.** `docker-compose.prod.yml`:
      `deploy.resources.reservations` on backend and postgres, an explicit
      worker limit (from the measured 264 MB → 1.04 GB RSS).
- [ ] **R3 — the invariant is a script.** `deploy/ci/compose-budget.sh`
      sums `deploy.resources.limits` across `staging + 2×slot` (plain
      `yq`/awk, no new dependency the CI runner lacks) and fails when CPU
      > the budget constant or RAM > host − prod headroom; the two
      constants are derived in a comment from `variables.tf`. **One line**
      in `ci.yml` runs it on the `docker-compose*.yml` path filter (the
      `changes` job gains a `compose` output).
- [ ] **R4 — an alarm for the host.** `deploy/monitoring/cpu-alarm.sh` +
      `.service` + `.timer`, hourly, same shape and Slack webhook as
      `disk-alarm.sh`: 5-minute load average > vCPU count, steal from
      `/proc/stat`, `MemAvailable` under a floor.
- [ ] **R5 — the health verdict.** `main.py:250`: `status = "ok" if all
      up else "degraded"`. The deploy proofs' field greps stay valid.

## Files

`docker-compose.staging.yml` · `docker-compose.slot.yml` ·
`docker-compose.prod.yml` · `deploy/ci/compose-budget.sh` (new) ·
`.github/workflows/ci.yml` (the `changes` filter + one step; `sabado-14`
landed first) · `deploy/monitoring/cpu-alarm.{sh,service,timer}` (new) ·
`backend/app/main.py` (line 250 only) · `backend/tests/test_health.py`
(or the file that already covers `/health`) · `deploy/README` lines if
the monitoring list is documented there.

## Verify

- [ ] Red test: `/health` answers `status: "degraded"` when the Redis ping
      raises (monkeypatch `redis.from_url`). RED today (`ok`).
- [ ] `deploy/ci/compose-budget.sh` on HEAD after R1 → exit 0 printing the
      sums; the same script with one worker cap edited past the host → non-zero
      naming the cap and the budget.
- [ ] `docker compose -f docker-compose.<each>.yml config` parses; the
      `stop_grace_period` and the limits appear in the rendered config.
- [ ] `bash -n deploy/monitoring/cpu-alarm.sh`; `systemd-analyze verify`
      is not available on macOS — say so in the PR body.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `just test` — all green.
- [ ] **Operator, after merge:** install the timer on the VM the way
      `disk-alarm.timer` was; update ADR-001 §16.1 in the docs repo (out
      of this repo's reach).

## Out of scope

The Google `setex` guard (`sabado-21`); the external uptime probe (P1-11
— an external monitor, operator); `validate_settings()` (P1-12); moving
staging to a second VM (ADR REC-4); `SENTRY_TRACES_SAMPLE_RATE`.
