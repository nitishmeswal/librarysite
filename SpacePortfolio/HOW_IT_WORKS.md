# SpacePortfolio/ — How It Works

A **Next.js 14 App Router + React 18 + Framer Motion + R3F** space-themed portfolio. Black-purple aesthetic, animated rotating star field as a global background, blackhole video in hero, Framer Motion variant-based slide-in animations everywhere.

## Stack
- **Next.js 14** with the App Router (`app/` directory).
- **React 18** (StrictMode by default).
- **Framer Motion** (`framer-motion` — the older package name; same library as `motion/react`).
- **React Three Fiber** + drei for the star background.
- **`react-intersection-observer`** for the `<InView>` render-prop component.
- **Tailwind CSS v3** (`tailwind.config.ts`).
- **TypeScript**.

## Folder map
```
SpacePortfolio/
├── app/
│   ├── layout.tsx          # Root layout: <StarsCanvas>, <Navbar>, children
│   ├── page.tsx            # Home page: Hero, About, Skills, Projects
│   └── globals.css         # Tailwind + custom CSS
├── components/
│   ├── main/               # Top-level page sections + StarBackground
│   └── sub/                # Reusable building blocks
├── constants/index.ts      # All static data (skills lists, project info)
├── utils/motion.ts         # Reusable Framer Motion variants
└── public/                 # Video, images, icons
```

Subfolders have their own deep-dive guides.

---

## 1. `app/layout.tsx` — root layout

```tsx
import type { Metadata } from "next";
import { Inter } from "next/font/google";
import "./globals.css";
import StarsCanvas from "@/components/main/StarBackground";
import Navbar from "@/components/main/Navbar";

const inter = Inter({ subsets: ["latin"] });

export const metadata: Metadata = {
  title: "Portfolio",
  description: "My portfolio",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className={`${inter.className} bg-[#030014] overflow-y-scroll overflow-x-hidden max-w-[1855px] mx-auto`}>
        <StarsCanvas />
        <Navbar />
        {children}
      </body>
    </html>
  );
}
```

### Line-by-line

- `import { Inter } from "next/font/google"` — Next.js's optimized font loader. `Inter({ subsets: ["latin"] })` downloads only Latin characters at build time, returns a class to apply.
- `export const metadata: Metadata` — Next.js App Router reads this object to set `<title>`, `<meta>` tags etc. Replaces the old `next/head` pattern.
- `RootLayout({ children })` — receives the current page as `children`. **This is a Server Component** (no `"use client"` directive).
- `<StarsCanvas />` — global animated background (mounted once at the root, shared across pages).
- `<Navbar />` — sticky nav at the top.
- `{children}` — current page renders here.

---

## 2. `app/page.tsx` — home page

```tsx
import About from "@/components/main/About";
import Hero from "@/components/main/Hero";
import Projects from "@/components/main/Projects";
import Skills from "@/components/main/Skills";

export default function Home() {
  return (
    <main className="h-full w-full">
      <div className="flex flex-col gap-20">
        <Hero />
        <About />
        <Skills />
        {/* <Projects /> */}
      </div>
    </main>
  );
}
```

- Also a Server Component — Next.js statically renders this at build time (no client JS to ship for the wrapper).
- The actual interactive bits (`Hero`, `Skills`) are themselves client components because they use `"use client"` + hooks.
- `<Projects />` is commented out — it's a half-built section. Worth noting in code review.

---

## 3. Global star background — `components/main/StarBackground.tsx`

```tsx
"use client";

import React, { useState, useRef, Suspense } from "react";
import { Canvas, useFrame } from "@react-three/fiber";
import { Points, PointMaterial } from "@react-three/drei";
// @ts-ignore
import * as random from "maath/random/dist/maath-random.esm";

const StarBackground = (props: any) => {
  const ref: any = useRef();
  const [sphere] = useState(() =>
    random.inSphere(new Float32Array(5000), { radius: 1.2 })
  );

  useFrame((state, delta) => {
    ref.current.rotation.x -= delta / 10;
    ref.current.rotation.y -= delta / 15;
  });

  return (
    <group rotation={[0, 0, Math.PI / 4]}>
      <Points ref={ref} positions={sphere} stride={3} frustumCulled {...props}>
        <PointMaterial
          transparent
          color="$fff"
          size={0.002}
          sizeAttenuation={true}
          dethWrite={false}
        />
      </Points>
    </group>
  );
};

const StarsCanvas = () => (
  <div className="w-full h-auto fixed inset-0 z-[20]">
    <Canvas camera={{ position: [0, 0, 1] }}>
      <Suspense fallback={null}>
        <StarBackground />
      </Suspense>
    </Canvas>
  </div>
);

