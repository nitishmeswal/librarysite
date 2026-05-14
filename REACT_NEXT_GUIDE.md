# React + Next.js Guide (SDE 1 level)

This guide is the **mental model** you need before reading the per-project guides. Everything below uses code straight from this repo so you see fundamentals in real production context.

---

## 1. Components are just functions that return JSX

JSX is sugar for `React.createElement(...)`. A component is a function that takes `props` and returns "what to render".

From `Personal-Portfolio/src/App.jsx`:

```jsx
const App = () => {
  return (
    <div className="container mx-auto max-w-7xl">
      <Navbar />
      <Hero />
      <About />
      <Project />
      <Experience />
      <Testimonial />
      <Contact />
      <Footer />
    </div>
  );
};

export default App;
```

**Breakdown**:
- `const App = () => { ... }` — arrow function. `App` IS the component. The name starts with a capital letter so JSX treats `<App />` as a component (lowercase would be treated as an HTML tag).
- `return (...)` — must return a single root element (here a `<div>`). The parentheses just let us put the JSX on multiple lines.
- `className` (not `class`) because `class` is a reserved keyword in JS.
- `<Navbar />`, `<Hero />`, ... — these are other components imported above. React renders them by calling their function with `props = {}`.
- `export default App` — makes the component importable elsewhere with `import App from "./App"`.

**Interview question**: *"Why must component names start with a capital letter?"* → Because JSX uses the case to differentiate between a built-in HTML element (`<div>`) and a custom React component (`<App>`). Lowercase = HTML, Uppercase = React component.

---

## 2. Props are how parents pass data to children

From `Personal-Portfolio/src/components/Card.jsx`:

```jsx
const Card = ({ style, text, image }) => {
  return (
    <motion.div
      className="absolute md:px-3 py-2 px-5 text-sm cursor-grab bg-storm rounded-3xl"
      style={style}
      drag
      dragConstraints={...}
      whileHover={{ scale: 1.05 }}
    >
      {text}
      {image && <img src={image} className="w-24 md:w-32" />}
    </motion.div>
  );
};
```

**Breakdown**:
- `({ style, text, image })` — destructuring `props`. Equivalent to `const Card = (props) => { const { style, text, image } = props; ... }`.
- `style={style}` — passes the `style` prop down to a DOM-ish element.
- `{text}` — JS expression in JSX. Whatever `text` evaluates to gets rendered.
- `{image && <img ... />}` — conditional rendering. If `image` is truthy, render the `<img>`. If falsy (`null`, `undefined`, `""`, `0`), render nothing. Be careful with `0` — `0 && <X />` renders `0`. Use `Boolean(image) && ...` if `image` could be `0`.

---

## 3. State with `useState`

Whenever the UI must "remember" something between renders, use `useState`.

From `Personal-Portfolio/src/sections/Navbar.jsx`:

```jsx
import { useState } from 'react';

const Navbar = () => {
  const [isOpen, setIsOpen] = useState(false);
  // ...
  return (
    <button onClick={() => setIsOpen(!isOpen)}>...</button>
  );
};
```

**Breakdown**:
- `useState(false)` — initial value is `false`. Returns an array of `[currentValue, setterFunction]`.
- `[isOpen, setIsOpen] = ...` — array destructuring.
- `setIsOpen(!isOpen)` — schedules a re-render. After the render, `isOpen` will be the new value.
- React batches updates inside event handlers, so calling `setIsOpen` doesn't immediately change `isOpen` in the current scope.

**Interview gotcha**: `setIsOpen(isOpen + 1)` then `setIsOpen(isOpen + 1)` in the same handler increments by 1, not 2 — both read the same stale `isOpen`. Use the functional form: `setIsOpen((prev) => prev + 1)`.

---

## 4. Refs with `useRef`

`useRef` gives you a mutable container `{ current: ... }` that persists across renders **without** causing a re-render when you mutate it. Two main uses:

### 4a. Access a DOM node

From `Personal-Portfolio/src/hooks/useScrollReveal.js`:

```js
const ref = useRef(null);

useEffect(() => {
  const observer = new IntersectionObserver(...);
  const currentRef = ref.current;
  if (currentRef) {
    observer.observe(currentRef);
  }
  return () => {
    if (currentRef) {
      observer.unobserve(currentRef);
    }
  };
}, [...]);

return [ref, isVisible];
```

`<div ref={ref}>` makes `ref.current` point to that DOM element after render.

### 4b. Store a value that doesn't trigger re-renders

From `Fizzi-3D-Website/src/slices/Hero/Bubbles.tsx`:

```ts
const bubbleSpeed = useRef(new Float32Array(count));
```

The speeds change every frame, but we don't want React to re-render every frame. So we mutate `bubbleSpeed.current[i]` directly.

---

## 5. `useEffect` — side effects after render

```jsx
useEffect(() => {
  // do something (subscribe, fetch, set up listener)
  return () => {
    // cleanup (unsubscribe, abort, remove listener)
  };
}, [deps]);
```

