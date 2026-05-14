# bee/ — How It Works

A tiny **pure HTML + CSS** site demonstrating: hero with mirrored "shadow text", section layouts, fixed decorative imagery, and a placeholder for a Three.js scene. There is **no JavaScript** here other than wherever you'd plug in your Three.js code in `#container3D`.

Files:
- `index (1).html` — the page markup.
- `main.css` — all styling.

> Note: filename has a space + parenthesis — it likely came from a download. In production you'd rename to `index.html`.

---

## 1. The page structure

`index (1).html`:

```html
<header>
   <div class="content-fit">
      <div class="logo">DDX</div>
      <nav>
         <ul>
            <li>Contacts</li>
            <li>Category</li>
            <li>Login</li>
         </ul>
      </nav>
   </div>
</header>
<div class="section" id="banner">
   <div class="content-fit">
      <div class="title" data-before="3D ANIMATION">3D ANIMATION</div>
   </div>
   <img src=".../flower.png" class="decorate" alt=""
      style="width: 50vw; bottom: 0; right: 0;">
   <img src=".../leaf.png" class="decorate" alt=""
      style="width: 30vw; bottom: 0; left: 0;">
</div>
<div class="section" id="intro">
   <div class="content-fit">
      <div class="number">01</div>
      <div class="des">
         <div class="title">3d animation design for website</div>
         <p>Lorem ipsum dolor sit, amet …</p>
      </div>
   </div>
</div>
…
<div id="container3D"></div>
```

**Line-by-line**:
- `<header>` — fixed top bar. Uses a blurred backdrop in CSS.
- `<div class="content-fit">` — common width-clamp wrapper (max 1200px or 90vw).
- `<div class="section" id="banner">` — each scrollable section is a `.section` div with a unique id.
- `data-before="3D ANIMATION"` — a custom data attribute. CSS reads it back with `attr(data-before)` to create the mirror reflection. **This is a very common trick.**
- `<img class="decorate" style="width: 50vw; bottom: 0; right: 0;">` — inline style positions the decoration. The `.decorate` class makes it `position: fixed`, behind everything (`z-index: -100`).
- `<div id="container3D"></div>` — empty container at the bottom. A Three.js scene's canvas goes here.

---

## 2. The mirror-reflection trick

This is the most interview-worthy bit. From `main.css`:

```css
#banner .title {
  color: #d1ff48;
  font-size: 11em;
  font-family: "devil breeze";
  font-weight: bold;
  position: relative;
  overflow: visible;
  text-align: center;
}
#banner .title::before {
  content: attr(data-before);
  position: absolute;
  z-index: -1;
  color: oklch(0.78 0.17 80.01 / 0.19);
  mask: linear-gradient(
    to bottom,
    #000 -80%,
    oklch(0 0 0/0),
    #000,
    oklch(0 0 0/0) 200%
  );
  transform: scaleY(-1) translateY(-0.44lh);
}
```

**Breakdown**:
- `position: relative;` on `.title` — makes it a positioning context for `::before`.
- `::before` — pseudo-element that lives "before" the element's content. We're using it as a fake reflection.
- `content: attr(data-before);` — pulls the text from the HTML attribute. So both the real text and the reflection say `"3D ANIMATION"` but we only had to write it once.
- `position: absolute; z-index: -1;` — sits behind the real text.
- `color: oklch(0.78 0.17 80.01 / 0.19);` — semi-transparent yellow. `oklch` is a modern color space; the `/ 0.19` is alpha.
- `mask: linear-gradient(...);` — a CSS mask: where the gradient is opaque, the pseudo-element is visible; where it's transparent, it's hidden. This makes the reflection fade in/out vertically.
- `transform: scaleY(-1)` — flips it upside down.
- `translateY(-0.44lh)` — `lh` is the line-height unit. Moves the flipped clone up by ~half a line so it touches the bottom of the real text.

**Why this is great**: zero JS, zero extra HTML, fully responsive (it scales with the font size).

---

## 3. Section + content-fit pattern

```css
.section {
  width: 100%;
  min-height: 100vh;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
}
.content-fit {
  width: min(1200px, 90vw);
  margin: auto;
  min-height: 100vh;
  position: relative;
  padding-block: 10em;
}
```

