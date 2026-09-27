---
id: adw-bug-10-the-advisor-server-tool-hangs-every-agent-node
type: bug
status: done
priority: 1
created: 2026-09-17
depends: []
attempts: [{"runId":"adw-bug-10-the-advisor-server-tool-hangs-every-agent-node-1789607702241","branch":"adw/adw-bug-10-the-advisor-server-tool-hangs-every-agent-node","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-10-the-advisor-server-tool-hangs-every-agent-node-1789607702241/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/64","provider":"claude","model":"sonnet"}]
---
# Every stall the factory has ever measured is one `advisor` server-tool call that never returns

> Measured 2026-09-17 across **every retained run workspace** (12 runs,
> `~/.claude/projects/*adw-factory-runs-*`): **10 stalls, 10 immediately
> preceded by a `server_tool_use` block named `advisor`. Zero stalls with any
> other preceding block.**

## 1. What happened

`adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live`
is blocked on three separate runs (`…-1789515578821`, `…-1789595055605`,
`…-1789600226674`), 9 `plan` attempts, 9 identical deaths:

```
▸ plan → ↻ repair round 1/2 → ▸ plan → ↻ repair round 2/2 → ▸ plan → ■ blocked
reason: API Error: Response stalled mid-stream. The response above may be incomplete.
```

The last run burned **29m48s** — 7m49s of it a cache-miss baseline, the rest
three plan attempts that each did real work and then died.

## 2. Root cause

Each attempt's transcript ends on the same two records:

```jsonc
{"ts":"2026-09-16T23:35:14.779Z","type":"assistant",
 "content":[{"type":"server_tool_use","id":"srvtoolu_01CKBv…","name":"advisor","input":{}}]}
{"ts":"2026-09-16T23:40:14.787Z","type":"assistant",
 "content":[{"type":"text","text":"API Error: Response stalled mid-stream…"}]}
```

`23:35:14.779 → 23:40:14.787` = **300.008s**. The other two attempts:
300.011s and 300.019s. The agent calls the server-side `advisor` tool, no
`advisor_tool_result` ever arrives, and the CLI's 300s idle-stream timeout
fires. Across all three fe-19 runs: **9 advisor calls, 0 results.** In other
tickets' runs advisor *does* return (`adw-perf-03`: 4 calls / 4 results), so
this is a hang that fires unpredictably — not an always-broken tool.

### Why the tool is there at all

`~/.claude/settings.json` carries `"advisorModel": "opus"`. The bundled CLI
gates the tool on exactly that:

```js
function Nte(){ if(truthy(process.env.CLAUDE_CODE_DISABLE_ADVISOR_TOOL)) return !1; … }
…
if(_) z.push({type:"advisor_20260301", name:"advisor", model:_});   // _ = resolved advisorModel
```

It is pushed into `extraToolSchemas` — a **server-side** tool schema sent
with the request, structurally separate from the client tool list.

### Why `AGENT_TOOLS` does not stop it

`adw-tools-01` declared a 6-tool whitelist via SDK `Options.tools`, and it
works — a run's own init message reports exactly
`["Bash","Edit","Glob","Grep","Read","Write"]`. **`advisor` is not in that
list and was called anyway**, because `Options.tools` governs client tools
only. The factory's "declared tool surface" is not closed.

### Why nobody noticed

`live-query.ts:497` claims, of `effort`:

> *"the pinned reasoning effort always rides through, so a run never inherits
> the operator's `~/.claude` global config."*

True of `effort` — and only of `effort`, because `effort` is passed
explicitly. `settingSources` is never set, so the SDK loads all filesystem
settings (SDK default) and every *other* key in the operator's global
settings leaks into every factory agent. `advisorModel` is one of them.

### `adw-bug-05` diagnosed the wrong cause

`adw-bug-05` recorded "ten stalls, all 302-339s" and concluded they came from
*"a single uninterrupted generation long enough to compose a whole
document"*. `adw-bug-07` then moved artifacts to files and
`prompts/bug-plan.md` grew a "build it up in sections as you go" instruction
citing that measurement. The measurement was real; the attribution was not —
the same ten stalls are, block-for-block, ten advisor calls. The file-artifact
change is still worth keeping (it made the fe-19 plan **durable and
salvageable**, which is the only reason this run is recoverable), but it could
never have fixed the stall, and the prompt instruction it added is
misinformation the agent now spends tokens obeying.

## 3. Cost

- 3 runs × 3 attempts × 5min of pure dead-wait = **45 minutes of nothing**,
  plus every attempt's discarded work on top.
- `TRANSIENT_RETRY_MAX_ROUNDS = 2` is sized for a network blip. Against a
  deterministic hang it is 2 guaranteed 5-minute losses before `blocked`.
- `transient-retry` arms **no backoff** (journal: `node-start` → `node-end`
  in 1ms), so the two retries hit the same condition instantly — the exact
  shape `adw-bug-09` fixed for `push`/`open-pr`, unfixed here.

## 4. The fix

**Primary — close the setting-inheritance hole.** Pass
`settingSources: ['project']` in `live-query.ts`'s `sdkQuery` options, with
the same never-omitted contract as `systemPrompt`/`tools`/`effort`. `'project'`
is required (not `[]`): `[]` would also stop `CLAUDE.md` from loading, which
would strip the TDD/purity ground rules from every factory agent. Verified
safe: the cLens hooks ride programmatically via `makeClensHooks` (`cli.ts:720`)
→ `options.hooks`, not from the settings file, so captures survive.

Side benefit: the clens captures currently show **every** `PreToolUse`/
`PostToolUse` twice, because the programmatic hook and the operator's global
settings hook both fire. `settingSources: ['project']` removes the duplicate.

**Belt and braces — name the disable explicitly.** Add
`CLAUDE_CODE_DISABLE_ADVISOR_TOOL=1` to the agent child env. It is not matched
by `SENSITIVE_ENV_KEY`, so it already rides the local-spawn path; container/
e2b need it in `optionsEnv`. This is what makes the guarantee survive a future
operator config the SDK reads from somewhere else.

**Correct the record.** Strike the `adw-bug-05` attribution from
`build.ts:1055-1062` (the `ARTIFACT_DIR` doc comment) and the "a single
uninterrupted generation … trips a ~300s idle-stream timeout" bullet in
`prompts/bug-plan.md` / `prompts/feature-plan.md`. Keep the write-as-you-go
advice if wanted, but on its real merit (durability), not a false cause.

**Out of scope, worth a ticket:** `transient-retry` has no backoff, and
`prompts/bug-plan.md` contradicts itself — line 14 says "Your final message
**is** the plan artifact: it is captured verbatim", the "Your written output"
section says the opposite. `adw-bug-07` changed the mechanism and left the
older sentence standing.

## 5. Verify

1. **Red first (Art. I):** a `live-query` unit test asserting `settingSources`
   arrives as `['project']` on the options the SDK receives — mirroring the
   existing `systemPrompt`/`tools` never-omitted tests. Red against today's
   code, which never sets the key.
2. A unit test that the agent child env carries
   `CLAUDE_CODE_DISABLE_ADVISOR_TOOL=1` on all three spawn paths.
3. Live: re-run fe-19. The `plan` node completes with an
   `advisor_tool_result`-free transcript and no `server_tool_use` block.
4. `bun run lint && bunx tsc --noEmit && bun test` green.
