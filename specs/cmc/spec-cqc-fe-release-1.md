# Spec: Content quality page in the CMC, first release (CQC frontend)

Status: draft for review (2026-09-26). This spec comes from the grilling
session (`decisions-2026-09-26.md`, D-1 to D-32), the draft API contract
(`backend-contract.md`), and the clickable prototype on real CQC data
(`prototype/`: variant B with drawer 1).

## Problem Statement

A broken SCORM course is usually discovered by a learner and reported to
support, often in Slack. Staff then triage it by hand with no reproducible
evidence. The Content Quality Checker (CQC) can launch a course in the real
Go1 player, navigate it, and produce an auditable playability report, but
nobody outside the CQC engineers can see those reports. They sit in S3 as
JSON. Support agents, content operations and partnerships managers cannot
answer three questions without an engineer:

- Is this course actually broken?
- What exactly failed, and how sure is the checker?
- Has a fixed version been checked since?

They also cannot trigger a check themselves.

## Solution

A new **Content quality** page in the CMC, shipped as an internal v1 visible
to every CMC user (no feature gate, D-1 amended 2026-09-28). It lists
every learning object the CQC has checked, one row per LO, with its latest
verdict, a check-by-check mini summary, the authoring tool, and whether the
SCORM is **connected** (a wrapper whose content is served by a remote host)
or **static**.

Staff can:
- filter the list;
- paste LO ids from Slack to trigger checks;
- re-run a check;
- open a drawer showing a single run. It gives the verdict and how strong
  the evidence is, a navigable strip of the six checks, what the run could
  not observe, the evidence behind each check (including what is missing
  and why), the run history, and the technical provenance;
- resolve or validate an LO with an outcome and a note.

Nothing is invented. Anything the backend cannot provide shows `N/A`, and
an action without a backend is disabled and says why.

## User Stories

### Access and navigation

1. As a staff user, I want a "Content quality" item in the CMC left menu, so that I can reach the page.
2. *(Withdrawn 2026-09-28 with the Statsig gate, D-1 amended: the page is visible to every CMC user.)*
3. As a staff user, I want the page to use my existing CMC session, so that I never sign in again or handle a token.
4. As a staff user, I want a clear message if the Content Quality Checker API is not configured, rejects my session (401), or says I lack the staff role (403), so that I know it isn't a content problem.
5. As a staff user, I want every filter and the open LO kept in the URL, so that I can share a filtered view or a drawer in Slack.

### The list

6. As a support agent, I want every LO the CQC has run on listed with its latest verdict (Fail-block, Needs-review, Pass), so that I can confirm whether a reported course was checked and what the result was.
7. As content ops, I want Fail-block first, then runs that failed to run, then running, then Needs-review, then Pass, each sorted by last run, so that the most urgent content is on top.
8. As content ops, I want a six-icon mini-strip per row in the fixed learner-journey order, so that I can see which check failed without opening the row.
9. As content ops, I want Needs-review shown in neutral styling and called "not verified", so that I never read "the checker couldn't see it" as "the course is broken".
10. As a partnerships manager, I want the authoring tool per LO (Rise 360, Storyline, Captivate, iSpring, Adapt, Lectora, Unknown, plus embedded tools), so that I can spot tool-specific problems.
11. As content ops, I want each LO marked Connected (served by <host>) or Static, so that I know when a problem may sit on the partner's server, not in the package.
12. As content ops, I want first detected, last run, days open, run count and runner (E2B or local) per row, so that I can prioritise old, confirmed problems.
13. As a staff user, I want "revision changed, not re-checked" on the row when Go1 has a newer revision than the one checked, so that I don't trust a stale verdict.
14. As a staff user, I want to see when a run is queued or running, with its progress, on the row and in an "In progress" strip, so that I know a check is under way.
15. As a staff user, I want a run that crashed shown as "Check failed to run", with its reason, never styled as Fail-block, so that instrument failures aren't mistaken for broken content.
16. As content ops, I want summary cards (Open, Fail-block, Revision changed, Partners affected), so that I can see the state of the backlog at a glance.
17. As content ops, I want All / Open / Resolved tabs with counts, so that I can work the open backlog and review resolutions.
18. As a staff user, I want a search by LO id or run id, so that I can jump to a course from a Slack message.
19. As a staff user, I want multi-select dropdown filters (Status, Runner, Authoring tool, Connected, Partner) in one row, showing "Any" when empty and a count per option, so that I can narrow the list without a wall of chips.
20. As a staff user, I want "Clear all filters" whenever any filter is active, so that I can get back to the full list.
21. As a staff user, I want the Partner filter visible but disabled with a reason until partner data exists, so that I know it's coming and why it's unavailable.
22. As a keyboard user, I want each row to open from a real button, so that I can triage without a mouse.

