# B1 Research: Three.js Ecosystem & Advanced Features

> Phase B1 raw masterplan research — ecosystem landscape mapping.
> Generated: 2026-03-20 | Sources: Official Three.js docs (r160+), pmndrs ecosystem, ThatOpen, Rapier

---

## 1. Lighting & Shadows

### 1.1 Light Types

Three.js provides seven light classes, all extending the base `Light` class (which extends `Object3D`).

| Light | Shadow Support | Use Case | Performance |
|-------|---------------|----------|-------------|
| `AmbientLight` | No | Uniform fill light from all directions; prevents pure-black areas | Cheapest — no per-fragment calc |
| `HemisphereLight` | No | Two-color gradient (sky color + ground color); outdoor ambient | Cheap — single gradient |
| `DirectionalLight` | Yes | Parallel rays simulating sunlight; orthographic shadow camera | Moderate — one shadow map |
| `PointLight` | Yes | Omnidirectional from a point (light bulb); cubemap shadow | Expensive — six shadow maps |
| `SpotLight` | Yes | Cone-shaped beam with angle + penumbra; perspective shadow camera | Moderate — one shadow map |
| `RectAreaLight` | No | Rectangular emitter for photometric area lighting; physically accurate | Expensive — no native shadow support |
| `IESSpotLight` | Yes | SpotLight with IES photometric profile data | Same as SpotLight + profile lookup |

**Constructor patterns:**

```javascript
// All lights take (color, intensity)
const ambient = new THREE.AmbientLight(0xffffff, 0.5);
const directional = new THREE.DirectionalLight(0xffffff, 1.0);
const point = new THREE.PointLight(0xffffff, 1.0, 100, 2); // color, intensity, distance, decay
const spot = new THREE.SpotLight(0xffffff, 1.0, 100, Math.PI / 6, 0.5, 2); // + angle, penumbra, decay
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x362d15, 0.5); // skyColor, groundColor, intensity
const rect = new THREE.RectAreaLight(0xffffff, 5, 4, 4); // color, intensity, width, height
```

**Key properties shared by shadow-casting lights:**

- `castShadow: boolean` — MUST be set to `true` to enable shadow mapping
- `shadow: LightShadow` — the shadow configuration object (DirectionalLightShadow, SpotLightShadow, PointLightShadow)

### 1.2 Shadow Mapping

Shadow mapping is configured on two levels: the renderer and per-light shadow objects.

**Renderer-level shadow map types:**

| Type | Constant | Quality | Performance |
|------|----------|---------|-------------|
| Basic | `THREE.BasicShadowMap` | Hard edges, no filtering | Fastest |
| PCF | `THREE.PCFShadowMap` | Percentage-closer filtering | Default, good balance |
| PCF Soft | `THREE.PCFSoftShadowMap` | Soft edges via blur | Slower, best visual |
| VSM | `THREE.VSMShadowMap` | Variance shadow maps; soft but can have light bleeding | Moderate |

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
```

**Per-light shadow configuration (DirectionalLightShadow):**

```javascript
const light = new THREE.DirectionalLight(0xffffff, 1);
light.castShadow = true;

// Shadow map resolution — ALWAYS use power-of-two values
light.shadow.mapSize.width = 2048;
light.shadow.mapSize.height = 2048;

// Shadow camera frustum (orthographic for DirectionalLight)
light.shadow.camera.left = -50;
light.shadow.camera.right = 50;
light.shadow.camera.top = 50;
light.shadow.camera.bottom = -50;
light.shadow.camera.near = 0.5;
light.shadow.camera.far = 500;

// Bias to prevent shadow acne
light.shadow.bias = -0.0001;
light.shadow.normalBias = 0.02;

// Soft shadow radius (only for PCFSoftShadowMap)
light.shadow.radius = 4;
light.shadow.blurSamples = 8;
```

**Common pitfalls:**

1. **Shadow acne** — Self-shadowing artifacts. Fix: adjust `shadow.bias` (negative values) and `shadow.normalBias` (positive values). Start with `bias = -0.0001`, `normalBias = 0.02`.
2. **Peter panning** — Objects appear to float above surfaces when bias is too large. Fix: use smallest bias that eliminates acne.
3. **Shadow frustum too large** — Low-resolution shadows. Fix: tighten `shadow.camera` frustum to cover only the visible area. Use `CameraHelper` to visualize.
4. **PointLight performance** — Each PointLight with shadows renders SIX shadow maps (cubemap faces). NEVER use more than 1-2 shadow-casting PointLights.
5. **Missing `castShadow`/`receiveShadow`** — MUST set `light.castShadow = true`, `mesh.castShadow = true`, and `groundMesh.receiveShadow = true`.

### 1.3 Environment Lighting (PMREMGenerator)

`PMREMGenerator` converts environment maps into Prefiltered Mipmapped Radiance Environment Maps for image-based lighting (IBL). This is essential for physically-based rendering with `MeshStandardMaterial` and `MeshPhysicalMaterial`.

**Class:** `THREE.PMREMGenerator(renderer)`

**Key methods:**

| Method | Input | Purpose |
|--------|-------|---------|
| `fromScene(scene, sigma)` | Three.js Scene | Generate PMREM from a 3D scene |
| `fromEquirectangular(texture)` | Equirectangular texture | Convert HDR panorama to PMREM |
| `fromCubemap(texture)` | CubeTexture | Convert cubemap to PMREM |
| `dispose()` | — | Free GPU memory (ALWAYS call after use) |

**Standard IBL workflow:**

```javascript
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const pmremGenerator = new THREE.PMREMGenerator(renderer);
const rgbeLoader = new RGBELoader();

