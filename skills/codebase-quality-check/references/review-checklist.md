# Review checklist

Organized by concern, not language, apply whatever's relevant to the
detected stack. Skip items that genuinely don't apply (e.g. "component
reusability" in a CLI tool with no UI), don't force-fit.

## 1. Baseline health

- [ ] Typecheck (or equivalent) passes with 0 errors.
- [ ] Lint passes, or existing warnings are understood and intentional.
- [ ] Test suite runs and passes. Note the count, 0 tests is itself a finding.
- [ ] Build/compile succeeds.
- [ ] If a category has no tooling configured at all, say so plainly,
      that's a finding, not a non-event.

## 2. File & module size

- [ ] List source files by line count, largest first.
- [ ] For each outlier: generated code, a legitimate single-purpose
      catalog/config/data file, or genuine god-object/god-function sprawl?
- [ ] Flag only the latter. A big file with one clear job isn't a
      problem; a medium file doing five unrelated things is.

## 3. Component / module reusability

- [ ] One clear way to build the common unit (UI component, service
      handler, data model), or multiple near-duplicate implementations?
- [ ] Variants expressed as props/parameters/config, or copy-pasted clones?
- [ ] Do reusable pieces stay presentational/logic-free where convention
      expects it (UI not embedding business logic, handlers not embedding
      view formatting)?
- [ ] Anything exported but referenced only by its own demo/story/test,
      dead in production but alive in the catalog? Verify with a real
      usage grep, don't assume from naming.

## 4. Dead code & duplication

- [ ] Grep exported symbols with zero real call sites (excluding
      tests/demos/stories).
- [ ] Two implementations solving the same problem slightly differently
      (two date formatters, two fetch wrappers), a sign of drift.
- [ ] Unused dependencies in the manifest (installed, never imported).

## 5. Data & content storage

- [ ] Structured content/config split by concern into small files/tables,
      or one monolithic blob?
- [ ] Cross-reference both directions: does every data entry get used
      somewhere, and does every usage resolve to a real entry? Orphaned
      data and dangling references are both findings.
- [ ] Single source of truth per fact, or the same value hardcoded in two
      places that can drift?
- [ ] Multi-locale/multi-tenant/multi-environment: do parallel data sets
      have matching keys/shape? Diff programmatically, don't eyeball.

## 6. Internal/dev-only surface isolation

- [ ] Internal tooling, admin panels, style guides, debug routes: actually
      excluded from "production" (not indexed, not in the public bundle,
      not reachable without the right guard)? Check the real exclusion
      mechanism (robots rules, route guards, build config), don't trust a
      comment claiming it.

## 7. Docs vs. code drift

- [ ] Find the repo's own source-of-truth docs (README, CONTRIBUTING,
      ROADMAP/CHANGELOG, ADRs, docs/), don't invent structure that isn't there.
- [ ] For each state claim ("step X done", "supports Y"), verify against
      the actual code/config. Flag overclaims and underclaims separately,
      both are drift.
- [ ] Check setup/run instructions match the manifest's real scripts.

## 8. Git/repo hygiene (context, not a verdict)

- [ ] Uncommitted changes, roughly how large/stale (flag if large and
      stale, not a defect by itself).
- [ ] Anything that looks like committed secrets, credentials, or
      accidentally-committed build output/dependencies, this is a P1 if
      found, regardless of everything else.
