# Fizzi-3D-Website / src/components — How It Works

The reusable building blocks.

| File | What it is |
| --- | --- |
| `Header.tsx` | Top header with the brand logo (12 lines, trivial). |
| `Footer.tsx` | Page footer. |
| `FizziLogo.tsx` | SVG of the brand logo. |
| `Bounded.tsx` | Generic container with max-width and padding. |
| `Button.tsx` | Prismic-linked CTA button. |
| `TextSplitter.tsx` | Splits a string into per-word + per-char spans (no GSAP plugin needed!). |
| `CircleText.tsx` | SVG with text along a circle (rotating disc). |
| `ViewCanvas.tsx` | The shared 3D Canvas with `<View.Port />`. |
| `FloatingCan.tsx` | A SodaCan wrapped in `<Float>` with a ref. |
| `SodaCan.tsx` | Renders the GLTF can model with a flavor texture. |

---

## 1. `Header.tsx`

```tsx
export default function Header() {
  return (
    <header className="-mb-28 flex justify-center py-4">
      <FizziLogo className="z-10 h-20 cursor-pointer text-sky-800" />
    </header>
  );
}
```

### `-mb-28` (Tailwind)

`margin-bottom: -7rem` (negative). Pulls the next element up by 7rem, making the logo overlap the content below it. Trick to position the logo over the hero without absolute positioning.

### `text-sky-800` on an SVG?

The SVG uses `fill="currentColor"` inside (look at FizziLogo.tsx), so the parent's text color cascades into the SVG. Tailwind text colors work on SVGs this way.

---

## 2. `Bounded.tsx` — the container component

```tsx
type BoundedProps = {
  as?: React.ElementType;
  className?: string;
  children: React.ReactNode;
};

export const Bounded = ({ as: Comp = "section", className, children, ...restProps }: BoundedProps) => {
  return (
    <Comp className={clsx("px-4 first:pt-10 md:px-6", className)} {...restProps}>
      <div className="mx-auto flex w-full max-w-7xl flex-col items-center">
        {children}
      </div>
    </Comp>
  );
};
```

### The `as` prop pattern

```tsx
as: Comp = "section"
```

Allows callers to override the HTML tag:

```tsx
<Bounded as="div">...</Bounded>     // renders <div>
<Bounded as="article">...</Bounded>  // renders <article>
<Bounded>...</Bounded>                // defaults to <section>
```

Standard "polymorphic component" pattern. Useful for accessibility (semantic HTML).

### `first:pt-10`

Tailwind `first:` prefix = applies to first child of parent. So the first `<Bounded>` on the page gets `padding-top: 2.5rem`, subsequent ones don't (they're already separated from the previous section).

### `mx-auto flex w-full max-w-7xl flex-col items-center`

The standard "centered max-width container with vertical stack" pattern. Re-used in every section.

---

## 3. `Button.tsx`

```tsx
type Props = {
  buttonLink: LinkField;
  buttonText: string | null;
  className?: string;
};

export default function Button({ buttonLink, buttonText, className }: Props) {
  return (
    <PrismicNextLink
      className={clsx(
        "rounded-xl bg-orange-600 px-5 py-4 text-center text-xl font-bold uppercase tracking-wide text-white transition-colors duration-150 hover:bg-orange-700 md:text-2xl",
        className,
      )}
      field={buttonLink}
    >
      {buttonText}
    </PrismicNextLink>
  );
}
```

### `<PrismicNextLink>`

A Next.js-aware Link component that handles Prismic LinkField. Prismic LinkFields can be:
- Internal links (uses Next.js `<Link>` for client-side nav).
- External links (uses `<a target="_blank">`).
- File links.

`<PrismicNextLink>` figures out which type and renders accordingly. Saves you a giant `if/else`.

### `transition-colors duration-150 hover:bg-orange-700`

Tailwind transition utilities:
- `transition-colors` — transition only color-related properties (`color`, `background-color`, `border-color`, `fill`, `stroke`).
- `duration-150` — 150ms transition duration.
- `hover:bg-orange-700` — on hover, change background.

Avoid `transition-all` — it animates EVERYTHING including `position`, `transform`, etc. which can cause unwanted animations.

---

## 4. `TextSplitter.tsx` — DIY SplitText (no GSAP plugin)

```tsx
export function TextSplitter({ text, className, wordDisplayStyle = "inline-block" }: Props) {
  if (!text) return null;
  const words = text.split(" ");

  return words.map((word, wordIndex) => {
    const splitText = word.split("");
    return (
      <span
        className={clsx("split-word", className)}
        style={{ display: wordDisplayStyle, whiteSpace: "pre" }}
        key={`${wordIndex}-${word}`}
      >
        {splitText.map((char, charIndex) => {
          if (char === " ") return ` `;
          return (
            <span key={charIndex} className={`split-char inline-block split-char--${wordIndex}-${charIndex}`}>
              {char}
            </span>
          );
        })}
        {wordIndex < words.length - 1 ? (
          <span className="split-char">{` `}</span>
        ) : (
          ""
        )}
      </span>
    );
  });
}
```

### Why a custom splitter instead of `SplitText`?

