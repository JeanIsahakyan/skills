---
name: autopilot
description: >-
  Universal autonomous feature/project builder: turns roughly one prompt into a whole working feature
  or project — maximum code, minimum ceremony — without shipping confidently-wrong garbage. Use this
  skill whenever the user wants something built end-to-end autonomously from a prompt or a spec/design
  doc: "build the whole X", "implement this entire feature", "scaffold and finish this project", "ship
  this from the doc", "do the whole thing", or when they say autopilot / build it all / one-shot it.
  Also use when a feature is described by a doc/issue/PR and the user wants it driven to done in one
  pass. It runs a premise-gate (refuses to build the wrong/redundant thing), a shared classification
  bus (so high-blast decisions can't be silently defaulted), skeleton-first scaffolding, an
  acceptance+invariants oracle, a bounded convergence loop, a human-judgment feedback gate for
  aesthetic/irreversible slices, and salience-ranked disclosure. Best for substantial multi-file
  builds, not trivial one-line edits.
---

# autopilot

Turn ~one prompt into a whole feature or project: **maximum code, minimum ceremony, without
confidently-wrong garbage.** This skill is a *policy*, not a fixed fleet of agents — it discovers
the project's substrate at runtime and adapts, so it works on any stack and any deliverable (typed
backend, untyped scripts, frontend, mobile, CLI, data/ETL, IaC, library/SDK, ML, docs, config).

The central design fact that shapes everything below: **a machine optimized only for the forward pass
(prompt → code) makes a confidently-wrong interpretation look maximally green.** The scaffold is
self-consistent, tests pass against a prediction authored from the same misread, and a tiny "DONE"
report buries the riskiest decisions. So "max code" is an *outcome* of cutting genuine waste
(narration, prose specs, self-re-reads) — **never a target you optimize toward.** The things that make
one-shot building *safe* live at the two ends: a **premise gate** in front (don't build the wrong
thing) and a **human-feedback oracle + abort-clean** in back (converge on a moving target, fail
recoverably). The middle just produces volume.

## Modes and depth

Invoke as `autopilot <mode> [--full|--spike] <prompt | path-to-spec>`.

- **`plan`** — run premise-gate + bus probe + skeleton + acceptance stubs + invariants, then **STOP**
  and hand back those reviewable artifacts. No behavior code. This is the human's leverage point for
  high-stakes or ambiguous work — review the *shape* and the *flagged decisions* before any fill.
- **`build`** *(default)* — run the full pipeline to a terminal, one final report. No mid-flight
  questions except the gated human-judgment oracle and the hard STOP boundaries below.
- **`ship`** — `build` + cross the outward boundary (commit / branch / PR). **Choosing this mode is the
  authorization** for that outward step. Genuinely destructive acts (deleting work you didn't create,
  a production apply) still stop unless named.

Depth: **`--full`** *(default)* = the production vertical (the acceptance suite, invariants oracle,
edge cases, integration, lint). **`--spike`** = the smallest runnable thing; drops the invariants
artifact and most of the acceptance suite for a throwaway/exploration build.

## The aggressive default — and its one override

Default posture: **never ask on under-specified details.** Take a least-surprise default, log it,
continue. This is what makes one-prompt builds fast.

The default is overridden in exactly two places, and only these:

1. **The premise gate** (Stage 0). A confirmed-false premise beats least-surprise-and-continue.
2. **A bus-classified hard boundary** — an irreversible/outward decision the bus marks `open`/`STOP`
   (see the classification axes). These are flagged loud or stopped, never silent-defaulted.

Everything else: decide, log, keep moving. The log is not optional — an unread default is operationally
identical to no decision, so disclosure (Stage 6) ranks the logged decisions by blast radius.

## Pipeline

```
PREMISE GATE → BUS PROBE → SKELETON → ACCEPTANCE+INVARIANTS → CONVERGENCE
  → HUMAN-JUDGMENT ORACLE → DISCLOSURE → terminal ∈ {GREEN, ESCALATE, UNVERIFIED, ABORTED-with-map}
```

