# mojito-awwwards-website / src/components — How It Works

Each component is one full section. We covered most in the parent guide; this file goes line-by-line on patterns and provides build-from-scratch checklists.

---

## Section catalog

| File | Showpiece | Difficulty (interview re-creation) |
| --- | --- | --- |
| `Navbar.tsx` | `scrollTrigger` background fade | ⭐ |
| `Hero.tsx` | Scroll-scrubbed video | ⭐⭐⭐⭐⭐ |
| `Cocktails.tsx` | Parallax leaves on scroll | ⭐⭐ |
| `About.tsx` | SplitText words + grid stagger | ⭐⭐ |
| `Art.tsx` | CSS mask reveal | ⭐⭐⭐⭐ |
| `Menu.tsx` | State-driven slider with animated swaps | ⭐⭐⭐ |
| `Contact.tsx` | SplitText words + leaf parallax | ⭐⭐ |

---

## 1. `Navbar.tsx` — build from scratch

Goal: nav that's transparent over hero, semi-transparent dark after scrolling past.

```jsx
gsap.fromTo("nav",
  { backgroundColor: "transparent" },
  {
    backgroundColor: "rgba(0,0,0,0.5)",
    backdropFilter: "blur(10px)",
    scrollTrigger: { trigger: "nav", start: "bottom top" },
  }
);
```

That's it — 6 lines for the effect. The `fromTo` form is required because we want to **define both** the initial state and the final state, not just one of them.

### `scrollTrigger` without a timeline

Notice we don't create a `gsap.timeline()` first — `scrollTrigger` can be passed directly to a `.to`/`.from`/`.fromTo` call:

```js
gsap.to(target, { ...props, scrollTrigger: {...} });
```

For a single tween, this is cleaner than wrapping in a timeline.

---

## 2. `Hero.tsx` — the showpiece, fully explained

### The 3 things happening here

1. **Entry animations** (on load) — title chars rise, paragraph lines rise.
2. **Hero scroll parallax** — leaves move opposite directions.
3. **Video scrub** — pinned video scrubs through frames as you scroll.

### Entry animations

```js
const heroSplit = new SplitText(".title", { type: "chars words" });
const paragraphSplit = new SplitText(".subtitle", { type: "lines" });

heroSplit.chars.forEach((char) => char.classList.add("text-gradient"));

gsap.from(heroSplit.chars, {
  yPercent: 100, duration: 1.8, ease: "expo.out", stagger: 0.05,
});

gsap.from(paragraphSplit.lines, {
  opacity: 0, yPercent: 100, duration: 1.8, ease: "expo.out", stagger: 0.05, delay: 1,
});
```

Two SplitText calls — chars for the title (animated rising), lines for the subtitle (animated rising). The `forEach` adds gradient styling per char.

`stagger: 0.05` = 50ms between each char/line. With `ease: "expo.out"`, characters decelerate dramatically — feels punchy.

### Video scrub — the big trick

```js
const tl = gsap.timeline({
  scrollTrigger: {
    trigger: 'video',
    start: 'center 60%',
    end: 'bottom top',
    scrub: true,
    pin: true,
  }
});

videoRef.current.onloadedmetadata = () => {
  if (!videoRef.current) return;
  tl.to(videoRef.current, { currentTime: videoRef.current.duration });
};
```

### Why `onloadedmetadata`?

`videoRef.current.duration` is `NaN` until the video's metadata header is fetched. The browser fires `loadedmetadata` event once the duration, dimensions, etc. are known. We wait for that, then add the tween.

### Important: this is a tween of a non-CSS property

GSAP normally tweens CSS properties (sets `style.foo`). Here we tween `currentTime` — a JS property on the HTMLVideoElement. GSAP detects it's not a CSS property and just sets it directly as `element.currentTime = newValue`. This works for **any numeric JS property of any object**.

### Performance considerations

- **Encoded video must be seekable**. Some `.mp4`/`.webm` files are encoded without keyframes for fast seeking — scrub will be janky. Solution: re-encode with `ffmpeg -i input.mp4 -c:v libx264 -profile:v high -level 4.0 -keyint_min 1 -g 1 output.mp4` to add a keyframe every frame.
- **Codec matters**. h.264 is fastest for scrubbing in browsers. AV1 and h.265 are slower to decode.
- **Mobile**: scrubbing is expensive on mobile. The author uses different start/end (`'top 50%'` vs `'center 60%'`) for mobile, but doesn't disable the scrub — could be smoother to disable entirely on low-end devices.

### Build the video scrub from scratch (interview-ready)

```jsx
function ScrollVideo({ src }: { src: string }) {
  const ref = useRef<HTMLVideoElement>(null);

  useGSAP(() => {
    const tl = gsap.timeline({
      scrollTrigger: { trigger: ref.current, start: 'top top', end: 'bottom top', scrub: true, pin: true },
    });
    if (!ref.current) return;
    ref.current.onloadedmetadata = () => {
      tl.to(ref.current, { currentTime: ref.current!.duration });
    };
  });

  return <video ref={ref} src={src} muted playsInline preload="auto" />;
}
```

20 lines. This is interview gold — practice typing it from scratch until it's reflex.

---

## 3. `Art.tsx` — CSS mask reveal

### What's a CSS mask?

A `mask-image` is a per-pixel alpha cutout for an element. Whites = visible, blacks = invisible. The element shows ONLY where the mask is white. Animating mask properties changes which parts are visible.

The `Art.tsx` code:

```js
.to('.masked-img', { scale: 1.3, maskPosition: 'center', maskSize: '400%' });
```

