# Degradation ladders

Every stage is defined as a **role + a degradation ladder**, not a fixed mechanism. autopilot does not
assume types, a test harness, a compile step, or that the deliverable is even code. Instead a cheap,
side-effect-free **substrate probe of the target path** selects which rung a stage operates at, and each
ladder bottoms out at an **honest floor** that states what it cannot guarantee rather than faking one.

This is what makes the skill universal: the same role ("make shape loudly unfinished", "verify behavior",
"converge") is realized differently on a Go backend, a Python notebook, a React SPA, a Terraform root, or
a Markdown doc — and the report always says which rung was reached.

The project matrix the ladders are stress-tested against: typed-compiled backend · untyped scripting ·
frontend SPA · mobile · CLI · data/ETL/notebook · IaC/k8s/Terraform · library/SDK · shell glue · ML ·
greenfield / brownfield / new-subtree-in-monorepo · non-code deliverable (doc/schema/config/asset).

## The substrate probe (run once, feeds every ladder)

Three flags, each answered by looking, not by a hardcoded stack rule:

- `HAS_TYPED_COMPILE` — is there a toolchain that does total static resolution? (a manifest + a
  typecheck/`--dry` command that exits clean on the empty scaffold).
- `HAS_LOAD_GATE_BELOW_COMPILE` — if no compiler, is there *any* resolve-without-behavior command? probe
  in order: configured linter/typechecker → bare `import`/`require`/`parse` → `terraform validate` →
  `kubectl --dry-run` → `*-lint` → `nbconvert --execute` (shallow) → framework `build`. First hit wins.
- `STRONGEST_UNFINISH_SIGNAL` — capability-descending: compiler-proven typed throw → runtime raise →
  failing assert → nonzero-exit → reserved sentinel value → grep-token `TODO(autopilot:id)`.

It also records `verification_substrate` (harness | REPL | runtime | human | none) and
`realization_channel` (can effects escape the workspace?). These land in the bus header; stages read them.

---

## The three universal axes (apply across every ladder)

These are the lessons that survived stripping the original repo-specific design. They are *not* a stage —
they are properties the bus classifies (see `bus.md`) and every stage honors.

1. **Provenance-of-value.** A value crossing a serialization/computation boundary (magic, default, field
   name/tag, format, units, version) is **invent-by-default** unless a *resolvable upstream* pins it,
   independent of whether a consumer edge is visible. Read the project's *stated invariants* (prose
   conventions, sibling-stack values), not only its import graph. "No discoverable consumer ≠ no
   consumer." A false STOP costs one question; a silent wrong default on a wire value costs an
   irreversible external write.

2. **Effect-locality / oracle-purity.** If `verify(unit)` mutates or consumes external state across runs
   (a `plan` against live state, a one-way migration, a publish, a quota API), the convergence loop is
   **disabled** for that unit (cap-1, escalate on any non-GREEN) and idempotence needs a **realization
   receipt**, not a workspace fingerprint. This is a property of *oracle statefulness*, not metric noise —
   so it catches both IaC and ML.

3. **Shape-resolves ≠ shape-is-contracted.** A load-gate proves a name binds and a type is internally
   consistent; it never proves it is the *right* name/arity/return-shape, and there is no marker for an
   **absent** boundary. On surfaces where the signature *is* the product (library/SDK, public route table,
   schema, proto), an invented signature is irreversible-outward, not cosmetic — so use a contract anchor
   + a negative-space diff, and where no anchor exists, report `UNVERIFIED-COMPLETE`.

---

## Stage 2 — Skeleton-first ladder

> Invariant: shape precedes behavior; shape is independently checkable; unfinished is never silent.

- **R0 typed+compiled** — file-tree + full signatures + named failing test stubs; sentinel = typed throw
  the compiler proves reachable; load-gate = `compile`. Markers can't survive (tests RED until filled).
- **R1 typed, no harness yet** — keep typed signatures + typed throw; the test stub becomes a named
  assertion file authored to fail; load-gate = compile.
- **R2 untyped/dynamic** — function/class/route stubs whose body is `raise NotImplementedError('autopilot:id')`;
  load-gate = import/parse + linter/typechecker if configured; markers grep-counted.
- **R3 executable-but-opaque output** (SPA, mobile, ML, notebook) — component/screen/cell/resource tree
  with named exports + a visible `UNIMPL(id)` placeholder; load-gate = build/render-mounts; behavior is
  handed to the oracle/human, not claimed.
