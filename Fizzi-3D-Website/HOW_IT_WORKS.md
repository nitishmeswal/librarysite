# Fizzi-3D-Website/ — How It Works

The **most ambitious project in this repo**. A Next.js 14 + React Three Fiber + GSAP + Prismic site for a fictional soda brand. 3D soda cans float in shared scenes, scroll-driven rotations, instanced bubble particles, sky-diving 3D text — this is portfolio-level work. **Master this and you've covered every advanced frontend topic that comes up in an interview**.

## Stack
- **Next.js 14** (App Router) + **React 18** + **TypeScript**.
- **React Three Fiber** (`@react-three/fiber`) — Three.js inside React.
- **drei helpers** (`@react-three/drei`) — `View`, `Float`, `Environment`, `Clouds`, `Cloud`, `Text`, `useGLTF`, `useTexture`, `Center`, `OrbitControls`.
- **GSAP** + ScrollTrigger + `useGSAP`.
- **Zustand** for a global "is ready" state.
- **Prismic** (headless CMS) + `SliceZone` + `PrismicNextImage` + `PrismicRichText`.
- **Tailwind CSS v3**.

## Folder map
```
Fizzi-3D-Website/
├── customtypes/          # Prismic content type definitions
├── public/               # Soda-can.gltf, label PNGs, HDRs, fonts, etc.
├── src/
│   ├── app/              # Next.js app router (layout.tsx, page.tsx, etc.)
│   │   ├── layout.tsx    # Root layout — Header + ViewCanvas + Footer
│   │   └── page.tsx      # Home page — Prismic SliceZone
│   ├── components/       # Header, Footer, ViewCanvas, FloatingCan, SodaCan, etc.
│   ├── hooks/            # useStore (Zustand), useMediaQuery (SSR-safe)
│   ├── slices/           # One folder per slice: Hero, Carousel, SkyDive, AlternatingText, BigText
│   └── prismicio.ts      # Prismic client config
├── next.config.mjs
├── tailwind.config.js
└── prismicio-types.d.ts
```

---

## 1. The core architectural pattern — **shared R3F Canvas with View portals**

This is the most important architectural decision in the project. Read it twice.

### The problem

You want 3D scenes scattered across the page — one in the Hero, one in the Carousel, one in the SkyDive section, etc. The naive approach: put a `<Canvas>` in each section. But:
- Each `<Canvas>` initializes its own WebGL context. Browsers limit WebGL contexts to ~16. Performance dies.
- Each context has separate state. You can't easily share lights, assets, environment maps.

### The solution: ONE `<Canvas>`, MULTIPLE `<View>` portals

```tsx
// app/layout.tsx
<main>
  {children}
  <ViewCanvas />  {/* Fixed-position single Canvas */}
</main>
```

```tsx
// components/ViewCanvas.tsx
<Canvas style={{ position: "fixed", top: 0, left: "50%", ... }}>
  <Suspense fallback={null}>
    <View.Port />
  </Suspense>
</Canvas>
```

The `<Canvas>` is **fixed positioned**, full screen, behind everything. It NEVER moves.

In each section, you place a `<View>` from drei:

```tsx
// slices/Hero/index.tsx
<View className="hero-scene pointer-events-none sticky top-0 z-50 -mt-[100vh] hidden h-screen w-screen md:block">
  <Scene />
  <Bubbles count={300} speed={2} />
</View>
```

The `<View>` is a **placeholder div** in the DOM. The drei library reads the View's bounding rect on every frame and uses a `scissor` to render the contents of the `<View>` ONLY in that rectangle. So you get the illusion of multiple separate 3D scenes, but in reality they all live in ONE WebGL context.

### Why this is brilliant

- **One context, infinite scenes** — you can have 10 `<View>` instances scattered across the page with no performance penalty.
- **Shared state** — lights, environment, post-processing apply to all views.
- **Scroll-friendly** — since the Canvas is fixed, scroll only moves the DOM, not the WebGL. The drei View handles re-positioning the scene's rendered output as the page scrolls.