GSAP's `SplitText` works at the DOM level — it MUTATES the rendered HTML after React renders. This can cause:
- Issues with React's reconciliation (React doesn't expect external DOM changes).
- Slower re-renders on dependency changes (SplitText has to re-split every time).
- Server-side rendering issues (SplitText needs `window`, doesn't work in SSR).

A pure-React approach (`TextSplitter`) produces the split spans **at render time**. They're part of the JSX tree, fully under React's control. SSR-friendly, hydration-friendly, no plugin needed.

### The structure

For "Hello World":
```html
<span class="split-word"><!-- word: Hello -->
  <span class="split-char">H</span>
  <span class="split-char">e</span>
  <span class="split-char">l</span>
  <span class="split-char">l</span>
  <span class="split-char">o</span>
</span>
<!-- space between words: -->
<span class="split-char"> </span>
<span class="split-word"><!-- word: World -->
  <span class="split-char">W</span>
  ...
</span>
```

### Why a separate `<span class="split-char">` for the space?

So the space character itself can be animated (or at least targeted by `.split-char` selectors uniformly). Without it, animations like `gsap.from(".split-char", { yPercent: 100 })` wouldn't see the spaces — words would crash into each other.

### `whiteSpace: "pre"`

Without this, the browser collapses multiple spaces into one. With `pre`, all whitespace is preserved.

### `display: wordDisplayStyle` prop

```tsx
wordDisplayStyle = "inline-block" | "block"
```

- `inline-block` (default) — words flow horizontally, can wrap.
- `block` — each word on its own line (useful for animation where you want stacked words).

### Build TextSplitter from scratch (interview-prep)

```tsx
function TextSplitter({ text }: { text: string }) {
  return (
    <>
      {text.split(" ").map((word, wi) => (
        <span key={wi} className="word" style={{ display: "inline-block" }}>
          {word.split("").map((char, ci) => (
            <span key={ci} className="char" style={{ display: "inline-block" }}>
              {char}
            </span>
          ))}
          {wi < text.split(" ").length - 1 && " "}
        </span>
      ))}
    </>
  );
}
```

15 lines for a fully-functional, SSR-safe text splitter. Now any GSAP target like `gsap.from(".char", { ... })` works.

---

## 5. `CircleText.tsx` — SVG with text-along-a-circle

```tsx
<svg viewBox="0 0 123 123" className={clsx("circle-text", className)}>
  <title id="circle-text">Love your gut. Love your life.</title>
  <path fill={backgroundColor} d="M122 61.5a61 61 0 11-122 0 61 61 0 01122 0z" />
  <path fill={textColor} className="animate-spin-slow origin-center" d="..." />
</svg>
```

Two paths:
1. **Background disc** — a circle filled with background color.
2. **Text path** — a complex SVG `d` attribute that draws each character glyph along a circle. (This was generated in design software; not handwritten.)

### `animate-spin-slow origin-center`

`origin-center` — set transform origin to center.
`animate-spin-slow` — Tailwind animation defined in `tailwind.config.js`:

```js
// tailwind.config.js
animation: {
  "spin-slow": "spin 20s linear infinite",
},
```

CSS-driven infinite rotation. No JS needed.

### Why an SVG instead of CSS `transform: rotate()`?

Both work. The SVG approach lets the text follow the circle (each glyph is a separately-drawn shape on the circle path). CSS `rotate` would rotate the whole text block as a unit — that's a different effect.

---

## 6. `ViewCanvas.tsx` (already covered in parent guide)

The shared 3D Canvas with `<View.Port />`. Most important architectural piece in the project.

### The `dynamic({ ssr: false })` Loader

```tsx
const Loader = dynamic(
  () => import("@react-three/drei").then((mod) => mod.Loader),
  { ssr: false },
);
```

drei's `<Loader>` is a loading-progress overlay that shows while 3D assets load. It uses `window` internally so it can't run on the server. `dynamic({ ssr: false })` is Next.js's way of saying "load this only on the client".

### `<Suspense fallback={null}>`

R3F uses React Suspense for async loading (textures, GLTF, fonts). The `<Suspense>` wraps the View.Port; while assets load, the fallback (`null` = nothing) is rendered. Once loaded, the actual scene renders.

### Why not show a loader as fallback?

Because the drei `<Loader>` IS the loader — it's positioned outside the Suspense boundary so it can show progress *during* the suspend. Suspense fallback is for inside the suspend tree.

---

## 7. `FloatingCan.tsx` (covered in parent)

Wraps a `<SodaCan>` in drei's `<Float>` with a forwarded ref. Simple, but the forwardRef pattern is essential.

---

## 8. `SodaCan.tsx` (covered in parent)

The actual GLTF-loaded 3D model. Three meshes: cylinder body, label, tab.

---

## What this folder teaches you

- **Polymorphic `as` prop** for flexible component tags.
- **Custom React `TextSplitter`** to avoid GSAP plugin and SSR issues.
- **SVG text on a circle** for the rotating-disc effect.
- **`dynamic({ ssr: false })`** for client-only Next.js components.
- **Suspense for 3D asset loading**.
- **forwardRef + group composition** for 3D animation handles.
- **`transition-colors` not `transition-all`** for specific transitions.
- **Tailwind `first:` prefix** for first-of-type targeting.
- **CSS-driven animation classes** in tailwind.config.js (`animate-spin-slow`).