### Triggering checks

23. As a support agent, I want to paste one or more LO ids (comma, space or newline separated) into a bar above the list and press Check, so that I can check what a customer reported in seconds.
24. As a staff user, I want a live estimate of the count, cost (about $2 per LO), duration and parallelism, and a warning for invalid ids or a batch over the limit, before I trigger, so that I don't spend money by accident.
25. As a staff user, I want several ids sent as one batch, and a toast saying how many were queued and which were rejected and why (not a number, already running, over the limit), so that I know what happened.
26. As a staff user, I want Re-run on each row and "Run check again" in the drawer, disabled while a run is in progress, so that I can check whether a fix worked without starting duplicate runs.
27. As a staff user, I want the list and the open drawer to refresh while any run is in progress and stop when all are finished, so that I see the result without reloading.

### The drawer (one run)

28. As content ops, I want a right-side drawer to open when I click a row, with the list still visible behind it and the open row highlighted, so that I keep my place.
29. As content ops, I want to step to the next or previous LO with j/k or ↑/↓ buttons, keeping the tab I was on, so that I can triage a queue quickly.
30. As a staff user, I want Esc, the close button, or a click on the backdrop to close the drawer and return focus to the row, so that I can continue with the keyboard.
31. As a support agent, I want the header to show the LO, the verdict, how strong the evidence is (with an explanation per level), and a context line (checked date · revision · launch mode with tracking · runner), so that I know how much to trust the verdict.
32. As a support agent, I want the checker's summary labelled as such and collapsed to two lines with "More", so that the header stays short.
33. As content ops, I want a case line (open or resolved, days open, first detected, run count, partner), so that I understand the history at a glance.
34. As a staff user, I want a run picker in the header when the LO has several runs, so that I can compare past results.
35. As content ops, I want a strip of the six checks in learner-journey order, where clicking a step scrolls that check to the top, expands it, shows its evidence inline and collapses the others, so that the overview doubles as navigation.
36. As content ops, I want the drawer to open focused on the first failed check (else the first not verified), so that what matters is already on screen.
37. As content ops, I want each check to show the check's reasons word for word, its severity, whether it counts toward the verdict, and declared vs observed values on demand, so that I understand why it has that status.
38. As a staff user, I want what the run could not observe listed word for word on the default tab, and pinned as a one-line link on the other tabs, so that I never miss the conditions on the verdict.
39. As a staff user, I want the evidence behind each check listed by kind, ranked by trust (most direct first), with the agent's own account clearly labelled as the lowest trust, so that I know which proof to rely on.
40. As a staff user, I want to open the SCORM API trace as a table (calls, writes, last status write), so that I can see whether the course ever reported completion.
41. As a staff user, I want to step through the accessibility snapshots of the player, so that I can see what the checker saw at each step.
42. As a staff user, I want to see the final screenshot when it is archived, so that I have visual proof.
43. As a staff user, I want evidence that wasn't archived shown greyed out (⊘), but still clickable, opening a panel with N/A, the cited file, its trust rank, why it's missing and what would make it available, so that gaps are honest and explained.
44. As an engineer, I want a Technical tab with launch, profile, dependency mode, agent, cost, duration, runner (with uncommitted-changes flag), code version, prompt hash, extraction and navigation stats, and the raw report and metadata JSON, so that I can debug the checker.
45. As an engineer, I want the fields a run didn't record listed on one "Not recorded for this run" line, so that N/A doesn't take up a grid of tiles.
46. As a staff user, I want banners in the drawer for: a newer attempt that failed to run (with a link to it), a check in progress (with a link to follow it), a revision changed, and a simulated or reopened state, so that the run I'm reading is never mistaken for the latest truth.
47. As a staff user, I want a run with no report to still show its Runs and Technical tabs, so that I can diagnose why it produced nothing.
48. As a support agent, I want "Copy link to this run" to copy a stable link to the run, so that I can paste reproducible evidence into a ticket.
49. As a staff user, I want "Open in player" in the drawer header, so that I can reproduce the problem myself.

### Resolve and validate

