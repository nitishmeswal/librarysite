# Personal-Portfolio/ — How It Works

A **React 19 + Vite + Tailwind v4** personal portfolio with a 3D astronaut, an interactive globe, animated particles, scroll-triggered reveals via a custom IntersectionObserver hook, mouse-driven 3D camera, and Framer Motion-powered word/character animations.

## Stack at a glance
- **React 19** (`react`, `react-dom`)
- **Vite** dev server
- **Tailwind CSS v4** (`@tailwindcss/vite` plugin)
- **Three.js** via `@react-three/fiber` + `@react-three/drei` + `maath`
- **Framer Motion** (`motion` package — Framer Motion's renamed npm package v12)
- **cobe** for the 3D globe
- **Web3Forms** for the contact form backend

## Folder map
```
Personal-Portfolio/
├── App.jsx              # Top-level layout: imports every section
├── main.jsx             # React 19 root + StrictMode
├── index.css            # Tailwind v4 + custom theme (@theme block)
├── sections/            # Big page sections (HOW_IT_WORKS in subfolder)
│   ├── Hero.jsx         # 3D Astronaut in <Canvas> + mouse-camera rig
│   ├── About.jsx        # Cards, Globe, scroll reveal grid
│   ├── Navbar.jsx       # Mobile menu w/ Framer Motion animation
│   ├── Project.jsx      # Hover preview w/ spring physics
│   ├── Experience.jsx   # Timeline
│   ├── Contact.jsx      # Form (Web3Forms)
│   └── Footer.jsx       # Social links
├── components/          # Reusables (HOW_IT_WORKS in subfolder)
│   ├── Astronaut.jsx, Globe.jsx, Particles.jsx,
│   ├── HeroText.jsx, FlipWords.jsx, ParrallaxBackground.jsx,
│   ├── ObitingCircles.jsx, Card.jsx, Alert.jsx, …
├── hooks/
│   └── useScrollReveal.js  # IntersectionObserver custom hook
└── constants/index.js   # All project / experience / social data
```

Subfolders have their own deep-dive guides. This file gives you the **top-down architecture**.

---

## 1. Entry point — `main.jsx`

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import './index.css';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>,
);
```

**Breakdown**:
- `createRoot(node).render(...)` — React 18+ concurrent root API.
- `<StrictMode>` — dev-only wrapper that double-invokes effects and renders to surface side-effect bugs. **Does not run in production builds.**

Compare with old `ReactDOM.render(<App />, node)` (React 17 and earlier).

---

## 2. App composition — `App.jsx`

```jsx
import About from './sections/About';
import Contact from './sections/Contact';
import Experience from './sections/Experience';
import Footer from './sections/Footer';
import Hero from './sections/Hero';
import Navbar from './sections/Navbar';
import Project from './sections/Project';
import Testimonial from './sections/Testimonial';

const App = () => {
  return (
    <div className="container mx-auto max-w-7xl">
      <Navbar />
      <Hero />
      <About />
      <Project />
      <Experience />
      <Testimonial />
      <Contact />
      <Footer />
    </div>
  );
};

export default App;
```

This is a single-page layout: one column, stacked sections. No router. The visual depth comes from CSS + animation, not navigation.

**Note**: `Testimonial` is imported but isn't in the sections list above (file may be missing). If you see `Testimonial is not defined` errors, that's why.

---

## 3. Tailwind v4 theme — `index.css`

The top of `index.css` looks like this (Tailwind v4 syntax — different from v3!):

```css
@import "tailwindcss";

@theme {
  --color-primary: #030412;
  --color-midnight: #06091f;
  --color-aqua: #33c2cc;
  --color-mint: #57db96;
  --color-royal: #5c33cc;
  --animate-orbit: orbit 50s linear infinite;
}
```

**What's different in Tailwind v4**:
- No more `tailwind.config.js` for the theme. Define everything in CSS using `@theme { --color-*: ... }`.
- Tailwind auto-generates utilities from your CSS variables (`bg-aqua`, `text-mint`, `animate-orbit`).
- Single `@import "tailwindcss";` instead of three `@tailwind base/components/utilities;` lines.

The `--animate-orbit` defines a custom keyframe animation that's used by `ObitingCircles.jsx` for orbiting icons.

The rest of `index.css` defines:
- Custom scrollbar styles.
- Scroll-reveal helper classes (`.scroll-reveal`, `.scroll-reveal-visible`, etc.) used together with `useScrollReveal`.
- Keyframes for `orbit`, `marquee`, `flicker`, etc.

---

## 4. The big architecture moves

### Mouse-tracking 3D camera (`sections/Hero.jsx`)

```jsx
import { Canvas, useFrame } from '@react-three/fiber';
import { Suspense } from 'react';
import { easing } from 'maath';
import HeroText from '../components/HeroText';
import ParrallaxBackground from '../components/ParrallaxBackground';
import { Astronaut } from '../components/Astronaut';
import Loader from '../components/Loader';

const Hero = () => {
  return (
    <section className="...">
      <HeroText />
      <ParrallaxBackground />
      <figure>
        <Canvas camera={{ position: [0, 1, 3] }}>
          <Suspense fallback={<Loader />}>
            <Float speed={5.5}>
              <Astronaut scale={0.23} position={[0, -1.5, 0]} />
            </Float>
            <Rig />
          </Suspense>
          <ambientLight intensity={1} />
          <directionalLight position={[10, 10, 10]} intensity={1.5} />
        </Canvas>
      </figure>
    </section>
  );
};

