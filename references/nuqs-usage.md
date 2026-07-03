# nuqs — Type-Safe URL Search-Param State

`nuqs` (~5.5 KB gzipped, zero runtime deps) makes URL query parameters behave like `useState`, with
the URL as the source of truth. Works in Next.js (App and Pages Router), plain React SPAs, React
Router v6–v8, TanStack Router, via framework-specific adapters.

## When to reach for it

Use `nuqs` for state that should be shareable/bookmarkable/back-forward-navigable and is small:
filters, search queries, pagination, sort order, active tab, map coordinates, toggles. **Don't** put
large objects, secrets, or anything not meant to be public in the URL — see Limits below.

## Setup

```bash
npm install nuqs
```

Wrap the app in the adapter for your framework — every `nuqs` hook must be under it.

```tsx
// Next.js App Router — app/layout.tsx
import { NuqsAdapter } from 'nuqs/adapters/next/app';
export default function RootLayout({ children }) {
  return <html><body><NuqsAdapter>{children}</NuqsAdapter></body></html>;
}
```
```tsx
// Next.js Pages Router — pages/_app.tsx
import { NuqsAdapter } from 'nuqs/adapters/next/pages';
export default function MyApp({ Component, pageProps }) {
  return <NuqsAdapter><Component {...pageProps} /></NuqsAdapter>;
}
```
```tsx
// Plain React SPA
import { NuqsAdapter } from 'nuqs/adapters/react';
createRoot(el).render(<NuqsAdapter><App /></NuqsAdapter>);
```
Requires Next.js ≥14.2, React 18/19. Legacy `nuqs@^1` exists for older Next.js. It's ESM-only.

## Core hooks

```tsx
'use client';
import { useQueryState, parseAsInteger, parseAsBoolean } from 'nuqs';

const [count, setCount] = useQueryState('count', parseAsInteger.withDefault(0));
const [dark, setDark] = useQueryState('dark', parseAsBoolean.withDefault(false));
```
`useQueryState(key, parser)` → `[value, setValue]`, synced with `?key=value`. No parser given = plain
string. Setting to `null` removes the param from the URL.

```tsx
import { useQueryStates, parseAsFloat } from 'nuqs';

const [{ lat, lng }, setCoords] = useQueryStates(
  { lat: parseAsFloat.withDefault(0), lng: parseAsFloat.withDefault(0) },
  { history: 'push' }
);
setCoords({ lat: 12.3, lng: 45.6 }); // batched into ONE URL update
```
`useQueryStates(parsersObj, options)` manages multiple keys as one unit — multiple `set` calls in the
same tick (even across different hooks) are batched into a single URL update.

**Built-in parsers**: `parseAsInteger`, `parseAsFloat`, `parseAsBoolean`, `parseAsIsoDate`,
`parseAsTimestamp`, `parseAsStringLiteral([...])` (enums), `parseAsJson(schema)`,
`parseAsArrayOf(parser)`. Custom parsers via `createParser`.

## Server-side (`nuqs/server`) — avoids `'use client'` restriction

```ts
// searchParams.ts — shared between server and client
import { createLoader, parseAsString, parseAsInteger } from 'nuqs/server';
export const searchParamsLoader = createLoader({
  q: parseAsString.withDefault(''),
  page: parseAsInteger.withDefault(1),
});
```
```tsx
// app/page.tsx — Server Component, no client JS cost for this part
export default async function Page({ searchParams }) {
  const { q, page } = await searchParamsLoader(searchParams);
  const results = await fetchSearchResults(q, page); // fresh per request
  return <ResultsList results={results} />;
}
```
For deeper trees, `createSearchParamsCache(parsersObj)` lets a root Server Component call
`cache.parse(searchParams)` once, then any descendant Server Component calls `cache.get(key)` — no
prop drilling, no re-parsing.

## The performance-critical default: shallow updates

By default, `nuqs` updates are **shallow**: client-only, `history.replaceState`, **no server
round-trip, no re-render of Server Components**. This is the main reason to prefer `nuqs` over
manually calling `router.push()` for filter-type state in Next.js — you get URL-synced state without
paying for a navigation on every keystroke/click.

- `shallow: true` (default): update URL, don't touch the server. Use for anything that's purely a
  client-rendering concern (toggling a client-rendered panel, a tab that only affects client state).
- `shallow: false`: forces a real navigation so Server Components / `getServerSideProps` / loaders
  re-run with the new params. Necessary whenever the new param value should change server-fetched
  data (e.g. `page`/`q` driving a server-side search). Combine with React's `useTransition` (pass
  `startTransition` in options) to get a pending state instead of a jank navigation.