50. As content ops, I want Resolve… on open LOs (on the row and in the drawer), with an outcome (fixed and verified, validated manually, false positive, accepted as is) and a required note, so that the team knows why a case was closed.
51. As content ops, I want "fixed and verified" to be allowed only when the latest run passes, so that nobody claims a fix without proof.
52. As content ops, I want Validate on passing LOs, so that I can record a human confirmation.
53. As content ops, I want a resolution to show who resolved it, when, and the note, and to reopen automatically when a newer run doesn't pass, so that the Resolved tab stays trustworthy.
54. As content ops, I want Reopen, so that I can undo a wrong resolution.

### Accessibility and robustness

55. As a screen-reader user, I want properly marked-up tabs, checks and filters (tabs with their panels, arrow-key and Home/End navigation, current-check and pressed/expanded states), focus kept inside the open drawer, and labelled regions, so that the page is fully operable.
56. As a low-vision user, I want text contrast of at least 4.5:1 and status never shown by colour alone (icon and word too), so that I can read every state.
57. As a staff user, I want quickly switching LOs to never show the previous LO's run, and an expired or failed download to show Retry instead of broken content, so that the drawer never lies.

## Implementation Decisions

**Page shape**
- One route, "Content quality", under the CMC's flat route table, with one
  left-menu item.
- The route renders a single root implementation from the feature folder:
  the list container plus the drawer.
- There is no Runs tab and no separate run route. The drawer is driven by
  the URL: `lo` (latest run of that LO) or `run` (a specific run).
- Stable run links resolve through `run`.

**Gating (none — amended 2026-09-28)**
- No feature gate. The CMC is an internal tool and Go1 ENG ships new CMC
  features without Statsig (Statsig is used on customer-facing apps). The
  page ships as an internal v1, visible to every CMC user; each slice is
  live as soon as it merges.
- The CQC base URL follows the existing runtime-config path: container env
  (`APP_CQC_API_URL`) → nginx SSI → `window.GO1` → app config. The
  `.env.example` gets the new key, and the shared k8s template needs the
  new env var (outside this repo). Unset, the page says the API is not
  configured.
- The route path is `/quality`: the menu marks items active by path prefix,
  so a path starting with `/content` would light up the "Content" item.

**Service boundary**
- A new plain typed CQC service owns a **dedicated axios instance** with the
  CQC base URL. It sends the signed-in staff user's own Go1 JWT, the one
  already held as the current user's `jwt`. No shared secret goes into the
  CMC env.
- The service returns typed DTOs. It is consumed through the existing
  `useQuery` / `useMutation` hooks, not the normalized entity layer.
- The service must not depend on component-owned types.

**API consumed** (draft contract; types to be regenerated from OpenAPI when
the backend publishes it):
- `GET /cqc/config` feeds the trigger defaults, estimates and limits.
- `GET /cqc/content`:
  - filters: `status`, `environment`, `authoring_tool`, `connected` (all
    repeatable), `case_state`, `q`, `offset`, `limit`
  - returns LO-level rows plus `summary` and `facets`
  - aggregation per LO is done by the backend
- `GET /cqc/content/{lo}/runs` returns the run history.
- `GET /cqc/checks/{run_id}` returns `run`, `metadata` (run-metadata.json),
  `report` (playability-report.json V1, rendered in full), and the names of
  the artefacts.
- `GET /cqc/checks/{run_id}/artefacts/{name}` returns `{ url }`, a fresh
  presigned URL. It is fetched on demand and never cached across sessions.
- `POST /cqc/checks` (one LO) and `POST /cqc/batches` (several LOs) return
  202. The batch response includes the rejected ids and reasons.
- `POST` / `DELETE /cqc/content/{lo}/resolution` resolve and reopen.