- **R4 non-executable structured** (IaC, openapi, proto, SQL migration, config) — declared-but-inert
  resource/endpoint/column set; sentinel = reserved placeholder value; load-gate = `validate`/`--dry-run`/
  `lint`, **never `apply`**; markers enforced by grep-guard.
- **R5 prose/asset** (doc, design asset, runbook) — heading/section/frame skeleton; sentinel =
  `TODO(autopilot:id)`; "load-gate" = structural lint (headings present, links resolve, frontmatter parses).
- **R6 honest floor** (snippet/shell glue with no boundary unit) — do **not** fake a scaffold; flat
  checklist of named steps with inline `# UNIMPL:id` + an external grep-guard; report that this rung was hit.

Known break: a pixel-aesthetic defect passes every gate R3 can erect (mounts ≠ looks right) — hand the
aesthetic acceptance entirely to Stage 5. Library/SDK at R0/R2 is the sharp one: the gate says the
signature *resolves*, never that it's the *contracted* signature — apply axis #3.

## Stage 1's blast classifier ladder (how reach/category degrade)

- **R0 declared contract** — manifest exports / versioned proto·openapi·migrations / SemVer'd package /
  typed signatures: diff the change against the declared surface; derive/invent split near-exact.
- **R1 typed-but-undeclared** — reachability via type+import graph: imported only within the subtree =
  local; referenced outside = external.
- **R2 untyped/scripting** — name-and-path heuristics + cross-module grep; default posture **conservative**
  (an ambiguous public-looking name is external until proven local).
- **R3 persistent-state-only** (ETL output, golden fixture, serialized model) — the *shape of the produced
  artifact* is the contract; in-place mutation of existing data is C3-destructive.
- **R4 side-effect-dominant, unobservable pre-commit** (Terraform/k8s apply, deploy, billing) — invert to
  **action-lexicon**: every verb crossing the process boundary (apply/create/destroy/charge/send/publish/
  push) is C5/C6 by default; only a dry-run/`plan`/`--check` affordance demotes to local-observable; absent
  that, STOP is the default.
- **R5 aesthetic / non-deterministic** — classify only the *structural* slice (changed prop, renamed
  route, deleted referenced asset, metric-threshold regression); hand the aesthetic core to Stage 5.
- **R6 honest floor** — no manifest/types/declared-output/dry-run: action-lexicon alone + the blunt rule
  *any write outside the working tree, deletion of existing content, or touch of a secret/credential/network
  is invent-by-default*; log line carries the "classified blind" caveat.