rgbeLoader.load('environment.hdr', (hdrTexture) => {
  const envMap = pmremGenerator.fromEquirectangular(hdrTexture);
  scene.environment = envMap.texture;  // PBR materials auto-use this
  scene.background = envMap.texture;   // Optional: visible background
  hdrTexture.dispose();
  pmremGenerator.dispose();
});
```

**Key detail:** When `scene.environment` is set, all `MeshStandardMaterial` and `MeshPhysicalMaterial` instances automatically use it for reflections and ambient lighting — no per-material configuration needed.

---

## 2. Animation System

### 2.1 Architecture Overview

Three.js animation follows a layered architecture:

```
AnimationClip (data container)
  └── KeyframeTrack[] (per-property animation curves)
        └── times[] + values[] (keyframe data)

AnimationMixer (playback engine, one per animated object)
  └── AnimationAction (playback controller per clip)
        └── loop, weight, timeScale, blendMode
```

### 2.2 AnimationMixer

**Constructor:** `new THREE.AnimationMixer(rootObject: Object3D)`

The mixer is bound to a root object. All animated properties are resolved relative to this root via `PropertyBinding`.

**Key methods:**

| Method | Signature | Purpose |
|--------|-----------|---------|
| `clipAction` | `(clip: AnimationClip, root?: Object3D) → AnimationAction` | Get or create an action for a clip |
| `existingAction` | `(clip: AnimationClip, root?: Object3D) → AnimationAction \| null` | Get existing action without creating |
| `update` | `(deltaTime: number) → void` | Advance all actions by delta time |
| `setTime` | `(timeInSeconds: number) → void` | Set absolute mixer time |
| `stopAllAction` | `() → void` | Stop and reset all actions |
| `uncacheClip` | `(clip: AnimationClip) → void` | Remove cached action |
| `uncacheRoot` | `(root: Object3D) → void` | Remove all cached actions for root |

**Properties:**
- `time: number` — current mixer time in seconds
- `timeScale: number` — global speed multiplier (1.0 = normal, 0.5 = half speed)

**Events (dispatched on the mixer):**
- `finished` — action finished playing (when `loop = LoopOnce` and `clampWhenFinished = true`)
- `loop` — action completed one loop cycle

**Critical pattern — ALWAYS call `mixer.update(delta)` every frame:**

```javascript
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  mixer.update(delta);  // NEVER skip this
  renderer.render(scene, camera);
}
```

### 2.3 AnimationClip

**Constructor:** `new THREE.AnimationClip(name: string, duration: number, tracks: KeyframeTrack[])`

A clip is a reusable container of keyframe data. Duration of `-1` auto-calculates from track data.

**Static methods:**
- `AnimationClip.findByName(clips: AnimationClip[], name: string) → AnimationClip` — find clip by name in array
- `AnimationClip.CreateFromMorphTargetSequence(name, morphTargets, fps, noLoop)` — create from morph targets
- `AnimationClip.CreateClipsFromMorphTargetSequences(morphTargets, fps, noLoop)` — batch create from morph targets
- `AnimationClip.parse(json)` / `AnimationClip.toJSON(clip)` — serialization

### 2.4 AnimationAction

Created via `mixer.clipAction(clip)`. NEVER construct directly.

**Key properties:**

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `loop` | Constant | `THREE.LoopRepeat` | `LoopOnce`, `LoopRepeat`, `LoopPingPong` |
| `clampWhenFinished` | boolean | `false` | Hold last frame when finished (requires `LoopOnce`) |
| `timeScale` | number | `1.0` | Per-action speed multiplier |
| `weight` | number | `1.0` | Blend weight (0.0 = no influence, 1.0 = full) |
| `blendMode` | Constant | `NormalAnimationBlendMode` | `NormalAnimationBlendMode` or `AdditiveAnimationBlendMode` |
| `paused` | boolean | `false` | Pause without stopping |
| `repetitions` | number | `Infinity` | Number of loop repetitions |

**Key methods:**

| Method | Purpose |
|--------|---------|
| `play()` | Start or resume playback |
| `stop()` | Stop and reset to start |
| `reset()` | Reset time, weight, and loop count |
| `fadeIn(duration)` | Fade weight from 0 to 1 |
| `fadeOut(duration)` | Fade weight from 1 to 0 |
| `crossFadeFrom(otherAction, duration, warpBoolean)` | Smooth transition FROM another action |
| `crossFadeTo(otherAction, duration, warpBoolean)` | Smooth transition TO another action |
| `setEffectiveTimeScale(timeScale)` | Set speed (respects global timeScale) |
| `setEffectiveWeight(weight)` | Set weight (respects enabled state) |

**Crossfade pattern (character animation state machine):**

```javascript
function switchAction(fromAction, toAction, duration = 0.5) {
  toAction.reset();
  toAction.setEffectiveTimeScale(1);
  toAction.setEffectiveWeight(1);
  toAction.play();
  fromAction.crossFadeTo(toAction, duration, true);
}
```

### 2.5 KeyframeTrack Types

| Track Class | Value Type | Property Example |
|-------------|-----------|------------------|
| `VectorKeyframeTrack` | Vector3 | `.position`, `.scale` |
| `QuaternionKeyframeTrack` | Quaternion | `.quaternion` |
| `NumberKeyframeTrack` | number | `.material.opacity` |
| `BooleanKeyframeTrack` | boolean | `.visible` |
| `ColorKeyframeTrack` | Color | `.material.color` |
| `StringKeyframeTrack` | string | `.morphTargetInfluences` key |

**Constructor:** `new KeyframeTrack(name: string, times: number[], values: number[], interpolation?: number)`

**Interpolation types:**
- `THREE.InterpolateDiscrete` — step function, no interpolation
- `THREE.InterpolateLinear` — linear interpolation (default)
- `THREE.InterpolateSmooth` — smooth (cubic) interpolation

### 2.6 GLTF Skeletal Animation

GLTF/GLB is the primary format for skeletal animation. Animations are loaded via `GLTFLoader`:

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const loader = new GLTFLoader();
loader.load('character.glb', (gltf) => {
  scene.add(gltf.scene);

  const mixer = new THREE.AnimationMixer(gltf.scene);
  // gltf.animations is an array of AnimationClip
  gltf.animations.forEach((clip) => {
    mixer.clipAction(clip).play();
  });
});
```

