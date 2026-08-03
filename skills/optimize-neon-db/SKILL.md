---
name: optimize-neon-db
description: >-
  Cut unnecessary database compute usage on Neon or any scale-to-zero
  serverless database (Postgres, MySQL, SQLite-over-HTTP, etc.) in a
  Next.js App Router app. Use when the user asks to reduce Neon/DB
  usage or cost, asks why their DB never idles, or wants a content site
  turned into static pages to save compute. Neon is the main example,
  but the fix applies to any compute-billed serverless database.
metadata:
  author: Fabian Freund
  version: 2.0.0
---

# Optimize serverless DB compute usage

Serverless databases (Neon, PlanetScale, Supabase, Turso, Aurora
Serverless, etc.) bill for active compute time and auto-suspend when
idle. The fix is rarely query efficiency, it's whether a page hits the
DB at all. A Next.js route with no `generateStaticParams` re-runs on
every request, with zero caching, because DB calls aren't `fetch()`-based.
Every visitor and crawler hit then keeps compute awake.

This applies just as much to metadata/feed routes as to normal pages:
`sitemap.ts`, `robots.ts`, `opengraph-image.tsx`, RSS `route.ts` files.
These often carry `force-dynamic` plus a DB query and get hit by
crawlers and link-preview bots on a schedule, independent of real
traffic. Usually the single biggest win in an audit.

Written for Next.js App Router (Step 1 reads the `next build` output,
which is Next-specific). In another framework, reuse the reasoning
(Steps 2, 5, 7) and translate the caching/revalidation calls to that
framework's equivalents.

Not for: query-level tuning (indexes, N+1), that's a different problem.
Not for pages that are genuinely personalized per user (auth, A/B),
those stay dynamic on purpose.

## Step 0: Ask how far to push it

Don't assume max effort. Ask:
- Static-generation wins only (Steps 1-6), or also cache dynamic
  routes that can't go fully static (Step 7)?
- Concrete target ("get under free tier") or just "less"?
- How much staleness is OK? Shapes the `revalidate` numbers below.

Revisit after Step 1. If there's barely anything to cache, skip
raising Step 7 at all.

## Step 1: Diagnose

```bash
rm -rf .next && npx next build
```

Read the route table: `○` static, `●` SSG, `ƒ` dynamic (hits DB every
request). Check the whole table, not just `[slug]` rows, metadata files
show up there too. Cross-check with:

```bash
grep -rn 'force-dynamic\|revalidate = 0\|revalidate: 0' src/app
```

For each hit, confirm it queries the DB. Don't fix anything yet.

## Step 2: Classify

A route is fixable if it reads only `params` (no `searchParams`,
`cookies()`, `headers()`) and its data only changes through a known
mutation path (admin/ingest API, not arbitrary per-request state).

Leave it dynamic if it reads `searchParams` (filters/search/sort,
forces the whole route dynamic, can't cache around it without Cache
Components) or `cookies()`/`headers()` (sessions, personalization).

## Step 3: Fix

Dynamic-segment pages: add `generateStaticParams`, reusing an existing
slug-list helper if one exists (often shared with the sitemap):

```ts
export async function generateStaticParams() {
  const { posts } = await listAllSlugs();
  return posts.map((p) => ({ slug: p.slug }));
}
```

Parameterized metadata files (`[slug]/opengraph-image.tsx`): same fix.

Fixed-path files (`sitemap.ts`, `robots.ts`, feed `route.ts`): no
params to enumerate, replace `force-dynamic`/`revalidate: 0` with:

```ts
export const revalidate = 86400; // ceiling, not the real freshness signal
```

Pick the number from how often content actually changes, not how fresh
you'd like it. A long window is fine if Step 4 confirms on-demand
revalidation covers the gap.

## Step 4: Confirm revalidation reaches these routes

Check for an existing helper (e.g. `revalidatePublic()`) called from
every mutation. If none exists, converting to static means edits won't
show up until the next deploy, flag that before proceeding.

Verify it empirically, don't just trust the code. A blanket
`revalidatePath("/", "layout")` doesn't obviously reach a metadata file:

