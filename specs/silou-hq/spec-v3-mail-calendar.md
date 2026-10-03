# silou-hq — v3 spec: mail and calendar

> **Status:** decisions taken by the operator 2026-10-03.
> **Amends:** rule #2 of `CLAUDE.md` (see Amendment). Everything in `docs/spec-v1.md`, as amended by
> `docs/spec-v2.md`, that this spec does not amend still binds.
> **Labels:** `ready-for-agent`.
> **Sources:**
> - The exploration report `~/personal_project/adw-factory/ai_docs/2026-10-03-hq-mail-calendar-exploration.md`.
> - Sabado's Gmail and Calendar integration (`~/personal_project/SABADO/sabado/backend/app/`: `auth/google.py`,
>   `extraction/fetch_gmail.py`, `extraction/triage.py`, `calendar_google.py`).
> - Microsoft Learn and Google developer docs, cited inline.
> - The operator round of 2026-10-03.

## Operator decisions (2026-10-03)

| # | Question | Decision |
|---|---|---|
| D1 | Which mailboxes | **Go1** (work) plus personal **Gmail** account(s). `go1.com` MX = `go1-com.mail.protection.outlook.com`, so Go1 mail is **Microsoft 365**. |
| D2 | Which calendars | **Perso:** the operator's wife's Google calendar. **Pro:** the Go1 Outlook calendar. |
| D3 | Calendar view | Switch **left/right** between a **pro** and a **perso** lens. |
| D4 | Outlook calendar freshness | Published ICS is acceptable **for v3 only**. It must not stay: v3 measures its lag, and Graph replaces it once Go1 consent exists. |
| D5 | Classifier | **No Jev and no LLM in v3.** Triage is rules plus the providers' own signals, behind a `Classifier` seam. The Jev-vs-local A/B comes later, in its own spec. |
| D6 | Writes to mail or calendar | **Read-only.** Every row links out to the provider's web UI. |
| D7 | Widget size | Unchanged defaults. The operator resizes. |
| D8 | Polling | Mail and calendar every **15 minutes**. There is no manual refresh route in v3. |
| D9 | Re-login | **Signing in again once a day (in the morning) is acceptable** for both personal and work accounts. HQ makes that one step, but never works around a consent or policy block. |

## Amendment to rule #2 (lands with this spec, before any v3 ticket is dispatched)

The current rule #2 reads: "HQ owns no data. Derive everything from files and the public surfaces
of plugged tools. Private data (snapshots, caches) lives in `cache/` and is never committed."

It is replaced by:

> 2. **HQ owns no data, except provider credentials and rebuildable caches.** Derive everything from
>    files, the public surfaces of plugged tools, and read-only provider APIs (Google, Microsoft Graph,
>    published ICS feeds). OAuth refresh tokens and secret feed URLs live **only in the macOS Keychain**
>    under `hq.<provider>.<accountId>`, never in a file, a log, the ledger or a response. Caches
>    (snapshots, mail metadata, triage verdicts) live in `cache/`, are never committed, and can be
>    deleted at any time without losing anything.

What follows from it:
- **Rule #1 is unchanged.** Connecting an account is a **CLI** (`bun run hq:connect <accountId>`), not
  an HTTP route. The CLI runs its own short-lived loopback listener, stores the token in the Keychain and
  exits. The server has no OAuth callback and makes no provider writes. `POST /action` stays the only
  write route.
- **Rule #3 is extended.** Providers are plugged sources. Any of them may be unreachable, expired or
  revoked, and HQ keeps working with the last good value plus an honest per-account state.
- **Rule #4 is extended.** Mail is served as **metadata and a ≤300-char snippet only**. No message body
  and no HTML ever reach the server's responses.

## Problem Statement

The Today widget (top-left) and the Mail widget both show "not connected yet" (spec-v1 story 41).
- To see the day, I open Google Calendar for the family side and Outlook for Go1.
- To find the few mails that matter, I scan two or three inboxes full of notifications and newsletters.

## Solution

HQ reads my calendars and mailboxes with read-only grants, refreshing every 15 minutes.
- **Today** lists today's events under two lenses, **perso** and **pro**, and I flip between them left and right.
- **Mail** shows only the mails that matter, across every mailbox, with chips to slice by category.
- Triage in v3 is deterministic: rules plus Gmail's and Outlook's own signals.
- Every account has an explicit role and lens, so nothing is guessed. A dead token says how to reconnect.

