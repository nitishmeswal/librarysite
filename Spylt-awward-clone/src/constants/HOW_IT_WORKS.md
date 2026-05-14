# Spylt-awward-clone / src/constants — How It Works

One file: `index.js`. Holds three arrays:

```js
const base = import.meta.env.BASE_URL;

const flavorlists = [
  { name: "Chocolate Milk", color: "brown", rotation: "md:rotate-[-8deg] rotate-0" },
  // 6 entries total
];

const nutrientLists = [
  { label: "Potassium", amount: "245mg" },
  // 5 entries total
];

const cards = [
  { src: `${base}videos/f1.mp4`, rotation: "rotate-z-[-10deg]", name: "Madison", img: `${base}images/p1.png`, translation: "translate-y-[-5%]" },
  // 7 entries total
];

export { flavorlists, nutrientLists, cards };
```

## Things worth pulling out

### 1. **Path-portable asset references**

```js
const base = import.meta.env.BASE_URL;
// ...
src: `${base}videos/f1.mp4`,
```

Same `BASE_URL` trick as in components — but here it's even smarter because the data **must** include full URLs (it can't dynamically resolve them at render time without an extra step).

### 2. **Per-card Tailwind classes in data**

```js
{ rotation: "rotate-z-[-10deg]", translation: "translate-y-[-5%]" }
```

Each card has its own slight rotation and translation baked into the data. The renderer:

```jsx
<div className={`vd-card ${card.translation} ${card.rotation}`}>
```

This puts **layout decisions in data** — useful when you want lots of variation without writing 7 different `<div>`s. Trade-off: data is no longer purely about content; it carries presentation hints. Acceptable for tight art-directed sites; avoid for plain content lists.

### 3. **Arbitrary value Tailwind**

`rotate-z-[-10deg]` — Tailwind's `rotate-z-*` doesn't have a built-in `-10deg` class, so we use the `[...]` arbitrary-value syntax. Tailwind generates a one-off class at build time.

### 4. **Inconsistent shape (real-world data smell)**

Some cards have `translation: "translate-y-[-5%]"`, others don't have a `translation` key at all:

```js
{ src: ..., rotation: "rotate-z-[4deg]", name: "Alexander", img: ... },         // no translation
{ src: ..., rotation: "rotate-z-[-4deg]", name: "Andrew", img: ..., translation: "translate-y-[-5%]" },  // has translation
```

The renderer `${card.translation}` returns `undefined` for cards without it → produces "undefined" string in className → does nothing visually (just a meaningless class). It works by accident. Cleaner:

```js
<div className={`vd-card ${card.translation || ""} ${card.rotation}`}>
```

This is the kind of robustness fix that wins interview points: "I noticed `card.translation` is sometimes undefined, leading to `undefined` rendered in the className. I'd add `|| ""` defaults to keep className clean."

---

## What this file teaches you

- **`import.meta.env.BASE_URL`** for portable static asset paths (Vite).
- **Embedding presentation hints in data** when content varies wildly — acceptable for design-heavy sites.
- **Tailwind's `[...]` arbitrary-value syntax** for one-off values.
- **Defending against missing data keys** with `|| ""` defaults.
- Centralizing data so sections become pure renderers.