---

## 3. Post-Processing

### 3.1 Three.js Built-in: EffectComposer

**Import:** `three/addons/postprocessing/EffectComposer.js`

EffectComposer manages a sequential chain of rendering passes. Each pass reads the previous pass's output and writes to the next.

**Constructor:** `new EffectComposer(renderer: WebGLRenderer, renderTarget?: WebGLRenderTarget)`

**Methods:**
- `addPass(pass: Pass)` — add a pass to the chain
- `insertPass(pass: Pass, index: number)` — insert at specific position
- `removePass(pass: Pass)` — remove a pass
- `render(deltaTime?: number)` — execute all passes
- `setSize(width: number, height: number)` — resize all passes
- `setPixelRatio(pixelRatio: number)` — set device pixel ratio

**Standard setup pattern:**

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { SMAAPass } from 'three/addons/postprocessing/SMAAPass.js';

const composer = new EffectComposer(renderer);

// Step 1: ALWAYS add RenderPass first — it captures the scene
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

// Step 2: Add effect passes
const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.5,  // strength
  0.4,  // radius
  0.85  // threshold
);
composer.addPass(bloomPass);

// Step 3: ALWAYS add antialiasing pass last
const smaaPass = new SMAAPass(window.innerWidth, window.innerHeight);
composer.addPass(smaaPass);