Run the stages in order. Each stage **reads** its classifications from the bus (`references/bus.md`);
no stage re-derives them. Use `references/pillars.md` for the full degradation ladder of each stage —
the rung autopilot operates at depends on what the substrate probe finds, and every stage has an honest
floor it bottoms out at rather than faking a guarantee.

### Stage 0 — Premise gate (the WHAT-check)

Before anything else, cheaply try to **falsify the goal** with target-local reads (reuse these reads
for the bus probe — they are not wasted):

- **already-satisfied** — does the requested behavior already exist? (the index, the cache, the retry,
  the dedup, the doc section).
- **self-contradictory** — does the goal demand two things the codebase makes mutually exclusive?
  (cache a mutation; dedup a stream where dupes are contractually allowed).
- **false-premise** — does the goal assert a fact a one-shot read disproves? ("X returns JSON" when X
  returns protobuf).

The premise gate exists to stop **confidently-wrong** output — **not** to stop output. So a tripped
check *always surfaces the finding loudly at the top*, then takes the cheapest correct of two actions:

- **Recover-and-deliver** *(the default whenever the corrected intent is obvious and safe)* — state the
  correction, then deliver the *corrected* thing. "Document the JSON response of this endpoint" against a
  protobuf endpoint becomes "this endpoint returns protobuf, not JSON — here are the protobuf docs." An
  already-satisfied request reports the existing implementation (and the nearest genuinely useful
  adjacent improvement, if one is obvious). **Delivering the right artifact beats halting empty-handed.**
- **STOP** *(only when recovery is unsafe or ambiguous)* — the corrected intent is genuinely unclear,
  would require inventing a material product decision, or the corrected action is irreversible/outward.
  Then halt with the evidence and one named question.

So: a no-op build is never silently performed, and a confidently-wrong build is never performed at all —
but **catching the wrong premise is the win; halting is the fallback, not the goal.** This is the one
place the aggressive default is overridden, and the override is "don't ship confidently-wrong," *not*
"don't ship." If the goal survives all three checks, proceed normally.

### Stage 1 — Bus probe (classify once, authoritatively)

Run **one** probe over the target path and write the sealed classification record `BUS.json` (full spec
in `references/bus.md`). Every later stage reads it; none re-classifies. The probe:

- Discovers the **substrate** (deliverable kind; whether effects can escape the workspace; whether any
  way to check behavior exists — a harness, a REPL, a runtime, a human) — *described*, never assumed.
- Classifies each decision the build will require on universal axes: **category** (C1 public-contract /
  C2 persistent-format / C3 destructive / C4 trust-boundary / C5 outward-effect / C6 real-world-mutation
  / LOW internal-cosmetic), **reach** (local / external / **unknown→escalate, never optimistic local**),
  **effect_locality**, **oracle_purity**, and **provenance**.
- Applies three load-bearing rules at write time so downstream just obeys a verdict:
  - **Provenance-of-value**: any value crossing a serialization/computation boundary (magic number,
    default, field name/tag, format, units, version) is **invent-by-default** unless a *resolvable
    upstream* pins it — regardless of whether a consumer edge is visible. "No discoverable consumer ≠
    no consumer." An agent cannot assert "I'm adopting"; it must cite a source that resolves.
  - **Greenfield-foundational override**: any decision in **C1/C2/C4** where there is *no per-axis
    precedent* is forced to `open`, even on greenfield. Birth-time architectural commitments (auth
    model, data model, wire format) are the *least* reversible, not the most — `precedent==0 ⇒
    reversible` is the inverse of the truth for foundational decisions.
  - **Effect-locality**: a decision whose verifier mutates/consumes external state is marked
    `stateful-consuming` (→ convergence cap-1) and `external` (→ needs a realization receipt, not a
    workspace fingerprint).

**Scale the apparatus to the work.** The full sealed `BUS.json` + reconciler exists to fix the
cross-stage seam — it earns its cost only when the build actually decomposes into multiple stages or
parallel agents. For a trivial single-artifact task (one file, one doc, a one-line edit), do **not**
emit the full apparatus: run the premise gate, classify the one or two decisions inline, act, and
disclose. The classification *rules* still apply (a boundary-crossing value is still invent/flagged);
only the bookkeeping collapses. Don't pay multi-agent ceremony for single-agent work — that overhead is
pure cost when there is no seam to protect.

