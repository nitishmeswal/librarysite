# GSAP Guide (from zero)

GSAP (GreenSock Animation Platform) is the animation library used in **every** animated site in this repo. This guide gives you exactly enough mental model to read the per-project guides and to whiteboard a scroll animation in an interview.

---

## 1. What GSAP actually does

GSAP changes object properties **over time** with smooth easing. It doesn't care whether the object is a DOM element, a Three.js position, or a plain JS object.

```js
gsap.to(target, vars);
```

- `target` — DOM element, selector string, or any object with numeric properties.
- `vars` — `{ x: 100, duration: 1, ease: "power2.out", onComplete: () => {} }`.

GSAP reads the current value, then animates from "current" to whatever you put in `vars`.

```js
gsap.from(target, vars);   // animate FROM vars → current
gsap.to(target, vars);     // animate current → vars
gsap.fromTo(target, fromVars, toVars);
gsap.set(target, vars);    // jump to value instantly (no animation)
```

---

## 2. Tween properties used in this repo

| Property | Means |
| --- | --- |
| `x`, `y`, `z` | translate (px by default; `xPercent: 100` = 100% of own width) |
| `xPercent`, `yPercent` | translate by % of own size — great for `100` to slide one's-own-height out of view |
| `rotation`, `rotate` | degrees |
| `scale` | uniform scale (1 = normal) |
| `opacity` | 0 → 1 |
| `backgroundColor`, `color`, `fill` | animatable colors |
| `clipPath` | string interpolation, used for `polygon(...)` reveals |
| `duration` | seconds |
| `delay` | seconds before starting |
| `stagger` | seconds between each child when target is an array |
| `ease` | named curve string |
| `repeat` | `-1` = infinite |
| `yoyo` | reverse on each repeat |

---

## 3. Easings — the most useful ones

- `power1.out` / `power2.out` / `power4.out` — smooth decel, the daily driver.
- `power2.inOut` — accel + decel; great for symmetric motion.
- `expo.out` — VERY fast start, slow end. Used a lot in awwwards sites.
- `back.out(1.7)` — overshoots slightly (good for "drop in" effect).
- `circ.inOut` — soft pop-feel.
- `sine.inOut` — gentle pendulum.
- `none` (a.k.a. `linear`) — required for `scrub: true` ScrollTrigger to feel linear.

---

## 4. Timelines

Multiple tweens chained:

```js
const tl = gsap.timeline({ delay: 1 });

tl.to(".hero-content", { opacity: 1, y: 0, ease: "power1.inOut" })
  .to(".hero-text-scroll", { clipPath: "polygon(...)", ease: "circ.out" }, "-=0.5")
  .from(titleSplit.chars, { yPercent: 200, stagger: 0.02, ease: "power2.out" }, "-=0.5");
```

**Position parameters** (the 3rd arg to a chained tween):
- omitted → starts when previous tween ends.
- `"-=0.5"` → start 0.5s **before** the end of the previous tween (overlap).
- `"+=0.5"` → 0.5s **after** the end.
- `"<"` → at the **start** of the previous tween.
- `">"` → at the **end** of the previous tween.
- `0` (a number) → absolute time on the timeline.
- `"label"` → at a named label.

This is from `Spylt-awward-clone/src/sections/HeroSection.jsx`:

```js
tl.to(".hero-content", { opacity: 1, y: 0, ease: "power1.inOut" })
  .to(".hero-text-scroll", {
    duration: 1,
    clipPath: "polygon(0% 0%, 100% 0%, 100% 100%, 0% 100%)",
    ease: "circ.out",
  }, "-=0.5")
  .from(titleSplit.chars, {
    yPercent: 200,
    stagger: 0.02,
    ease: "power2.out",
  }, "-=0.5");
```

---

## 5. ScrollTrigger — scroll-driven animation

```js
import { ScrollTrigger } from "gsap/ScrollTrigger";
gsap.registerPlugin(ScrollTrigger);

gsap.to(target, {
  x: 500,
  scrollTrigger: {
    trigger: ".hero",
    start: "top top",     // when top of trigger hits top of viewport
    end: "bottom top",    // when bottom of trigger hits top of viewport
    scrub: true,          // tie progress to scroll (number = smoothing)
    pin: true,            // pin trigger in place while animating
    markers: true,        // dev-only visual markers
  },
});
```

### `start` / `end` syntax

`"top center"` = *"when the top edge of the trigger reaches the center of the viewport"*. First word = position on the trigger, second word = position on the viewport. Can also use px (`"+=2000"` means "2000px after `start`").

### `scrub`

- `true` → progress = scroll position (no smoothing).
- `1` or `1.5` → smooth (catch-up takes ~N seconds).

### `pin`

The trigger element sticks in place while you scroll through the `start → end` range. The page below it appears to wait.

From `Fizzi-3D-Website/src/slices/Hero/index.tsx`:

```js
const scrollTl = gsap.timeline({
  scrollTrigger: {
    trigger: ".hero",
    start: "top top",
    end: "bottom bottom",
    scrub: 1.5,
  },
});

scrollTl
  .fromTo("body",
    { backgroundColor: "#FDE047" },
    { backgroundColor: "#D9F99D", overwrite: "auto" },
    1,
  )
  .from(".text-side-heading .split-char", {
    scale: 1.3, y: 40, rotate: -25, opacity: 0,
    stagger: 0.1, ease: "back.out(3)", duration: 0.5,
  });
```