// Render loop: use composer.render() INSTEAD of renderer.render()
function animate() {
  requestAnimationFrame(animate);
  composer.render();
}
```

### 3.2 Available Built-in Passes

| Pass | Import Path | Purpose |
|------|-------------|---------|
| `RenderPass` | `postprocessing/RenderPass` | Scene capture (MUST be first) |
| `ShaderPass` | `postprocessing/ShaderPass` | Custom GLSL shader effects |
| `UnrealBloomPass` | `postprocessing/UnrealBloomPass` | Bloom/glow (Unreal Engine style) |
| `SSAOPass` | `postprocessing/SSAOPass` | Screen-space ambient occlusion |
| `GTAOPass` | `postprocessing/GTAOPass` | Ground truth ambient occlusion |
| `SSRPass` | `postprocessing/SSRPass` | Screen-space reflections |
| `OutlinePass` | `postprocessing/OutlinePass` | Object outlines (selection highlight) |
| `SMAAPass` | `postprocessing/SMAAPass` | Subpixel morphological antialiasing |
| `FXAAPass` | `postprocessing/ShaderPass` + `FXAAShader` | Fast approximate antialiasing |
| `TAARenderPass` | `postprocessing/TAARenderPass` | Temporal antialiasing |
| `BokehPass` | `postprocessing/BokehPass` | Depth-of-field |
| `FilmPass` | `postprocessing/FilmPass` | Film grain + scanlines |
| `GlitchPass` | `postprocessing/GlitchPass` | Digital glitch effect |
| `HalftonePass` | `postprocessing/HalftonePass` | Halftone dot patterns |
| `LUTPass` | `postprocessing/LUTPass` | Color grading via lookup table |
| `SAOPass` | `postprocessing/SAOPass` | Scalable ambient occlusion |

### 3.3 pmndrs/postprocessing (Higher-Performance Alternative)

**npm:** `postprocessing` (v6.39.0+)
**License:** Zlib

This library is a drop-in replacement for Three.js built-in post-processing with significant performance advantages.

**Key differences from Three.js EffectComposer:**

| Feature | Three.js Built-in | pmndrs/postprocessing |
|---------|-------------------|----------------------|
| Effect merging | Each pass = separate draw call | Auto-merges effects into single pass |
| Geometry | Full-screen quad (2 triangles) | Single triangle (eliminates diagonal fragments) |
| Color workflow | Manual sRGB handling | Automatic linear workflow with sRGB detection |
| HDR support | Manual buffer config | HalfFloatType frame buffers by default |
| Custom effects | Write full ShaderPass | Extend `Effect` class with fragment shader only |

**Key classes:**
- `EffectComposer` — orchestrates passes (same name, different implementation)
- `RenderPass` — scene capture
- `EffectPass` — combines multiple effects into one pass
- `Effect` — base class for custom effects

**Available effects (21+ categories):** Bloom, Blur, Chromatic Aberration, Color Grading (Brightness/Contrast, Hue/Saturation, LUT, Sepia), Depth of Field, Depth Picking, Dot-Screen, Glitch, God Rays, Grid, Noise, Outline, Pixelation, Scanline, Shock Wave, SSAO, Texture, Tone Mapping, Vignette.

**Critical anti-pattern:** NEVER mix Three.js built-in EffectComposer with pmndrs/postprocessing in the same project — they use incompatible pass architectures.

---

## 4. Physics Integration

### 4.1 cannon-es

**npm:** `cannon-es`
**Type:** Pure JavaScript physics engine (TypeScript fork of cannon.js)
**License:** MIT
**Maintained by:** pmndrs

**Key classes:**

| Class | Purpose |
|-------|---------|
| `World` | Physics simulation container |
| `Body` | Physics-enabled object with mass, velocity, position |
| `Vec3` | 3D vector (NOT Three.js Vector3) |
| `Quaternion` | Rotation quaternion |
| `Shape` (subclasses) | Collision geometry |

**Shape types:** `Sphere`, `Box`, `Cylinder`, `Plane`, `ConvexPolyhedron`, `Trimesh`, `Heightfield`, `Particle`

**Constraint types:** `DistanceConstraint`, `PointToPointConstraint`, `HingeConstraint`, `LockConstraint`, `ConeTwistConstraint`

**Three.js integration pattern:**

```javascript
import * as CANNON from 'cannon-es';
import * as THREE from 'three';

const world = new CANNON.World({ gravity: new CANNON.Vec3(0, -9.82, 0) });

// Create physics body
const body = new CANNON.Body({
  mass: 1,
  shape: new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5)),
  position: new CANNON.Vec3(0, 5, 0),
});
world.addBody(body);

// Create Three.js mesh
const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial()
);
scene.add(mesh);

// Sync in render loop
function animate() {
  world.step(1 / 60);
  mesh.position.copy(body.position);      // CANNON.Vec3 → THREE.Vector3
  mesh.quaternion.copy(body.quaternion);   // CANNON.Quaternion → THREE.Quaternion
  renderer.render(scene, camera);
}
```

**Key properties:**
- `World.hasActiveBodies` — boolean for frame invalidation when physics are dormant
- `World.frictionGravity` — customize friction in zero-gravity scenarios
- Bodies auto-wake from sleep when forces are applied

**Performance notes:** Sleeping body optimization reduces CPU. Use `body.sleepSpeedLimit` and `body.sleepTimeLimit`. Tree-shaking supported for bundle size.

### 4.2 Rapier (dimforge)

**npm:** `@dimforge/rapier3d` (native WASM) or `@dimforge/rapier3d-compat` (embedded WASM, easier setup)
**Type:** Rust-based physics engine compiled to WASM
**License:** Apache-2.0

**Initialization (async required for WASM):**

```javascript
// Option A: Dynamic import (native WASM)
const RAPIER = await import('@dimforge/rapier3d');

// Option B: Compat package (embedded WASM)
import RAPIER from '@dimforge/rapier3d-compat';
await RAPIER.init();
```

**World setup:**

```javascript
const gravity = { x: 0.0, y: -9.81, z: 0.0 };
const world = new RAPIER.World(gravity);
```

**Rigid body creation (descriptor pattern):**

```javascript
const rigidBodyDesc = RAPIER.RigidBodyDesc.dynamic()
  .setTranslation(0.0, 5.0, 0.0);
const rigidBody = world.createRigidBody(rigidBodyDesc);

