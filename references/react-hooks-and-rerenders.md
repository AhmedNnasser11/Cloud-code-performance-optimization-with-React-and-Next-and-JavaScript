# React Hooks, useEffect, and Unnecessary Re-renders

Based on react.dev guidance ("You Might Not Need an Effect," "Passing Data Deeply with Context,"
"Referencing Values with Refs," rules of hooks) plus standard render-optimization practice.

## 1. "You Might Not Need an Effect" — the core review checklist

An Effect is for **synchronizing with something outside React** (a DOM API, network, subscription,
third-party widget, or another framework). If a `useEffect` isn't doing that, it's very likely wrong.

Run every `useEffect` you see through this list:

- **Deriving state from props/state?** Don't `useEffect` + `setState` to compute a value from other
  state — just compute it during render.
  ```jsx
  // ❌ Anti-pattern: redundant state + effect
  const [fullName, setFullName] = useState('');
  useEffect(() => {
    setFullName(firstName + ' ' + lastName);
  }, [firstName, lastName]);

  // ✅ Just compute it
  const fullName = firstName + ' ' + lastName;
  ```
  This causes an extra render every time (render with stale value → effect fires → re-render with
  correct value) and is a very common source of "why does my component render twice."

- **Expensive derived value?** Use `useMemo`, not an Effect + state.
  ```jsx
  // ✅ Memoize expensive computation, still during render, no extra render pass
  const visibleTodos = useMemo(() => filterTodos(todos, filter), [todos, filter]);
  ```

- **Resetting/adjusting state when a prop changes?** Prefer a `key` prop to reset the whole component,
  or compute during render, rather than an Effect that calls `setState`.
  ```jsx
  // ❌ Effect that resets state on prop change
  useEffect(() => { setComment(''); }, [userId]);

  // ✅ Force remount with a key — no effect needed
  <Profile userId={userId} key={userId} />
  ```

- **Responding to a user event (click, submit, input)?** That logic belongs in the **event handler**,
  not an Effect. Effects run after render regardless of *why* the state changed, which is wrong for
  "this happened because the user did X" logic (e.g. showing a toast after a POST, or a one-time
  analytics event on a button click).
  ```jsx
  // ❌ Effect reacting to a state flag set by a click
  useEffect(() => {
    if (submitted) { showToast('Order placed!'); }
  }, [submitted]);

  // ✅ Directly in the handler
  function handleSubmit() {
    post('/orders', order);
    showToast('Order placed!');
  }
  ```

- **Fetching data on mount with no cache/race-condition handling?** This is the most common
  legitimate-looking-but-flawed Effect. At minimum it needs a cleanup flag to avoid race conditions;
  in most real apps this should be a framework data-fetching mechanism instead:
  - Next.js App Router: fetch in a Server Component, or use `<Suspense>` + a data library.
  - Otherwise: React Query / SWR, which handle caching, dedupe, and race conditions for you.
  ```jsx
  // If you must do it manually, at least guard against races:
  useEffect(() => {
    let ignore = false;
    fetchResults(query).then(json => { if (!ignore) setResults(json); });
    return () => { ignore = true; };
  }, [query]);
  ```

- **Legitimate Effect uses** (keep these as Effects): subscribing to a browser/DOM event not exposed
  as a prop, controlling a non-React widget, sending analytics on component visibility (not on a
  click), manually managing focus/scroll on mount. The test: "would this logic need to run even if
  this component had been rendered by something other than a click/user interaction?" If yes, Effect.
  If it's tied to a specific interaction, put it in the handler.

## 2. Hook correctness quick-reference

