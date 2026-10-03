---
id: cqc-fe-40-checks-come-before-limitations-in-plain-words-192e22
type: feat
status: done
priority: 1
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-38-the-journey-track-is-the-navigation-be1def]
attempts: [{"runId":"cqc-fe-40-checks-come-before-limitations-in-plain-words-192e22-1791024776087","branch":"adw/cqc-fe-40-checks-come-before-limitations-in-plain-words-192e22","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-40-checks-come-before-limitations-in-plain-words-192e22-1791024776087/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/176","provider":"codex","model":"gpt-6-sol"}]
---
# Checks come before limitations, in plain words

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 22–30, 32.

## The problem

In `RunDrawer/ChecksTab.tsx` the limitations come **before** the checks: an uppercase eyebrow
"What this run could not observe 6", a bullet, then a grey box "Also on most runs of this
profile" with five long bullets. The checks start about 650 px down. The bullets show raw field
names to content ops (`package_extraction_snapshot`, `provenance.content_hash`,
`HARNESS_LAUNCH_MODE=preview`, `navigation_trace.completed_path`, `END_OF_COURSE`,
`LESSONS_DONE=4`). Collapsed checks truncate their reason with "…"; the failed check is a full
pink block whose "Reasons" and "Evidence" headings each hold one item, plus a nested "Declared vs
observed" disclosure; "Blocker · counts toward verdict" sits far top-right; "passed ›" has a
chevron and nothing to open.

## The target

- **Order:** the selected check (from the journey track, fe-38) first, then the other checks,
  then "What this run could not see (n)" as a collapsed section at the bottom. Run-specific and
  profile-wide limitations are two short sub-lists inside it, not a nested box.
- **Plain words:** a small, tested map from known limitation codes/field names to one human
  sentence each; unknown codes fall back to the raw text in mono at secondary size (still
  visible, per the "no static data" rule — the map translates, it never invents).
- **Check row:** name, state word, and the reason on up to two lines (no single-line ellipsis).
  "Blocker" sits next to the state word. Rows with nothing to expand have no chevron.
- **Expanded failure:** one block: reason sentence, then a "Declared vs observed" two-column
  table shown inline (not a disclosure), then evidence links. A left accent border in the error
  colour instead of a full pink fill; no "Reasons" / "Evidence" headings for single items.
- The space below the checks is not left empty on the page: the section ends where its content
  ends (no fixed heights).

## Acceptance criteria

- [ ] Red tests first:
  - the first check card precedes the limitations section in DOM order;
  - a known code (`HARNESS_LAUNCH_MODE=preview`) renders its human sentence; an unknown code
    renders its raw text;
  - a passed check with no details renders no expand control;
  - the failed check renders declared-vs-observed without a disclosure button.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