const colliderDesc = RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5);
const collider = world.createCollider(colliderDesc, rigidBody);
```

**Rigid body types:** `dynamic()`, `fixed()`, `kinematicPositionBased()`, `kinematicVelocityBased()`

**Collider shapes:** `cuboid(hx, hy, hz)`, `ball(radius)`, `capsule(halfHeight, radius)`, `cylinder(halfHeight, radius)`, `cone(halfHeight, radius)`, `trimesh(vertices, indices)`, `convexHull(points)`, `heightfield(nrows, ncols, heights, scale)`

**Simulation loop:**

```javascript
function gameLoop() {
  world.step();
  const position = rigidBody.translation(); // { x, y, z }
  const rotation = rigidBody.rotation();    // { x, y, z, w }
  // Copy to Three.js mesh
  mesh.position.set(position.x, position.y, position.z);
  mesh.quaternion.set(rotation.x, rotation.y, rotation.z, rotation.w);
}
```

**Debug rendering:**

```javascript
const { vertices, colors } = world.debugRender();
// vertices: Float32Array of line segment endpoints
// colors: Float32Array of RGBA values
```

**cannon-es vs Rapier comparison:**

| Feature | cannon-es | Rapier |
|---------|-----------|--------|
| Language | TypeScript/JS | Rust → WASM |
| Bundle size | ~150KB | ~500KB (WASM) |
| Performance | Good for <200 bodies | Excellent for 1000+ bodies |
| Async init | No | Yes (WASM loading) |
| Determinism | Non-deterministic | Cross-platform deterministic |
| CCD | Limited | Full continuous collision detection |
| API style | Mutable objects | Descriptor/builder pattern |

### 4.3 react-three-rapier (R3F Integration)

**npm:** `@react-three/rapier`
**Version:** v2 (React 19 + @react-three/fiber v9) / v1 (React 18)

Wraps Rapier in declarative React components for React Three Fiber.

**Core components:**

```tsx
import { Physics, RigidBody, CuboidCollider, BallCollider } from '@react-three/rapier';

// MUST wrap in Suspense (WASM async loading)
<Canvas>
  <Suspense fallback={null}>
    <Physics gravity={[0, -9.81, 0]} debug>

      {/* Dynamic body — auto-generates collider from mesh */}
      <RigidBody colliders="hull" restitution={0.7}>
        <mesh>
          <boxGeometry args={[1, 1, 1]} />
          <meshStandardMaterial />
        </mesh>
      </RigidBody>

      {/* Static ground */}
      <RigidBody type="fixed">
        <mesh>
          <planeGeometry args={[50, 50]} />
          <meshStandardMaterial />
        </mesh>
      </RigidBody>

    </Physics>
  </Suspense>
</Canvas>
```

**Collider components:** `BallCollider`, `CuboidCollider`, `CapsuleCollider`, `TrimeshCollider`, `ConvexHullCollider`, `MeshCollider`

**Auto-collider modes:** `"cuboid"`, `"ball"`, `"trimesh"`, `"hull"`, `false` (manual)

**Collision events:**

```tsx
<RigidBody
  onCollisionEnter={({ manifold, target, other }) => { /* ... */ }}
  onCollisionExit={() => { /* ... */ }}
  onContactForce={({ totalForce }) => { /* ... */ }}
>
```

**Sensor colliders (trigger volumes):**

```tsx
<RigidBody colliders={false}>
  <mesh><boxGeometry /></mesh>
  <CuboidCollider args={[2, 2, 2]} sensor
    onIntersectionEnter={() => console.log('entered zone')}
    onIntersectionExit={() => console.log('left zone')}
  />
</RigidBody>
```

**Joint types:** `FixedJoint`, `SphericalJoint`, `RevoluteJoint`, `PrismaticJoint`, `RopeJoint`, `SpringJoint`

**Instanced physics (1000+ bodies):**

```tsx
<InstancedRigidBodies instances={instanceData} colliders="ball">
  <instancedMesh args={[undefined, undefined, 1000]} />
</InstancedRigidBodies>
```

---

## 5. React Three Fiber (R3F)

### 5.1 Core Concepts

**npm:** `@react-three/fiber`
**Version:** v9 (React 19) / v8 (React 18)
**License:** MIT

React Three Fiber is a React renderer for Three.js. It maps JSX elements directly to Three.js objects — every Three.js class is available as a lowercase JSX element.

**JSX → Three.js mapping:**

| JSX | Three.js Constructor |
|-----|---------------------|
| `<mesh />` | `new THREE.Mesh()` |
| `<boxGeometry args={[1,1,1]} />` | `new THREE.BoxGeometry(1,1,1)` |
| `<meshStandardMaterial color="red" />` | `new THREE.MeshStandardMaterial({color: 'red'})` |
| `<ambientLight intensity={0.5} />` | `new THREE.AmbientLight(undefined, 0.5)` |
| `<group />` | `new THREE.Group()` |

**`args` prop:** Maps to constructor arguments. `args={[1, 2, 3]}` passes three arguments to the constructor.

### 5.2 Canvas Component

The root component that creates renderer, scene, camera, and render loop:

```tsx
import { Canvas } from '@react-three/fiber';

<Canvas
  camera={{ position: [0, 5, 10], fov: 75 }}
  shadows                   // Enable shadow maps
  gl={{ antialias: true }}  // WebGLRenderer options
  dpr={[1, 2]}             // Device pixel ratio range
  frameloop="demand"        // "always" | "demand" | "never"
>
  {/* 3D content here */}
</Canvas>
```

### 5.3 Core Hooks

**useFrame — render loop subscription:**

```tsx
import { useFrame } from '@react-three/fiber';

function RotatingBox() {
  const meshRef = useRef();
  useFrame((state, delta) => {
    meshRef.current.rotation.y += delta;
    // state.clock, state.camera, state.scene, state.gl available
  });
  return <mesh ref={meshRef}><boxGeometry /><meshStandardMaterial /></mesh>;
}
```

**useThree — access renderer context:**

```tsx
const { gl, scene, camera, size, viewport, clock } = useThree();
```

**useLoader — async asset loading with Suspense:**

```tsx
import { useLoader } from '@react-three/fiber';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

