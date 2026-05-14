# Spylt-awward-clone/ — How It Works

An **awwwards-style scroll-storytelling site** for a fictional drink "SPYLT". This is the **gold-standard GSAP project** in this repo. Read this guide carefully — almost everything you'd want to know about ScrollTrigger + SplitText + pinning is in here.

## Stack
- **React 19 + Vite** (no Next.js).
- **GSAP 3** + **ScrollTrigger** + **SplitText** plugins.
- **`@gsap/react`** for the `useGSAP` hook (auto-cleanup in React).
- **`react-responsive`** for `useMediaQuery`.
- **Tailwind CSS** + custom CSS.

## Folder map
```
Spylt-awward-clone/
├── public/             # images, videos, svgs
├── src/
│   ├── components/     # NavBar, ClipPathTitle, FlavorTitle, FlavorSlider, VideoPinSection
│   ├── sections/       # Hero, Message, Flavor, Nutrition, Benefit, Testimonial, Footer
│   ├── constants/      # flavorlists, nutrientLists, cards
│   ├── App.jsx
│   ├── main.jsx
│   └── index.css
├── index.html
└── vite.config.js
```

---

## 1. The entry — `main.jsx` + `App.jsx`

```jsx
// main.jsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.jsx'

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

```jsx
// App.jsx
import { ScrollTrigger } from "gsap/all";
import gsap from "gsap";
import { useEffect } from "react";
// ... section imports

gsap.registerPlugin(ScrollTrigger);

const App = () => {
  useEffect(() => {
    const onLoad = () => ScrollTrigger.refresh();
    if (document.readyState === "complete") {
      onLoad();
    } else {
      window.addEventListener("load", onLoad);
      return () => window.removeEventListener("load", onLoad);
    }
  }, []);

  return (
    <main>
      <NavBar />
      <HeroSection />
      <MessageSection />
      <FlavorSection />
      <NutritionSection />
      <BenefitSection />
      <TestimonialSection />
      <FooterSection />
    </main>
  );
};
```

### Why `ScrollTrigger.refresh()` on load?

ScrollTrigger calculates start/end positions based on element positions on the page. But:
- Images may not be loaded yet when ScrollTrigger initializes.
- Fonts may not be loaded → text reflows after layout.

So you tell ScrollTrigger to **recalculate** once everything's loaded. Common bug if you forget this: triggers fire at the wrong scroll positions on first load, fine after a window resize.

The `if (document.readyState === "complete")` check handles the case where the React app mounts **after** the `load` event already fired (e.g., when navigating into the page client-side).

### `gsap.registerPlugin(ScrollTrigger)`

Run **once**, at module load. GSAP needs to know which plugins are available so they tree-shake correctly. Doing this in `App.jsx` is fine because the file is imported once.

---

## 2. `HeroSection.jsx` — split text, scroll-pinned hero, clip-path reveal

```jsx
useGSAP(() => {
  const titleSplit = SplitText.create(".hero-title", { type: "chars" });

  const tl = gsap.timeline({ delay: 1 });

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

  const heroTl = gsap.timeline({
    scrollTrigger: {
      trigger: ".hero-container",
      start: "1% top",
      end: "bottom top",
      scrub: true,
    },
  });
  heroTl.to(".hero-container", { rotate: 7, scale: 0.9, yPercent: 30, ease: "power1.inOut" });
});
```

### `useGSAP` vs `useEffect`

```jsx
import { useGSAP } from "@gsap/react";

useGSAP(() => { /* gsap code */ });
```

`useGSAP` is a **drop-in replacement** for `useEffect` for GSAP code. It auto-tracks every animation created inside and calls `.revert()` on cleanup (component unmount). Without it, you'd leak animations and ScrollTriggers across re-renders.

### `SplitText.create(".hero-title", { type: "chars" })`

Splits the text in `.hero-title` into per-character `<div>` wrappers. The plugin DOM-rewrites:

```html
<h1 class="hero-title">Freaking Delicious</h1>
```

becomes (approximately):

```html
<h1 class="hero-title">
  <div class="split-line">
    <div class="char">F</div>
    <div class="char">r</div>
    <div class="char">e</div>
    ...
  </div>
