# paralax/ — How It Works

A **mouse-driven 3D parallax landscape** with a layered mountain/fog illustration. As your mouse moves, the layers shift in opposite directions and rotate slightly, creating a strong "depth" illusion. On page load, GSAP plays an intro that flies the layers in from below.

Files:
- `main.html` — the layered images + text.
- `main.css` — positioning every layer + a vignette.
- `main.js` — the mouse-tracking + GSAP intro.

This is one of the **most interview-friendly** projects in the repo because it combines vanilla JS, the DOM, CSS transforms, and GSAP — all in ~80 lines of JS.

---

## 1. The markup pattern

`main.html` is a single `<main>` with many `<img>` layers. Each layer has `data-*` attributes that the JS reads:

```html
<main>
  <div class="data">
    <div class="vignette hide"></div>
    <img src="…/background.png"
         data-speedx="0.3" data-speedy="0.38" data-speedz="0"
         data-rotation="0" data-distance="-200"
         alt="image" class="parallax bg-img" />

    <img src="…/mountain-9.png"
         data-speedx="0.125" data-speedy="0.155" data-speedz="0.15"
         data-rotation="0.02" data-distance="1700"
         alt="image" class="parallax mountain-9" />

    <div class="text parallax"
         data-speedx="0.07" data-speedy="0.07" data-speedz="0"
         data-rotation="0.11" data-distance="0">
      <h2>China</h2>
      <h1>Zhangjiakou</h1>
    </div>

    <!-- many more layers -->

    <img src="…/sun-rays.png" alt="image" class="sun-rays hide" />
    <img src="…/black-shadow.png" alt="image" class="black-shadow hide" />
  </div>
</main>
```

**Breakdown**:
- `class="parallax"` — marker class. The JS uses `document.querySelectorAll(".parallax")` to find all layers.
- `data-speedx="0.3"` — how strongly this layer responds to mouse X. Bigger value = bigger swing.
- `data-speedy="0.38"` — same for Y.
- `data-speedz="0.15"` — depth response (perspective Z).
- `data-rotation="0.02"` — how much to rotate when mouse moves horizontally.
- `data-distance="1700"` — used in the intro animation: how far below the screen this layer starts before flying in.
- `class="hide"` — three special layers (vignette, sun-rays, black-shadow) start hidden and fade in via the intro timeline.
- `<div class="text parallax">` — the title is a layer too, so it parallaxes with everything else.

**Why use `data-*`?** It's the standard HTML5 way to attach typed metadata to elements without polluting class names or globals. Read in JS as `el.dataset.speedx`.

---

## 2. The CSS — absolute positioning + transition

The CSS sets a fixed scene viewport and absolutely positions every layer at hand-tuned coordinates:

```css
main {
  position: relative;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
}
.parallax {
  pointer-events: none;
  transition: 0.45s cubic-bezier(cubic-bezier(0.2, 0.49, 0.32, 0.94));
}
.data .bg-img {
  position: absolute;
  width: 3200px;
  top: calc(50% - 390px);
  left: calc(50% + 50px);
  z-index: 1;
}
.data .mountain-9 {
  position: absolute;
  z-index: 5;
  width: 670px;
  top: calc(50% + 313px);
  left: calc(50% - 557px);
}
```

