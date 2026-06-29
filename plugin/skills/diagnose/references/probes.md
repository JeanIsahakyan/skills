# Capability probe + probe registry

Probes are how the abductive loop spends its budget: a probe is a **controlled experiment over one axis
with a predicted outcome**. None are hardcoded per stack — each is a *parameterized intent* ("split this
search dimension in half and observe") **realized** against the repo via a one-time capability probe.

## The capability probe (run once, Phase 1)

A cheap, read-only probe that discovers — never assumes — how **this** repo works, and writes
`<out>/CAP.json`. It must answer:

- **build / run / reproduce / observe:** the exact commands to build, to run the failing thing, and to
  read the symptom signature (a test id, a CLI invocation, an HTTP request, a binlog replay, a log grep).
- **axis realizers:** for each search axis, the concrete operator available here:
  - commit → `git bisect` (is it a git repo with a known-good ref?)
  - input → a delta/minimization harness, or hand-minimization of the payload
  - config/env → which knobs/flags/env vars exist and are togglable
  - persistent-state → how to inspect/replay on-disk state (binlog/DB/cache dump; replay-to-offset)
  - interleaving/timing → a repeat-runner? a sanitizer? a way to force order/concurrency?
  - build/toolchain → alternate flags/compilers/opt-levels
- **instrument-blindness (load-bearing):** for every instrument, what it is BLIND to. Examples: a race
  detector (TSAN/-race) on a **single-threaded event loop** observes nothing about logical-ordering bugs →
  a clean result is **non-exonerating**; a coverage tool says nothing about data values; a type checker
  says nothing about runtime state. **Never branch on a negative from a structurally-blind instrument.**
- **achievable control surface:** the controls that ACTUALLY exist, not the ones a generic debugger
  assumes. If the test runner has no seed/order/repeat flag, in-memory test-order flakiness can't be
  actuated here — the real nondeterminism axis is elsewhere (on-disk/persistent state, process timing,
  container iteration order, float non-associativity, wall-clock). A realizer the probe **names but can't
  find** (a binary that doesn't exist, an access you don't have) is a **BLOCKED** outcome for any probe
  that needs it — stated with exactly what would unblock it, never silently skipped.

`CAP.json` is consulted by every probe; a probe whose realizer is a `CAP` gap cannot run.

## Symptom → axis → discriminating-probe registry

Classify the defect, then bias the priors toward its characteristic axis (a **prior**, not a commitment —
keep ≥2 axes live). For each symptom, the cheapest experiment that halves the axis:

| Symptom | Likely axis (prior) | Discriminating probe |
|---|---|---|
| failing test (deterministic) | recent change / logic | `git bisect` the commit range to the first-bad commit, then localize within it |
| crash / panic + stacktrace | the faulting frame's inputs/state | reproduce with the captured input; minimize it; inspect the state the frame reads |
| wrong output, no error | data flow | binary-search the pipeline: assert the value at the midpoint; halve toward where correct→wrong |
| perf regression | a commit OR an accumulation/rate | bisect with a *threshold* oracle if commit-shaped; if gradual/load-shaped, profile + measure the rate, not a point |
| flaky / "alone passes, in suite fails" | interleaving / shared persistent state / ordering | **amplify first** (drive p̂ → 0/1 via the achievable knob); only then bisect a now-reproducible handle; A/B alone-vs-in-suite to find the neighbor that dirties shared state |
| build failure | toolchain / deps / generated code | bisect the commit / diff the toolchain+env / regenerate codegen and diff |
| env-dependent ("works on my machine") | config/env delta | differential: capture the env where it fails vs where it passes; toggle one delta at a time |
| prod incident from logs | the request/state that triggered it | reconstruct the trigger from logs; replay it against a controlled instance |

**The flaky trap (the hardest, and the most punishing real-world class).** For an intermittent failure, the deterministic
axes are a *lie*: `git bisect` will converge on an innocent commit because a "good" run merely didn't fire
the race that time. So the registry **refuses deterministic-axis probes until the symptom is amplified to a
reproducible handle** (≥M/N under a captured control). If amplification can't make it deterministic within
budget, the honest output is the rate + CI, the narrowed interleaving/ordering class, and the
discriminating next-evidence — never the suspected location dressed as proven.

**Replay-differential as a first-class probe.** For "save and replay disagree" / "replica diverges from
primary" / order-over-a-long-sequence bugs, the discriminating probe is a **differential replay of a
captured op-log** (run the same sequence two ways, diff the resulting state, bisect to the first diverging
op) — not toggling one statement. The cause there is a whole-sequence invariant, not a point.
