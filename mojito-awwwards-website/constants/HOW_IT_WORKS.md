# mojito-awwwards-website / constants — How It Works

One file: `index.ts`. Exports several arrays/objects consumed by sections.

## Exports

```ts
export {
  navLinks,        // [{ id, title }] - main nav links
  cocktailLists,   // [{ name, country, detail, price }] - most popular cocktails
  mockTailLists,   // [{ name, country, detail, price }] - mocktails
  profileLists,    // [{ imgPath }] - profile thumbnails
  featureLists,    // string[] - bullet list of features
  goodLists,       // string[] - bullet list of "good" things
  openingHours,    // [{ day, time }]
  storeInfo,       // { heading, address, contact: { phone, email } }
  socials,         // [{ name, icon, url }]
  allCocktails,    // [{ id, name, image, title, description }] - menu slider
};
```

## Patterns to internalize

### 1. Multiple related arrays, same shape

`cocktailLists` and `mockTailLists` have identical shape:

```ts
{ name: string, country: string, detail: string, price: string }
```

Why two arrays instead of one with a `type` discriminator? Easier rendering — each list has its own `<h2>` and `<ul>`. If you wanted ONE source of truth:

```ts
const drinks = [
  { name: "Chapel Hill", type: "cocktail", country: "AU", detail: "Battle", price: "$10" },
  { name: "Tropical Bloom", type: "mocktail", country: "US", detail: "Battle", price: "$10" },
];

// Filter at render:
const cocktails = drinks.filter(d => d.type === "cocktail");
const mocktails = drinks.filter(d => d.type === "mocktail");
```

Slightly more code at render, but cleaner data model. Trade-off.

### 2. Plain string arrays for simple lists

```ts
const featureLists = [
  "Perfectly balanced blends",
  "Garnished to perfection",
  "Ice-cold every time",
  "Expertly shaken & stirred",
];
```

No need for objects when you just have a flat list of strings. Render with:

```jsx
{featureLists.map((feature, i) => <li key={i}>{feature}</li>)}
```

⚠️ Using `index` as the key is fine ONLY if the list is **static and not reordered**. If the list could change, use a stable id.

### 3. Nested object for grouped fields

```ts
const storeInfo = {
  heading: "Where to Find Us",
  address: "...",
  contact: { phone: "...", email: "..." },
};
```

Grouping `phone` and `email` under `contact` makes the data structure self-documenting. Access: `storeInfo.contact.phone`.

### 4. `allCocktails` shape for the slider

```ts
const allCocktails = [
  { id: 1, name: "Classic Mojito", image: "...", title: "...", description: "..." },
  // ...
];
```

`id` field is included even though it's not strictly necessary for a static array. Best practice: every record has a stable id for `key` props and future-proofing (if you swap to a DB, ids stay stable).

### 5. Asset paths as strings, NOT imported

```ts
{ icon: "/images/insta.png" }
```

Vite serves `public/` directly — paths starting with `/` reference `public/`. Alternative is to `import` images:

```ts
import instaIcon from "../assets/insta.png";
// then use: { icon: instaIcon }
```

Importing lets Vite hash the filename for cache busting and code-split. Using paths from `public/` is simpler but doesn't get hashing.

### 6. TypeScript without explicit types

```ts
const navLinks = [
  { id: "cocktails", title: "Cocktails" },
  // ...
];
```

TypeScript infers the type: `{ id: string; title: string }[]`. If you want stricter, use `as const`:

```ts
const navLinks = [...] as const;
```

Then `navLinks[0].id` has type `"cocktails"` (literal), not just `string`. Useful for typed `keyof` derivations.

---

## What this file teaches you

- **Centralize static data** in `constants/` folder.
- **Use plain string arrays** for simple lists.
- **Use objects with `id` fields** for anything that might grow or be replaced by a DB.
- **Group related fields with nested objects** for self-documentation.
- **Use `public/` for static assets**, paths relative to `/`.
- **TypeScript can infer types** without explicit type annotations.
