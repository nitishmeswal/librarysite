# Spylt-awward-clone / src/components — How It Works

The reusable / smaller building blocks used by `sections/`.

| File | What it is |
| --- | --- |
| `NavBar.jsx` | Tiny floating logo (no menu — just an `<img>` in a `<nav>`). |
| `ClipPathTitle.jsx` | A title element that starts hidden behind a clip-path, ready to reveal. |
| `FlavorTitle.jsx` | The "We have 6 freaking delicious flavors" text block with its own animations. |
| `FlavorSlider.jsx` | The horizontal slider with all 6 flavor cards. |
| `VideoPinSection.jsx` | Circular clipped video that expands to full screen on scroll. |

---

## 1. `NavBar.jsx`

```jsx
const NavBar = () => {
  return (
    <nav className="fixed top-0 left-0 z-50 md:p-9 p-3">
      <img src={`${import.meta.env.BASE_URL}images/nav-logo.svg`} className="md:w-24 w-20" />
    </nav>
  );
};
```

Three useful patterns here:

### `import.meta.env.BASE_URL`

Vite injects `import.meta.env.BASE_URL` based on the `base` config in `vite.config.js`. Default is `/`. If you deploy at `https://example.com/spylt/`, you'd set `base: "/spylt/"` and all references would become `/spylt/images/...`.

**Why use this**: portable across deployment paths. Hardcoding `/images/...` breaks if the site is served from a subpath.

### `fixed top-0 left-0 z-50`

Float the nav over everything. `z-50` ensures it sits above other content. `pointer-events: none` could be added to the `<nav>` if needed (but here the logo is the only interactive part).

### Responsive padding `md:p-9 p-3`

Mobile-first: `p-3` (small padding on mobile), `md:p-9` (larger padding on md+). Tailwind's responsive prefixes always **min-width**: `md:` = "from 768px up", not "less than 768px".

---

## 2. `ClipPathTitle.jsx`

```jsx
const ClipPathTitle = ({ title, color, bg, className, borderColor }) => {
  return (
    <div className="general-title">
      <div
        style={{
          clipPath: "polygon(50% 0, 50% 0, 50% 100%, 50% 100%)",
          borderColor: borderColor,
        }}
        className={`${className} border-[.5vw] text-nowrap opacity-0`}
      >
        <div className="pb-5 md:px-14 px-3 md:pt-0 pt-3" style={{ backgroundColor: bg }}>
          <h2 style={{ color: color }}>{title}</h2>
        </div>
      </div>
    </div>
  );
};
```

### How it gets revealed

This component renders with:
- `clipPath: polygon(50% 0, 50% 0, 50% 100%, 50% 100%)` — degenerate vertical line at center → invisible.
- `opacity: 0` — fully transparent.

Then `BenefitSection.jsx`'s timeline animates it to:
- `opacity: 1`
- `clipPath: polygon(0% 0%, 100% 0, 100% 100%, 0% 100%)` — full rectangle.

The combo (`clipPath` + `opacity`) tweens smoothly → text wipes in horizontally.

### Why two animated properties?

If only `clipPath` animated, the user might briefly see the title at any clipped state but partially. Animating `opacity` from 0 → 1 makes the transition softer.

### Props pattern

```jsx
<ClipPathTitle
  title={"Shelf stable"}
  color={"#faeade"}
  bg={"#c88e64"}
  className={"first-title"}
  borderColor={"#222123"}
/>
```

Every variant of the title (different colors/text) is one props object. **A presentational component with no state** — just renders props.

`className` is passed as a target hook for the parent's GSAP query (`.first-title`). The parent timeline then animates each title by class.

---

## 3. `FlavorTitle.jsx`

```jsx
useGSAP(() => {
  const firstTextSplit = SplitText.create(".first-text-split h1", { type: "chars" });
  const secondTextSplit = SplitText.create(".second-text-split h1", { type: "chars" });

  gsap.from(firstTextSplit.chars, {
    yPercent: 200,
    stagger: 0.02,
    ease: "power1.inOut",
    scrollTrigger: { trigger: ".flavor-section", start: "top 30%" },
  });

  gsap.to(".flavor-text-scroll", {
    duration: 1,
    clipPath: "polygon(0% 0%, 100% 0%, 100% 100%, 0% 100%)",
    scrollTrigger: { trigger: ".flavor-section", start: "top 10%" },
  });

  gsap.from(secondTextSplit.chars, {
    yPercent: 200,
    stagger: 0.02,
    ease: "power1.inOut",
    scrollTrigger: { trigger: ".flavor-section", start: "top 1%" },
  });
});
```

### The three scroll cascades

Each animation uses a different `start` — `30%`, `10%`, `1%` — so they fire in sequence as the user scrolls.

`start: "top 30%"` = trigger top reaches 30% of viewport height (i.e., 30% down from the top of the viewport).

**Without `scrub`**: the animation plays once when start is reached (not reversed when scrolled back unless `toggleActions` is set).

---

## 4. `FlavorSlider.jsx`

Covered fully in the parent guide. Key insight repeated:

```js
const scrollAmount = sliderRef.current.scrollWidth - window.innerWidth;
```

`scrollWidth` = total content width (including overflow). `innerWidth` = viewport width. The difference is how much we need to translate horizontally for the last item to come into view.

### `flex-none`

```html
<div className="... flex-none ${flavor.rotation}">
```

`flex-none` (Tailwind) = `flex: none` = "don't grow, don't shrink, take your specified width". Without this, flex items shrink to fit the container — you'd never have horizontal overflow.

---

## 5. `VideoPinSection.jsx`

Covered in the parent guide. One thing not yet covered:

### Mobile-mode early exit

```jsx
useGSAP(() => {
  if (!isMobile) {
    const tl = gsap.timeline({ scrollTrigger: { ... pin: true } });
    tl.to(".video-box", { clipPath: "circle(100% at 50% 50%)" });
  }
});
```

If `isMobile`, no animation is created. The initial style is also conditional:

```jsx
<div
  style={{
    clipPath: isMobile ? "circle(100% at 50% 50%)" : "circle(6% at 50% 50%)",
  }}
  className="size-full video-box"
>
```

On mobile, the video is already at full circle (fully visible). No scroll-driven reveal. This is a **gracefully-degraded experience** for mobile.

---

## Patterns across this folder

1. **Static `style={}` for initial state**, GSAP for transitions. Inline-style initial state ensures FOUC-free first paint.
2. **`className` is a prop hook**: pass a unique class so the parent timeline can target it (`.first-title`, `.second-title`).
3. **`size-full` (Tailwind)** = `width: 100%; height: 100%;` — saves keystrokes.
4. **Conditional mobile fallback** = don't bother running expensive animations on small screens.
5. **`import.meta.env.BASE_URL`** for path-portable asset URLs.

---

## Build this from scratch in an interview

**Prompt**: "Build a reusable text-reveal-on-scroll component."

```jsx
const RevealText = ({ children, className }) => {
  const ref = useRef();
  useGSAP(() => {
    const split = SplitText.create(ref.current, { type: "chars" });
    gsap.from(split.chars, {
      yPercent: 200,
      stagger: 0.02,
      scrollTrigger: { trigger: ref.current, start: "top 80%" },
    });
  });
  return <span ref={ref} className={className}>{children}</span>;
};

// Usage:
<RevealText className="hero-title">Freaking Delicious</RevealText>
```

15 lines. Reusable. Done.
