# Fabian Builds — Agent Skills

A collection of [Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) I use with Claude Code and other AI coding agents, shared here so anyone can install and use them too.

Each skill is a folder under `skills/` containing a `SKILL.md` — a set of instructions an agent reads and follows when the task matches. You don't need to understand the internals to use one; just install it and mention what you want to do.

More skills will be added over time.

## Skills in this repo

| Skill | What it does |
|---|---|
| [`optimize-neon-db`](skills/optimize-neon-db/SKILL.md) | Audits a Next.js App Router site and cuts unnecessary database compute usage on Neon (or any other scale-to-zero serverless database) — usually the fix for "why does my DB never go idle" or "why is my database bill so high." |

## Install

You don't need to read or copy any files by hand. Paste the prompt below into a coding agent (Claude Code, or similar) — it works whether you point it at this repo remotely or you've already cloned it locally.

```
Get me the skills from https://github.com/fabianfreund/Fabian-Builds-Agent-Skills
(clone it if it isn't already local). List every skill you find under its
skills/ folder with a one-line description of each, and ask me which one(s)
I want to install.

For each skill I pick:
- If the `skillshare` tool is available on this machine, use it to install the
  skill properly (check skillshare's own help/docs for the exact command).
- Otherwise, copy that skill's folder as-is into ~/.claude/skills/<skill-name>/
  (create ~/.claude/skills/ first if it doesn't exist).

Tell me exactly what got installed and where when you're done.
```

### What this actually does

- **Global install** (no `skillshare`): the skill folder is copied to `~/.claude/skills/<skill-name>/`, making it available to Claude Code in every project on your machine.
- **Via [skillshare](https://github.com/search?q=skillshare)**: if you manage your skills across multiple tools/projects with skillshare, the agent will use it instead so the skill stays in sync with however you've already got things set up (global vs. per-project, symlink vs. copy).

Either way, nothing runs automatically — the skill only activates when you ask for something it matches (e.g. "why is my Neon bill so high").