Rules:
- Runs **after** the browser paints (use `useLayoutEffect` if you need it before paint).
- The cleanup runs before the next effect re-runs and when the component unmounts.
- `[]` → runs once on mount, cleanup on unmount.
- `[a, b]` → re-runs when `a` or `b` changes.
- No array → runs on every render (usually wrong).

From `Personal-Portfolio/src/components/FlipWords.jsx`:

```jsx
useEffect(() => {
  if (!isAnimating)
    setTimeout(() => {
      startAnimation();
    }, duration);
}, [isAnimating, duration, startAnimation]);
```

**What this does**: every time `isAnimating` becomes false, schedule the next word swap after `duration` ms.

**Interview gotcha**: *"Why is `startAnimation` in the deps array?"* → Because the linter (`react-hooks/exhaustive-deps`) tracks closures. If `startAnimation` is recreated on every render, the effect re-runs every render. That's why `startAnimation` is wrapped in `useCallback` upstream.

---

## 6. `useCallback` and `useMemo`

- `useCallback(fn, deps)` returns the same function reference between renders **as long as deps don't change**. Useful when passing callbacks to memoized children or to `useEffect` deps.
- `useMemo(() => compute(), deps)` returns the same value between renders unless deps change.

From `Personal-Portfolio/src/components/FlipWords.jsx`:

```jsx
const startAnimation = useCallback(() => {
  const word = words[words.indexOf(currentWord) + 1] || words[0];
  setCurrentWord(word);
  setIsAnimating(true);
}, [currentWord, words]);
```

Without `useCallback`, `startAnimation` would be a fresh function on every render. The `useEffect` that depends on it would re-run every render. With `useCallback`, the function reference only changes when `currentWord` or `words` change.

---

## 7. Custom hooks — extract logic, not UI

A custom hook is just a function that:
1. Starts with `use` (linter rule).
2. Calls other hooks inside.

Full example: `Personal-Portfolio/src/hooks/useScrollReveal.js`:

```js
import { useEffect, useRef, useState } from 'react';

export const useScrollReveal = (options = {}) => {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          if (options.once) {
            observer.unobserve(entry.target);
          }
        } else if (!options.once) {
          setIsVisible(false);
        }
      },
      {
        threshold: options.threshold || 0.1,
        rootMargin: options.rootMargin || '0px',
      }
    );

    const currentRef = ref.current;
    if (currentRef) {
      observer.observe(currentRef);
    }

    return () => {
      if (currentRef) {
        observer.unobserve(currentRef);
      }
    };
  }, [options.once, options.threshold, options.rootMargin]);

  return [ref, isVisible];
};
```

