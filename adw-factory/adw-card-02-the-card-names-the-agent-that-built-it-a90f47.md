---
id: adw-card-02-the-card-names-the-agent-that-built-it-a90f47
type: feat
status: done
priority: 1
created: 2026-09-28
caps: {minutes: 90, turns: 300, stallMinutes: 20}
depends: []
attempts: [{"runId":"adw-card-02-the-card-names-the-agent-that-built-it-a90f47-1790609830123","branch":"adw/adw-card-02-the-card-names-the-agent-that-built-it-a90f47","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-card-02-the-card-names-the-agent-that-built-it-a90f47-1790609830123/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/silouone/adw-factory/pull/151"}]
---
# A board card never says whether Claude or Codex built it

> **Operator, 2026-09-28.** The board now runs two providers side by side —
> `cqc-*` cards are built by Codex on `gpt-5.6-sol`, `adw-*` and `sabado-*`
> by Claude on `sonnet`. Nothing on the card says which. Provider is the
> single most useful grouping fact on the board right now and it is
> invisible; reading it means opening the run screen.
>
> **The ask:** an at-a-glance provider mark on every card — Claude, Codex,
> or **both**, when a ticket's attempts split across providers.

## The data already exists and already crosses the wire

No projection work. `RunView.provider: Provider` (`"claude" | "codex" |
"unknown"`, `projection.ts:41`) is computed by `deriveProvider`
(projection.ts:258 — the first agent `node-end` whose `usage.model`
resolves) and is already carried by `toWireRunView` (projection.ts:81).

Verified on live journals today: `usage.model` is `gpt-5.6-sol` on every
Codex run and `sonnet` on every Claude run, and `inferProvider` classifies
both correctly (`/^gpt-|codex/i` → codex). This is a **render-only**
ticket.

> Note for the builder, not a requirement: `usage.profile.provider` is still
> journaled as `"claude"` on Codex runs banked before `adw-usage-06` (#…)
> reached `main`. Do **not** read `profile.provider`. `deriveProvider`'s
> model-string path is the one that is right today, and it is already the
> board's path.

## "Both" is a property of the card, not the run

Provider resolution is run-scoped (README: `--provider` > `target.provider`
> `"claude"`), so a single run has exactly one provider. A **card is a
ticket** (`TicketCard`, `board-groups.ts`), holding `attempts: readonly
RunView[]`. So "built by both" means: this ticket's attempts span both
providers — e.g. a Claude attempt blocked, a Codex attempt re-ran it.

`Card.tsx` today reads `card.latest` for everything except the attempt-dot
strip. This derivation is the second thing on the card to read
`card.attempts`.

(`adw-profile-02-a-provider-per-stage` would one day make a *single* run
mixed. Out of scope — do not design for it.)

## Requirements

- [ ] **R1 — red first: the pure derivation, `board-groups.ts`.** A new
      exported pure function beside `healthOf`/`isLive`:
      `providersOf(card: TicketCard): readonly Provider[]` — the distinct
      providers across `card.attempts`, `"unknown"` dropped when at least
      one attempt resolved, deterministic order (`claude` before `codex`).
      An all-unknown card returns `["unknown"]`. Tests land in
      `test/web/ui/board-groups.test.ts`, mirroring the existing
      `healthOf`/`tallyCardHealth` test shape: one-attempt claude,
      one-attempt codex, two attempts split across both, an unknown-only
      card, and a mixed `[codex, unknown]` card (→ `["codex"]`).
      `board-groups.ts` is a relocated-pure-logic module, not one of R5's
      protected files — this belongs here, not in a component body.
- [ ] **R2 — red first: the mark on the card, `Card.tsx` /
      `test/web/ui/Card.test.tsx`.** Render `providersOf(card)` in the
      `.tc-top` row, immediately **left of** `.tc-ago`, as
      `<span class="tc-prov">` holding one `<i class="prov claude">` /
      `<i class="prov codex">` per resolved provider. Both present → both
      glyphs, side by side, claude first. Constraints:
      - one single-width Unicode glyph per provider, consistent with the
        card's existing single-symbol convention (`$`, `◷`, `Σ`, `✓`, `✕`):
        **`✳` for claude, `◉` for codex**;
      - each glyph carries a `title` naming the provider in words
        (`"built by claude"` / `"built by codex"`), and a both-provider
        card's wrapper `title` says so (`"claude and codex"`);
      - `"unknown"` renders **nothing at all** — no placeholder glyph, no
        empty box. A card with no agent node yet is simply unmarked. Pin
        this in a test: an unknown-only card has no `.tc-prov` element.
- [ ] **R3 — the CSS, `src/web/ui/css.ts`.** `.tc-prov` beside the existing
      `.tc-top`/`.tc-ago` rules; `.prov` inherits the small-glyph sizing
      `.tag-st i` already uses. Two distinct colours from the existing
      theme vars (`src/web/theme.ts`) — **no new colour tokens**:
      `.prov.claude{color:var(--model)}` (the amber already meaning "model"
      everywhere on this UI), `.prov.codex{color:var(--tool)}`. Extend
      `test/web/ui/css.test.ts` if it asserts selector presence.
- [ ] **R4 — scope.** No change to `projection.ts`, `card.ts`,
      `board.ts`, `metrics.ts` or the wire payload. No provider *filter*
      chip (that is its own ticket if it is ever wanted). The run screen is
      untouched.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web`: a
`cqc-*` card carries the Codex mark, an `adw-*` card the Claude mark, and
a ticket re-run across both providers carries both — each named on hover.
