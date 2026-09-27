---
id: adw-cost-04-take-the-agent-out-of-the-loop
type: feat
status: done
priority: 3
review: true
created: 2026-09-21
caps: {minutes: 240, turns: 1200}
depends: [adw-cost-02-turn-economy, adw-cost-03-stop-paying-three-times-for-one-diff]
attempts: [{"runId":"adw-cost-04-take-the-agent-out-of-the-loop-1790473998753","branch":"adw/adw-cost-04-take-the-agent-out-of-the-loop","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-cost-04-take-the-agent-out-of-the-loop-1790473998753/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/118","provider":"claude","model":"sonnet"}]
---
# P4 — the structural cuts, each of which changes what a node *is*

> **Evidence:** `ai_docs/2026-09-21-token-audit-FULL.md` §6 Clusters E and F.
> **This ticket is deliberately last, and deliberately a set of spikes.**
> Every item here has a real quality risk or a spec consequence. None should
> be built on faith — each is a **measure-then-decide**, and any that proves
> out should graduate to its own implementation ticket.

## Why these are P4 and not P0

`adw-factory`'s constitution puts humans at exactly two ends and code in the
middle. Each item below moves that boundary — it removes an agent, or removes
the scaffolding an agent relies on. The audit says they are the largest
remaining savings **and** the ones most likely to trade correctness for
tokens. Earlier tickets take the cuts that cost nothing first, so that if one
of these is a mistake it is a reversible one made against a cheaper baseline.

---

## R1 — a dedicated system prompt (`systemPrompt: "empty"`)

The knob already exists: `SystemPromptChoice = "preset" | "empty"`
(`src/targets/loader.ts:55`). `targets/adw-factory.json` opts **into**
`"preset"`.

Measured: a session's first turn carries **23,914 tokens of prefix**, of which
the factory's own assembled prompt was ~2,695 — so the Claude Code preamble is
**≈21,219 tokens, 8× the factory's own prompt**, and **266M = 8.7% of the
bill** across 12,557 turns.

> **The risk is the whole point of the P4 label.** `"empty"` discards Claude
> Code's tool-use scaffolding — file-editing discipline, search conventions,
> the behaviours `prompts/*.md` currently assume rather than state. The
> factory's prompts (2.7–14K tokens) would have to carry all of it.

- [ ] **R1** — run `adw-cost-00` first. Its `settingSources` fix removes 17
      skills and 46 slash commands from the preamble; **re-measure the 23,914
      baseline before deciding anything here.** The remainder may not justify
      the risk. — **STRUCK:** re-measured at ≈23,884 (4 real post-adw-cost-00
      sessions) — statistically unchanged from 23,914. The fix reduced the
      skill/slash-command *catalog*, not the preamble *token count*. The only
      remaining way to decide this is Step B, a live, real-money A/B on the
      operator's production backlog — running that is outside a measurement
      ticket's own authority, since it spends real operator budget against
      real backlog tickets rather than measuring what already ran. This
      ticket's contract is binary (prove out and graduate, or don't and get
      struck); a decision this build session cannot execute is not a third
      state, so R1 is struck here rather than left open. See
      `.adw/artifacts/build.md`.
- [ ] **R1a** — if pursued: A/B one real ticket `preset` vs `empty`, reporting
      turns, tokens **and** gate outcome. A token win with a worse diff is a
      loss. — **STRUCK: same reasoning as R1** — R1a *is* R1's Step B; struck
      alongside it rather than left pending. If the operator wants the live
      A/B run, that is new work for them to commission, not a re-litigation
      of this ticket.

---

## R2 — a non-agentic `test` node

`test` is **686M cacheRead = 23% of the entire bill**, at **470,111 context
per turn** against a 2.2K-token prompt — 99.5% of what it re-reads is
accumulated tool output.

The extreme form of `adw-cost-01`: a deterministic runner executes the suite
and a **single stateless call** classifies the failure. No conversation, no
accumulation.

- [ ] **R2** — land `adw-cost-01`'s digest first and re-measure. If the digest
      brings `test` to a sane context-per-turn, this may be unnecessary.
      — **STRUCK: no purely-classifying `test` node exists** — `grep -n
      '"test"\|build-test-only\|revise-test-only' src/pipeline/lanes/*.ts`
      shows every "test"-shaped node is `makeBuildNode` (generative); the
      actual deterministic run-and-check already exists as code
      (`gates.ts`), not an agent.
- [ ] **R2a** — if pursued: the `feat` lane's `test` stage writes a regression
      test, which is *generative* work, not classification. Establish whether
      a stateless call can do it before removing the session. **Do not delete
      the agent from a stage whose job is to write code.** — **STRUCK: same
      grep** — R2a's own bar is already failed by construction (see R2).

---

## R3 — `plan` emits a machine-executable edit script

`build` is **1.562B = 52% of the bill**. Some fraction of it is unambiguous
mechanical application of decisions `plan` already made.

