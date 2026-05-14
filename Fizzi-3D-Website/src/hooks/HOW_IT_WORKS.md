# Fizzi-3D-Website / src/hooks — How It Works

Two hooks: a Zustand store and an SSR-safe media query hook. Both are covered in the project's root guide; this file goes deeper on the *why*.

---

## 1. `useStore.ts` — Zustand global state

```ts
import { create } from "zustand";

interface State {
  ready: boolean;
  isReady: () => void;
}

export const useStore = create<State>((set) => ({
  ready: false,
  isReady: () => set({ ready: true }),
}));
```

### How to use it

```tsx
// In any component:
const ready = useStore((state) => state.ready);    // subscribe to `ready`
const isReady = useStore((state) => state.isReady); // get the setter

// In a useEffect or callback:
isReady();  // sets ready to true
```

### The selector pattern (`(state) => state.x`)

```tsx
const ready = useStore((state) => state.ready);
```

This selector tells Zustand: "subscribe me to changes in `state.ready`". If `state.unrelated` changes, this component does NOT re-render. Automatic memoization.

If you grab the whole state without a selector:
```tsx
const state = useStore();  // bad — re-renders on every change
```
…the component re-renders on every state change. Avoid.

### Comparing Zustand to other state libraries

| Library | Setup ceremony | Re-render scope | Persistence |
| --- | --- | --- | --- |
| Redux | Reducer, action creator, store, provider | Manual `useSelector` (similar to Zustand) | Plug-in |
| Recoil | Atom, selector, provider | Auto via fine-grained graphs | Plug-in |
| Jotai | Atom, provider | Auto via fine-grained atoms | Plug-in |
| **Zustand** | `create((set) => ({...}))` | Manual `useStore(selector)` | Plug-in (`persist` middleware) |
| Context API | Provider, context, manual updates | Whole tree by default | None |

Zustand wins on simplicity for small-to-medium apps. ~1kb gzipped. No Provider needed (uses module-scoped state).

### Why no Provider?

The store is created at module load (top-level `create(...)`). It lives in the JS module — not in React state. The hook subscribes to it from anywhere. This means you can use the store outside React components too:

```ts
import { useStore } from "@/hooks/useStore";

// In a plain JS function:
function setupApp() {
  useStore.setState({ ready: false });
  const currentState = useStore.getState();
}
```

`useStore.getState()` and `useStore.setState()` are escape hatches that don't trigger React re-renders.

### Caveat: SSR with Zustand

By default, Zustand's module-scoped state can leak between SSR requests if you're not careful. For Next.js apps with SSR-rendered Zustand state, the convention is:

```ts
// Create a factory function
const createStore = () => create((set) => ({...}));
```

And initialize per-request. For client-only state (like `ready: false` here), the module-scoped approach is fine — the server never sets `ready: true` so there's no leak.

---

## 2. `useMediaQuery.ts` — SSR-safe React 18 media query

```ts
import { useCallback, useSyncExternalStore } from "react";

export function useMediaQuery(query: string, serverFallback: boolean): boolean {
  const subscribe = useCallback(
    (onStoreChange: () => void) => {
      const mediaQueryList = matchMedia(query);
      mediaQueryList.addEventListener("change", onStoreChange);
      return () => {
        mediaQueryList.removeEventListener("change", onStoreChange);
      };
    },
    [query],
  );

  return useSyncExternalStore(
    subscribe,
    () => matchMedia(query).matches,
    () => serverFallback,
  );
}
```

### `useSyncExternalStore` (React 18+)

A React 18 hook designed for subscribing to **external stores** (anything not managed by `useState`/`useReducer`). Examples:
- `matchMedia` (this file).
- `window.navigator.onLine`.
- `localStorage` changes.
- WebSocket messages.
- Custom event emitters.

