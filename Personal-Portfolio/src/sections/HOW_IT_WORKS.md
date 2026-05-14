# Personal-Portfolio / src/sections — How It Works

Each file in this folder is a **page section** (Hero, About, etc.). They're stacked top-to-bottom by `App.jsx`. This guide walks every file with the actual code, line-by-line.

---

## 1. `Hero.jsx`

```jsx
import { Canvas, useFrame } from "@react-three/fiber";
import HeroText from "../components/HeroText";
import ParrallaxBackground from "../components/ParrallaxBackground";
import { Astronaut } from "../components/Astronaut";
import { Float, OrbitControls } from "@react-three/drei";
import { useMediaQuery } from "react-responsive";
import { easing } from "maath";
import { Suspense } from "react";
import Loader from "../components/Loader";

const Hero = () => {
  const isMobile = useMediaQuery({ maxWidth: 853 });
  return (
    <section id="home" className="flex items-start justify-center min-h-screen overflow-hidden md:items-start md:justify-start c-space">
      <HeroText />
      <ParrallaxBackground />
      <figure
        className="absolute inset-0"
        style={{ width: "100vw", height: "100vh" }}
      >
        <Canvas camera={{ position: [0, 1, 3] }}>
          <Suspense fallback={<Loader />}>
            <Float>
              <Astronaut
                scale={isMobile && 0.23}
                position={isMobile && [0, -1.5, 0]}
              />
            </Float>
            <Rig />
          </Suspense>
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
      delta
    );
  });
}

export default Hero;
```

### Line-by-line

