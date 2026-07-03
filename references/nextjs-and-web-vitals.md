# Next.js APIs, Core Web Vitals, and Profiling

## Core Web Vitals (MDN definitions, current thresholds)

| Metric | Measures | Good threshold | Main levers |
|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Time to render the largest visible element | ≤ 2.5s | Fast TTFB (SSG/ISR/CDN), preload the LCP image/font, avoid render-blocking resources, prioritize the LCP image (`priority` in `next/image`) |
| **INP** (Interaction to Next Paint) | Delay from any interaction to the next visual update — replaced FID | < 200ms | Reduce main-thread JS work, break up long tasks, `startTransition` for non-urgent updates, avoid heavy synchronous handlers |
| **CLS** (Cumulative Layout Shift) | Unexpected layout movement | < 0.1 | Reserve space via `width`/`height`/aspect-ratio on images/embeds, avoid injecting content above existing content, use skeletons with fixed dimensions |
| **TTFB** (Time to First Byte) | Server response latency | Lower is better; not a Core Web Vital itself but gates everything else | CDN, SSG/ISR over SSR where possible, efficient server code |

## Next.js built-in optimizations

- **`next/image`**: auto responsive `srcset`, lazy-loads offscreen images by default, generates
  modern formats. Use the `priority` prop only on the actual LCP image (forces eager load + preload)
  — using it on multiple images defeats its purpose.
- **`next/font`**: self-hosts and inlines font files, avoids extra network round-trip to Google Fonts,
  sets sensible `font-display` defaults. Only load the weights/styles you actually use.
- **`next/dynamic`** (or `React.lazy` + `Suspense`): code-split heavy/rarely-used components
  (charts, maps, modals) so they aren't in the initial bundle.
  ```jsx
  const HeavyChart = dynamic(() => import('heavy-chart-lib'), { ssr: false });
  ```
- **`<Script>`** component: control loading strategy for third-party scripts —
  `strategy="lazyOnload"` for non-critical analytics/widgets, `strategy="afterInteractive"` for
  things needed soon but not blocking, `beforeInteractive` only for scripts that truly must run
  before hydration.
- **Automatic code-splitting**: each page/route only ships the JS it needs; further split with
  dynamic imports for large conditionally-rendered pieces.
- **Bundling**: Turbopack (default in recent Next.js) or Webpack — both tree-shake and split chunks.
  Use `@next/bundle-analyzer` to find bloat and duplicate dependencies.

## Resource hints & network

- `<link rel="preconnect" href="...">` — warm up DNS/TCP/TLS for a critical third-party origin.
- `<link rel="preload" as="font"|"style"|"script">` — fetch a known-critical resource early.
- `<link rel="dns-prefetch">` — cheaper, DNS-only hint for less-critical third-party hosts.
- Serve over HTTP/2 or HTTP/3 (multiplexed, header-compressed) — most CDNs/hosts do this by default.
- Use `stale-while-revalidate` caching for API responses that can tolerate brief staleness.

## Profiling & measurement tools

- **React DevTools Profiler** / `<Profiler>` API — shows `actualDuration` per component render, finds
  which components are expensive or re-rendering unexpectedly. Disabled in production builds by
  default (profiling adds overhead).
- **Chrome DevTools → Performance panel** — flame chart of CPU/paint/layout, flags long tasks (>50ms).
- **Lighthouse** / PageSpeed Insights — automated Core Web Vitals + best-practice audits.
- **WebPageTest** — network/CPU throttled testing (e.g. "Slow 4G", 6x CPU slowdown), waterfalls,
  filmstrips — good for realistic mobile conditions.
- **Real User Monitoring (RUM)** — Vercel Speed Insights, `web-vitals` JS library, or your analytics
  provider — measures actual users' Core Web Vitals in production, which Lighthouse (lab data) can
  diverge from. Don't over-optimize for a Lighthouse score at the expense of real-user metrics.

## Benchmarking methodology

- Always test a production build (`next build && next start`, or deployed), never `next dev` — dev
  mode is intentionally unoptimized (no minification, extra instrumentation).
- Test both cold (empty cache) and warm (cache primed) loads — they represent different real users.
- Throttle CPU and network to simulate real device/connection conditions, not just your dev machine.
- Take the median or p75 across multiple runs — single runs are noisy.
