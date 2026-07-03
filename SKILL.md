---
name: nextjs-react-performance
description: Guidance for diagnosing and fixing performance problems in Next.js/React front-ends — rendering strategy (SSR/SSG/ISR/CSR/RSC), unnecessary re-renders, useEffect misuse, every React hook (including useSyncExternalStore, useEffectEvent, use, useActionState, useOptimistic), design patterns (HOC, Compound Components, Render Props, Provider, Controlled/Uncontrolled), Next.js App Router structure (private folders, route groups, parallel/intercepting routes, layouts vs templates, Server Actions, middleware), Zustand (selectors, slices, SSR-safe stores), nuqs URL search-params state (client + server), bundle size, Core Web Vitals (LCP/INP/CLS/TTFB), and JS runtime performance. Also trigger for performance/speed/laggy complaints, pasted components with "why is this re-rendering", useEffect questions, hook correctness, "HOC vs hook", folder structure, Zustand or nuqs usage, or SSR/SSG/ISR/CSR/RSC choice — even without the word "performance".
---

# Next.js / React Performance

Use this skill to audit, explain, or fix performance in React and Next.js codebases. It combines four
knowledge areas — load a reference file only when that area is actually relevant to the task, don't
dump all of them into a response.

| Reference file | Load when the task involves... |
|---|---|
| `references/rendering-strategies.md` | Choosing/explaining SSR, SSG, ISR, CSR, RSC, streaming, hydration |
| `references/react-hooks-and-rerenders.md` | `useEffect` review, "why is this slow", unnecessary re-renders, `children` prop, `memo`/`useMemo`/`useCallback`, any hook correctness question (including `useSyncExternalStore`, `useEffectEvent`, `use`, `useActionState`, `useOptimistic`, etc.) |
| `references/react-component-patterns.md` | HOC, Compound Components, Render Props, Provider pattern, Container/Presentational, Controlled vs. Uncontrolled — "which pattern should I use", or reviewing code that uses one of these |
| `references/nextjs-project-patterns.md` | App Router file/folder conventions — private folders (`_folder`), route groups, parallel routes (`@slot`), intercepting routes, `layout` vs `template`, `loading`/`error` boundaries, Route Handlers vs Server Actions, `middleware.js` |
| `references/zustand-usage.md` | Zustand store setup/review, selector usage, avoiding over-subscription, SSR/Next.js store patterns, "Zustand vs Context" |
| `references/nuqs-usage.md` | URL search-param state — `nuqs` setup, `useQueryState`/`useQueryStates`, shallow vs server-syncing updates, throttle/debounce for URL updates, server-side loaders/cache |
| `references/js-runtime-performance.md` | Raw JS perf — array/string/object method costs, event loop, long tasks, GC, debounce/throttle, Web Workers |
| `references/nextjs-and-web-vitals.md` | Next.js-specific APIs (`next/image`, `next/font`, `next/dynamic`, `<Script>`), Core Web Vitals (LCP/INP/CLS/TTFB), MDN performance concepts, profiling tools |
| `references/checklist-and-patterns.md` | Final review pass, "give me a checklist", before/after code pattern examples |

## How to work a request

1. **Classify the ask.** Is this (a) reviewing real code the user pasted/uploaded, (b) a conceptual
   question (e.g. "SSR vs ISR?"), or (c) "just make my app fast" (broad audit)? This determines which
   reference file(s) to open — usually 1-2, rarely all of them.

2. **If the user pasted a component or file, actually read it before answering.** Don't give generic
   advice. Look specifically for the patterns flagged in `react-hooks-and-rerenders.md`:
   - `useEffect` calls that exist to sync/derive state that could just be computed during render
   - Effects that exist purely to respond to an event (should be in the event handler instead)
   - Missing/incorrect dependency arrays
   - New object/array/function literals passed as props on every render (breaks `memo`)
   - Components not accepting `children` where composition would avoid a re-render cascade
   - Expensive work with no `useMemo`/`useCallback`, or `useMemo`/`useCallback` used where it's not needed
   - Client components (`'use client'`) wrapping content that could stay server-rendered

3. **Always give the "why," not just the "do this."** State the concrete mechanism (e.g. "this
   re-renders every child because a new `{}` object literal is created each render, defeating
   `React.memo`'s shallow comparison") — one sentence is enough, don't lecture.

4. **Prefer the smallest fix that solves the actual bottleneck.** Don't reflexively suggest
   `useMemo`/`useCallback`/`memo` everywhere — over-memoization has its own cost (see
   `react-hooks-and-rerenders.md` § When NOT to memoize). Measure-first framing: if the user hasn't
   profiled, suggest React Profiler or Chrome DevTools Performance before assuming where the cost is.

5. **For Next.js rendering-mode questions**, don't default to "just use SSG" — ask what the actual
   data freshness requirement is (or infer it from context) and map it to
   `rendering-strategies.md`'s SSR/SSG/ISR/CSR/RSC trade-off table.

6. **Knowledge freshness caveat**: this skill's reference material reflects React/Next.js guidance as
   of Claude's training data (through Jan 2026). React and Next.js APIs move fast. If the user's
   installed version behaves differently than described, or they mention a newer API, trust the
   user/their code and suggest they check react.dev / nextjs.org directly if precision matters (e.g.
   version-specific behavior, a very recent RFC/release).

## Output format

- For code review: inline the fixed snippet directly under the problem, don't make the user diff two
  huge blocks. Keep prose between snippets short.
- For conceptual questions: prose is fine, use the comparison tables in the reference files rather
  than re-deriving them.
- Don't create a file/artifact for a normal Q&A exchange — only produce a file if the user is asking
  for a document (e.g. "write up our performance findings" or "give me a checklist doc").
