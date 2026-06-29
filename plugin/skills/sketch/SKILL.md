---
name: sketch
description: >-
  Iterate a UI/UX surface with a human — at the highest fidelity the target can honestly afford — and
  climb from a quick sketch up to a runnable prototype, all in one loop. It is medium- and stack-agnostic:
  the surface can be a screen (web / iOS / Android / desktop), a terminal UI, a spatial/VR scene, or a
  voice/audio flow, and the skill discovers the medium and stack at runtime (no per-platform code baked
  in). Use whenever the user wants to sketch, mock, wireframe, lay out, or prototype an interface or flow
  before/while building it: "sketch this screen", "mock up the X flow", "wireframe it", "show me the UI",
  "prototype this", "design the onboarding", "what should this look like", "let me see it running",
  "draft the voice flow", "lay out the VR scene". It runs a flow-map → render → self-critique → show →
  "type accept to lock" loop (the loop only ends on the literal word `accept`), climbs a fidelity ladder
  (text sketch → static visual → live preview → real-target run) only as far as the available tooling
  truly reaches, and degrades HONESTLY — it never fakes fidelity it can't render and hands off the exact
  command when a device/headset/SDK is required.
---

# sketch

Iterate any UI/UX surface with a human, at the highest fidelity the target can **honestly** afford, in
**one loop** — and climb from a rough sketch toward a runnable prototype as the tooling allows. There is
no separate "wireframe mode" and "prototype mode": they are the same loop at different **fidelity rungs**.

