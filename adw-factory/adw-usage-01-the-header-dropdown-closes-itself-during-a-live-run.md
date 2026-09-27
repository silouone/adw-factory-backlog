---
id: adw-usage-01-the-header-dropdown-closes-itself-during-a-live-run
type: bug
status: done
priority: 1
created: 2026-09-17
depends: []
attempts: [{"runId":"adw-usage-01-the-header-dropdown-closes-itself-during-a-live-run-1789600846897","branch":"adw/adw-usage-01-the-header-dropdown-closes-itself-during-a-live-run","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-01-the-header-dropdown-closes-itself-during-a-live-run-1789600846897/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/61","provider":"claude","model":"sonnet"}]
---
# The board's project dropdown closes itself on every SSE tick

> Found 2026-09-17 while prototyping `adw-v1.7-subscription-headroom`. Not
> caused by that work — this is a defect in shipped code, and it is ticketed
> separately so it can dispatch in parallel with `adw-usage-02`.

## 1. The defect

Open `adw web` with a run in flight, click **All projects ▾**, and the menu
slams shut before you can pick anything. Every second. It is only usable when
the factory is idle — which is the opposite of when you need to filter.

## 2. Why, precisely

`adw-fe-14` §7 moved the fragment boundary so the header became part of the
live surface: `renderBoardHeader` is called by `renderGridBody`, and the
`/events` route pushes that whole fragment into `#grid-body.innerHTML`.

`<details class="menu" id="fx-menu">` lives **inside** that fragment. So every
push destroys and recreates it, and `open` is a property of the destroyed
element.

The SSE handler is aware of the problem and only half-solves it:

```js
es.onmessage = function(e){
  document.getElementById('grid-body').innerHTML = e.data;
  apply();                     // restores FILTER state…
};
```

`apply()` re-applies `proj` and `health` to the new DOM — which is why
filtering survives a swap — but **nothing restores `menu.open`**. The closure
that holds `proj`/`health` outside the fragment is exactly the right idiom;
the menu's open state simply was never added to it.

Note the asymmetry that hid this: the *filter* survives, so the screen looks
like it is handling swaps correctly. Only the *menu* is lost, and only while
something is running.

## 3. The fix

Extend the existing outside-the-fragment closure to carry the menu's open
state, and re-apply it in the same place `apply()` already re-applies the
filter. No new element, no new listener, no change to the fragment boundary.

Two properties the fix must hold, because both are already true today and
must stay true:

- `apply()` is called on first paint **and** after every swap — the restore
  must ride that same path, not a second one.
- Clicking an option, clicking outside, and `Escape` all still close the menu
  and that closure must record it, or the menu will re-open itself on the next
  tick. **A menu that reopens is a worse bug than one that closes.**

## 4. Red test first (Art. I)

`test/web/render.test.ts` is the home; `adw-fe-16` established the idiom of
asserting against the rendered fragment and its inline script as strings.

### The red set is ONE behaviour

> **The menu's open state survives an `innerHTML` swap of `#grid-body`.**

That is the only assertion that is genuinely red today, and it must fail for
that reason — not because the string `fx-menu` is missing somewhere.

**Deliberately NOT in the red set:** "selecting an option closes the menu",
"`Escape` closes the menu", "an outside click closes the menu". All three are
**already true in shipped code** — `filterScript` has every one of those paths
today. A test asserting them is green before the fix exists, so it cannot be
red-first, and forcing it red would mean asserting a closure variable the
agent has not written yet — an implementation detail, which this repo's
testing stance forbids.

They still matter: they are what stops the fix becoming a menu that **reopens
itself**, which is a worse bug than one that closes. So they live in §6 as
regression checks that must **stay** green. Do not try to make them fail.

## 5. Out of scope

- Any change to the fragment boundary itself. `adw-fe-14` §7 put it where it
  is deliberately; this ticket works within it.
- The run screen (`#run-console`) has the identical swap shape but no
  `<details>` in its header today. Do not pre-emptively change it.
- `adw-fe-19`'s overlapping-block and blank-drawer defects — separate ticket,
  in flight.

## 6. Verify

- `bun run lint && bunx tsc --noEmit && bun test` all green
- the new test red before the fix, green after

**Regression checks — green BEFORE and AFTER (see §4):**

- selecting an option still closes the menu
- `Escape` still closes the menu
- an outside click still closes the menu

If any of these three flips to red, the fix has produced a menu that reopens
itself on the next tick. That is the failure mode to watch for.

**Manually:** `just web` with a run in flight — open the dropdown, watch it
survive at least 10 consecutive pushes, then confirm all three close paths
still work.