### Stage 2 — Skeleton-first (shape, loudly unfinished)

Emit the **shape** before any behavior: every boundary the feature consists of (file, exported symbol,
route, schema column, resource block, doc heading…) exists as a named, addressable slot, and **every
unfilled slot carries a uniform, greppable, countable "unfinished" marker** whose survival into "done"
is a hard failure. Pick the *strongest unfinish signal the substrate supports* (typed throw > runtime
raise > failing assert > nonzero-exit > reserved sentinel value > `TODO(autopilot:id)` grep-token) and
the cheapest *load-gate* that proves the shape **resolves** without proving behavior (compile / import /
parse / `terraform validate` / render-mounts / structural lint).

Two guards keep this honest — shape that *resolves* is not shape that is *correct*:

- **Flag-not-silent on bus-HIGH**: a slot the bus marked `open` is emitted as a flagged open decision,
  not a resolved scaffold. On a library/SDK the invented public signature **is** the product and is
  irreversible-outward — never a cosmetic default.
- **Contract-anchor + negative-space**: locate an external contract anchor (existing callers, an
  interface to satisfy, an openapi/proto/`__all__`/exports list). Diff the emitted boundary set against
  it — a boundary that was *forgotten* is invisible to every fill-side gate, so an **absent** boundary
  must be a loud failure. Where no anchor exists, report the boundary set as `UNVERIFIED-COMPLETE`, and
  report marker-count as "syntactic progress," not as trust.

### Stage 3 — Acceptance + invariants oracle (RED before code)

Author the acceptance checks in the substrate's native harness **before** the code, run them to confirm
they fail for a **clean behavior reason** (not a compile/dispatch error — that distinction is the whole
value of the gate), then fill code until they pass.

The independence that matters has two axes, and the second is the one usually missed:

- **Author independence** (satisfier ≠ asserter) licenses *shape* claims only.
- **Referential independence** licenses *behavior* claims: a **ground-truth** oracle whose answer is
  obtained *without re-reading the spec* — a fixture with human-supplied expected outputs, a golden from
  a prior trusted run, or a different-provenance computation. Round-trip/idempotency/conservation are
  **tautological** oracles: keep them, but tag them "shape-only — does NOT verify the transform is the
  *right* transform." A rung is "verified" only if a ground-truth oracle exists; otherwise it is
  **capped at "shape-verified, semantics-UNVERIFIED"** and surfaced as a flagged question — the same
  treatment as an aesthetic slice, applied to ETL/ML/CLI semantics.

### Stage 4 — Convergence loop + resumable cursor (the backward pass)

This is the load-bearing half that "min everything" tends to cut — and the one thing that cannot be cut,
because it is what makes aggressively-defaulted unreviewed bulk code *converge on intent* instead of
compounding away from it. Fund it **first**; code volume is the residual.

- **Loop**: build → run the discovered check → on RED, re-spawn the owning unit's agent with the failing
  output + notes, bounded by a hard iteration cap, then **escalate to STOP on cap-exceeded.** A fill
  agent whose provisional signature can't express its body **raises an amendment** (one round-trip)
  rather than shipping wrong-but-compiling code.
- **Effect-locality guard**: if the unit's oracle is `stateful-consuming` (e.g. `terraform plan` against
  live state, a one-way migration, a publish), the loop is **disabled** — run once, cap-1, any non-GREEN
  is immediate escalate, because re-running doesn't re-measure the same thing.
- **Resumable cursor**: persist a phase cursor so re-running the *same prompt* is **idempotent** —
  already-landed work (by file/decision fingerprint) is detected and skipped, never double-applied. For
  external effects, idempotence needs a **realization receipt** (the resource id / migration version /
  published tag), not a workspace fingerprint.

A non-monotone oracle (an ML metric over a slow noisy run) is **not** a convergence target: run once,
checkpoint the metric+seed, escalate on miss — the cursor half survives, the loop half does not.

### Stage 5 — Human-judgment oracle with a feedback edge

