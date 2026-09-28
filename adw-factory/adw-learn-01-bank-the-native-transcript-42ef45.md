---
id: adw-learn-01-bank-the-native-transcript-42ef45
type: feat
status: done
priority: 2
created: 2026-09-28
review: false
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-learn-01-bank-the-native-transcript-42ef45-1790578742189","branch":"adw/adw-learn-01-bank-the-native-transcript-42ef45","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-learn-01-bank-the-native-transcript-42ef45-1790578742189/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/143","provider":"claude","model":"sonnet"}]
---
# Bank every agent session's native transcript under the run directory

> First of the `adw-learn-*` group: the operator decided 2026-09-28 that
> *"the factory learns from its own runs"* is the story this project tells
> (`ai_docs/2026-09-28-prompt-training-readiness.md`). Every ticket in the
> group is zero-token plumbing that makes the run record training-grade.
> `review: false`: hard-gated.

## The defect this closes

The single highest-value learning asset the factory produces is not banked.
Measured 2026-09-28 on the operator's disk:

| store | files | carries |
|---|---|---|
| `runs/*/.clens/sessions/*.jsonl` | 457 | hook events only: tool in, tool out, prompt, Stop with `last_assistant_message` |
| `~/.claude/projects/*adw-factory-runs-*/*.jsonl` | 383, 298 MB | **assistant text, `thinking` blocks, `tool_use`, tool results**, per turn |

The second store is the Claude Code CLI's own transcript for each session.
It lives outside `runs/`, under a path derived from the workspace cwd, and
Claude Code's housekeeping may delete it. Every hook payload already names it:
`transcript_path` on `PreToolUse`, `PostToolUse`, `Stop`, `SessionEnd`.

`makeClensHooks` (`src/observability/clens.ts:~250-360`) already has the seam:
`transcriptFetch.fetch(path)` copies a transcript out of a container or E2B
sandbox and rewrites the payload. On the **worktree** path, the default, that
seam is `undefined` and the transcript is never copied anywhere. Three of the
biggest fixes in this repo's history (the 176-turn busy-poll, refused git
calls, the artifact written to the final message) came from reading one or two
of these transcripts by hand. Nobody has read the other 380.

## Requirements

- [ ] **R1** A new journal event, `transcript`, mirroring `capture` and
      `prompt`'s sidecar-index shape byte-for-byte: `{ type: "transcript",
      sessionId, node, path, bytes, hash, ok, error? }`. The body lives at
      `runs/<runId>/transcripts/<sessionId>.jsonl`; the journal stays the
      index (Art. VIII). `hash` is `hashPrompt`'s sha256 (`prompt-sink.ts:43`),
      reused, not re-implemented.
- [ ] **R2** On every `Stop` and `SessionEnd` hook event, on **all three**
      isolation kinds, copy the transcript at `transcript_path` to the R1 path
      and append one `transcript` record. Later copies overwrite earlier ones
      for the same session (the transcript grows until the session ends); the
      **last** record per session is its status, exactly as `capture`.
- [ ] **R3** Worktree path: the copy is a host-side file read of
      `transcript_path`. Container and remote paths: reuse the existing
      `transcriptFetch.fetch` host copy rather than fetching twice.
- [ ] **R4** Failure policy is `capture`'s (Art. VI): a missing or unreadable
      transcript is journaled `ok:false` with an `error` naming the path, never
      thrown, and never blocks the agent stream or turns a green run red.
- [ ] **R5** `just usage <run-id>` and `scripts/run-scorecard.ts` are unchanged;
      add `just transcripts <run-id>` listing each session's node, bytes and
      status from the journal.
- [ ] **R6** A one-shot backfill script, `scripts/backfill-transcripts.ts`:
      for every banked run whose cLens capture carries a `transcript_path` that
      still exists on this host, copy it to the R1 path and append the R1
      record. Read-only on everything else. Runs once, idempotent, reports
      counts.
- [ ] **R7** `.gitignore` already covers `runs/`; assert nothing under
      `runs/*/transcripts/` can reach a pushed diff (the workspace is a
      sibling directory, so this is a test on the path, not on push).

## Verify

- [ ] Red first (Art. I): tests written, confirmed red, reviewed, then green.
- [ ] A fake agent session with a fake `transcript_path` → one `transcript`
      record `ok:true`, body at the R1 path, hash matches the file.
- [ ] An unreadable `transcript_path` on Stop → `ok:false` with the path in
      `error`; the node still returns `next`.
- [ ] Backfill against the current `runs/` banks every transcript whose source
      still exists and reports how many were already gone.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Out of scope

Reading or summarising the transcripts (`adw-learn-06`). Codex sessions:
`codex exec` has no equivalent transcript on disk; its cLens capture stays the
record, and a `transcript` record is simply never written for a codex node.
