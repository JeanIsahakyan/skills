# Shared classification bus

The bus exists to close a propagating seam: without it, each stage re-derives its own notion of
blast/reach/posture, the notions disagree, and **one wrong posture bit at the probe stage propagates
through three stages and silently buries the run's most consequential inventions** — each downstream
stage "correctly" trusting wrong upstream input. The bus makes **one probe classify once**,
authoritatively, and makes "every stage used the *same* classification" a **checked fact, not an
assertion.**

The single sharpest idea: **the seal is a hypothesis, not a verdict.** Classification is re-derived
against the *realized artifact*, and upward blast-drift can never go GREEN.

## Table of contents

1. Files & identity
2. Record schema (`BUS.json`)
3. Canonical category enum
4. `DecisionRow`, `EvidenceRef`, `LocusRef`, keying
5. Write protocol (single-writer-once, append-only-immutable-prefix, `classify()`)
6. Read protocol + checked-fact reconciliation
7. Desync detection (three nets)
8. Staleness / mid-run mutation rule

---

## 1. Files & identity

- `.planning/autopilot/<run_id>/BUS.json` — the one authoritative record. Written once by the
  orchestrator (the bus probe), then **read-only**. Classification axes live here and **nowhere else** —
  there is no second store a stage could put a category in.
- `.planning/autopilot/<run_id>/BUS-OBSERVATIONS.jsonl` — append-only: realization receipts,
  reclassify-on-realization checks, human verdicts. Never mutates `BUS.json`.
- `.planning/autopilot/<run_id>/RELIANCE.jsonl` — append-only: per-stage echoes of *what each stage
  actually did* (the realized-effect fingerprint), the input to reconciliation.

If the premise gate STOPped, `BUS.json` is written with `premise.verdict="stop"`, `decisions=[]`,
`sealed=true`, and stages do not run. Absence of `BUS.json` means "do not proceed."

## 2. Record schema (`BUS.json`)

```
{
  schema_version: 1
  run_id: string
  sealed: bool                 // readers MUST refuse a record with sealed=false
  sealed_at: iso8601
  classifier_id: "<model>@<probe-version>#<ruleset-hash>"
                               // ruleset-hash is over rules{} below → "same classifier" == "same rules + same enum".
                               // computed, never agent-supplied free text.
  prefix_seal: { length:int, hash:string }   // hash over canonicalized decisions[0..length-1]; advances only on append
  record_hash: string          // hash over {regime, defaults, rules, category_enum, decisions[], prefix_seal};
                               // EVERY reader recomputes at read time and at every terminal (eager, not lazy)
  premise: {
    verdict: "proceed" | "stop"
    stop_class: "already-satisfied" | "self-contradictory" | "false-premise" | null
    statement: string          // one-line restatement of the ask
    evidence_ref: EvidenceRef | null
  }
  bound_stack: {               // the DISCOVERED substrate — consequence words, never mechanism words
    deliverable_kind: "code" | "config" | "schema" | "doc" | "design-asset" | "data-artifact" | "mixed"
    surface: [string]          // freeform discovered tags (["backend","typed"],["spa"],["iac"],…) — informational; no rule keys off exact values
    realization_channel: "workspace-only" | "external-capable" | "unknown"   // can effects escape the workspace? → effect_locality default
    verification_substrate: "present" | "absent" | "unknown"                 // ANY way to check behavior (harness | REPL | runtime | human | none) — NOT "tests"
  }
  regime: {
    novelty: "greenfield" | "brownfield" | "new-subtree"      // DESCRIPTIVE label only
    precedent_density: "none" | "sparse" | "dense"
    reversibility_default: "reversible" | "irreversible" | "mixed"           // greenfield does NOT imply reversible
    precedence: [ PrecedenceFact ]   // per-axis precedent findings — what the foundational override actually keys on
  }
  defaults: {
    rung_floor: "silent-default" | "flag" | "open"   // minimum rung on EVERY decision before per-decision escalation
    respawn_policy: "allowed" | "cap-1"              // global convergence ceiling; per-decision oracle_purity can tighten
  }
  rules: RulesDecl             // the FULL override table, INLINED and under record_hash → the validator RECOMPUTES rung, never hand-mirrors a subset
  category_enum: CategoryEnumDecl
  decisions: [ DecisionRow ]   // append-only; the heart of the record
}

PrecedenceFact = {
  axis: "public-contract" | "persistent-format" | "trust-boundary"   // the foundational axes (C1/C2/C4)
  precedent: "exists" | "absent" | "unknown"                         // is there ANY prior commitment on THIS axis in the blast radius?
  evidence_ref: EvidenceRef                                          // a found-precedent locus, or kind="absence" proving none
}
```

