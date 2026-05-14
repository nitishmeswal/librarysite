# site-gloria/ — How It Works

A **full multi-section e-commerce / fashion landing page**, hand-written in raw HTML/CSS/JS. This is the "real-world template" — 730 lines of HTML, 4272 lines of CSS, and ~9700 lines of JS. It uses the **BEM naming convention** and demonstrates a lot of patterns you'd see in a production marketing site.

Files:
- `index.html` — markup (730 lines).
- `css/style.css` — full hand-authored stylesheet.
- `css/style.min.css` — minified version (currently empty).
- `js/app.js` — JS library (custom UI kit).
- `js/app.min.js` — minified version.
- `fonts/`, `img/` — assets.

Because this is a large template, this guide gives you the **patterns you can extract** rather than walking every line.

---

## 1. BEM naming convention

Look at the markup:

```html
<header class="header">
  <div class="header__container">
    <a href="" class="header__logo">…</a>
    <div class="header__menu menu">
      <nav class="menu__body">
        <ul class="menu__list">
          <li class="menu__item">
            <a href="#" class="menu__link">WOMEN</a>
          </li>
```

BEM = **Block, Element, Modifier**:
- **Block**: `.header`, `.menu`, `.button` — standalone reusable component.
- **Element**: `.header__logo`, `.menu__list` — child of a block. Double underscore.
- **Modifier**: `.button--primary`, `.menu__link--active` — variant. Double dash.

Why? It makes CSS predictable: any selector tells you exactly where it lives. No collisions between blocks. No deep nesting required.

```css
.header { … }
.header__container { … }
.header__logo { … }
.menu { … }
.menu__list { display: flex; gap: 2em; }
.menu__link { color: #111; }
.menu__link--active { color: #f00; }
```

In an interview, knowing BEM signals you've worked on real-world CSS at scale.

---

## 2. Layout: `[class*="_container"]` wildcard

The CSS uses an attribute selector for the centered container pattern:

```css
[class*="_container"] {
  max-width: var(--container-width);
  margin: 0 auto;
  padding-inline: var(--container-padding);
}
```

- `[class*="_container"]` — matches any class that **contains** `_container` (e.g. `.header__container`, `.page__container`).
- `padding-inline` — logical property. Same as `padding-left + padding-right` but respects writing direction (RTL languages get it on the right).
- `--container-width`, `--container-padding` — CSS custom properties (variables) defined elsewhere on `:root`.

You get one rule applying to every container in the site. Combine with BEM and life is good.

---

## 3. The webp detection trick

`js/app.js` starts with:

```js
function isWebp() {
  function testWebP(callback) {
    let webP = new Image;
    webP.onload = webP.onerror = function() {
      callback(webP.height == 2);
    };
    webP.src = "data:image/webp;base64,UklGRjoAAABXRUJQVlA4IC4AAACyAgCdASoCAAIALmk0mk0iIiIiIgBoSygABc6WWgAA/veff/0PP8bA//LwYAAA";
  }
  testWebP((support) => {
    let className = support === true ? "webp" : "no-webp";
    document.documentElement.classList.add(className);
  });
}
```

**Breakdown**:
- Creates a tiny in-memory `Image` and tries to load a 2px webp encoded as a data-URI.
- If `webP.height === 2` after `onload`, the browser supports webp.
- Sets a class `webp` or `no-webp` on `<html>`.
- CSS can then write `.webp .bg { background-image: url(x.webp); } .no-webp .bg { background-image: url(x.jpg); }`.

**Why**: even though all modern browsers now support webp, this pattern (feature detection + class on `<html>`) is **the standard** for serving the optimal format with no JS in the critical render path.

---

## 4. Slide up/down primitive (the jQuery-style helper)

```js
let _slideUp = (target, duration = 500, showmore = 0) => {
  if (!target.classList.contains("_slide")) {
    target.classList.add("_slide");
    target.style.transitionProperty = "height, margin, padding";
    target.style.transitionDuration = duration + "ms";
    target.style.height = `${target.offsetHeight}px`;
    target.offsetHeight;                           // force reflow
    target.style.overflow = "hidden";
    target.style.height = showmore ? `${showmore}px` : `0px`;
    target.style.paddingTop = 0;
    target.style.paddingBottom = 0;
    target.style.marginTop = 0;
    target.style.marginBottom = 0;
    window.setTimeout(() => {
      target.hidden = !showmore;
      target.style.removeProperty("transition-duration");
      target.style.removeProperty("transition-property");
      // … and so on
      target.classList.remove("_slide");
    }, duration);
  }
};
```

