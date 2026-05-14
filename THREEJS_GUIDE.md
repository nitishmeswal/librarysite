# Three.js + React Three Fiber Guide (from zero)

3D in the browser feels mysterious until you've seen a tiny scene work end-to-end. This guide gives you that, plus the React Three Fiber (R3F) translation. Every snippet is taken straight from the projects.

---

## 1. The three components every 3D scene needs

```
Scene  →  Camera  →  Renderer
   ↑
   Meshes (Geometry + Material)
```

- **Scene** — container for everything in 3D space.
- **Camera** — point of view. Perspective (with FOV) or orthographic (flat).
- **Renderer** — WebGL bridge that draws the scene to a `<canvas>`.
- **Mesh** — a 3D object = `Geometry` (the shape) + `Material` (the surface).

---

## 2. Minimal vanilla Three.js scene

The tiniest possible Three.js setup (this is essentially what `Ironhill-section-rebuild/script.js` does, simplified):

```js
import * as THREE from 'three';

// 1. Scene
const scene = new THREE.Scene();

// 2. Camera (FOV, aspect, near, far)
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 5;

// 3. Renderer
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// 4. A mesh
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshBasicMaterial({ color: 0xff8800 });
const cube = new THREE.Mesh(geometry, material);
scene.add(cube);

// 5. Animate
function animate() {
  cube.rotation.x += 0.01;
  cube.rotation.y += 0.01;
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

**Line-by-line**:
- `PerspectiveCamera(fov, aspect, near, far)` — FOV in degrees, aspect ratio, near/far clipping planes (objects outside aren't drawn).
- `camera.position.z = 5` — back up the camera so we can see the cube at origin.
- `renderer.setSize(w, h)` — match canvas size to window.
- `new THREE.BoxGeometry(1, 1, 1)` — a unit cube.
- `MeshBasicMaterial` — no lighting, just flat color (cheap, useful for debug).
- `requestAnimationFrame(animate)` — browser tells us when to draw the next frame (~60fps).

---

## 3. Orthographic camera for full-screen shaders

The Ironhill rebuild uses an **orthographic** camera because it's drawing a fullscreen quad with a custom shader (no perspective needed):

```js
const camera = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
```

Bounds: left=-1, right=1, top=1, bottom=-1, near=0, far=1. Then a `PlaneGeometry(2, 2)` fills the whole view exactly.

---

## 4. Materials you'll see in this repo

| Material | Use case |
| --- | --- |
| `MeshBasicMaterial` | Flat, no lighting. Cheapest. |
| `MeshStandardMaterial` | PBR (physically based). Reacts to lights. Has `roughness`, `metalness`. |
| `MeshLambertMaterial` | Diffuse only, faster than Standard. |
| `ShaderMaterial` | You write your own GLSL vertex+fragment shaders. |
| `PointsMaterial` (`PointMaterial` in drei) | For star/particle fields. |

From `Fizzi-3D-Website/src/components/SodaCan.tsx`:

```ts
const metalMaterial = new THREE.MeshStandardMaterial({
  roughness: 0.3,
  metalness: 1,
  color: "#bbbbbb",
});
```

- `metalness: 1` — looks metallic (reflects environment).
- `roughness: 0.3` — mostly shiny.

---

## 5. Textures (image maps)

From `Fizzi-3D-Website/src/components/SodaCan.tsx`:

```ts
const labels = useTexture(flavorTextures);
labels.blackCherry.flipY = false;
// ...
<meshStandardMaterial roughness={0.15} metalness={0.7} map={label} />
```

- `useTexture({ key: '/url.png' })` (from `@react-three/drei`) loads PNGs and returns Three textures, keyed.
- `flipY = false` — by default, Three flips Y for image textures (browsers store images top-down vs WebGL bottom-up). For glTF assets exported with UVs already flipped, you need `false`.
- `map={label}` — the texture goes on the `map` slot of the material (the diffuse/color map).

---

## 6. Lights

```jsx
<ambientLight intensity={2} color="#9DDEFA" />
<directionalLight intensity={6} position={[0, 1, 1]} />
<pointLight intensity={30} color="#8C0413" decay={0.6} />
```

- **ambient** — uniform light from all directions; no shadows.
- **directional** — like sunlight; parallel rays from one direction.
- **point** — bulb-style; falls off with distance.

From `Fizzi-3D-Website/src/slices/SkyDive/Scene.tsx`:

```tsx
<ambientLight intensity={2} color="#9DDEFA" />
<Environment files="/hdr/field.hdr" environmentIntensity={1.5} />
```

`Environment` (from drei) loads an HDR image as the environment map — reflective materials show that image in their reflections without needing a real skybox.

---

## 7. Vertex and Fragment shaders (GLSL)

Shaders are tiny programs that run on the GPU.

- **Vertex shader**: runs once per vertex. Decides where the vertex appears on screen.
- **Fragment shader**: runs once per pixel of every triangle face. Decides the pixel's color.

From `Ironhill-section-rebuild/script.js` — vertex shader passes UVs through, fragment shader does a noise-based dissolve:

```glsl
// vertex
varying vec2 vUv;
void main() {
  vUv = uv;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

- `uv` — built-in attribute; the 2D texture coords of this vertex.
- `vUv = uv;` — stash for the fragment shader.
- `gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);` — standard transform. Takes object-space `position` → screen space.

```glsl
// fragment (excerpt)
uniform float uProgress;
uniform vec2 uResolution;
uniform vec3 uColor;
uniform float uSpread;
varying vec2 vUv;

float Hash(vec2 p) { /* pseudo-random */ }
float noise(vec2 p) { /* smooth random */ }
float fbm(vec2 p) { /* fractal noise */ }

void main() {
  vec2 uv = vUv;
  float aspect = uResolution.x / uResolution.y;
  vec2 centeredUv = (uv - 0.5) * vec2(aspect, 1.0);

  float dissolveEdge = uv.y - uProgress * 1.2;
  float noiseValue = fbm(centeredUv * 15.0);
  float d = dissolveEdge + noiseValue * uSpread;

  float pixelSize = 1.0 / uResolution.y;
  float alpha = 1.0 - smoothstep(-pixelSize, pixelSize, d);

  gl_FragColor = vec4(uColor, alpha);
}
```

**What this does**: as `uProgress` increases from 0 → 1, a noise-perturbed horizontal edge sweeps up the screen, and pixels below the edge become transparent. Result: an animated "dissolve up" overlay. `uProgress` is driven from JS by Lenis scroll progress.

You set uniforms from JS:

```js
const material = new THREE.ShaderMaterial({
  vertexShader,
  fragmentShader,
  uniforms: {
    uProgress: { value: 0 },
    uResolution: { value: new THREE.Vector2(w, h) },
    uColor: { value: new THREE.Vector3(r, g, b) },
    uSpread: { value: 0.5 },
  },
  transparent: true,
});

material.uniforms.uProgress.value = scrollProgress;
```

---

## 8. React Three Fiber (R3F) — Three.js as JSX

R3F lets you write Three scenes declaratively:

```jsx
import { Canvas } from "@react-three/fiber";

<Canvas camera={{ fov: 30 }}>
  <ambientLight intensity={1} />
  <mesh position={[0, 0, 0]} rotation={[0, Math.PI, 0]}>
    <boxGeometry args={[1, 1, 1]} />
    <meshStandardMaterial color="#bbb" metalness={1} roughness={0.3} />
  </mesh>
</Canvas>
```

**Mapping**:
- Lowercase `<mesh>`, `<group>`, `<boxGeometry>` etc. → R3F intrinsic elements for the corresponding `THREE.Mesh`, `THREE.Group`, `THREE.BoxGeometry`.
- `args={[...]}` → constructor arguments.
- `position={[x,y,z]}` → shortcut for `mesh.position.set(x,y,z)`.

---

## 9. `useFrame` — every render frame

```jsx
import { useFrame } from "@react-three/fiber";

function Spinner() {
  const ref = useRef();
  useFrame((state, delta) => {
    ref.current.rotation.x += delta;
  });
  return <mesh ref={ref}>...</mesh>;
}
```

- `delta` — seconds since last frame.
- `state` — has `state.clock`, `state.mouse`, `state.camera`, ...

From `SpacePortfolio/components/main/StarBackground.tsx`:

```tsx
useFrame((state, delta) => {
  ref.current.rotation.x -= delta / 10;
  ref.current.rotation.y -= delta / 15;
});
```

The star field rotates slowly, time-step independent of framerate (because we use `delta`).

---

## 10. Drei — useful R3F helpers

Drei (`@react-three/drei`) ships a ton of helpers. Ones used in this repo:

| Helper | What it does |
| --- | --- |
| `useGLTF` | Loads `.gltf`/`.glb` 3D models. Returns `{ nodes, materials, animations, scene }`. |
| `useTexture` | Loads image textures (returns Three `Texture`). |
| `useAnimations` | Plays animations stored in a glTF file. |
| `Float` | Wraps children in a slowly bobbing/rotating group. |
| `Environment` | Loads HDR as environment map (reflections). |
| `OrbitControls` | Mouse-controlled camera (drag to orbit, wheel to zoom). |
| `Text` | 3D text mesh (loads a font, renders true 3D text). |
| `Cloud`, `Clouds` | Volumetric cloud meshes. |
| `View`, `View.Port` | Render multiple "views" into one fullscreen canvas (Fizzi pattern). |
| `Loader` | Progress UI for `<Suspense>` while assets load. |
| `Points` (with PointMaterial) | Particle systems (StarBackground). |

---

## 11. Loading a glTF model

From `Fizzi-3D-Website/src/components/SodaCan.tsx`:

```ts
import { useGLTF, useTexture } from "@react-three/drei";

useGLTF.preload("/Soda-can.gltf");

export function SodaCan({ flavor = "blackCherry", scale = 2, ...props }) {
  const { nodes } = useGLTF("/Soda-can.gltf");
  // nodes.cylinder is the can body, nodes.Tab is the pull-tab, etc.
  return (
    <group {...props} dispose={null} scale={scale} rotation={[0, -Math.PI, 0]}>
      <mesh geometry={(nodes.cylinder as THREE.Mesh).geometry} material={metalMaterial} />
      {/* ... */}
    </group>
  );
}
```

- `useGLTF.preload(...)` — start fetching the model immediately so the first render isn't blocked.
- `useGLTF(...)` — suspends the component until the file loads. Place inside a `<Suspense>` boundary.
- `nodes.X` — named meshes inside the glTF. Each has its own geometry; you reuse them with whatever material you want.

---

## 12. Instanced meshes (lots of identical objects)

If you have 300 bubbles, don't create 300 meshes — that's 300 draw calls. Use `InstancedMesh`: one mesh, one geometry, one material, many transforms.

From `Fizzi-3D-Website/src/slices/Hero/Bubbles.tsx`:

```ts
const o = new THREE.Object3D();
// ...
useEffect(() => {
  const mesh = meshRef.current;
  for (let i = 0; i < count; i++) {
    o.position.set(rand(), rand(), rand());
    o.updateMatrix();
    mesh.setMatrixAt(i, o.matrix);
  }
  mesh.instanceMatrix.needsUpdate = true;
}, [count]);

return (
  <instancedMesh ref={meshRef} args={[undefined, undefined, count]} material={material} geometry={geometry} />
);
```

- `Object3D` is just a transform helper. We compose a position → update its matrix → copy into the instanced mesh at index `i`.
- `instanceMatrix.needsUpdate = true` tells the GPU to re-upload the matrix buffer.

This pattern is **the** performance trick for particle systems.

---

## 13. Camera animation tied to mouse

From `Personal-Portfolio/src/sections/Hero.jsx`:

```jsx
import { Canvas, useFrame } from '@react-three/fiber';
import { easing } from 'maath';

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

- `state.mouse.x`, `state.mouse.y` are -1..1, normalized.
- `easing.damp3` is a critically-damped exponential approach — buttery smooth, framerate-independent.
- The camera leans slightly toward where the mouse is. Tiny detail, huge feel.

---

## 14. R3F + GSAP mixed

GSAP can animate any object. When R3F gives you a `ref` to a `<group>`, you get a Three.js `Group`. Its `.position`, `.rotation`, `.scale` are vectors with numeric `.x`, `.y`, `.z`. GSAP can tween them.

From `Fizzi-3D-Website/src/slices/Hero/Scene.tsx`:

```ts
gsap.set(can1Ref.current.position, { x: -1.5 });
gsap.set(can1Ref.current.rotation, { z: -0.5 });

// later: scroll-driven
.to(can1Ref.current.position, { x: -0.2, y: -0.7, z: -2 }, 0)
.to(can1Ref.current.rotation, { z: 0.3 }, 0)
```

This is the entire glue between "scroll" and "soda can flies into formation". GSAP changes `position.x/y/z` over time. R3F renders the new transform every `useFrame`. No magic.

---

## 15. View portals (Fizzi pattern)

Spinning up multiple `<Canvas>` instances is expensive. Drei's `View` lets you have **one** `<Canvas>` at the root of the app, and several "view" rectangles in DOM components that each render their own scene into that single canvas.

```tsx
// root layout
<Canvas>
  <Suspense fallback={null}>
    <View.Port />
  </Suspense>
</Canvas>

// somewhere in the page
<View className="hero-scene ..."><Scene /></View>
<View className="alternating-text-view ..."><Scene /></View>
```

Each `<View>` is a DOM `<div>` positioned wherever you want. Drei uses scissoring to draw the matching scene only inside that rectangle. One canvas → one WebGL context → many "scenes" sharing it.

---

## 16. Build-from-scratch checklists

### A. A spinning cube in vanilla Three.js
See section 2 above. You should be able to write that whole snippet from memory.

### B. A spinning cube in R3F

```jsx
import { Canvas, useFrame } from "@react-three/fiber";
import { useRef } from "react";

function Cube() {
  const ref = useRef();
  useFrame((_, dt) => {
    ref.current.rotation.x += dt;
    ref.current.rotation.y += dt;
  });
  return (
    <mesh ref={ref}>
      <boxGeometry args={[1, 1, 1]} />
      <meshStandardMaterial color="orange" />
    </mesh>
  );
}

export default function App() {
  return (
    <Canvas camera={{ position: [0, 0, 5] }}>
      <ambientLight />
      <directionalLight position={[2, 2, 2]} />
      <Cube />
    </Canvas>
  );
}
```

### C. Mouse-tracking camera (the Personal-Portfolio trick)

```jsx
function Rig() {
  useFrame((state, dt) => {
    easing.damp3(state.camera.position, [state.mouse.x, state.mouse.y, 3], 0.5, dt);
  });
  return null;
}
```

### D. Star field

```jsx
import { Points, PointMaterial } from "@react-three/drei";
import * as random from "maath/random";

function Stars() {
  const ref = useRef();
  const [sphere] = useState(() => random.inSphere(new Float32Array(5000), { radius: 1.2 }));
  useFrame((_, dt) => {
    ref.current.rotation.x -= dt / 10;
    ref.current.rotation.y -= dt / 15;
  });
  return (
    <Points ref={ref} positions={sphere} stride={3}>
      <PointMaterial color="#fff" size={0.005} sizeAttenuation depthWrite={false} />
    </Points>
  );
}
```

---

## 17. Mental model summary

- A scene is: a **graph of `Group`s and `Mesh`es**, rendered by a **renderer**, through a **camera**.
- `Mesh = Geometry + Material`. Materials can be PBR (`MeshStandardMaterial`) or custom shader (`ShaderMaterial`).
- `useFrame` is your `requestAnimationFrame`. `delta` keeps motion framerate-independent.
- For lots of identical objects → `InstancedMesh`.
- For loading models → `useGLTF` + drei + `<Suspense>`.
- For reflective realism → `Environment` HDR.
- GSAP animates Three transforms exactly like it animates DOM transforms — they're just `.x/.y/.z`.

If you can explain "what does a fragment shader do?" and "how does R3F connect to Three.js?", that already puts you ahead of 99% of SDE 1 candidates on the 3D front.
