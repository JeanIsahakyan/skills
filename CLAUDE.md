# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What this repo is

A collection of **custom, portable Claude Code skills**. Each skill is a directory
with a `SKILL.md` entry point, optional `references/` (progressive-disclosure docs)
and `evals/` (test prompts). The source of truth lives here; skills are installed by
copying or symlinking into `~/.claude/skills/`.

### Skill conventions

- One skill per top-level directory. `SKILL.md` carries YAML frontmatter (`name`,
  `description`) — the `description` is the trigger and should be concrete and a
  little "pushy" about when to use it.
- Keep `SKILL.md` lean (aim < 500 lines). Push heavy reference material into
  `references/*.md` and point to it, so it loads only when needed.
- Every non-trivial skill should be **iterated against real evals** (with-skill vs a
  no-skill baseline), not just written once. A skill that ties or loses to a bare
  competent agent isn't pulling its weight — find the case where it wins and make
  that the reason it exists.

## Working on this repo as an AI

This repository is developed **entirely through AI assistance**. There is no separate
human implementation track. A few conventions calibrate that work.

### Time estimates

**Don't translate work into human-day scales.** "A week", "2-3 months", "6 months"
anchor against human engineering pace — the wrong frame when the work is done by an
AI inside a conversation.

- For work the AI will write itself: estimate by **scope** (lines, files, skills,
  integration points) and **session count**, not wall-clock.
- "Adding a skill — a day or two" is misleading. Say "in one session" or "in two
  sessions", or just describe the scope.
- For work the **user** will do (review, decisions, installing/testing a skill):
  wall-clock estimates are fine — bounded by human availability.

### Session capacity

Practical baselines per session on a 1M-token context window: a new skill from
scratch (SKILL.md + references + evals + an iteration loop) fits comfortably in one
session; a major redesign across several skills may take two. Trigger for a fresh
session: the user wants to pause, a large pivot, or context past ~70% with sizeable
work remaining.

### Communication style

- Respond in **Russian by default**; keep technical terms as-is (skill, eval,
  prompt, frontmatter, PR, CI, token).
- Skip social niceties ("great question", "hope this helps", "great point").
- Skip softening hedges when actually confident — "do X", not "maybe consider X".
- Disagree explicitly when disagreeing; don't soften with "interesting approach,
  but…". Don't oversell your own artifact — if a skill mostly ties a baseline, say so.
- Say "I don't know" without padding.
- Treat the user as a senior peer, not someone to comfort.
- Direct ≠ rude — directness is about removing performance, not about being harsh.
