---
id: adw-gate-scope-principle
type: chore
status: done
priority: 2
created: 2026-09-11
depends: []
attempts: [{"runId":"adw-gate-scope-principle-1789129196574","branch":"adw/adw-gate-scope-principle","workspace":"/Users/silouane/adw-factory/runs/adw-gate-scope-principle-1789129196574/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/3","provider":"claude","model":"sonnet"}]
---
# Gates should only run tests whose failure implicates the change

## Context

Distilled from 2026-09-11, when four factory runs blocked on gate failures that
had nothing to do with the tickets being built.

A target's `gates` are the factory's definition of done. Today they run whatever
the target's command runs — so a test asserting a THIRD-PARTY CLI's liveness or
stdout format can block a run, and the operator is told the ticket failed. The
`codex doctor` test was exactly this and has been removed (`bfa753f`).

This matters far beyond this repo. Every real target has tests that need a live
service, a credential, a network, or a fast machine. If the factory cannot
distinguish **"your change broke this"** from **"the environment is off"**, it
will block constantly on other people's repos — which is the stated goal for
v1 (see `ai_docs/2026-09-10-deep-state-and-roadmap.md`).

## The gap

There is no way for a target to say "these tests are expected red/slow here."
`base-green-check` (`red-check.ts:151`) hard-fails on ANY non-zero gate on the
untouched checkout, with no allowance. That is correct as a default — a bug fix
must not be built on an unprovable base — but it makes onboarding any imperfect
repo a hard stop, with a message that blames the ticket.

## To weigh at pickup — do NOT assume a design

- (a) A `knownRed` / `allowFailures` list in the target config, matched against
      gate output. Simple; risks becoming a place to hide real breakage.
- (b) Split gates into `blocking` and `advisory`, where advisory failures are
      journaled and surfaced in the PR body but do not block.
- (c) Require a base-green SNAPSHOT taken at onboarding: the base's failures are
      recorded once, and only NEW failures block. Most honest, most work.
- (d) Do nothing in the factory; document that a target must have a green suite.
      Defensible — but it is the constraint that makes onboarding hardest, and
      it should be a decision rather than a default.

(c) is the one that generalizes; (a) is the one that ships this week. The
operator picks, and the amendment rule applies if the `TargetConfig` contract
has to change (`plan §5`).

## Out of scope

`adw-gates-config-error-regex` — a distinct defect (a red suite misclassified
as a config error). Fix that first; it is smaller and strictly blocking.

## Run log

**2026-09-11 — run `adw-gate-scope-principle-1789129196574` blocked before any
code, zero agent tokens on implementation.** This ticket has no `Requirements`
or `Verify` section — only the "To weigh at pickup" menu of four designs (a)–(d)
with the explicit instruction "do NOT assume a design" and "the operator picks."
Contrast with `adw-gates-config-error-regex`, whose own "weigh at pickup" note
still shipped a concrete `Requirements`/`Verify` block that any of its three
options would satisfy — a builder could pick an implementation detail without
picking the *contract*. Here the four options are not implementation details of
one contract; they change `TargetConfig` in mutually exclusive, externally
visible ways (a `knownRed`/`allowFailures` field vs. a `blocking`/`advisory`
gate split vs. a persisted base-green snapshot vs. no factory change at all).
Confirmed `specs/adw-v1-plan.md` §5 (the `TargetConfig` contract) records none
of `knownRed`, `allowFailures`, `advisory`, `blocking`, or a snapshot mechanism
today — no design has been locked elsewhere that this run could implement
against. Per the ground rules' amendment rule ("stop and propose a spec
amendment... never silently diverge") and the backlog's refinement gate (a
ticket lacking approved, concrete requirements should not go `in-progress`),
picking one of (a)–(d) unilaterally would be exactly the silent divergence the
process forbids. Recommend: operator picks a-d (the ticket already flags (a) as
the one that ships this week, (c) as the one that generalizes), then the
ticket is refined with a `Requirements`/`Verify` section naming the chosen
`TargetConfig` shape before the next pickup.
