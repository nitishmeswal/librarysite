# Spylt-awward-clone / src/sections — How It Works

Each section file = one full-screen section of the page. They all follow the same shape:

1. `useGSAP(() => { /* animations */ })` — set up timelines/ScrollTriggers.
2. Return JSX with target class names that the animations reference.

We've already covered most sections in the parent `Spylt-awward-clone/HOW_IT_WORKS.md`. This file goes line-by-line on the smaller ones we didn't fully cover and provides build-from-scratch checklists.

---

## Section reference table

| File | Animation type | Key trick |
| --- | --- | --- |
| `HeroSection.jsx` | Intro timeline + scrub-rotate | SplitText chars + clip-path reveal |
| `MessageSection.jsx` | Scrub color sweep + line reveal | Word-by-word color tween via stagger |
| `FlavorSection.jsx` | Horizontal pinned scroll | `scrollWidth - innerWidth` translation |
| `NutritionSection.jsx` | Sequential text reveals | `useEffect` derives list from media query |
| `BenefitSection.jsx` | Sequential clip-path titles | Per-title polygon animation |
| `TestimonialSection.jsx` | Pinned + cards + hover-video | Array of refs for videos |
| `FooterSection.jsx` | None (static + mobile/desktop swap) | `useMediaQuery` for img vs video |

---

## 1. `FooterSection.jsx` deep dive

The simplest one — no GSAP, just responsive logic.

```jsx
import { useMediaQuery } from "react-responsive";

const FooterSection = () => {
  const isMobile = useMediaQuery({ query: "(max-width: 768px)" });

  return (
    <section className="footer-section">
      <img src={`${base}images/footer-dip.png`} className="w-full object-cover -translate-y-1" />

      <div className="2xl:h-[110dvh] relative md:pt-[20vh] pt-[10vh]">
        <h1 className="general-title text-center text-milk py-5">#CHUGRESPONSIBLY</h1>

        {isMobile ? (
          <img src={`${base}images/footer-drink.png`} className="absolute top-0 object-contain" />
        ) : (
          <video
            src={`${base}videos/splash.mp4`}
            autoPlay playsInline muted
            className="absolute top-0 object-contain mix-blend-lighten"
          />
        )}

        {/* social icons, links, newsletter, copyright */}
      </div>
    </section>
  );
};
```

### Things worth pulling out

- **`useMediaQuery({ query: "(max-width: 768px)" })`** — returns `true` if viewport matches. Hook re-runs on resize.
- **`mix-blend-lighten`** CSS — blends the video with its background using the "lighten" mode (keeps brighter pixels). Used here so the white splash overlays naturally on the brown background.
- **`-translate-y-1`** — Tailwind for `transform: translateY(-1px)`. Hides a 1px gap that browsers sometimes show between adjacent images.
- **`dvh` unit** (`110dvh`) — "dynamic viewport height". On mobile, the browser address bar can shrink the viewport. `dvh` adapts; `vh` is the static height including the address bar.
- **`mt-[20vh]` arbitrary value syntax** — Tailwind v3 lets you use any value inside `[]` brackets.

---

## 2. `HeroSection.jsx` — interview-prep walkthrough

Already covered in parent guide. Here's the **build-from-scratch checklist** for an interview:

### Recreate the hero in 5 minutes

```html
<section class="hero" style="height: 100vh; background: #000;">
  <h1 class="hero-title" style="font-size: 8rem; color: white;">Freaking Delicious</h1>
</section>
```

```js
gsap.registerPlugin(SplitText);

const split = SplitText.create(".hero-title", { type: "chars" });
gsap.from(split.chars, {
  yPercent: 200,
  stagger: 0.02,
  duration: 0.6,
  ease: "power2.out",
});
```

Done. 8 lines for the awwwards "letters fly up" effect.

To add the scrub-rotate:

```js
gsap.registerPlugin(ScrollTrigger);

gsap.to(".hero", {
  rotate: 7,
  scale: 0.9,
  yPercent: 30,
  scrollTrigger: { trigger: ".hero", start: "1% top", end: "bottom top", scrub: true },
});
```

Another 6 lines.

That's it. The whole hero feel = 14 lines of GSAP.

---

## 3. `MessageSection.jsx` — interview-prep walkthrough

The **color sweep on scroll** is the showpiece. Recreate:

```html
<p class="msg">Stir up your fearless past and fuel your future</p>
```

```js
const split = SplitText.create(".msg", { type: "words" });

gsap.to(split.words, {
  color: "#faeade",     // bright target color
  stagger: 1,           // long stagger value
  scrollTrigger: {
    trigger: ".msg",
    start: "top center",
    end: "bottom center",
    scrub: true,
  },
});
```

