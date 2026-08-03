---
name: optimize-neon-db
description: >-
  Audit and fix a Next.js App Router site for unnecessary database
  compute usage on Neon or any other scale-to-zero / compute-billed
  serverless database (Postgres, MySQL, SQLite-over-HTTP, whatever's
  behind the ORM). Use when the user asks to "reduce Neon usage" (or
  PlanetScale/Supabase/Turso/Aurora Serverless/etc.), "cut database
  costs", "why is my DB bill high", "use less DB", "does my DB idle",
  "optimize database compute", or reports that a serverless database
  project seems to hit the database more than it should. Neon is the
  primary worked example throughout, but nothing in the technique is
  Neon- or Postgres-specific.
metadata:
  author: Fabian Freund
  version: 1.5.0
---

# Optimize serverless DB compute usage in a Next.js App Router site

Serverless database providers bill for **active compute time**, not per
query, and auto-suspend to zero after a period of inactivity — Neon
(Postgres, billed in CU-hours) is the primary example throughout this
skill, but the same billing shape and the same fix apply to PlanetScale,
Supabase, Turso/libSQL, Aurora Serverless v2, CockroachDB Serverless, or
any other database that scales compute to zero when idle. The single
biggest lever for cost is usually not query efficiency — it's whether
pages hit the database **at all** on a normal page view, regardless of
which database or ORM is on the other end of the query. A Next.js App
Router route with a dynamic segment (`[slug]`) that has no
`generateStaticParams` is fully server-rendered on *every single
request* by default, with zero caching, because ORM/DB calls (Drizzle,
Prisma, etc.) aren't `fetch()`-based, so Next.js has no automatic way to
cache them. On a content site (directory, blog, catalog), that means
every visitor and every crawler hit keeps compute awake and resets the
idle-suspend timer — the DB may never get the chance to scale to zero.
Nothing about this diagnosis depends on Postgres specifically: the same
`ƒ` marker in the build table means "hits the database on every request"
whether that database is Neon, PlanetScale, or anything else.

The same problem hides in **metadata and feed routes** just as often as
in `[slug]` pages, and is easy to miss because they don't look like
"pages": `app/sitemap.ts`, `app/robots.ts`, `app/**/opengraph-image.tsx`
/ `twitter-image.tsx`, `app/manifest.ts`, and any hand-rolled
`route.ts` serving RSS/Atom/JSON feeds. These frequently carry an
explicit `export const dynamic = "force-dynamic"` (often with a comment
like "always read fresh") plus a DB query, sometimes added deliberately
by a previous author who wanted live data and didn't think about the
compute cost. Treat these as **higher priority than ordinary content
pages, not lower** — a human might assume "nobody visits `/sitemap.xml`
or `/blog/feed.xml`", but search crawlers re-fetch sitemaps on their own
schedule, RSS readers poll feeds independent of any real visitor, and
link-preview bots (Discord, Slack, Twitter/X, iMessage) hit OG-image
routes on every share. That traffic is automated, frequent, and
completely decoupled from whether a human is actually looking at the
site — exactly the pattern that keeps a serverless database instance
from ever reaching its idle-suspend window, whatever database engine it
happens to be.

This skill finds those routes — both `[slug]` pages and fixed-path
metadata/feed routes — and converts the ones that are safe to convert
into prerendered static or ISR-cached output, without breaking
legitimately dynamic pages (search, filters, personalization) or
introducing stale-data regressions on writes that bypass the site's
normal content-revalidation path. Steps 1-6 cover this baseline pass.
Step 7 is a further, **opt-in** escalation — caching the database
*queries* behind routes that must stay dynamic (search, filtered
listings) — for when the user explicitly wants DB usage pushed as close
to zero as possible, not just "the pages that were free wins."

## When to use

Use this skill when the user:
- Asks how to reduce Neon (or PlanetScale, Supabase, Turso, Aurora
  Serverless, or any other scale-to-zero database) usage/cost
