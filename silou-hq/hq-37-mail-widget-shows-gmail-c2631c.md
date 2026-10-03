---
id: hq-37-mail-widget-shows-gmail-c2631c
type: feat
status: in-progress
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 400}
depends: [hq-35-today-shows-perso-and-pro-calendars-fbb29b]
attempts: [{"runId":"hq-37-mail-widget-shows-gmail-c2631c-1791039650413","branch":"adw/hq-37-mail-widget-shows-gmail-c2631c","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-37-mail-widget-shows-gmail-c2631c-1791039650413/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/42","provider":"claude","model":"claude-sonnet-5-5","rebased":"39b62fef6eb9be7b56d49e1a155184190f09f216"}]
---
# The Mail widget lists your Gmail metadata, important first, every row opening the thread in Gmail

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

## What to build

Stories 11 and 13 (spec → Mail), end to end for **Gmail**, with display = provider signals only (rules come in hq-38):
- **A Gmail adapter:**
  - `messages.list` with `q="newer_than:2d -in:sent -in:chats -in:draft"`, `maxResults=100`, paginated up to a 300 cap, logging saturation;
  - then `/batch/gmail/v1` of `messages.get?format=metadata&metadataHeaders=From,To,Subject,Date,List-Unsubscribe`, in groups of 25, matched by `Content-ID`, falling back to one request per message if the batch answer can't be parsed;
  - **never `format=full` or `raw`** (assert it on the recorded requests);
  - pacing under 5k quota units/min (a get costs 20); 429/`rateLimitExceeded` back off honouring `Retry-After`.
  - It produces `MailMeta`: subject ≤200, snippet ≤300 plain text, `providerSignals` from `labelIds` (`gmail:IMPORTANT`, `gmail:CATEGORY_*`), `listUnsubscribe`, and `link` = `https://mail.google.com/mail/u/<email>/#all/<threadId>`.
- **The cache** `cache/mail/<accountId>.json` keeps 7 days. Only unseen ids are fetched after a cold start.
- **A `mail` source** polled every `polling.mailMin` (15), with per-account state.
- **`GET /mail.json`:** `{ items, counts, accounts }`. Each item carries a temporary verdict: important if `gmail:IMPORTANT`, otherwise normal. **No body field, ever.**
- **The Mail widget:** important mail first, newest first. Each row shows sender, subject, snippet, an account chip and age, and opens `link` in a new tab. It has states for no accounts, "Nothing important", and a fix line per account that is not ok.

## Red first

- Adapter fixtures: list pagination with the cap, a batch answer with Content-IDs out of order, an unparseable batch → per-message fallback, a 429 with `Retry-After`, 401 → `needs-reconnect`, RFC 2047 encoded subjects, a snippet with HTML entities → plain text.
- Requests: every recorded `messages.get` has `format=metadata`.
- Cache: a second poll fetches only new ids.
- HTTP: `/mail.json` shape; the leak guard (no token, and no body marker from the fixture).
- UI (happy-dom): no accounts; nothing important; a reconnect line; the row link target.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] Operator live check (after hq-32): the widget lists your Gmail's important mail from the last 2 days; a row opens the right thread.

## Blocked by

- hq-35. Serialised because both add a source to the graph store, a route to the server and a widget to the desk.
