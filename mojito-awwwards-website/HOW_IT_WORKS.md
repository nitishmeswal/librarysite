# mojito-awwwards-website/ — How It Works

An awwwards-style cocktail bar site featuring the **scroll-scrubbed video** trick — as you scroll, a video plays frame-by-frame in sync. Plus animated leaves parallax, masked image reveal, custom cocktail slider.

## Stack
- **React 19 + Vite + TypeScript**.
- **GSAP 3** + ScrollTrigger + SplitText (the same trio as Spylt).
- **`@gsap/react`** (`useGSAP`).
- **`react-responsive`** for media-query hook.
- **Tailwind CSS v4** (`@import "tailwindcss"`).

## Folder map
```
mojito-awwwards-website/
├── public/
├── constants/index.ts       # All data (cocktails, nav, socials, hours)
├── src/
│   ├── components/          # Navbar, Hero, Cocktails, About, Art, Menu, Contact
│   ├── App.tsx              # Composes all components
│   ├── main.tsx             # ReactDOM root
│   └── index.css            # Tailwind v4 + custom CSS
├── index.html
└── vite.config.ts
```

---

## 1. Entry & registration — `App.tsx`

```tsx
import { gsap } from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { SplitText } from 'gsap/SplitText'
import Navbar from './components/Navbar';
import Hero from './components/Hero';
// ... other imports

gsap.registerPlugin(ScrollTrigger, SplitText);

const App = () => {
  return (
    <main>
      <Navbar />
      <Hero />
      <Cocktails />
      <About />
      <Art />
      <Menu />
      <Contact />
    </main>
  )
}
```

### `gsap.registerPlugin(ScrollTrigger, SplitText)`

Registering once at module load. ScrollTrigger and SplitText are paid plugins (SplitText is part of GSAP Premium / Club GreenSock — recently made free for everyone). They must be registered before any animation references them, or GSAP will silently ignore them.

---

## 2. The marquee feature — scroll-scrubbed video — `Hero.tsx`

```tsx
const Hero = () => {
  const videoRef = useRef<HTMLVideoElement>(null);
  const isMobile = useMediaQuery({ maxWidth: 767 })

  useGSAP(() => {
    const heroSplit = new SplitText(".title", { type: "chars words" });
    const paragraphSplit = new SplitText(".subtitle", { type: "lines" });

    heroSplit.chars.forEach((char) => char.classList.add("text-gradient"));

    gsap.from(heroSplit.chars, {
      yPercent: 100, duration: 1.8, ease: "expo.out", stagger: 0.05,
    });

    gsap.from(paragraphSplit.lines, {
      opacity: 0, yPercent: 100, duration: 1.8, ease: "expo.out", stagger: 0.05, delay: 1,
    });

    gsap.timeline({
      scrollTrigger: { trigger: "#hero", start: "top top", end: "bottom top", scrub: true },
    })
      .to(".right-leaf", { y: 200 }, 0)
      .to(".left-leaf", { y: -200 }, 0);

    const startValue = isMobile ? 'top 50%' : 'center 60%';
    const endValue = isMobile ? '120% top' : 'bottom top';

    const tl = gsap.timeline({
      scrollTrigger: {
        trigger: 'video',
        start: startValue,
        end: endValue,
        scrub: true,
        pin: true,
      }
    });

    if (!videoRef.current) return;

    videoRef.current.onloadedmetadata = () => {
      if (!videoRef.current) return;
      tl.to(videoRef.current, { currentTime: videoRef.current.duration });
    };
  }, []);

  return (
    <>
      <section id="hero" className="noisy">
        <h1 className="title">MOJITO</h1>
        <img src="/images/hero-left-leaf.png" className="left-leaf" />
        <img src="/images/hero-right-leaf.png" className="right-leaf" />
        <div className="body">
          {/* subtitles, view-cocktails link */}
        </div>
      </section>

      <div className="video absolute inset-0">
        <video ref={videoRef} src="/videos/output.mp4" muted playsInline preload="auto" />
      </div>
    </>
  );
};
```

