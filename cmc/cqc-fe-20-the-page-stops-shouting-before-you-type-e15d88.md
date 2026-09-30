---
id: cqc-fe-20-the-page-stops-shouting-before-you-type-e15d88
type: chore
status: done
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-20-the-page-stops-shouting-before-you-type-e15d88-1790757375205","branch":"adw/cqc-fe-20-the-page-stops-shouting-before-you-type-e15d88","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-20-the-page-stops-shouting-before-you-type-e15d88-1790757375205/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/149","provider":"codex","model":"gpt-6-sol"}]
---
# The page stops showing an error before the user has typed, and triage gets the top of it

Audit: adw-factory `ai_docs/2026-09-30-cqc-frontend-ux-audit.md`, Parts 2 and 3.

## 1. Validation fires on an untouched field

`TriggerChecks/index.tsx:279-283`

```tsx
{parsed.validIds.length === 0 && (
  <Text element="p" fontSize={1}>Enter at least one valid LO ID.</Text>
)}
```

`validIds.length === 0` is true for an empty textarea, so the message is on screen at first
paint — and it is not styled as an error, so it reads as a second helper line under
"Separate IDs with commas, spaces or new lines."

`patterns.md` → Forms: *"Validate on submit or on blur, not on each keystroke"*, and an error
is `Text color="danger" fontSize={1}` below the control, linked with `aria-describedby`, with
`error` set on the input.

- [ ] Nothing about validity is shown until the field is blurred or the form submitted.
- [ ] When it is shown, it is a danger-styled error tied to the control, and the `TextArea`
      carries `error`.
- [ ] The Check button's disabled state is unchanged — it may still be disabled from the
      start; that is an affordance, not a complaint.

## 2. The disabled Partner filter breaks the filter row's baseline

`CheckedContentList/CheckedContentFilters.tsx` — the sixth control is disabled with
"Partner data is not available yet" beneath it, so that column is two lines taller than the
other five.

The CQC placeholder policy requires the disabled control and the named dependency; it does
not require the reason to sit in the layout flow.

- [ ] The filter row has one baseline across all six controls.
- [ ] The reason is still available to every user — including screen-reader users — without
      adding a second line to the row. Say in the PR how.

## 3. Triage gets the top of the page

"Run checks" is a paste box for ids from Slack. It is occasional; triage is the daily job.
It currently sits above the backlog summary and the list, and on the live stack its button
cannot work at all yet (`POST /cqc/checks` is backend release 2, `cqc-be-22`).

Move it below the backlog summary, or into a header action that opens it — `patterns.md`
allows at most one `ButtonFilled color="accent"` primary in the header. Choose one and say
why in the PR. Do not change what it does.

- [ ] The backlog summary and the list are the first things below the page title.
- [ ] Every behaviour of the trigger section is unchanged: parsing, the estimate, the batch
      limit warning, the toast.

## Base and constraints

**Base branch: `cqc/release-1`.** No new hex, `rgb(`, px spacing or px font size — this
feature has none today. tslint, jest and build green; the dev server compiles
(`node scripts/start.js`).

## Blocked by

- (nothing — independent of fe-17/18/19)
