---
id: cqc-fe-30-evidence-is-a-list-beside-its-viewer-d60eaf
type: bug
status: in-progress
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-29-the-strip-and-tabs-stay-put-and-checks-are-cards-31f799]
attempts: [{"runId":"cqc-fe-30-evidence-is-a-list-beside-its-viewer-d60eaf-1790892521079","branch":"adw/cqc-fe-30-evidence-is-a-list-beside-its-viewer-d60eaf","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-30-evidence-is-a-list-beside-its-viewer-d60eaf-1790892521079/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/165","provider":"codex","model":"gpt-6-sol"}]
---
# Evidence items draw on top of each other; the Evidence tab becomes a list beside its viewer

Design pass 2026-10-01. Evidence (operator-only, **not in your worktree** — do not look for it): adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`.
Today: `shots/appcfg-w1440-d0-tab1-s2.png` (the overlap), `shots/appcfg-state-evidence-item3.png`.
Target: `shots/proto-w1440-d0-tab1-s0.png`.

## The bug (measured, all three LOs on dev)

Every evidence row overlaps the next one. A DOM text-collision probe reports, on every drawer:

```
"Trust rank" ⨯ "Screenshot"     overlap 65x15
"of 8 ·"     ⨯ "Page snapshot"  overlap 33x16
"Available"  ⨯ "Accessibility snapshots"  overlap 57x16
"Evidence"   ⨯ "SCORM trace"    overlap 51x9      ← the label over the first item, in the Checks tab
```

Cause: `RunDrawer/Evidence/EvidenceList.tsx:41-77` puts a three-line `View` (kind / summary /
"Trust rank N of 8 · Available") inside a Go1d `ButtonMinimal`, which has a **fixed 40px
height** (rule R1, `cqc-fe-27`). The content overflows above and below; `li marginBottom` cannot
make room. The first item's upward overflow is what covers the "Evidence" label
(`ChecksTab.tsx:455-457`). Also `:68-70`: "·Available" — the space after the middle dot
collapses at the end of a flex item.

Your own screenshot of LO 10672378 ("Media is reachable") shows the same: "Page snapshot"
written over "Runner-polled frame state … Trust rank 6 of 8 · Available".

## The target — the prototype's Evidence tab

```
6 limitations apply to this run.  Read them                       ← pin (tinted), already exists as "View limitations in Checks"
ⓘ 1 of 4 cited items was not archived by this run (⊘). They stay listed so the gap is
  visible; open one to see why and what would fix it.              ← info banner, not a warning
┌ Most direct → least direct · ⊘ = not archived ┐ ┌──────────────────────────────────┐
│ ┌──────────────────────────────────────────┐  │ │ SCORM API trace  Read-only …     │
│ │ SCORM trace                               │  │ │ [56 API calls][21 writes][last …]│  ← viewer, right
│ │ Read-only SCORM API trace, 56 call(s)     │  │ │ #  ms   call        element value│
│ └──────────────────────────────────────────┘  │ │ …                                │
│ ┌ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┐  │ └──────────────────────────────────┘
│ ┆ ⊘ Screenshot  [unavailable]              ┆  │
│ ┆ Final rendered frame                     ┆  │  ← unavailable: dashed border, muted, badge
│ └ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┄ ┘  │
└───────────────────────────────────────────────┘
   ~260px                                            rest of the width
```

Go1d translation:

- Each item is a `View element="button"` with **auto height** (never a Go1d `Button`), border,
  radius, padding; title bold `fontSize={1}`, summary `fontSize={1}` subtle; the trust rank as a
  small token-coloured badge, not a third run-on line. Selected item: accent border + `soft`
  background, `aria-current`. Unavailable item: dashed border, muted text, "unavailable" badge,
  still clickable (it opens the "why it is missing" panel — keep today's behaviour).
- The Evidence tab is two columns at drawer width ≥ 800px: the list (~260px) and the viewer
  beside it; one column below that. The viewer shows the selected item; the first viewable item
  is selected on open.
- The same item component is used for the evidence rows inside an expanded check card
  (`cqc-fe-29`), where they render as compact chips.
- The middle dot is its own `Text marginX={1}` so "· Available" keeps its space.
- **Viewer error copy**: today a failed download always says "The link may have expired". Say
  that only for HTTP 403 from the presigned URL; for a network/CORS failure say
  "Couldn't download this file." with Retry. (The dev bucket has no CORS rule today —
  `cqc-be-46-errors-and-artefacts-reach-the-browser-e2fd2c` in the `cqc` backlog — so every download fails there; the copy must not
  blame expiry.)

## Acceptance criteria

- [ ] Red tests first: an evidence item is not rendered inside a Go1d `Button`/`ButtonMinimal`;
      the list and the viewer are siblings in a row container at desktop width; "· Available"
      text contains the space; a fetch rejection without a status shows "Couldn't download this
      file." and a 403 shows the expiry copy.
- [ ] Unavailable evidence stays listed, clickable and explained.
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

Sweep `pass2-app-drawer.mjs` (`FIXCFG=1`): `collisions_1` (Evidence tab) and the checks-tab
"Evidence ⨯ …" collision are empty for all three LOs.