| Hook | Correct usage | Common mistake |
|---|---|---|
| `useState` | Local, render-triggering state | Storing derived values that should be computed inline; storing large objects that get replaced wholesale each render |
| `useEffect` | Sync with an external system after commit | Used to derive state or react to events (see §1) |
| `useLayoutEffect` | Same as `useEffect` but fires synchronously before paint — only when you must measure/mutate the DOM before the browser paints (e.g. tooltip positioning) | Used by default "just in case" — it blocks paint, strictly worse than `useEffect` when not needed |
| `useMemo` | Cache an **expensive calculation** or a value passed to a memoized child so identity is stable | Wrapping cheap calculations "for performance" — the memoization overhead can exceed the savings |
| `useCallback` | Stabilize a function identity being passed to a memoized child or a dependency array | Used on every function "just in case" without a `memo`-wrapped consumer downstream — no benefit |
| `useRef` | Mutable value that must NOT trigger a re-render, or a DOM node handle | Used as a substitute for state when the UI actually needs to reflect the value (won't re-render) |
| `useContext` | Share state across a subtree without prop drilling | Put at too broad a Provider scope so every consumer re-renders on any context change — split into multiple contexts by concern |
| `useReducer` | Complex state transitions, multiple sub-values updated together | Used for a single primitive value where `useState` is clearer |
| `useTransition` / `startTransition` | Mark a state update as non-urgent so urgent updates (typing, clicks) aren't blocked | Not used for expensive renders triggered by input, causing typing lag |
| `useDeferredValue` | Defer re-rendering an expensive subtree until after a more urgent render | Confused with debouncing — it's not a timer, it's a render-priority hint |
| `useSyncExternalStore` | Subscribe to a mutable external data source (browser API, third-party store, custom pub/sub) in a way that's safe under concurrent rendering | Subscribing to external stores via `useEffect` + `useState` instead — that pattern can tear (show inconsistent values) under concurrent features; this hook exists specifically to fix that |
| `useEffectEvent` (Effect Event, stable in recent React) | Extract non-reactive logic out of an Effect — read the *latest* props/state inside an Effect without making them dependencies and re-triggering the Effect | Used as a general escape hatch from the dependency array (defeats the linter's correctness guarantee) instead of only for genuinely non-reactive reads, e.g. logging the current theme on a reactive `roomId` connection without reconnecting when theme changes |
| `useId` | Generate a stable, unique ID for accessibility attributes (`aria-*`, `htmlFor`) that matches between server and client render | Used as a general-purpose key generator or for list `key`s — it's not for that, and it's not a performance hook at all |
| `useImperativeHandle` | Customize what a parent gets when it attaches a `ref` to a child (used with `forwardRef`) — expose an imperative API instead of the raw DOM node | Overused to avoid proper data-flow — if the parent just needs a value to render, that should be a prop/state, not an imperative ref method |
| `useDebugValue` | Label custom hooks in React DevTools | Included in production logic thinking it does something functional — it's dev-tooling only, zero runtime effect |
| `use` (React 19) | Read a Promise or Context value, can be called conditionally/in loops (unlike other hooks) — used with `<Suspense>` to await data in render | Confusing with `useEffect`-based fetching, or awaiting a Promise that isn't cached/stable across renders, causing an infinite re-fetch loop |
| `useActionState` (React 19) | Manage state driven by a form action, including pending/error state, integrated with Server Actions | Reimplementing this manually with several `useState` + `useTransition` calls when the built-in hook already models the pending/result/error cycle |
| `useOptimistic` (React 19) | Show an optimistic UI update while an async action (e.g. a Server Action) is in flight, auto-reverting on failure | Manually juggling temporary state + rollback logic in `useEffect`/`useState` for the same effect |

Rules of Hooks (non-negotiable, breaks correctness not just perf): only call hooks at the top level
(never in loops/conditions/nested functions), only call from React function components or custom
hooks.

## 3. Avoiding unnecessary re-renders

**Why a component re-renders:** its own state changed, a parent re-rendered (and it isn't memoized),
or a subscribed context changed.

- **`React.memo`**: skips re-render if props are shallowly equal. Useless if you pass new
  object/array/function literals as props each render — those fail the shallow-equality check every
  time.
  ```jsx
  // ❌ Defeats memo — new object every render
  <ExpensiveChild style={{ color: 'red' }} onClick={() => doThing(id)} />

  // ✅ Stable references
  const style = useMemo(() => ({ color: 'red' }), []);
  const handleClick = useCallback(() => doThing(id), [id]);
  <ExpensiveChild style={style} onClick={handleClick} />
  ```

- **The `children` pattern** — often a better fix than memoization. If a parent re-renders because of
  its own state, everything passed as `props.children` was already created **before** that state
  changed and React can bail out of re-rendering it, even without `memo`, because the element
  reference didn't change.
  ```jsx
  // ❌ ExpensiveTree re-renders every time count changes, even though it doesn't use count
  function Parent() {
    const [count, setCount] = useState(0);
    return (
      <div>
        <button onClick={() => setCount(c => c + 1)}>{count}</button>
        <ExpensiveTree />
      </div>
    );
  }

  // ✅ Lift the state-owning part out; pass ExpensiveTree down as children.
  // Counter re-renders on click, but `children` is the same element reference,
  // so ExpensiveTree is not re-rendered.
  function Parent({ children }) {
    return (
      <div>
        <Counter />
        {children}
      </div>
    );
  }
  // usage: <Parent><ExpensiveTree /></Parent>
  ```
  This is the standard fix for "state colocated with a component that also renders something
  unrelated and expensive" — move the state down into a small component and pass the expensive part
  as `children` (or another element prop) from above it.

- **Colocate state** as close as possible to where it's used, instead of at a high-level parent, so
  state changes only re-render the small subtree that needs it.

- **Split contexts by concern** (e.g. `ThemeContext` vs `UserContext` vs `CartContext`) instead of one
  big context object — a single context change re-renders every consumer regardless of which field
  actually changed.

- **List rendering**: always use a stable, unique `key` (not array index if the list can reorder/
  filter/insert) — wrong keys cause React to misattribute state across re-renders and can force full
  remounts instead of efficient updates.

## 4. useSyncExternalStore and useEffectEvent in detail

These two are easy to skip but matter a lot once you're past basic `useState`/`useEffect` usage.

**`useSyncExternalStore`** — the correct way to read a value that lives outside React (window size,
`navigator.onLine`, a third-party store like Redux/Zustand internals, a WebSocket's latest message)
without risking "tearing" under React's concurrent rendering (where different parts of the tree could
otherwise observe different values of the same external state during one render pass).
```jsx
// ❌ Fragile hand-rolled subscription
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  useEffect(() => {
    const onResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', onResize);
    return () => window.removeEventListener('resize', onResize);
  }, []);
  return width;
}

// ✅ Concurrent-safe
function useWindowWidth() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    () => window.innerWidth,       // client snapshot
    () => 0                         // server snapshot (SSR-safe default)
  );
}
```

**`useEffectEvent`** — lets an Effect read the *latest* value of a prop/state without that value
being a reactive dependency (i.e., without re-running the Effect when it changes). This solves the
common "my Effect re-runs too often because of a value it only needs to read, not react to" problem,
without lying to the dependency-array linter by omitting a real dependency.
```jsx
// ❌ Either stale `theme` inside the effect, or the effect reconnects every time theme changes
useEffect(() => {
  const connection = createConnection(roomId);
  connection.on('connected', () => showNotification('Connected!', theme));
  connection.connect();
  return () => connection.disconnect();
}, [roomId, theme]); // reconnects on every theme change — not what you want

// ✅ theme is read fresh each time, but doesn't trigger a reconnect
const onConnected = useEffectEvent(() => {
  showNotification('Connected!', theme);
});
useEffect(() => {
  const connection = createConnection(roomId);
  connection.on('connected', () => onConnected());
  connection.connect();
  return () => connection.disconnect();
}, [roomId]); // only reconnects when roomId actually changes
```

## 5. When NOT to memoize

Don't reach for `useMemo`/`useCallback`/`memo` by default:
- Cheap calculations (string concat, small array `.map`) — memoization bookkeeping can cost more than
  just recomputing.
- Components that re-render rarely or are cheap to render regardless.
- Values only consumed by non-memoized children — there's nothing for the stable reference to help.

Measure first (React Profiler, "Highlight updates when components render" in DevTools) rather than
memoizing speculatively — over-memoized code is harder to read and can itself hide bugs (stale
closures from wrong dependency arrays).
