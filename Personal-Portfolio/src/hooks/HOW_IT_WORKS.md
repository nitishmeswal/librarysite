# Personal-Portfolio / src/hooks — How It Works

This folder contains exactly one file: a **custom React hook that wraps the IntersectionObserver browser API** so any component can have a "fade in when scrolled into view" behavior with one line.

## File: `useScrollReveal.js`

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

---

## Line-by-line

| Line | Meaning |
| --- | --- |
| `useState(false)` | Track whether the element has scrolled into view. Default = not visible. |
| `useRef(null)` | The DOM node we'll observe. Caller attaches it via `<div ref={ref}>`. |
| `useEffect(() => { ... }, [options.once, options.threshold, options.rootMargin])` | Run after mount. Re-run only if these options change. |
| `new IntersectionObserver(callback, options)` | Browser API. Watches an element and calls the callback when its visibility crosses thresholds. |
| `([entry]) => { ... }` | The callback receives an array of entries (one per observed element). We destructure the first. |
| `entry.isIntersecting` | `true` if any part of the element is currently within the threshold of visibility. |
| `setIsVisible(true)` | Trigger a re-render so consumers can show/hide content. |
| `if (options.once) observer.unobserve(entry.target)` | "Fire and forget" mode — once visible, stop watching so re-entering doesn't re-trigger. |
| `else if (!options.once) setIsVisible(false)` | If NOT once-mode and the element leaves the viewport, hide it again. |
| `threshold: options.threshold \|\| 0.1` | Trigger when 10% of element is visible by default. Pass `0.5` for half, etc. |
| `rootMargin: options.rootMargin \|\| '0px'` | Virtual margin around viewport. Negative values shrink the trigger zone (`'-100px'` = trigger when 100px deeper than viewport edge). |
| `observer.observe(currentRef)` | Start watching. |
| Cleanup `observer.unobserve(currentRef)` | Runs on unmount or when deps change. Critical to avoid memory leaks. |
| `return [ref, isVisible]` | Tuple, mimicking `useState`'s `[value, setter]` shape. |

---

## Why a tuple return?

```jsx
const [titleRef, titleVisible] = useScrollReveal({ once: true });

return <h2 ref={titleRef} className={titleVisible ? 'visible' : 'invisible'}>Hello</h2>;
```

By returning `[ref, isVisible]` (instead of `{ ref, isVisible }`), the caller can **name them whatever they want** with destructuring. This matters when you use the hook several times in one component:

```jsx
const [titleRef, titleVisible] = useScrollReveal({ once: true });
const [gridRef, gridVisible] = useScrollReveal({ once: true });
```

If we returned an object, you'd have to do `const { ref: titleRef, isVisible: titleVisible }` every time. Tuples are cleaner.

---

## Why `once` mode matters

Without `once`, the observer keeps firing each time you scroll past. With `once`:
- The reveal animation only plays once (no flicker if the user scrolls fast).
- You stop observing → frees up browser bookkeeping.

When NOT to use once:
- Sticky elements that fade in/out as you scroll past sections.
- Lazy-load image sentinels (you may want them to re-trigger if the layout changes).

---

## How it's used in the project

Search `useScrollReveal` in `sections/About.jsx` and `sections/Contact.jsx`:

```jsx
const [titleRef, titleVisible] = useScrollReveal({ once: true });
// ...
<h2 ref={titleRef} className={`scroll-reveal ${titleVisible ? 'scroll-reveal-visible' : ''}`}>
  About Me
</h2>
```

`.scroll-reveal` (in `index.css`) sets `opacity: 0; transform: translateY(2rem);`. Adding `.scroll-reveal-visible` sets `opacity: 1; transform: translateY(0);`. The CSS `transition` animates between them.

---

## Common gotchas

1. **`ref.current` is null on first render**. The effect runs after the first render, so `ref.current` is already populated by then. Just don't try to access it from the render body.
2. **Putting `options` directly in deps**. We unpack `options.once`, `options.threshold`, `options.rootMargin` so the deps array contains primitives. If we had `[options]` and the caller passed a new object literal each render (`useScrollReveal({ once: true })`), the effect would re-run every render.
3. **No SSR safety**. `IntersectionObserver` doesn't exist on the server. In Next.js, you'd guard with `if (typeof window !== "undefined")` or only use this in client components.

---

## Build a "useInView" hook in an interview

The above is essentially [react-intersection-observer](https://www.npmjs.com/package/react-intersection-observer)'s `useInView`. Knowing how to write it yourself is a great interview signal — it shows you understand effects, refs, browser APIs, and cleanup.

Minimal version (4 lines smaller than above):

```js
export function useInView(options = {}) {
  const ref = useRef(null);
  const [inView, setInView] = useState(false);
  useEffect(() => {
    const node = ref.current;
    if (!node) return;
    const obs = new IntersectionObserver(([e]) => setInView(e.isIntersecting), options);
    obs.observe(node);
    return () => obs.disconnect();
  }, []);
  return [ref, inView];
}
```

`observer.disconnect()` is the easy way to stop watching everything — doesn't matter that we only watched one node.