`PrecedenceFact` is *per-axis* on purpose: a new-subtree that introduces the **first** wire format gets
`persistent-format: absent` even though `novelty=new-subtree`. The greenfield-foundational override
fires off `precedent==absent`, **never off the coarse `novelty` bit.**

```
RulesDecl = {
  version: int
  rung_order: ["silent-default","flag","open"]      // total order for max()
  conf_floor: { "<CategoryCode>": float }           // per-category confidence floor; under the hash so the validator recomputes the low-confidence raise
  override_table: [ OverrideRule ]                  // ordered; each can only RAISE rung
}
OverrideRule = { id:string, when:string, raise_to:"flag"|"open" }   // `when` is a closed-vocabulary predicate over a DecisionRow + regime.precedence
```

Canonical `override_table` (each RAISES only — you may tighten posture, never loosen):

| id | when | raise_to |
|---|---|---|
| `reach-unknown` | `reach == "unknown"` | `open` |
| `provenance-invented` | `provenance == "invented" && crosses_boundary` | `open` |
| `blast-foundational` | `category in {C1,C2,C4}` | `flag` |
| `greenfield-foundational` | `category in {C1,C2,C4} && precedence_for(category) == "absent"` | `open` |
| `blast-destructive-outward` | `category in {C3,C5,C6}` | `flag` |
| `low-confidence` | `confidence < conf_floor[category]` | `flag` |

## 3. Canonical category enum

The bus **owns** this enum (inlined + versioned as `category_enum`, with a `blast_rank` per member so
disclosure sorts by a stable key, no hardcoded order anywhere else):

```
C1  public-contract       interface/name/signature others depend on
C2  persistent-format     serialization/format/version read back by a different version or process
C3  destructive           deletes or in-place-mutates existing data/state
C4  trust-boundary        auth/crypto/secret/permission/security
C5  outward-effect        send/publish/notify/deploy — leaves the workspace, observable outside
C6  real-world-mutation   apply/create/charge against a live external system (cloud, payment, infra)
LOW internal-cosmetic     internal-only, no cross-boundary or external consequence
```

## 4. `DecisionRow`, evidence, keying

```
DecisionRow {              // universal — NO field names a type system, test, compiler, or binlog
  decision_key: "<intent_slug>@<anchor>"   // STABLE identity; path is NOT in the key (move/split-safe)
  title: string
  intent_slug: <closed set: "error-format","auth-check","delete-policy","publish-target","field-name",
                "default-value","wire-magic","format","units","version", …>
  locus: LocusRef          // WHERE it lives NOW — a mutable attribute, not identity
  category: CategoryCode   // the SINGLE blast classification (highest applicable class)
  reach: "local" | "external" | "unknown"        // unknown → conservative escalation, never local
  effect_locality: "workspace-local" | "external"
  oracle_purity: "idempotent-rereadable" | "stateful-consuming"   // stateful → convergence cap-1
  provenance: "invented" | "borrowed"            // borrowed ONLY on proof of a resolvable upstream
  provenance_boundary: "magic"|"default"|"field-name-or-tag"|"format"|"units"|"version" | null
  confidence: float        // 0..1
  evidence: [ EvidenceRef ]   // pointers (NOT prose); also carry the fingerprints staleness keys on
  rung: "silent-default" | "flag" | "open"       // AFTER rules; recomputable by the validator
  verdict: "silent-default" | "flag" | "open" | "STOP"   // STOP only from a premise per-decision contradiction
  overrides_applied: [string]   // rule ids that fired (validator recomputes this set and asserts equality)
  owning_stages: [string]       // which stages MUST surface/realize this (drives silent-burial + orphan checks)
  status: "hypothesis" | "frozen" | "dirty" | "resolved"
  computed_at_cursor: int       // convergence-cursor ordinal at last (re)probe
  epoch: int                    // bumped each recompute wave
}
```

