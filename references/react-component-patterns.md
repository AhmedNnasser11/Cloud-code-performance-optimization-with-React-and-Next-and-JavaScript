# React Component Patterns (HOC, Compound Components, and others)

Each pattern below includes what it's for, how to implement it, and — since this is a performance
skill — what it costs or saves in re-renders/bundle size, and when a simpler alternative is better.

## 1. Higher-Order Components (HOC)

A function that takes a component and returns a new component with added behavior/props.

```jsx
function withLoading(Component) {
  return function WithLoading({ isLoading, ...props }) {
    if (isLoading) return <Spinner />;
    return <Component {...props} />;
  };
}
const UserListWithLoading = withLoading(UserList);
```

**Performance notes**
- Defining a HOC-wrapped component **inside** another component's render creates a new component
  type every render, forcing React to unmount/remount the whole subtree (losing state, re-running
  effects) instead of updating it. Always define HOCs at module scope, never inline in a render.
  ```jsx
  // ❌ New component identity every render — remounts every time
  function Parent() {
    const Enhanced = withLoading(UserList); // recreated each render
    return <Enhanced isLoading={loading} />;
  }

  // ✅ Defined once, outside render
  const EnhancedUserList = withLoading(UserList);
  function Parent() {
    return <EnhancedUserList isLoading={loading} />;
  }
  ```
- HOCs don't inherently cause extra re-renders beyond what the props passed down trigger, but layered
  HOCs ("wrapper hell") make it harder to see where a re-render originates and harder for tools like
  React DevTools to attribute cost.
- **Mostly superseded by custom hooks** in modern React — a custom hook (`useLoading()`) usually gives
  the same reuse without an extra component layer in the tree, and is easier to compose. Prefer hooks
  for new code; you'll still see HOCs in older codebases and some libraries (e.g. `connect` from
  Redux, `withRouter`-style APIs).

## 2. Compound Components

A parent component implicitly shares state with its children via Context, so the children can be
composed flexibly while still coordinating (e.g. `<Tabs>`, `<Tab>`, `<TabPanel>`).

```jsx
const TabsContext = createContext(null);

function Tabs({ children, defaultValue }) {
  const [active, setActive] = useState(defaultValue);
  const value = useMemo(() => ({ active, setActive }), [active]);
  return <TabsContext.Provider value={value}>{children}</TabsContext.Provider>;
}
function Tab({ value, children }) {
  const { active, setActive } = useContext(TabsContext);
  return (
    <button onClick={() => setActive(value)} aria-selected={active === value}>
      {children}
    </button>
  );
}
function TabPanel({ value, children }) {
  const { active } = useContext(TabsContext);
  return active === value ? <div>{children}</div> : null;
}

// usage — fully composable, no prop drilling
<Tabs defaultValue="a">
  <Tab value="a">A</Tab>
  <Tab value="b">B</Tab>
  <TabPanel value="a">Content A</TabPanel>
</Tabs>;
```

**Performance notes**
- **Every consumer of the context re-renders whenever the context value changes**, even if that
  consumer only cares about part of it. Always `useMemo` the context value object — otherwise you
  create a new object every render of the provider and every consumer re-renders regardless of
  whether `active` actually changed.
- If the compound component tree is large (many `Tab`/`TabPanel` instances), consider splitting into
  two contexts (e.g. one for `active` value, one for `setActive` — the setter is referentially stable
  from `useState` and rarely needs to trigger re-renders) so consumers that only call `setActive`
  don't re-render when `active` changes.
- This is a good pattern specifically *because* it avoids prop drilling — the alternative (passing
  `active`/`setActive` as explicit props through every layer) causes intermediate components to
  re-render too, even ones that don't use the values, just to pass them through.

## 3. Render Props

Pass a function as a prop/children that a component calls with internal state, letting the caller
control rendering.

```jsx
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <div onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}>
      {children(pos)}
    </div>
  );
}
// usage
<MouseTracker>{(pos) => <p>{pos.x}, {pos.y}</p>}</MouseTracker>;
```

**Performance notes**
- Every state change in the render-prop component re-renders whatever the render function returns —
  there's no way to bail out of that with `memo` because a new function is created and called each
  render. This makes render props a poor fit for a state that changes rapidly (like `MouseTracker`
  above) wrapping something expensive.
- **Mostly superseded by custom hooks** for the same reason as HOCs — `usePosition()` returning
  `{ x, y }` gives the same capability without forcing a render-prop function call and without adding
  a wrapper component to the tree. Prefer hooks in new code; render props still show up in some
  libraries (e.g. older React Router, some UI libraries' internals).

## 4. Provider Pattern (Context for cross-cutting state)

Wrap a subtree in a Context Provider to avoid prop drilling for things like theme, auth, or locale.

**Performance notes**
- Same rule as Compound Components: memoize the value object, split contexts by concern (don't put
  `theme`, `user`, and `cart` in one context if they change independently), and keep providers as low
  in the tree as the data's actual scope requires — a provider at the root re-renders the entire app
  on any value change.
- For state that changes very frequently (e.g. scroll position, mouse position, animation frames),
  Context is usually the wrong tool — it will re-render every consumer on every change. Use a ref, a
  dedicated state library with selector-based subscriptions (see Zustand file), or `useSyncExternalStore`.

## 5. Container / Presentational split

Separate data-fetching/state ("container") from rendering ("presentational", ideally a pure function
of props). Largely superseded by hooks (a `useX()` hook plays the container's role) and by Server
Components in Next.js (the server component fetches, a client component presents) — but the
underlying principle (keep expensive/stateful logic separate from cheap, easily-memoized rendering)
is still exactly why `memo` works well on presentational components and poorly on stateful ones.

## 6. Controlled vs. Uncontrolled Components

- **Controlled**: value lives in React state, component is just a view (`<input value={v}
  onChange={...} />`). Every keystroke triggers a re-render of whatever owns that state.
- **Uncontrolled**: value lives in the DOM, read via `ref` when needed (`<input ref={inputRef}
  defaultValue="..." />`). No re-render per keystroke.

**Performance notes**
- For high-frequency inputs (large forms, search-as-you-type over a big list, rich text), uncontrolled
  inputs — or controlled inputs with the state colocated in a small leaf component, not a shared
  parent — avoid re-rendering large subtrees on every keystroke. Combine with `useDeferredValue` if
  the input needs to drive an expensive derived render (e.g. filtering a big list) without blocking
  typing.

## 7. Composition over configuration (`children` / slots)

Already covered in depth in `react-hooks-and-rerenders.md` §3 — passing pre-built elements as
`children` (or named slot props) instead of a big prop-config object is both more flexible and avoids
re-render cascades, because the element reference passed as `children` doesn't change even when the
parent re-renders for an unrelated reason.

## Which pattern to reach for

| Need | Prefer |
|---|---|
| Share stateful logic across components | Custom hook (not HOC/render props, unless targeting a library API that requires them) |
| Coordinate a set of related components without prop drilling | Compound components (Context) |
| Cross-cutting app-wide state (theme, auth, locale) | Provider pattern, memoized value, split by concern |
| High-frequency local state (typing, dragging, scrolling) | Uncontrolled / colocated state / refs, avoid Context |
| Flexible child content inside a wrapper | `children`/slot props (composition) |
| Global state shared across many unrelated components, with fine-grained subscriptions | Zustand or similar (see `zustand-usage.md`) — better re-render characteristics than Context for this at scale |