## User Stories

**Accounts**
1. As the operator, I want each account declared in `hq.config.json` with a provider, roles (`mail`, `calendar`) and a lens (`pro`, `perso`), so that which account feeds what is explicit, never "the newest credential wins".
2. As the operator, I want `bun run hq:connect <accountId>` to open the provider's consent page, store the token in the Keychain and exit, so that adding a mailbox is one command.
3. As the operator, I want `bun run hq:disconnect <accountId>` to revoke the grant at the provider and delete the Keychain item, so that disconnecting really disconnects.
4. As the operator, I want each account's state (`ok`, `needs-reconnect`, `locked`, `not-connected`, `unreachable`) shown in the widget with the exact command to fix it, so that a dead token is never silent.
4b. As the operator, I want **one** morning command, `bun run hq:connect --expired`, that signs me back into every account needing it, one browser tab after another, so that a daily re-login (D9) costs me under a minute.

**Calendar**
5. As the operator, I want today's events, plus tomorrow's until noon, in the Today widget, with *now* and *next* marked, so that I see my day at a glance.
6. As the operator, I want to switch the calendar between **perso** and **pro** with ‹ › buttons, the ← → keys while the widget is focused, and a horizontal trackpad swipe, with my last lens remembered, so that work and family never clutter each other.
7. As the operator, I want all-day events shown as dates and timed events in my local timezone, so that nothing shifts across timezones or DST.
8. As the operator, I want my wife's Google calendar in the perso lens, so that the family day is visible.
9. As the operator, I want my Go1 Outlook calendar in the pro lens from its published ICS link, with recurring meetings expanded and the feed's freshness shown ("as of 10:12"), so that I can trust it, or know not to.
10. As the operator, I want an event row to open the event in its provider, so that HQ never needs write scopes.

**Mail**
11. As the operator, I want the Mail widget to list only **important** mail from every mail-role account, newest first, so that the inbox noise stays out of HQ.
12. As the operator, I want chips **all · pro · perso · finance/admin · travel · newsletters** with counts, so that I can pull out one category quickly.
13. As the operator, I want every mail row to show sender, subject, snippet, account and age, and to open the thread in Gmail or Outlook on the web, so that reading and acting happen where the mail lives.
14. As the operator, I want a mail the rules can't settle shown as **normal** (visible under "all"), never dropped, so that triage can only cost me noise, never a missed mail.
15. As the operator, I want each verdict to carry its reason ("VIP sender", "Gmail: Promotions", "List-Unsubscribe"), so that I can see why a mail was ranked as it was.

## Implementation Decisions

### Modules

| Module | Role | Pure? |
|---|---|---|
| `src/accounts/config.ts` | Parse and validate `accounts` in `hq.config.json`. Fail fast, naming the account id | yes |
| `src/accounts/keychain.ts` | The **only** module that runs `security` (find/add/delete-generic-password). Injected everywhere else | edge |
| `src/accounts/oauth.ts` | PKCE pair, auth URL, code exchange, refresh, revoke, for Google and Microsoft, over an injected `fetch` | pure but for `fetch` |
| `scripts/connect.ts` | `hq:connect` / `hq:disconnect`: a loopback listener on `127.0.0.1:<ephemeral>`, opens the browser, writes the Keychain | edge |
| `src/calendar/google.ts` | Google Calendar payloads → `CalEvent[]` | yes (parser) |
| `src/calendar/ics.ts` | ICS text → expanded `CalEvent[]` in a window | yes |
| `src/calendar/graph.ts` | Graph `calendarView` payloads → `CalEvent[]` (hq-39) | yes |
| `src/mail/gmail.ts` | Gmail list and batch payloads → `MailMeta[]` | yes (parser) |
| `src/mail/graph.ts` | Graph `messages` payloads → `MailMeta[]` (hq-39) | yes |
| `src/mail/rules.ts` | `(MailMeta, RuleContext) → Verdict` | yes |
| `src/mail/classifier.ts` | The `Classifier` seam. v3 ships `rulesOnly` | yes |
| `src/sources/mail.ts`, `src/sources/calendar.ts` | Fan out per account and merge; per-account status | edge |
| `src/web/widgets/today.tsx`, `mail.tsx` | The UI | — |

