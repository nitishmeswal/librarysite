# SpacePortfolio / utils — How It Works

One file: `motion.ts`. Pure Framer Motion variant helpers used across the site.

## File: `motion.ts`

```ts
export function slideInFromLeft(delay: number) {
  return {
    hidden: { x: -100, opacity: 0 },
    visible: {
      x: 0,
      opacity: 1,
      transition: { delay: delay, duration: 0.5 },
    },
  };
}

export function slideInFromRight(delay: number) {
  return {
    hidden: { x: 100, opacity: 0 },
    visible: {
      x: 0,
      opacity: 1,
      transition: { delay: delay, duration: 0.5 },
    },
  };
}

export const slideInFromTop = {
  hidden: { y: -100, opacity: 0 },
  visible: {
    y: 0,
    opacity: 1,
    transition: { delay: 0.5, duration: 0.5 },
  },
};

export const slideInFromBottom = {
  hidden: { y: 100, opacity: 0 },
  visible: {
    y: 0,
    opacity: 1,
    transition: { delay: 0.5, duration: 0.5 },
  },
};
```

### What a Framer Motion "variant" is

A variant is a named animation state with values to animate to. Format:

```ts
{
  stateName1: { /* properties */, transition: { ... } },
  stateName2: { ... },
}
```

Then on a `<motion.div>`:

```jsx
<motion.div
  variants={myVariants}
  initial="stateName1"   // start here
  animate="stateName2"   // animate to here
/>
```

Framer reads the difference between the two states and tweens.

### Why functions for left/right, constants for top/bottom?

- `slideInFromLeft(delay)` — needs `delay` parameterized because every text block on the hero has a different `delay` to create a cascade.
- `slideInFromTop` — used once, with a fixed delay, so it's just a constant.

A function returning an object is a **factory**: every call returns a fresh object. That's important because Framer Motion compares object references — if you reused the same object across components, they'd share state (probably fine here, but bad practice in general).

### Properties Framer Motion understands

Inside `hidden`/`visible`:
- `x`, `y` — translate (px).
- `opacity` — 0 → 1.
- `scale` — 1 = normal.
- `rotate` — degrees.
- `backgroundColor`, `color` — CSS colors (interpolates).
- `pathLength` — for SVG path drawing animations.
- Pretty much any CSS property + custom MotionValues.

### Variants propagate to children

This is the killer feature. If a parent `<motion.div>` has variants and changes to "visible", any child `<motion.*>` that defines the same variant names will animate too:

```jsx
<motion.div variants={{ hidden: {...}, visible: {...} }} initial="hidden" animate="visible">
  <motion.h1 variants={titleVariants}>Title</motion.h1>
  <motion.p variants={textVariants}>Text</motion.p>
</motion.div>
```

The child variants get triggered without you having to wire each one. This SpacePortfolio doesn't use this pattern (each `<motion.div>` is independent with `<InView>`), but it's worth knowing.

### Common transition properties

```ts
transition: {
  delay: 0.5,            // seconds before starting
  duration: 0.5,         // seconds
  ease: "easeOut",       // or "linear", "easeIn", "easeInOut", or cubic-bezier array
  type: "spring",        // or "tween" (default)
  stiffness: 100,        // spring only
  damping: 10,           // spring only
  mass: 1,               // spring only
  staggerChildren: 0.1,  // for parents — delay between each child's animation
}
```

### What this file teaches you

- **Variants are the right way** to define declarative animations in Framer Motion — reusable, composable, parameterized.
- Co-locating variant definitions in a `utils/` file keeps components clean.
- TypeScript: `function name(delay: number)` typed parameter. The return type is inferred — Framer Motion doesn't require you to import its `Variants` type, but you could (`import { Variants } from "framer-motion"`).

### Build this from scratch in an interview

If they say "show me a reusable slide-in animation":

```ts
// utils/motion.ts
export const slideIn = (direction: "left" | "right" | "top" | "bottom", delay = 0) => {
  const axis = direction === "left" || direction === "right" ? "x" : "y";
  const sign = direction === "left" || direction === "top" ? -1 : 1;
  return {
    hidden: { [axis]: 100 * sign, opacity: 0 },
    visible: { [axis]: 0, opacity: 1, transition: { delay, duration: 0.5 } },
  };
};
```

One function instead of four. Same result. Always good to know how to refactor.