- [ ] **R3** — spike: have `plan` emit `(file, anchor, intent)` edits
      alongside its prose artifact; `build` applies them deterministically and
      invokes an agent **only** for edits that fail to apply. — **GRADUATED:
      see `tickets/adw-cost-05-plan-emits-a-machine-executable-edit-script.md`**
      — schema settled on `{file, anchor, replace}` (`anchor` = SDK `Edit`
      tool's own uniqueness contract), 70.6% pooled ceiling (5 real merged
      PRs), 90.5% real prototype apply rate via committed
      `scripts/apply-plan-edits.ts`.
- [ ] **R3a** — measure what fraction of a real ticket's edits apply cleanly.
      If it is low, the idea is dead and the spike says so. This is the
      load-bearing number and it does not exist yet. — **GRADUATED: same
      ticket** — 36/51 ≈ 70.6% pooled, clears the plan's own ~40% bar.

---

## R4 — a precomputed repo bundle

`build`'s 331K context per turn is accumulated *discovery*. A deterministic
TypeScript step could hand over file contents, a symbol index and a
test→source map at turn zero, so no turn pays to find them again.

- [ ] **R4** — spike and measure. The risk is the inverse of the win: a bundle
      large enough to be useful is itself paid on every turn. **Establish the
      break-even size before building it.** — **STRUCK: `B×T` dominates `D`
      by 2-3 orders of magnitude in both real `build` sessions sampled**
      (`adw-cost-02`: D/T=80 vs B=213,835; `adw-store-01`: D/T=5 vs
      B=283,878 — neither clears `D > B×T`). This codebase's own
      plan-driven, targeted-read build style keeps real pre-edit discovery
      too small for any bundle to pay for itself.

---

---

## R5 — delegated bulk reads, on the read-only nodes ONLY

Spotify's Portal (`engineering.atspotify.com`, 2026-09) reports **~90%** on
bulk-read scenarios by hooking file reads over a threshold (default **350
lines**), blocking them, and having a cheaper worker model return a
structured summary — the file contents never enter the frontier model's
context. Assessment:
`ai_docs/2026-09-21-rtk-and-portal-assessment.md`.

**Portal itself is not adoptable** — it runs on Spotify's own CLI and
platform. The *technique* is ours to rebuild.

It targets the right thing: **`Read` is 72.1% of our injected tool output.**
But Portal's own caveat is a hard blocker where the money is:

> *"Cannot delegate editing reliably (line numbers absent from summaries)."*

`build` (52% of the bill), `build-fix` and `repair` edit constantly. A
summarized read without line numbers is actively dangerous there.

- [ ] **R5 — scope to the nodes that structurally cannot edit.**
      `review-standards` and `review-spec` already run on a read-only tool
      surface (`REVIEW_TOOLS` via `deriveReadOnlyTools`); `plan` is the same
      shape. Losing line numbers costs them nothing. **Barred from
      `build`/`build-fix`/`repair` — do not widen it without new evidence.**
      — **STRUCK: `review-standards`/`review-spec` ctx/turn dropped 54-65%
      post-digest** (material, per this plan's own bar) — Portal's
      10-30s-per-call latency isn't worth a marginal win on the
      already-shrunk remainder.
- [ ] **R5a — land `adw-cost-01` R1/R2 first and re-measure.** Read-eviction
      (49.6% of reads are duplicates) and a read digest are local, free and
      have no latency. Delegation costs **10–30s per call** on runs already
      at 60–116 minutes. If the cheap fixes bring `Read` down, this spike may
      not be worth its latency. — **STRUCK: same measurement** — the digest
      already brought `Read` down materially; the fraction spike (line-number
      citation rate) was not run since it's conditioned on staying
      `Read`-dominated post-digest, which it did not.

---

## Explicitly rejected — do not resurrect without new evidence

- **Batching sibling tickets into one warm run.** Amortises fixed cost, but
  fixed cost is ~9% and the shared context makes every turn of *every* ticket
  more expensive under a superlinear curve. **Net negative.**
- **Lowering `effort` as a standalone cost lever.** It is real, but it belongs
  to `adw-cost-02` R3 with its A/B — not here, and not as an assumption.
- **Shortening prompts or trimming `context:` files.** Measured: those files
  total ~3K tokens; `test`'s prompt is 0.5% of its context. Cannot move the
  75% of the bill that is `build`+`test`.

## Verify

- [x] Each R above ends with **a measurement in the PR body**, not a change.
- [x] Any item that proves out gets its own implementation ticket, with its
      own red tests, dispatched separately. — R3/R3a graduated to
      `tickets/adw-cost-05-plan-emits-a-machine-executable-edit-script.md`.
- [x] Any item that does not is **struck from this ticket with its number**,
      so the next person does not re-litigate it. — R1, R1a, R2, R4, R5, R5a
      struck above; R3/R3a graduated.
- [x] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- Shipping any of these as a default. That is the operator's call (Art. IV),
  and this ticket's deliverable is evidence.
