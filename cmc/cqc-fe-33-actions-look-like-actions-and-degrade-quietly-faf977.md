---
id: cqc-fe-33-actions-look-like-actions-and-degrade-quietly-faf977
type: bug
status: done
priority: 2
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-28-the-drawer-header-fits-in-a-quarter-6ae6c9, cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4]
attempts: [{"runId":"cqc-fe-33-actions-look-like-actions-and-degrade-quietly-faf977-1790892529629","branch":"adw/cqc-fe-33-actions-look-like-actions-and-degrade-quietly-faf977","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-33-actions-look-like-actions-and-degrade-quietly-faf977-1790892529629/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/166","provider":"codex","model":"gpt-6-sol","rebased":"f9120601504f8f314f5adfc14c33f990edcba905"}]
---
# Actions look like actions, the Resolve dialog uses radios, and a missing config is said once — not on every row

Design pass 2026-10-01. Evidence (operator-only, **not in your worktree** — do not look for it): adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`.
Today: `shots/app-01-fullpage.png` (config down), `shots/appcfg-state-footer-resolve.png`,
`shots/appcfg-w1440-d0-checks-s0.png` (footer). Target: `shots/proto-w1440-d0-checks-s0.png`
(footer), prototype Resolve modal (`docs/cqc/prototype/app.js` `resolveModal`, described below).

## The problems

1. **When `GET /cqc/config` fails, every row grows to ~200px.** Each action renders its own
   reason under its button: "Checks are unavailable becau…" (ellipsised,
   `common/RunCheckAction.tsx:158-177`) and "Case action configuration is unavailable."
   (`common/CaseActions.tsx:396-407`), stacked in the Actions cell. The trigger textarea is
   disabled outright (`TriggerChecks/index.tsx:265`), so nobody can even type. On dev the config
   *does* fail in a clean browser today (missing CORS header — backend `cqc-be-41`), which is how
   this was measured; the state must be designed whatever the cause.
2. **The drawer footer's actions read as text.** "Copy link to this run" and "Resolve…" are
   `ButtonMinimal` (`RunDrawer/RunDrawerFooter.tsx:71-75`, `CaseActions.tsx:385-395`); "Link
   copied" is inserted inline (`:76-80`) and shifts the row.
3. **The Resolve dialog's outcomes have no radio affordance**: `role="radiogroup"` over
   `ButtonMinimal role="radio"` (`CaseActions.tsx:445-481`); the only cue is a grey fill, and
   "Fixed and verified" is a faded line with its reason two lines lower.

## The target

- **Config down = one message.** A single `Banner type="warning"` at the top of the list:
  "Checks and case actions are unavailable: the check configuration couldn't be loaded.
  [Try again]". Row actions render **disabled with the reason in `title` and
  `aria-describedby`** (a visually hidden id), no visible helper text, so rows keep their height.
  The trigger textarea stays editable; only `Check` is disabled, with the same reason.
- **Footer** (prototype): `[Copy link to this run]` as a bordered secondary `Button`, the
  keyboard hint "j / k next or previous LO · Esc close" subtle, spacer, `[Resolve…]` bordered,
  `[Run check again]` primary. The copy result goes through the CMC's existing
  `NotificationManager.success("Link copied")` (already used in the app) — no inline text.
- **Resolve dialog** (prototype): Go1d `RadioGroup`/`RadioInput` cards, one per outcome, each
  with its label bold and a one-line explanation:
  - **Fixed and verified** — "The latest run passes after a fix." (disabled unless the latest
    run passes; the reason "Needs a passing latest run." sits *inside* the card)
  - **Validated manually** — "A person checked the content works for learners; the checker
    could not observe it."
  - **False positive** — "The checker or the environment was wrong, not the content."
  - **Accepted as is** — "Known issue, no action planned (explain why)."
  Selected card: accent border + `soft` background. Then the note field (label "Note (required,
  shared with the team)", placeholder "What did you check? Link the Slack thread or ticket.").
  A subtle line under the title: "Applies to run <run id>. A later run that does not pass reopens
  the case automatically." Replace the hand-written arrow-key handling with the Go1d radio group's.
- Keep `cqc-fe-22`'s rule: actions the backend says don't exist are disabled with the reason.

## Acceptance criteria

- [ ] Red tests first:
  - with config failing, the list renders exactly one config banner and **no** per-row reason
    text node; each row action is disabled with an `aria-describedby` reason;
  - with config failing, the trigger textarea is enabled and `Check` is disabled;
  - the footer's Copy link and Resolve are `Button`s (not `ButtonMinimal`); copying calls the
    notification, and no "Link copied" text appears in the footer;
  - the Resolve dialog renders radio inputs (`input[type=radio]`), four, with the disabled one
    carrying its reason text.
- [ ] Resolve / Validate / Reopen behaviour and API calls unchanged.
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

Sweep `pass3-list.mjs` **without** `FIXCFG` (config down): row heights ≤ 96px.