For the irreducibly-human slice (visual/aesthetic, prose/doc quality, "is this design good", UX feel) a
machine oracle cannot self-certify, and a second agent grading the screenshot shares the first's
aesthetic priors (the tautological-green loop, relocated). So **"needs-human-verify" is not a dead-end
leaf — it is an oracle node with a feedback edge:**

- Emit a structured handoff: the artifact bundle to inspect (screenshot / rendered doc / `plan` diff)
  and **one named falsifiable yes/no question per high-blast slice**, ranked by blast radius.
- The human verdict re-enters the convergence loop as RED/GREEN; a "no" re-spawns the owning agent with
  the note (bounded human-iteration cap — taste targets escalate, they don't loop forever).
- For prose deliverables this oracle also does **content-grounding**: every factual claim that references
  the code is grep-verified against the code; unverifiable claims are flagged. Structural gates (headings
  present, links resolve, YAML validates) verify *form*; this verifies *content is true*.

### Stage 6 — Disclosure (salience-ranked, length scales with risk)

The final report is **not minimum-bytes** — minimum ceremony means *minimum bytes above the irreducible
risk-disclosure floor.* Rank by blast radius (the bus's `blast_rank`):

- **Top**: the 2–3 highest-blast inventions, each with the rejected alternative and the **one-command
  flip** to change it. These are the decisions worth a human's attention; never bury them.
- **Collapsed below**: the cosmetic/derive tail as a count.
- **Always present**: the rung reached and what is *not* verified. A `shape-verified, semantics-
  UNVERIFIED` or `needs-human-verify` slice is stated loudly at the top with the exact command to check
  it. Silence is never an option; the floor is an explicit confession, not a gap.

Disclosure reads its salience from the **same bus classification** the build acted on — so a decision
flagged high-blast for arbitration is byte-identically the one disclosed. This is a checked fact, not an
assertion (see desync detection in `references/bus.md`).

### Terminals

Every run ends in exactly one of four states — never a silent "done":

- **GREEN** — converged; acceptance + invariants + (where applicable) human verdict all clean.
- **ESCALATE** — a STOP that needs a human: an unanswerable boundary, a cap-exceeded unit, a
  realization-drift, a stateful oracle that came back non-GREEN.
- **UNVERIFIED** — landed, but a slice could not be machine-checked (aesthetic / no oracle / external);
  flagged top-of-report with the check command.
- **ABORTED-with-map** — the run could not complete. Either **revert** to the last cursor-coherent
  checkpoint, or, if revert is unsafe, **leave the partial work with every sentinel marker intact plus a
  MAP at the report head** ("these N boundaries are stubbed, this migration is half-applied, here is the
  exact state and the one command to finish-or-roll-back"). The grep-sweep for surviving markers is the
  abort-cleanliness check — an abort is clean iff every incomplete boundary is loudly marked and mapped.

## Orchestration — how many agents

autopilot is a policy first; **most invocations should spawn few or zero subagents.** Decide at runtime
from the *discovered decomposition*, not a constant:

- **Trivial in a known substrate** → inline, 0 subagents.
- **Decomposable** → use the Workflow tool: a discovery/skeleton pass, then **fan out one fill-agent per
  independent boundary cluster** (worktree-isolated only if they mutate concurrently and would
  conflict), then the convergence + verification panel.
- Fan out **only** when leaf-derivation yields ≥3 genuinely independent units. On software whose logic
  lives in one central mutable model, the dependency graph is a star, decomposition yields ~1 fat unit,
  and parallel fill buys nothing — inline a single focused writer instead. Do not pay scaffold +
  orchestration overhead for zero parallelism.

Typical totals: trivial 0; a normal feature ~2–3 (one discovery + inline impl + 1–2 verify); a large
multi-file feature ~8–12, capped by the concurrency limit. The number is discovered, not fixed.

## References

- `references/bus.md` — the shared classification bus: exact record schema, write-once / append-only
  protocol, read + checked-fact reconciliation, desync detection, and the staleness rule. Read this when
  implementing Stage 1 or any stage's classification lookup.
- `references/pillars.md` — the full degradation ladder for every stage (ideal substrate → fallbacks →
  honest floor) and how the runtime substrate probe selects the rung. Read this when a stage needs to
  operate on an unusual substrate (untyped, unobservable, non-code, side-effect-dominant).
