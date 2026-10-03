---
id: adw-bug-44-a-large-hunk-kills-the-run-with-e2big-3f5abf
type: bug
status: in-review
priority: 1
created: 2026-10-03
depends: []
attempts: [{"branch":"adw-bug-44-a-large-hunk-kills-the-run-with-e2big-3f5abf","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/191","provider":"claude","model":"claude-opus-5-5"}]
---
# A large hunk kills the run with E2BIG: hunk content travels in one shell argument

## Why

sabado-49 (run `sabado-49-…-828c2e-1791012552844`, 2026-10-03) ended `blocked` at
`assemble-test` with `E2BIG: argument list too long, posix_spawn '/bin/sh'`, after a
finished build (~1.5 h). The build added `backend/app/extraction/eval/corpus/quentin-ocr.json`
(1.27 MB).

`writeHunkFiles` (`src/pipeline/nodes/review.ts`) writes each hunk with
`printf '%s' '<base64>' | base64 -d > <path>`. The whole command is ONE `sh -c`
argument, so a 1.27 MB file becomes ~1.7 MB of argv, which is over macOS ARG_MAX (1 MiB).
On Linux (container / e2b kinds) the per-argument cap MAX_ARG_STRLEN is 128 KiB, so any
hunk over ~96 KB would die there. The same function feeds the review node,
review-fix (`fix-loop.ts`) and ci-round, so each of those would crash on the same diff.
`writeReviewArtifact` uses the same technique for the reviewer's raw text.

## What to build

- [ ] Content written through the `exec` seam never travels in one argument larger than a
      bounded chunk. Small content keeps the single-command write; larger content is appended
      in base64 chunks to `<path>.b64`, then decoded once. No Workspace interface change, so
      all three workspace kinds keep working.
- [ ] `writeHunkFiles` and `writeReviewArtifact` share this one helper.
- [ ] A failing chunk is still a graceful `fail()` naming the ticket, node and path.

## Tests (red first)

`test/pipeline/nodes/review-large-hunk.test.ts`:
- a ~1.5 MB hunk is written byte-exact through a real `sh -c` (red: E2BIG, the production error);
- no exec command for a large hunk exceeds 128 KiB;
- a chunk write failure mid-hunk → `fail()` naming ticket, node, path and stderr.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`
