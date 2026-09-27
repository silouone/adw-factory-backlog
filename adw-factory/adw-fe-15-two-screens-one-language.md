---
id: adw-fe-15-two-screens-one-language
type: feat
status: done
priority: 2
created: 2026-09-14
depends: [adw-fe-14-grid-screen-console-design]
attempts: [{"runId":"adw-fe-15-two-screens-one-language-1789714388413","branch":"adw/adw-fe-15-two-screens-one-language","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-15-two-screens-one-language-1789714388413/workspace","outcome":"blocked","provider":"claude","model":"sonnet","pr":"https://github.com/silouone/adw-factory/pull/76"}]
---
# Align the run screen with the board — same words, same marks, same numbers

> The factory has two screens. `adw-fe-13` designed the run screen,
> `adw-fe-14` designs the board, and they were designed four weeks apart by
> different sessions. This ticket is the reconciliation, and it is **entirely
> a diff against `render-run.ts`** — no new capability.
>
> Depends on `adw-fe-14` so the language is settled before it is copied, and
> transitively on `adw-bug-01` so §2 and §4 are measurable at all.

## Salvaged and completed by hand 2026-09-18 — outside a run

The factory run (`adw-fe-15-two-screens-one-language-1789714388413`) reached
green gates and then **blocked in the review↻fix loop**, not on the work: the
spec reviewer raised one finding — §2's figure was never re-derived — and the
fix agent had no way to satisfy it, because the banked journals the figure
needs (`runs/`, gitignored) are not reachable from an agent sandbox. Two
rounds of fix produced two identical re-raises of the same finding and the
run exhausted its ceiling. The diff itself was never in doubt.

**That finding is now closed, for real.** `just model-tool-split` was run on
the operator's own checkout, against 43 banked runs:

> **74.6% model time** (965.9m model / 328.0m tool, across 105 agent blocks).

`adw-fe-13`'s body carries the figure, its derivation and the verdict — the
thesis held, the number did not (`~90%` overstated it by ~15 points). §2's
boxes are now genuinely checked, not waived.

§1, §3, §4, §5 and §6 were already complete in the run's own diff.

## Why this is its own ticket

Two screens that disagree about the same lane, the same colour and the same
token count are worse than one screen. Every item below is a place where the
board will be right and the run screen will still be wrong the moment
`adw-fe-14` merges.

## 1. `workspace` → `code` — **DONE by hand 2026-09-15, outside a run**

`render-run.ts` labelled the shared non-agent lane
`⌘ workspace / deterministic · no tokens`.

- [x] Label it **`code`**, matching the board. `render.ts` now exports
      `roleLabel` / `roleTitle`; `render-run.ts` imports them, so one table
      names the lane on both screens (Art. VIII).
- [x] **Display only.** `WORKSPACE_LANE` remains the data key — `gantt.ts` owns
      it, and both screens read it.
- [x] Keep the honest caveat reachable: `gates` lives in this lane and is the
      **target's** command, not factory code. Carried as the lane name's
      `title`, from the same `ROLE_TITLE` entry the board uses.

The remaining sections (§2–§6) are still owed. This ticket stays
`queued` for them — no attempt was spent on §1.

## 2. Re-derive "~90% model time" — it may be an artifact

`adw-fe-13` §5 calls the model-vs-tool colour split **"the page's thesis"** and
justifies it with *"this session measured agents at ~90% model time"*.

That measurement was taken while `metrics.ts` was reporting **100% model time
on every node of every run** — see `adw-bug-01` Defect B.

- [x] Once `adw-bug-01` has landed, **re-derive the figure** from banked
      journals and record what it actually is. **Done 2026-09-18:
      74.6% model time**, via `just model-tool-split` over 43 banked runs
      (965.9m model / 328.0m tool / 0.0m blocked-on-subagent, across the 105
      agent blocks that carry a resolved split).
- [x] **If the real split is materially different, say so and adjust.** It is
      materially different and it is said so, in `adw-fe-13`'s amendment: the
      thesis **held**, the number did **not**. Model time still dominates
      ~3:1, so the colour split stays justified; but "~90%" overstated it by
      some 15 points and tool time is a quarter of agent wall-clock, not a
      rounding error.
- [x] Amend `adw-fe-13`'s ticket body with the corrected figure and a pointer
      to `adw-bug-01`. Done — `## Amendment 2026-09-18 (adw-fe-15) — the
      figure, re-derived` carries the table, the derivation command and the
      verdict.

