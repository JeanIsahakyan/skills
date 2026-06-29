---
name: spec
description: >-
  Turn a fuzzy product idea into a precise, written feature spec / PRD BEFORE any code — by interviewing
  the user with plain-language PRODUCT questions only and deriving every technical decision automatically
  from the existing codebase (never asking the user a tech question). The output is a single SPEC.md with
  a hard wall: narrative product behaviour on top (reads like a PM wrote it), a derived technical appendix
  sealed at the bottom (reads like an engineering handoff). Use this whenever the user wants to spec a
  feature, write a PRD, formalize requirements, design a flow, document how a behaviour should work, or
  turn a vague ask into a contract before implementation: "spec this", "write a spec for X", "design the
  X flow", "PRD for X", "how should this feature work", "let's spec out X", "formalize requirements for
  Y". Runs a codebase-context agent (reads the repo) + a product-interviewer loop (open-ended questions
  until coverage converges) + a spec-writer agent. Best for any non-trivial feature where the product
  intent is fuzzy but the stack is already established.
---

# spec

Author a feature/product spec by deeply interviewing the user about **how the feature should behave** —
and never about how it should be built. Every technical decision is inferred from the existing codebase
and lives in a separate appendix at the bottom of the document. The spec is the **product source of
truth**, not an implementation plan.

This skill is feature- and stack-agnostic: it reads whatever repo it's in, interviews in product
language, and writes a spec that follows the repo's existing conventions in its technical half.

## Core principles

1. **Product questions only.** Never ask the user about databases, frameworks, libraries, file paths,
   types, naming, indexes, schemas, queues, caches, transports, deployment, or anything else technical. If
   the answer can be derived from reading the repo — derive it. If it cannot, it is still a product
   question, not a tech one.
2. **Codebase is the source of truth for tech.** Stack, patterns, conventions, similar features, naming
   style, error model, validation library — all read from the repo, never asked.
3. **Open-ended on ambiguity.** When something the user says is ambiguous, fuzzy, or could go multiple
   ways, ask **open-ended** clarifying questions in plain product language. Do not present technical
   multiple-choice. Adapt the spec only after the user answers.
4. **Convergence loop.** Keep asking until there are no open product questions left. The user signals
   convergence by stopping correcting; you signal it by writing the spec.
5. **Universal.** Works for tooling, content systems, payment flows, onboarding, search, identity,
   anything. It never assumes a domain.
6. **Output shape is fixed.** See "Output structure" — feature description on top, technical appendix
   strictly at the bottom under a single dedicated heading.

## Flow

The skill runs in four phases, using **at least three subagents** for work that benefits from isolation
and parallelism. The user-facing conversation stays in the main thread.

### Phase 1 — Capture intent (main thread)

Ask the user one open question: *"What feature would you like to spec? Describe it in plain product terms
— what should it let people do, and why?"*

Do not move on until you have:
- A working name for the feature (a slug, e.g. `undo-action`)
- A one-paragraph description of what it does and who it's for
- The trigger that brought it up (new product ask? bug? user feedback? leadership ask?)

If any of these are missing or fuzzy, ask one open follow-up. Never list options.

Pick a slug and an output dir: `--out <dir>` if given, else `specs/<slug>/`. Announce it: *"I'll write the
spec to `<out>/SPEC.md`."* Create the directory. (Companion audit files `_context.md` and `_interview.md`
also live there.)

### Phase 2 — Codebase context (parallel, background)

Spawn the **codebase-context-agent** in the background (see `agents/codebase-context-agent.md`), passing
it the output dir. It reads the repo and produces `<out>/_context.md` containing:
- Tech stack actually in use
- Naming and file-organization conventions
- Validation/error/auth/storage patterns
- Closest existing features and where they live
- Any constraints the spec must respect (package boundaries, generated clients, migration policy, CI gates)