</h1>
```

The returned `titleSplit.chars` is an array of those `.char` elements. Now you can animate them individually with stagger.

`type` options:
- `"chars"` — characters
- `"words"` — words (each wrapped)
- `"lines"` — visual lines (computed after layout)
- `"words, lines"` — both, gives you `splitObj.words` and `splitObj.lines`

### The intro timeline

`gsap.timeline({ delay: 1 })` — wait 1s after page load, then play.

`.to(".hero-content", { opacity: 1, y: 0 })` — fade in the wrapper.

`.to(".hero-text-scroll", { clipPath: "polygon(0% 0%, 100% 0%, 100% 100%, 0% 100%)" }, "-=0.5")` — animate to a "fully revealed" clip-path. The starting state was `polygon(50% 0, 50% 0, 50% 100%, 50% 100%)` (set in inline style) — a zero-width vertical line at center. End state is full rectangle — text "wipes in" horizontally.

The `"-=0.5"` is the **position parameter**: start this tween 0.5s BEFORE the previous one ends (negative = overlap).

`.from(titleSplit.chars, { yPercent: 200, stagger: 0.02 }, "-=0.5")` — `.from` means "animate FROM these values TO current values". So every char starts 200% below its position and slides up. `stagger: 0.02` = 20ms between each char.

### The scrub timeline (scroll-driven)

```jsx
const heroTl = gsap.timeline({
  scrollTrigger: {
    trigger: ".hero-container",
    start: "1% top",
    end: "bottom top",
    scrub: true,
  },
});
heroTl.to(".hero-container", { rotate: 7, scale: 0.9, yPercent: 30 });
```

- `trigger: ".hero-container"` — observe this element.
- `start: "1% top"` — **start when 1% from the top of trigger reaches top of viewport**. (Format: `"<trigger-position> <viewport-position>"`.)
- `end: "bottom top"` — end when bottom of trigger reaches top of viewport (i.e., trigger is fully scrolled past).
- `scrub: true` — link the timeline's progress to the scroll progress. The animation is **driven by scroll**, not by time.

As you scroll past the hero, it rotates 7°, shrinks to 0.9, drops 30% — feels like the section is being put away into a drawer.

### The `clipPath` trick

`clipPath: "polygon(...)"` defines an arbitrary polygonal clipping region. Common patterns:
- `polygon(0 0, 100% 0, 100% 100%, 0 100%)` = full rectangle (no clip).
- `polygon(50% 0, 50% 0, 50% 100%, 50% 100%)` = degenerate zero-width line at center.
- Animating between them = "wipe reveal" effect.

GSAP can animate the **string** `clipPath` value if both states are valid polygons with the same number of points. Magic.

---

## 3. `MessageSection.jsx` — text-by-text color sweep

```jsx
useGSAP(() => {
  const firstMsgSplit = SplitText.create(".first-message", { type: "words" });
  const secMsgSplit = SplitText.create(".second-message", { type: "words" });
  const paragraphSplit = SplitText.create(".message-content p", {
    type: "words, lines",
    linesClass: "paragraph-line",
  });

  gsap.to(firstMsgSplit.words, {
    color: "#faeade",
    ease: "power1.in",
    stagger: 1,
    scrollTrigger: {
      trigger: ".message-content",
      start: "top center",
      end: "30% center",
      scrub: true,
    },
  });

  // ... similar for secMsgSplit ...

  const paragraphTl = gsap.timeline({
    scrollTrigger: { trigger: ".message-content p", start: "top center" },
  });
  paragraphTl.from(paragraphSplit.words, {
    yPercent: 300, rotate: 3, ease: "power1.inOut", duration: 1, stagger: 0.01,
  });
});
```

### Per-word color sweep tied to scroll

`.first-message` is split into individual word divs. The tween animates each word's `color` to `#faeade` with `stagger: 1` — but combined with `scrub: true`, the stagger maps to scroll position:
- When you scroll to the start of trigger, the first word starts changing color.
- As scroll progresses, the next word starts.
- It's like **highlighting words as you read**.

This is one of the most recognizable awwwards-site effects. Burn the pattern into memory.

### The `lines` split for paragraph reveal

```jsx
SplitText.create(".message-content p", { type: "words, lines", linesClass: "paragraph-line" });
```

`linesClass: "paragraph-line"` adds a class to each line wrapper. The CSS for `.paragraph-line` typically sets `overflow: hidden` so words sliding up from `yPercent: 300` stay clipped within their line — they only become visible as they slide into the line's box.

This is the **"text rising from below" reveal** you see everywhere. Trick = `overflow: hidden` on line wrappers + `yPercent: 300 → 0`.

---

## 4. `FlavorSection.jsx` + `FlavorSlider.jsx` — horizontal scroll-driven slider

```jsx
const FlavorSlider = () => {
  const sliderRef = useRef();
  const isTablet = useMediaQuery({ query: "(max-width: 1024px)" });

  useGSAP(() => {
    const scrollAmount = sliderRef.current.scrollWidth - window.innerWidth;

    if (!isTablet) {
      const tl = gsap.timeline({
        scrollTrigger: {
          trigger: ".flavor-section",
          start: "2% top",
          end: `+=${scrollAmount + 1500}px`,
          scrub: true,
          pin: true,
        },
      });

      tl.to(".flavor-section", {
        x: `-${scrollAmount + 1500}px`,
        ease: "power1.inOut",
      });
    }
    // ... text scroll animations ...
  });
  ...
};
```