**Key type shapes** (from the prototype's mock API, trimmed):

```ts
interface ContentRow {
  lo_id: string;
  latest_run: RunSummary | null;      // latest succeeded run on Go1's current revision
  active_run: RunSummary | null;      // queued | provisioning | preflight | running | finalising
  last_failed_run: RunSummary | null; // newer than latest_run: "Check failed to run"
  revision_changed: boolean | null;
  first_detected_at: string | null; last_run_at: string | null; run_count: number;
  case_state: "open" | "resolved" | "none";
  authoring: { tool: string; embedded_tools: string[] } | null;
  connected: boolean | null; content_hosts: string[] | null; // SCORM wrapper vs static
  resolution: { outcome: string; note: string; by: string; at: string; run_id: string;
                active: boolean; reopened_by_run_id: string | null } | null;
}
// RunSummary adds: lifecycle_status, status_reason, playability_status, claim_strength,
// environment, git_dirty, captured_at, cost_usd, origin ("backend" | "import"),
// asset_portal_id, package {vault_uuid, go1_asset_revision},
// checks_summary [{check_id, criterion, status}], progress, triggered_by, batch_id
```

**Domain rules** (one pure module, owned by the feature, tested directly):
- **Journey order:** launch, navigation, completion, runtime, media,
  resume. Unknown criteria go last.
- **Check labels:** e.g. `completion-recorded` → "Completion is recorded".
  An unknown check id is humanised, never shown raw.
- **Status vocabulary:** Pass → "passed", Fail-block → "failed",
  Needs-review → "not verified", anything else → "Unknown status: X". The
  unions stay open-ended.
- **Evidence availability:** for each cited evidence item, work out
  available / unavailable, the resolved file, the viewer kind (SCORM table,
  snapshots stepper, image, text, raw) and the trust rank (1–8).
  Accessibility snapshots are added as an extra item. A narrative with its
  cited file missing falls back to the agent's last message, with a note
  saying so.
- **Missing-evidence reasons:** one explanation per evidence kind.
- **Default focused check:** the first Fail-block, else the first
  non-Pass.

**Components**
- The list uses the existing generic table with loading, empty and error
  states.
- The filters reuse the CMC's shared filter multi-select (Go1d
  `MultiSelect`, "Any" default, stays open while ticking).
- The drawer uses the CMC's own `Drawer` at size md (900px), extended for:
  - focus moving to the heading on open and on LO change;
  - focus trapped inside the drawer;
  - focus returned to the row opener on close;
  - Escape handled before any input guard.
- The resolve form is a modal owned by its trigger.
- Every Go1d component and prop is checked against the installed Go1d
  revision before use, as the repo conventions require.

**Drawer layout (drawer 1)**
- A sticky header with:
  - breadcrumb, position ("3 of 27"), previous/next buttons, close;
  - "LO <id>" plus the title from Go1 (`N/A` when unknown);
  - the verdict pill, the claim-strength explanation, the run picker (only
    with more than one run), and "Open in player";
  - the context line, the clamped checker summary, and the case line.
- Below the header, the check strip.
- Tabs:
  - **Checks** (default): limitations first, word for word; then the checks
    in journey order. When a check is focused, the others are compact.
  - **Evidence** ("N of M available"): a trust-ordered list with a viewer,
    and one banner stating the archive gap.
  - **Runs**: only with more than one run.
  - **Technical**.
- A sticky footer: "Copy link to this run", the keyboard hint,
  Resolve / Validate / Reopen, and "Run check again" (primary).

**Data fetching**
- Detail and history are fetched together and committed only if they still
  match the current selection, so stale responses are dropped.
- Polling refreshes the list and the open run while any run is
  non-terminal, and stops when all are terminal. The interval is 10 s in
  the real app.
- Downloads check the HTTP status. The SCORM trace is parsed defensively.
  Cached failures offer Retry, which asks for a fresh URL.

**Go1 enrichment**
- Titles come from the existing content-gateway service, batched per
  visible page. The current revision, used for "revision changed", comes
  from the backend row.
- Missing records show `N/A`; the row stays.
- "Open in player" reuses the CMC's existing one-time-token player flow.

**Shared limitations**
- The prototype groups limitations that appear in 60% or more of reports.
  The frontend cannot compute that in production.
- The release shows all limitations word for word, ungrouped, unless the
  backend adds a `shared` flag per limitation (requested in the contract).

**Stack constraints**
- React 16.8, TS 3.5: no optional chaining, no nullish coalescing, no
  `Array.flat` / `flatMap`, no `Promise.allSettled`.
- Absolute `src/...` imports across ownership boundaries.
- Containers own effects, queries, mutations and history; components get
  typed props and callbacks.

## Testing Decisions

- **Good tests** drive the page the way a user does: click, type, press
  keys. They assert what is visible or accessible (text, roles, names,
  pressed/expanded/current states, focus). They mock only at the service
  boundary. They never assert internal state, hook calls or helper
  internals.