### The `View.Port` pattern

`<View.Port />` is a single React node placed inside the Canvas. It collects all `<View>` portals across the app and renders them in turn. Like React Portals, but for Three.js scenes.

---

## 2. Entry — `app/layout.tsx`

```tsx
const alpino = localFont({
  src: "../../public/fonts/Alpino-Variable.woff2",
  display: "swap",
  weight: "100 900",
  variable: "--font-alpino",
});

export default function RootLayout({ children }) {
  return (
    <html lang="en" className={alpino.variable}>
      <body className="overflow-x-hidden bg-yellow-300">
        <Header />
        <main>
          {children}
          <ViewCanvas />
        </main>
        <Footer />
      </body>
      <PrismicPreview repositoryName={repositoryName} />
    </html>
  );
}
```

### `next/font/local` for self-hosted fonts

```tsx
const alpino = localFont({
  src: "../../public/fonts/Alpino-Variable.woff2",
  display: "swap",
  weight: "100 900",
  variable: "--font-alpino",
});
```

- **`src`** — path to the font file (relative to the layout).
- **`display: "swap"`** — show fallback font while custom font loads, then swap.
- **`weight: "100 900"`** — variable font weight range.
- **`variable: "--font-alpino"`** — exposes the font as a CSS variable.

Then `<html className={alpino.variable}>` sets the CSS variable on `<html>`, and Tailwind config maps `font-alpino` to `var(--font-alpino)`.

### `<PrismicPreview>`

For Prismic content editors — lets them preview drafts. Doesn't affect end-users; only fires when in preview mode.

### Why `overflow-x-hidden`?

Because some 3D scenes use `-translate-x-1/2` etc. that might push elements outside the viewport. This prevents a horizontal scrollbar.

---

## 3. Entry — `app/page.tsx` (Prismic + SliceZone)

```tsx
export async function generateMetadata(): Promise<Metadata> {
  const client = createClient();
  const home = await client.getByUID("page", "home");

  return {
    title: prismic.asText(home.data.title),
    description: home.data.meta_description,
    openGraph: { title: home.data.meta_title ?? undefined, images: [{ url: home.data.meta_image.url ?? "" }] },
  };
}

export default async function Index() {
  const client = createClient();
  const home = await client.getByUID("page", "home");

  return <SliceZone slices={home.data.slices} components={components} />;
}
```

### `async` server components

Both functions are `async`. Next.js 14 (App Router) lets server components fetch data directly with `await`. The page's HTML is rendered server-side with the data already filled in.

### `generateMetadata` for SEO

Next.js calls this function to generate the `<title>`, `<meta>` tags, OpenGraph tags. The result is included in the SSR'd HTML. Critical for SEO.

### `SliceZone` from Prismic

`SliceZone` is Prismic's "composer" — it takes an array of "slices" (each slice is a content block with a `slice_type` like `"hero"` or `"carousel"`) and a map of `components` (which component to render for each `slice_type`).

```tsx
// slices/index.ts
export const components = {
  hero: HeroSlice,
  carousel: CarouselSlice,
  sky_dive: SkyDiveSlice,
  // ...
};
```

Content editors arrange slices in the Prismic dashboard. The frontend picks the right component for each slice and passes the slice data. **Pure CMS-driven UI**.

---

## 4. Global state — `hooks/useStore.ts` (Zustand)

```ts
import { create } from "zustand";

interface State {
  ready: boolean;
  isReady: () => void;
}

export const useStore = create<State>((set) => ({
  ready: false,
  isReady: () => set({ ready: true }),
}));
```

### Why Zustand?

Tiny state library. ~1kb. No reducers, no actions, no `Provider`. Just `create()`, define state and setters, and use the returned hook anywhere.

```tsx
// Read state:
const ready = useStore((state) => state.ready);

// Update state:
const isReady = useStore((state) => state.isReady);
isReady();  // calls the setter
```

The selector pattern (`(state) => state.ready`) ensures components only re-render when their selected slice changes — automatic memoization.

