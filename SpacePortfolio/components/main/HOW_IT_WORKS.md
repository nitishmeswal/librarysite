# SpacePortfolio / components/main — How It Works

Top-level page sections + the global star background. Each file is one section of the site.

| File | What it does |
| --- | --- |
| `StarBackground.tsx` | Global rotating star particle field (mounted in layout, behind everything). |
| `Navbar.tsx` | Sticky top bar with logo, anchor links, social icons. |
| `Hero.tsx` | Looping blackhole video + `<HeroContent />`. |
| `About.tsx` | Profile photo + name + bio, all sliding in. |
| `Skills.tsx` | 4 skill panels each rendering icons via `<SkillDataProvider>`. |
| `Projects.tsx` | Grid of project cards (commented out on home page). |
| `Footer.tsx` | Bottom links. |

We already covered `StarBackground.tsx` and `Hero.tsx` in the parent guide. This page goes deep on the other sections.

---

## 1. `Navbar.tsx` — sticky nav with anchor scrolling

```tsx
import { Socials } from "@/constants";
import Image from "next/image";

const Navbar = () => {
  return (
    <div className="w-screen md:w-full h-[65px] fixed top-0 shadow-lg shadow-[#2A0E61]/50 bg-[#03001417] backdrop-blur-md z-50 px-10 m-0 max-w-[1855px] items-center rounded-full">
      <div className="w-full h-full flex flex-row items-center justify-between m-auto px-[0px] md:px-[10px]">
        <a href="#home" className="h-auto w-auto flex flex-row items-center">
          <Image src="/logo.png" alt="logo" width={50} height={50}
                 className="cursor-pointer hover:animate-spin w-10" />
          <span className="font-bold ml-[10px] block text-gray-300 z-50 md:text-lg text-xl">
            Jenin Joseph
          </span>
        </a>
        <div className="hidden w-3/6 lg:w-1/3 h-full md:flex flex-row items-center justify-between md:mx-auto lg:pr-12">
          <div className="flex items-center justify-between w-full h-auto border border-[#7042f861] bg-[#0300145e] mr-[15px] px-[20px] py-[10px] rounded-full text-gray-200">
            <a href="#about" className="cursor-pointer">About me</a>
            <a href="#skills" className="cursor-pointer">Skills</a>
            <a href="#projects" className="cursor-pointer">Projects</a>
          </div>
        </div>
        <div className="flex flex-row gap-5 text-white">
          {Socials.map((social) => (
            <a href={social.link} key={social.name} target="_blank" rel="noopener noreferrer">
              <Image src={social.src} alt={social.name} width={24} height={24}
                     className="cursor-pointer hover:animate-spin" />
            </a>
          ))}
        </div>
      </div>
    </div>
  );
};
```

### What's interesting

- **Anchor-based scroll navigation**: `<a href="#about">` jumps to `<section id="about">`. Modern browsers also handle `scroll-behavior: smooth` (set globally in `globals.css`) so the jump becomes a smooth scroll.
- **`fixed top-0 z-50`** with `backdrop-blur-md` and a 0x000000+alpha background — classic glass-morphism sticky header.
- **`hover:animate-spin`** — Tailwind utility for `animation: spin 1s linear infinite`. Applied conditionally on hover.
- **`hidden ... md:flex`** — mobile-first: hide on small, show as flex on md+. Means mobile users don't see the inline nav (presumably a hamburger is missing here — TODO).
- **`target="_blank" rel="noopener noreferrer"`** — security best practice for external links. Without `noopener` the new tab can call `window.opener` and run scripts in your origin. `noreferrer` strips the Referer header. **Memorize this combo.**
- `<Image>` from `next/image` — automatic optimization, lazy loading, responsive sizes.

### React 19 vs older `<Image>`

`next/image` requires the `width` and `height` props (so layout shift is avoided). The `width=10` Tailwind class overrides it visually. The `width` prop is mostly used to compute the intrinsic aspect ratio.

---

## 2. `About.tsx` — the about section

```tsx
<section id="about" className="flex flex-col md:flex-row relative items-center justify-center min-h-screen w-full h-full">
  <div className="md:absolute w-auto h-auto md:top-[80px] z-[5]">
    <InView triggerOnce={false}>
      {({ inView, ref }) => (
        <motion.div ref={ref} initial="hidden"
                    animate={inView ? "visible" : "hidden"}
                    variants={slideInFromTop}>
          About <span className="...">Me</span>
        </motion.div>
      )}
    </InView>
  </div>
  ...
</section>
```

