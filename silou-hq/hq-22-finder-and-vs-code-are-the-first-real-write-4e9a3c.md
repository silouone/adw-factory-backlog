---
id: hq-22-finder-and-vs-code-are-the-first-real-write-4e9a3c
type: feat
status: in-review
priority: 1
depends: [hq-21-the-bezel-opens-on-click-b71d05]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-22-finder-and-vs-code-are-the-first-real-write-4e9a3c-1791019309910","branch":"adw/hq-22-finder-and-vs-code-are-the-first-real-write-4e9a3c","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-22-finder-and-vs-code-are-the-first-real-write-4e9a3c-1791019309910/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/25","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Show in Finder and Open in VS Code: the first real write, ledgered

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

The write surface's first slice (spec-v2 → The write surface; stories 34, 36, 58–62) and the OPEN sector:
- **`POST /action {nodeId, verb, payload?}`**: the only non-GET not refused with 405.
- **The origin guard.** 403 unless:
  - `content-type` is `application/json`;
  - `Host` ∈ {`127.0.0.1:<port>`, `localhost:<port>`};
  - `Origin` ∈ {`http://127.0.0.1:<port>`, `http://localhost:<port>`};
  - `Sec-Fetch-Site`, when present, is `same-origin`.

  The set comes from config, never from the request's `Host`.
- **The pure verb allowlist** (the full table from the spec; only `reveal` and `code` are wired in this ticket, every other verb → 403 "not yet").
- **Target resolution on the server.** This Mac's entries of the recorded file allowlist; a this-Mac repo resolver for `repo` nodes; `code` opens the enclosing git root, or the folder for memory. M1 entries → 403. A `path` in the payload is ignored.
- **The executor** is injected and receives argv arrays only (`open -R <file>`, `open -a "Visual Studio Code" <dir>`), fire-and-forget.
- **The ledger.** `cache/actions.jsonl`, append-only, every attempt including refusals: `{at, nodeId, verb, outcome, argv|error}`. `GET /ledger.json` returns the newest 200.
- **The OPEN sector** goes live: "Show in Finder" and "Open in VS Code", with past-tense toasts ("Shown in Finder") and a failed toast that says why.

## Red first

The HTTP surface, built with injected deps:
- every non-GET other than `POST /action` is still 405 (the v1 suite, kept);
- the origin guard: a wrong or missing Origin, cross-site, a non-JSON content type, and the **rebinding pair** `Host: evil.test:<port>` + `Origin: http://evil.test:<port>` → 403, ledgered;
- unknown id → 404; a verb outside the kind → 403; an M1 node → 403; a payload `path` is ignored and the recorded path is used;
- the exact argv for a skill, a memory file (its folder) and a repo (its git root);
- the ledger lines for done and refused; `/ledger.json` caps at 200.
- **The guard test is rewritten:** the server branches on exactly one write method and path, `POST /action`.

## Acceptance criteria

- [ ] Nothing in the request can name a filesystem path, a command or a shell string.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] "Show in Finder" on a skill and "Open in VS Code" on a memory file really open.

## Blocked by

- hq-21 (the OPEN sector lives in the bezel).