### What's it used for here?

- 3D scene calls `isReady()` once it's set up.
- Other components wait for `ready === true` before running their entry animations.
- Pattern: "wait for 3D scene to mount before fading in text".

### Build Zustand from scratch (interview-prep)

```ts
function create(initializer) {
  let state;
  const listeners = new Set();
  const setState = (partial) => {
    state = { ...state, ...(typeof partial === 'function' ? partial(state) : partial) };
    listeners.forEach(l => l());
  };
  const getState = () => state;
  state = initializer(setState);
  return function useStore(selector) {
    const [, force] = useReducer(c => c + 1, 0);
    useEffect(() => listeners.add(force) && (() => listeners.delete(force)), []);
    return selector(getState());
  };
}
```

20 lines. That's all Zustand really is. Worth understanding because state management questions come up in interviews.

---

## 5. SSR-safe media query — `hooks/useMediaQuery.ts`

```ts
import { useCallback, useSyncExternalStore } from "react";

export function useMediaQuery(query: string, serverFallback: boolean): boolean {
  const subscribe = useCallback(
    (onStoreChange: () => void) => {
      const mediaQueryList = matchMedia(query);
      mediaQueryList.addEventListener("change", onStoreChange);
      return () => mediaQueryList.removeEventListener("change", onStoreChange);
    },
    [query],
  );

  return useSyncExternalStore(
    subscribe,
    () => matchMedia(query).matches,
    () => serverFallback,
  );
}
```

### `useSyncExternalStore` — the React 18 hook

This is the "official React way" to subscribe to external (non-React) state — DOM events, browser APIs, third-party stores. It takes:
1. **`subscribe(onChange)`** — function that hooks up a listener and returns a cleanup. Called when component mounts; cleanup is called on unmount.
2. **`getSnapshot()`** — returns the current value (client side).
3. **`getServerSnapshot()`** — returns the SSR value (server side).

The hook automatically re-renders when `onChange` is called, reading the new value via `getSnapshot()`.

### Why bother with this instead of a plain useEffect?

- **SSR-safe** — `matchMedia` doesn't exist on the server. Plain `useState(() => matchMedia(...).matches)` crashes during SSR. `useSyncExternalStore` uses `getServerSnapshot` instead.
- **No flicker on first render** — server renders with `serverFallback`, client hydrates with actual value.
- **React Concurrent-safe** — handles tearing in concurrent rendering.

### Compare to `react-responsive`'s `useMediaQuery`

`react-responsive` is simpler but has hydration issues in SSR. This custom hook is the "right" way for Next.js.

### Interview gold

If asked "how do you make a media-query React hook?" — this is the perfect answer. Mention SSR, concurrent rendering, no flicker.

---

## 6. The 3D building blocks — `components/SodaCan.tsx` + `FloatingCan.tsx`

### `SodaCan.tsx`

```tsx
useGLTF.preload("/Soda-can.gltf");

const flavorTextures = {
  lemonLime: "/labels/lemon-lime.png",
  grape: "/labels/grape.png",
  blackCherry: "/labels/cherry.png",
  strawberryLemonade: "/labels/strawberry.png",
  watermelon: "/labels/watermelon.png",
};

const metalMaterial = new THREE.MeshStandardMaterial({
  roughness: 0.3,
  metalness: 1,
  color: "#bbbbbb",
});

export function SodaCan({ flavor = "blackCherry", scale = 2, ...props }) {
  const { nodes } = useGLTF("/Soda-can.gltf");
  const labels = useTexture(flavorTextures);

  labels.strawberryLemonade.flipY = false;
  labels.blackCherry.flipY = false;
  // ... etc

  const label = labels[flavor];

  return (
    <group {...props} dispose={null} scale={scale} rotation={[0, -Math.PI, 0]}>
      <mesh
        castShadow receiveShadow
        geometry={(nodes.cylinder as THREE.Mesh).geometry}
        material={metalMaterial}
      />
      <mesh
        castShadow receiveShadow
        geometry={(nodes.cylinder_1 as THREE.Mesh).geometry}
      >
        <meshStandardMaterial roughness={0.15} metalness={0.7} map={label} />
      </mesh>
      <mesh
        castShadow receiveShadow
        geometry={(nodes.Tab as THREE.Mesh).geometry}
        material={metalMaterial}
      />
    </group>
  );
}
```

