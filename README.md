# skills

My custom [Claude Code](https://claude.com/claude-code) skills.

Each skill is a self-contained directory with a `SKILL.md` (the entry point Claude
loads) plus optional `references/` (docs loaded on demand) and `evals/` (test
prompts). They are hand-built, iterated against real evals, and kept here as the
source of truth — separate from the live `~/.claude/skills/` install location.

## What goes here

Skills that are **general-purpose and reusable across projects** — not tied to one
codebase. Project-specific skills live with their project; portable ones live here.

## Install a skill

Copy (or symlink) a skill directory into your Claude Code skills folder:

```bash
cp -r autopilot ~/.claude/skills/autopilot
# or, to keep it in sync with this repo:
ln -s "$(pwd)/autopilot" ~/.claude/skills/autopilot
```

Claude Code picks it up automatically; the skill's `description` decides when it
triggers.

## Skills

| Skill | What it does |
|-------|--------------|
| [`autopilot`](autopilot/) | Universal autonomous feature/project builder — turns ~one prompt into a whole working, tested feature without asking questions, refusing to build the wrong/redundant thing up front (premise gate) and surfacing the irreversible decisions + honest verification claims at the end. Validated against evals (see the skill's `evals/`). |

## Notes

Skills here are developed and maintained primarily through AI assistance — see
[`CLAUDE.md`](CLAUDE.md) for how that work is calibrated.
