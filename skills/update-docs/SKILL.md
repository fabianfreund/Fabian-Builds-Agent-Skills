---
name: update-docs
description: Audit and update project docs such as CLAUDE.md, AGENTS.md, README.md, ROADMAP.md, TODO.md, and docs/ files. Use when docs need cleanup, alignment with code, preservation of existing knowledge, roadmap/todo maintenance, or incorporation of available client feedback and briefings.
---

# update-docs

Review and clean up the project's `CLAUDE.md`, `AGENTS.md`, `README.md`, `ROADMAP.md`, `TODO.md`, and `docs/` files so they follow the established structure conventions without losing useful information.

## Rules

### Start with an audit

Inspect the current docs before rewriting anything. If the docs are already accurate, lean, and aligned, keep them as-is and report that no change was needed. Prefer targeted edits over broad rewrites.

Before changing docs:

- Read `CLAUDE.md` and `AGENTS.md` when present.
- Read `README.md` when present, especially for user-facing setup, overview, and status.
- Glob and read `docs/*.md`.
- Check for `ROADMAP.md`, `TODO.md`, `docs/ROADMAP.md`, and `docs/TODO.md`.
- Inspect the relevant code/config paths needed to verify claims.
- Look for available client feedback, briefs, specs, planning notes, issue descriptions, or quoted instructions in the current thread/repo and treat them as input.

### Preserve knowledge, keep text lean

Do not lose documentation content. When shortening, moving, or restructuring, preserve every useful fact, constraint, workflow, environment variable, command, product decision, client requirement, and hard-won gotcha. Remove only duplication, stale statements, filler, and claims disproven by code.

Keep docs concise and scannable. Prefer short paragraphs, bullets, and tables. Do not add overbloated explanatory text, speculative future plans, or generic project-management prose.

### Client feedback and briefs

Incorporate client feedback, briefs, and explicit user instructions when available, but do not let them overwrite confirmed repo facts. If a brief conflicts with implementation, document the mismatch clearly as planned/desired behavior versus currently implemented behavior.

### CLAUDE.md / AGENTS.md — keep it minimal

These files are the index layer. They must stay short. The structure is:

```
# <Title> — <Project Name>

<One-paragraph description of what the project does and who uses it.>

**Status:** <Current phase / one-line status.>

---

## Load these docs when relevant

| Topic | File |
|---|---|
| <topic description> | `docs/<file>.md` |
...

---

## Dev workflow

\`\`\`bash
<commands to run the project locally>
\`\`\`

## Stack at a glance

<Comma/dot-separated list of key technologies>

---

## Docs hygiene

Keep this file short. Detailed notes go in `docs/`. Update the relevant doc file when you learn something new (patterns, gotchas, env vars, architecture decisions). Only touch this file if the top-level summary or table is genuinely outdated.
```

**What belongs here:** project name, one-liner, status, doc index table, dev commands, stack list, docs-hygiene rule.
**What does NOT belong here:** implementation details, code snippets (except dev commands), patterns, gotchas, env var lists, architecture decisions — all of that goes in `docs/`.

### README.md — user-facing entrypoint

Keep `README.md` aligned with the code and the detailed docs. It should answer what the project is, how to run it, where to find deeper docs, and what the current status is. Do not duplicate all architecture details from `docs/`; link to the relevant files.

If no `README.md` exists and the repo would benefit from one, create a lean README. If an existing README is already good, keep it and report that.

### docs/ files — structured detail

Each `docs/<topic>.md` file covers one topic. Standard sections (use only what applies):

```
# <Topic>

## Overview
<1–3 sentences on what this layer does and why.>

## <Main concept / Key sections>
...

## Patterns
<Reusable patterns, conventions, gotchas — bullet list or short subsections.>

## Env vars
| Var | Purpose |
|---|---|
| `VAR_NAME` | what it does |

## Hard rules
- <Things that must never change or that burned us before.>
```

Keep sections short and scannable. Prefer tables and bullets over paragraphs. No redundant content across files — if something is already covered in another doc, link to it instead of repeating.

### ROADMAP.md and TODO.md — keep planning aligned

Maintain a roadmap and a todo file when the project has planning needs.

- Prefer existing locations (`ROADMAP.md`, `TODO.md`, `docs/ROADMAP.md`, `docs/TODO.md`) instead of creating duplicates.
- Create missing roadmap/todo files only when they are useful for the repo or explicitly requested.
- `ROADMAP.md` should capture phases, priorities, planned work, and known gaps.
- `TODO.md` should capture actionable next tasks with enough context to execute.
- Keep both aligned with code, docs, and client feedback. Mark uncertain or brief-driven items as planned/requested, not implemented.
- Remove or update completed/stale items only after checking the repo or relevant docs.

### Splitting docs/ files — one topic per file

If a `docs/` file is growing too large or covers multiple distinct concerns, split it. Create a new `docs/<topic>.md` for each cohesive topic and update the doc-index table in `CLAUDE.md`. Examples of good split boundaries:

- A large `ARCHITECTURE.md` covering both frontend routing and backend API → split into `FRONTEND.md` and `BACKEND.md`
- A `STYLE_GUIDE.md` that also contains animation patterns → split off `ANIMATION.md`
- A `TESTING_GUIDE.md` that also documents CI/CD → split off `CI.md`

The right granularity: one file per topic a developer would search for independently. If two sections are always read together, keep them together. If they're consulted separately, split them.

## What to do when this skill is invoked

1. Audit first. Read the relevant root docs, `docs/*.md`, roadmap/todo files, and available client feedback or briefs.
2. Verify important claims against code/config where practical.
3. Identify issues:
   - Implementation details, patterns, or env vars living in `CLAUDE.md`/`AGENTS.md` instead of `docs/`
   - Missing doc files for topics referenced but not yet written
   - Missing or stale `README.md`, `ROADMAP.md`, or `TODO.md` coverage
   - `docs/` files that are missing standard sections or have bloated paragraphs
   - Duplicate content across files
   - Outdated doc-index rows (files in docs/ not listed, or listed files that don't exist)
   - Brief/client-requested behavior presented as implemented without code support
4. If everything is already good, make no edits and report what was checked.
5. Otherwise, make targeted fixes:
   - Move misplaced content from `CLAUDE.md`/`AGENTS.md` into the correct `docs/` file (create it if needed)
   - Rewrite `CLAUDE.md`/`AGENTS.md` to the minimal template above only when needed
   - Align or create `README.md`, `ROADMAP.md`, and `TODO.md` when useful
   - Restructure any `docs/` files that don't follow the standard sections
   - Update the doc-index table to match the actual files on disk
   - Preserve all useful existing content while making it leaner
6. Report what was checked, what was kept, and what was moved, created, or restructured. Keep the summary short.
