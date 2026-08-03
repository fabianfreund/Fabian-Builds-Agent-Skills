---
name: update-docs
description: Audit and update project docs such as CLAUDE.md, AGENTS.md, README.md, ROADMAP.md, TODO.md, and docs/ files. Use when docs need cleanup, alignment with code, preservation of existing knowledge, roadmap/todo maintenance, or incorporation of available client feedback and briefings.
---

# update-docs

Clean up a project's `CLAUDE.md`, `AGENTS.md`, `README.md`, `ROADMAP.md`, `TODO.md`, and `docs/` files to a consistent structure without losing information.

## Audit first

Read before editing: `CLAUDE.md`/`AGENTS.md`, `README.md`, all `docs/*.md`, `ROADMAP.md`/`TODO.md` (root or `docs/`), and any client feedback/briefs in the current thread or repo. Check code/config to verify claims. If docs are already accurate, lean, and aligned, make no changes and say so. Prefer targeted edits over rewrites.

## Preserve knowledge, cut filler

Never lose a fact, constraint, workflow, env var, command, decision, requirement, or gotcha. Only remove duplication, stale statements, and filler. Prefer short paragraphs, bullets, tables. No speculative future plans or generic project-management prose.

Client feedback and briefs are input, not gospel: incorporate them, but don't let them overwrite confirmed repo facts. If a brief conflicts with the actual implementation, document both, planned/desired vs. currently implemented.

## CLAUDE.md / AGENTS.md: index layer, keep it minimal

```
# <Title> — <Project Name>

<One paragraph: what it does, who uses it.>

**Status:** <one line>

---

## Load these docs when relevant

| Topic | File |
|---|---|
| <topic> | `docs/<file>.md` |

---

## Dev workflow

\`\`\`bash
<commands to run locally>
\`\`\`

## Stack at a glance

<key technologies>

---

## Docs hygiene

Keep this file short. Detailed notes go in `docs/`. Update it when the summary or table goes stale.
```

Belongs here: name, one-liner, status, doc index, dev commands, stack list, hygiene rule. Does not belong: implementation details, patterns, env vars, architecture decisions, those go in `docs/`.

## README.md: user-facing entrypoint

Answers what the project is, how to run it, where the deeper docs are, current status. Link to `docs/` instead of duplicating it. Create one if missing and useful; keep as-is if already good.

## docs/ files: one topic per file

```
# <Topic>

## Overview
<1-3 sentences>

## <Main concept>
...

## Patterns
<conventions, gotchas>

## Env vars
| Var | Purpose |
|---|---|

## Hard rules
- <things that must never change, or that burned us before>
```

No redundant content across files, link instead of repeating. Split a file when it covers multiple distinct concerns (e.g. an `ARCHITECTURE.md` covering both frontend and backend → split into two). Right granularity: one file per topic someone would search for independently.

## ROADMAP.md / TODO.md

Use existing locations, don't duplicate. Create only if useful or requested. Roadmap: phases, priorities, gaps. Todo: actionable next tasks with enough context to execute. Mark brief-driven items as planned, not implemented. Update stale items only after checking the repo.

## When invoked

1. Audit (above).
2. Verify claims against code where practical.
3. Look for: misplaced implementation detail in CLAUDE.md/AGENTS.md, missing doc files for referenced topics, stale README/ROADMAP/TODO, docs/ files missing standard sections, duplicate content, stale doc-index rows, unimplemented brief items presented as done.
4. Nothing wrong → no edits, report what was checked.
5. Otherwise: move misplaced content into the right `docs/` file, rewrite CLAUDE.md/AGENTS.md to the template only if needed, align/create README/ROADMAP/TODO, restructure non-conforming docs/ files, fix the doc-index table, preserve everything useful.
6. Report what was checked, kept, moved, created, or restructured. Keep it short.