Rows carry **no** `classifier_id` field — the single top-level `classifier_id` governs every row, so a
second classifier is *structurally impossible* (nowhere to put it).

```
EvidenceRef {
  kind: "locus" | "external-citation" | "absence" | "prior-axis" | "human"
  selector: string        // locus: "path::anchor"; citation: url/id; absence: the negative-space query
                          // ("consumer-of:job.fetch"); prior-axis: "<other_key>.reach"; human: a verdict id
  polarity: "present" | "absent"    // absent = "I decided X BECAUSE nothing was found" — the dangerous, must-re-examine case
  derivation: "content-hash" | "semantic-projection" | "query-result" | "external-stamp"
  projection: string | null   // when derivation=semantic-projection, the named projection (e.g. "crc32-of-normalized-source");
                              // the staleness gate re-applies THIS, not raw bytes
  fingerprint: string         // re-derived at gate time, never trusted from cache
  note: string                // <=140 chars, optional
}
LocusRef { path: string, anchor: string }   // anchor = symbol/cell-id/json-pointer/line-range/heading/resource-id
```

`kind="absence"` is first-class: "I searched for a consumer and found none" is **evidence for**
invent-by-default, not against it — and it force-re-examines the moment a consumer lands.

**Keying.** `decision_key = "<intent_slug>@<anchor>"`. Path is excluded so a re-spawn that moves/splits
a file keeps the same key. Stages may only **look up** keys; minting a new key is a `PHANTOM-CLASSIFICATION`
fault. Two rows with the same key is a probe error and fails validation.

## 5. Write protocol

**Who.** Exactly one writer — the orchestrator — holds the write convention for `BUS.json` (a
reconciler-checked convention, not an OS capability). The probe and all stages are **readers**. A stage
that discovers a new decision does **not** write the bus; it emits a CLAIM carrying **raw evidence**
(paths, the actual minted value, a grep/diff result) — **never a pre-judged boolean.** The orchestrator
runs `classify()` over that evidence and appends the authoritative row.

**When.** (1) premise gate first; on STOP write the stopped record and stop. (2) On proceed, write
header (`bound_stack`, `regime`, `rules`, `category_enum`) + the first wave of decisions discoverable
from the contract-anchor/skeleton; `sealed=true`; compute `prefix_seal` + `record_hash`. (3) `decisions[]`
then **grows append-only** as later stages surface mid-implementation decisions.

**`classify(evidence, header)` — deterministic, one-pass, predicates evaluated by the bus over evidence
(never trusting an agent-asserted boolean):**

```
category        = highest_applicable_blast_class(evidence)
provenance      = "borrowed" IFF evidence carries a verifiable adopted-source ref (a path::anchor that
                  exists and pins this value); ELSE "invented"     // unproven adopt → invented → loud
provenance_boundary = (provenance=="invented") ? which_seam(...) : null
reach           = "local" IFF evidence proves in-workspace containment; "external" IFF an escape is observed; ELSE "unknown"
effect_locality = (realization_channel=="workspace-only") ? "workspace-local"
                  : observed_escape(evidence) ? "external" : "workspace-local"
oracle_purity   = consumes_or_mutates_on_read(evidence) ? "stateful-consuming" : "idempotent-rereadable"
rung            = defaults.rung_floor ; for rule in rules.override_table (in order):
                     if eval(rule.when, row, regime.precedence): rung = max(rung, rule.raise_to)
verdict         = premise_contradiction(row) ? "STOP" : rung
status          = "hypothesis"     // a sealed row classifies INTENT; a hypothesis until convergence realizes it
```

**Append (escalate-only).** Before any append, re-derive `prefix_seal.hash` over the existing prefix and
compare; a mismatch means a stage mutated a sealed row → `ABORT-with-map`. An existing key may only be
**superseded** by a new row that *raises* risk (`supersedes:<key>`); equal-or-lower risk is a NOOP. After
append, advance `prefix_seal` + recompute `record_hash`. Sealed axes are never rewritten in place.

