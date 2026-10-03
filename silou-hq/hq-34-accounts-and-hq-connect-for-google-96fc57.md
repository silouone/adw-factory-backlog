---
id: hq-34-accounts-and-hq-connect-for-google-96fc57
type: feat
status: done
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 400}
depends: []
attempts: [{"runId":"hq-34-accounts-and-hq-connect-for-google-96fc57-1791030667036","branch":"adw/hq-34-accounts-and-hq-connect-for-google-96fc57-3","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-34-accounts-and-hq-connect-for-google-96fc57-1791030667036/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/32","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Accounts are declared in config, and `hq:connect` links a Google account into the Keychain

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

## What to build

Stories 1–4 (spec → Accounts and config; Connecting):
- **`accounts` and `polling` in `hq.config.json`.** Parse them with the spec's `Account` shape and fail fast, naming the account id: an unknown provider, a role the provider can't serve (`ics` + `mail`), a duplicate id, a missing lens.
- **A Keychain module, the only code that runs `security`.** It does find, add and delete for items named `hq.<provider>.<accountId>`. "Not found" maps to `not-connected`. A locked keychain or denied access maps to `locked`.
- **OAuth for Google, pure but for an injected `fetch`:**
  - PKCE pair and auth URL: `access_type=offline`, `prompt=consent`, scopes `calendar.readonly gmail.readonly`, loopback `http://127.0.0.1:<port>`;
  - code exchange;
  - refresh, where `invalid_grant` maps to `needs-reconnect`;
  - revoke.
- **`bun run hq:connect <accountId>`** starts a one-shot loopback listener on an ephemeral port, opens the browser, checks `state`, exchanges the code and writes the refresh token to the Keychain, then exits. **No refresh token in the answer → exit non-zero** and name the account. For an `ics` account it prompts for the URL instead and stores that.
- **`bun run hq:connect --expired`** (story 4b, D9) reconnects every account currently `needs-reconnect`, one after another, and prints a one-line summary. With none expired, it says so and exits 0.
- **`bun run hq:disconnect <accountId>`** revokes at Google, deletes the Keychain item and deletes the account's `cache/` files.
- **The server stays as it is:** no OAuth route. An account-status reader (`accountId → AccountState`) exists for hq-35 and hq-37 to use.

## Red first

- Config parsing: valid, and each invalid case, with the error naming the id.
- OAuth with a fake `fetch`: auth URL params including the S256 challenge, the exchange body, refresh, `invalid_grant` → `needs-reconnect`, revoke, and a missing refresh_token → error.
- Keychain with a fake `security` runner: argv shape for each operation; not-found → `not-connected`.
- `--expired` with a fake account-state reader: it visits only the expired accounts, in config order.
- The leak guard: no token or secret appears in any log line the CLI prints (assert on captured output).
- The existing guard test still finds exactly one write route.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] Operator live check (after hq-32): `bun run hq:connect perso-gmail` stores a token; `security find-generic-password -s hq.google.perso-gmail` finds it; `hq:disconnect` revokes it, and Google's "third-party access" page no longer lists the app.

## Blocked by

- None. It can start immediately. The live check needs hq-32.