This runs while you interview the user. The user never sees these technical details unless you weave them
into a *product* question (e.g. "we already have a media-upload flow elsewhere — should this reuse the
same picker UX, or is it a different surface?" — phrased as product, not tech).

### Phase 3 — Product interview (main thread, driven by an interviewer agent)

Spawn the **product-interviewer-agent** (see `agents/product-interviewer-agent.md`) to **propose** the
next batch of deep product questions, given `_context.md`, the user's intent, and answers so far. The
agent returns a small, ordered batch of open-ended questions. You then ask them in the main thread one at
a time (or in tight clusters where natural).

Question shape — the style required:

> ✅ "When two users trigger the same action at the same moment, who sees the confirmation first?"
> ✅ "If a user removes someone they were previously linked with, what happens to the existing thread of
>    state — disappear, archive, become read-only?"
> ✅ "Should the undo affordance work after the counterparty has already acted, or is it locked the moment
>    a binding outcome would form?"
> ❌ "Should we use the relational DB or the cache for the undo state?" (technical — never ask)
> ❌ "Should the undo endpoint be POST or DELETE?" (technical — never ask)
> ❌ "What's the TTL on the undo cache?" (technical — never ask)

**Open-ended over multiple-choice.** When something is uncertain, prefer free-form questions.
Multiple-choice is only acceptable when the user has already implied a small finite set; otherwise the
user's words shape the spec, not yours.

After each user answer:
- Append the answer to a running notes file `<out>/_interview.md`
- Re-invoke the interviewer agent with the updated state to get the next batch
- Stop when the agent returns an empty batch (no more product ambiguity)

The interviewer is the only agent that decides when interviewing is done. It must justify "done" by
emitting a coverage checklist (actors, golden path, edge cases, failure modes, invariants, success
criteria — all addressed). Do not skip this.

### Phase 4 — Spec assembly (spec-writer agent)

Spawn the **spec-writer-agent** (see `agents/spec-writer-agent.md`) with three inputs: `_interview.md`,
`_context.md`, and the user's original intent paragraph. It writes the final `<out>/SPEC.md` following the
**Output structure** below.

Present it to the user with one question: *"Anything off? I can refine any section, or we can ship it
as-is."* If they ask for changes, route back through Phase 3 with a narrowed scope, then re-run the
writer. If they say it's good, stop. Keep `_context.md` and `_interview.md` as audit trails, but make sure
`SPEC.md` reads standalone.

## Output structure

The generated `SPEC.md` always follows this shape. The order matters; technical content is **strictly at
the bottom**, under a single `## Technical appendix` heading, preceded by the warning blockquote. This is
non-negotiable — it makes the spec readable to non-engineers and machine-extractable for downstream tools.

```
---
title: '<Feature name>'
domain: <inferred from the codebase area this touches>
last_updated: <YYYY-MM-DD>
---

# <Feature name>

## Purpose
One paragraph: who this is for, what it lets them do, why now.

## Glossary
Terms used in this spec, defined in product language.

## Actors
Every human, machine, or external system the feature touches, with one-line roles.

## User-facing behavior
The product story end-to-end. Subsections per surface (web, mobile, plugin) when relevant. No code, no APIs.

## Flows
Each named flow as a numbered narrative — golden path first, then edge cases.

## States and transitions
Every meaningful state the feature can be in, and what causes a transition.
Written in product language ("active" → "archived"), not DB language.

## Invariants
Bullet list of things that must always be true, in product terms.
("A user never appears in their own recommendation list.")

## Edge cases
Discovered during the interview. One bullet per case, with the resolved behavior.

## Success criteria
How we'll know it works — measurable in product terms (completions per session,
time-to-first-action, drop-off after step N, etc.).

## Out of scope
Things explicitly not in this spec, with a one-line reason.

## Open questions
Anything the user could not yet answer. Empty if the interview converged.

---

## Technical appendix
> Everything below this line is derived from the codebase, not from the user.
> The product description above stands on its own. Treat this section as
> implementation guidance that follows existing conventions — it can change
> without changing the product spec.

### Stack & placement
Where in the codebase this lives, which packages/modules it touches, which existing
code it reuses.

### Data model
Entities and fields in the existing ORM/style. Migrations needed.

### API surface
Endpoints/methods, named in the existing convention. Note any client/codegen
regeneration step if the contract changes.

### Integration points
Reused services (auth guard, geo service, loader pattern, client, etc.) named
exactly as they appear in the repo.

### Risks & follow-ups
Things the codebase context surfaced that the product spec should be aware of.
```

## Subagent roster

| Agent | File | Role |
|-------|------|------|
| Codebase context agent | `agents/codebase-context-agent.md` | Reads the repo and produces `_context.md` |
| Product interviewer agent | `agents/product-interviewer-agent.md` | Proposes the next batch of open-ended product questions and signals when interviewing is done |
| Spec writer agent | `agents/spec-writer-agent.md` | Assembles the final `SPEC.md` with the fixed structure |

Spawn via the Agent tool. The codebase-context-agent runs in the background during Phase 2; the other two
run synchronously when their phase is reached.

## Style rules for the interview

- One topic per question.
- Use the user's words, not yours. If they said "ghost", don't translate to "soft delete".
- Never present a technical tradeoff. If two product paths exist, name them in product terms ("instant
  chat vs. delayed reveal") and ask which feels right.
- Keep it short. State the question, optionally give one example, stop.
- Reflect uncertainty back. If an answer is fuzzy, say so and ask the smallest clarifying question.

## Stop conditions

Stop the interview when **all** hold: the interviewer agent returns an empty next-batch; every actor named
in Phase 1 appears in at least one flow; every flow has both a golden path and at least one edge case; the
user has not introduced a new concept in the last two turns.

Stop the whole skill when the user accepts the written `SPEC.md`. Iterate Phase 3+4 otherwise.

## What this skill never does

- Asks the user technical questions.
- Writes code.
- Picks a tech stack — only describes the one already in the repo.
- Skips the technical appendix — it's always present, even if minimal.
- Mixes product and technical content in the same section.

## Args supported

- `<idea>` / free-form intent — optional; if absent, Phase 1 asks for it.
- `--out <dir>` — output dir (default `specs/<slug>/`).

## Deliverable

`<out>/SPEC.md` — product source of truth on top, derived technical appendix at the bottom — plus the
`_context.md` and `_interview.md` audit trails. That file is the output; if you're in a git repo and want
a record, commit it — but that's your call, not something this skill does for you.