Animating `maskSize` from 100% to 400% — the mask grows 4x → more of the image becomes visible. Combined with `scale: 1.3` zooming the underlying image, the effect is a dramatic "blooming" reveal.

### CSS for the mask

```css
.masked-img {
  -webkit-mask-image: url('/images/some-shape.svg');
          mask-image: url('/images/some-shape.svg');
  -webkit-mask-size: 100%;
          mask-size: 100%;
  -webkit-mask-position: center;
          mask-position: center;
  -webkit-mask-repeat: no-repeat;
          mask-repeat: no-repeat;
}
```

Browser support: needs both `-webkit-mask-*` and standard `mask-*` for max compat (Safari only fully supports `-webkit-` prefix).

### Build mask reveal from scratch

```html
<style>
  .reveal-frame {
    width: 100vw; height: 100vh;
    background: url('/photo.jpg') center/cover;
    -webkit-mask: radial-gradient(circle, black 5%, transparent 5%);
            mask: radial-gradient(circle, black 5%, transparent 5%);
  }
</style>

<div class="reveal-frame"></div>

<script>
gsap.to('.reveal-frame', {
  webkitMaskSize: '500%', maskSize: '500%',
  scrollTrigger: { trigger: '.reveal-frame', start: 'top top', end: 'bottom top', scrub: true, pin: true },
});
</script>
```

A radial-gradient mask creates a circular spotlight. Growing it reveals the photo. ~15 lines for an awwwards-worthy effect.

---

## 4. `Menu.tsx` — state-driven slider

```tsx
const [currentIndex, setCurrentIndex] = useState<number>(0);

useGSAP(() => {
  gsap.fromTo('#title', { opacity: 0 }, { opacity: 1, duration: 1 });
  gsap.fromTo('.cocktail img', { opacity: 0, xPercent: -100 },
    { xPercent: 0, opacity: 1, duration: 1, ease: 'power1.inOut' });
  // ... details h2, details p ...
}, [currentIndex]);

const goToSlide = (index: number) => {
  const newIndex = (index + totalCocktails) % totalCocktails;
  setCurrentIndex(newIndex);
};

const getCocktailAt = (offset: number) =>
  allCocktails[(currentIndex + offset + totalCocktails) % totalCocktails];
```

### Why `useGSAP(fn, [currentIndex])`?

- On first render: animations run (entry).
- On `setCurrentIndex(...)`: `currentIndex` changes → `useGSAP` cleans up previous animations and re-runs the effect with new data → new entry animations play.

Without the deps array, animations would only run once.

### The modulo wrap math

```js
(currentIndex + offset + totalCocktails) % totalCocktails
```

`+ totalCocktails` is the safety: when `currentIndex - 1 = -1`, the result is `(-1 + 4) % 4 = 3` (wraps correctly). Without `+ totalCocktails`, `(-1) % 4 = -1` in JavaScript (modulo of negative numbers is negative in JS) → invalid index.

This is a **classic gotcha** worth knowing for any wrap-around logic.

---

## 5. `Cocktails.tsx` — entering elements

```js
parallaxTimeline
  .from('#c-left-leaf', { x: -100, y: 100 })
  .from('#c-right-leaf', { x: 100, y: 100 });
```

Both leaves enter from opposite bottom corners, sliding up and toward center. With `scrub: true`, the user "pulls them in" as they scroll.

The default position parameter (no `0` or `"<"`) means each `.from` runs **after** the previous one ends. So leaf 1 first, then leaf 2. If you wanted them simultaneous, add `0` to both: `.from('#c-left-leaf', {...}, 0)` etc.

---

## 6. `About.tsx` — multi-selector + heading-grid combo

```js
scrollTimeline
  .from(titleSplit.words, { ... })
  .from('.top-grid div, .bottom-grid div', { ... }, '-=0.5');
```

The second `.from` targets BOTH grids' children via a comma-separated selector. GSAP queries them all and combines into one stagger group. With `'-=0.5'`, this animation starts 0.5s before the previous one ends — slight overlap.

---

## 7. `Contact.tsx` — common-pattern recap

The standard awwwards section finish:
1. Split heading into words, animate them in.
2. Animate other text elements (h3/p) in.
3. Parallax leaves moving on scroll.

```js
timeline
  .from(titleSplit.words, { opacity: 0, yPercent: 100, stagger: 0.02 })
  .from('#contact h3, #contact p', { opacity: 0, yPercent: 100, stagger: 0.02 })
  .to('#f-right-leaf', { y: '-50' })
  .to('#f-left-leaf', { y: '-50' }, '<');
```

`'<'` aligns the two leaf tweens — they happen simultaneously instead of sequentially.

---

## What this folder teaches you

- **Scroll-scrubbed video** — the iPhone product page effect, in ~20 lines.
- **CSS mask reveal** with `maskSize` animation.
- **State-driven sliders** with `useGSAP(deps)` re-running animations on state change.
- **Modulo wrap math** for cyclic indices.
- **Multi-selector GSAP targets** with comma-separated selectors.
- **`.from()` for "element starts hidden/offset, animates in"** vs `.to()` for "element animates out".
- **`fromTo()` when you need both endpoints explicit**.

### Top 3 things to commit to muscle memory

1. **`gsap.to(video, { currentTime: video.duration })` inside `scrollTrigger`** — this is the most impressive single GSAP trick.
2. **`SplitText.create(el, { type: 'chars' })` + `gsap.from(split.chars, { yPercent: 100, stagger: 0.02 })`** — the universal text reveal.
3. **`(index + length) % length`** — modulo wrap for any cyclic index.