**Why `stagger: 1`?** With `scrub: true`, the stagger doesn't represent real seconds — it represents **scroll progress slots**. A big stagger like 1 spreads the words out across the entire trigger range so each word's color change happens at its own scroll point.

---

## 4. `FlavorSection.jsx` — interview-prep walkthrough

The **horizontal pinned scroll** is the showpiece. Recreate:

```html
<section class="flavor-section" style="height: 100vh; display: flex; align-items: center;">
  <div class="slider" style="display: flex; gap: 2rem;">
    <div class="card">1</div>
    <div class="card">2</div>
    <div class="card">3</div>
    <div class="card">4</div>
    <div class="card">5</div>
  </div>
</section>
```

```js
const slider = document.querySelector(".slider");
const amount = slider.scrollWidth - window.innerWidth;

gsap.to(".slider", {
  x: -amount,
  scrollTrigger: {
    trigger: ".flavor-section",
    start: "top top",
    end: `+=${amount}px`,
    scrub: true,
    pin: true,
  },
});
```

### What's happening

1. The section is pinned at the top while user scrolls.
2. Vertical scroll distance equal to `amount` is required to "release" the pin.
3. During that distance, the `.slider` translates from `x: 0` to `x: -amount`.
4. Visually: user scrolls down → cards move left → reaches last card → pin released → vertical scroll resumes.

---

## 5. `BenefitSection.jsx` — interview-prep walkthrough

The **sequential clip-path reveal** is the showpiece. Recreate:

```html
<div class="title" style="clip-path: polygon(50% 0, 50% 0, 50% 100%, 50% 100%);">
  Shelf stable
</div>
<div class="title" style="clip-path: polygon(50% 0, 50% 0, 50% 100%, 50% 100%);">
  Protein + Caffeine
</div>
```

```js
const tl = gsap.timeline({
  scrollTrigger: { trigger: ".titles", start: "top center", end: "bottom center", scrub: true },
});

document.querySelectorAll(".title").forEach((el) => {
  tl.to(el, { clipPath: "polygon(0 0, 100% 0, 100% 100%, 0 100%)" });
});
```

Each `.to()` is added sequentially → first title finishes, then second starts, etc. With `scrub: true`, scroll position drives all of them.

---

## 6. `TestimonialSection.jsx` — interview-prep walkthrough

The **video-cards-on-hover-play** is the showpiece. Recreate:

```jsx
function TestimonialCards({ videos }) {
  const refs = useRef([]);
  return (
    <div className="cards">
      {videos.map((src, i) => (
        <video
          key={i}
          ref={(el) => (refs.current[i] = el)}
          src={src}
          muted
          loop
          playsInline
          onMouseEnter={() => refs.current[i].play()}
          onMouseLeave={() => refs.current[i].pause()}
        />
      ))}
    </div>
  );
}
```

That's it. The whole pattern:
1. `useRef([])` to hold an array.
2. Callback ref `(el) => (refs.current[i] = el)` to store at index i.
3. `onMouseEnter` / `onMouseLeave` to call methods on the DOM nodes.

---

## Common pitfalls in this codebase (interview-talking-point gold)

- **No global ScrollTrigger config**: e.g., they could `ScrollTrigger.config({ ignoreMobileResize: true })` to avoid recalc on iOS address-bar shrink. They don't.
- **No `markers: true` during development**: useful for debugging start/end positions visually. Always set `markers: true` while developing, remove for production.
- **No `gsap.context()`**: in a real React app with mountable/unmountable components, wrap animations in `gsap.context(() => { /* */ }, scopeRef)` so cleanup is clean. `useGSAP` does this for you automatically. The codebase uses `useGSAP` ✓ correctly.
- **`useEffect` derive-state anti-pattern** in NutritionSection — could be `const lists = isMobile ? slice : full` inline. Real interview gold.
- **`useGSAP` deps**: by default no deps means animations are created once and reverted on unmount. If your animations depend on state, pass `{ dependencies: [state] }` and `useGSAP` will re-run.

---

## What this folder teaches you

- **Section composition pattern**: every section is self-contained with its own `useGSAP` block.
- **Scrub timelines** that turn scroll into animation.
- **SplitText for any per-element animation**.
- **Position parameters** for orchestrating timelines.
- **clipPath polygons** as the "any-shape reveal" primitive.
- **Mobile-aware animation gates** (`if (!isMobile)`).

Continue to [components/HOW_IT_WORKS.md](../components/HOW_IT_WORKS.md) for the dumb building blocks.
