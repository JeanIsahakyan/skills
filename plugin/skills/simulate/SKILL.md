---
name: simulate
description: >-
  Stress-test a spec or design doc BEFORE writing code by mentally executing it end-to-end — acting
  simultaneously as every component (data stores, caches, services, clients, external systems,
  transports), tracking explicit state diffs after every step, and running golden, edge, and
  adversarial scenarios until findings settle to LOW severity. Use this skill whenever the user has a
  spec / design doc / RFC / PRD and wants to validate it before building: "simulate this spec",
  "stress-test this design", "act as every component", "walk through every flow", "pretend you are the
  whole system", "find the holes / races / edge cases in this design", "is this spec internally
  consistent", "check for ambiguities / contradictions / missing rules before we implement". It runs a
  spec-analyzer → parallel scenario-simulators → findings-aggregator loop autonomously (no questions
  mid-run), one file per scenario, and produces a deduped, severity-ranked FINDINGS.md. Best for any
  non-trivial design with state, concurrency, auth, money, or external systems — exactly where spec
  bugs hide.
---

# simulate

Validate a spec end-to-end by walking through it as **every component simultaneously** — data stores,
services, clients, external systems, transports. Track explicit state after every meaningful step. Run
golden paths, edge cases, and adversarial scenarios. Loop autonomously until only LOW-severity findings
remain. The deliverable is a folder of per-scenario traces plus a deduped, severity-ranked `FINDINGS.md`
— **caught before a line of code is written.**

This skill is stack- and domain-agnostic: it reads a spec, builds a mental model of the system it
describes, and executes that model against itself. It never writes implementation code and never edits
the spec.

## Core principles

1. **State is sacred.** After every meaningful write / state transition / cache update / token issuance,
   show the diff. Read-only lookups don't count.
2. **Simulate every component, never collapse them.** If an action writes the DB, emits an event, updates
   a cache, and sends an email, all four show up. Missing one is a simulation bug — the spec probably
   left it unspecified too.
3. **Negative > positive.** ≥ 30% of scenarios are failures, races, and attacks. That is where spec bugs
   hide.
4. **Match spec vocabulary exactly.** Same table names, column names, event names, error codes.
   Discrepancies between simulation and spec hide real bugs under noise.
5. **When the spec is silent, STOP and flag.** Don't invent behaviour. Mark `⚠ Spec gap: <what's
   missing>`, pick the most defensible assumption, explain why, and continue. Collect gaps into findings.
6. **Run autonomously.** The skill operates in a loop with no user interaction inside a run. It plans,
   simulates, aggregates, decides whether to keep going, and stops on its own when severity settles to LOW.
7. **One file per scenario.** Each scenario writes its own file so simulations can run in parallel and
   individual scenarios can be re-run without disturbing the others.

## Output location — one folder per spec, one file per scenario

All simulation artifacts live in **one folder**, next to the spec (or wherever `--out` points):

```
<spec>.sim/                           (default: <spec-filename-without-ext>.sim/ beside the spec)
├── _analysis.md                      (mental model, written by spec-analyzer-agent)
├── _queue.json                       (live scenario queue, mutated each iteration)
├── sim-001-<scenario-slug>.md        (one per scenario, written by scenario-simulator-agent)
├── sim-002-<scenario-slug>.md
├── ...
└── FINDINGS.md                       (live aggregate, written by findings-aggregator-agent)
```

The spec itself stays **read-only**. Keeping all sims in one self-naming folder makes runs trivially
parallel across specs — each spec writes to its own `<spec>.sim/`, no clashes.

## Subagent roster

Three specialised subagents. Their full prompts live in `agents/`.

| Agent | File | Role |
|-------|------|------|
| Spec analyzer | `agents/spec-analyzer-agent.md` | Reads the spec once. Extracts actors / data models / interfaces / numbered invariants / scenario inventory. Writes `_analysis.md` + `_queue.json`. |
| Scenario simulator | `agents/scenario-simulator-agent.md` | Runs **one** scenario from the queue. Writes one self-contained `sim-NNN-<slug>.md` with the full deterministic trace, state diffs, invariant checks, and severity-classified findings. Multiple instances run in parallel within an iteration. |
| Findings aggregator | `agents/findings-aggregator-agent.md` | Reads every `sim-*.md` + `_analysis.md`. Dedupes findings into `FINDINGS.md`, classifies severity, tracks invariant coverage, decides whether to continue the loop, and may append follow-up scenarios to `_queue.json`. |

## Orchestration flow

The orchestrator (this SKILL.md) drives the outer loop. The agents do the work. Pass each agent the
**explicit paths** it needs (the spec path and the `<spec>.sim/` output dir) — the agents construct no
paths of their own.

### Phase 0 — Resolve the spec and output dir

Resolve the target spec, in this order:

1. Explicit path arg → a file that exists.
2. An `@path/to/spec.md` reference in the user message.
3. A spec pasted inline → write it to `./<inline-slug>.md` first, then proceed.
4. Otherwise the most-recently-edited likely spec (`*.md` whose content reads like a spec/design/RFC/PRD)
   in the working area — confirm the pick in the report header.
5. Nothing resolvable → ask the user for the path. Stop.

Compute the output dir: `--out <dir>` if given, else `<spec-dir>/<spec-stem>.sim/` (e.g. `docs/payments.md`
→ `docs/payments.sim/`).

If the output dir already has an `_analysis.md`, treat this as a **resume**: load the existing
`_analysis.md` and `_queue.json` and continue from where the previous run stopped. Do not re-analyze
unless the spec has been edited since the analysis was written (compare mtime).

### Phase 1 — Analyze

If `_analysis.md` does not exist (or the spec is newer):

- Spawn the **spec-analyzer-agent** synchronously, passing the spec path and the output dir.
- It writes `_analysis.md` (the mental model) and `_queue.json` (initial scenario queue, prioritised).
- Return only the one-line confirmation; do not echo the content.

### Phase 2 — Loop

Repeat until a stop condition is reached:

1. **Pick the next batch.** Read `_queue.json`. Take the next **N** pending scenarios (`N = 3` by default;
   `--parallel=K`). If fewer than N remain, take what is there. If zero remain and no follow-ups are
   pending, stop with reason `"queue empty"`.
2. **Simulate in parallel.** Spawn **N scenario-simulator-agent** calls **in parallel** in a single
   message. Each receives one scenario plus its output path (`<out>/sim-<NNN>-<scenario-slug>.md`). The
   counter `NNN` increases monotonically across iterations (zero-padded to 3 digits).
3. **Wait for all N to complete.**
4. **Aggregate.** Spawn the **findings-aggregator-agent** synchronously. It reads `_analysis.md`, every
   `sim-*.md`, and the previous `FINDINGS.md` (if any). It rewrites `FINDINGS.md`, may append follow-up
   scenarios to `_queue.json`, and returns a JSON decision block with `should_continue`, `stop_reason`,
   and stats.
5. **Decide.** If `should_continue == false` → exit the loop. Otherwise → next iteration.

### Phase 3 — Stop and report

Stop conditions (any one):

- `should_continue == false` from the aggregator (only LOW remaining, queue empty, all invariants exercised).
- Iteration cap hit: **8 iterations max**.
- Scenario cap hit: **60 scenarios max** across the whole run.
- Hard error from an agent that prevents progress.

On stop, print a terse inline report:

```
Simulation complete: <N> iterations, <M> scenarios written.
Folder: <spec>.sim/
Severity: HIGH <h> · MED <m> · LOW <l>
Invariants: <verified>/<total>
Stop reason: <reason>