- Asks whether they pay for DB compute while idle vs. only on real traffic
- Wants a content-heavy Next.js App Router site to "bake into" static pages
- Reports a serverless database bill that seems high for the actual traffic

Do NOT use this skill's concrete steps as-is for:
- Query-level optimization (missing indexes, N+1 queries, slow joins) —
  that's a different problem; this skill is about whether the DB gets
  hit *at all* on a given request, not how expensive the hit is once
  it happens.
- Apps where every dynamic page is genuinely personalized (per-user
  dashboards, auth-gated content) — those need to stay dynamic; this
  skill's value is concentrated in public, DB-backed content pages.

This skill is written and documented for **Next.js App Router** —
that's the default, primary, fully-detailed case, and the diagnostic
step (Step 1) reads the `next build` route table specifically, which
only exists in Next.js. If you're working in a different framework
(Remix, SvelteKit, Nuxt, plain Express, etc.), the concrete
commands/APIs here (`generateStaticParams`, `unstable_cache`,
`revalidatePath`/`revalidateTag`) won't apply directly — but the
underlying principle transfers regardless of framework: find every
route/handler that queries the database on every request regardless of
whether the data actually changed, work out which ones are safe to
cache or prerender, wrap the read path in whatever caching primitive
that framework offers (HTTP `Cache-Control` headers, a loader-level
cache, a CDN edge cache, etc.), and wire invalidation into the actual
mutation paths. An agent in a non-Next.js repo can use this skill's
*reasoning* (Steps 2, 5, and 7's classification logic especially) even
though Steps 3/4/6's specific commands need translating to that
framework's equivalents.

## Instructions

### Step 0: Calibrate scope with the user before diagnosing

Don't assume "reduce DB usage" always means "push it as far as
possible" — ask first, before running the Step 1 diagnostic, so the
depth of the audit matches what the user actually wants rather than
defaulting to maximum effort (or, just as bad, stopping short of what
they'd have wanted if they'd known it was on the table). A short check-in
covers this; it doesn't need to be a long interview:

- **How far do they want this pushed?** The safe, essentially-free wins
  (Steps 1-6: static generation for content pages and metadata/feed
  routes) versus going further into Step 7's territory (caching the
  queries behind routes that must stay dynamic — search, filters,
  listings) — which is more work, touches more of the app, and trades a
  small self-healing staleness window for near-zero DB touches on reads.
- **Is there a concrete target, or just "less"?** "Get back under the
  free tier," "stop paying for the Pro plan," "I don't care about the
  number, just don't make it worse" — a concrete number changes how
  aggressive to be and gives you something to check progress against
  instead of optimizing indefinitely.
- **How much staleness is acceptable?** This shapes the `revalidate`
  ceilings chosen in Steps 3b and 7 — a site that publishes hourly wants
  a shorter fallback window than one that publishes weekly, independent
  of whatever on-demand invalidation is also in place.

Revisit this mid-audit if Step 1's findings change the picture — e.g. if
the diagnostic turns up only a couple of static-page opportunities,
Step 7 may not be worth raising at all; if it turns up several
`searchParams`-driven listing/search routes doing real query work, it's
worth surfacing as an option even if the user didn't ask for it by name.

### Step 1: Diagnose — read the build output

Run a production build and inspect the route table Next.js prints:

```bash
rm -rf .next && npx next build
```

Look at the legend at the bottom and the marker next to each route:

```
○  (Static)   prerendered as static content
●  (SSG)      prerendered as static HTML (uses generateStaticParams)
ƒ  (Dynamic)  server-rendered on demand
```

Any route with a dynamic segment (`[slug]`, `[id]`, etc.) marked `ƒ` is
being rendered fresh — and therefore hitting the DB fresh — on every
single request. Static index routes with no params (home, category
listing, etc.) often already show `○` automatically if they don't read
`searchParams`/`cookies`/`headers`, since Next renders them once at build
time.