### The pin-then-translate-horizontal pattern

This is **the** trick for "horizontal scroll" sections:

1. **Measure**: `sliderRef.current.scrollWidth - window.innerWidth` = how far we need to translate horizontally to show all content.
2. **Pin**: `pin: true` keeps the section "stuck" while we scroll.
3. **Translate horizontally**: as the user scrolls vertically, the section moves horizontally by `-scrollAmount` pixels.
4. **`end: `+=${scrollAmount + 1500}px``** — the section is pinned for that scroll distance. Vertical scroll → horizontal motion.

User experience: scroll down → slider moves left → eventually unpin → continue scrolling normally.

### Conditional on `!isTablet`

The horizontal scroll only fires on desktop. On tablet/mobile, the slider is just a regular vertical scroll. Common pattern for awwwards-style sites: complex effects on desktop, simplified on mobile.

---

## 5. `BenefitSection.jsx` — sequential clip-path reveals

```jsx
const revealTl = gsap.timeline({
  delay: 1,
  scrollTrigger: {
    trigger: ".benefit-section",
    start: "top 60%",
    end: "top top",
    scrub: 1.5,
  },
});

revealTl
  .to(".benefit-section .first-title",  { clipPath: "polygon(0% 0%, 100% 0, 100% 100%, 0% 100%)" })
  .to(".benefit-section .second-title", { clipPath: "polygon(0% 0%, 100% 0, 100% 100%, 0% 100%)" })
  .to(".benefit-section .third-title",  { clipPath: "polygon(0% 0%, 100% 0, 100% 100%, 0% 100%)" })
  .to(".benefit-section .fourth-title", { clipPath: "polygon(0% 0%, 100% 0, 100% 100%, 0% 100%)" });
```

Each title is wrapped by `<ClipPathTitle>`:

```jsx
const ClipPathTitle = ({ title, color, bg, className, borderColor }) => {
  return (
    <div className="general-title">
      <div
        style={{
          clipPath: "polygon(50% 0, 50% 0, 50% 100%, 50% 100%)",  // start: collapsed
          borderColor: borderColor,
        }}
        className={`${className} border-[.5vw] opacity-0`}
      >
        <div className="pb-5 md:px-14" style={{ backgroundColor: bg }}>
          <h2 style={{ color: color }}>{title}</h2>
        </div>
      </div>
    </div>
  );
};
```

Each title starts with `clipPath: polygon(50% 0, ...)` collapsed to a vertical line + `opacity: 0`. The timeline tweens each to the full polygon sequentially as you scroll.

### `scrub: 1.5`

`scrub: 1.5` instead of `scrub: true` adds a 1.5-second "lag" — the animation catches up to scroll position over 1.5 seconds. Feels less twitchy than `scrub: true`. Used widely on awwwards sites.

---

## 6. `TestimonialSection.jsx` — pinned scroll with cards rising

```jsx
useGSAP(() => {
  gsap.set(".testimonials-section", { marginTop: "-140vh" });  // overlap previous

  const tl = gsap.timeline({
    scrollTrigger: {
      trigger: ".testimonials-section",
      start: "top bottom",
      end: "200% top",
      scrub: true,
    },
  });
  tl.to(".testimonials-section .first-title", { xPercent: 70 })
    .to(".testimonials-section .sec-title",   { xPercent: 25 }, "<")
    .to(".testimonials-section .third-title", { xPercent: -50 }, "<");

  const pinTl = gsap.timeline({
    scrollTrigger: {
      trigger: ".testimonials-section",
      start: "10% top",
      end: "200% top",
      scrub: 1.5,
      pin: true,
    },
  });
  pinTl.from(".vd-card", {
    yPercent: 150, stagger: 0.2, ease: "power1.inOut",
  });
});
```

### `gsap.set(...)` for instant style writes

`gsap.set(target, props)` applies styles immediately (no animation). Used here to set negative margin so this section overlaps the previous — creates a layered effect.

### `"<"` position parameter