Top open findings (full list in FINDINGS.md):
- [<SEV>] <one-line summary> — sim-<NNN>
- [<SEV>] <...>
```

Never paste the full FINDINGS.md inline — it lives in the file. If HIGH findings remain, lead the report
with them: catching a spec problem before implementation is the whole point.

## Parallelism

- **Within one iteration**, scenario-simulator-agent calls run in parallel (single message, multiple tool
  uses).
- **Across specs**, invoke the skill twice (different spec paths) — each writes to its own `<spec>.sim/`
  folder, no contention.
- The aggregator step is always sequential after a parallel batch — it needs every file on disk first.

## Guardrails

- **Iteration cap** — 8. Beyond this the loop is probably oscillating; stop and report.
- **Scenario cap** — 60. Hard ceiling on `sim-*.md` files per spec.
- **Per-iteration progress check** — if an iteration adds no new HIGH or MED findings and the queue did
  not shrink, the aggregator must declare `should_continue: false` even if its other heuristics disagree.
  This prevents busy-looping on LOWs.
- **No spec mutation.** This skill never edits the spec or any companion doc. Read-only.
- **No code generation.** It produces simulation traces and findings — never implementation code.

## Args supported

- `<spec-path>` — the spec to simulate (required unless resolvable from context).
- `--out <dir>` — output dir (default `<spec-dir>/<spec-stem>.sim/`).
- `--parallel=K` — scenario-simulator agents per iteration (default 3, max 6).
- `--depth=shallow|deep` — bound scenario inventory (default deep).
- `--attacks-only` — skip golden-path scenarios; queue holds edge + attack only.
- `--focus=<phrase>` — scope scenarios to a named area of the spec.

## What this skill never does

- Asks the user mid-run. The whole loop is autonomous.
- Writes a single monolithic simulation file. One file per scenario, always.
- Invents behaviour the spec did not specify. Always flag, assume defensibly, continue.
- Edits the spec. Read-only.
- Generates implementation code.
- Skips invariant verification in the aggregator step.

## Deliverable

The `<spec>.sim/` folder — every `sim-NNN-*.md` trace, `_analysis.md`, `_queue.json`, and the
severity-ranked `FINDINGS.md` — is the output. That folder is the simulation record; `FINDINGS.md` is the
thing to act on. If you're in a git repo and want a record, commit the folder — but that's your call, not
something this skill does for you.
