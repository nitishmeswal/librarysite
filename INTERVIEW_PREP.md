# SDE 1 Frontend Interview Prep

This file is a **practice deck**: questions you'll probably get, model answers, plus 5 "describe a project you built" pitches grounded in this repo.

---

## Part 1 — Conceptual questions you must crush

### Q1. What happens when a React component re-renders?

**Answer**: When state or props change, React calls the component function again, gets a new tree of JSX, and **diffs** that against the previous tree. It updates only the DOM nodes that actually changed. Hooks like `useState`/`useRef` keep their identity across renders thanks to React internal indexing (which is why hooks must run in the same order — "rules of hooks").

### Q2. `useState` vs `useRef`?

**Answer**: `useState` triggers a re-render when you update it; `useRef` does not. Use state for values the UI depends on; use refs for "I need to remember this between renders but don't want to re-render the component" (e.g. DOM node, timer id, animation handle, mutable counter).

### Q3. Why does `useEffect` have a dependency array?

**Answer**: It tells React when to re-run the effect. Empty `[]` = run once after mount. `[a, b]` = run after mount AND every time `a` or `b` changes. No array = every render. Stale closures (where the effect captures old props/state) are the most common bug — always include in deps everything the effect uses.

### Q4. Controlled vs uncontrolled inputs?

**Answer**:
- **Controlled**: `<input value={state} onChange={e => setState(e.target.value)} />` — React owns the value.
- **Uncontrolled**: `<input defaultValue="x" ref={ref} />` — the DOM owns the value, you read it with `ref.current.value`.
- Controlled is the default in modern React. Uncontrolled is fine for "fire-and-forget" inputs like search boxes or file pickers.

### Q5. What is the `key` prop for?

**Answer**: It tells React how to match elements in a list across re-renders. Without stable keys, React mis-identifies which item is which when the list re-orders, which causes state leaks and incorrect animations. Use a stable id; avoid array index when items can re-order.

### Q6. What's a React Portal?

**Answer**: A way to render a child component into a DOM node outside the parent. Used for modals, tooltips, toasts so they aren't clipped by `overflow:hidden` ancestors. `ReactDOM.createPortal(<Toast />, document.body)`.

### Q7. Server vs Client components (Next.js App Router)?

**Answer**: Server components run on the server (or at build time), can be `async`, can fetch directly, but **cannot use hooks or browser APIs**. They ship zero JS to the browser. Client components are marked with `"use client"`, are bundled to the browser, can be interactive (state, effects, refs). Default is server.

### Q8. What is hydration?

**Answer**: SSR/SSG pages send HTML to the browser so the user sees content fast. Then React downloads the JS, attaches event listeners to the existing DOM, and "becomes interactive". That attachment process is hydration. Mismatch between server-rendered HTML and client tree = hydration error.

### Q9. CSS Box Model?

**Answer**: An element's layout box = `content + padding + border + margin`. With `box-sizing: border-box` (the modern default in resets), `width` includes padding+border, so a `width: 100%` doesn't overflow when you add padding. Margins collapse vertically between sibling block elements.

### Q10. What's `position: fixed/absolute/relative/sticky`?

**Answer**:
- **static** (default) — normal flow.
- **relative** — normal flow, but you can `top/left` to nudge it; remains a positioning context for absolute children.
- **absolute** — removed from flow; positioned relative to nearest positioned ancestor.
- **fixed** — positioned relative to viewport; doesn't scroll.
- **sticky** — relative until it hits a defined threshold (`top: 0`), then fixed within its container.

Sticky drives the pinned 3D canvases in `Fizzi-3D-Website` (see `hero-scene sticky top-0`).

### Q11. What's `transform: perspective(...)` doing?

**Answer**: Adds a vanishing point so child transforms can use `translateZ` and `rotateY/X` in 3D. Without `perspective`, `translateZ` does nothing visible. The `paralax/` project uses this for "depth on mousemove".

### Q12. Difference between `transition` and `animation`?

**Answer**:
- `transition: opacity .3s ease;` — animates when a property changes. You don't define keyframes; the browser interpolates between start and end values.
- `@keyframes spin { 0%{...} 100%{...} } .x { animation: spin 1s infinite; }` — explicit keyframes, runs autonomously.

### Q13. How does `IntersectionObserver` work?

