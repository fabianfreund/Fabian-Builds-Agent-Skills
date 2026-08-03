# Fabian Builds: Agent Skills

Agent Skills I use with Claude Code, shared here so anyone can install them too. More will be added over time.

## Skills

| Skill | What it does |
|---|---|
| [`optimize-neon-db`](skills/optimize-neon-db/SKILL.md) | Cuts unnecessary database compute usage in a Next.js app on Neon (or any scale-to-zero database). Fixes "why does my DB never go idle" / "why is my database bill so high." |
| [`update-docs`](skills/update-docs/SKILL.md) | Audits and cleans up a repo's CLAUDE.md, AGENTS.md, README.md, ROADMAP.md, TODO.md, and docs/ files. Keeps them lean and accurate without losing content. |
| [`codebase-quality-check`](skills/codebase-quality-check/SKILL.md) | Full-repo health audit: messy files, dead code, reusability, docs-vs-code drift, baseline typecheck/lint/test/build. Tech-stack agnostic, gives a prioritized cleanup report. |

## Install

Paste this into a coding agent:

```
Get me the skills from https://github.com/fabianfreund/Fabian-Builds-Agent-Skills
(clone it if it isn't already local). List every skill under its skills/
folder with a one-line description, and ask me which one(s) I want.

For each skill I pick: use the `skillshare` tool to install it if available
on this machine, otherwise copy the skill's folder into
~/.claude/skills/<skill-name>/ (create that folder if needed).

Tell me what got installed and where.
```

Nothing runs automatically. A skill only activates when you ask for something it matches.