### **The scroll-scrubbed video trick** (most important part)

```js
tl.to(videoRef.current, { currentTime: videoRef.current.duration });
```

GSAP can animate **any numeric property** of any object, including DOM properties of media elements. Here:
- The tween target is the `<video>` element itself.
- The animated property is `currentTime` (HTMLVideoElement's playback position in seconds).
- The target value is `videoRef.current.duration` (total video length).

When this is wrapped in a `scrollTrigger` with `scrub: true`, the result is:

**Scroll position 0% → video at currentTime=0. Scroll position 100% → video at currentTime=duration.**

So as you scroll, the video **scrubs** frame by frame. This is exactly the Apple iPhone product page effect.

### Why it's complicated:

- You need the video duration. Videos load metadata asynchronously. So we wait for `onloadedmetadata` before creating the tween.
- The video MUST be `muted` and `playsInline` (autoplay restrictions don't matter here because we're not playing it — we're just scrubbing).
- `preload="auto"` tells the browser to fetch the whole video aggressively so seeks are smooth.
- The element MUST be in the DOM. `videoRef.current` check guards against null.

### The `pin: true` part

```js
scrollTrigger: { trigger: 'video', start: 'center 60%', end: 'bottom top', scrub: true, pin: true }
```

`pin: true` keeps the video sticky to the viewport while the timeline runs. So as you scroll the page, the video stays in place and its `currentTime` updates → looks like a fullscreen scrubbing experience.

### The leaves parallax

```js
gsap.timeline({
  scrollTrigger: { trigger: "#hero", start: "top top", end: "bottom top", scrub: true },
})
  .to(".right-leaf", { y: 200 }, 0)
  .to(".left-leaf", { y: -200 }, 0);
```

Right leaf moves down 200px, left leaf moves up 200px — opposing directions create parallax depth. The `, 0` at the end is the position parameter (`0` = start of timeline) — both happen simultaneously.

### `text-gradient` class added to each char

```js
heroSplit.chars.forEach((char) => char.classList.add("text-gradient"));
```

After splitting "MOJITO" into characters, we add a `.text-gradient` class to every character so each gets the gradient text styling (defined in CSS as a `bg-clip-text` gradient). You can't apply that styling to the parent `<h1>` and expect it to inherit perfectly per-character — applying it per-char gives nicer control.

---

## 3. `Navbar.tsx` — scroll-triggered nav background

```tsx
useGSAP(() => {
  const navTween = gsap.timeline({
    scrollTrigger: { trigger: 'nav', start: 'bottom top' }
  });

  navTween.fromTo('nav',
    { backgroundColor: 'transparent' },
    { backgroundColor: '#00000050', backgroundFilter: 'blur(10px)', duration: 1, ease: 'power2.inOut' }
  );
});
```

### The pattern: "transparent → solid as you scroll past"

- Trigger is the `<nav>` itself.
- `start: 'bottom top'` = animation starts when the **bottom** of `<nav>` reaches the **top** of the viewport (i.e., the moment the user scrolls past the nav).
- `fromTo` defines explicit start and end states.
- Result: as you scroll past the nav into the hero, the nav becomes semi-transparent black with backdrop blur.

### Bug to spot

`backgroundFilter: 'blur(10px)'` — should be `backdropFilter`. The wrong property name is silently dropped. The blur effect doesn't actually appear; only the color change does.

---

## 4. `Cocktails.tsx` — parallax leaves entering from corners

```tsx
useGSAP(() => {
  const parallaxTimeline = gsap.timeline({
    scrollTrigger: {
      trigger: '#cocktails',
      start: 'top 30%',
      end: 'bottom 80%',
      scrub: true,
    }
  });

  parallaxTimeline
    .from('#c-left-leaf', { x: -100, y: 100 })
    .from('#c-right-leaf', { x: 100, y: 100 });
});
```

### **`.from()` vs `.to()`**

- `.from(target, props)` — animate FROM `props` TO current values.
- `.to(target, props)` — animate FROM current values TO `props`.

So `from('#c-left-leaf', { x: -100, y: 100 })` means "the leaf starts 100px down-left and animates back to its natural position".

With `scrub: true`, the animation **reverses** as you scroll back. Makes the leaves "fly into" the page.

---

## 5. `About.tsx` — split heading + grid fade

```tsx
useGSAP(() => {
  const titleSplit = SplitText.create('#about h2', { type: 'words' });

  const scrollTimeline = gsap.timeline({
    scrollTrigger: { trigger: '#about', start: 'top center' }
  });

  scrollTimeline
    .from(titleSplit.words, { opacity: 0, duration: 1, yPercent: 100, ease: 'expo.out', stagger: 0.02 })
    .from('.top-grid div, .bottom-grid div', {
      opacity: 0, duration: 1, ease: 'power1.inOut', stagger: 0.04,
    }, '-=0.5');
});
```

The standard pattern again:
- Heading splits into words, words rise from below.
- Grid items fade in with stagger, overlapping with heading (`'-=0.5'`).

### Multi-selector in `gsap.from('.top-grid div, .bottom-grid div', ...)`

GSAP accepts comma-separated CSS selectors as the target — selects elements from both grids and treats them as one group for the stagger.

---

## 6. `Art.tsx` — mask reveal (the second showpiece)

```tsx
const Art = () => {
  const isMobile = useMediaQuery({ maxWidth: 767 });

  useGSAP(() => {
    const start = isMobile ? 'top 20%' : 'top top';

    const maskTimeline = gsap.timeline({
      scrollTrigger: {
        trigger: '#art',
        start,
        scrub: 1.5,
        pin: true,
      }
    });

    maskTimeline
      .to('.will-fade', {
        opacity: 0, stagger: 0.2, ease: 'power1.inOut',
      })
      .to('.masked-img', {
        scale: 1.3, maskPosition: 'center', maskSize: '400%',
        duration: 1, ease: 'power1.inOut',
      })
      .to('#masked-content', { opacity: 1, duration: 1, ease: 'power1.inOut' });
  });
  ...
};
```

### The mask trick

CSS `mask-image` clips an element by another image (think: cookie cutter). Animating `mask-size` and `mask-position` makes the visible region grow/shift, revealing more of the underlying image.

```css
.masked-img {
  mask-image: url('/images/some-mask.png');
  mask-size: 100%;
  mask-position: 50% 50%;
}
```

Animating `maskSize: '400%'` enlarges the mask 4x — what was previously a small clipping window now shows the entire image. Looks like a zoom-reveal.

### The 3-step timeline

1. Fade out everything with `.will-fade` (lists, "The ART" headline).
2. Zoom and reveal the masked image.
3. Fade in `#masked-content` (the final tagline).

All scrub-tied to scroll + pinned. Very smooth, very awwwards.

---

## 7. `Menu.tsx` — custom cocktail slider with arrow navigation

```tsx
const Menu = () => {
  const [currentIndex, setCurrentIndex] = useState<number>(0);

  useGSAP(() => {
    gsap.fromTo('#title', { opacity: 0 }, { opacity: 1, duration: 1 });
    gsap.fromTo('.cocktail img', { opacity: 0, xPercent: -100 },
      { xPercent: 0, opacity: 1, duration: 1, ease: 'power1.inOut' });
    gsap.fromTo('.details h2', { yPercent: 100, opacity: 0 },
      { yPercent: 0, opacity: 1, ease: 'power1.inOut' });
    gsap.fromTo('.details p', { yPercent: 100, opacity: 0 },
      { yPercent: 0, opacity: 1, ease: 'power1.inOut' });
  }, [currentIndex]);

  const totalCocktails = allCocktails.length;

  const goToSlide = (index: number): void => {
    const newIndex = (index + totalCocktails) % totalCocktails;
    setCurrentIndex(newIndex);
  };

  const getCocktailAt = (indexOffset: number): Cocktail => {
    return allCocktails[(currentIndex + indexOffset + totalCocktails) % totalCocktails];
  };

  const currentCocktail = getCocktailAt(0);
  const prevCocktail = getCocktailAt(-1);
  const nextCocktail = getCocktailAt(1);
  ...
};
```

### The slider math

`(currentIndex + totalCocktails) % totalCocktails` — handles negative indices. If `currentIndex - 1` is `-1`, then `(-1 + 4) % 4 = 3` → wraps to the last cocktail. Classic modulo trick.

### `useGSAP(() => { ... }, [currentIndex])`

The `[currentIndex]` is a **deps array** for `useGSAP`. Whenever `currentIndex` changes, the effect re-runs → re-creates the entry animations for the new cocktail. The previous animations are auto-reverted by `useGSAP` (this is its superpower over `useEffect`).

### Why this is cool

- No animation library overhead.
- Pure state-driven: `setCurrentIndex` is the only state mutation.
- Display logic via simple math: `getCocktailAt(-1)`, `getCocktailAt(0)`, `getCocktailAt(1)`.
- Animation tied to deps: every state change = fresh animation.

---

## 8. `Contact.tsx` — final scroll sequence

```tsx
useGSAP(() => {
  const titleSplit = SplitText.create('#contact h2', { type: 'word' });

  const timeline = gsap.timeline({
    scrollTrigger: { trigger: '#contact', start: 'top center' },
    ease: "power1.inOut"
  });

  timeline
    .from(titleSplit.words, { opacity: 0, yPercent: 100, stagger: 0.02 })
    .from('#contact h3, #contact p', { opacity: 0, yPercent: 100, stagger: 0.02 })
    .to('#f-right-leaf', { y: '-50', duration: 1, ease: 'power1.inOut' })
    .to('#f-left-leaf', { y: '-50', duration: 1, ease: 'power1.inOut' }, '<');
});
```

### Bug to spot

`SplitText.create('#contact h2', { type: 'word' })` — should be `'words'`. The plural form is what SplitText expects. The singular form might silently default to chars or fail.

Always double-check `type: 'words'` plural.

---

## 9. Tailwind v4 in this project

Look at `src/index.css`:

```css
@import "tailwindcss";

@theme {
  --color-bar-color: #...;
  --color-mid-brown: #...;
}
```

Tailwind v4 uses `@theme` blocks (CSS-first config) instead of `tailwind.config.js`. `--color-foo` automatically generates `bg-foo`, `text-foo`, `border-foo` utilities.

This is the major v4 change. v3 had `theme.extend.colors.foo` in JS config. v4 has them in CSS.

---

## What this project teaches you

- **Scroll-scrubbed video** (`gsap.to(videoEl, { currentTime: duration })`) — the iPhone-product-page effect.
- **`useGSAP(deps)`** for animations that re-run on state change.
- **Modulo wrap for sliders**: `(index + length) % length`.
- **CSS masks animated via `maskSize`** for zoom-reveals.
- **`.from()` reverse animations** — element starts at offset, animates back.
- **`gsap.fromTo()`** when you need explicit start and end values.
- **Multi-selector GSAP targets** (`'.a, .b'`) for cross-section animation groups.
- **Tailwind v4 `@theme` blocks** in CSS instead of JS config.
- **Common SplitText bug**: `type: 'word'` (singular) vs `'words'` (correct).

See subfolder:
- [src/components/HOW_IT_WORKS.md](./src/components/HOW_IT_WORKS.md)
- [constants/HOW_IT_WORKS.md](./constants/HOW_IT_WORKS.md)
