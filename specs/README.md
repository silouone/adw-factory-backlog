# specs/: the central spec home

Specs cited by tickets live here, one folder per target: `specs/<target>/<name>.md`.
Ticket-store scans read only `<backlog>/<target>/` and never this folder.
At the plan stage only, the factory hands a run the specs its ticket cites
(ticket `adw-spec-01`).

## Backfill 2026-10-03: copied, originals not yet removed

Only specs cited by at least one ticket were backfilled. File names are kept, so a
ticket citing a repo path (`docs/cqc/spec-…`, `/…/silou-hq/docs/spec-v1.md`)
resolves by file name.

| Here | Source at copy time |
|---|---|
| `adw-factory/*` (20 files: v1, v1-plan, v1.1–v1.17 except v1.6 and v1.12-epic-drain, plus constitution) | adw-factory `main` `specs/` (v1.17 was moved, not copied) |
| `silou-hq/spec-v1.md`, `spec-v2.md`, `spec-v3-mail-calendar.md` | silou-hq branch `spec/v3-mail-calendar` `docs/`. v1 carries one extra amendment line not yet on `main`; v3 is not on `main` yet |
| `cqc/spec-cqc-backend-release-1.md` | content-quality-checker `docs/backend/` |
| `cmc/spec-cqc-fe-release-1.md` | cmc `docs/cqc/`, an UNTRACKED file in that repo |

Not backfilled:
- `adw-factory`: `adw-readme-visuals`, `adw-v1-tasks`, `adw-v1.6-ghost-chain` and
  `adw-v1.12-epic-drain` (no ticket cites them).
- `clens`: `specs/codex-rollout-import.md` is cited by 2 tickets but **exists nowhere**.