**Answer**: You give it a callback and an element to observe. The browser calls your callback when the element crosses a configurable threshold of visibility relative to a root (viewport by default). It's more efficient than `scroll` events because the browser does the math on the compositor thread. See `Personal-Portfolio/src/hooks/useScrollReveal.js`.

### Q14. How does `requestAnimationFrame` differ from `setInterval`?

**Answer**: `rAF` schedules a callback before the next browser paint (typically 60Hz, but adapts to monitor refresh and visibility). `setInterval` fires on a fixed ms cadence and keeps firing in background tabs. For animations, `rAF` is mandatory — it pauses in background tabs (battery friendly) and syncs with paint.

### Q15. Explain a smooth scroll animation pipeline.

**Answer**: User wheel/touch input → smooth-scroll library (Lenis) interpolates an internal scroll value → on each `requestAnimationFrame`, Lenis updates `window.scrollTo` and emits a `scroll` event → GSAP's ScrollTrigger reads the new scroll position and computes progress for each registered trigger → tweens update their target properties (DOM, Three transforms, video `currentTime`) → browser paints. Wiring: see `Ironhill-section-rebuild/script.js`.

### Q16. Accessibility basics?

**Answer**:
- Semantic HTML first (`<button>`, `<nav>`, `<main>`, `<h1>`-`<h6>`).
- `alt` on every `<img>` (empty string `alt=""` for decorative).
- Keyboard nav: focus visible, `tabindex="0"` if needed.
- ARIA roles only when semantics aren't enough (`role="dialog"`, `aria-expanded`).
- Color contrast ≥ 4.5:1 for body text (WCAG AA).
- Screen-reader-only text with `class="sr-only"` (see `Fizzi-3D-Website/src/slices/Carousel/index.tsx` — `<span className="sr-only">{label}</span>`).

### Q17. What's a closure?

**Answer**: A function that "remembers" variables from the scope where it was created. Every event handler in React captures the props/state at the render time it was defined — that's why "stale closure" bugs happen with `setInterval` + `setState(prev + 1)` inside `useEffect`. The functional update form `setX(prev => prev + 1)` avoids stale closures.

### Q18. `var` vs `let` vs `const`?