**Breakdown**:
- `min-height: 100vh` on `.section` — every section is at least one screen tall.
- `display: flex; justify-content: center; align-items: center` — centers content.
- `width: min(1200px, 90vw)` — narrower of "1200px" or "90% of viewport width". A super-clean responsive cap that needs no media query.
- `padding-block: 10em` — top + bottom padding, sized in `em` (relative to the local font-size).

You can reuse this pattern for ANY landing page. Memorize it.

---

## 4. Fixed decoration trick

```css
.section .decorate {
  position: fixed;
  z-index: -100;
  pointer-events: none;
}
```

Decorative images live behind everything (`z-index: -100`), don't intercept clicks (`pointer-events: none`), and stay put as you scroll because `position: fixed`. The actual placement is done inline (`bottom: 0; right: 0;`).

---

## 5. Fixed header with backdrop blur

```css
header {
  padding-block: 1em;
  position: fixed;
  top: 0;
  width: 100%;
  z-index: 10px;     /* note: bug — should be a unitless number */
  backdrop-filter: blur(20px);
  z-index: 100;
  background-color: #1b1b1b11;
  background-image: repeating-linear-gradient(
    to right,
    transparent 0 500px,
    #eee1 500px 501px
  );
}
```

**Notes**:
- `z-index: 10px;` is **invalid CSS** (z-index is unitless). It's immediately overwritten on the next line by `z-index: 100;`. Live demo of why redeclaring vs cleaning up matters — the second wins, but the first is dead noise.
- `backdrop-filter: blur(20px)` — frosted-glass effect over whatever scrolls underneath.
- `background-color: #1b1b1b11` — `#RRGGBBAA` hex with alpha. `11` = `0x11/0xff` ≈ 6.7%.
- `repeating-linear-gradient` — paints vertical 1px lines every 500px (a faint grid).

---

## 6. Three.js mount point

```css
#container3D {
  position: fixed;
  inset: 0;
  z-index: 100;
  pointer-events: none;
}
```

`inset: 0` is shorthand for `top: 0; right: 0; bottom: 0; left: 0;`. So the 3D container fills the viewport, sits above sections (`z-index: 100`), and doesn't intercept pointer events. You'd render a transparent Three.js canvas in here and it'd float over the page beautifully.

```css
@media screen and (max-width: 767px) {
  #container3D {
    position: sticky;
  }
}
```

On mobile it becomes `sticky` to behave better with mobile scroll quirks.

---

## 7. Responsive font sizes (no media query needed for hero)

```css
#banner .title { font-size: 11em; }

@media screen and (max-width: 1023px) {
  #banner .title { font-size: 5em; }
}
@media screen and (max-width: 767px) {
  #banner .title { font-size: 3em; }
}
```

Three breakpoints, classic. You could replace this with one `clamp(3em, 11vw, 11em)` and it'd be even cleaner.

---

## 8. Build this from scratch

To recreate this hero in an interview in ~10 minutes:

```html
<header>
  <nav>…</nav>
</header>

<section class="hero">
  <h1 class="title" data-text="3D ANIMATION">3D ANIMATION</h1>
</section>
```

```css
.hero { min-height: 100vh; display: grid; place-items: center; }
.title {
  font-size: clamp(3rem, 11vw, 11rem);
  color: #d1ff48;
  position: relative;
}
.title::before {
  content: attr(data-text);
  position: absolute;
  inset: 0;
  z-index: -1;
  color: rgba(209, 255, 72, 0.2);
  transform: scaleY(-1) translateY(-0.5em);
  mask: linear-gradient(to bottom, #000, transparent);
}
```

Done. Big hero with mirror reflection in 25 lines of CSS.

---

## 9. What this folder teaches you

- The `data-*` + `attr()` + `::before` pattern is huge.
- `position: fixed` with `inset: 0` and `pointer-events: none` is the standard "overlay" recipe.
- `width: min(1200px, 90vw)` is a one-line responsive container.
- Modern color functions like `oklch()` are valid CSS and useful for perceptually-uniform colors.

That's the entire project. Now go look at `paralax/` next — that one introduces JS with a beautiful mouse-driven 3D parallax.
