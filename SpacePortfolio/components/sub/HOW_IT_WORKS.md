# SpacePortfolio / components/sub — How It Works

The smaller, reusable building blocks composed by the `main/` sections.

| File | Used in | Purpose |
| --- | --- | --- |
| `HeroContent.tsx` | `Hero.tsx` | All text + button + image inside hero. |
| `SkillText.tsx` | `Skills.tsx` | "My Skills" header + tagline. |
| `SkillDataProvider.tsx` | `Skills.tsx` | Renders a single skill icon with fade-in. |
| `ProjectCard.tsx` | `Projects.tsx` | One project card (image + title + description). |

---

## 1. `HeroContent.tsx` — the hero text + image

The file is ~170 lines but mostly repetition. Key block:

```tsx
"use client";
import { motion } from "framer-motion";
import { slideInFromLeft, slideInFromRight, slideInFromTop } from "@/utils/motion";
import { BsStars } from "react-icons/bs";
import Image from "next/image";
import { InView } from "react-intersection-observer";

const HeroContent = () => {
  return (
    <InView triggerOnce={false}>
      {({ inView, ref }) => (
        <motion.div
          ref={ref}
          initial="hidden"
          animate={inView ? "visible" : "hidden"}
          className="flex md:flex-row flex-col-reverse items-center justify-center gap-10 md:gap-0 md:px-20 px-5 mt-40 w-full z-20"
        >
          {/* Left column: text */}
          <div className="h-full w-full md:w-3/6 flex flex-col gap-5 justify-center text-start">
            {/* Pill badges */}
            <div className="hidden md:flex flex-row items-center md:gap-5 gap-1">
              <InView triggerOnce={false}>
                {({ inView, ref }) => (
                  <motion.div ref={ref} initial="hidden"
                              animate={inView ? "visible" : "hidden"}
                              variants={slideInFromTop}
                              className="Welcome-box py-[8px] px-[7px] border border-[#7042f88b] opacity-[0.9]">
                    <BsStars className="text-[#b49bff] mr-[10px] h-5 w-5" />
                    <h1 className="Welcome-text text-[13px]">Fullstack Developer</h1>
                  </motion.div>
                )}
              </InView>
              {/* … two more identical pills, "Tech Innovator", "Team Lead" … */}
            </div>
            {/* Main headline */}
            <motion.div variants={slideInFromLeft(0.5)} ...>
              <span>Coding<span className="...">Dreams</span> into</span>
            </motion.div>
            {/* Subtitle, button */}
          </div>
          {/* Right column: image */}
          <motion.div variants={slideInFromRight(0.8)}>
            <Image src="/mainIconsdark.svg" alt="hero" height={650} width={650} />
          </motion.div>
        </motion.div>
      )}
    </InView>
  );
};
```

### Things to learn

- **`react-icons/bs`** — Bootstrap icons exposed as React components. Other namespaces: `fa` (FontAwesome), `ai` (Ant), `md` (Material), `io` (Ionicons). Tree-shakable.
- **Inline `<motion.h1>` for pill badges** — each pill is wrapped in its own `<InView>` so it animates in on entering viewport.
- **Nested `<InView>`** — outer for the container, inner for individual pills. They're independent; the inner one fires when its own ref enters the viewport.
- **Gradient text in headline** — `<span className="bg-gradient-to-r from-purple-500 to-cyan-500 bg-clip-text text-transparent">Dreams</span>`.

### The "if I rewrote this" opportunity

This file has the same `<InView>{({ ref, inView }) => <motion.div ...>}</InView>` block 4-5 times. Extract an `<AnimatedBlock>` wrapper as discussed in `components/main/HOW_IT_WORKS.md`.

---

## 2. `SkillText.tsx` — section heading