**Don't stop at `[slug]` routes.** The build table also marks
`app/sitemap.ts`, `app/robots.ts`, `app/manifest.ts`,
`opengraph-image.tsx`/`twitter-image.tsx`, and any `route.ts` (RSS/Atom
feeds, etc.) as `ƒ` when they're forced dynamic — scan the *entire* route
table for `ƒ`, not just the `[slug]`-shaped rows. Then cross-check with a
direct grep, since it's easy to skim past a one-line metadata file in a
long build table:

```bash
grep -rn 'force-dynamic\|revalidate = 0\|revalidate: 0' src/app  # or app/, per project layout
```

For each hit, open the file and check whether it also queries the
database. A `force-dynamic` sitemap/feed/OG-image route that does is
exactly the pattern this skill targets — often the single highest-impact
fix in the whole audit, because these routes get hit by crawlers and
bots on a schedule regardless of real traffic (see the intro above).

Flag every `ƒ` route — `[slug]`-shaped or not — as a candidate, but don't
treat any of them as automatically fixable yet — check Step 2 first.

### Step 2: Root-cause and separate real candidates from fake ones

For each `ƒ` route — dynamic-segment page or fixed-path metadata/feed
route alike — check whether it legitimately needs to be dynamic. A route
is a **good candidate to fix** if:

- It reads only `params` (its own dynamic segment, if any) — not
  `searchParams`, `cookies()`, or `headers()`. Fixed-path metadata/feed
  routes (`sitemap.ts`, `feed.xml/route.ts`, `robots.ts`) almost always
  pass this automatically, since there's nothing dynamic to read besides
  the DB call itself — that's exactly why they tend to be `force-dynamic`
  by accident/habit rather than by necessity.
- Its content only changes via a controlled mutation path (an admin
  panel, a content-ingest API, an approval flow) — not via arbitrary
  per-request state.

