---
id: hq-31-ask-hq-the-mirror-voice-68c0fd
type: feat
status: queued
priority: 3
depends: [hq-30-ask-hq-the-cone-and-the-modal-e3b916]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: []
---
# ASK HQ: the Mirror voice spectrum that speaks and listens

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 69, 70 and the voice half of 67 (spec-v2 → ASK HQ → the spectrum):
- **Pure, seeded spectrum maths** as functions of `(seed, t, level)`:
  - 64 log bands, 80 Hz–4 kHz;
  - the voice model: f0 ≈ 132 Hz with prosody, syllables at 3.6 Hz, 5 formants, 16 harmonics;
  - attack/release `.55/.18`, or `.22/.07` while speaking; peak hold 18 frames, then a fall of .012 per frame.
- **Mirror:**
  - 200 bars, 3 px with round caps;
  - each bar reads an fbm-chosen band, has its own gain (0.45–1.4) and rides a two-scale noise envelope;
  - spikes: ~2.5 % of bars at 8 Hz when speaking, ~4 % at 14 Hz when listening;
  - reach: out `6+78·amp` and in `3+26·amp` when speaking; in `6+66·amp` when listening.
- **Speaking** radiates out in ember. **Listening** draws in in gold, with three contracting rings and brightness following the mic level. The dust cap pulses while speaking and turns gold while listening.
- **🎙 mic:** `getUserMedia` level and FFT (8192) only, driving the listening visual locally. **No speech-to-text** (it is the backend spec's call).
- **🔊** speaks the reply through local `speechSynthesis`; the speaking visual runs from the voice model.
- Variants A · LED and C · Ribbons and the demo pill are **not** built.

## Red first

Pure:
- determinism for a seed;
- every value bounded;
- **irregularity:** no two adjacent bars share a band, and gains vary;
- the attack/release and peak-hold curves on a step input;
- the spike rate within tolerance over 10 s of simulated frames.

## Acceptance criteria

- [ ] The mic permission being denied leaves the shell working, with an honest "mic unavailable".
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] With a real mic in a visible tab, it reads like a voice, not an equaliser.

## Blocked by

- hq-30 (the cone and the modal).