### `useGLTF.preload()`

```tsx
useGLTF.preload("/Soda-can.gltf");
```

Called at module load (outside the component). Starts fetching the GLTF file before any component renders. Critical for perceived load time — the model is already loading when the user navigates to the page.

### `useGLTF("/Soda-can.gltf")`

A drei hook that loads a GLTF model and returns:
- `nodes` — a map of named objects in the model (`cylinder`, `cylinder_1`, `Tab`).
- `materials` — a map of named materials.
- `animations` — list of animation clips.

GLTF files are the standard 3D model format for web (like .glb is its binary form).

### `useTexture` for loading image textures

```tsx
const labels = useTexture(flavorTextures);
```

Loads all five label images in parallel. Returns an object keyed by the input keys. The plugin uses Suspense — your component suspends until textures are loaded.

### `flipY = false`

GLTF UV coordinates have a different orientation than Three.js's default. Setting `flipY = false` corrects upside-down labels. Common gotcha.

### The three meshes

Each `<mesh>` is one of the GLTF's named geometries:
1. **`cylinder`** — outer metal body (gray, shiny).
2. **`cylinder_1`** — paper label (colored, low metalness).
3. **`Tab`** — pull tab on top (gray).

The label mesh uses `<meshStandardMaterial map={label}>` to apply the flavor texture as the diffuse map.

### `dispose={null}` on the group

By default, R3F auto-disposes geometries when components unmount. For preloaded GLTF assets, we don't want this — we want to keep them in memory because multiple components share them. `dispose={null}` opts out.

### `FloatingCan.tsx`

```tsx
const FloatingCan = forwardRef<Group, FloatingCanProps>(
  ({ flavor, floatSpeed = 1.5, rotationIntensity = 1, floatIntensity = 1, floatingRange = [-0.1, 0.1], children, ...props }, ref) => {
    return (
      <group ref={ref} {...props}>
        <Float speed={floatSpeed} rotationIntensity={rotationIntensity} floatIntensity={floatIntensity} floatingRange={floatingRange}>
          {children}
          <SodaCan flavor={flavor} />
        </Float>
      </group>
    );
  },
);
```

### `<Float>` from drei

Auto-applies a sinusoidal floating animation to children. Floats them up/down + slightly rotates. Saves you writing a custom `useFrame` to do this manually.

- `speed` — how fast.
- `rotationIntensity` — how much it rotates.
- `floatIntensity` — overall multiplier for translation.
- `floatingRange` — y-axis range in world units.

### `forwardRef` for passing refs to 3D groups

```tsx
const FloatingCan = forwardRef<Group, FloatingCanProps>((props, ref) => {
  return <group ref={ref} {...props}>...</group>;
});
```

The component itself wraps the can in a `<group>`, and the parent's ref points to that group. GSAP then animates the group's position/rotation. **Standard React pattern for letting parents grab refs to internal DOM/Three nodes**.

### `displayName` for devtools

```tsx
FloatingCan.displayName = "FloatingCan";
```

Without this, React DevTools shows `ForwardRef` instead of `FloatingCan`. Set this on every `forwardRef` component.

---

## 7. The instanced bubbles — `slices/Hero/Bubbles.tsx`

Stunning effect: 300 transparent bubbles floating upward, recycling at the top. **This is interview gold for "performance".**

