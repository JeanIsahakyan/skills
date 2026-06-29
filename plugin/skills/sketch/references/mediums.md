# Mediums

The probe picks the **medium** before anything else, because the medium decides what "a surface" is, what
rung-0 looks like, and which critique checks are high-severity. **ASCII is one medium's rung-0, not the
universal floor.** Treating every surface as a 2D screen is the bug this taxonomy exists to prevent.

Every surface the renderer produces carries a **`contract_view`**:

```json
{ "medium": "<medium>", "body": <medium-specific structure>, "lossy_axes": ["<axis>", ...] }
```

`lossy_axes` lists the dimensions this rung's view **cannot** represent (empty for faithful media). When
non-empty, `accept` locks the **structure**, not the felt experience (see the SKILL honesty contract).

The durable artifact stores the `contract_view.body` per surface — never just a flattened picture.

---

## visual-2d — screens (web / mobile / desktop)

- **A surface =** a screen.
- **Rung-0 body =** an ASCII layout (box-drawing; sizing: mobile 36–44 cols, web/desktop 60–80, unknown
  48). Element notation: `[ Primary ]`, `( Secondary )`, `(◉)/( )` radio, `[x]/[ ]` checkbox, `[ Email… ]`
  input, `<photo>` image, tabs/bottom-nav as ASCII glyphs, `…` for clipped edges. One character set per
  document; match the first accepted surface's charset exactly.
- **lossy_axes =** none (faithful at R0 for layout intent).
- **High-severity critique checks:** a brief element missing; charset/style contradicts an accepted
  surface; a prior correction silently undone; contradicts a documented spec invariant (caps/counts);
  width way out of bounds (mobile >60 / desktop >100); unlabeled interactive elements; mixed charsets.

## terminal UI — text-grid

- **A surface =** a screen, but the text grid **is** the production medium, not a sketch of it.
- **Rung-0 body =** ASCII at the real terminal width; R0 is already near-production.
- **lossy_axes =** none.
- **High-severity checks:** grid-width / TTY-cell overflow; same coverage/consistency rules as visual-2d.
  (R2≡R3: the accepted grid is literally the running UI.)

## spatial-3d — VR / AR / spatial

- **A surface =** a *volume*, often one volume of co-present panels (not a sequence of screens). The
  lockable invariant is the **between-panel relationships**, not per-panel pixels.
- **Rung-0 body =** a scene-graph: panels/nodes with `pose` (position + orientation), anchors,
  gaze-order, ray/controller targets, comfort-zone metadata. Self-labelled **"spatial truth, NOT a
  flattened 2D projection"**. Do **not** auto-flatten to a 2D grid — that silently lies about depth.
- **lossy_axes =** `["depth", "occlusion", "gaze-comfort", "head-motion"]`.
- **High-severity checks:** a panel with no `pose`; a panel/ray-target outside arm's-reach or the comfort
  FOV from the declared pose; panels that occlude each other from that pose; a head-locked panel that
  would induce motion discomfort; collision in the walk envelope (single-volume composition).

## aural — voice / IVR / audio

- **A surface =** a *dialogue turn / state* (the medium is a continuous, interruptible stream — set
  `continuity: streaming` and lock the **dialogue state machine**, not a fixed screen).
- **Rung-0 body =** a dialogue-script: an ordered turn list, each turn with **first-class slots** (not
  prose): `{ system_prompt, expected_intents, on_no_input_reprompt, on_no_match_reprompt, barge_in,
  confirmation_needed, timeout_ms }`, plus the recovery edges between states.
- **lossy_axes =** `["timing", "prosody", "barge-in-feel", "audio-cue-quality"]`.
- **High-severity checks:** a turn with no `on_no_input` and/or `on_no_match` path; a destructive intent
  with no `confirmation_needed`; an unreachable state or a state with no exit; an intent set with no
  fallback. (A "picture of audio" is meaningless — never render visual chrome for a voice flow.)

## physical-surface — watch / embedded / TV / kiosk

- **A surface =** a screen at a constrained budget with an odd input model.
- **Rung-0 body =** ASCII at the **real** surface geometry (a 16×2 LCD is a 16×2 box; a watch is
  tall-narrow ~30 cols; a TV is a sparse 10-foot grid), with the input model (rotary crown, single
  button, d-pad, IR remote, touch-grid) annotated as affordances.
- **lossy_axes =** usually none for layout, but note when the real pixel/segment budget can't be shown.
- **High-severity checks:** content exceeds the real surface bounds; an interaction the input model can't
  perform (e.g. multi-touch on a single-button device); illegible density for the viewing distance.

## structural-only — design, no code target

- **A surface =** a screen/flow; the deliverable **is** the surfaces themselves.
- **Rung-0 body =** ASCII (as visual-2d). Tops out at R1 (structure + an optional still). No run.
- **lossy_axes =** none. This is the default when the medium/target is ambiguous — degrade here, never to
  an optimistic web guess.

---

## How the critique stays medium-parametric

The core critique rubric is fixed (verdict `good_to_show | redraw`; redraw only on **≥1 high-severity**
issue; **≤2** internal redraws then surface). The **high-severity checklist is supplied by the medium**
(above) — the renderer/critique read the `contract_view` for that medium, not a hardcoded width-and-tap
list. So the self-reviewer audits the dimension that actually defines the surface (turn-recovery for
voice, gaze-comfort for spatial, width for screens) instead of grading a flattening it can't see. Core
still owns the gate, the cap, and the verdict schema; the medium owns *what counts as a high issue*.