**Why the run itself could not close this, and what did.** `runs/` is
gitignored and does not exist inside an agent sandbox (`ls runs` → ENOENT at
plan time, at build time, and again in each review-response pass). The run was
therefore correct to refuse to invent a number — the checklist above forbids
exactly that. What it could build, it did: `src/web/model-tool-split.ts`
(pure; `aggregateModelToolSplit`/`modelTimePct` sum `metrics.time` across every
block of an enriched `GanttView`) and the thin edge
`scripts/run-model-tool-split.ts` + `just model-tool-split`, mirroring
`scripts/run-metrics.ts`'s "quote the script, never a remembered number"
precedent.

The operator then ran that tool on the real checkout, where `runs/` lives, and
the figure above is its verbatim output. The tool the run built is what
produced the number — the run's work was the deliverable, the data access was
the only missing ingredient, and it was never something a sandboxed agent
could have supplied.

## 3. The two chips that must agree — same rule as the board

`render-run.ts`'s header chips are `[dur][turns][billed][context]`, and its
per-node card stats show `billed` and `context` separately.

- [x] If a **cost chip** is added here — and it should be, the run screen is
      where an operator asks "what did this cost" — it obeys every rule
      `adw-fe-14` §5 sets: priced from the **full breakdown**, per block at its
      own model's rate, an unknown model yields **no** estimate, always `~`
      with the rate and its date in the tooltip. Added as the header's 5th
      chip; an unpriced run shows `—` with the board's own `est-none`
      wording verbatim, never a guessed rate.
- [x] **Any token figure shown next to a price uses the same denominator as
      the price.** `billedTokens` beside a cache-inclusive cost reads as
      wrong by ~260× on this factory's runs. The cost chip sits immediately
      after `context` (the same cache-inclusive denominator it prices from);
      `billed` stays non-adjacent.
- [x] Keep `billed` and `context` as their own labelled chips if useful — the
      run screen has room for both, and there they are unambiguous. The rule
      is about **adjacency to a price**, not about hiding the number.
      Unchanged.
- [x] Use one shared formatter and one shared estimator across both screens
      (Art. VIII). Two implementations of a price will diverge. `estimateUsd`/
      `rateFor` moved from `card.ts` into `pricing.ts` (now imported by
      both); `fmtUsd` exported from `render.ts`, imported by `render-run.ts`.
      Found and fixed in the same move: `estimateUsd`'s basis string never
      carried `PRICING_AS_OF`'s date, contrary to `adw-fe-14` §5's own rule —
      now fixed for both screens from the one function. **Second correction
      from an earlier pass**, made in response to a review finding: the "no
      priced model" tooltip wording had been left as two independent string
      literals (`render.ts`'s inline string, `render-run.ts`'s own
      `NO_ESTIMATE_TITLE` constant) — a rewording of one would have silently
      desynced the two screens' "same words" claim, guarded only by a test,
      not by structure. Moved to one export, `NO_ESTIMATE_TITLE` in
      `pricing.ts`, imported by both.

## 4. Tool dots — one visual language, two scales

After `adw-bug-01`, both screens have real tool-call dots for the first time.

- [x] Keep the run screen's foot-ticks — at full width they carry more than the
      board's card-scale dots can. Unchanged.
- [x] **The captured-vs-absent distinction must read the same on both.** The
      run screen already has `cap-none` ("no capture recorded"); the board uses
      solid-span-vs-dotted-rail. Make sure an operator moving between them
      reads the same fact the same way. Checked directly: both already
      signal "missing capture" with the same `--warn` colour token (the
      board's dashed border, this screen's italic caption) — no code change
      needed, now pinned by a grep-level test so a future palette change
      can't silently desync them.
- [x] Same dot colour rule on both. The real gap: the board's `.mc-dot`
      overrides its lane-hued default with the block's outcome colour
      (`blocked`/`warn`/`live`); this screen's `.ticks .dot` was
      unconditionally lane-hued, so a failed block's tool-call ticks never
      turned red here. Fixed — same rule (outcome overrides lane hue), this
      screen's own established `bad`/`warn`/`live` vocabulary.

## 5. Shared design system — extract it

Both screens now carry the same `:root` token block, the same `ui-monospace`
stack, the same chip, pill and focus rules, copy-pasted.