**Breakdown**:
- `position: relative` on `<main>` — makes it the positioning context for absolutely-positioned children.
- `overflow: hidden` — layers that overflow the viewport are clipped (important since they're 3200px wide).
- `pointer-events: none` on `.parallax` — clicks pass through. Only the `.text` (which doesn't have `pointer-events: none` re-added but lives above with `z-index: 9` and re-enables them via `pointer-events: auto`) is interactive.
- `transition: 0.45s cubic-bezier(...)` — when JS changes `transform`, the browser interpolates over 0.45s. **This is the smoothing** — without it, the layers would jerk.
- Note: the `cubic-bezier(cubic-bezier(...))` is a typo (extra `cubic-bezier()` wrapping). Browsers fall back gracefully but it's bug-noise.
- `calc(50% - 390px)` — centers the image (50%) then offsets by a hard-coded number of pixels. Each layer has its own tuned offset so the composition looks right at default viewport.
- `z-index: 1…21` — explicit stacking order from background (1) to foreground (21).

---

## 3. The vignette

```css
.vignette {
  position: absolute;
  z-index: 100;
  width: 100%;
  height: 100%;
  top: 0;
  left: 0;
  background: radial-gradient(
    ellipse at center,
    rgba(0, 0, 0, 0) 65%,
    rgba(0, 0, 0, 0.7)
  );
  pointer-events: none;
}
```

A radial gradient that's transparent in the middle and dark at the edges. Sits on top of everything (`z-index: 100`). This is **the** standard vignette recipe — memorize it.

---

## 4. The JS (line-by-line)

`main.js`:

```js
const parallax_el = document.querySelectorAll(".parallax");
const main = document.querySelector("#main");
let xValue = 0,
  yValue = 0;

let rotateDegree = 0;
```

- `document.querySelectorAll(".parallax")` — returns a static `NodeList` of every element with that class.
- `let xValue = 0, yValue = 0;` — module-scoped state for the current mouse offset from center.

```js
function update(cursorPosition) {
  parallax_el.forEach((el) => {
    let speedX = el.dataset.speedx;
    let speedY = el.dataset.speedy;
    let speedZ = el.dataset.speedz;
    let rotationSpeed = el.dataset.rotation;

    let isInLeft =
      parseFloat(getComputedStyle(el).left) < window.innerWidth / 2 ? 1 : -1;
    let zValue =
      (cursorPosition - parseFloat(getComputedStyle(el).left)) * isInLeft * 0.1;

    el.style.transform = ` perspective(2300px) translateZ(${
      zValue * speedZ
    }px) rotateY(${rotateDegree * rotationSpeed}deg) translateX(calc(-50% + ${
      -xValue * speedX
    }px)) translateY(calc(-50% + ${yValue * speedY}px))`;
  });
}
update(0);
```

**Line-by-line**:
- `parallax_el.forEach((el) => { … })` — iterate every layer.
- `el.dataset.speedx` — reads `data-speedx`. **Note: `dataset` strips hyphens and camelCases. `data-foo-bar` → `dataset.fooBar`. Here lowercase only, so no surprise.**
- `getComputedStyle(el).left` — the final resolved `left` value in px (e.g. `"50%"` becomes `"600px"` after layout).
- `parseFloat(...)` — converts the px string to a number.
- `isInLeft = … < window.innerWidth / 2 ? 1 : -1` — is this layer in the left half of the screen? Sign is used to invert Z depending on side, so elements appear to push back/forward depending on cursor location.
- `zValue` — depth in px based on how far the cursor is from the element.
- `el.style.transform = '...'` — the magic transform:
  - `perspective(2300px)` — adds vanishing-point perspective so `translateZ` and `rotateY` actually do 3D things. Without this, they're flat.
  - `translateZ(...)` — depth (positive = closer to viewer).
  - `rotateY(...)` — yaw rotation (around vertical axis).
  - `translateX(calc(-50% + Npx))` — preserves the CSS centering trick (`-50%` of own width) while adding the parallax offset.
  - `translateY(calc(-50% + Npx))` — same for Y.
- `update(0)` — initial call so transforms are applied even before the user moves the mouse.

```js
window.addEventListener("mousemove", (e) => {
  if (timeline.isActive()) return;

  xValue = e.clientX - window.innerWidth / 2;
  yValue = e.clientY - window.innerHeight / 2;
  rotateDegree = (xValue / (window.innerWidth / 2)) * 20;
  update(e.clientX);
});
```

- `if (timeline.isActive()) return;` — **important**: don't run parallax while the GSAP intro is playing. Otherwise mouse movement would fight the timeline.
- `e.clientX - window.innerWidth / 2` — offset from screen center. Negative on the left half, positive on the right.
- `rotateDegree = (xValue / (window.innerWidth / 2)) * 20` — normalize x to `-1..1`, then scale to `-20..20` degrees max.
- `update(e.clientX);` — re-render all layers.

---

## 5. The GSAP intro timeline

```js
let timeline = gsap.timeline();

Array.from(parallax_el)
  .filter((el) => !el.classList.contains("text"))
  .forEach((el) => {
    timeline.from(
      el,
      {
        top: `${el.offsetHeight / 2 + +el.dataset.distance}px`,
        duration: 3.5,
        ease: "power3.out"
      },
      "1"
    );
  });

timeline
  .from(".text h1", {
    y: window.innerHeight -
       document.querySelector(".text h1").getBoundingClientRect().top + 200,
    duration: 2
  }, "2.5")
  .from(".text h2", { y: -150, opacity: 0, duration: 1.5 }, "3")
  .from(".hide", { opacity: 0, duration: 1.5 }, "3");
```

**Breakdown**:
- `gsap.timeline()` — create the master timeline.
- `Array.from(parallax_el)` — convert NodeList → real array so we can `.filter()`.
- `.filter((el) => !el.classList.contains("text"))` — exclude the title layer (handled separately below).
- `timeline.from(el, { top: ..., duration: 3.5, ease: "power3.out" }, "1")`:
  - **`from`** means animate FROM `vars` → current. So we move `top` from a faraway value (offscreen below) back to where CSS positioned it.
  - `el.offsetHeight / 2 + +el.dataset.distance` — half the image height plus the per-layer "distance" data attribute, giving each layer its own start offset.
  - **`+el.dataset.distance`** — the leading `+` coerces the string to a number (`"1700"` → `1700`).
  - `"1"` — absolute position on the timeline: **all layers start at second 1**. They animate in parallel, not sequentially.
- The text comes in later (`"2.5"`, `"3"`), and the vignette/sun-rays/black-shadow elements (`.hide`) fade in at `"3"`.

**Why `from` instead of `to`?** Because the final state (real CSS top) is what we want visible at the end. With `from`, you describe the *start* and GSAP figures out the *end* is "whatever it would be now".

---

## 6. The interaction guard

```js
if (timeline.isActive()) return;
```

This is the simplest possible state machine: mouse moves don't matter while the intro plays. It also avoids the messy initial state where everything is way offscreen.

---

## 7. Build this from scratch

A 30-line "minimum viable parallax" you can write on a whiteboard:

```html
<div class="scene">
  <img src="bg.png"     class="layer" data-speed="0.1">
  <img src="midground.png" class="layer" data-speed="0.3">
  <img src="foreground.png" class="layer" data-speed="0.6">
</div>
```

```css
.scene { position: relative; height: 100vh; overflow: hidden; perspective: 2000px; }
.layer { position: absolute; inset: 0; transition: transform 0.4s ease-out; }
```

```js
const layers = document.querySelectorAll(".layer");
window.addEventListener("mousemove", (e) => {
  const x = e.clientX - innerWidth / 2;
  const y = e.clientY - innerHeight / 2;
  layers.forEach((l) => {
    const s = +l.dataset.speed;
    l.style.transform = `translate(${-x * s}px, ${-y * s}px)`;
  });
});
```

You'd add GSAP intro and perspective rotateY as flair, but the core idea is exactly this.

---

## 8. What this folder teaches you

- `data-*` + `dataset` for per-element config.
- `getComputedStyle` + `parseFloat` for reading layout values.
- `perspective() + translateZ() + rotateY()` for real 3D feel from CSS transforms.
- `transition` for free smoothing between transform updates (cheaper than rAF).
- GSAP `from()` + position parameters for an intro timeline.
- A simple guard (`timeline.isActive()`) to keep input handlers from fighting animations.

---

## 9. Bugs you can mention in an interview

- `transition: 0.45s cubic-bezier(cubic-bezier(...))` — wrapped function call is invalid; the browser likely ignores the timing function and falls back to default.
- `z-index: 10px` (in `bee/`) — invalid unit on z-index.
- Several layers (`.mountain-3`, `.fog-2`) share the same `z-index: 16` — fine, but unintentional.

Spotting these in code reviews is a great interview skill.
