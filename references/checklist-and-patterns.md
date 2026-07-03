# Consolidated Checklist & Code Patterns

Use this as a final pass, or when the user explicitly wants a checklist/audit summary.

## Checklist

**Rendering**
- [ ] Right strategy per route: SSG/ISR for mostly-static, SSR only where freshness is required,
      RSC + Suspense as the default shape, CSR limited to truly client-only interactive islands.
- [ ] `'use client'` boundaries pushed as low in the tree as possible.
- [ ] Slow data-dependent sections wrapped in `<Suspense>` so they stream independently.

**React re-renders**
- [ ] No `useEffect` used to derive state that could be computed during render (see
      `react-hooks-and-rerenders.md` §1).
- [ ] No `useEffect` reacting to a user-triggered flag — logic moved into the event handler.
- [ ] Object/array/function literals passed as props are stable (`useMemo`/`useCallback`) where the
      child is `memo`-wrapped and this matters.
- [ ] Expensive, state-owning components pass unrelated expensive subtrees down via `children` rather
      than re-rendering them on every state change.
- [ ] Context split by concern; not one giant context object.
- [ ] List keys are stable/unique, not array index (unless list never reorders/filters).
- [ ] Memoization used where profiling showed a real cost — not applied speculatively everywhere.

**Bundle / JS**
- [ ] Heavy/rarely-used components behind `next/dynamic` or `React.lazy`.
- [ ] Named ES module imports (`import { x } from 'lib'`) instead of whole-library imports.
- [ ] Bundle analyzer run at least once to catch duplicate/oversized dependencies.
- [ ] Third-party scripts loaded via `<Script strategy="lazyOnload">` or equivalent, not
      render-blocking `<script>` tags.
- [ ] Long synchronous tasks (>50ms) broken up or moved to a Web Worker.

**Images / fonts / CSS**
- [ ] `next/image` used everywhere instead of raw `<img>`; `priority` only on the actual LCP image.
- [ ] `next/font` used with only the needed weights.
- [ ] Critical CSS inlined / non-critical CSS deferred.
- [ ] Every image/embed has explicit dimensions or aspect-ratio to prevent CLS.

**Network**
- [ ] CDN + appropriate cache headers for static assets and SSG/ISR pages.
- [ ] `preconnect`/`preload` for known-critical origins/assets.
- [ ] HTTP/2 or HTTP/3 in use.

**Measurement**
- [ ] Tested against a production build, not `next dev`.
- [ ] Core Web Vitals checked against thresholds (LCP ≤2.5s, INP <200ms, CLS <0.1) via Lighthouse
      *and* RUM (lab vs field data can diverge).
- [ ] React Profiler used to confirm which component is actually expensive before optimizing it.

## Before/after patterns

**Effect-as-derived-state → computed value**
```jsx
// ❌
const [fullName, setFullName] = useState('');
useEffect(() => setFullName(`${first} ${last}`), [first, last]);

// ✅
const fullName = `${first} ${last}`;
```

**CSR data fetch on a static/public page → SSG/ISR**
```jsx
// ❌
export default function Page() {
  const [data, setData] = useState(null);
  useEffect(() => { fetch('/api/data').then(r => r.json()).then(setData); }, []);
  return <div>{data ? data.title : 'Loading...'}</div>;
}

// ✅
export const getStaticProps = async () => {
  const data = await fetch('https://api.example.com/data').then(r => r.json());
  return { props: { data }, revalidate: 3600 }; // ISR
};
export default function Page({ data }) {
  return <div>{data.title}</div>;
}
```

**Whole-library import → dynamic, code-split import**
```jsx
// ❌
import Chart from 'heavy-chart-library';

// ✅
const Chart = dynamic(() => import('heavy-chart-library'), {
  ssr: false,
  loading: () => <p>Loading chart...</p>,
});
```

**Unoptimized image → `next/image`**
```jsx
// ❌
<img src="/hero.jpg" alt="Hero" width={1200} height={800} />

// ✅
<Image src="/hero.jpg" alt="Hero" width={1200} height={800} priority />
```

**State-owning parent re-rendering an unrelated expensive tree → `children` pattern**
```jsx
// ❌
function Page() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <ExpensiveTree />
    </>
  );
}

// ✅
function CounterShell({ children }) {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children}
    </>
  );
}
// usage: <CounterShell><ExpensiveTree /></CounterShell>
```
