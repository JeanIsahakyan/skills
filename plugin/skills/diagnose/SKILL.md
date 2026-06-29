---
name: diagnose
description: >-
  Backward root-cause debugging: start from an observed defect and work to a PROVEN cause — never a
  guessed one. Use whenever something is broken and you need to know WHY before fixing: "why is this
  failing / broken", "debug this", "find the root cause", "this test is flaky / passes alone fails in
  the suite", "it crashes / errors / returns the wrong thing", "this regressed", "track down this bug",
  "what's causing X", a stacktrace, a failing CI run, a heisenbug, a perf regression, a prod incident
  from logs. It is the inverse of the build skills: the cause is UNKNOWN and the work is abductive
  elimination. It refuses to name a cause until the cause reproduces on demand, vanishes under an
  isolated control, and returns when un-controlled — killing hypotheses only by evidence that forbids
  them, never by plausibility. It stops at "cause proven + minimal fix proposed + handoff" and does NOT
  build the fix (that's autopilot's or your job). Stack-agnostic via runtime discovery (no per-stack
  hardcode); on a bounded-exhausted search it reports the narrowed space + the one discriminating
  experiment, never a confident guess.
---

# diagnose

The collection's only **backward** skill. Everything else flows forward (intent → artifact) or confirms a
known change; `diagnose` starts from an **observed defect** and infers the **cause** that produced it.
Its whole value is method: it makes the cause a **checked fact before any fix is written** — because a
fix authored from a misread cause sails through any forward verifier authored from the same misread.

## The one rule

**No cause is named until it is differentially proven.** A hypothesis becomes "the cause" only when:
the symptom **reproduces** on demand → **vanishes** when the suspected mechanism is held under an
*isolated* control → **returns** when the control is released. Hypotheses are killed **only by evidence
that forbids them**, never by lower plausibility. The classic ad-hoc failure — latch onto the first
plausible cause and patch a symptom (confidently-wrong) — is structurally impossible here.

## Boundary — diagnose finds, it does not fix

It stops at **cause proven + a minimal, mechanism-impossible fix PROPOSED + handoff** (to `autopilot` to
build the fix, or to you). It never crosses the build boundary. Keeping diagnose out of the fix keeps its
falsification discipline uncontaminated by fix-confirmation bias.

## Phases

Run in order. Each phase reads its classifications from a single-writer, append-only **CASE ledger**
(`<out>/CASE.jsonl`); no phase re-derives what an earlier one established. Heavy machinery lives in the
two references; this is the spine.

### Phase 0 — Oracle validity (is this even a bug?)

**Before anything**, pin the exact machine-checkable **symptom signature** (assert text + `file:line` /
byte-diff of wrong output / panic frame / `p99 > threshold` / log pattern). Then immediately try to
**falsify that it is a defect at all**, two cheap moves:
- **blame-the-assertion** — find the contradicted code path; if it's load-bearing-by-design (a named
  invariant, a deliberate conditional, a comment explaining it), the *oracle* is suspect, not the code.
- **provenance** — `git blame` the oracle vs the implicated code; if the oracle is **newer** than the
  behavior, the test is the likelier-wrong party.