**Validator invariants** (recomputed, not hand-mirrored): re-run the full override table from
`{category,reach,provenance,confidence,precedence}` and assert (a) computed rung == stored rung; (b)
computed overrides == `overrides_applied`; (c) `invented ⇒ provenance_boundary != null`; (d) `reach=="unknown"
⇒ rung=="open"`; (e) `category∈{C1,C2,C4} && precedence_for(category)=="absent" ⇒ rung=="open"`; (f) every
rung ≥ `rung_floor`; (g) keys unique + `intent_slug` in the closed set; (h) `prefix_seal` + `record_hash`
re-derive. A probe regression that drops any raise **fails here** because the rule lives in the hashed
record and is recomputed, not trusted.

## 6. Read protocol + checked-fact reconciliation

**Principle.** Write-once / read-many. No stage computes a classification; it **looks up**
`bus.decisions[key]` and **obeys.** "Same classifier" reduces to "same bytes" (`record_hash`); "verdict
matches what shipped" reduces to "**realized-effect fingerprint** matches the classified property."

Read rules: **R0** (eager integrity) — on stage entry **and** exit, and at every terminal (GREEN
included), re-derive `record_hash` + `prefix_seal`; a flip between reads is a `WRITE-VIOLATION` hard
fault. **R1** read-only. **R2** act only via a key, never re-classify. **R3** a needed-but-absent key is
*not* defaulted locally — emit `category="__ABSENT__"` → reconciler raises `COVERAGE-GAP` (re-probe).

**Reliance ledger** (`RELIANCE.jsonl`) — each stage, for each decision it acts on, appends the
**realized-effect fingerprint** (not a copy of the row): for a wire-magic decision, the magic value the
stage *actually emitted* (the projection over the *written* artifact); for an outward effect, the
realization receipt. Plus `relied_posture/category`, `realization_evidence`, `action_taken`,
`record_hash_seen`, `classifier_id_seen`.

**Reconciler** (after each stage and as a final gate; any fault → hard stop, non-zero exit, terminal in
{ESCALATE, ABORTED-with-map}; **never warn-and-continue**):

- **F1 PHANTOM-CLASSIFICATION** — acted on a key the bus never owns.
- **F2 CLASSIFIER-SPLIT** — `record_hash_seen`/`classifier_id_seen` ≠ current (read a forked/stale bus).
- **F3 REALIZATION-DRIFT** — re-run `classify_blast` over the realized artifact/receipt; if realized
  `blast_rank` > sealed, or a `borrowed` row realizes with no provable upstream → drift; **upward drift
  never GREENs.**
- **F4 POSTURE-VIOLATION** — behavior inconsistent with the posture read (see the `action_consistent`
  table below).
- **F5 SILENT-BURIAL** — a `flag`/`open` decision no owning stage ever surfaced.
- **F6 ORPHAN-LOCUS** — a convergence-touched locus that maps to no decision key (fail-closed, never
  skeleton-silent).

`action_consistent(action, row)` — universal table: `silent-default` allowed **only if** `category==LOW
&& reach==local && effect_locality==workspace-local && provenance!=invented`; `stateful-consuming` ⇒
action is `cap-1-*` with a realization receipt (a `respawn`/`retry` tag is a violation); `invented` ⇒
`rung∈{flag,open}` and `action != silent-defaulted`; `C1/C2/C4 with precedence absent` ⇒ `rung∈{flag,open}`.
There is no fourth path: act-without-declare trips F5/F6; declare-without-matching-realized-effect trips
F3; declare-with-wrong-verdict trips F2/F4. A stage passes only by reading the verdict verbatim **and**
shipping an effect whose fingerprint the bus's own projection reproduces.

## 7. Desync detection (three independent nets, any one fires hard)

1. **Within-run** (two stages holding different categories at once) — impossible by construction (one
   writer, one `classifier_id`, one store, sealed + prefix-hashed). Caught operationally by F2 and by R0
   eager hash checks — a flip with zero further appends is still caught, unlike a lazy pre-append check.
