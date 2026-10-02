---
id: hq-30-ask-hq-the-cone-and-the-modal-e3b916
type: feat
status: queued
priority: 2
depends: [hq-19-the-rings-make-room-3f1a92]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: []
---
# ASK HQ: the speaker cone at the centre and its modal shell

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 63–68, 71, 72 (spec-v2 → ASK HQ):
- **The idle cone** at the rings' centre, 64·s px: ember conic surround, a dark cone with ridges every 5 px and directional shading, a domed dust cap (inset 34 %), the label `ASK HQ`; on hover, scale 1.08 and an ember shadow.
- **Open is a modal.**
  - The camera never moves; the cone grows to 360·s px and becomes the reply frame.
  - **An invisible shield** takes every other pointer.
  - ✕ LEAVE (above the cone) or Esc leaves.
  - The halo is a local radial dim (920·s px, .78 → 0 at 460·s px); the header tucks.
- **The composer:** 📎 attach and drop onto the centre (chips with size and ×), text, 🎙 and 🔊 buttons (wired in hq-31), send.
- **The reply** is one formatted message (h4, lists, code, bold) in a round 280·s px area inside the cone.
- **The agent seam,** client-side only (no server route):

  ```ts
  type AgentState = "idle" | "thinking" | "speaking" | "listening";
  interface AgentAdapter {
    readonly id: string;              // "none" in v2
    readonly connected: boolean;
    ask(q: { text: string; attachments: readonly File[] }, signal: AbortSignal):
      AsyncIterable<{ state: AgentState } | { markdown: string }>;
  }
  ```

  Only the `none` adapter ships: brief `thinking`, then "No agent is connected to ASK HQ yet." Nothing runtime-specific lives in the shell.
- **Thinking visual:** organic noise clusters drifting and swelling at their own speeds, rare sparks, 3 px hairlines (pure, seeded noise).

## Red first

- Pure: the `none` adapter's yield sequence; seeded noise is deterministic and bounded; the halo size under the UI scale.
- UI (happy-dom): open → the shield is present and blocks; Esc closes; the `none` reply is the honest message (no fake answer); attachment chips add and remove.
- HTTP: no new route (the guard test still sees exactly one write route).

## Acceptance criteria

- [ ] Opening ASK HQ never zooms or pans the rings.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-19 (the centre is freed).