function Model() {
  const gltf = useLoader(GLTFLoader, '/model.glb');
  return <primitive object={gltf.scene} />;
}

// Wrap in Suspense for loading states
<Suspense fallback={<LoadingSpinner />}>
  <Model />
</Suspense>
```

### 5.4 Event System

R3F provides a pointer event system that works like DOM events:

```tsx
<mesh
  onClick={(e) => { e.stopPropagation(); /* ... */ }}
  onPointerOver={(e) => setHovered(true)}
  onPointerOut={(e) => setHovered(false)}
  onPointerMove={(e) => { /* e.point, e.face, e.distance */ }}
  onPointerDown={(e) => { /* ... */ }}
  onPointerUp={(e) => { /* ... */ }}
>
```

**Event object properties:** `point` (Vector3 intersection), `distance`, `face`, `faceIndex`, `object`, `eventObject`, `uv`, `ray`.

### 5.5 Performance Patterns

- **InstancedMesh** — use `<instancedMesh>` for thousands of identical objects
- **LOD** — `<lod>` with distance-based detail levels
- **Suspense** — lazy-load models and textures
- **frameloop="demand"** — only render when state changes (use `invalidate()` to trigger)
- **useMemo** — memoize geometries and materials
- **dispose** — R3F auto-disposes on unmount; set `dispose={null}` to prevent

---

## 6. Drei (R3F Helpers)

**npm:** `@react-three/drei`
**Purpose:** Production-ready abstractions for React Three Fiber

### 6.1 Key Components by Category

**Controls:**
- `OrbitControls` — mouse orbit/zoom/pan
- `MapControls` — map-style panning
- `FlyControls` — first-person flight
- `PresentationControls` — touch-friendly drag rotation
- `ScrollControls` — scroll-linked animations
- `KeyboardControls` — keyboard input abstraction
- `TransformControls` — translate/rotate/scale gizmo
- `PivotControls` — pivot-point manipulation
- `DragControls` — drag objects in 3D space

**Environment & Lighting:**
- `Environment` — load HDR/EXR environment maps with presets (sunset, dawn, night, warehouse, forest, apartment, studio, city, park, lobby)
- `Lightformer` — custom environment light shapes
- `Sky` — procedural sky dome
- `Stars` — starfield background
- `Cloud` — volumetric clouds
- `ContactShadows` — ground-plane contact shadows

**Staging & Layout:**
- `Center` — auto-center objects
- `Float` — floating animation
- `Stage` — pre-configured presentation stage with lighting
- `Bounds` — auto-fit camera to content
- `BBAnchor` — bounding-box anchoring

**Text & UI:**
- `Text` — SDF text rendering (troika-three-text)
- `Text3D` — extruded 3D text geometry
- `Html` — HTML DOM elements positioned in 3D space
- `Billboard` — always-face-camera quad
- `Hud` — heads-up display overlay

**Materials:**
- `MeshReflectorMaterial` — real-time reflective surfaces
- `MeshTransmissionMaterial` — glass/transmission effects
- `MeshRefractionMaterial` — refraction
- `MeshWobbleMaterial` — wave distortion
- `MeshDistortMaterial` — noise distortion
- `shaderMaterial` — typed shader material helper

**Loaders & Assets:**
- `useGLTF` — GLTF/GLB loading with draco/meshopt support
- `useTexture` — texture loading with Suspense
- `useKTX2` — KTX2 compressed texture loading
- `useFBX` — FBX model loading
- `useAnimations` — animation clip management for loaded models

**Performance:**
- `Instances` / `Merged` — instanced rendering helpers
- `Detailed` — LOD wrapper
- `Preload` — preload assets
- `BakeShadows` — bake shadows once (static scenes)
- `AdaptiveDpr` — dynamic resolution scaling
- `PerformanceMonitor` — FPS monitoring with callbacks

**Shapes:** `Box`, `Sphere`, `Plane`, `Cylinder`, `Cone`, `Torus`, `RoundedBox`, `Tube`, `Ring`, `Dodecahedron`, `Icosahedron`, `Octahedron`, `Tetrahedron`

**Advanced:**
- `Edges` — visible edge lines
- `Outlines` — object outlines (post-process)
- `Trail` — motion trails
- `MarchingCubes` — metaball rendering
- `Decal` — surface decals
- `Splat` — Gaussian splatting (3DGS)
- `Sampler` — surface point sampling

---

## 7. WebGPU & TSL (Three Shading Language)

### 7.1 WebGPURenderer

**Import:** `three/addons/renderers/WebGPURenderer.js` (or `three/renderers/webgpu/WebGPURenderer.js`)

**Constructor:** `new WebGPURenderer({ antialias?: boolean, alpha?: boolean, ... })`

**Browser support detection:**

```javascript
import { WebGPU } from 'three/addons/capabilities/WebGPU.js';

