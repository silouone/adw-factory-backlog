---
id: adw-m7-04-codex-ci-round-capture
type: feat
status: done
priority: 3
created: 2026-07-20
epic: adw-m7
depends: [adw-m7-02-codex-capture]
attempts: [{"runId":"adw-m7-04-codex-ci-round-capture-1789402398499","branch":"adw/adw-m7-04-codex-ci-round-capture","workspace":"/Users/silouane/adw-factory/runs/adw-m7-04-codex-ci-round-capture-1789402398499/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/41","provider":"claude","model":"sonnet"}]
---
# Codex CI-repair-round capture stamp + verdict

> Minted 2026-07-20 from adw-m7-02's final-validator finding #5. A post-epic
> follow-up: adw-m7 is DONE (exit criteria met on the fresh-dispatch path);
> this closes the resume-path capture gap. Coarse — refine at pickup.

## Scope

adw-m7-02 delivered codex cLens capture on the **fresh-dispatch** worktree lane.
The **CI-repair (resume)** path has a gap: the `-c hooks.*` args ride resume too,
so hooks DO fire and append to the SAME `<sid>.jsonl` in the original run's
`.clens` sink — but `ci-round.ts` never calls `checkCodexCapture`. Net effect:
the ci-round-appended records are NOT `adw_ticket_id`-stamped (partial stamp),
and the round gets no positive `ok:true` verdict (only the negative-only
`checkCaptureCompleteness` runs there). Unlike Claude, which re-registers its
SDK hook bag per round via `makeHooks` and gets a fresh in-process `ok:true`.

## Fix direction (from the m7-02 validator)

Call `checkCodexCapture(dirname(attempt.workspace), sid, ticket.id, append)` in
the **codex branch** of the ci-round mini-lane — an idempotent re-stamp of the
shared session file (fresh records already carry the same ticket id) plus an
`ok:true` verdict on the round's own journal. Mirror the fresh-dispatch call
site (`cli.ts` runSelected). `sid` = the resumed session id (`lastAgentSessionId`
of the prior run's journal, already resolved in ciRound).

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; a unit test that a codex
ci-round stamps + journals `capture ok:true`. **LIVE bar (billed, operator-gated):**
a codex ticket driven through a CI-repair round ends with a fully-stamped session
file + an `ok:true` round verdict. **Budget the m4-04 marker-koan** — a resumed
codex agent treats a forced CI-red as intentional → E8 retrigger → blocked-after-
cap; design the red signal to look like a genuine failure, or accept the koan and
verify the capture stamp/verdict regardless of the round's repair outcome.