`"<"` means "start at the same time as the previous tween" (i.e., align with previous tween's START).

`">"` would mean "after the previous tween's END" (default).

`"<0.5"` = "start 0.5s after previous tween's start".

This is **the** way to orchestrate multiple animations to play together.

### Video play/pause on hover

```jsx
const vdRef = useRef([]);

const handlePlay = (index) => vdRef.current[index].play();
const handlePause = (index) => vdRef.current[index].pause();

{cards.map((card, index) => (
  <div key={index} onMouseEnter={() => handlePlay(index)} onMouseLeave={() => handlePause(index)}>
    <video ref={(el) => (vdRef.current[index] = el)} src={card.src} muted loop playsInline />
  </div>
))}
```

### **Array of refs** pattern

```jsx
const vdRef = useRef([]);
// In render:
<video ref={(el) => (vdRef.current[index] = el)} ... />
```

`useRef([])` initializes `vdRef.current` to an empty array. The callback ref `(el) => (vdRef.current[index] = el)` stores each video element at its index. Now `vdRef.current[3]` is the 4th video element.

This is the React way to "give me access to a list of DOM nodes". Alternative: `useRef(new Array(cards.length))`, or `useRef(new Map())` if order isn't stable.

### Video attributes for autoplay-safe behavior

`muted loop playsInline` — `playsInline` is essential for iOS Safari to play inline rather than going fullscreen.

---

## 7. `BenefitSection.jsx`'s `VideoPinSection.jsx` — circular clip-path reveal

```jsx
const VideoPinSection = () => {
  const isMobile = useMediaQuery({ query: "(max-width: 768px)" });

  useGSAP(() => {
    if (!isMobile) {
      const tl = gsap.timeline({
        scrollTrigger: {
          trigger: ".vd-pin-section",
          start: "-15% top",
          end: "200% top",
          scrub: 1.5,
          pin: true,
        },
      });
      tl.to(".video-box", {
        clipPath: "circle(100% at 50% 50%)",
        ease: "power1.inOut",
      });
    }
  });

  return (
    <section className="vd-pin-section">
      <div
        style={{ clipPath: isMobile ? "circle(100% at 50% 50%)" : "circle(6% at 50% 50%)" }}
        className="size-full video-box"
      >
        <video src={`${import.meta.env.BASE_URL}videos/pin-video.mp4`} playsInline muted loop autoPlay />
        ...
      </div>
    </section>
  );
};
```

### Circular clip path

`clipPath: "circle(6% at 50% 50%)"` = a 6% radius circle centered at 50% 50%. The video is visible only inside that circle.

Animating to `circle(100%)` expands the circle to fill the viewport → video reveals.

`circle(% at X Y)` is the syntax. Y position is from the top.

---

## 8. `NutritionSection.jsx` — conditional state on viewport

```jsx
const [lists, setLists] = useState(nutrientLists);

useEffect(() => {
  if (isMobile) setLists(nutrientLists.slice(0, 3));
  else setLists(nutrientLists);
}, [isMobile]);
```

### Pattern: derive state from media query

When media query changes (window resize crosses the 768px breakpoint), `isMobile` changes → effect runs → `lists` updates → re-render with shorter list.

**Caveat**: This is a classic "derived state" trap. You could just compute `const lists = isMobile ? nutrientLists.slice(0, 3) : nutrientLists` inline — no state needed, since `lists` is always a function of `isMobile`. In a real review, suggest this refactor.

---

## 9. CSS, Tailwind, conventions

Open `src/index.css` to see custom utility classes used everywhere:
- `flex-center` = `flex items-center justify-center` (most-used helper).
- `col-center` = `flex flex-col items-center justify-center`.
- `abs-center` = `absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2`.
- `general-title` = sets shared font for the giant section titles.

These custom utilities live in `@layer components` (Tailwind directive). Extract common compounds into utilities to avoid 50-class `className=""` strings.

---

## What this project teaches you

- **ScrollTrigger fundamentals**: `trigger`, `start`, `end`, `scrub`, `pin`, `markers`.
- **Position parameter**: `"-=0.5"` (overlap), `"<"` (sync start), `">"` (sequential).
- **SplitText**: split into chars/words/lines, animate each independently.
- **clipPath as animation primitive**: polygon and circle interpolation.
- **Pinned horizontal scroll** with `scrollWidth - innerWidth`.
- **Video hover play/pause** with an array of refs.
- **`useGSAP` for auto-cleanup** in React.
- **`gsap.set()` for instant initial styles**.
- **`ScrollTrigger.refresh()` on `load`** to fix positions after assets load.
- **Mobile-vs-desktop conditional animation** using `useMediaQuery`.

This project alone is worth talking about for an entire interview round. **Memorize the structure of one ScrollTrigger config** — you'll be able to recreate any awwwards-style scroll story.

See subfolders for deeper breakdowns:
- [src/sections/HOW_IT_WORKS.md](./src/sections/HOW_IT_WORKS.md)
- [src/components/HOW_IT_WORKS.md](./src/components/HOW_IT_WORKS.md)
- [src/constants/HOW_IT_WORKS.md](./src/constants/HOW_IT_WORKS.md)
