# Next.js App Router Project Patterns

File/folder conventions in `app/` and what they're for — relevant to performance because several of
these directly control what gets bundled, prefetched, or streamed.

## Private folders (`_folderName`)

Prefixing a folder with `_` opts it out of routing — Next.js won't treat it as a route segment, so you
can colocate components, hooks, utils, and tests next to the routes that use them without accidentally
creating a URL for them.

```
app/
  dashboard/
    _components/
      RevenueChart.jsx
      UserTable.jsx
    _hooks/
      useDashboardData.js
    page.jsx
```

**Why it matters for performance**: colocation makes it obvious which components are only used by one
route, which makes it safe to `next/dynamic`-import them from that route alone rather than accidentally
sharing (and bundling into) a common chunk used by unrelated routes.

## Route groups (`(folderName)`)

Parentheses create a folder that organizes routes **without** affecting the URL path — used to apply a
different layout to a subset of routes, or just to group related routes in the file tree.

```
app/
  (marketing)/
    layout.jsx        // lightweight layout: nav, footer, no heavy client JS
    page.jsx           // "/"
    about/page.jsx      // "/about"
  (app)/
    layout.jsx        // app shell: sidebar, auth check, heavier client JS
    dashboard/page.jsx  // "/dashboard"
```

**Why it matters for performance**: this is the standard way to keep your marketing/public pages on a
minimal layout (fast, mostly static, small JS) while your authenticated app section has a heavier
layout (auth, client state, bigger sidebar) — without one layout's JS/CSS bleeding into the other.

## Parallel routes (`@slotName`)

Render multiple independent pages in the same layout simultaneously (e.g. a dashboard with an
`@analytics` slot and a `@team` slot that load independently).

```
app/
  dashboard/
    layout.jsx          // receives {children, analytics, team} props
    @analytics/page.jsx
    @team/page.jsx
    page.jsx             // children
```

**Why it matters for performance**: each slot can have its own `loading.jsx`, so slow sections stream
in independently instead of one slow data source blocking the whole dashboard — this is essentially
Suspense boundaries expressed at the routing level.

## Intercepting routes (`(.)`, `(..)`, `(..)(..)`, `(...)`)

Show a route in a different context depending on navigation origin — the classic case is a photo
modal: clicking a photo from a feed opens it as a modal (intercepted), but a direct URL visit or
refresh renders the full page.

```
app/
  feed/page.jsx
  photo/[id]/page.jsx           // full page version
  feed/(.)photo/[id]/page.jsx   // intercepted: renders as modal on top of feed
```

**Why it matters for performance**: the modal version can be a much lighter render (skip the full
page's data/layout) since it's shown on top of already-loaded content, only falling back to the full,
heavier page when there's no client-side navigation context to intercept from (direct load/refresh).

## Layouts vs. templates

- **`layout.jsx`**: persists across navigations within its segment — state is preserved, it does not
  re-render on navigation between child routes. Use for shared chrome (nav, sidebar).
- **`template.jsx`**: same shape as layout but **remounts** on every navigation — use only when you
  specifically need fresh state/effects per navigation (e.g. re-triggering an enter animation). Don't
  reach for `template.jsx` by default — it re-runs effects and loses state on every navigation, which
  is usually a performance/UX regression compared to `layout.jsx`.

## `loading.jsx` / `error.jsx` / `not-found.jsx`

Automatic Suspense/error boundaries per route segment.
- `loading.jsx` wraps the segment in `<Suspense>` automatically — you get streaming for free without
  manually wrapping the page content.
- Prefer segment-level `loading.jsx` over a single global spinner — it lets unrelated parts of the
  layout (nav, sidebar) render immediately while only the slow segment shows its own fallback.

## Route Handlers (`route.js`) vs. Server Actions

- **Route Handlers** (`app/api/.../route.js`): traditional request/response API endpoints — needed
  when a non-React client (webhook, external service, mobile app) needs to call your backend.
  - **Server Actions** (`'use server'` functions called directly from components/forms): avoid a
  round-trip API layer for form submissions and mutations from your own UI — smaller client JS (no
  manual `fetch` + serialization boilerplate) and integrate with `useActionState`/`useOptimistic` for
  pending/optimistic UI. Prefer Server Actions for first-party mutations; keep Route Handlers for
  external/public API surface.

## `middleware.js`

Runs before a request completes, at the edge — used for auth redirects, A/B routing, locale
detection. Keep it minimal: it runs on every matched request, so heavy logic here adds latency to
every navigation, not just one page. Scope it with a `matcher` config to only the paths that need it.

## General colocation vs. shared code

- Code used by exactly one route → colocate in that route's `_folder`.
- Code used by multiple routes → lift to a shared `components/`, `lib/`, or `hooks/` directory at a
  level common to those routes (or the project root) — but don't lift prematurely; duplicated small
  pieces are often cheaper than a shared abstraction that couples unrelated routes' bundles together.