**Pattern to memorize**:
1. Set `transition` on the properties you want to animate (height/margin/padding).
2. Set `height` to the **current** `offsetHeight` (else CSS can't transition from `auto`).
3. **`target.offsetHeight;`** — reading any layout property forces the browser to flush styles (a "reflow"). This makes the next style change actually animate instead of being batched.
4. Set the target height to `0`. Browser transitions over `duration` ms.
5. After the timeout, clean up inline styles and set `hidden`.

**Why this is interview gold**: It demonstrates you understand the **forced-reflow trick** — explicitly reading a layout property to flush pending style writes. Phrasing it like "I read `offsetHeight` to force a synchronous style/layout pass so the transition starts from the current value, not the new one" sounds amazing.

---

## 5. CSS pattern: `_icon-` placeholder via `[class*="_icon-"]:before`

```css
[class*=_icon-]:before {
  font-family: "icons";
  font-style: normal;
  font-weight: normal;
}
```

The site uses an icon font (loaded via `@font-face`) and a global rule that applies font settings to **any** class beginning with `_icon-`. Individual classes set the actual glyph via `content`:

```css
._icon-search:before { content: "\e900"; }
._icon-cart:before   { content: "\e901"; }
```

Then in HTML: `<i class="_icon-search"></i>`. Lightweight, accessibility-friendly (you can add `aria-label` to the element).

---

## 6. Loader pattern

```html
<main class="page">
  <div class="loader">
    <!-- placeholder for animated loader -->
  </div>
  …
</main>
```

A `.loader` overlay covers the page initially. CSS gives it `position: fixed; inset: 0; z-index: 9999;` plus opacity transition. The JS removes it (or adds a `loaded` class to body) on `window.load`. Standard pattern.

---

## 7. Read these files in this order

For a quick walk-through with maximum learning per minute:

1. **`index.html`** — first 200 lines. See the BEM structure, semantic tags, and how a marketing page is laid out.
2. **`css/style.css`** — first 200 lines and lines around any selector you spot in the HTML. Note the variables `--container-width`, breakpoints, and `@font-face` rules.
3. **`js/app.js`** — first ~200 lines. The site uses an IIFE (`(() => { "use strict"; … })()` wrapper) to encapsulate the entire UI kit.

You don't need to read every line. The point is to see what a "real" hand-built site feels like, contrasted with React/Next/Vite projects elsewhere in this repo.

---

## 8. What this folder teaches you

- **BEM naming**: predictable CSS class structure that scales.
- **Attribute selectors**: `[class*="_container"]`, `[class*="_icon-"]` for cross-cutting rules.
- **Feature detection**: probe support (e.g. webp), then attach a class to `<html>` so CSS can branch.
- **The forced-reflow trick**: reading `offsetHeight` to flush pending styles before a transition.
- **`padding-inline` / `margin-inline`**: logical properties that respect text direction.
- **Icon fonts**: how to bundle UI icons as a font.
- **Loader overlay**: simple `position: fixed; inset: 0;` pattern.

---

## 9. Build this from scratch

A 5-minute "marketing landing" template:

```html
<header class="header">
  <div class="header__container">
    <a class="header__logo">LOGO</a>
    <nav class="menu">
      <ul class="menu__list">
        <li class="menu__item"><a class="menu__link">Home</a></li>
        <li class="menu__item"><a class="menu__link">Shop</a></li>
      </ul>
    </nav>
  </div>
</header>

<main>
  <section class="hero">
    <div class="hero__container">
      <h1 class="hero__title">Big Heading</h1>
      <p class="hero__lede">…</p>
      <a class="button button--primary">Shop now</a>
    </div>
  </section>
</main>
```

```css
:root { --container-width: 1200px; --container-padding: 1rem; }
[class*="__container"], [class*="_container"] {
  max-width: var(--container-width);
  margin-inline: auto;
  padding-inline: var(--container-padding);
}
.menu__list { display: flex; gap: 2rem; list-style: none; padding: 0; }
.button { display: inline-block; padding: 0.75rem 1.5rem; border-radius: 0.5rem; }
.button--primary { background: black; color: white; }
```

That's basically the skeleton this whole template scales up from.
