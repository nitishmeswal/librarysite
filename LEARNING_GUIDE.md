# Library Site — Learning Guide (SDE 1 Frontend Interview Prep)

This repo is a **collection of 11 production-grade animated / 3D / scroll-driven websites**. It is a perfect playground for an SDE 1 frontend interview because it touches every major modern frontend topic in tiny, copyable form:

- HTML / CSS / vanilla JS fundamentals
- React + Next.js component architecture
- Animation: GSAP timelines, ScrollTrigger, SplitText, Framer Motion
- 3D graphics: Three.js + React Three Fiber + drei
- Smooth scroll with Lenis
- Headless CMS (Prismic)
- Tailwind CSS (v3 and v4)
- State (Zustand, useState, useRef, useSyncExternalStore)
- Performance patterns (instanced meshes, requestAnimationFrame, IntersectionObserver)

## How to use this set of guides

There are 5 root-level guides plus a `HOW_IT_WORKS.md` in **every** project folder (and in major sub-folders for the bigger projects).

| File | What it covers |
| --- | --- |
| `LEARNING_GUIDE.md` (this file) | Big picture, learning order, what to read first |
| `REACT_NEXT_GUIDE.md` | React + Next.js fundamentals you must know cold for SDE 1 |
| `GSAP_GUIDE.md` | GSAP from zero — timelines, eases, ScrollTrigger, SplitText, useGSAP |
| `THREEJS_GUIDE.md` | Three.js + R3F from zero — scene, camera, renderer, mesh, shaders |
| `INTERVIEW_PREP.md` | Concrete interview answers, build-from-scratch checklists, common Qs |

Then for each project:

| Project | What you learn |
| --- | --- |
| `bee/` | Pure HTML+CSS layout, fixed/sticky decoration, `::before` mirror reflection |
| `paralax/` | Mouse-based 3D parallax with `perspective()` and GSAP intro timeline |
| `paralax copy/` | Same as above (reference copy) |
| `site-gloria/` | Big "real" HTML/CSS/JS site — BEM naming, vanilla layout patterns |
| `Ironhill-section-rebuild/` | Three.js + custom GLSL fragment shader + Lenis + SplitText scroll reveal |
| `Personal-Portfolio/` | React 19 + Vite + R3F + Framer Motion + Tailwind v4 + custom IntersectionObserver hook |
| `SpacePortfolio/` | Next.js 14 + R3F particles + Framer Motion `slideInFrom*` variants |
| `Spylt-awward-clone/` | React 19 + GSAP `useGSAP` + SplitText + horizontal pinned scroll |
| `mojito-awwwards-website/` | Vite + TS + GSAP + **video scrubbing on scroll** |
| `Fizzi-3D-Website/` | Next.js + Prismic CMS + R3F + GSAP — production-grade 3D + headless CMS |
| `CODEBASE_ANALYSIS.md` | Existing overview (kept as-is) |

---

## Recommended learning order

This is the order I'd follow if I had ~2 weeks before an SDE 1 interview.

### Week 1 — Fundamentals

1. **`REACT_NEXT_GUIDE.md`** — make sure rendering, hooks, refs, props, controlled vs uncontrolled, useEffect deps, lifting state, key prop, conditional rendering are second nature.
2. **`bee/`** — pure CSS layout, see how a "hero with mirror reflection" is done with `attr()` + `mask` + `transform: scaleY(-1)`.
3. **`paralax/`** — vanilla JS mouse parallax (very interview-friendly: shows you understand DOM, dataset, transforms, perspective).
4. **`site-gloria/`** — read selectively. Skim it to see what a "real" hand-written HTML/CSS site looks like in BEM convention.
5. **`GSAP_GUIDE.md`** + **`Ironhill-section-rebuild/`** — GSAP `to/from/timeline`, `ScrollTrigger`, `SplitText`, Lenis smooth scroll.

### Week 2 — Animation + 3D + production patterns

6. **`THREEJS_GUIDE.md`** — minimal Three.js scene from scratch, then R3F equivalent.
7. **`SpacePortfolio/`** — easiest React+R3F site (just rotating stars, Framer Motion variants).
8. **`Personal-Portfolio/`** — IntersectionObserver hook, COBE globe, particles canvas, motion + 3D mixed.
9. **`Spylt-awward-clone/`** — GSAP `useGSAP` hook, horizontal pinned scroll, SplitText character animations.
10. **`mojito-awwwards-website/`** — video scrubbing on scroll (`currentTime` driven by ScrollTrigger).
11. **`Fizzi-3D-Website/`** — combine everything: Next.js App Router + Prismic + global `<View>` 3D canvas + Zustand + GSAP scroll choreography.

### The day before the interview

- Re-read **`INTERVIEW_PREP.md`** — it gives you concrete answers to "How does a scroll animation work?", "What is a ref?", "How do you avoid re-renders?", "Explain hydration", etc.
- Pick **2 projects** you understand best (I recommend `paralax/` and `Spylt-awward-clone/`) and be ready to whiteboard them. Interviewers love when you can say "here's how I'd build a scroll-driven hero — set up GSAP timeline, register ScrollTrigger, use refs to target DOM, …".

---

## What an SDE 1 frontend interview actually wants

For an SDE 1 frontend, you are **not** expected to invent novel animation engines. You ARE expected to:

1. **Explain React rendering** — what triggers a re-render, why keys matter, what happens on mount/unmount, the difference between `useEffect` and `useLayoutEffect`.
2. **Use hooks correctly** — `useState`, `useRef`, `useEffect`, custom hooks. Know the rules (only at top level, only inside components/hooks).
3. **Write idiomatic JS** — destructuring, array methods (`map/filter/reduce`), spread, template literals, async/await.
4. **Write semantic, accessible HTML/CSS** — flexbox/grid, responsive design, basic accessibility (`alt`, `aria-*`, `sr-only`).
5. **Talk through a feature build** — given "build a hero with scroll-triggered text reveal", you should be able to whiteboard it in 5 minutes.
6. **Demonstrate curiosity about animation/3D** — even rough familiarity with `transform`, `transition`, `requestAnimationFrame`, and one library (GSAP) puts you well ahead of most SDE 1 candidates.

If you understand the contents of these guides you are **well above** the SDE 1 bar. The fancy 3D stuff (Three.js, R3F, custom shaders) is bonus — it's "wow" material to bring up when asked "tell me about a project you're proud of".

---

## Convention used in every per-folder guide

Every per-folder `HOW_IT_WORKS.md` follows the same shape:

1. **What this folder/file does** (one paragraph).
2. **The actual code, copy-pasted verbatim**, in fenced code blocks.
3. **Line-by-line breakdown** — what each line does, why it's there, what would break if you removed it.
4. **Patterns you can reuse** — generalizable lessons.
5. **Build it from scratch** — minimal version you could write in an interview.

Code is copied **exactly as-is** from the project. If something looks wrong (typo, dead code), the guide will say so rather than silently fix it.
