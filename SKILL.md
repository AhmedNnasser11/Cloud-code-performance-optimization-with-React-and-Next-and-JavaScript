---
name: nextjs-react-performance
description: Comprehensive guidance for diagnosing and fixing performance problems in Next.js and React front-ends — rendering strategy (SSR/SSG/ISR/CSR/RSC), unnecessary re-renders, incorrect or unnecessary useEffect usage, hook misuse, bundle size, Core Web Vitals (LCP/INP/CLS/TTFB), and general JavaScript runtime performance. Use this skill whenever the user asks to optimize, speed up, profile, or review the performance of a React or Next.js app or component, whenever they paste a component and ask "why is this slow / re-rendering / laggy", whenever they ask about useEffect necessity or dependency arrays, whenever they ask how to avoid unnecessary re-renders or prop drilling causing renders, whenever they ask about Core Web Vitals or Lighthouse scores, or whenever they're deciding between SSR/SSG/ISR/CSR/RSC. Trigger even if the user doesn't say the word "performance" explicitly — e.g. "this page feels laggy", "my component re-renders too much", "should I use useEffect here", "how do I speed up my Next.js site".
---

# Next.js / React Performance

Use this skill to audit, explain, or fix performance in React and Next.js codebases. It combines four
knowledge areas — load a reference file only when that area is actually relevant to the task, don't
dump all of them into a response.

| Reference file | Load when the task involves... |
|---|---|
| `references/rendering-strategies.md` | Choosing/explaining SSR, SSG, ISR, CSR, RSC, streaming, hydration |
| `references/react-hooks-and-rerenders.md` | `useEffect` review, "why is this slow", unnecessary re-renders, `children` prop, `memo`/`useMemo`/`useCallback`, any hook correctness question |
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