Known break the bus fixes (axes #1): on library/SDK and untyped stacks a wire-format/magic change has no
manifest export, no destructive verb, no in-tree consumer edge — yet is C1+C2. The provenance-of-value
rule forces it to invent/`open` regardless; "no discoverable consumer" routes to `reach=unknown → escalate`,
never to local.

## Stage 3 — Acceptance + invariants ladder

- **R0 tested+typed+observable** — native acceptance suite RED-before-code + a property/metamorphic/golden
  oracle authored by a different agent; typecheck+lint as free pre-gates.
- **R1 tested-untyped** — same RED loop; type pre-gate replaced by runtime assertions + lint; oracle leans
  on property tests (covers what types would have).
- **R2 property/smoke-only** (library/SDK, ETL, notebook, ML eval, shell) — executable contract/smoke
  (import + assert signatures call non-trivially; run on a tiny fixture + assert output shape/row-count/
  schema; ML metric crosses a pre-committed threshold; `--dry-run` exit 0). Oracle = conservation/
  metamorphic invariant.
- **R3 typecheck/lint-only** (typed change, no runner; most of an SPA) — compile/lint + build + render-smoke
  (mounts without throwing). The aesthetic fraction is explicitly **not** claimed verified.
- **R4 human-review checkpoint** (SPA visual, IaC that would mutate cloud, irreversible/outward) — produce
  artifact + captured diff (screenshot / `terraform plan` / migration DDL / rendered doc) + one named
  falsifiable question, STOP, route to a human. Standing up real verification here is itself high-blast.
- **R5 honest floor** — even a dry artifact/smoke is impossible: ship with a loud `UNVERIFIED` tag in STATE
  and top-of-report, naming exactly what was not checked, why, and the one command to check it.

Load-bearing fix (the third independence axis): split every oracle into **tautological** (round-trip,
idempotency, conservation — keep, but tag "shape-only: does NOT verify the transform is the *right*
transform") vs **ground-truth** (answer obtained *without re-reading the spec* — a human-supplied fixture,
a golden from a trusted run, a different-provenance computation). A rung is "verified" only with a
ground-truth oracle; tautological-only is **capped** at "shape-verified, semantics-UNVERIFIED" and routed
to a flagged question — the SPA R4 treatment, applied to ETL/ML/CLI semantics. Without this, a join keyed
wrong (but row-count-conserving) passes every metamorphic check green.

## Stage 4 — Convergence + cursor ladder

- **R0 executable behavioral oracle** — `cargo/go/pytest`, CLI golden stdin/stdout/exit-code, library API
  contract test. Full RED-before-code + bounded re-spawn; the independent invariants oracle runs here.
- **R1 static/type/schema oracle** — `tsc --noEmit`, `terraform validate`+read-only `plan`, `kubeval`,
  openapi-lint, `shellcheck`. GREEN = validates clean + no new diagnostics vs a pre-change baseline. Proves
  shape, not behavior.
- **R2 execution-smoke** — run safely on a sampled input (`bash -n`+dry-run, `nbconvert --execute`). GREEN =
  runs without raising + output shape matches a declared contract. Values not proven.
- **R3 structural/self-consistency** (SPA, cloud-mutating IaC, doc) — GREEN = every declared boundary
  exists, no surviving sentinels, imports/links resolve, snapshot if a baseline exists. Completeness-of-
  shape, not correctness.
- **R4 threshold-over-noise** (ML) — GREEN = metric clears a declared threshold; cap 1-2; record metric+seed;
  RED = below threshold, escalate fast. The loop's monotonicity assumption fails here — run once, checkpoint,
  hand optimization to a human.
- **R5 human-gate floor** — no machine oracle at any rung. Work is LANDED, terminal state `needs-human-verify`,
  surfaced top-of-report with the exact one-command check. Never silently GREEN.

Effect-locality guard (axis #2) overrides the rung: a `stateful-consuming` oracle (e.g. `terraform plan`
made a real RED/GREEN signal needs `apply`, which is forbidden) caps at 1 and records a **realization
receipt** (resource id / migration version / plan-hash); "already landed?" is answered by a read-only probe
of the *external* system, not the workspace fingerprint. IaC therefore terminates `needs-human-apply` with
`terraform apply` as the one human command — never a silent GREEN-on-parse.

## Stage 5 — Human-judgment ladder (with feedback edge)

For the aesthetic/prose/design/irreducibly-human slice, an authored oracle shares the impl's priors (same
training distribution grading its own homework), so it cannot be the independent oracle. The ladder is a
single rung with a **feedback edge**, not a dead end: structured handoff (artifact bundle + one named
yes/no question per high-blast slice, blast-ranked) → human verdict re-enters the convergence loop as
RED/GREEN → a "no" re-spawns the owning agent with the note, bounded human-iteration cap. For prose, add
**content-grounding**: every factual claim referencing the code is grep-verified; unverifiable claims are
flagged.

## Cross-stage seams the bus must enforce (see bus.md)

- **Regime → blast posture.** Scale/regime says "greenfield → invent-dominant → silent-default," but
  greenfield is where C1/C2/C4 foundational commitments are *least* reversible. The bus's
  greenfield-foundational override forces those to `open` off **per-axis precedent**, not the coarse
  novelty bit.
- **Skeleton → acceptance.** The scaffold enumerates from one reading of the prompt; the acceptance oracle
  authors its prediction from the *same* reading and tests only enumerated boundaries — so a misread or an
  omitted surface becomes "verified." This is why Stage 0 (premise gate) and the contract-anchor +
  negative-space check (axis #3) exist, and why a ground-truth oracle (Stage 3 fix) is required before a
  rung counts as verified.
- **Acceptance-green → convergence-monotone.** A stateful or tautological oracle makes the loop climb
  toward a decaying or self-confirming green; the effect-locality guard + the ground-truth requirement
  break it.

The deepest blind spot all stages share is **premise/goal validity** — a flawless build of the *wrong*
feature passes every gate. That is why the pipeline starts with the premise gate, the one place the
aggressive default is overridden.