Each provider module follows `src/factory/adapter.ts`:
- it is the only place that knows that provider's payload shape;
- its parsing is pure;
- it never throws to the store. It returns `{ ok: true, items } | { ok: false, state, reason }`, and the reason names the account id and the endpoint.

### Shapes

```ts
type Lens = "pro" | "perso";
type Role = "mail" | "calendar";
type Account =
  | { id: string; provider: "google"; email: string; roles: Role[]; lens: Lens; calendars?: string[] }
  | { id: string; provider: "microsoft"; email: string; roles: Role[]; lens: Lens }
  | { id: string; provider: "ics"; label: string; roles: ["calendar"]; lens: Lens };
type AccountState = "ok" | "needs-reconnect" | "locked" | "not-connected" | "unreachable";

type CalEvent = {
  id: string;              // `${accountId}:${providerId}[:${occurrenceStart}]`
  accountId: string; lens: Lens;
  title: string;
  start: { date: string } | { at: number };   // all-day keeps a DATE; timed = epoch ms
  end:   { date: string } | { at: number };
  location?: string;
  link?: string;           // provider web URL, when the provider gives one
};

type MailMeta = {
  id: string; threadId: string; accountId: string; lens: Lens;
  from: { name: string; address: string }; to: string[];
  subject: string;         // ≤200
  snippet: string;         // ≤300, plain text
  at: number;
  providerSignals: string[];   // e.g. "gmail:IMPORTANT", "gmail:CATEGORY_PROMOTIONS", "outlook:focused", "outlook:importance-high"
  listUnsubscribe: boolean;
  link: string;
};

type Category = "pro" | "perso" | "finance_admin" | "travel" | "newsletter" | "notification" | "promotion";
type Verdict = { importance: "important" | "normal" | "noise"; category: Category; reasons: string[]; by: string };
```

### Accounts and config

`hq.config.json` gains `accounts` (no secrets: ids, emails, roles, lenses) and `polling` (`{ mailMin: 15, calendarMin: 15 }`). The operator's set at the time of writing:

```jsonc
"accounts": [
  { "id": "perso-gmail", "provider": "google", "email": "<operator>@gmail.com", "roles": ["mail", "calendar"],
    "lens": "perso", "calendars": ["<wife>@gmail.com"] },
  { "id": "go1-cal-ics", "provider": "ics", "label": "Go1 (published)", "roles": ["calendar"], "lens": "pro" },
  { "id": "go1", "provider": "microsoft", "email": "<operator>@go1.com", "roles": ["mail"], "lens": "pro" }
]
```

- **Wife's calendar, the default path:** she shares her Google calendar with the operator's Gmail ("See all event details"). It then appears in the operator's `calendarList`, so `calendars` names it by id. This needs no token for her account.
- **Wife's calendar, the alternative:** her own `google` account entry, connected with `hq:connect` on her consent. Both work without code changes.
- **`calendars` absent** means `primary` only. Only ids in `calendarList` are read; an unknown id gives a per-account error that names it.
- **The `go1` account** gains `"calendar"` in its roles once hq-39 is live and consent exists. `go1-cal-ics` is removed at that point.
- **Mail lens:** an account's `lens` is the default `pro`/`perso` category for its mail.

### Connecting (the CLI)