```tsx
"use client";
import { motion } from "framer-motion";
import { slideInFromLeft, slideInFromTop } from "@/utils/motion";
import { InView } from "react-intersection-observer";

const SkillText = () => {
  return (
    <div className="w-full h-auto pt-20 flex flex-col items-center justify-center">
      <InView triggerOnce={false}>
        {({ inView, ref }) => (
          <motion.div
            ref={ref}
            initial="hidden"
            animate={inView ? "visible" : "hidden"}
            variants={slideInFromTop}
            className="text-[40px] font-medium text-center text-gray-200 z-50"
          >
            My
            <span className="text-transparent bg-clip-text bg-gradient-to-r from-purple-500 to-cyan-500">
              {" "}Skills{" "}
            </span>
          </motion.div>
        )}
      </InView>
      <InView triggerOnce={false}>
        {({ inView, ref }) => (
          <motion.div
            ref={ref}
            initial="hidden"
            animate={inView ? "visible" : "hidden"}
            variants={slideInFromLeft(0.5)}
            className="cursive text-[20px] text-gray-200 mb-10"
          >
            Never miss a task, deadline or idea
          </motion.div>
        )}
      </InView>
    </div>
  );
};
```

Two stacked animated blocks — same patterns as everywhere.

---

## 3. `SkillDataProvider.tsx` — single skill icon with fade-in (uses the `useInView` HOOK, not the `<InView>` render-prop)

```tsx
"use client";
import { motion } from "framer-motion";
import { useInView } from "react-intersection-observer";
import Image from "next/image";

interface Props {
  src: string;
  width: number;
  height: number;
  index: number;
}

const SkillDataProvider = ({ src, width, height, index }: Props) => {
  const { ref, inView } = useInView({ triggerOnce: true });
  const imageVariants = {
    hidden: { opacity: 0 },
    visible: { opacity: 1 },
  };
  const animationDelay = 0.3;
  return (
    <motion.div
      ref={ref}
      initial="hidden"
      variants={imageVariants}
      animate={inView ? "visible" : "hidden"}
      custom={index}
      transition={{ delay: index * animationDelay }}
    >
      <Image src={src} width={width} height={height} alt="skill image" />
    </motion.div>
  );
};
```

### Line-by-line

- **`useInView({ triggerOnce: true })`** — the hook form (cleaner than the render-prop in a small component). Returns `{ ref, inView, entry }`.
- `triggerOnce: true` — once the icon is in view, the observer disconnects. Animation plays once.
- **Cascade delay**: `transition={{ delay: index * 0.3 }}` — first icon at 0s, second at 0.3s, third at 0.6s, etc. Creates a wave effect.
- `custom={index}` — passes a custom value into the variant. Useful only if a variant is a function like `(i) => ({...})`. Here `imageVariants` is a plain object, so `custom={index}` does nothing — leftover code. Worth knowing the pattern though.

### The `custom` prop with function variants

```ts
const variants = {
  hidden: { opacity: 0 },
  visible: (i: number) => ({ opacity: 1, transition: { delay: i * 0.3 } }),
};
<motion.div variants={variants} custom={index} initial="hidden" animate="visible" />
```

Then `custom={index}` actually does something — Framer calls `variants.visible(index)`.

The author here just put `transition` on `motion.div` directly with `delay: index * 0.3`. Functionally equivalent.

---

## 4. `ProjectCard.tsx` — pure presentation

```tsx
const ProjectCard = ({ src, title, description }: Props) => {
  return (
    <div className="relative overflow-hidden rounded-lg shadow-lg border border-[#2A0E61]">
      <Image src={src} alt={title} width={1000} height={1000} className="w-full object-contain" />
      <div className="relative p-4">
        <h1 className="text-2xl font-semibold text-white">{title}</h1>
        <p className="mt-2 text-gray-300">{description}</p>
      </div>
    </div>
  );
};
```

A **dumb component** — no state, no hooks, just renders props. This is the right kind of component to have lots of: every project is rendered by this same card.

### Why `width={1000} height={1000}` if the image is responsive?

Those are the intrinsic dimensions for `next/image`. The actual visual size comes from `className="w-full"` and `object-contain`. Without `width`/`height` props, Next throws an error (it needs them to reserve space and avoid layout shift).

---

## Patterns to internalize

- **Render-prop vs hook for `<InView>`** — they're equivalent, prefer the hook in new code.
- **`useInView` + `triggerOnce`** — animate once when entering viewport.
- **Cascade delay** with `delay: index * 0.3` — universal pattern for staggered reveals.
- **Dumb presentational components** for repeated UI — pass everything as props.
- **`react-icons` for vector icons** as React components.

For testing in interviews: rebuild `SkillDataProvider` from scratch — it's small but demonstrates `useInView`, variants, `custom`, and `delay`. Solid 10-minute live-coding exercise.
