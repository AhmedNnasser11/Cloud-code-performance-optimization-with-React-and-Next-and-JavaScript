# Zustand — Performance-Correct Usage

Zustand is a small external-store state manager. Its main performance advantage over Context is
**selector-based subscriptions**: a component only re-renders when the specific slice it selected
changes, not on every store update — but only if you use selectors correctly.

## Basic store

```js
import { create } from 'zustand';

const useCartStore = create((set, get) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) => set((state) => ({ items: state.items.filter(i => i.id !== id) })),
  total: () => get().items.reduce((sum, i) => sum + i.price, 0),
}));
```

## The #1 mistake: subscribing to the whole store

```jsx
// ❌ Re-renders on ANY store change, even unrelated fields
const { items, addItem } = useCartStore();

// ✅ Select only what this component needs
const items = useCartStore((state) => state.items);
const addItem = useCartStore((state) => state.addItem);
```

Calling the hook with no selector (or destructuring the whole state) subscribes the component to
every field in the store — exactly the Context over-subscription problem Zustand is meant to avoid.
Always pass a selector function.

## Selecting multiple fields without extra re-renders

A selector returning a new object/array every call breaks reference equality and causes a re-render
on every store change even if the selected values didn't change:

```jsx
// ❌ New object every render → always "changed" by default equality
const { items, total } = useCartStore((state) => ({ items: state.items, total: state.total }));

// ✅ Option 1: separate selector calls (each independently memoized by Zustand)
const items = useCartStore((state) => state.items);
const total = useCartStore((state) => state.total);

// ✅ Option 2: shallow-compare the combined selector
import { useShallow } from 'zustand/react/shallow';
const { items, total } = useCartStore(
  useShallow((state) => ({ items: state.items, total: state.total }))
);
```

## Slicing large stores

For a big app-wide store, split into logical slices (still one store, organized by domain) rather than
one flat object, so selectors stay simple and intent is clear:

```js
const useAppStore = create((set) => ({
  ...createCartSlice(set),
  ...createUserSlice(set),
  ...createUiSlice(set),
}));
```
Or use multiple independent stores (`useCartStore`, `useUserStore`) if the domains are truly unrelated
— this also means a change in one never even risks affecting the other's subscribers.

## Actions don't need selectors to be "safe"

Functions defined in the store (`addItem`, `removeItem`) are stable references across renders by
default (Zustand doesn't recreate the store object), so selecting an action doesn't need `useShallow`
or memoization on your end — `useCartStore((state) => state.addItem)` is already stable.

## Zustand + Next.js App Router / SSR

- **Don't create the store as a module-level singleton for SSR apps** — a shared singleton store leaks
  state between requests/users on the server. Create the store per-request (e.g. in a Provider that
  instantiates the store in a `'use client'` root layout wrapper, or via a factory function + React
  Context to hand out a per-tree instance) and pass initial/hydrated state down from Server Components
  as plain props.
  ```jsx
  // store/cart-store.js
  export const createCartStore = (initState) => create((set) => ({ ...initState, /* actions */ }));

  // providers/cart-provider.jsx ('use client')
  const CartStoreContext = createContext(null);
  export function CartStoreProvider({ initState, children }) {
    const storeRef = useRef(createCartStore(initState));
    return <CartStoreContext.Provider value={storeRef.current}>{children}</CartStoreContext.Provider>;
  }
  export function useCartStore(selector) {
    const store = useContext(CartStoreContext);
    return useStore(store, selector); // useStore from 'zustand'
  }
  ```
- Only components that actually need client-side interactivity should touch the store — keep Server
  Components server-only and pass Zustand-derived state down as props/initial state, not by importing
  the store into a Server Component (it can't use hooks anyway).
- If using `persist` middleware (localStorage), guard against SSR hydration mismatches — the server
  has no `localStorage`, so the first client render must match the server's render before persisted
  state "kicks in" (Zustand's `persist` handles this via `skipHydration`/`onRehydrateStorage`, but be
  aware initial paint may briefly show default state before persisted state applies).

## Zustand vs. Context vs. `useSyncExternalStore` — when to reach for which

| Situation | Prefer |
|---|---|
| A few components need shared, infrequently-changing state (theme, locale) | Context is fine — simpler, no extra dependency |
| Many unrelated components need slices of frequently-changing global state | Zustand (or similar) — selector subscriptions avoid the over-render problem Context has at scale |
| Subscribing directly to a genuinely external source (browser API, non-React library) | `useSyncExternalStore` directly, or a library built on it (Zustand itself is built on it internally) |
| Complex server-driven mutation state (forms, Server Actions) | `useActionState`/`useOptimistic` (built into React/Next.js) rather than modeling it in a client store |