- **Primary seam (the only one across modules): the CQC service module.**
  Tests render the Content quality route inside a router with the service
  mocked. They cover:
  - the menu item and the page header;
  - not configured / 401 / 403 / 500 with Retry;
  - loading, empty, and filtered-empty with "Clear all filters";
  - list order and the mini-strip accessible names;
  - multi-select filters updating the URL and resetting the offset;
  - the paste bar: estimate, invalid ids, single check vs batch,
    rejected-ids toast;
  - Re-run disabled while running, and polling stopping on terminal runs
    (fake timers);
  - the drawer opening from the row button, focus on the heading, j/k
    keeping the tab, Esc returning focus to the row;
  - a step click focusing a check with inline evidence;
  - limitations visible on Checks and pinned elsewhere;
  - unavailable evidence opening the N/A panel with its reason;
  - SCORM trace rendering, and Retry on a failed download;
  - a run with no report keeping Runs and Technical;
  - stale-response protection (resolve B before A; A must not render);
  - the resolve modal (validation, fixed-and-verified disabled when not
    Pass, success banner, reopen);
  - "Copy link to this run".
- **Secondary seam: the pure domain module**, which owns rules worth pinning
  alone: journey order, labels and humanised fallback, status vocabulary,
  evidence availability and resolved file, and the default focused check.
- The service itself gets a thin test of exact paths, params and headers
  against a mocked axios instance, following the existing service tests.
- **Prior art:**
  - page tests that mock services and render the route in a router, such as
    the Translated content and Flagged content list pages;
  - the CMC `Drawer` and `FilterMultiSelect` behaviour as used on the
    retirement-request and partner-agreement pages;
  - the repository's testing reference card (cover loading, error and
    retry, URL filters, modal flows, and focus-relevant behaviour).
- **Verification:**
  - focused `npm test` on the feature pattern;
  - TSLint on the changed files;
  - `npm run build`, because routing, the menu, runtime config and a new
    are touched.

## Out of Scope

- The case workflow beyond resolve / validate: owner, partner notification,
  response due, partner clock, decision log.
- Partner communication ("Route partner fix"), Suppress / Restore, and
  availability actions.
- Affected learner attempts.
- Partner-library bulk runs (a provider picker listing all interactive LOs).
  Paste-many is in scope.
- Launch mode `launch` (blocked on CQC blocker B2). It shows as disabled
  with its reason.
- Per-step screenshots (they don't exist; that needs a runner change) and
  the screenshot upload itself (a CQC publisher fix). The screenshot viewer
  is built and shows "unavailable" until then.
- A package classification computed in the frontend: the backend provides
  connected / static.
- A judge-model `claim_strength`: the frontend only displays the field.
- Pedagogical or metadata quality, translations, and the other
  vision-prototype pages.

## Further Notes

**Backend dependencies** (see `backend-contract.md` for the full list):
- the JWT-validating Lambda API, recording `triggered_by`;
- the per-LO aggregation endpoint, with summary and facets;
- the artefact endpoint returning `{ url }`;
- the failure record for crashed runs;
- the resolution store;
- the authoring and connected fields on every run;
- a `shared` flag per limitation;
- publisher fixes: screenshots, `scorm-log.json`, a non-null run timestamp,
  and a consistent revision key.

**Backend decisions that change this spec (2026-09-26).** Canonical API:
`content-quality-checker/docs/backend/spec-cqc-backend-release-1.md`,
decisions `BE-n` in the same folder.
- **Revision (answered).** "Revision" is the Go1 SCORM asset revision,
  `package.go1_asset_revision`. `revision_changed` compares it with the
  highest revision in `go1-scormassets` (BE-32, BE-34). It stays `null`
  (shown as N/A) until Go1 DevOps grants the backend read access.
- **Which LOs are listed (supersedes D-4).** Only runs made by the new backend,
  plus legacy runs explicitly imported (`origin: "import"`) (BE-5, BE-8). The
  Runner filter values become `fargate`, plus `local` / `e2b` for imports.
- **Release 1 has no active runs.** `active_run` is always `null`, so the
  "In progress" strip, polling and Re-run stay inactive until the trigger ships
  (BE-1, BE-23).
- **Verdicts are navigation-only for now.** Expect almost every run to be
  Needs-review (`/config` `verdict_scope`) (BE-9).
- **Connected** comes from the backend's package classification (BE-37).

**Open questions:**
- Does "open" include Needs-review? The prototype counts it: 27 of 35 LOs
  are "open".

**Ship plan:**
1. Internal v1: the page, list and drawer read path against the
   backend's read endpoints. The list only holds imported or backend runs
   (BE-5, BE-8), so it starts with whatever has been imported.
2. The trigger (paste bar, Re-run) and polling, once the POST endpoints
   exist.
3. Resolve, once the store exists.

Each step works on its own. Unavailable actions are disabled and explain
why.

The prototype (`docs/cqc/prototype`, `run.sh`) is the visual reference and
runs on real S3 data. Delete it after this release ships.