if (WebGPU.isAvailable()) {
  const renderer = new THREE.WebGPURenderer({ antialias: true });
} else {
  const renderer = new THREE.WebGLRenderer({ antialias: true }); // fallback
}
```

**Browser compatibility:**
- Chrome / Edge 113+
- Safari 18+ (macOS / iOS)
- Firefox: experimental, behind flag

### 7.2 Node-Based Materials

WebGPU introduces node-based materials that replace GLSL shaders with a composable node graph system:

| Node Material | Classic Equivalent |
|--------------|-------------------|
| `MeshStandardNodeMaterial` | `MeshStandardMaterial` |
| `MeshPhysicalNodeMaterial` | `MeshPhysicalMaterial` |
| `MeshBasicNodeMaterial` | `MeshBasicMaterial` |
| `MeshLambertNodeMaterial` | `MeshLambertMaterial` |
| `MeshPhongNodeMaterial` | `MeshPhongMaterial` |
| `MeshToonNodeMaterial` | `MeshToonMaterial` |
| `MeshMatcapNodeMaterial` | `MeshMatcapMaterial` |
| `LineBasicNodeMaterial` | `LineBasicMaterial` |
| `PointsNodeMaterial` | `PointsMaterial` |
| `SpriteNodeMaterial` | `SpriteMaterial` |

### 7.3 TSL (Three Shading Language)

TSL is a JavaScript-based shader authoring system. Instead of writing GLSL/WGSL strings, you compose shader logic using JavaScript function calls that build a node graph.

**Core TSL node categories:**

| Category | Functions |
|----------|-----------|
| Math | `add`, `sub`, `mul`, `div`, `abs`, `sin`, `cos`, `pow`, `mix`, `clamp`, `smoothstep` |
| Vector | `cross`, `dot`, `normalize`, `length`, `reflect`, `refract`, `faceforward` |
| Matrix | `inverse`, `transpose`, `determinant` |
| Texture | `texture()`, `cubeTexture()`, `texture3D()`, `textureSize()` |
| Lighting | `lights()`, `shadow()`, `bumpMap()`, `normalMap()` |
| Post-processing | `bloom()`, `dof()`, `fxaa()`, `ssao()`, `ssr()`, `chromaticAberration()` |
| Storage | `storage()`, `storageBarrier()`, `compute()` |

**TSL example — custom node material:**

```javascript
import { MeshStandardNodeMaterial } from 'three/addons/materials/nodes/MeshStandardNodeMaterial.js';
import { color, uniform, uv, sin, time } from 'three/nodes';

const material = new MeshStandardNodeMaterial();
material.colorNode = sin(time.mul(2.0)).mul(0.5).add(0.5); // Animated color
material.normalNode = /* custom normal calculation */;
```

### 7.4 Compute Shaders

WebGPU enables GPGPU compute shaders via TSL:

- `ComputeNode` — define compute shader logic
- `StorageBufferNode` — GPU storage buffer read/write
- `StorageTextureNode` — GPU texture read/write

**Use cases:** Particle simulations, fluid dynamics, terrain generation, GPU-accelerated physics.

### 7.5 Migration Path: WebGL → WebGPU

1. **Renderer swap:** Replace `WebGLRenderer` with `WebGPURenderer` (API is mostly compatible)
2. **Materials:** Classic materials (`MeshStandardMaterial`) still work under WebGPU. Migrate to NodeMaterials only when custom shading is needed.
3. **Custom shaders:** Replace `ShaderMaterial` with node-based approach using TSL. GLSL is NOT supported under WebGPU — WGSL is the native language.
4. **Post-processing:** WebGPU has its own `PostProcessing` class (not EffectComposer).
5. **Feature detection:** ALWAYS implement fallback to WebGL for unsupported browsers.

### 7.6 Advanced WebGPU Features

- `TiledLighting` — efficient light culling for many lights
- `CSMShadowNode` — cascaded shadow maps
- `BundleGroup` — draw call batching / optimization
- `TimestampQueryPool` — GPU performance timing
- `WGSLNodeBuilder` — compiles TSL node graph to WGSL shader code

---

## 8. IFC/BIM Viewer

### 8.1 Historical Context

The IFC ecosystem for Three.js has evolved through several generations:

| Library | Status | npm |
|---------|--------|-----|
| `web-ifc` | Active | `web-ifc` |
| `web-ifc-three` | **Deprecated** | `web-ifc-three` |
| `@thatopen/components` | Active (successor) | `@thatopen/components` |
| `@thatopen/components-front` | Active (browser features) | `@thatopen/components-front` |

### 8.2 web-ifc (Low-Level IFC Parser)

**npm:** `web-ifc`
**Purpose:** WASM-based IFC file parser. Reads IFC STEP files and provides access to geometry and property data. Does NOT create Three.js objects directly — that is the job of the higher-level libraries.

### 8.3 Legacy: web-ifc-three (IFCLoader)

**npm:** `web-ifc-three`
**Status:** DEPRECATED — use `@thatopen/components` instead.

Provided `IFCLoader` extending Three.js `Loader`:

```javascript
// DEPRECATED — shown for reference only
import { IFCLoader } from 'web-ifc-three/IFCLoader';