- `useMediaQuery({ maxWidth: 853 })` — from `react-responsive`. Returns `true` if viewport is ≤853px (mobile).
- `<HeroText />` and `<ParrallaxBackground />` — siblings of the 3D canvas, painted **behind** the canvas because of stacking order in CSS (`figure` has `absolute inset-0`, so it overlays them).
- `<Canvas camera={{ position: [0, 1, 3] }}>` — initial camera at `(0, 1, 3)` (slightly up, 3 units back). FOV defaults to 75°.
- `<Suspense fallback={<Loader />}>` — required around components that call `useGLTF`/`useTexture` etc. While the asset loads, `<Loader />` (which uses drei's `useProgress`) shows a progress UI.
- `<Float>` — drei helper that wraps children in a slowly bobbing animation. Defaults: `speed=1, rotationIntensity=1, floatIntensity=1`.
- `scale={isMobile && 0.23}` — **bug worth noting**: `isMobile && 0.23` evaluates to `false` on desktop, and `<group scale={false}>` is invalid (Three will likely fall back to `1`). Cleaner would be `scale={isMobile ? 0.23 : 1}`.
- `<Rig />` — the mouse-camera follower.

### The `Rig` component (mouse-camera follower)

```jsx
function Rig() {
  return useFrame((state, delta) => {
    easing.damp3(
      state.camera.position,
      [state.mouse.x / 10, 1 + state.mouse.y / 10, 3],
      0.5,
      delta
    );
  });
}
```

- `useFrame((state, delta) => ...)` — runs on every render frame inside `<Canvas>`.
- `state.mouse.x`, `state.mouse.y` — normalized mouse position from R3F, `-1..1`.
- `state.camera.position` — a Three.js `Vector3` you can mutate directly.
- `easing.damp3(vector, target, smoothTime, delta)` — from `maath`. Critically-damped exponential interpolation toward `target`. `smoothTime` is roughly "seconds to halfway there". `delta` is frame time for framerate-independence.
- Result: camera leans gently toward the mouse, never overshoots, smooth as silk.

**Note**: `Rig` returns the result of `useFrame`, which is `undefined`. That's fine — it means `Rig` renders nothing visually, it just hooks into the frame loop. Cleaner would be to return `null` explicitly:

```jsx
function Rig() {
  useFrame((state, delta) => { /* ... */ });
  return null;
}
```

---

## 2. `Navbar.jsx`

```jsx
import { useState } from "react";
import { motion } from "motion/react";

function Navigation() {
  return (
    <ul className="nav-ul">
      <li className="nav-li"><a className="nav-link" href="#home">Home</a></li>
      <li className="nav-li"><a className="nav-link" href="#about">About</a></li>
      <li className="nav-li"><a className="nav-link" href="#work">Work</a></li>
      <li className="nav-li"><a className="nav-link" href="#contact">Contact</a></li>
    </ul>
  );
}

const Navbar = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <div className="fixed inset-x-0 z-20 w-full backdrop-blur-lg bg-primary/40">
      <div className="mx-auto c-space max-w-7xl">
        <div className="flex items-center justify-between py-2 sm:py-0">
          <a href="/" className="text-xl font-bold ...">VX</a>
          <button
            onClick={() => setIsOpen(!isOpen)}
            className="flex cursor-pointer ... sm:hidden"
          >
            <img
              src={isOpen ? "assets/close.svg" : "assets/menu.svg"}
              className="w-6 h-6"
              alt="toggle"
            />
          </button>
          <nav className="hidden sm:flex">
            <Navigation />
          </nav>
        </div>
      </div>
      {isOpen && (
        <motion.div
          className="block overflow-hidden text-center sm:hidden"
          initial={{ opacity: 0, x: -10 }}
          animate={{ opacity: 1, x: 0 }}
          style={{ maxHeight: "100vh" }}
          transition={{ duration: 1 }}
        >
          <nav className="pb-5">
            <Navigation />
          </nav>
        </motion.div>
      )}
    </div>
  );
};
```

### Patterns

- **Inner helper component `Navigation`**: defined inside the same file, not exported. Reused for both desktop and mobile menus. Avoids duplication.
- `useState(false)` — `isOpen` tracks if mobile menu is shown.
- `setIsOpen(!isOpen)` — flips it. **Gotcha**: if you call this twice in the same handler, it only flips once (stale closure). Use `setIsOpen(prev => !prev)` if you'd ever chain calls.
- `<img src={isOpen ? "close.svg" : "menu.svg"}>` — swap icon based on state.
- `hidden sm:flex` — Tailwind responsive: hidden on mobile, flex on ≥`sm` breakpoint.
- `{isOpen && <motion.div ...>}` — mount the mobile menu only when open. Unmounting triggers Framer to reverse animations on next open.
- `initial={{...}} animate={{...}} transition={{...}}` — Framer Motion declarative animation. On mount, animates from initial to animate over `duration: 1s`.

---

## 3. `About.jsx`

```jsx
import React, { useRef } from "react";
import Card from "../components/Card";
import { Globe } from "../components/Globe";
import CopyEmailButton from "../components/CopyEmailButton";
import { Frameworks } from "../components/Frameworks";
import { useScrollReveal } from "../hooks/useScrollReveal";

const About = () => {
  const grid2Container = useRef();
  const [sectionRef, isVisible] = useScrollReveal({ threshold: 0.1, once: true });

  return (
    <section id="about" className="c-space section-spacing">
      <h2 className="text-heading ">About Me</h2>
      <div
        ref={sectionRef}
        className={`grid grid-cols-1 gap-4 md:grid-cols-6 md:auto-rows-[18rem] mt-12 scroll-reveal ${isVisible ? 'visible' : ''}`}
      >
        {/* Grid 1 */}
        <div className="flex items-end grid-default-color grid-1">
          <img src="assets/coding-pov.png" className="absolute scale-[1.75] ..." />
          <div className="z-10">
            <p className="headtext">Hi I'm Kasam</p>
            <p className="subtext">Over the last 4 years, ...</p>
          </div>
        </div>

        {/* Grid 2 — scattered draggable cards */}
        <div className="grid-default-color grid-2">
          <div ref={grid2Container} className="flex items-center justify-center w-full h-full">
            <p className="flex items-end text-5xl text-gray-500">VX CODE CRAFT</p>
            <Card style={{ rotate: "75deg", top: "30%", left: "20%" }} text="GRASP" containerRef={grid2Container} />
            <Card style={{ rotate: "-30deg", top: "60%", left: "45%" }} text="React" containerRef={grid2Container} />
            <Card style={{ rotate: "90deg", bottom: "30%", left: "75%" }} text="Nextjs" containerRef={grid2Container} />
            <Card style={{ rotate: "-45deg", top: "55%", left: "0%" }} text="Design Principles" containerRef={grid2Container} />
            <Card style={{ rotate: "20deg", top: "10%", left: "38%" }} text="SRP" containerRef={grid2Container} />
            <Card style={{ rotate: "30deg", top: "70%", left: "70%" }} image="assets/logos/csharp-pink.png" containerRef={grid2Container} />
            <Card style={{ rotate: "-45deg", top: "70%", left: "25%" }} image="assets/logos/dotnet-pink.png" containerRef={grid2Container} />
            <Card style={{ rotate: "-45deg", top: "5%", left: "10%" }} image="assets/logos/blazor-pink.png" containerRef={grid2Container} />
          </div>
        </div>

        {/* Grid 3 — Globe */}
        <div className="grid-black-color grid-3">
          <div className="z-10 w-[50%]">
            <p className="headtext">Time Zone</p>
            <p className="subtext">I am currently based in the UK, and open to work</p>
          </div>
          <figure className="absolute left-[30%] top-[10%]">
            <Globe />
          </figure>
        </div>

        {/* Grid 4 — CopyEmailButton */}
        <div className="grid-special-color grid-4">
          <div className="flex flex-col items-center justify-center gap-5 size-full">
            <p className="text-center headtext">
              Do You want to Start a Project Together?
              <CopyEmailButton />
            </p>
          </div>
        </div>

        {/* Grid 5 — Tech Stack */}
        <div className="grid-default-color grid-5">
          <div className="z-1 w-[50%]">
            <p className="headtext">Tech Stack</p>
            <p className="subtext">I specialize in a variety of languages, frameworks, and tools ...</p>
          </div>
          <div className="absolute inset-y-0 md:inset-y-9 w-full h-full start-[50%] md:scale-125">
            <Frameworks />
          </div>
        </div>
      </div>
    </section>
  );
};
```

### Key patterns

- **Bento grid**: `grid-cols-1 gap-4 md:grid-cols-6 md:auto-rows-[18rem]` plus per-cell classes (`.grid-1` `.grid-2` `.grid-3`...) that span different columns/rows. Look at `index.css` to see e.g. `.grid-1 { grid-column: span 3 / span 3; }`.
- **`useRef()` as a constraint container**: `grid2Container` is passed to each `Card` so the Card's `<motion.div drag dragConstraints={containerRef}>` can clamp dragging to the bento cell.
- **Scroll reveal**: `[sectionRef, isVisible] = useScrollReveal({ threshold: 0.1, once: true });`. The grid wrapper toggles a `.visible` class when the section scrolls 10% into view.

---

## 4. `Project.jsx`

```jsx
import { useState } from "react";
import Projects from "../components/Projects";
import { myProjects } from "../constants";
import { motion, useMotionValue, useSpring } from "motion/react";
import { useScrollReveal } from "../hooks/useScrollReveal";

const Project = () => {
  const x = useMotionValue(0);
  const y = useMotionValue(0);
  const springX = useSpring(x, { damping: 10, stiffness: 50 });
  const springY = useSpring(y, { damping: 10, stiffness: 50 });
  const handleMouseMove = (event) => {
    x.set(event.clientX + 20);
    y.set(event.clientY + 20);
  };

  const [preview, setPreview] = useState(null);
  const [sectionRef, isVisible] = useScrollReveal({ threshold: 0.1, once: true });

  return (
    <section onClick={handleMouseMove} className="relative c-space section-spacing" id="projects">
      <h2 className="text-heading">My Selected Projects</h2>
      <div ref={sectionRef} className={`bg-gradient-to-r ... scroll-reveal-fade ${isVisible ? "visible" : ""}`} />
      {myProjects.map((project, index) => (
        <Projects
          key={project.id}
          {...project}
          setPreview={setPreview}
          index={index}
        />
      ))}
      {preview && (
        <motion.img
          className="fixed top-0 left-0 z-50 object-cover h-56 rounded-lg shadow-lg pointer-events-auto w-80"
          src={preview}
          style={{ x: springX, y: springY }}
        />
      )}
    </section>
  );
};
```

### What's brilliant about this code

- **`useMotionValue`** holds a number without re-rendering React when it changes. Updating `x.set(...)` updates the linked DOM element directly. Animate 60fps without choking React.
- **`useSpring`** wraps a MotionValue to add spring physics (damping/stiffness control the feel).
- **`event.clientX + 20`** — offset so the preview image doesn't sit directly under the cursor.
- **`onClick={handleMouseMove}`** — note: this should likely be `onMouseMove`, but as-written the preview position only updates on click. Minor bug worth catching in code review.
- **`{preview && <motion.img ...>}`** — only render the preview when a project is hovered (`setPreview(img)` is called by the child `Projects` component on hover, then `setPreview(null)` on leave).
- **`style={{ x: springX, y: springY }}`** — Framer Motion writes these as `transform: translate(x, y)` directly to the DOM.

---

## 5. `Contact.jsx`

```jsx
import { useState } from "react";
import Alert from "../components/Alert";
import { Particles } from "../components/Particles";
import { useScrollReveal } from "../hooks/useScrollReveal";

const WEB3FORMS_ACCESS_KEY = "b858e62d-4977-4c34-80b2-3955129ea0e5";

const Contact = () => {
  const [formData, setFormData] = useState({ name: "", email: "", message: "" });
  const [isLoading, setIsLoading] = useState(false);
  const [showAlert, setShowAlert] = useState(false);
  const [alertType, setAlertType] = useState("success");
  const [alertMessage, setAlertMessage] = useState("");
  const [sectionRef, isVisible] = useScrollReveal({ threshold: 0.1, once: true });

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const showAlertMessage = (type, message) => {
    setAlertType(type);
    setAlertMessage(message);
    setShowAlert(true);
    setTimeout(() => setShowAlert(false), 5000);
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsLoading(true);
    try {
      const form = new FormData(e.target);
      form.append("access_key", WEB3FORMS_ACCESS_KEY);
      form.append("subject", `New contact from ${formData.name}`);

      const response = await fetch("https://api.web3forms.com/submit", {
        method: "POST",
        body: form,
      });
      const result = await response.json();

      if (result.success) {
        setFormData({ name: "", email: "", message: "" });
        showAlertMessage("success", "Your message has been sent!");
      } else {
        throw new Error(result.message || "Something went wrong");
      }
    } catch (error) {
      console.error(error);
      showAlertMessage("danger", "Something went wrong!");
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <section id="contact" className="relative flex items-center c-space section-spacing">
      <Particles className="absolute inset-0 z-50" quantity={100} ease={80} color={"#ffffff"} refresh />
      {showAlert && <Alert type={alertType} text={alertMessage} />}
      <div
        ref={sectionRef}
        className={`flex flex-col items-center justify-center max-w-md ... scroll-reveal ${isVisible ? 'visible' : ''}`}
      >
        <form className="w-full" onSubmit={handleSubmit}>
          <input id="name" name="name" type="text" value={formData.name} onChange={handleChange} required />
          <input id="email" name="email" type="email" value={formData.email} onChange={handleChange} required />
          <textarea id="message" name="message" rows={4} value={formData.message} onChange={handleChange} required />
          <button type="submit">{!isLoading ? "send" : "Sending..."}</button>
        </form>
      </div>
    </section>
  );
};
```

### Controlled-form pattern (textbook)

- `useState({ name: "", email: "", message: "" })` — single state object for the whole form.
- `handleChange` uses **computed property names**: `[e.target.name]: e.target.value`. The `name` attr on each input matches a key in `formData`. One handler, all fields.
- `value={formData.name}` + `onChange={handleChange}` = controlled input.
- `required` on each — browser-level validation. The submit handler won't fire until they're filled.

### Form submission pattern (Web3Forms)

- `new FormData(e.target)` — builds form payload from all `name`d fields automatically. Plus you can `.append()` extra fields.
- `WEB3FORMS_ACCESS_KEY` is **public** (it's tied to an email destination, not an account secret). Still, in production you'd put it in `.env` and access via `import.meta.env.VITE_WEB3FORMS_KEY`.
- `fetch(..., { method: "POST", body: form })` — when `body` is a `FormData`, fetch sets `Content-Type: multipart/form-data` for you. Don't set it manually or you'll break the boundary.
- `try/catch/finally` — clean async error handling. `finally` resets `isLoading`.

### Toast pattern

- `showAlertMessage(type, msg)` mutates 3 pieces of state then sets a 5s timer to hide.
- Worth knowing: if the user submits twice in a row, the second `setTimeout` runs concurrently with the first — both will set `showAlert(false)`, which is harmless. In production you'd `clearTimeout` to be safe.

---

## 6. `Experience.jsx`

```jsx
import { experiences } from '../constants';
import Timeline from '../components/Timeline';

const Experience = () => {
  return (
    <section className="c-space section-spacing" id="work">
      <h2 className="text-heading">My Work Experience</h2>
      <div className="mt-12">
        <Timeline experiences={experiences} />
      </div>
    </section>
  );
};
```

Tiny — just renders the `<Timeline>` component fed by `constants/index.js` data. All the visual logic is inside `Timeline`.

---

## 7. `Footer.jsx`

```jsx
import { socials } from '../constants';

const Footer = () => {
  return (
    <footer className="...">
      <p>© 2025 VX</p>
      <div className="flex gap-4">
        {socials.map(({ name, href, icon: Icon }) => (
          <a key={name} href={href}><Icon /></a>
        ))}
      </div>
    </footer>
  );
};
```

The clean pattern: destructure `icon: Icon` while mapping. The lowercase `icon` from the data is renamed to capital `Icon` so JSX can render it as a component.

---

## Patterns to extract for any portfolio you build

1. **Bento grid** via Tailwind `grid-cols-N` + `auto-rows-Xrem` + per-cell `grid-column`/`grid-row` rules.
2. **Scroll reveal** via a custom IntersectionObserver hook + a CSS class toggle.
3. **Mouse follower** via Framer Motion MotionValues + Spring.
4. **Controlled form** via single `useState` object + computed-property `handleChange`.
5. **3D hero** via R3F `<Canvas>` + `<Suspense>` + `<Float>` + a `useFrame` "Rig" that follows the mouse.
6. **Mobile menu** via `useState` + Framer Motion mount animation.

You can mix-and-match these to build any portfolio you want.