function Rig() {
  return useFrame((state, delta) => {
    easing.damp3(
      state.camera.position,
      [state.mouse.x / 10, 1 + state.mouse.y / 10, 3],
      0.5,
      delta,
    );
  });
}
```

**Breakdown**:
- `<Canvas camera={{ position: [0, 1, 3] }}>` — sets initial camera position (3 units back, 1 up).
- `<Suspense fallback={<Loader />}>` — shows the `Loader` (which uses `useProgress` from drei) while the glTF model loads.
- `<Float speed={5.5}>` — drei helper that bobs/rotates children automatically.
- `<Astronaut scale={0.23} position={[0, -1.5, 0]} />` — renders the loaded 3D astronaut.
- `<Rig />` — the magic. `useFrame` runs every render frame. `easing.damp3` interpolates the camera position toward `[mouse.x/10, 1+mouse.y/10, 3]` with a 0.5s "half-time". Result: the camera leans toward where the mouse is, smoothly.
- `state.mouse.x, state.mouse.y` are R3F-provided values in `-1..1` range.

**This 6-line Rig is what makes the hero feel alive.** Memorize it.

### Scroll reveal hook (`hooks/useScrollReveal.js`)

See `hooks/HOW_IT_WORKS.md` for the line-by-line.

### Parallax mountains (`components/ParrallaxBackground.jsx`)

```jsx
import { useScroll, useTransform, useSpring, motion } from 'motion/react';

const ParrallaxBackground = () => {
  const { scrollYProgress } = useScroll();
  const x = useSpring(scrollYProgress, { damping: 50 });
  const mountain3Y = useTransform(x, [0, 0.5], ['0%', '70%']);
  const planetsX = useTransform(x, [0, 0.5], ['0%', '-20%']);
  const mountain2Y = useTransform(x, [0, 0.5], ['0%', '30%']);
  const mountain1Y = useTransform(x, [0, 0.5], ['0%', '0%']);

  return (
    <section className="absolute inset-0">
      <div className="..." style={{ ... }}>
        <motion.div className="..." style={{ y: mountain3Y, backgroundImage: ... }} />
        <motion.div className="..." style={{ x: planetsX, ... }} />
        <motion.div className="..." style={{ y: mountain2Y, ... }} />
        <motion.div className="..." style={{ y: mountain1Y, ... }} />
      </div>
    </section>
  );
};
```

**Pattern**:
- `useScroll()` from `motion/react` — returns `{ scrollYProgress }`, a `MotionValue` (0..1).
- `useSpring(scrollYProgress, { damping: 50 })` — wraps it in a spring so it interpolates smoothly.
- `useTransform(input, inputRange, outputRange)` — maps the spring value 0..0.5 to a CSS percentage 0%..70%.
- Each `motion.div` gets one of these values as its `y` or `x`, and Framer Motion writes the CSS transform.

**Why MotionValues** instead of `useState` + `useEffect`? Because they update the DOM directly without triggering React re-renders. Massive perf win for scroll animations.

---

## 5. State management

This project uses **only local component state** — no Zustand, no Context. Every section is independent.

- `useState` for form fields, dropdowns, current word index.
- `useRef` for DOM nodes (canvas, refs into `IntersectionObserver`, GSAP-style anchors).
- `useScrollReveal` (custom hook) — only one piece of "shared logic".

This is fine because the page has no cross-section data dependencies. As soon as you'd add e.g. "dark mode toggle" or "cart count", you'd reach for Context or Zustand.

---

## 6. Build this from scratch (top-down plan)

1. `npm create vite@latest portfolio -- --template react`.
2. `npm i tailwindcss@latest @tailwindcss/vite three @react-three/fiber @react-three/drei maath motion cobe`.
3. Add `@import "tailwindcss"` + `@theme` to `index.css`.
4. Create one `App.jsx` that imports `Hero, About, Project, ...` from `sections/`.
5. Build each section bottom-up — `Hero` first (Canvas + Astronaut), `About` next (cards + globe), then forms.
6. Add `useScrollReveal` and apply it to anything you want to fade in.
7. Tweak Tailwind theme values until the colors match your brand.

A practical 2-week build.

---

## 7. What this project teaches you

- **React 19 root API** (`createRoot`).
- **Vite** as a build/dev tool (zero config to start).
- **Tailwind v4 `@theme`** syntax.
- **R3F + drei** for declarative 3D — `<Canvas>`, `<Suspense>`, `<Float>`, `useGLTF`, `useAnimations`, `useFrame`.
- **`easing.damp3`** from `maath` — buttery framerate-independent interpolation.
- **Framer Motion `MotionValue`s** — `useScroll`, `useSpring`, `useTransform` for direct-DOM scroll animation.
- **IntersectionObserver** wrapped in a custom hook.
- **Web3Forms** as a no-backend contact form solution.

Continue with the subfolder guides:
- [sections/HOW_IT_WORKS.md](./sections/HOW_IT_WORKS.md)
- [components/HOW_IT_WORKS.md](./components/HOW_IT_WORKS.md)
- [hooks/HOW_IT_WORKS.md](./hooks/HOW_IT_WORKS.md)
- [constants/HOW_IT_WORKS.md](./constants/HOW_IT_WORKS.md)
