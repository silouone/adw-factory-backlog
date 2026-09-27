---
id: adw-m3-05-replay-audit
type: chore
status: done
priority: 2
created: 2026-07-14
epic: adw-m3
depends: [adw-m3-02-otlp-exporter, adw-m3-03-traceparent, adw-m3-04-clens-capture]
attempts: []
---
# Replay audit — M3 exit gate

## Context

The epic's exit criterion made operational: answer "what did run X do and
why" from artifacts alone — journal + spans + captured sessions — without
rerunning anything (success metrics 3 and 5, S5.4, Art. VI "a run that
cannot be replayed from its artifacts is a defective run"). Uses the
scratch repo from M2 (`silouone/adw-m2-scratch`, still exists) or a fresh
one; every live command launches `env -u GITHUB_TOKEN -u GH_TOKEN` (M2
live finding: a stale shell token 401s gh and Bun cannot sanitize child
env in-process).

## Requirements

- [x] One full toy chore runs the complete lane with all three layers ON
      (tracing + filesystem exporter + cLens capture), reaching a real PR
      or an explained terminal state — **run `m3s-001-1784104068186`,
      green, PR https://github.com/silouone/adw-m2-scratch/pull/4**
- [x] A written reconstruction lands in this ticket's body: the node
      timeline, gate results, repair rounds, agent sessions and PR —
      derived ONLY from `runs/<runId>/` (journal.jsonl, spans/, sessions/)
      plus the ticket file's git history; no rerun, no memory of the run
      — see "Reconstruction" below
- [x] Artifact-completeness checklist, verified not assumed (metric 3):
      - every journaled node has exactly one span and vice versa
      - the span tree is fully parented (root ← node spans; agent spans
        typed AGENT; TRACEPARENT derivable per agent span from the
        exported OTLP as `00-<traceId>-<spanId>-01` — **amended** from
        "extractable from the journaled env": the env is not a journaled
        artifact; extraction is validated by the W3C-propagator unit
        proof, and live in-process receipt is a documented M6 item, see
        Gaps)
      - every agent node has a captured, ticket-tagged cLens session,
        referenced by a journal `capture` record
      - the PR body quotes the journal's headline stats AND session ids
      - abnormal-exit flush: kill one run mid-flight; journal is valid
        JSONL up to the kill and every pre-kill span file exists (S2.6)
        — **run `m3s-002-1784104446010`, SIGKILL at the commit node**
- [x] Every gap found (missing event, orphan span, untagged session)
      becomes a fix ticketed and closed INSIDE epic adw-m3 before the epic
      closes — **no artifact defect found**; two findings recorded under
      "Gaps & findings" (one checklist-wording amendment applied above,
      one verification probe deferred to M6 with rationale)
- [x] The reconstruction procedure is written up in this body as the
      runbook for future audits (M6 measures metric 3 with it)

## Reconstruction — run m3s-001-1784104068186 (from artifacts only)

Sources: `runs/m3s-001-1784104068186/{journal.jsonl, spans/, .clens/sessions/}`
+ `tickets/m3s-001.md` git history in the target clone. Nothing rerun.

**Identity.** Journal line 1: `run-start` for ticket `m3s-001`, run
`m3s-001-1784104068186`, isolation `worktree`. Every span in `spans/*.json`
carries traceId `2382058ba2a59fe37cd6dc347b4588d2` =
`sha256("m3s-001-1784104068186")[0..32)` — the documented hash, so the trace
is findable from the runId alone.

**Node timeline** (journal `node-start`/`node-end` pairs; identical story in
the 9 OTLP files, whose lexical order is the export order): dispatch →
provision → assemble-prompt → build → gates → commit → push → open-pr, every
node routing `next` (span attr `adw.routing`), zero `round` events (repair
never ran), `run-end green`, total duration 137055ms (`run-end.durationMs`;
root span `adw.outcome=green`).

**Gates.** The gates `node-end` carries `{kind:"gates"}` details: one gate
`check` — pass (36ms). Quoted verbatim in the PR body's gate table.

**Agent session.** The build `node-end` carries agent usage (sessionId
`32628fbd-94b9-4439-9811-ef53d81b51da`); the journal `capture` record binds
the SAME id to
`.clens/sessions/32628fbd-94b9-4439-9811-ef53d81b51da.jsonl` (`ok:true`) —
the sessionId-equality join (amended in adw-m3-04). The session file holds
22 events, is ticket-tagged (`m3s-001` throughout its payloads), and is
readable by cLens tooling: `clens list` sees it; `clens distill 32628fbd`
produces a summary (2m10s, claude-sonnet-5, 16 tool calls, 1 file modified).
The agent's summary (journal → PR body) records it created TANKA.md and
could not run state-changing Bash — expected: the factory's deterministic
nodes own gates/commit/push (M2 obs-001 stance), and they did.

**PR.** The ticket's git history records the attempt
(`adw: m3s-001 record attempt` → `in-review`, pr
https://github.com/silouone/adw-m2-scratch/pull/4). The PR body quotes the
journal's stats (rounds 0, duration 137055ms, run id) and the captured
session id + path (S5.4), and carries the never-merges footer (Art. IV).

**Span tree.** 9 spans: root `run` (spanId drawn first) ← 8 node children,
all `parentSpanId` = root, `build` typed `AGENT`, the 7 deterministic nodes
`TOOL`. Root closes last (final OTLP file). TRACEPARENT for the build agent
is derivable from its span: `00-2382058b…-<buildSpanId>-01`.

## Abnormal-exit flush — run m3s-002-1784104446010 (killed)

SIGKILL delivered while the commit node ran (watcher keyed on the gates
progress line; gates finished in ms). From artifacts alone:

- `journal.jsonl`: 13 lines, every line valid JSON, ending at
  `node-start:commit` — no `node-end:commit`, no `run-end`. The story is
  legible: the run died inside commit.
- `spans/`: exactly the five ENDED nodes (dispatch, provision,
  assemble-prompt, build, gates) — every pre-kill span file exists (S2.6,
  SimpleSpanProcessor's synchronous per-span write); no root span, no
  commit span (neither ended) — consistent with the journal cut.
- capture record `ok:true` + session JSONL present for the killed run's
  build agent, distillable like the green run's.
- journal↔span consistency: 5 completed node executions = 5 spans; the
  started-but-unfinished commit has a `node-start` and no span — exactly
  the truthful shape for a mid-node kill.
- The kept workspace survives with the agent's SENRYU.md and the factory
  commit `adw: m3s-002 build (run m3s-002-1784104446010)` (the kill landed
  after the commit node had committed, before it returned) — autopsy-ready
  (Art. VII).

## Gaps & findings (none is an artifact defect; no fix ticket required)

1. **Checklist wording amendment (applied above).** "TRACEPARENT
   extractable from the journaled env" assumed the agent env is journaled;
   it never was (M1 journal shape). The honest artifact claim: the header
   is derivable per agent span from the exported OTLP
   (`00-<traceId>-<spanId>-01`), and the extraction path (W3C propagator →
   child span parents under the agent node's span) is pinned by the
   adw-m3-03 unit proof. Amendment recorded per the amendment rule.
2. **Live in-process receipt is unverifiable under the current sandbox —
   deferred to M6.** m3s-002 asked its agent to write `TRACE.md` with its
   `TRACEPARENT` env var; the captured session shows 23 blocked `printenv`
   attempts: under `permissionMode: acceptEdits` with no allowlist the
   build agent has NO Bash, so it cannot observe its own env (it edits;
   the factory verifies — the deliberate M2 stance). The env merge itself
   is pinned by exact-env tests at the `query()` seam on all three agent
   nodes. M6 probe options: an `allowedTools` read-only extension of
   `AgentQueryOptions`, or an env-echo step in a factory-owned node.
   Recorded as an M6 watch item, not fixed in-epic (a permission-surface
   change is product scope, not an observability artifact defect).
3. **Operational friction note (repeat of M2 obs-001 flavor).** Both live
   agents spent turns asking for Bash approval. Harmless to outcomes
   (green PR regardless) but wasteful; same M6 item as (2) covers it.

## Runbook — reconstructing any run from artifacts

1. `runs/<runId>/journal.jsonl` — read it first; it is the one run history
   and the AUTHORITATIVE record on any conflict (Art. VIII). Line 1
   `run-start` names ticket + isolation; the `node-start`/`node-end` pairs
   are the timeline; `node-end.details` carry gate results
   (`kind:"gates"`) and agent usage incl. sessionId (`kind:"agent"`);
   `round` events are repair rounds; `capture` events bind agent sessions
   to files — the LAST capture record per session is its status (an
   `ok:false` downgrade means the JSONL is incomplete, adw-m3-06); `run-end`
   carries outcome + duration. A journal that stops without `run-end` =
   the run was killed inside the first node lacking a `node-end`.
2. `runs/<runId>/spans/*.json` — OTLP `ExportTraceServiceRequest` files;
   consider ONLY names matching `\d{6}.json` (writes are tmp+rename
   atomic, adw-m3-06 F7 — a crash may orphan one ignorable `.tmp-` file);
   lexical filename order = export order; root span `run` closes last.
   Expect traceId = `sha256(runId)[0..32)` (`traceIdFromRunId`). Cross
   -check: completed node executions in the journal ↔ node spans, 1:1;
   `adw.routing` per span ↔ `node-end.outcome`; `adw.span_type` AGENT on
   build/repair/ci-repair. Reconciliation rule (adw-m3-06 F6): span end
   and journal `node-end` are two writes with a microsecond window — an
   ended span whose node lacks a `node-end` means death exactly in that
   boundary window; trust the journal. Any OTLP-JSON viewer ingests the
   files as-is (this audit read them raw — that satisfies the m3-02
   spot-check with the JSON mapping verified field-by-field in its suite).
3. `runs/<runId>/.clens/sessions/<sessionId>.jsonl` — one per agent
   session; `<sessionId>` equals the journal's usage/capture sessionId,
   and each persisted event carries `adw_ticket_id` (adw-m3-06 F8, full
   capture mode). `clens list` / `clens distill <id>` from the run dir
   summarize it. NOTE (adw-m3-06 F2, by design): a CI repair RESUMES the
   build session, so cLens appends its events to the PRIOR run's session
   file — the file is the SESSION's artifact, shared by every run that
   continued it; both runs' journals reference it by absolute path. The
   dir chain is owner-only (0700).
4. The ticket file's git history in the target repo — status transitions,
   attempt records (runId → branch, workspace, pr), CI rounds charged.
5. The PR body — the operator-facing digest; its gate table, stats and
   session ids must match (1) (a failed capture is flagged "capture
   incomplete", never silently listed). The CI mini-lane, if any ran, has
   its own `runs/<ciRunId>/` with the same shape (trace-per-runId).

## Verify

Written reconstruction matches the actual run; completeness checklist all
green; `bun run lint && bunx tsc --noEmit && bun test` still green after
any fixes.

## Out of scope

Success-metric measurement across many runs (adw-m6-02); dashboards (spec
non-goal).
