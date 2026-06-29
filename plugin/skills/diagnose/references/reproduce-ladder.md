# Reproduce gate + the can't-reproduce ladder

Reproduce-first is the gate that beats ad-hoc debugging: **no hypothesis is ranked, no probe is designed,
and no fix is proposed until diagnose holds a REPRO HANDLE** — a named, replayable command that produces
the symptom on demand (deterministic) or at a measured rate (statistical). The genuinely hard part is not
the deterministic case; it is the descent when you *can't* cheaply reproduce — and the discipline is that
"can't reproduce" is **not an exit**, it is a typed ladder with an honest floor.

## Determinism classes (Phase 2 output)

Run the reproduce command N times; record the signature hit-rate and classify:

- **deterministic** — fails every run under the handle. Downstream probes are single-shot.
- **statistical** — fails k/N (record `p̂` + a confidence interval). **Every downstream probe becomes a
  two-sample test**, never single-shot: compare the signature rate control-OFF vs control-ON over N runs
  each, with N large enough that a real swing (e.g. 40% → 0%) can't be a fluctuation. A single green run
  **never** kills a flaky hypothesis.
- **unreproduced** — not seen within the budget. Descend the ladder; if it bottoms out, emit the armed
  trap (below). Never proceed to causes from here.

## The can't-reproduce ladder (descend in order)

1. **Amplify on the achievable knob.** Drive the rate toward 0 or 1 using a control that *exists here*
   (`CAP.json`): more iterations, the real load/concurrency, a fixed seed/order *if the runner has one*,
   the failing environment/container rather than local. "0/1000 local, 3/900 in-container" is itself
   evidence — it localizes to the env/concurrency delta. The knob that moves the rate **is** the axis.
2. **Find the nondeterminism source before assuming a mechanism.** Vary ONE source at a time and watch the
   rate: writer-count (single-thread vs many) → race vs not; suite-order / prior-test-first → shared
   persistent-state residue (an *ordering* bug, not a race); fixed seed → seed-dependent data; pinned
   wall-clock → time/decay dependence; container iteration order / float order → environment. For a
   **single-writer** runtime the source is usually *not* threads — it is on-disk replay state, process
   timing, iteration order, or float non-associativity; don't burn budget on thread-interleaving knobs
   that don't apply.
3. **Force the medium deterministic.** Where a sanitizer/checker *can* see the class (and is not
   structurally blind — see instrument-blindness in `probes.md`), rebuild under it: a TSAN race report or
   an ASAN overflow is a deterministic repro even when the failure was statistical, collapsing the whole
   problem. A **clean** result from a blind instrument is non-exonerating — do not treat it as evidence.
4. **Honest floor — the armed trap.** If the budget is spent and the rate can't be lifted above noise nor
   made deterministic: emit terminal **NOT-REPRODUCED** with an **ARMED TRAP** package, not a guess:
   - the **narrowed cause-space** (e.g. "shared static between test_X and test_Y, OR a real seam race at
     the flush/queue boundary — not yet discriminated"), with `p̂` + CI and what was tried;
   - **committed instrumentation** that turns the next occurrence into a deterministic repro: an assert of
     the suspected invariant at the seam, a structured dump of the suspect variables, an op-log / seed
     capture, a sanitizer canary in CI;
   - a **per-hypothesis discriminator** — the exact next-occurrence evidence that would kill each branch.

   The skill physically cannot emit "the cause is X" without a satisfied differential-confirm; absent it,
   the only legal output is the trap.

## Why this beats ad-hoc

The default agent, unable to reproduce locally, treats a local green run as exoneration and pattern-matches
a plausible cause. The ladder forbids that: a green run from a blind instrument or an un-amplified handle is
**not evidence**, and the only outputs are a reproducible handle, a narrowed space, or an armed trap — never
a confident guess.