export default StarsCanvas;
```

### Line-by-line

- `"use client"` — required at top because the file uses `useState`, `useRef`, `useFrame` (browser-only). Next.js would error otherwise on the server.
- `useState(() => random.inSphere(new Float32Array(5000), { radius: 1.2 }))`:
  - **Lazy initial state**: passing a function means `random.inSphere(...)` runs only on the first render (not on subsequent re-renders). Important because allocating a 5000-element typed array is expensive.
  - `Float32Array(5000)` — pre-allocates a buffer of 5000 floats (this represents ~1666 points × 3 floats per xyz).
  - `random.inSphere(buf, { radius: 1.2 })` — fills the buffer with random points inside a sphere of radius 1.2.
- `useFrame((state, delta) => { ref.current.rotation.x -= delta / 10; })`:
  - Every frame, decrement rotation by `delta/10` radians/sec → ~0.1 rad/s spin.
  - **Using `delta` keeps spin speed framerate-independent.** If we hardcoded `0.001`, then 60Hz vs 120Hz monitors would spin at different speeds.
- `<group rotation={[0, 0, Math.PI / 4]}>` — extra 45° roll for visual interest.
- `<Points positions={sphere} stride={3}>`:
  - `positions` is the typed array.
  - `stride={3}` tells Three.js: every 3 floats = one point's xyz.
  - `frustumCulled` (default true) — don't render points outside the camera's view frustum (perf).
- `<PointMaterial size={0.002} sizeAttenuation transparent>`:
  - Tiny dots.
  - `sizeAttenuation=true` makes farther points draw smaller (gives sense of depth).
  - `transparent=true` enables alpha blending.

### Bugs worth noting

- `color="$fff"` is a typo — should be `"#fff"`. Three falls back to default white-ish, so visually it works.
- `dethWrite={false}` — typo, should be `depthWrite`. The prop is silently dropped, so depth-write defaults to true.

These are great "fixed in PR review" examples for interviews.

### `<Canvas>` wrapper

- `<div className="w-full h-auto fixed inset-0 z-[20]">` — covers the viewport. `z-[20]` puts it above the page background.
- `<Canvas camera={{ position: [0, 0, 1] }}>` — close camera so the small points feel "huge".
- `<Suspense fallback={null}>` — required if any child suspends (though here `random.inSphere` is sync, so it's defensive).

---

## 4. Hero with a blackhole video — `components/main/Hero.tsx`

```tsx
import React from "react";
import HeroContent from "../sub/HeroContent";

const Hero = () => {
  return (
    <div className="relative h-full w-full" id="home">
      <video
        autoPlay
        muted
        loop
        className="rotate-100 absolute md:top-[-240px] lg:top-[-335px] top-[-400px] left-0 z-[0] w-full h-full object-cover"
      >
        <source src="/blackhole.webm" type="video/webm" />
      </video>
      <HeroContent />
    </div>
  );
};
```

### Key bits

- `<video autoPlay muted loop>` — modern browsers **require `muted`** for `autoPlay` to actually play. If you forget `muted`, the video won't auto-start.
- `<source src="/blackhole.webm" type="video/webm">` — vs hardcoding `src` on `<video>`, using `<source>` lets you list multiple fallbacks (e.g. mp4 + webm).
- `className="rotate-100 ..."` — `rotate-100` isn't a valid Tailwind class out of the box (Tailwind has `rotate-90`, `rotate-180` etc.). Probably another typo — works by accident as `0deg` if Tailwind drops it silently.
- `object-cover` — the video is cropped to fill the container, preserving aspect ratio.
- `z-[0]` — explicitly behind everything else.

---

## 5. The variant-based Framer pattern — `utils/motion.ts`

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
```

### The pattern

A **variant** in Framer Motion is a named animation state. Define `hidden` + `visible`, then on the component pass:

```jsx
<motion.div
  variants={slideInFromLeft(0.5)}
  initial="hidden"
  animate="visible"
>
```

Framer interpolates between the two named states. **Why this pattern wins**:
- Reusable: the same `slideInFromLeft` is used in many places with different delays.
- Composable: parents can pass variants down so children orchestrate.
- Cleaner JSX: no inline `initial={{x:-100,opacity:0}} animate={...}` repeated 20 times.

There are 4 helpers:
- `slideInFromLeft(delay)`, `slideInFromRight(delay)` — parameterized.
- `slideInFromTop`, `slideInFromBottom` — constants (delay is fixed).

---

## 6. `<InView>` render-prop wrapper

`react-intersection-observer` ships a hook (`useInView`) AND a render-prop component (`<InView>`):

```jsx
<InView triggerOnce={false}>
  {({ inView, ref }) => (
    <motion.div
      ref={ref}
      initial="hidden"
      animate={inView ? "visible" : "hidden"}
      variants={slideInFromLeft(0.5)}
    >
      …
    </motion.div>
  )}
</InView>
```

### How it works

- `<InView>` takes a function as its child (render-prop pattern).
- It calls the function with `{ inView, ref, entry }`.
- You attach `ref` to your element. When the element scrolls into view, `inView` becomes `true`, the child re-renders, Framer animates to `visible`.
- `triggerOnce={false}` means it re-fires on every entry/exit. Set `true` for "fade in once".

### Render-prop pattern

Why use a render-prop instead of the hook (`useInView`)? Two reasons:

1. **No need to convert your component to a function component first** (legacy class-component fix).
2. **You can use it inline** in JSX without lifting to a wrapper component.

Both patterns are fine. The hook is more idiomatic in modern React. This codebase uses the render-prop.

---

## 7. Architectural decisions

- **Global background mounted once** in layout: cheap (one WebGL context for whole app).
- **Page is a Server Component**: `app/page.tsx` doesn't have `"use client"` — Next.js pre-renders it. The interactive components inside (with `"use client"`) get bundled separately and hydrate.
- **No state management library**: tiny app, all state is local.

---

## What this project teaches you

- **Next.js App Router** server/client component split.
- **`next/font` for self-hosted Google Fonts** at build time.
- **Lazy initial state** in `useState(() => expensive())`.
- **`useFrame` + framerate-independent animation** with `delta`.
- **R3F `<Points>` + `<PointMaterial>`** for cheap particle fields.
- **Framer Motion variants** (the cleanest way to do "slide in from X" patterns).
- **`react-intersection-observer`** for scroll-into-view animation.
- **Auto-playing background video** + `muted; autoplay; loop`.

Continue with:
- [components/main/HOW_IT_WORKS.md](./components/main/HOW_IT_WORKS.md)
- [components/sub/HOW_IT_WORKS.md](./components/sub/HOW_IT_WORKS.md)
- [utils/HOW_IT_WORKS.md](./utils/HOW_IT_WORKS.md)
- [constants/HOW_IT_WORKS.md](./constants/HOW_IT_WORKS.md)
