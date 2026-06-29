# Fidelity ladder

The sketch↔prototype spectrum is **one loop at four fidelity rungs**. The loop body
(render → self-critique → show → correct → literal-`accept`) is identical at every rung; only the produced
artifact and the tooling it needs change. Start at the floor, climb one rung at a time, each a fresh
accept-gate, stop at the highest rung the **(medium × reachable tooling)** pair actually supports.

## The rungs

| Rung | What it is | Needs |
|------|------------|-------|
| **R0** | The medium's structural/sketch view — the always-reachable floor (ASCII for a screen, a dialogue-script for voice, a scene-graph for a spatial scene). | Nothing but the renderer. |
| **R1** | A static visual still — real pixels, no behavior (a rendered image / SVG / screenshot, or an offline scene still). | A still-render path (headless browser, `ImageRenderer`, an SVG fallback that needs no toolchain). |
| **R2** | A live interactive preview — clickable, mocked data, rendered *in an environment* (not the final device). | A cheap host/preview the probe found (dev server, simulator, terminal PTY, desktop canvas). |
| **R3** | A real-target run — runs in the actual runtime / simulator / device / headset. | The real toolchain + a device/sim/headset. |

`--rung R0` caps the climb at a sketch; omitting it lets the loop climb to the reachable ceiling.

## Capability probe (run once, in Phase 0)

Determine, per rung, one verdict ∈ `reachable | degraded | blocked`, **with evidence** (quote the probe;
never guess). Example shape (`<out>/_capability.json`):

```json
{
  "medium": "visual-2d",
  "stack": "expo/react-native",
  "rungs": {
    "R0": { "verdict": "reachable" },
    "R1": { "verdict": "reachable", "how": "headless render", "evidence": "command -v <renderer> → ok" },
    "R2": { "verdict": "blocked",   "needs": "iOS simulator", "evidence": "xcrun simctl list → no booted runtime" },
    "R3": { "verdict": "blocked",   "needs": "Xcode + device", "evidence": "no xcodebuild on PATH" }
  },
  "ceiling": "R1",
  "handoff_required": true
}
```

Probe rules:
- Probe **concretely**: `command -v <tool>`, a booted simulator (`xcrun simctl list`/`adb devices`), a
  browser, a headset bridge, a TTS CLI for voice R1, etc.
- **Absence → blocked, not faked.** A blocked rung is recorded with what's missing; the loop parks below.
- **Conservative on unknown.** If the medium or stack can't be resolved, floor at `structural-only` / R0.
- The **R0 floor is universal** — every medium has a structural view reachable with zero tooling. R1 has a
  secondary universal floor too: an SVG-from-structure still needs no platform tooling, so a static visual
  is usually reachable even when the native still-renderer is absent.

## Reachability by medium (typical)

- **visual-2d (web):** full ladder; R2≈R3 (a browser is the real runtime). The richest case.
- **visual-2d (iOS/Android native):** R0+R1 reachable; **R2 means the simulator** (not a web preview —
  that would be faking native fidelity), reachable only if a sim/emulator is booted; R3 needs the real
  toolchain+device. Headless box → floor R1 + handoff.
- **visual-2d (desktop):** R0+R1; R2/R3 collapse (`run` ≈ `preview`), gated on the matching OS toolchain;
  foreign-OS target (WinUI on Linux) → R1 + cross-build handoff.
- **terminal UI:** R0 ASCII **is** near-production; R2≡R3 = run the TUI in a real PTY. Tops out at R3
  cheaply — the inverse of VR. Almost always fully reachable.
- **spatial-3d (VR/AR):** R0 = scene-graph text (labelled lossy); R1 = offline 3D still if a scene tool
  exists else the SVG/structure floor; R2 needs a headset/vendor sim; R3 needs a headset. Usually parks at
  R0/R1 + handoff. Never an "imagine the headset view".
- **aural (voice/IVR):** R0 = dialogue-script; R1 = TTS-rendered audio **if** a TTS CLI is present, else
  "script-only, no audio toolchain"; R2 = run the dialog model against typed/spoken utterances; R3 =
  deploy to the assistant/IVR runtime (almost always handoff). No browser-clickthrough substitute.
- **physical-surface (watch/embedded/TV/kiosk):** R0 at the *real* surface budget (a 16×2 LCD is a 16×2
  box, honestly); R1 still at exact dimensions; R2/R3 need the sim or the board → reachable only if found,
  else floor R1 + flash/deploy handoff. Odd input (crown/single-button/IR remote) is annotated, not faked.
- **structural-only (design, no code target):** there is no R2/R3 by definition — the deliverable **is**
  the surfaces/flow. Tops out at R1 (structure + a still). This is a ceiling, not a degradation.

## Honest degradation + handoff (enforced at every rung)

1. Never emit an artifact at a rung you could not actually produce — no fake screenshots, no
   browser-pretending-to-be-native, no "imagine the headset".
2. When a rung is blocked, the **show** step prints exactly which capability was missing (quote the probe
   evidence), and the loop parks at the highest **reached** rung.
3. Emit a **handoff**: the real project files + the exact command a human-with-the-missing-capability runs
   to climb the last rung (`open X.xcodeproj && ⌘R`; `gradlew installDebug`; `deploy to <headset>`; …).
4. A blocked higher rung **never invalidates** a lower accepted rung — R0/R1 acceptance stands.
5. The litmus the critique enforces at **every** rung: *"could a reviewer be misled into thinking they're
   seeing real-target fidelity?"* — if yes, downgrade the claim and label it. This is the source skill's
   "report the boot failure verbatim, never fake a running server", generalized to all rungs and media.

Terminal vocabulary: a fidelity-elevation that built but couldn't run ends **`built-not-run`** (with the
command), never a silent green — mirroring how `verify`/`autopilot` treat UNVERIFIED.