```tsx
const o = new THREE.Object3D();

export function Bubbles({ count = 300, speed = 5, bubbleSize = 0.05, opacity = 0.5, repeat = true }) {
  const meshRef = useRef<THREE.InstancedMesh>(null);
  const bubbleSpeed = useRef(new Float32Array(count));
  const minSpeed = speed * 0.001;
  const maxSpeed = speed * 0.005;

  const geometry = new THREE.SphereGeometry(bubbleSize, 16, 16);
  const material = new THREE.MeshStandardMaterial({ transparent: true, opacity });

  useEffect(() => {
    const mesh = meshRef.current;
    if (!mesh) return;

    for (let i = 0; i < count; i++) {
      o.position.set(
        gsap.utils.random(-4, 4),
        gsap.utils.random(-4, 4),
        gsap.utils.random(-4, 4),
      );
      o.updateMatrix();
      mesh.setMatrixAt(i, o.matrix);
      bubbleSpeed.current[i] = gsap.utils.random(minSpeed, maxSpeed);
    }
    mesh.instanceMatrix.needsUpdate = true;
    return () => {
      mesh.geometry.dispose();
      (mesh.material as THREE.Material).dispose();
    };
  }, [count, minSpeed, maxSpeed]);

  useFrame(() => {
    if (!meshRef.current) return;
    material.color = new THREE.Color(document.body.style.backgroundColor);

    for (let i = 0; i < count; i++) {
      meshRef.current.getMatrixAt(i, o.matrix);
      o.position.setFromMatrixPosition(o.matrix);
      o.position.y += bubbleSpeed.current[i];

      if (o.position.y > 4 && repeat) {
        o.position.y = -2;
        o.position.x = gsap.utils.random(-4, 4);
        o.position.z = gsap.utils.random(0, 8);
      }

      o.updateMatrix();
      meshRef.current.setMatrixAt(i, o.matrix);
    }
    meshRef.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <instancedMesh
      ref={meshRef}
      args={[undefined, undefined, count]}
      position={[0, 0, 0]}
      material={material}
      geometry={geometry}
    />
  );
}
```

### Why **instanced** mesh?

If you rendered 300 separate `<mesh>`es with the same geometry, Three.js would issue 300 draw calls. Too slow.

`InstancedMesh` renders **one geometry, N instances**, in a SINGLE draw call. Each instance can have its own transform (position, rotation, scale) stored in an instance matrix array on the GPU.

### How instances are positioned

The `Object3D o` (a "dummy" helper outside the component) is reused for every bubble:

```ts
o.position.set(x, y, z);  // set its position
o.updateMatrix();          // compute its 4x4 transform matrix
mesh.setMatrixAt(i, o.matrix);  // write that matrix into instance i
```

The dummy is just a calculator — its transforms are extracted and written into the InstancedMesh's matrix array.

After all setMatrixAt calls, `mesh.instanceMatrix.needsUpdate = true` tells GPU to re-upload the matrix array.

### `useFrame` for per-frame updates

```tsx
useFrame(() => { /* runs ~60 times per second */ });
```

R3F's hook for the requestAnimationFrame loop. Replaces `requestAnimationFrame` calls.

### The per-bubble update

For each bubble:
1. Read its current matrix (`getMatrixAt`).
2. Extract position (`setFromMatrixPosition`).
3. Move it up by `bubbleSpeed[i]`.
4. If it's above the screen, recycle to the bottom.
5. Write the new matrix back (`setMatrixAt`).

### The bubble-color trick

```ts
material.color = new THREE.Color(document.body.style.backgroundColor);
```

Reads the current body background color (which is animated by GSAP elsewhere) and applies it to all bubbles. So bubbles match the page color → they "blend" with the background, looking subtle and elegant.

### Build instanced particles from scratch (interview-ready)

```tsx
function Particles({ count = 100 }) {
  const mesh = useRef<THREE.InstancedMesh>(null);
  const dummy = useMemo(() => new THREE.Object3D(), []);

  useEffect(() => {
    if (!mesh.current) return;
    for (let i = 0; i < count; i++) {
      dummy.position.set(Math.random()*10-5, Math.random()*10-5, Math.random()*10-5);
      dummy.updateMatrix();
      mesh.current.setMatrixAt(i, dummy.matrix);
    }
    mesh.current.instanceMatrix.needsUpdate = true;
  }, [count]);

  return (
    <instancedMesh ref={mesh} args={[null, null, count]}>
      <sphereGeometry args={[0.1]} />
      <meshStandardMaterial color="white" />
    </instancedMesh>
  );
}
```