```tsx
const [isPending, startTransition] = useTransition();
const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1)
  .withOptions({ shallow: false, startTransition }));
```

## Rate-limiting URL updates — don't skip this for high-frequency inputs

Every update by default is throttled (~50ms, ~100–120ms on Safari) to respect browser History API
rate limits. For known high-frequency sources, tune it explicitly rather than relying on the default:

```tsx
import { useQueryState, parseAsString } from 'nuqs';
// search-as-you-type: only the final value matters → debounce
const [q, setQ] = useQueryState('q', parseAsString.withDefault('')
  .withOptions({ limitUrlUpdates: { method: 'debounce', timeMs: 300 } }));

// a slider where intermediate positions are meaningful → throttle
const [x, setX] = useQueryState('x', parseAsInteger.withDefault(0)
  .withOptions({ limitUrlUpdates: { method: 'throttle', timeMs: 100 } }));
```
**Debounce** = wait until input pauses, send once. **Throttle** = send at most once per interval,
including intermediate values. Wrong choice here is the same class of bug as the general
debounce-vs-throttle mistake in `js-runtime-performance.md` — pick based on whether intermediate
values matter.

## Other options that affect cost

| Option | Default | Note |
|---|---|---|
| `history` | `'replace'` | `'push'` adds a browser-history entry per update — use only for state that's genuinely page-like (tabs, wizard steps), never globally for every field, or you pollute back/forward navigation. |
| `scroll` | `false` | `true` scrolls to top like a real navigation — usually wrong for shallow filter updates. |
| `clearOnDefault` | `true` (v2) | Setting state back to its default removes the param from the URL — keeps URLs shorter/cleaner. |
| `urlKeys` | none | Map verbose internal names to short URL keys (`{ latitude: 'lat' }`) — reduces URL length, helps stay under the practical ~2,000-character URL limit. |

## Serializer (no React needed — for building `<Link href>`s, API calls)

```ts
import { createSerializer, parseAsInteger } from 'nuqs';
const serialize = createSerializer({ page: parseAsInteger.withDefault(1) });
const href = serialize('/items?sort=asc', { page: 5 }); // "/items?sort=asc&page=5"
```

## Why this beats the common hand-rolled alternative

```tsx
// ❌ Common anti-pattern: duplicate state + Effect to keep it in sync with the URL
const [page, setPage] = useState(1);
useEffect(() => {
  const params = new URLSearchParams(window.location.search);
  params.set('page', page);
  router.replace(`?${params.toString()}`);
}, [page]);
// ...plus another Effect to read the initial value from the URL on mount, plus no back/forward
// button support, plus an extra render + Effect firing on every page change.

// ✅ nuqs: the URL *is* the state — no duplicate state, no Effect, no extra render pass
const [page, setPage] = useQueryState('page', parseAsInteger.withDefault(1));
```
This is a direct instance of the "don't `useEffect` to derive/sync state" principle from
`react-hooks-and-rerenders.md` §1 — nuqs makes the URL the single source of truth instead of a second
copy of state that needs manual syncing.

## When to reach for nuqs vs. plain `useState`/Zustand

| State | Prefer |
|---|---|
| Shareable via URL / bookmarkable / back-button-able (filters, pagination, tabs, search query) | nuqs |
| Should affect server-rendered data on change | nuqs with `shallow: false`, parsed server-side via `createSearchParamsCache`/`createLoader` |
| Purely client-side UI state with no reason to persist across reload/share (a modal's open/closed state, hover state) | plain `useState` |
| Global app state unrelated to the current route/URL (cart contents, auth session) | Zustand/Context (see `zustand-usage.md`) — wrong tool for non-URL-shaped state |

## Practical checklist

- [ ] Keep `shallow: true` (default) unless the param must drive a server refetch.
- [ ] Debounce free-text inputs (search boxes); throttle continuous inputs (sliders, drag).
- [ ] Use `history: 'push'` only for page-like/step-like state, never as a global default.
- [ ] Use `.withDefault()` so consuming code doesn't need `null` checks everywhere.
- [ ] Use `urlKeys` to shorten param names once you have more than a couple of filters.
- [ ] Don't put large objects, arrays of many items, or secrets in URL state — move those to a proper
      store (Zustand/Context/server) and keep only small, shareable values in the URL.
- [ ] Pair `shallow: false` updates with `useTransition`'s `startTransition` for a pending UI instead
      of a jank navigation.
- [ ] For RSC trees with many consumers of the same params, use `createSearchParamsCache` instead of
      prop-drilling parsed values down.