2. **Seal-vs-reality** (the seal is byte-honest but the classification was wrong, or the artifact drifted
   after convergence wrote into the classified loci) — caught by **reclassify-on-realization**: the seal
   is a **hypothesis**; when convergence reports a key realized, re-run `classify_blast` against the
   *written* locus/receipt (not a copy of the row) and append a realization-check. Upward `blast_rank`
   forces ESCALATE, never GREEN. This re-introduces a two-derivation comparison (intent vs realized)
   **without un-sealing** `BUS.json`. It is the net that catches a magic routed through codegen — the
   realized header's actual magic is reclassified even when the landing step wrote a different file than
   the row's evidence names.
3. **Effect-mismatch** (stage claims a posture but shipped otherwise) — caught by F3 + F4: the reliance
   echo is the realized fingerprint, so a stage that silently-defaults an invented C2 value cannot
   produce a matching fingerprint, and a stage that re-spawns a stateful effect cannot produce a single
   receipt.

Every fault names the stage, the `decision_key`, and the diverged axis.

## 8. Staleness / mid-run mutation rule

The bus is computed once, but the workspace mutates as code lands (a landed file creates a new consumer
edge; a landed symbol changes reach; an applied migration changes effect_locality). A row's
classification is valid exactly as long as the **evidence it was derived from still projects to the same
fingerprint** — and the authority for "stale?" is a **re-derive of the evidence NOW**, never a trusted
footprint of the last landed change (footprint-intersection is demoted to a *prioritization hint* for
ordering the dirty queue).

**Evidence fingerprints are semantic projections**, which is what makes this both correct and cheap:

- `content-hash` — hash of a raw slice (cosmetic text).
- `semantic-projection` — the value that crosses the boundary, e.g. `projection="crc32-of-normalized-source"`
  ⇒ the fingerprint is the **magic**, not the bytes. A cosmetic edit that doesn't move the magic
  re-freezes for free; a codegen-routed magic change is caught because the projection is recomputed from
  source regardless of which file the landing step wrote.
- `query-result` — hash of a negative-space query result; the encoding of "no discoverable consumer ≠ no
  consumer."
- `external-stamp` — a probe-time stamp, marked **never-trust-frozen** (external evidence is force-stale
  across any landed change).

**Gate** (`validate-bus`, runs at every convergence-cursor advance, *before* the next stage reads):

```
for row in decisions:
  if row.status=="resolved" && row.oracle_purity=="stateful-consuming":
     # a consumed stateful oracle CANNOT be re-probed (re-reading consumes again / the effect already fired)
     if any evidence fingerprint re-derives != stored:
        freeze hard, reason "stateful-consumed; touched post-realization"; escalate to HUMAN oracle; continue
  stale = false
  for ev in row.evidence:
     if ev.polarity=="absent": stale ||= could_satisfy(ev.selector, current_workspace)   # an INVENT made under "no consumer" re-opens the moment a consumer lands
     else:                     stale ||= (re_derive(ev) != ev.fingerprint)               # re-read/re-project NOW; do not trust the cached fingerprint
  if stale: mark_dirty(row)
propagate_dirty_via_prior_axis_edges()    # transitive: B with evidence prior-axis:A dirties when A dirties
if dirty_queue non-empty: FAIL_HARD("stage tried to read bus with dirty entries — re-probe required")
```

**Re-probe** (bounded; only dirty rows; same `classify()`): if axes + fingerprints are unchanged →
re-freeze quietly (`epoch++`). Otherwise append a supersedes-row if risk rose (escalate-only); **a frozen
`silent-default` that re-probes to `flag`/`open` MUST resurface in disclosure salience** (a now-load-bearing
contract cannot stay buried).

Both extremes avoided: **never-recompute** is avoided because the gate re-derives every row's evidence at
every cursor advance and absent/external evidence is force-stale; **recompute-everything** is avoided
because re-deriving a fingerprint is cheap and the expensive `classify()` runs only when a fingerprint
actually differs. Cost is O(evidence) hashing per gate — negligible against agent-spawn cost.