A route should **stay dynamic** (don't touch it) if it reads:

- `searchParams` — filters, sort, search, pagination. Reading
  `searchParams` anywhere in the page forces the whole route dynamic in
  the App Router's pre-`cacheComponents` model; you cannot partially
  cache around it without adopting Cache Components / PPR, which is a
  much bigger change than this skill covers.
- `cookies()` / `headers()` — session state, personalization, A/B
  bucketing, geo-based content.

Don't "fix" these — forcing them static breaks the feature they exist
for.

### Step 3: Fix

The mechanism differs by route shape — pick the matching sub-step.

#### Step 3a: Dynamic-segment pages — add `generateStaticParams`

For each qualifying route, add a `generateStaticParams` export that
returns every valid value for the dynamic segment(s). Before writing a
new query, **check for an existing "list all slugs" helper** — projects
with a sitemap (`app/sitemap.ts`) almost always already have one, since
the sitemap needs the exact same list of live/public slugs. Reuse it
instead of duplicating the query:

```ts
// app/blog/[slug]/page.tsx
import { listAllSlugs } from "@/lib/queries"; // same helper the sitemap uses

export async function generateStaticParams() {
  const { posts } = await listAllSlugs();
  return posts.map((p) => ({ slug: p.slug }));
}
```

For routes with two dynamic segments (e.g. `[category]/[platform]`
intersection pages), look for an existing helper that already computes
valid combinations (again, often shared with the sitemap) and filter it
the same way the sitemap does — don't invent a different inclusion rule
than the one already used to decide what's indexable.

This turns the route into SSG: prerendered once (at build time for
params known then), served from a static cache on every subsequent
request, and — because `dynamicParams` defaults to `true` — still
resolves correctly for any slug not yet known at build time (rendered
once on first request, then cached).

#### Step 3b: Fixed-path metadata/feed routes — `generateStaticParams` if parameterized, otherwise time-based `revalidate`

Two shapes here:

- **Parameterized metadata files** (`app/[slug]/opengraph-image.tsx`,
  `[slug]/twitter-image.tsx`) take the exact same fix as Step 3a —
  `generateStaticParams`, reusing the same slug-list helper:

  ```ts
  // app/category/[slug]/opengraph-image.tsx
  import { listAllSlugs } from "@/lib/queries";

  export async function generateStaticParams() {
    const { categories } = await listAllSlugs();
    return categories.map((c) => ({ slug: c.slug }));
  }
  ```

- **Fixed-path files** with no dynamic segment (`app/sitemap.ts`,
  `app/robots.ts`, an RSS/Atom `route.ts`) have no params to enumerate —
  replace `force-dynamic`/`revalidate: 0` with a plain time-based
  `export const revalidate = <seconds>` instead:

  ```ts
  // app/sitemap.ts
  // Was: export const dynamic = "force-dynamic"; export const revalidate = 0;
  export const revalidate = 86400; // 1 day — a ceiling, not the real freshness signal (see Step 4)
  ```

  Pick the number based on how often content actually changes, not how
  fresh you'd *like* it to be — a directory that publishes a few times a
  week doesn't need an hourly sitemap re-render just because "fresher
  feels better." Erring toward a longer window is usually fine *if* Step
  4 confirms on-demand revalidation already covers the gap; this number
  is the fallback ceiling for when that mechanism doesn't fire, not the
  primary freshness mechanism.

### Step 4: Confirm on-demand revalidation still covers these pages

Check whether the project already calls `revalidatePath` or
`revalidateTag` from its content-mutation code paths (admin API routes,
ingest/webhook handlers). A common pattern is a single helper like
`revalidatePublic()` called from every mutation, invalidating the whole
public route tree (`revalidatePath("/", "layout")`). If that exists,
newly-static pages are already covered — no extra plumbing needed.

If no such revalidation exists at all, the site was presumably relying
entirely on request-time freshness (which is what made these routes
dynamic in the first place, whether intentionally or not) — flag this to
the user before proceeding, since converting to static without any
revalidation path means content edits won't show up until the next
deploy.

**Don't just read the code — verify it empirically for at least one
converted route**, especially metadata/feed routes (Step 3b), since
whether a blanket `revalidatePath("/", "layout")` actually reaches a
metadata file like `sitemap.ts` (which doesn't render through the app's
normal layout tree the way a page does) isn't obvious from reading the
call alone:

```bash
rm -rf .next && npx next build && npx next start &

curl -s http://localhost:3000/sitemap.xml | grep -c "some-existing-slug-not-yet-added"  # expect 0

curl -s -X POST -H "x-api-token: $TOKEN" -H "content-type: application/json" \
  -d '{"name":"Revalidation Probe","slug":"revalidation-probe", ...}' \
  http://localhost:3000/api/<ingest-endpoint>

curl -s http://localhost:3000/sitemap.xml | grep -c "revalidation-probe"  # expect 1, immediately, no wait
```

If it comes back `1` right away, the time-based `revalidate` window
chosen in Step 3b is confirmed to be just a fallback ceiling, not the
real freshness path — safe to set generously (a day, even a week) since
actual edits invalidate on demand regardless. If it comes back `0`, the
blanket revalidation helper isn't reaching this route — either add an
explicit `revalidatePath` call for it at the mutation site, or keep the
`revalidate` window short enough that staleness stays acceptable. Clean
up the probe row/entry afterward and stop the server.

### Step 5: Audit for staleness regressions on writes that bypass revalidation

This is the step most likely to be skipped, and skipping it turns a cost
win into a visible bug. Search for **public, high-frequency mutation
endpoints** that update data rendered on a page you just made static, but
that don't go through the project's content-revalidation helper — things
like:

- Like/upvote/vote counters
- View counters
- Comment counts
- Any other "user interaction" endpoint, as opposed to "content editing"

These will now serve a stale baked-in value indefinitely, because
nothing tells Next.js the page changed. Fix each one with a **narrowly
scoped** `revalidatePath` call right after the mutation, targeting just
that one page — not the whole site:

```ts
// app/api/posts/[slug]/like/route.ts
import { revalidatePath } from "next/cache";

export async function POST(req, { params }) {
  const { slug } = await params;
  // ... update the like count in the DB ...
  revalidatePath(`/blog/${slug}`); // just this page, not the whole tree
  return Response.json({ ok: true });
}
```

Don't call the blanket "revalidate everything" helper here — that would
regenerate every static page on every vote/like, defeating the purpose
of making them static in the first place.

**Verify the scoping empirically, not just its existence.** Confirming
`revalidatePath` is *called* isn't the same as confirming it only
invalidates the one item you meant — a copy-pasted blanket helper here
would still "work" in the sense of not showing stale data, while quietly
regenerating the whole site on every single vote. The `x-nextjs-cache`
response header (local `next start`) / `x-vercel-cache` header (deployed
on Vercel) proves the scope directly, by checking that the mutated
item's page flips to a fresh render while an *unrelated* item's page
stays untouched:

```bash
rm -rf .next && npx next build && npx next start &

# Before: both pages served from cache
curl -sI http://localhost:3000/items/target-item  | grep -i x-nextjs-cache   # HIT
curl -sI http://localhost:3000/items/control-item | grep -i x-nextjs-cache   # HIT

curl -s -X POST http://localhost:3000/api/items/target-item/vote \
  -H "content-type: application/json" -d '{"value":1}'

# After, immediately, no wait:
curl -sI http://localhost:3000/items/target-item  | grep -i x-nextjs-cache   # expect MISS — regenerated
curl -sI http://localhost:3000/items/control-item | grep -i x-nextjs-cache   # expect HIT  — untouched
```

`target-item` flipping `HIT → MISS` while `control-item` stays `HIT`
confirms the invalidation is scoped exactly as intended. If the control
item *also* flips to `MISS`, the revalidation call is broader than it
should be — go find the blanket helper being called instead of the
narrow one. Undo any DB-level test state this leaves behind (the vote
row, the counter) before moving on, same as the probes in Step 4.

### Step 6: Verify

```bash
rm -rf .next && npx next build
```

Confirm the target routes flipped from `ƒ` to `●` (dynamic-segment pages)
or `○` with `Revalidate`/`Expire` columns populated (fixed-path
metadata/feed routes fixed via Step 3b) — and that routes which should
have stayed dynamic (search, filters) are still `ƒ`. Then type-check:

```bash
npx tsc --noEmit
```

### Step 7: Hard-static mode — cache the query, not just the route (opt-in — ask first)

Steps 1-6 convert whatever CAN become fully static. Some routes
genuinely can't: search, sort/filter/pagination UIs driven by
`searchParams`. Reading `searchParams` anywhere forces the whole route
dynamic regardless of `generateStaticParams` (Step 2) — those routes
will always show `ƒ` in the build table, and that's correct, not a bug
to keep chasing.

But "the route renders dynamically" and "the route hits the database"
are two separate facts. The route can stay `ƒ` while the **query**
behind it is cached — closing the gap between "as static as this route
can be" and "as close to zero DB hits as physically possible."

This is a bigger commitment than Steps 1-6: it means auditing *every*
mutation path in the app, including ones easy to miss (cron jobs
especially), and accepting a small self-healing staleness window as the
fallback if an invalidation path is ever missed. **Don't do this
unprompted.** "Reduce DB usage" does not automatically imply "wrap every
remaining read in a cache layer" — ask first, e.g.: *"The
dynamic listing/search routes still hit the DB on every visit since they
can't be made static — I can cache their underlying queries too so
they're near-zero-touch, at the cost of a small (self-healing)
staleness window if a write path is ever missed. Want me to push it
that far?"*

If yes:

1. **Re-run Step 1's diagnostic across the entire build output**,
   including API routes this time, not just the page/metadata routes
   already touched in Steps 1-6.
2. **Classify every remaining `ƒ` route:**
   - *Legitimate write* (ingest, admin, votes, submissions, cron
     refreshes) — leave untouched. This is exactly the DB activity that
     should remain.
   - *Read that can't be static but CAN be cached* (search, filtered
     listings, anything `searchParams`-driven that queries the DB) —
     candidate for this step.
   - *Genuinely per-request* (session/auth-based, A/B, geo) — leave
     alone, same rule as Step 2.
