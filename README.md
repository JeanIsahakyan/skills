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

**This repo *is* the marketplace.** There's nothing to upload anywhere — a Claude Code
"marketplace" is just a git repo with a `.claude-plugin/marketplace.json`. Pushing to
GitHub is publishing; installing means pointing Claude Code at this repo.

### As a plugin (recommended)

```
/plugin marketplace add JeanIsahakyan/skills
/plugin install skills@jeanisahakyan
```

First line registers this repo as a marketplace (Claude Code clones it and reads
`.claude-plugin/marketplace.json`); second installs the `skills` plugin from it — which
bundles every skill under `plugin/skills/`. After the marketplace is added you can also
just run `/plugin install skills`. Pull updates with `/plugin marketplace update jeanisahakyan`.

> The `skills@jeanisahakyan` form is `<plugin-name>@<marketplace-name>` — `skills` is the
> plugin (`plugin/.claude-plugin/plugin.json`), `jeanisahakyan` is the marketplace
> (`.claude-plugin/marketplace.json` → `name`). They're separate names by design, not the
> `owner/repo` GitHub path.

### Manually (single skill, no plugin system)

Copy (or symlink) one skill directory into your agent's skills folder
(`~/.claude/skills/` for Claude Code):

```bash
cp -r plugin/skills/autopilot ~/.claude/skills/autopilot
# or, to keep it in sync with this repo:
ln -s "$(pwd)/plugin/skills/autopilot" ~/.claude/skills/autopilot
```

The agent picks it up automatically; the skill's `description` decides when it
triggers.

## Skills

| Skill | What it does |
|-------|--------------|
| [`autopilot`](plugin/skills/autopilot/) | Universal autonomous feature/project builder — turns ~one prompt into a whole working, tested feature without asking questions, refusing to build the wrong/redundant thing up front (premise gate) and surfacing the irreversible decisions + honest verification claims at the end. Validated against evals (see the skill's `evals/`). |
| [`simulate`](plugin/skills/simulate/) | Stress-test a spec / design doc **before** writing code by mentally executing it as every component at once — data stores, services, clients, external systems — running golden / edge / adversarial scenarios with explicit state diffs, looped autonomously (spec-analyzer → parallel simulators → findings-aggregator) until findings settle to LOW. Produces a deduped, severity-ranked `FINDINGS.md`. Generalized from a production feature pipeline's spec-simulation stage. |

## Adding a skill

This is a growing collection — new skills land over time. To add one:

1. Create `plugin/skills/<name>/SKILL.md` (plus optional `references/` and `evals/`).
2. Append a row to the **Skills** table above.
3. Commit and push.

That's it. Skills are auto-discovered under `plugin/skills/`, so they ship in the
`skills` plugin automatically — **no `.claude-plugin/` manifest edits per skill.** The
README table is the single living list of what's here. Bump `version` in
`plugin/.claude-plugin/plugin.json` when you want a release marker for
`/plugin marketplace update`.

## Notes

Skills here are developed and maintained primarily through AI assistance — see
[`CLAUDE.md`](CLAUDE.md) for how that work is calibrated.
