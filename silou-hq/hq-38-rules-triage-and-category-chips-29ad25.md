---
id: hq-38-rules-triage-and-category-chips-29ad25
type: feat
status: in-review
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 400}
depends: [hq-37-mail-widget-shows-gmail-c2631c]
attempts: [{"runId":"hq-38-rules-triage-and-category-chips-29ad25-1791045643422","branch":"adw/hq-38-rules-triage-and-category-chips-29ad25","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-38-rules-triage-and-category-chips-29ad25-1791045643422/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/43","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Mail is triaged by rules into important / normal / noise, with category chips and a reason on every verdict

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

## What to build

Stories 12, 14 and 15 (spec → Triage (v3: rules only)):
- **The `Classifier` seam:** `(MailMeta, RuleContext) → Verdict`, with `by` set to its id. Implement **`rules-v1`** exactly as the spec's ordered table: muted, VIP, Promotions, Social/Forums, List-Unsubscribe, finance/admin, travel, provider-important, Updates, fallback. **Unsure means normal**: only steps 1, 3, 4 and 5 can give `noise`.
- **`RuleContext`:**
  - `vip` = the addresses you wrote to in the last 180 days (Gmail `in:sent` To headers, metadata only), refreshed daily and cached;
  - `vipExtra`, `muted` and `triage.lists` (finance/admin, travel, action words) from `hq.config.json`;
  - `ownAddresses`.
- **The verdict cache** is keyed `messageId + classifierId`, so changing the classifier re-triages.
- **`/mail.json`** now carries the `rules-v1` verdict, with `reasons`, and `counts` per category plus `important`.
- **The widget:**
  - chips **all · pro · perso · finance/admin · travel · newsletters** with counts;
  - the default view is important across all categories;
  - a chip shows the important and normal mail of its category;
  - a "show noise" toggle;
  - each row's reasons shown on hover/focus.

## Red first

- `rules-v1` table-driven: one fixture per step, precedence (a VIP sender with List-Unsubscribe → important), the fallback → normal, and no path to `noise` outside steps 1/3/4/5 (a property-style test over the signal combinations).
- VIP derivation from fixture sent-mail metadata; the 180-day cut-off at a fixed clock.
- Cache: a new classifier id invalidates; the same id doesn't.
- UI (happy-dom): counts, chip filtering, the noise toggle, the reasons.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] Operator live check: on a day of real mail, the default view shows only mail you'd call important. Note misses and false alarms in this ticket's PR to tune `triage.lists`. They are also the seed for the classifier A/B spec.

## Blocked by

- hq-37 (the Gmail source, the cache and the Mail widget).
