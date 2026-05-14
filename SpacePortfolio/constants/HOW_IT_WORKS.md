# SpacePortfolio / constants — How It Works

One file: `index.ts`. Holds **all static data** for the site — the skill lists rendered as orbiting icons, project metadata, social links.

## Shape

Each entry looks like:

```ts
export const Skill_data = [
  {
    skill_name: "Html 5",
    Image: "/html.png",
    width: 80,
    height: 80,
  },
  // ...
];

export const Frontend_skill = [ /* same shape */ ];
export const Backend_skill = [ /* same shape */ ];
export const Full_stack = [ /* same shape */ ];
export const Other_skill = [ /* same shape */ ];
```

And consumed in `components/main/Skills.tsx` like:

```tsx
{Frontend_skill.map((image, index) => (
  <SkillDataProvider
    key={index}
    src={image.Image}
    width={image.width}
    height={image.height}
    index={index}
  />
))}
```

## Why centralize data?

Same as Personal-Portfolio:
- **Single source of truth**: change a logo path here, every render updates.
- **Easy to grow**: append to the array.
- **Easy to migrate**: replace with a fetch later.
- **Type-friendly**: with TypeScript you can `as const` to lock down the literal types:

```ts
export const Skill_data = [
  { skill_name: "Html 5", Image: "/html.png", width: 80, height: 80 },
  // ...
] as const;
```

`as const` infers the literal types (string instead of `string`, number instead of `number`) — useful when you `keyof` derive types from this data.

## TypeScript inferring shape from data

In a real codebase you'd define the type once:

```ts
type Skill = { skill_name: string; Image: string; width: number; height: number };

export const Skill_data: Skill[] = [ /* ... */ ];
```

This catches "oops, I forgot `height`" at compile time. The repo's version skips the explicit type — TS infers it but you lose some autocomplete in editors.

## Pattern: data shape consistency

All five skill arrays share the **same shape**, so the same `<SkillDataProvider>` works for any of them. **This is good architecture.** If one array had `Image` and another had `image`, the consumer would need a per-list switch.

## What this file teaches you

- **Keep static data in a `constants/` file** separate from components.
- **Use consistent shapes** across related arrays so a single renderer can handle all.
- **TypeScript inference + optional explicit types** for safety.

That's the whole point of this folder.
