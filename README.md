# Cloud Code Performance Optimization with React, Next.js, and JavaScript

AI skill pack for diagnosing and fixing front-end performance problems. Loaded by OpenCode/Claude when users ask about React/Next.js performance, re-renders, Core Web Vitals, rendering strategies, or JavaScript runtime bottlenecks.

## Contents

| File | Covers |
|---|---|
| `SKILL.md` | Skill definition — triggers, usage guide, output format |
| `references/rendering-strategies.md` | SSR, SSG, ISR, CSR, RSC, streaming, hydration trade-offs |
| `references/react-hooks-and-rerenders.md` | `useEffect` correctness, `memo`/`useMemo`/`useCallback`, re-render causes, `children` composition |
| `references/js-runtime-performance.md` | Event loop, long tasks, GC, array/string/object costs, Web Workers |
| `references/nextjs-and-web-vitals.md` | `next/image`, `next/font`, `next/dynamic`, LCP/INP/CLS/TTFB, profiling tools |
| `references/checklist-and-patterns.md` | Audit checklist, before/after code patterns |

## Usage

The AI loads this skill automatically when the user asks performance-related questions about a React or Next.js codebase. No manual setup needed.
