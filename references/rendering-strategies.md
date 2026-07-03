# Rendering Strategies: SSR / SSG / ISR / CSR / RSC + Streaming

## Decision table

| Strategy | Perf impact | Complexity | Use when |
|---|---|---|---|
| **SSG** (Static, `getStaticProps` / static Server Components) | Fastest TTFB/LCP, served from CDN | Low-medium (build step) | Content is the same for every visitor and doesn't need to change per-request (marketing pages, blogs, docs). Stale until rebuild. |
| **ISR** (`revalidate`) | Near-SSG speed, bounded staleness | Medium | Same as SSG but content changes periodically (product listings, articles) — best default for "mostly static, occasionally updated." First request after staleness pays a rebuild cost (cache MISS), then served fast again. |
| **SSR** (`getServerSideProps` / dynamic Server Components) | Fast first paint, but per-request server cost + hydration delay | Medium (server cost) | Content must be fresh or personalized on every request (auth-gated dashboards, per-user data, request-dependent SEO pages). |
| **CSR** (client fetch, e.g. `useEffect`/SWR) | Slowest FCP/LCP (blank shell until JS+data load), hurts SEO | Low to build | Highly interactive, behind-auth app sections where SEO doesn't matter and fast *subsequent* client-side navigation matters more than first load. |
| **RSC + streaming** (Next.js App Router default) | Fast LCP for the static shell, progressive fill-in, smaller client JS | High (requires Suspense-first data design) | Data-heavy pages where you want server rendering's speed without shipping all the data-fetching JS to the client. Needs to be adopted deliberately — mixing client/server components without `<Suspense>` boundaries gives little benefit. |

## How to choose

Ask: **does this page's content need to be different per request, and does it need to be fresh at
request time?**
- No, and rarely changes → SSG
- No, but changes periodically → ISR
- Yes (per-user/auth/highly dynamic) → SSR, or RSC with `export const dynamic = 'force-dynamic'`
- Content is client-only, behind auth, SEO irrelevant → CSR is acceptable, but still prefer RSC for
  the shell and CSR only for the truly interactive island.

## Streaming & hydration mechanics (why RSC feels fast)

1. Static shell (layout, nav, `<Suspense>` fallbacks) is sent immediately, often from CDN.
2. Server streams additional HTML chunks as each Suspense boundary's data becomes ready.
3. **Selective hydration**: each streamed chunk hydrates independently, so the browser doesn't have to
   wait for the entire page's JS before any part becomes interactive.
4. Without Suspense boundaries around slow data, you lose this benefit — the page blocks on the
   slowest piece.

**The "no-JS gap":** between HTML paint and hydration completing, event handlers are inert. Streaming/
selective hydration shrinks this gap by hydrating in pieces instead of all-at-once.

## Practical Next.js App Router rules

- Default to Server Components; add `'use client'` only where you need interactivity, browser APIs,
  or hooks like `useState`/`useEffect`. Client components ship JS to the browser — Server Components
  do not.
- Wrap slow/data-dependent sections in `<Suspense fallback={...}>` so they stream independently
  instead of blocking the whole page.
- Mark route behavior explicitly when needed: `export const dynamic = 'force-static'` or
  `'force-dynamic'`.
- Don't nest a client component boundary higher than necessary — a `'use client'` at the top of a
  large tree forces everything below it into the client bundle even if most of it doesn't need to be.

## Migration note

Moving a legacy SSR (Pages Router, `getServerSideProps`) app to App Router/RSC is not a drop-in
change — it requires rethinking data fetching as server-first and adding Suspense boundaries. Done
half-way (client/server components mixed without Suspense), it can leave you with the download cost
of CSR and none of RSC's streaming benefit.