20 lines for 1000s of particles in one draw call.

---

## 8. The slice patterns — see `slices/HOW_IT_WORKS.md`

Each slice in `src/slices/<SliceName>/` contains:
- `index.tsx` — the React component for the slice.
- `Scene.tsx` (if 3D) — the R3F scene rendered inside a `<View>`.
- `mocks.json` — sample data for local development.
- `model.json` — Prismic content-type definition.
- `screenshot-default.png` — preview for Prismic dashboard.

We'll cover each in detail in `slices/HOW_IT_WORKS.md`.

---

## 9. Build-from-scratch checklists

### Build a shared R3F Canvas (10 minutes)

```tsx
// In root layout:
<main>{children}<ViewCanvas /></main>

// ViewCanvas.tsx:
import { Canvas } from "@react-three/fiber";
import { View } from "@react-three/drei";

export default function ViewCanvas() {
  return (
    <Canvas style={{ position: "fixed", inset: 0, zIndex: -1, pointerEvents: "none" }}>
      <View.Port />
    </Canvas>
  );
}

// In any section:
import { View } from "@react-three/drei";

<View className="h-screen w-screen">
  <ambientLight />
  <mesh><boxGeometry /><meshStandardMaterial color="red" /></mesh>
</View>
```

### Build floating 3D cans (15 minutes)

```tsx
import { Canvas } from "@react-three/fiber";
import { Float, useGLTF } from "@react-three/drei";

function Can() {
  const { nodes } = useGLTF("/can.gltf");
  return <mesh geometry={nodes.body.geometry} />;
}

<Canvas>
  <Float speed={1.5} rotationIntensity={1} floatIntensity={1}>
    <Can />
  </Float>
</Canvas>
```

### Build scroll-driven 3D rotation (15 minutes)

```tsx
const canRef = useRef<Group>(null);

useGSAP(() => {
  gsap.to(canRef.current!.rotation, {
    y: Math.PI * 2,
    scrollTrigger: { trigger: ".section", start: "top top", end: "bottom top", scrub: true },
  });
});

return <group ref={canRef}><Can /></group>;
```

---

## What this project teaches you

- **Shared R3F Canvas with View portals** — the architectural breakthrough for multiple 3D scenes on one page.
- **`useGLTF` + `useTexture`** — drei's resource hooks for 3D content.
- **`<Float>` for auto-animating subtle motion** — saves writing `useFrame` for routine effects.
- **`InstancedMesh` for 1000s of particles** in one draw call.
- **`useFrame` for per-frame logic** in R3F.
- **`useSyncExternalStore`** for SSR-safe browser APIs (media queries, network status, online/offline).
- **Zustand** for global state with no Provider.
- **Prismic + SliceZone** for CMS-driven UI.
- **`forwardRef` for passing refs to 3D groups** — animations target group transforms.
- **`useGSAP({ dependencies: [...] })`** to re-run animations on state change.
- **`next/font/local`** for self-hosted fonts.
- **Scroll-triggered body background color changes** with `gsap.to('body', { backgroundColor })`.

### Deeper guides

- [src/slices/HOW_IT_WORKS.md](./src/slices/HOW_IT_WORKS.md) — every slice broken down.
- [src/components/HOW_IT_WORKS.md](./src/components/HOW_IT_WORKS.md) — the building blocks (FloatingCan, SodaCan, Bounded, TextSplitter, etc.).
- [src/hooks/HOW_IT_WORKS.md](./src/hooks/HOW_IT_WORKS.md) — Zustand store + media query hook.
- [src/app/HOW_IT_WORKS.md](./src/app/HOW_IT_WORKS.md) — Next.js App Router setup.
