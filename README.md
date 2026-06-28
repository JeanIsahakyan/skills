# skills

My custom skills for AI coding agents.

Each skill is a self-contained directory with a `SKILL.md` (the entry point the agent
loads) plus optional `references/` (docs loaded on demand) and `evals/` (test
prompts). They are hand-built, iterated against real evals, and kept here as the
source of truth — separate from the live install location.

The skill format and the `~/.claude/skills/` install path below are
[Claude Code](https://claude.com/claude-code)'s, but the skills themselves are written
to be agent- and stack-agnostic in spirit.

## What goes here

Skills that are **general-purpose and reusable across projects** — not tied to one
codebase. Project-specific skills live with their project; portable ones live here.

## Install

### As a plugin (recommended)

This repo is a Claude Code plugin marketplace. Install everything in one go:

```
/plugin marketplace add JeanIsahakyan/skills
/plugin install ji-skills@jeanisahakyan-skills
```

That registers the marketplace and installs the `ji-skills` plugin (which bundles all
the skills under `skills/`). Updates land with `/plugin marketplace update jeanisahakyan-skills`.

### Manually (single skill)

Copy (or symlink) one skill directory into your agent's skills folder
(`~/.claude/skills/` for Claude Code):

```bash
cp -r skills/autopilot ~/.claude/skills/autopilot
# or, to keep it in sync with this repo:
ln -s "$(pwd)/skills/autopilot" ~/.claude/skills/autopilot
```

The agent picks it up automatically; the skill's `description` decides when it
triggers.

## Skills

| Skill | What it does |
|-------|--------------|
| [`autopilot`](skills/autopilot/) | Universal autonomous feature/project builder — turns ~one prompt into a whole working, tested feature without asking questions, refusing to build the wrong/redundant thing up front (premise gate) and surfacing the irreversible decisions + honest verification claims at the end. Validated against evals (see the skill's `evals/`). |

## Notes

Skills here are developed and maintained primarily through AI assistance — see
[`CLAUDE.md`](CLAUDE.md) for how that work is calibrated.
