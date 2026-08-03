---
name: codebase-quality-check
description: >-
  Run a full-repo health audit: big/messy files, component reusability,
  data-storage cleanliness, docs-vs-code drift, baseline checks
  (typecheck/lint/test/build). Tech-stack agnostic, detects the stack
  instead of assuming one. Use when the user asks for a codebase health
  check, quality audit, "is this clean", "is this ready to extend", or
  wants a prioritized cleanup report.
pattern: reviewer
---

# Codebase Quality Check

A generalist audit of a repo's structural health: is the code organized,
reusable, and honestly documented, or accumulating debt. Not a security
review (`/security-review`), not a line-by-line bug hunt (`/code-review`).

Works on any stack. Detect it, don't assume Next.js/npm/Python/whatever.

## Step 0: Scope

Whole-repo audit (default) or a subdirectory/package the user names. In a
monorepo with multiple independent projects, ask which one, or audit each
separately rather than blending unrelated stacks into one report.

## Step 1: Detect the stack

Check, in order of how definitive: manifest/lockfiles (`package.json`,
`pyproject.toml`/`requirements.txt`, `Cargo.toml`, `go.mod`, `Gemfile`,
`composer.json`, `pom.xml`/`build.gradle`, `mix.exs`, `*.csproj`); task
runners (`package.json` scripts, `Makefile`, `justfile`, `tox.ini`); CI
config (often the ground truth for what "passing" actually means).

Read `references/review-checklist.md` now, it's organized by concern, not
by language, so it applies regardless of stack.

## Step 2: Baseline (does it currently work)

Run the repo's own typecheck/lint/test/build commands, not a guessed
generic one. Skip cleanly if a category has no tooling configured, note it
as a gap, don't fake a result. Capture pass/fail and error counts, this is
what the rest of the report hangs off, if typecheck is broken everything
else is noise until that's fixed.

## Step 3: Survey structure

List source files by line count, largest first (exclude lockfiles,
generated/vendored code, build output). Get the directory tree. For each
outlier, open it and judge why it's large, generated file, legitimate
catalog page, or a module doing too many things. Size alone isn't the
finding.

## Step 4: Apply the checklist

Work through `references/review-checklist.md` section by section. Verify
every finding: grep for actual usage before calling something dead code,
check the doc claim against the real file/route/command before calling it
stale. Don't speculate.

Classify:
- **P1 (fix now)**: breaks the baseline, actively misleads, or blocks
  extending the codebase.
- **P2 (fix soon)**: real debt that doesn't block anything today but will
  compound.
- **P3 (nice to have)**: cosmetic or a judgment call the user may decline.

## Step 5: Report

Plain language, short.
1. **Baseline**: typecheck/lint/test/build results, one line each.
2. **What's healthy**: a short list of verified-good things (don't skip
   this, an all-complaints report reads as biased and buries priorities).
3. **Priority action list**: P1 → P3, each as what/where/why/one-line fix.

No checklist score or grade, a percentage across unrelated concerns is
meaningless. State findings and priority. Keep it skimmable under a
minute.