- [x] Extract the shared palette, type and chip/pill CSS into one module both
      renderers use (Art. VIII). New module `src/web/theme.ts` (`themeCss()`)
      carries the palette, base type, focus rule — and now the chip/pill
      base too. **Correction from an earlier pass of this same ticket**,
      made in response to a review finding: chip/pill CSS was first left
      out, reasoning that a shared, unscoped `.chip{...}` rule would leak
      onto the board's filter-toggle buttons (`.gh-r .chip`, a different,
      clickable widget with its own `:hover`/`.on` states). Re-checked that
      reasoning directly against the actual rules rather than accepting it:
      `.gh-r .chip` is MORE specific than a bare `.chip` (two classes vs.
      one) and already redeclares every property the shared base would set,
      with the one value that differs (`background:var(--panel2)`, not
      `--bg`) already overridden in place — so it wins the cascade
      unconditionally regardless of source order, and a shared base cannot
      change that widget's rendered appearance. Extracted
      `.chip,.kpi{display:inline-flex;align-items:center;border-radius:999px;
      border:1px solid var(--line);background:var(--bg);
      font-variant-numeric:tabular-nums}` into `theme.ts`; `render-run.ts`'s
      `.chip` and `render.ts`'s `.kpi`/`.gh-r .chip` now only carry their own
      padding/gap/icon-size/colour deltas on top. No markup change was
      needed — this was a CSS-only move after all, not the "markup, not
      CSS-only" change the earlier pass believed it required. `.st`/
      `.tag-st` (the status pill) stays unmerged, still on purpose — that
      pair genuinely differs in vocabulary AND visual treatment (a
      background tint vs. none), unlike this chip/pill pair which only ever
      differed in padding/gap.
- [x] `fmtMs`, the outcome glyphs and the outcome→class mapping are duplicated
      too. One home each. `render.ts`'s own private `fmtMs` deleted — it now
      imports `timeline.ts`'s (which already said "this module is now the
      one place that owns time-axis math," but `render.ts` hadn't switched
      over yet). **Found a real bug in the process, not just duplication:**
      the two copies zero-padded seconds differently (`1m 05s` vs `1m 5s`) —
      same duration, two different strings depending which screen rendered
      it. Fixed by making `timeline.ts`'s canonical. `card.ts`'s
      `kindOfBlock` (a line-for-line copy of `timeline.ts`'s `kindOf`)
      deleted in favor of the import; `outcomeOfBlock` now delegates to
      `timeline.ts`'s `outcomeClass` and only translates the result into the
      board's own CSS vocabulary (a real, load-bearing difference, kept).
- [x] Both screens keep the quality floor: responsive to ~900px, visible
      keyboard focus, `prefers-reduced-motion` respected. Unchanged
      (`prefers-reduced-motion` now shared via `theme.ts`; each screen keeps
      its own `@media(max-width:900px)` layout rule, which is inherently
      screen-specific).

## 6. Navigation

- [x] The run screen's `← all runs` crumb returns to the board **with the
      operator's filter intact** if `adw-fe-14` put the filter in the URL. If
      it did not, say so — a crumb that silently discards the filter is a small
      lie the operator pays for on every trip.

      **Saying so: it did not.** Checked both sides directly. The crumb
      (`render-run.ts`) links to `gridUrl`, which `server.ts` builds as
      `/?token=...` — no filter params. The board's project/health filter
      (`render.ts`) lives in a client-side JS closure, never written to
      `location.search` or any URL — `location.search` only forwards
      `token`/`id` to `/events`/`/usage.json`. There is therefore no filter
      state on either side of the crumb to round-trip. No code change made
      here: moving the filter into the URL is itself new capability on the
      board (explicitly out of scope — filters are `adw-console-01`'s).
      Flagging this for `adw-console-01` or a dedicated follow-up rather
      than building it in this ticket.

## Verify

- [x] No screen says `workspace` in operator-facing text; `WORKSPACE_LANE`
      still appears as a data key in `gantt.ts`. True since §1 (2026-09-15);
      re-checked with a grep-level regression test this build.
- [x] The re-derived model/tool figure is recorded in `adw-fe-13`'s body with
      its derivation, and this ticket states whether the thesis held.
      **Done 2026-09-18, by the operator, outside a run** — 74.6% model time
      over 43 banked runs; `adw-fe-13`'s amendment section carries the figure,
      the table, the reproducing command and the verdict (thesis held, number
      did not). The sandbox could not reach `runs/`; the operator's checkout
      could.
- [x] Palette, type and chip rules exist once, imported twice — grep proves
      it. Proven by `test/web/render-run.test.ts`'s and
      `test/web/render.test.ts`'s `themeCss` import grep tests (palette,
      type, focus, reduced-motion) plus the new chip/pill base grep test —
      see §5 above.
- [x] A `feat` run and a `bug` run both render legibly on both screens, and
      the same run reads consistently across them. Verified via the existing
      hand-built-fixture unit suites (no live `just web` run possible — no
      `runs/` directory exists in this sandbox): both `render.test.ts` and
      `render-run.test.ts` already carry multi-lane fixtures exercising the
      full lane/kind/outcome matrix this ticket's changes touch (cost chip,
      dot colour, shared theme), and all pass.
- [x] `bun run lint && bunx tsc --noEmit && bun test`. Green: lint clean,
      typecheck clean, 2275 pass / 4 skip (pre-existing, external
      credentials — codex-cli smoke and E2B) / 0 fail across 84 files.

## Out of scope

New capability on either screen. Filters, the dependency view and host
RAM/CPU stay with `adw-console-01`.