This skill bakes in **no platform code**. It discovers the medium (is this a screen? a voice flow? a
spatial scene? a terminal UI?), the stack (from the repo's manifests + conventions), and what tooling is
actually reachable — then renders using the repo's own conventions and your own knowledge of the
discovered stack. A new platform is not a new file to write; it's a thing the runtime probe + the model
handle. (See "Why no profiles" at the bottom.)

## The one rule — literal `accept`

A surface is locked **only** when the user replies with the literal word `accept` (single word,
case-insensitive, surrounding whitespace/punctuation ignored). **Everything else is a correction** —
append it and re-render. Soft-positives ("yes", "looks good", "lgtm", "ship it") do **not** lock; on one,
ask exactly once: *"Should I lock this in? Reply `accept` to confirm, or keep correcting."* and do not
lock until they actually type `accept`. There is **no cap on the human loop** — the user decides when each
surface is done. Even in any auto/unattended mode this accept gate is **never** bypassed: UI is
irreducibly a judgment call. The literal word is a small commitment that prevents premature locking and
gives a clean undo — any non-accept word just continues the loop.

## Core principles

1. **Medium first, never assume a screen.** Probe the medium before anything else: `visual-2d` (web /
   mobile / desktop screens), `spatial-3d` (VR/AR), `aural` (voice / IVR / audio), `physical-surface`
   (watch / embedded / TV / kiosk), or `structural-only` (design with no code target). The medium decides
   what "a surface" and "rung 0" even are. ASCII is **one medium's** rung-0, not the universal floor.
2. **One loop, many rungs.** The same render → self-critique → show → accept loop runs at every fidelity
   rung. See `references/fidelity-ladder.md`.
3. **Climb only as far as the tooling truly reaches.** Start at the floor rung (always reachable), climb
   one rung at a time, each a fresh accept-gate, and stop at the highest rung the (medium × discovered
   tooling) actually supports.
4. **Never fake fidelity.** If a rung needs a device / simulator / headset / SDK the probe didn't find,
   do not render a fake of it. Park at the highest reached rung, say exactly which capability was missing
   (quote the probe), and emit a **handoff** (the real files + the exact command a human-with-the-device
   runs). Honesty covers both *run* (built-not-run) and *fidelity* (see the honesty contract below).
5. **Self-review before the human sees it.** A critique step reviews each render and can force a redraw
   only on a **high-severity** issue, capped at **2 internal redraws**, then surfaces to the human
   regardless. The human is never the first to notice an obvious problem, and never gets rate-limited.
6. **Lock the style on first accept, enforce it after.** Pass every accepted surface to the renderer and
   critique so later surfaces stay consistent; style drift across a session is a high-severity issue.
7. **Mining improves the kit.** When a render or build needs a primitive the repo's design kit lacks, add
   it back to that kit (whatever kit the probe found — not a hardcoded one), log it to `decision-log.md`,
   and future runs reuse it. Prototyping continuously upgrades the real design system.
8. **Product language only with the human.** Ask only about what the user sees/feels/does, never about
   stack. Derive the technical side from the repo, silently.

## Phase 0 — Discover (medium + stack + capability)

Before sketching, run a cheap probe (this is the only "platform knowledge", and it's discovered, not
declared):

- **Medium** — from the user's words + the repo: `visual-2d | spatial-3d | aural | physical-surface |
  structural-only`. When ambiguous, default conservative to `structural-only` (design-only), never an
  optimistic web guess.
- **Stack** — from manifests / `CLAUDE.md` / sibling code (like a codebase-context read): language,
  UI framework, the repo's design kit/component conventions, build/run commands.
- **Capability per rung** — what is actually reachable *right now*: can I render a still? serve a live
  preview? run on a simulator/device/headset? Probe concretely (`command -v …`, a booted simulator, a
  browser, a headset bridge) and record the evidence. This sets the ceiling rung.

Full probe + rung definitions: `references/fidelity-ladder.md`. Medium taxonomy + per-medium contract:
`references/mediums.md`.

## Flow

1. **Capture intent** (main thread). One product question: *"What surface or flow do you want to sketch?
   Tell me in plain words what the user is doing."* Get a working name (slug) and a one-paragraph intent.
   Pick an output dir: `--out <dir>` else `sketch/<slug>/`. Create it.
2. **Flow-map** (`agents/flow-mapper-agent.md`, once). It reads intent + any spec, picks the **medium**,
   and returns an ordered queue of **surfaces** (moments where the user encounters something new) — for
   `visual-2d` these are screens, for `aural` they are dialogue turns/states, for `spatial-3d` panels in a
   volume, etc. Medium-neutral, product language.
3. **Per-surface loop** (the heart) at the current rung:
   1. **Render** (`agents/renderer-agent.md`) — produce the surface's `contract_view` for the medium at
      this rung (ASCII for visual-2d; a dialogue-script for aural; a scene-graph for spatial-3d; …), plus
      its `lossy_axes`. Pass `accepted_surfaces` for consistency + all corrections-so-far.
   2. **Critique** (`agents/critique-agent.md`) — medium-parametric self-review; redraw only on ≥1
      high-severity issue, ≤2 passes, then surface.
   3. **Show** the `contract_view` in the main thread. If `lossy_axes` is non-empty, print them above it:
      *"This locks the structure; <axes> can only be judged when run on <device> (see handoff)."*
   4. **Ask, verbatim:** *"Does this match what you have in mind? Reply with corrections, or type
      `accept` when it's right."*
   5. **Read** — literal `accept` locks; anything else is a correction → re-render.
4. **Climb (optional).** After the floor surfaces are accepted, offer the next reachable rung (still →
   live preview → real run), if any. Each higher rung is a **fresh accept-gate** — the human re-accepts
   the higher-fidelity artifact; a blocked higher rung never invalidates a lower accepted one.
5. **Finalize.** Write the durable artifact (below). Tell the user where it is and the highest rung
   reached (and, if parked below the ceiling, the handoff command).

## Honesty contract (the load-bearing part)

- **Run honesty:** a rung that needs to *run* but can't returns `built-not-run` with the reason and the
  exact command — never a fabricated running URL / screenshot / "looks great on device".
- **Fidelity honesty:** for a lossy medium (`lossy_axes` non-empty — e.g. spatial loses depth/occlusion/
  gaze-comfort; aural loses timing/prosody/barge-in-feel), `accept` locks the **structure**
  (panels/anchors/gaze-order, or the dialogue state machine + recovery edges) — **not** the felt
  experience, which stays **UNVERIFIED** until run on the real device. The artifact records
  `contract_medium` + `lossy_axes` so the signed-off file is self-labelling about what was never seen.
- **Never let the floor fake the medium.** A flattened 2D projection of a spatial layout, or a transcript
  of a voice flow, must be labelled "a lossy projection of a richer medium", not presented as the truth.

## Subagent roster

| Agent | File | Role |
|-------|------|------|
| Flow mapper | `agents/flow-mapper-agent.md` | Picks the medium; returns the ordered surface queue (medium-neutral). |
| Renderer | `agents/renderer-agent.md` | Renders one surface's `contract_view` at the current rung, in the discovered medium + stack conventions. |
| Critique | `agents/critique-agent.md` | Medium-parametric self-review before the human; redraw only on ≥1 high-severity, ≤2 passes. |

The critique's high/low rubric is **medium-parametric** — the medium supplies its own high-severity checks
(visual: width/charset/coverage; aural: every turn has a no-input/no-match path + confirmation before a
destructive intent; spatial: every panel has a pose + reach/gaze affordance). See `references/mediums.md`.

## Output (durable artifact)

`<out>/SKETCH.md` — frontmatter `{slug, medium, fidelity_reached, contract_medium, lossy_axes}` + one
section per accepted surface (its `contract_view` as the visual/structural contract + behavior **Notes**
captured during iteration) + an optional flow/journey section + a "Higher-fidelity artifact" back-pointer
(path + the exact run/handoff command) when a higher rung was produced. Plus `decision-log.md` for any
kit additions. This artifact is the source of truth the build/spec read — not a frozen picture.

## Args

- `<intent>` — optional; Phase 1 asks if absent.
- `--out <dir>` — output dir (default `sketch/<slug>/`).
- `--medium <m>` — force the medium (skip detection) when you know it.
- `--rung <R0|R1|R2|R3>` — cap the climb (e.g. `--rung R0` = sketch only, never prototype).

## What this skill never does

- Locks a surface on anything but the literal `accept`.
- Assumes a 2D screen before probing the medium.
- Renders a fidelity rung it could not actually produce (no fake screenshots / fake running app).
- Asks the user a technical question.
- Bakes in per-platform code — the medium/stack/tooling are discovered at runtime.

## Why no profiles (the design choice)

Pre-writing a `profiles/<platform>/` module per platform is hidden hardcoding — it re-introduces the exact
stack-binding this skill exists to avoid, just multiplied. Instead: the **model is the universal adapter**
(it already knows how to write a SwiftUI view, a scene-graph, a dialogue script) and the **runtime probe**
supplies what's specific to *this repo* (its kit, its build command, its reachable tooling). Per-PROJECT
config is discovered from the repo or declared inline; there are no per-PLATFORM files shipped with the
skill. The honesty contract (probe what's reachable, never fake the rest) is what keeps this safe without
a profile author to encode "this platform can't run here".

## References

- `references/fidelity-ladder.md` — the R0–R3 rungs, the capability probe, climbing, and honest handoff.
- `references/mediums.md` — the medium taxonomy, each medium's rung-0 + `contract_view`, and the
  medium-parametric critique checks.