### The repeating pattern

Every animated block here is wrapped in `<InView>{({ inView, ref }) => ...}</InView>`. Inside, a `<motion.div>` with `initial="hidden"`, `animate={inView ? "visible" : "hidden"}`, `variants={slideInFromXxx(...)}`.

**This is verbose**. In production you'd extract a wrapper:

```tsx
function AnimatedBlock({ variant, children, className }: Props) {
  const { ref, inView } = useInView({ triggerOnce: false });
  return (
    <motion.div ref={ref} initial="hidden" animate={inView ? "visible" : "hidden"}
                variants={variant} className={className}>
      {children}
    </motion.div>
  );
}
```

Then every block becomes one-liner: `<AnimatedBlock variant={slideInFromLeft(0.5)}>...</AnimatedBlock>`.

### Why the developer didn't refactor

Probably copy-pasted across sections during prototyping. **Refactoring this for interview-prep practice is a great exercise.** Try it: pull out `<AnimatedBlock>`, replace all 10+ inline copies, see how much cleaner the section gets.

### Bugs to spot

- `className="... brder my-[20px] ..."` — `brder` is a typo for `border`. Tailwind silently ignores unknown classes, so no error.
- `<img src="/jenin.jpg" alt="profile" width={250} />` — plain `<img>` instead of `next/image`. Means no automatic optimization. Inconsistent with Navbar.

---

## 3. `Skills.tsx` — 4 skill panels

The structure (one panel example, repeated 4×):

```tsx
<div className="w-full lg:w-1/2 h-full">
  <InView triggerOnce={false}>
    {({ inView, ref }) => (
      <motion.div
        ref={ref}
        initial="hidden"
        animate={inView ? "visible" : "hidden"}
        variants={slideInFromLeft(0.5)}
        className="rounded-md text-[white] w-full ... border border-[#7042f88b]"
      >
        <span className="bg-gradient-to-r from-purple-500 to-cyan-500 bg-clip-text text-transparent text-2xl font-bold">
          Frontend
        </span>
        <br />
        <div className="flex flex-row flex-wrap my-4 gap-5">
          {Frontend_skill.map((image, index) => (
            <SkillDataProvider
              key={index}
              src={image.Image}
              width={image.width}
              height={image.height}
              index={index}
            />
          ))}
        </div>
      </motion.div>
    )}
  </InView>
</div>
```

### Gradient text trick

```html
<span class="bg-gradient-to-r from-purple-500 to-cyan-500 bg-clip-text text-transparent">
  Gradient Text
</span>
```

Translates to:
```css
background-image: linear-gradient(to right, #a855f7, #06b6d4);
-webkit-background-clip: text;
background-clip: text;
color: transparent;
```

The trick: make the text color **transparent** so the background shows through, then clip the background to the text shape. Universal CSS pattern — works for any text.

### The 4-panel grid

- Two rows × two columns on `lg+`.
- One column on smaller screens (`flex-col` → `lg:flex-row`).
- Each panel iterates a different array from constants (`Frontend_skill`, `Backend_skill`, etc.).

---

## 4. `Footer.tsx` (and `Projects.tsx`)

Both are commented out in `page.tsx` (`{/* <Projects /> */}`). They exist in code but aren't rendered. Mention this in interviews — being able to spot dead code is real value.

`<Projects>` renders a grid of `<ProjectCard>` instances (we cover `ProjectCard.tsx` in the sub/ guide). The structure is the standard `.map()` over data, wrap each in `<InView>` + `<motion.div>`, render card.

---

## Patterns to take away

1. **Sticky-blur navbar** with `fixed`, `backdrop-blur`, `bg-[color]/[alpha]` — universal recipe.
2. **`hidden md:flex`** for "mobile-first hide / show on desktop".
3. **`target="_blank" rel="noopener noreferrer"`** — burn this into muscle memory.
4. **InView + variants** — common animate-on-scroll pattern (and the optimization opportunity to extract a wrapper).
5. **Gradient text** = `bg-gradient + bg-clip-text + text-transparent`.
6. **`next/image` vs `<img>`** — use `next/image` consistently for production.