3. **Wrap the underlying data-fetching function** (not the route
   handler) in `unstable_cache`, reusing the *same tag* as whatever
   content-cache tag Steps 1-6 already established, if one exists — one
   mutation then invalidates everything at once instead of needing
   per-endpoint plumbing:

   ```ts
   // src/db/queries.ts
   export const listItems = unstable_cache(listItemsUncached, ["list-items"], {
     revalidate: 86400, // ceiling, not the real freshness signal
     tags: [CONTENT_TAG],
   });
   ```

   Keep route handlers thin — parse params, call the cached function,
   return the result. The route itself stays `ƒ`; the DB hit is what
   got eliminated, not the render. Deterministic zero-input cases (e.g.
   a search endpoint's "no query yet" default view) are worth caching
   even if the parameterized cases have a long-tail low hit rate — it's
   the same cache layer at no extra cost, and the zero-input case is
   often the most frequently hit one.

   Check **this project's actual `revalidateTag` signature** in
   `node_modules/next/dist/docs` before assuming the single-argument
   form works — some Next.js versions require a second `profile`
   argument (e.g. `revalidateTag(tag, { expire: 0 })` for immediate
   invalidation, matching `revalidatePath`'s existing behavior). Don't
   trust prior Next.js knowledge here; this specific signature has
   changed across versions.

4. **Exhaustively re-run Step 5's mutation audit** — this is where hard
   static mode earns its name, and where it's easiest to leave a real
   gap. Specifically hunt for mutation paths that never needed
   revalidation before because every read used to be live:
   - Cron/scheduled jobs (price refreshes, review syncs, any background
     data pull) — these predate the caching work and are the single most
     commonly missed invalidation source.
   - Any endpoint writing data *joined into* a newly-cached query, even
     if it doesn't touch that query's "own" table — trace what the
     query actually reads, not just what it's named after.
   Wire the same invalidation call into every one you find.
5. **Verify empirically**, same technique as Step 4 — but trigger *each*
   mutation path found in step 4 above, not just the obvious one (e.g.
   an ingest write AND a manual cron run), confirming the cached
   endpoint reflects each one immediately.
6. **Report back a full route-by-route summary**: every route that still
   touches the DB, labeled "legitimate write" or "cached read, ceiling
   Xs." This is the actual deliverable of hard-static mode — not just
   less DB usage, but the user having complete visibility into exactly
   what still touches it and why, so they can sanity-check that nothing
   which should be a write got miscategorized as a cacheable read.

## Examples

**Example:** Game/creator/category directory site on Neon
User says: "How can we use less Neon DB / reduce our database bill?"
Actions:
1. Ran `next build`, found `/games/[slug]`, `/creators/[slug]`,
   `/category/[slug]`, `/tag/[slug]`, `/lists/[slug]` all marked `ƒ`,
   while `/games` and `/platform/[slug]` were also `ƒ` but legitimately
   so (they read `searchParams` for sort/filter).
2. Found an existing `listAllSlugs()` helper already used by
   `app/sitemap.ts`; reused it for `generateStaticParams` on each
   qualifying page instead of writing new queries.
3. Confirmed an existing `revalidatePublic()` helper (called from every
   admin/ingest mutation) already covered invalidation for the newly
   static pages.
4. Found a public `/api/games/[slug]/vote` endpoint that updated a vote
   count shown on the now-static game page but never called
   `revalidatePath`. Added `revalidatePath(`/games/${slug}`)` right
   after the vote write.
5. Rebuilt — target routes flipped from `ƒ` to `●` — and ran `tsc
   --noEmit` clean.
Result: every game/creator/category/tag/list detail page now serves from
a static cache instead of hitting Neon on every visit; Neon compute can
actually scale to zero between content edits and votes, instead of being
kept awake by ordinary page traffic.

**Example:** Same site, follow-up session — Neon still barely idles
User shares a Neon usage graph and asks: "why does our Neon RAM barely
switch off?" — despite the `[slug]`-page work above already being done.
Actions:
1. Re-ran `next build`; nearly every content page was already `●`/`○` as
   expected — the obvious candidates were already fixed.
2. Grepped for `force-dynamic\|revalidate = 0` across `src/app` (Step 1's
   second pass) and found three misses the build table's `ƒ [slug]`
   framing had let slip through on the previous pass: `app/sitemap.ts`
   and two `feed.xml/route.ts` RSS handlers, all `force-dynamic` with a
   DB call and a comment along the lines of "always read fresh — keeps
   build fast." Also found three `[slug]/opengraph-image.tsx` files
   querying the DB per-request with no `generateStaticParams` at all.
3. Fixed the parameterized OG-image routes with `generateStaticParams`
   (Step 3a, reusing the existing slug helper) and the fixed-path
   sitemap/feed routes with `export const revalidate = 86400` (Step 3b).
4. Verified empirically (Step 4): built + started production, created a
   game via the ingest API, and confirmed `sitemap.xml` and
   `games/feed.xml` reflected it on the very next request — proving the
   site's existing `revalidatePublic()` hook already reached these
   routes, so the 1-day number was a safe fallback ceiling, not the real
   freshness path.
Result: the routes hit by crawlers/bots on a predictable schedule
(sitemap re-fetches, RSS polling, link-preview OG-image fetches) — which
had been keeping compute awake far more reliably than actual human
traffic — stopped hitting the DB on every hit. This is exactly the class
of route the intro warns is easy to miss on a first pass, because "check
the `[slug]` rows in the build table" doesn't surface a one-line
`sitemap.ts` sitting in the same table under a totally different name.

**Example:** Same site, third session — pushing to hard-static mode
User confirms the build-table/metadata-route work fixed the Neon graph
(clean suspend/resume cycles now, confirmed via the Neon dashboard's
System Operations tab and `x-vercel-cache: HIT` headers on game pages),
then asks: *"can we optimize it to the ground so the DB barely runs?
only when we add a new game, or someone votes — the rest static?"*
Actions:
1. Asked first, per Step 7's rule, since this is a bigger commitment
   than the baseline pass: confirmed the user wanted the remaining
   `searchParams`-driven listing/search routes cache-wrapped, accepting
   the staleness-ceiling tradeoff.
2. Re-ran Step 1's diagnostic across the *whole* build table this time,
   including API routes. Found `/games` and `/platform/[slug]` still `ƒ`
   (correctly — they read `searchParams` for sort/filter) and
   `/api/search` still `ƒ` running 2-3 uncached queries per request,
   including a fully deterministic "no query yet" default view hit
   every time the search box opens.
3. Classified every remaining `ƒ` route: ~15 were legitimate writes
   (ingest, admin, votes, submissions, the price-refresh cron) — left
   untouched. `/games`, `/platform/[slug]`, `/api/search` were reads
   that could be cached.
4. Wrapped the underlying query functions (`listGames`,
   `listCategories`, `getPlatformBySlug`, two combo-filter helpers, and
   the search handler's core logic) in `unstable_cache`, all sharing one
   tag, `revalidate: 86400` ceiling.
5. Step 5's mutation audit found the real gap: the twice-weekly
   price/review-refresh cron had *never* called any revalidation at all
   (it predated the whole caching effort, since previously every read
   was live) — so price updates would have silently never reached either
   the cached queries or the already-static game detail pages. Added the
   project's central revalidation call to the cron route.
6. Hit this project's actual `revalidateTag` signature mismatch —
   checked `node_modules/next/dist/docs` rather than assuming stable
   Next.js behavior, and found this version requires a `profile` second
   argument; used `{ expire: 0 }` to match `revalidatePath`'s existing
   immediate-invalidation semantics.
7. Verified empirically: built + started production, created a game via
   the ingest API, confirmed `/games` reflected it on the very next
   request with no wait.
8. Reported back a full categorized list of every route still touching
   the DB — writes vs. cached reads — so the user could see exactly what
   remained and confirm nothing was miscategorized.
Result: the only things touching Neon on an ordinary visit are the
deliberate writes the user named (add a game, cast a vote) plus a couple
of others just as legitimate (submissions, the price cron) — every read
path, including the ones that can't be made static, now serves from
cache until something actually changes.

**Example:** Proving an upvote endpoint's revalidation is correctly
scoped (any project — this pattern isn't specific to any one site)
User has a directory/marketplace/blog app where item detail pages were
just converted to static (Step 3a) and asks: "does upvoting still show
up right away, and does it only regenerate the one page I voted on?"
Actions:
1. Read `app/api/items/[slug]/vote/route.ts`: it already calls
   `revalidatePath('/items/${slug}')` right after writing the vote —
   looked correctly scoped, but "looks right" isn't proof.
2. Ran the Step 5 verification recipe locally: built + started
   production, fetched a target item and an unrelated control item
   (both `x-nextjs-cache: HIT`), cast a vote on the target item only,
   then re-fetched both immediately with no wait.
3. Target item: `HIT → MISS`, and the embedded vote count in the HTML
   went from 0 to 1. Control item: stayed `HIT`, unchanged — confirming
   the `revalidatePath` call regenerates exactly one page, not the whole
   site's static cache.
4. Cleaned up the test vote row and counter in the dev database
   afterward, same as any other local probe.
Result: verified — not assumed — that voting is both instant (no cron,
no daily ceiling, no stale cache) and cheap (one page regenerates, not
hundreds). This is the general pattern for any per-item mutation on an
otherwise-static list: likes, votes, stock/availability counters,
comment counts — the same `HIT`-target/`HIT`-control check applies
regardless of what the counter actually represents.

## Troubleshooting

**Error:** A route stays `ƒ` even after adding `generateStaticParams`
**Cause:** It's also reading `searchParams`, `cookies()`, or `headers()`
somewhere in the page or a component it renders — any one of these
forces the whole route dynamic regardless of `generateStaticParams`.
**Solution:** Either accept it staying dynamic (if the feature needs it),
or move the dynamic-API-dependent part into a `<Suspense>` boundary if
the project uses Next's Cache Components / PPR model — that's a larger
architectural change outside this skill's scope.

**Error:** Build takes much longer or times out after adding
`generateStaticParams`
**Cause:** The slug-listing query (or the number of params it returns)
is large or expensive, and it's now running once per build instead of
being deferred to request time.
**Solution:** Make sure the underlying query is efficient (proper
indexes, no N+1). If the dataset is very large, consider returning an
empty array from `generateStaticParams` (so nothing prerenders at build
time) and relying entirely on on-demand generation + caching after first
request — still far better than fully dynamic on every request.

**Error:** Content edits don't show up after publishing
**Cause:** No revalidation path was wired up before this skill's changes
(Step 4 flagged this), or the mutation endpoint that changed the content
doesn't call `revalidatePath`/`revalidateTag`.
**Solution:** Add the missing revalidation call at the mutation site,
scoped to the affected path(s).