**What this does**: as you scroll through the hero section, body background fades yellow → green, and each character of the side heading scales/rotates into place. All driven entirely by scroll position.

---

## 6. SplitText — character/word/line animation

```js
import { SplitText } from "gsap/SplitText";
gsap.registerPlugin(SplitText);

const split = new SplitText(".title", { type: "chars words lines" });
gsap.from(split.chars, { yPercent: 100, stagger: 0.05 });
```

`SplitText` wraps each char/word/line in a `<span>` so you can target it.

From `mojito-awwwards-website/src/components/Hero.tsx`:

```js
const heroSplit = new SplitText(".title", { type: "chars words" });
const paragraphSplit = new SplitText(".subtitle", { type: "lines" });

heroSplit.chars.forEach((char) => char.classList.add("text-gradient"));

gsap.from(heroSplit.chars, {
  yPercent: 100,
  duration: 1.8,
  ease: "expo.out",
  stagger: 0.05,
});

gsap.from(paragraphSplit.lines, {
  opacity: 0,
  yPercent: 100,
  duration: 1.8,
  ease: "expo.out",
  stagger: 0.05,
  delay: 1,
});
```

**What this does**: each character slides up 100% of its own height (was hidden below baseline) into place with a `0.05s` stagger, creating a "typewriter wave" effect.

---

## 7. The `useGSAP` React hook

You should NOT call `gsap.timeline(...)` inside a render — it would re-run every render. The `@gsap/react` package gives you a hook:

```jsx
import { useGSAP } from "@gsap/react";

useGSAP(() => {
  gsap.to(".hero", { opacity: 1, duration: 1 });
}, { dependencies: [ready, isDesktop], scope: containerRef });
```

- The callback runs **once** after mount (like `useEffect`).
- All tweens/timelines created inside it are tracked. On unmount, they're killed automatically.
- `dependencies` re-runs the effect when those change.
- `scope` (optional) — limits selectors to inside that ref.

---

## 8. Lenis — smooth scroll glue

```js
import Lenis from 'lenis'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

const lenis = new Lenis()
function raf(time) {
  lenis.raf(time)
  ScrollTrigger.update()
  requestAnimationFrame(raf)
}
requestAnimationFrame(raf)
lenis.on('scroll', ScrollTrigger.update)
```

**Why**: native scroll is "tick-tick-tick" (synchronous to wheel events). Lenis interpolates scroll smoothly. Wiring `ScrollTrigger.update` makes GSAP and Lenis agree on the current scroll position.

From `Ironhill-section-rebuild/script.js`.

---

## 9. Video scrubbing with ScrollTrigger (mojito pattern)

`<video>` elements have a `.currentTime` property in seconds. Set it = treat the video as a timeline you scrub.

From `mojito-awwwards-website/src/components/Hero.tsx`:

```ts
videoRef.current.onloadedmetadata = () => {
  if (!videoRef.current) return;
  tl.to(videoRef.current, {
    currentTime: videoRef.current.duration,
  });
};
```

The `tl` here has `scrollTrigger: { trigger: 'video', scrub: true, pin: true }` so scrolling drives `currentTime` from 0 → end.

---

## 10. Build-from-scratch checklist (you should be able to do this on a whiteboard)

### A. Fade in a hero on load

```js
gsap.from(".hero", { opacity: 0, y: 30, duration: 1, ease: "power2.out" });
```

### B. Stagger reveal a heading by character

```js
const split = new SplitText(".hero-title", { type: "chars" });
gsap.from(split.chars, { yPercent: 100, stagger: 0.05, ease: "expo.out" });
```

### C. Scroll-pinned section with horizontal scroll

```js
const tl = gsap.timeline({
  scrollTrigger: {
    trigger: ".flavor-section",
    start: "top top",
    end: () => `+=${slider.scrollWidth - window.innerWidth}`,
    scrub: true,
    pin: true,
  },
});
tl.to(".flavor-section", { x: () => -(slider.scrollWidth - window.innerWidth) });
```

(This is exactly the trick used in `Spylt-awward-clone/src/components/FlavorSlider.jsx`.)

### D. Scroll-linked color change

```js
gsap.timeline({
  scrollTrigger: { trigger: ".hero", start: "top top", end: "bottom bottom", scrub: 1 },
}).fromTo("body", { backgroundColor: "#FDE047" }, { backgroundColor: "#D9F99D" });
```

### E. In React with useGSAP

```jsx
const container = useRef();
useGSAP(() => {
  gsap.from(".heading", { y: 30, opacity: 0 });
}, { scope: container });
return <div ref={container}><h1 className="heading">Hi</h1></div>;
```

---

## 11. Mental model summary

- **`to`/`from`/`fromTo`/`set`** — single tweens.
- **`timeline()`** — sequence tweens with position parameters (`"<"`, `"-=0.5"`, etc.).
- **`scrollTrigger`** — bind tween/timeline to scroll position with `start`/`end`/`scrub`/`pin`.
- **`SplitText`** — chop text into chars/words/lines so you can animate each.
- **`useGSAP`** — React-safe wrapper that handles cleanup.
- **Lenis** — smooth scroll; wire its `raf` to `ScrollTrigger.update`.

If you can sketch all 5 of those in code on a whiteboard, you're ready for any animation question they'd throw at an SDE 1.