**Answer**: `var` is function-scoped, hoisted, can be redeclared (legacy). `let` is block-scoped, can be reassigned. `const` is block-scoped, cannot be reassigned (but the value can still be mutated if it's an object). Use `const` by default, `let` when you need reassignment, never `var`.

### Q19. `==` vs `===`?

**Answer**: `==` coerces types (`"1" == 1` is true). `===` is strict equality (`"1" === 1` is false). Always use `===` unless you specifically want type coercion. There's exactly one case where `==` is useful: `value == null` is true for both `null` and `undefined`.

### Q20. Event bubbling vs capturing?

**Answer**: When you click a `<button>` inside a `<div>` inside `<body>`, the event fires on the body first (capture phase), then bubbles up from the button (bubble phase). `addEventListener(type, handler)` listens during the bubble phase by default. Pass `{ capture: true }` to catch it on the way down. Bubbling is the basis of event delegation — attach one listener on a parent to handle many children.

---

## Part 2 — "Tell me about a project" pitches (60-second answers)

### Pitch A — Personal Portfolio (React 19 + R3F)

> "I built a personal portfolio in React 19 with Vite. The hero has a 3D astronaut rendered with React Three Fiber that loads from a glTF file with `useGLTF` and plays a bobbing animation via `useAnimations`. The camera leans toward the mouse using `easing.damp3` for buttery-smooth follow.
>
> For section reveals I wrote a custom `useScrollReveal` hook on top of IntersectionObserver, which returns `[ref, isVisible]` so any element can opt in. The project section uses Framer Motion's `useMotionValue` + `useSpring` to give the preview-image preview a spring-physics drift as you hover. The whole thing is Tailwind v4 with custom theme colors defined in `@theme`."

### Pitch B — Mojito (GSAP video scrubbing)

> "A landing page where the hero video plays as you scroll. The trick: a `<video muted playsInline preload="auto">` lets us set `currentTime` from JS, and GSAP's ScrollTrigger gives us a `scrub: true, pin: true` timeline whose only tween is `gsap.to(video, { currentTime: video.duration })`. Scroll progress drives video time. SplitText handles the per-character `yPercent: 100` reveal of the title.
>
> The interesting bit is that we wait for `onloadedmetadata` before creating the tween, because `video.duration` is `NaN` until then."

### Pitch C — Spylt (horizontal pinned scroll)

> "Spylt has a section that scrolls horizontally. The technique is: pin the section vertically, then animate its `x` from 0 to `-(scrollWidth - windowWidth)` while scrolling. The scroll distance is set with `end: \`+=${scrollAmount + 1500}px\`` so the user has a long vertical scroll to traverse the horizontal range. SplitText animates each character of the heading with `yPercent: 200, stagger: 0.02` so they swing up sequentially as you enter the section."

### Pitch D — Ironhill (Three.js custom shader + Lenis)

> "Hero overlay with a noise-driven dissolve that reveals the background as you scroll. I wrote a small fragment shader with fractal Brownian motion that combines a `uv.y - uProgress` edge with noise, then `smoothstep` for an anti-aliased edge. `uProgress` is a uniform updated from JS using Lenis-smoothed scroll position. The whole stack is: Lenis → ScrollTrigger.update → uniform update → next animation frame renders."

### Pitch E — Fizzi (Next.js + Prismic + 3D)

> "A Next.js App Router site where the page content is authored in Prismic and rendered through a SliceZone. The interesting architecture is a single `<Canvas>` mounted in the root layout, and each section uses drei's `<View>` to render its own 3D scene into a rectangle of that shared canvas. That keeps GPU usage low.
>
> Five 3D soda cans glTF-imported, choreographed by GSAP scroll-driven timelines that animate their `position.x/y/z` and `rotation.z`. Hundreds of bubbles are drawn with an `InstancedMesh` so it's one draw call regardless of count."

---

## Part 3 — Live-coding flashcards

If they say "build a hero with a scroll-triggered text reveal", here's the 60-line answer:

```jsx
import { useEffect, useRef } from "react";
import gsap from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { SplitText } from "gsap/SplitText";

gsap.registerPlugin(ScrollTrigger, SplitText);

export default function Hero() {
  const titleRef = useRef(null);

  useEffect(() => {
    if (!titleRef.current) return;
    const split = new SplitText(titleRef.current, { type: "chars" });
    const tween = gsap.from(split.chars, {
      yPercent: 100,
      opacity: 0,
      stagger: 0.04,
      ease: "expo.out",
      scrollTrigger: {
        trigger: titleRef.current,
        start: "top 80%",
      },
    });
    return () => {
      tween.kill();
      split.revert();
    };
  }, []);

  return (
    <section className="hero">
      <h1 ref={titleRef} className="text-7xl font-bold overflow-hidden">
        Welcome to my site
      </h1>
    </section>
  );
}
```

If they say "make a list of items where each fades in as you scroll past it":

```jsx
import { useScrollReveal } from "./useScrollReveal";

function Item({ children }) {
  const [ref, visible] = useScrollReveal({ once: true });
  return (
    <li ref={ref} className={`transition-opacity duration-500 ${visible ? "opacity-100" : "opacity-0"}`}>
      {children}
    </li>
  );
}
```

If they say "fetch and display a list":

```jsx
function PostList() {
  const [posts, setPosts] = useState([]);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    fetch("/api/posts")
      .then(r => r.ok ? r.json() : Promise.reject(r.status))
      .then(data => !cancelled && setPosts(data))
      .catch(e => !cancelled && setError(e))
      .finally(() => !cancelled && setLoading(false));
    return () => { cancelled = true; };
  }, []);

  if (loading) return <p>Loading…</p>;
  if (error) return <p>Error: {String(error)}</p>;
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

The `cancelled` flag is the classic "don't set state after unmount" guard.

---

## Part 4 — Things you should NOT skip in revision

- Box model + flexbox + grid (whiteboard a card layout in 2 minutes).
- Tailwind utility classes you use daily (`flex`, `grid`, `items-center`, `gap-`, `p-`, `m-`, `text-`, `bg-`, `rounded-`, `shadow-`, `hover:`, `md:`, `lg:`).
- 3 hooks: `useState`, `useRef`, `useEffect`.
- 1 custom hook (`useScrollReveal` is perfect).
- 1 animation library (GSAP or Framer — pick the one you used most).
- One 3D thing — even just "I know R3F is React-flavored Three.js, here's a 10-line cube" earns big points.
- Be able to draw the request lifecycle of a Next.js App Router page.

You don't need to be an expert in everything. You need to be **clear** about what you know, and **honest** about what you don't — "I haven't worked with Suspense for data fetching yet, but here's how I'd approach it…".

Good luck — you've got this.