**Line by line**:
- `useState(false)` — track whether the element has scrolled into view.
- `useRef(null)` — the element we want to observe.
- `new IntersectionObserver(callback, options)` — browser API that fires `callback` when the observed element enters/leaves the viewport.
- `entry.isIntersecting` — true if any part of the element is in the viewport.
- `options.once` — if true, fire once then stop observing (so re-entering doesn't re-trigger).
- `threshold: 0.1` — fire when 10% of the element is visible.
- `rootMargin: '0px'` — virtual margin around viewport. `-100px` = trigger 100px before scrolling into view.
- `observer.observe(currentRef)` — start watching.
- The cleanup `observer.unobserve(currentRef)` runs when the component unmounts so we don't leak observers.
- `return [ref, isVisible]` — same pattern as `useState`.

**Usage**:

```jsx
const [titleRef, titleVisible] = useScrollReveal({ once: true });
return <h2 ref={titleRef} className={titleVisible ? 'opacity-100' : 'opacity-0'}>Hello</h2>;
```

---

## 8. Conditional rendering patterns

```jsx
{condition && <Component />}                       // render or nothing
{condition ? <A /> : <B />}                        // either/or
{items.length === 0 ? <Empty /> : <List items={items} />}
```

From `Fizzi-3D-Website/src/slices/Hero/index.tsx`:

```jsx
{isDesktop && (
  <View className="...">
    <Scene />
    <Bubbles count={300} speed={2} repeat={true} />
  </View>
)}
```

Only mounts the 3D scene on desktop. On mobile, the canvas isn't even in the tree — saves CPU/GPU.

---

## 9. Lists with `.map()` and `key`

```jsx
{items.map((item, i) => (
  <Card key={item.id} {...item} />
))}
```

Rules:
- `key` must be stable and unique among siblings. Prefer a real id over `i` (index).
- Why? React uses `key` to match elements across renders. If you re-order a list with index keys, React mis-matches state.

From `SpacePortfolio/components/sub/HeroContent.tsx`, lists of nav items, skill icons, etc., follow this pattern.

---

## 10. Next.js fundamentals (App Router)

Next.js 13+ uses the **App Router**: directory `app/` defines routes by folder names.

From `Fizzi-3D-Website/src/app/layout.tsx`:

```tsx
import "./app.css";
import Header from "@/components/Header";
import ViewCanvas from "@/components/ViewCanvas";
import Footer from "@/components/Footer";

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en" className={alpino.variable}>
      <body className="overflow-x-hidden bg-yellow-300">
        <Header />
        <main>
          {children}
          <ViewCanvas />
        </main>
        <Footer />
      </body>
    </html>
  );
}
```

**Breakdown**:
- `app/layout.tsx` is the **root layout** — it wraps every page.
- `children` is the page being rendered (e.g. `app/page.tsx`).
- This is a **Server Component** by default. It runs on the server (or at build time). It cannot use `useState`, `useEffect`, browser APIs.
- To make a component client-side, add `"use client"` at the top — required for any component using hooks or browser APIs.

From `Fizzi-3D-Website/src/components/ViewCanvas.tsx`:

```tsx
"use client";

import { Canvas } from "@react-three/fiber";
// ...
export default function ViewCanvas() {
  return <Canvas>...</Canvas>;
}
```

The `"use client"` directive means: ship this component's JS to the browser and run it there.

**Interview question**: *"What's the difference between a server and client component?"* → Server components render on the server, can fetch data directly, can be `async`, and ship zero JS to the browser. Client components are interactive (can use state, events, refs) and their JS is bundled and sent to the browser.

---

## 11. Server-side data fetching (App Router)

From `Fizzi-3D-Website/src/app/page.tsx`:

```tsx
export default async function Index() {
  const client = createClient();
  const home = await client.getByUID("page", "home");
  return <SliceZone slices={home.data.slices} components={components} />;
}
```

**Breakdown**:
- `async function Index()` — yes, the component itself is async. React waits for the promise before rendering.
- `client.getByUID(...)` — fetches CMS content at request time (or build time, depending on caching).
- No `useEffect`, no loading state needed. The page only renders once data is ready.

---

## 12. `useSyncExternalStore` — subscribing to non-React state

From `Fizzi-3D-Website/src/hooks/useMediaQuery.ts`:

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

**Why this matters**:
- `useSyncExternalStore` is the React 18+ way to integrate with browser APIs / external libraries safely (it avoids tearing in concurrent rendering).
- 3 args: `subscribe(onChange)`, `getSnapshot()` (client), `getServerSnapshot()` (SSR).
- Used here for `matchMedia` so the component re-renders when the viewport crosses a breakpoint.

---

## 13. Zustand (tiny global state)

From `Fizzi-3D-Website/src/hooks/useStore.ts`:

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

Usage:

```tsx
const ready = useStore((state) => state.ready);
const setReady = useStore((state) => state.isReady);
```

**Why over Context?** Zustand re-renders only the components that read the slice they subscribed to. Context re-renders **every** consumer when any value changes.

---

## 14. Forwarding refs

From `Fizzi-3D-Website/src/components/FloatingCan.tsx`:

```tsx
import { forwardRef, ReactNode } from "react";
import { Group } from "three";

const FloatingCan = forwardRef<Group, FloatingCanProps>((props, ref) => {
  return <group ref={ref}>...</group>;
});

FloatingCan.displayName = "FloatingCan";

export default FloatingCan;
```

**Why**: a parent (here `Scene`) wants to control the can's `group.position` via GSAP. Refs don't pass through component boundaries normally — `forwardRef` punches a hole so `<FloatingCan ref={can1Ref} />` works.

In React 19+ you can just pass `ref` as a normal prop without `forwardRef`. Both patterns appear in this repo.

---

## 15. Performance basics

- Hoist constants outside the component (`const FLAVORS = [...]`) — they don't get recreated on every render.
- Use `useCallback`/`useMemo` only when measured: needed for stable references passed to memoized children or effect deps.
- For long lists, virtualize (not used in this repo, but ask about it).
- For 3D, prefer **instanced meshes** (one draw call, many positions) — see `Bubbles.tsx`.

---

## 16. CSS in React

This repo uses **Tailwind CSS** (utility classes) primarily, plus some inline `style={}` for dynamic values. Both are valid.

```jsx
<div className="grid h-screen place-items-center text-center" />
<div style={{ background: nextColor }} />
```

Tailwind v4 adds **CSS variables in `@theme`** — see `Personal-Portfolio/src/index.css` for the new syntax.

---

## What this means for your interview

If you can answer all these in your own words:

- "What's the difference between props and state?"
- "Why does `useEffect` have a deps array?"
- "What is the role of `key` in a list?"
- "What's the difference between server and client components in Next.js App Router?"
- "What's the difference between `useRef` and `useState`?"
- "How would you avoid unnecessary re-renders?"

…you've covered ~80% of an SDE 1 React interview. The other 20% is hands-on coding (a small component, maybe a fetch + display, maybe an accessibility tweak) — and every project in this repo is a model answer for "show me a feature".