- **Google.** One GCP project owned by the operator, separate from sabado's.
  - **Desktop app** OAuth client, loopback redirect `http://127.0.0.1:<port>`, PKCE.
  - The consent screen is **In production (unverified)**, never "Testing", because Testing expires refresh tokens after 7 days (sabado #232). A personal-use app with under 100 users is exempt from verification and CASA. The "unverified app" warning is clicked through once per account. ([oauth2 expiration](https://developers.google.com/identity/protocols/oauth2#expiration), [unverified apps](https://support.google.com/cloud/answer/7454865))
  - Scopes: `calendar.readonly` and `gmail.readonly`. Only `format=metadata` is ever fetched; see Mail.
  - `access_type=offline` and `prompt=consent`.
  - **No refresh token in the answer → the CLI fails loudly**, naming the account.
- **Microsoft (hq-39).** An Entra app registration, **public client** ("Mobile and desktop applications" platform, not Web, not SPA). Loopback redirect `http://localhost` (the port is ignored for localhost). PKCE.
  - Scopes: `offline_access User.Read Mail.Read Calendars.ReadBasic`.
  - The refresh token lasts 90 days and **rotates on every use**, so the new one is written back on every refresh. ([refresh tokens](https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens))
- **ICS.** `bun run hq:connect go1-cal-ics` prompts for the published URL and stores it in the Keychain. The URL is a bearer secret and is never logged, served or put in the ledger.
- **Disconnect** revokes the grant at Google (`oauth2.googleapis.com/revoke`). Microsoft has no per-app revoke for public clients; the CLI deletes the Keychain item and prints where to remove the grant (myapps.microsoft.com). Then it deletes the account's cache.
- **Re-login is a daily-tolerable event (D9).** Any refresh failure that needs a person maps to `needs-reconnect`:
  - Google `invalid_grant`.
  - Microsoft `invalid_grant` or `interaction_required`. This includes Conditional Access sign-in-frequency and MFA prompts (`AADSTS50076`, `AADSTS50079`, `AADSTS70043`, `AADSTS700082`).
  - The last good data stays on screen, marked stale.
  - `hq:connect --expired` walks every account in that state.
  - **D9 does not unblock admin consent.** `AADSTS65001` and "Need admin approval" stay `not-connected` until Go1 IT consents (hq-33). Borrowing a Microsoft first-party client id to slip past tenant consent is out of scope: it circumvents the employer's controls.
- **Keychain under launchd.** HQ runs as the `com.silou.hq` LaunchAgent. If `security` cannot read the item (keychain locked, or no access from the agent context), the account's state is `locked`, and the other sources keep working. This is **unverified** under launchd and is checked in hq-32.

### Calendar

- **Window:** from the local start of today to tomorrow 12:00. Polled every 15 minutes.
- **Google:**
  - `GET calendar/v3/calendars/{id}/events` with `timeMin`, `timeMax`, `singleEvents=true` and `orderBy=startTime`, so Google expands recurring events.
  - Cancelled events are skipped.
  - `calendarList.list` is called on each refresh to validate the configured ids.
- **ICS:**
  - Fetch the URL with `If-None-Match` / `If-Modified-Since` when the server supports them.
  - Parse, then expand `RRULE` with `EXDATE` and `RECURRENCE-ID` overrides, resolving `VTIMEZONE` (Windows zone names included, which Outlook emits).
  - **The library is chosen in hq-36 by a spike under Bun:** `node-ical` (`expandRecurringEvent`) or `ical.js`. Whichever is chosen, it is wrapped by `src/calendar/ics.ts` and tested on fixtures.
- **Why published ICS can lag (D4).**
  - HQ fetches the URL itself every 15 minutes, so the "hours" figures quoted online do not apply directly. Those come from calendar apps (Outlook.com, Google Calendar) that cache *subscribed* feeds for 3–24 hours.
  - What remains unknown is how often Exchange Online regenerates the published file. Microsoft doesn't document it.
  - So hq-36 **measures** the lag: every fetch logs `fetchedAt`, `Last-Modified`/`ETag` and a content hash to `cache/ics-freshness.jsonl`.
  - The widget shows "as of HH:MM", from `Last-Modified` or the time the content last changed.
  - The real fix is Graph with Go1's consent (hq-39), which is live data with no lag beyond the poll.
- **Merge:** events are grouped by lens, then sorted all-day first, then by start. Overlapping events stay as separate rows.
- **Display (Today widget, below the routines "next up").**
  - The header reads `‹ perso · pro ›`. The active lens is ember and the other is dim.
  - Rows: time (or "all day"), title, location (dim), and a dot in the account's colour.
  - Past events are dimmed. A *now* marker sits between rows; the *next* event is bold, with "in 25 min".
  - Each lens has an empty state: "Nothing in perso today."
  - Each account state that is not `ok` gets one line with its fix, for example "go1-cal-ics: not connected — `bun run hq:connect go1-cal-ics`".
  - The lens is kept in `localStorage` (`hq.calendar.lens`).
  - Switching: the ‹ › buttons, ←/→ while the widget has focus, and a horizontal trackpad swipe (`wheel` with |deltaX| > |deltaY| over the widget, debounced). The swipe must not trigger the rings' pan, so the widget stops propagation.

### Mail

- **Poll:** every 15 minutes per mail-role account. Look back 2 days on a cold start. After that, only ids not yet in `cache/mail/<accountId>.json` are fetched.
- **Gmail:**
  - `messages.list` with `q="newer_than:2d -in:sent -in:chats -in:draft"` and `maxResults=100`, paginated up to a 300 cap. Saturation is logged.
  - Then `POST /batch/gmail/v1` of `messages.get?format=metadata&metadataHeaders=From,To,Subject,Date,List-Unsubscribe`, in groups of 25. Answers are matched by `Content-ID`; if the batch answer can't be parsed, fall back to one request per message (sabado `fetch_gmail.py:542`).
  - **Never `format=full` or `raw`.**
  - Pacing: a get costs 20 quota units (since 1 May 2026). A 300-mail cold start is ~6k units against a 6k/min/user budget, so batches are spaced to stay under 5k/min. 429 or `rateLimitExceeded` back off with `Retry-After`.
  - `gmail.readonly` is used because `gmail.metadata` refuses the `q` parameter. Only metadata is fetched.
- **Microsoft Graph (hq-39):**
  - `GET /me/mailFolders/inbox/messages?$filter=receivedDateTime ge <t>&$select=id,conversationId,from,toRecipients,subject,bodyPreview,receivedDateTime,importance,inferenceClassification,categories,flag,webLink,internetMessageHeaders&$top=50`.
  - `bodyPreview` is truncated to 300 characters.
  - `internetMessageHeaders` is checked only for `List-Unsubscribe`.
  - **Unverified:** `$select` of `internetMessageHeaders` on a list call. If it's refused, `listUnsubscribe` is `false` for Graph mail and the rules lean on `inferenceClassification`.
- **Cache:** `cache/mail/<accountId>.json` holds the `MailMeta`s from the last 7 days plus their verdicts, keyed `messageId + classifierId`. A new classifier id re-triages and nothing else does.

### Triage (v3: rules only)

The `Classifier` interface is `(mail: MailMeta, ctx: RuleContext) => Verdict`, with `by` set to its id. v3 ships one implementation, `rules-v1`. The later A/B (Jev, a local model) adds implementations behind the same seam and changes nothing upstream.

`RuleContext`:
- `vip`: the addresses the operator has written to in the last 180 days. These are read from `in:sent` To headers (Gmail) and Sent Items (Graph), cached daily.
- `vipExtra` / `muted`: lists in `hq.config.json`.
- `ownAddresses`: the operator's own addresses.

`rules-v1` checks in order. The first match decides importance, and each step that matches appends a reason:

| # | When | importance | category |
|---|---|---|---|
| 1 | sender in `muted` | noise | from the signals below, else the account lens |
| 2 | sender in `vip` or `vipExtra` | **important** | account lens |
| 3 | `gmail:CATEGORY_PROMOTIONS` | noise | promotion |
| 4 | `gmail:CATEGORY_SOCIAL` / `CATEGORY_FORUMS` | noise | notification |
| 5 | `listUnsubscribe` | noise | newsletter |
| 6 | sender domain or subject matches the finance/admin list (banks, impots.gouv, URSSAF, CAF, insurers, utilities; config list) | important if the subject matches the action list (`facture`, `échéance`, `paiement`, `invoice`, `due`, `action required`, …), else normal | finance_admin |
| 7 | sender domain or subject matches the travel list (airlines, SNCF, booking sites; config list) | important if the departure falls within 7 days and can be parsed from the subject, else normal | travel |
| 8 | `gmail:IMPORTANT`, or `outlook:focused` + `outlook:importance-high`, or `outlook:flagged` | **important** | account lens |
| 9 | `gmail:CATEGORY_UPDATES` | normal | notification |
| 10 | otherwise | **normal** | account lens |

- **Unsure means normal.** v3 has no path that makes unknown mail `noise`, except steps 1, 3, 4 and 5.
- The finance, admin, travel and action lists live in `hq.config.json` (`triage.lists`), so tuning needs no code change.
- The chips filter on `category`. "pro" and "perso" mean the account-lens categories. The widget's default view is **important across all categories**. Selecting a chip shows the important *and* normal mail of that category; noise stays hidden unless "show noise" is toggled.

### HTTP surface (GET only; rule #1 unchanged)

| Route | Returns |
|---|---|
| `/calendar.json` | `{ lenses: { perso: CalEvent[]; pro: CalEvent[] }, accounts: {id, state, asOf?, fix?}[], window }` |
| `/mail.json` | `{ items: (MailMeta & {verdict})[], counts: Record<Category\|"important", number>, accounts: {id, state, fix?}[] }`, the last 7 days, capped at 300 items |

No response carries a token, a feed URL, a message body or an HTML part. A test asserts this by scanning every serialised response for the fixture's secrets and body markers.

## Testing Decisions

- **The seams.** Test-first, fixtures in and values out:
  - `accounts/config` parsing;
  - each provider parser (`google`, `ics`, `gmail`, `graph`);
  - `rules-v1`, table-driven, one row per rule step plus the unsure fallback;
  - the lens/merge/window maths;
  - the HTTP surface.
- **Fixtures.** Real payload shapes with fake data, stored in `test/fixtures/{google,gmail,graph,ics}/`. ICS fixtures must cover:
  - a weekly RRULE with an EXDATE;
  - a moved occurrence (`RECURRENCE-ID`);
  - an all-day event;
  - a Windows timezone (`Romance Standard Time`);
  - an event across the DST change on 2026-10-25.
- **OAuth.** Unit-tested with an injected `fetch`, covering: PKCE challenge, auth URL params, code exchange, refresh, rotation written back (Microsoft), `invalid_grant` → `needs-reconnect`, and revoke. **No live provider in any test.**
- **Keychain.** Injected in every test. `keychain.ts` itself gets one test with a fake `security` runner: argv shape, and a not-found → `not-connected` mapping.
- **UI (happy-dom).** Cover:
  - Today with no calendar accounts;
  - each lens empty;
  - the lens switch by button and by key, persisted;
  - one account `needs-reconnect`;
  - Mail with no accounts, all noise ("Nothing important"), and chips with counts.
- **Leak guard.** Over every route, with fixture accounts, the response contains no refresh token, access token, ICS URL or body text.

## Out of Scope

- Jev, any LLM, or any learned classifier. Labelling and evaluation UI. **This is the next spec: the classifier A/B.**
- Any write to mail or calendar: archive, label, mark-read, RSVP, create event.
- Push or webhooks (Gmail `watch`, Graph subscriptions). Incremental sync tokens (Gmail `historyId`, Calendar `syncToken`, Graph delta) are allowed as an optimisation inside an adapter but are not required.
- A full-screen mail or calendar view. Week and month views. Mail bodies.
- Notifications ("meeting in 10 min").
- Mail or calendar as graph nodes on the rings.

## Further Notes

- **Go1 needs consent, and one request unlocks everything.**
  - Microsoft's default consent policy excludes `Mail.Read` and `Calendars.Read`/`ReadBasic` from user consent ([consent policies](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/manage-app-consent-policies)).
  - hq-33 (operator) probes it. Try `hq:connect go1`. If the consent page offers "Accept", it is done. If it says "Need admin approval", file the admin-consent request with Go1 IT, which covers `Mail.Read` plus `Calendars.ReadBasic` in one go.
  - Until then, Go1 mail shows `not-connected` with that reason, and the pro calendar uses ICS.
  - Reading work mail into a local-only tool keeps the data on this Mac, with no third party. That is the operator's call with Go1 IT, made in hq-33.
- **The Entra app needs a tenant.** The quickstart lists an Azure account. Whether registration costs anything is unverified and believed to be free. Go1 IT may prefer to register the app in the Go1 tenant themselves; the code only needs a client id and an authority.
- **Unverified items, each checked by the ticket that relies on it:**
  - Keychain reads under launchd (hq-32).
  - Exchange's published-ICS regeneration cadence (hq-36 measures it).
  - Bun compatibility of `node-ical` and `ical.js` (hq-36 spike).
  - `internetMessageHeaders` in a Graph list `$select` (hq-39).
  - Whether the Go1 tenant has publishing enabled (hq-33).
- **Sabado traps not repeated here:**
  - Testing-mode tokens.
  - `primary`-only calendars.
  - All-day stored as midnight UTC.
  - "Newest credential wins".
  - No revoke on disconnect.
  - Unused scopes.