The hook takes 3 arguments:
1. **`subscribe(onStoreChange)`** — function that attaches a listener and returns a cleanup. Triggered when component mounts; cleanup runs on unmount or deps change.
2. **`getSnapshot()`** — function that returns the current value (client side).
3. **`getServerSnapshot()` (optional)** — function that returns the SSR value (server side).

### Why is this better than `useEffect` + `useState`?

The naive version:

```ts
function useMediaQuery(query) {
  const [matches, setMatches] = useState(() => matchMedia(query).matches);  // SSR error!

  useEffect(() => {
    const mq = matchMedia(query);
    setMatches(mq.matches);
    mq.addEventListener("change", e => setMatches(e.matches));
    return () => mq.removeEventListener("change", ...);
  }, [query]);

  return matches;
}
```

Problems:
1. **SSR crash** — `matchMedia` is undefined on the server. `useState(() => matchMedia(...))` blows up.
2. **Hydration mismatch** — initial render uses default (e.g., `false`), then effect updates to actual value → React warns about hydration mismatch.
3. **Tearing** — in React 18 Concurrent Mode, a long render could observe stale state mid-render. `useSyncExternalStore` is designed to avoid this.

### Why `useCallback(subscribe, [query])`?

`useSyncExternalStore` re-subscribes when the `subscribe` function reference changes. Without `useCallback`, every render creates a new `subscribe` → endless re-subscription. With `useCallback([query])`, it stays stable as long as `query` is the same string.

### Usage

```tsx
const isDesktop = useMediaQuery("(min-width: 768px)", true);
```

The second argument is the server fallback. Set it based on what you expect server-rendered HTML to assume (e.g., `true` to optimize for desktop SSR).

### Why doesn't the project use `react-responsive`?

The `react-responsive` library is simpler but has SSR issues (hydration warnings). For Next.js SSR, this custom hook is the cleaner choice.

The Spylt and Mojito projects (Vite, no SSR) use `react-responsive` — no SSR, no problem.

---

## Build the hooks from scratch (interview-ready)

### Mini Zustand

```ts
function create(initializer) {
  let state;
  const listeners = new Set<() => void>();
  const setState = (partial: any) => {
    state = { ...state, ...(typeof partial === 'function' ? partial(state) : partial) };
    listeners.forEach(l => l());
  };
  state = initializer(setState);

  function useStore(selector = (s: any) => s) {
    const [, forceUpdate] = useReducer(x => x + 1, 0);
    useEffect(() => {
      listeners.add(forceUpdate);
      return () => listeners.delete(forceUpdate);
    }, []);
    return selector(state);
  }

  return useStore;
}
```

20 lines. Now `const store = create((set) => ({ count: 0, inc: () => set({ count: count+1 }) }))` works.

### Mini SSR-safe media query

```ts
function useMediaQuery(query: string, serverFallback: boolean): boolean {
  if (typeof window === 'undefined') return serverFallback;

  const [matches, setMatches] = useState(() => matchMedia(query).matches);
  useEffect(() => {
    const mq = matchMedia(query);
    setMatches(mq.matches);
    const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, [query]);

  return matches;
}
```

Simpler version. Works for most cases. Has minor hydration mismatch issues for the very first render — addressed by `useSyncExternalStore` in the project.

---

## What this folder teaches you

- **Zustand vs Context vs Redux**: when to use which.
- **Selector pattern** for fine-grained subscriptions.
- **`useSyncExternalStore`** for any external state (media queries, online status, localStorage, custom emitters).
- **SSR considerations**: `matchMedia` undefined on server, hydration mismatches, server fallback values.
- **`useCallback` for stable function references** when passed to other hooks.

### Most common interview questions covered here

- "How would you build a simple state manager?" → Mini Zustand above.
- "How do you make a hook that listens to window resize?" → Same pattern as `useMediaQuery`.
- "How do you handle SSR with hooks that need `window`?" → Server fallback + `useSyncExternalStore` or `typeof window === 'undefined'` guard.
