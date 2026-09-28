---
id: adw-ctx-01-the-tool-output-digest-has-never-applied
type: bug
status: queued
priority: 1
created: 2026-09-28
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: []
---
# The tool-output digest has been a no-op since it shipped, and the journal says it worked

Parent: `adw-ctx`. Origin: `adw-cost-01-bound-what-a-tool-call-may-inject`
(PR #104, 2026-09-21).

## The defect (verified 2026-09-28)

`wrapHooksWithToolOutputDigest` returns
`hookSpecificOutput.updatedToolOutput` as a **string**
(`src/observability/tool-output-digest-hook.ts:201-206`, `:227-232`,
`:283-288`). Claude Code 2.1.209 rejects it on every call:

> PostToolUse hook returned updatedToolOutput that does not match Read's
> output shape; using original output … expected object, received string

- The full original output reaches the model.
- The rejection appears in **276 of 516** factory-run SDK transcripts under
  `~/.claude/projects/*runs-adw-*`.
- In `adw-learn-04` alone: 96 Read and 11 Bash rejections, exactly matching
  the journal's 64 `size-truncated` + 32 `dedup-suppressed` + 11 Bash
  `size-truncated` events.
- Those events log `bytesInjected` and `ok: true` as if applied (`:192-199`,
  `:218-225`). The journal claims ~92K Read bytes reached the model; the
  true figure is 1.51M.

**Latent second bug:** `readDedupState` is created once per wrapped stage
config (`:166`, built at `lanes/shared.ts:747`), not per session. After a
context hop, the fresh session's *first* reads come back as
`dedup-suppressed` ("already read"). That is verified in the journal: hop 2's
first `journal.ts`/`engine.ts` reads at 19:17:11/13 UTC. Fixing only the
shape would hand new sessions stubs for files they never saw.

## Fix (red first)

1. Return `updatedToolOutput` in the tool's own response shape. That means
   the received `tool_response` object with its content replaced, the same
   structures `readTextContentOf` / `bashOutputOf` already parse.
   - **Pin the real shape from a live CLI**, not from our own parser.
   - Include a contract test with a captured real `tool_response` for Read
     and for Bash.
2. Key dedup state by `session_id` from the hook input.
3. Journal `ok: true` / `bytesInjected` only for an accepted rewrite. If the
   CLI can report rejection, record it; otherwise say plainly in the event
   that acceptance is unconfirmed.
4. Live bar (required): one worktree dispatch whose SDK transcript contains
   **zero** `does not match … output shape` lines while the journal shows
   digest events.

## Done when

- The live bar above is met.
- In a hop, the first reads of the new session are never `dedup-suppressed`.
- Full suite, `lint` and `tsc` green.