const ifcLoader = new IFCLoader();
await ifcLoader.ifcManager.setWasmPath('./');
ifcLoader.load('model.ifc', (ifcModel) => {
  scene.add(ifcModel);
});
```

**Capabilities:** Spatial tree navigation, property extraction, subset generation (show/hide elements by category), raycasting for element picking.

### 8.4 Modern: @thatopen/components

**npm:** `@thatopen/components` (core) + `@thatopen/components-front` (browser-specific)
**Version:** v3.3.2 (latest as of 2026)
**License:** AGPL-3.0 (core) — check license compatibility
**Language:** TypeScript (93.4%)

**Architecture:** Component-based system built on Three.js. Requires familiarity with Three.js API.

**Setup pattern:**

```javascript
import * as OBC from '@thatopen/components';

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);

// Create world with Three.js scene, camera, renderer
const world = worlds.create();
// world.scene, world.camera, world.renderer available
components.init();
```

**Key capabilities:**
- IFC file loading and parsing (uses web-ifc internally)
- Spatial tree navigation
- Property extraction and querying
- Post-production rendering effects
- Dimensioning tools
- Floorplan navigation
- DXF export
- Section planes

**Documentation:** https://docs.thatopen.com/intro

**Anti-patterns:**
1. NEVER use `web-ifc-three` for new projects — it is deprecated
2. ALWAYS check the AGPL-3.0 license of `@thatopen/components` — it has copyleft implications for commercial use
3. NEVER load large IFC files without streaming/chunking — browser memory limits apply

---

## 9. Cross-Cutting Concerns

### 9.1 Version Compatibility Matrix

| Package | Minimum Version | Notes |
|---------|----------------|-------|
| Three.js | r160+ | ES modules, physically correct lights default |
| React Three Fiber | v9 (React 19) / v8 (React 18) | v9 requires React 19 |
| Drei | Latest (matches R3F version) | ALWAYS match to R3F major version |
| postprocessing | v6.36+ | Requires Three.js r160+ |
| cannon-es | 0.20+ | TypeScript support |
| @dimforge/rapier3d | 0.12+ | WASM, cross-platform deterministic |
| @react-three/rapier | v2 (React 19) / v1 (React 18) | Match to React version |
| @thatopen/components | v3.x | AGPL-3.0 license |

### 9.2 Import Paths

```javascript
// Three.js core
import * as THREE from 'three';

// Three.js addons (loaders, controls, post-processing)
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';

// React Three Fiber
import { Canvas, useFrame, useThree, useLoader } from '@react-three/fiber';

// Drei
import { OrbitControls, Environment, Html, Text, useGLTF } from '@react-three/drei';

// Physics
import * as CANNON from 'cannon-es';
import RAPIER from '@dimforge/rapier3d-compat';
import { Physics, RigidBody } from '@react-three/rapier';

// Post-processing (pmndrs)
import { EffectComposer, RenderPass, EffectPass, BloomEffect } from 'postprocessing';
```

### 9.3 Universal Anti-Patterns

1. **Memory leaks** — ALWAYS dispose geometries, materials, textures, and render targets when no longer needed. Call `.dispose()` explicitly.
2. **Creating objects in render loop** — NEVER create new `Geometry`, `Material`, or `Texture` instances inside `useFrame` or `requestAnimationFrame`. Create once, reuse.
3. **Blocking the main thread** — NEVER load large assets synchronously. Use async loaders with progress callbacks.
4. **Missing resize handling** — ALWAYS update `renderer.setSize()`, `camera.aspect`, and `camera.updateProjectionMatrix()` on window resize.
5. **Pixel ratio mismatch** — ALWAYS call `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))` to cap at 2x for performance.
6. **Uncompressed textures** — ALWAYS use KTX2/Basis compressed textures for production. Use `KTX2Loader` with `BasisTextureLoader`.
7. **Unoptimized GLTF** — ALWAYS use Draco compression for geometry and meshopt for animation data in production GLTF files.

---

## 10. Skill Coverage Mapping

Based on this research, the following skill areas are identified for the Three.js Skill Package:

| Skill Area | Key Topics | Estimated Complexity |
|------------|-----------|---------------------|
| **Lighting & Shadows** | 7 light types, shadow mapping, bias tuning, PMREMGenerator, IBL | High |
| **Animation** | AnimationMixer, AnimationClip, AnimationAction, crossfade, skeletal, GLTF | High |
| **Post-Processing** | EffectComposer (both libs), 20+ effects, custom passes | Medium |
| **Physics (cannon-es)** | World, Body, Shapes, constraints, Three.js sync | Medium |
| **Physics (Rapier)** | WASM init, rigid bodies, colliders, joints, determinism | Medium |
| **React Three Fiber** | Canvas, hooks, events, JSX mapping, performance | High |
| **Drei Helpers** | 50+ components across controls, environment, text, materials | Medium |
| **WebGPU & TSL** | WebGPURenderer, node materials, compute shaders, migration | High |
| **IFC/BIM** | @thatopen/components, spatial tree, properties, Three.js integration | Medium |

---

*End of B1 Ecosystem Research. Total topics: 10 sections, 7 core research areas, 50+ classes documented.*