```bash
rm -rf .next && npx next build && npx next start &
curl -s localhost:3000/sitemap.xml | grep -c "not-added-yet"   # 0
curl -s -X POST -H "x-api-token: $TOKEN" -d '{"name":"Probe","slug":"probe"}' \
  localhost:3000/api/<ingest-endpoint>
curl -s localhost:3000/sitemap.xml | grep -c "probe"           # expect 1, no wait
```

`1` right away means the `revalidate` number is a safe fallback
ceiling, set it generously. `0` means the helper isn't reaching this
route, add an explicit call or shorten the window. Clean up the probe
row after.

## Step 5: Audit writes that bypass revalidation

Find mutation endpoints (votes, likes, view counts) that update data on
a now-static page but don't call the revalidation helper. They'll serve
stale values forever. Fix with a narrowly scoped call, not the blanket
helper:

```ts
export async function POST(req, { params }) {
  const { slug } = await params;
  // update the DB
  revalidatePath(`/blog/${slug}`); // just this page
  return Response.json({ ok: true });
}
```

Verify the scope is actually narrow with the `x-nextjs-cache` (local) /
`x-vercel-cache` (deployed) header: hit a target item and an unrelated
control item, mutate the target, re-check both immediately. Target
should flip `HIT → MISS`, control should stay `HIT`. If control also
flips, a blanket helper is being called instead of the narrow one.

## Step 6: Verify

```bash
rm -rf .next && npx next build && npx tsc --noEmit
```

Fixed routes flipped `ƒ` to `●`/`○`. Routes that should stay dynamic
(search, filters) are still `ƒ`.

## Step 7: Hard-static mode (opt-in, ask first)

Some routes can't go static (`searchParams`-driven search/filters), but
the query behind them can still be cached even though the route stays
`ƒ`. This is more work, it means auditing every mutation path including
cron jobs, and trades a small self-healing staleness window for
near-zero DB touches. Ask before doing this.

If yes:
1. Re-run Step 1 across all routes, including API routes.
2. Classify: legitimate write (leave it) vs. cacheable read
   (`searchParams`-driven query) vs. genuinely per-request (leave it).
3. Wrap the underlying data function, not the route handler, in
   `unstable_cache`, one shared tag so one mutation invalidates
   everything:
   ```ts
   export const listItems = unstable_cache(listItemsUncached, ["list-items"], {
     revalidate: 86400,
     tags: [CONTENT_TAG],
   });
   ```
   Check this project's actual `revalidateTag` signature in
   `node_modules/next/dist/docs` first, some versions need a second
   `profile` arg (`revalidateTag(tag, { expire: 0 })` for immediate).
4. Re-run Step 5's audit exhaustively. Cron/scheduled jobs are the most
   commonly missed invalidation source, they predate the caching work.
5. Verify each mutation path found in step 4, same technique as Step 4.
6. Report a full route-by-route list: write vs. cached read, so nothing
   gets miscategorized.

## Example

Directory site on Neon. User: "why is my DB bill high?"
1. `next build` showed `[slug]` pages dynamic, added
   `generateStaticParams` (Step 3), reusing the sitemap's slug helper.
2. Grep found `sitemap.ts` and two feed routes `force-dynamic` with DB
   calls, hit by crawlers on a schedule. Switched to `revalidate: 86400`.
3. Existing `revalidatePublic()` helper already covered invalidation,
   verified with the curl probe, confirmed instant.
4. Found a vote endpoint updating a now-static page without
   revalidating. Added scoped `revalidatePath`, verified with the
   HIT/control-MISS header check.
5. Later, asked to push further: cached the remaining `searchParams`-driven
   listing/search queries (Step 7). Audit caught a price-refresh cron
   that never revalidated anything, fixed it.
Result: only real writes (votes, ingest, cron) touch the DB now,
everything else is static or cached until something changes.

## Troubleshooting

Route stays `ƒ` after `generateStaticParams`: it's also reading
`searchParams`/`cookies()`/`headers()` somewhere in the tree, any one
forces the whole route dynamic. Accept it, or use Suspense if the
project uses Cache Components.

Build much slower after `generateStaticParams`: the slug query now runs
at build time. Make sure it's indexed/efficient. For huge datasets,
return `[]` from `generateStaticParams` and rely on on-demand
generation after first request.

Edits don't show up: no revalidation call at the mutation site (Step
4), or it's not scoped to the route in question (Step 5).
