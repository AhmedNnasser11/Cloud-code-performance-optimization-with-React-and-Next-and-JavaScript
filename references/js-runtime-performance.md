# JavaScript Runtime Performance

## Event loop & long tasks
- JS runs on a single main thread shared with layout/paint. Any synchronous task >50ms is a "long
  task" and blocks user input and rendering.
- Break up heavy synchronous work with `setTimeout`, `requestIdleCallback`, or by chunking loops.
- In React, use `startTransition`/`useTransition` to mark expensive state updates as non-urgent so
  the browser can still respond to clicks/typing first; use `useDeferredValue` to delay re-rendering
  an expensive subtree until after a more urgent update.
- Offload genuinely heavy computation (parsing, image processing, large sorts/transforms) to a **Web
  Worker** so it doesn't touch the main thread at all.

## Memory & garbage collection
- GC is automatic but pauses can still cause jank if you churn a lot of short-lived objects in hot
  paths (e.g. creating new objects/arrays inside a render or a scroll/animation handler).
- Common leak sources: event listeners not removed, `setInterval`/`setTimeout` not cleared, closures
  retaining large objects, detached DOM nodes referenced from JS. Always clean up in `useEffect`
  return functions.
- Chrome DevTools → Memory panel: heap snapshots and "detached DOM tree" search catch most leaks.

## V8-level micro-optimizations (matter in hot loops, not typical app code)
- Keep object shapes consistent — don't add/delete properties dynamically or change a property's
  type across calls; this breaks V8's hidden-class/inline-cache optimizations.
- Avoid highly polymorphic functions (same function called with many different argument shapes/types)
  in tight loops.
- These optimizations are rarely worth chasing outside of genuinely hot loops (animation frames,
  large data processing) — don't micro-optimize typical UI code for this.

## Method/pattern cost cheat sheet
- **Debounce vs throttle**: debounce delays execution until input stops (good for search-as-you-type,
  resize-end); throttle limits execution to at most once per interval (good for scroll/mousemove
  handlers that must still fire regularly). Wrong choice is a very common cause of janky scroll/input
  handling.
- **Array methods**: `.map/.filter/.reduce` are clear and fine for typical UI-scale data; for very
  large arrays in a hot path, a single `for` loop that does the work in one pass beats chaining
  multiple `.map().filter()` calls (each chain allocates an intermediate array).
- **String concatenation** in a loop: prefer building an array and `.join('')`, or template literals,
  over repeated `+=` in very large loops — modern engines optimize `+=` reasonably well now, but this
  still matters for large N.
- **`JSON.parse`/`JSON.stringify`** on large payloads is synchronous and can itself be a long task —
  consider streaming/chunking or moving to a worker for very large payloads.
- **Avoid `try { require(x) }`-style dynamic/conditional imports of whole libraries** — prefer static
  ES module `import { specificThing } from 'lib'` so bundlers can tree-shake; `require()` and dynamic
  patterns often defeat tree-shaking.
- Prefer native platform APIs (`structuredClone`, `Array.prototype.at`, `Object.groupBy` where
  available) over hand-rolled or lodash equivalents when only using a small utility — reduces bundle
  size and avoids an extra dependency's overhead.

## CSS/paint-adjacent JS costs
- Reading layout properties (`offsetWidth`, `getBoundingClientRect`, etc.) right after writing styles
  forces synchronous layout ("layout thrashing"). Batch all reads, then all writes.
- Animate with `transform`/`opacity` (compositor-only) instead of `width`/`height`/`top`/`left`
  (triggers layout) when doing JS-driven animation.