If the oracle is suspect → terminal **SPEC-SUSPECT** ("symptom reproduces but contradicts apparently
intentional behavior at `<file:line>` — resolve oracle-vs-code with a human first"). A sound loop on a
wrong oracle is *more* dangerous than ad-hoc, so this is the first gate, not a late check. The oracle is
an explicit, contestable field in every later artifact.

### Phase 1 — Capability probe (runtime discovery, no per-stack profile)

One probe discovers how **this** repo builds / runs / reproduces / observes the signature, and the
**axis realizers** (how to bisect commits / minimize input / toggle config / replay state / force
interleavings *here*). Two hard rules (see `references/probes.md`):
- **Instrument-blindness:** record what each instrument is BLIND to. A clean run from a structurally-blind
  instrument is **non-exonerating** — never branch on it (e.g. a race detector on a single-threaded event
  loop proves nothing).
- **Achievable control surface:** establish the controls that actually exist, don't assume them. If there
  is no seed/order/repeat knob, the in-memory flaky axes don't exist; the real nondeterminism axis may be
  on-disk/persistent state + process timing. A named-but-absent realizer is a **BLOCKED** terminal, never
  silently "ran it".

### Phase 2 — Reproduce gate (hard precondition)

Run the reproduce command N times; classify determinism ∈ `deterministic | statistical(p̂=k/N, CI) |
unreproduced`. **Nothing downstream runs** (no ranking, no probe, no fix) until determinism is
deterministic-or-statistical. If unreproduced within budget → terminal **NOT-REPRODUCED** with an
**ARMED TRAP** (committed instrumentation at the suspected seam + the exact next-occurrence evidence map)
— never a guessed cause. The can't-reproduce ladder + statistical handling: `references/reproduce-ladder.md`.

### Phase 3 — Seed the ledger (anti-tunnel)

Enumerate hypotheses across a **ranked set of axes** — force **≥2 axes live** so the loop can't tunnel on
the first. Each hypothesis is a **falsifiable mechanism (not a location)** with `predicts[]` and
`forbids[]`; a row with no `forbids[]` is rejected at write time. The symptom shape only sets **priors**
over axes; it never *commits* one.

### Phase 4 — Abductive loop (the core)

Each round, design the **single most-discriminating probe** — the experiment whose outcome most evenly
splits the live ledger by prior, minus cost (irreversible/stateful probes are cap-1 and down-weighted).
Usually a bisection on the dominant axis (`git bisect` / binary-search the input / toggle one config /
replay a log to offset N/2 / **differential-replay of a captured op-log** / forced interleaving) — but
only on a deterministic-or-statistical handle. Run it; **kill every hypothesis whose `forbids[]` the
observation hit**. For statistical symptoms every probe is a **two-sample test** with a confidence floor —
a single green run never kills a flaky hypothesis. A probe that kills nothing is low-information → the
selector must switch axis next round (anti-loop). Budget hit without localization → terminal **NARROWED**
(survivor ledger + bisected intervals + the one next-probe the budget didn't reach). Probe registry by
symptom: `references/probes.md`.

### Phase 5 — Localize + upstream-closure

When the live set collapses to one mechanism AND the axis is bisected to a point: trace the **corrupt
value** (not the symptom) one hop upstream — is the state already wrong *before* this point? If so, this
point is a **suppressor**; push one hop up and re-enter. **Bisect-integrity:** reject a "first-bad commit"
whose diff is non-semantic (format/move/refactor/opt-flags) unless you can forward-port the suspected
semantic line to the parent and reproduce — a refactor has no semantic line to port (the bug is older).

### Phase 6 — Cause-vs-symptom via all-consumers

A true cause gates **every independent consumer** of the suspect state — enumerate readers by grepping the
field's uses (the read path AND snapshot/serialize AND stats AND replication), not just the observed
symptom. A suppressor passes all *manifestations* but fails *all-consumers*. Fails → terminal
**SYMPTOM-ONLY** ("corrupt state persists upstream at `<byte/offset>`").

### Phase 7 — Differential confirm (mechanism-level)

Three legs, all required: (a) control-OFF → signature present; (b) control-ON → signature gone AND the
full suite stays green; (c) revert → signature returns. Two teeth that make it a *cause* proof, not just
on-path:
- The toggle must be **isolated** — flip only the suspect's *semantics* with timing/codegen/memory-layout
  held (a runtime flag or a same-codegen branch). Any reorder/added-logging toggle is **confounded** and
  cannot certify.
- It must prove the **hazard** is gone, not just the symptom — an asserted seam-invariant that holds under
  control and *fires* under un-control, or survival under an injected worst-case (slowest tick / forced
  interleaving / adversarial input). A symptom-rate that vanishes but whose hazard can't be shown removed
  is **TIMING-SUPPRESSED**, not proven — falls to the trap floor, never GREEN.

The proposed fix must be **mechanism-impossible** (the bad ordering can no longer occur), not assert-greening.

### Phase 8 — Adversarial falsification + search-empty

A second view (`agents/falsifier-agent.md`) that shares **no reasoning** — only `CASE.jsonl` + the oracle
+ the repro — tries to break: **necessity** (symptom WITH the control on), **sufficiency** (symptom off via
an unrelated change → confound), **minimality** (a simpler control also closes it → the real cause is
narrower), and **co-factor** (does reverting a *different* live candidate also fix it? → the cause is
conditional `A∧B`; report both — this is the conjunctive-fault catch). Then re-run the most-discriminating
probe **after** the fix to confirm the search space is now **empty** (the fix didn't mask a second latent
fault on the same path).

## Deliverable — the Cause Dossier

`<out>/CAUSE.md`, salience-ordered: (1) the isolated differential-confirm receipt (present→absent→returns,
verbatim, + the hazard-gone proof); (2) the cause as a falsifiable mechanism + the minimal reproducer +
the explicit oracle field; (3) the **killed-hypothesis ledger** (every rival + the evidence that killed it
— the audit that proves no tunneling); (4) the all-consumers + upstream-closure verdict; (5) the minimal
mechanism-impossible fix **proposed** + handoff (one-command repro for whoever builds it). Claim wording:
*"proven necessary-and-sufficient for the symptom under the stated oracle"* — not "fact".

## Terminals (never a silent done, never a guess)

`CAUSE-CONFIRMED` · `NARROWED` (budget hit: survivor ledger + intervals + one next-probe) ·
`NOT-REPRODUCED` (+ armed trap) · `SPEC-SUSPECT` (oracle likely wrong) · `SYMPTOM-ONLY` (suppressor;
corrupt state persists upstream) · `BLOCKED` (a required realizer is a capability gap — state exactly what
access unblocks it).

## Depth-scaling

When the oracle is valid, the repro is deterministic, and round-1 yields **one** hypothesis whose isolated
mechanism-level differential closes immediately, collapse to `oracle-check → reproduce → confirm → propose`
and skip the multi-round ledger ceremony. The full falsification apparatus (≥2 axes, the adversarial pass)
unfolds **only** when ≥2 axes survive round 1. Scale the apparatus to the bug.

## Subagent roster

| Agent | File | Role |
|-------|------|------|
| Oracle check | `agents/oracle-check-agent.md` | Phase 0 — is this a defect, or is the expectation wrong? Pins the signature; blame + provenance. |
| Probe | `agents/probe-agent.md` | Runs one most-discriminating probe (a controlled experiment over one axis); returns evidence + which hypotheses it kills. The loop worker. |
| Falsifier | `agents/falsifier-agent.md` | Phase 8 — shares no reasoning; tries to refute the leading cause (necessity / sufficiency / minimality / co-factor). |

## What this skill never does

- Names a cause without a differential-confirm receipt (reproduces → vanishes-under-isolated-control → returns).
- Proposes a fix before a cause is proven, or builds the fix (it hands off).
- Branches on a clean result from a structurally-blind instrument.
- "Fixes" intentional behavior — when the oracle is suspect it stops at SPEC-SUSPECT.
- Emits a guessed cause on a bounded-exhausted search — it emits NARROWED / an armed trap instead.

## References

- `references/probes.md` — the capability probe (build/run/reproduce/observe + instrument-blindness +
  achievable control surface) and the symptom → axis → discriminating-probe registry.
- `references/reproduce-ladder.md` — the reproduce gate, determinism classes, the can't-reproduce ladder,
  statistical two-sample handling, and the armed-trap floor.
