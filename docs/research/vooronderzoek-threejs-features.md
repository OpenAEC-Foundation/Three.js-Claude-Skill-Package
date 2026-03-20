# Vooronderzoek: Three.js Features — Lighting, Shadows, Animation, Post-Processing, Physics, Audio

> Research date: 2026-03-20
> Sources: Official Three.js documentation (threejs.org/docs), GitHub source (mrdoob/three.js), r160+
> Scope: Lighting system, shadow system, animation system, post-processing, physics integration, audio system

---

## 1. Lighting System

### 1.1 Light Class Hierarchy

All lights in Three.js inherit from the abstract `Light` base class, which itself extends `Object3D`:

```
EventDispatcher
  └── Object3D
        └── Light (abstract)
              ├── AmbientLight
              ├── HemisphereLight
              ├── DirectionalLight
              ├── PointLight
              ├── SpotLight
              ├── RectAreaLight
              └── LightProbe
```

The `Light` base class provides two shared properties:
- `.color` (Color) — the light's color, default `0xffffff`
- `.intensity` (number) — strength multiplier, default `1`
- `.isLight` (boolean, readonly) — type-checking flag, ALWAYS `true`

The base class exposes one method: `.dispose()` to release GPU resources.

### 1.2 AmbientLight

Globally illuminates ALL objects in the scene equally. Has NO direction and CANNOT cast shadows.

**Constructor:** `new AmbientLight( color?: ColorRepresentation, intensity?: number )`
- `color` — default `0xffffff`
- `intensity` — default `1`

**Properties (own):**
- `.isAmbientLight` (boolean, readonly) — ALWAYS `true`

**Usage pattern:**
```js
const ambient = new THREE.AmbientLight( 0x404040, 0.5 );
scene.add( ambient );
```

**Anti-pattern:** NEVER use AmbientLight as the only light source — it produces flat, dimensionless scenes. ALWAYS combine with at least one directional or point light.

### 1.3 HemisphereLight

Positioned directly above the scene. Color fades from sky color (top) to ground color (bottom). CANNOT cast shadows.

**Constructor:** `new HemisphereLight( skyColor?: ColorRepresentation, groundColor?: ColorRepresentation, intensity?: number )`
- `skyColor` — default `0xffffff`
- `groundColor` — default `0xffffff`
- `intensity` — default `1`

**Properties (own):**
- `.groundColor` (Color) — the lower hemisphere color
- `.isHemisphereLight` (boolean, readonly) — ALWAYS `true`

**Usage pattern:**
```js
const hemi = new THREE.HemisphereLight( 0xffffbb, 0x080820, 1 );
scene.add( hemi );
```

### 1.4 DirectionalLight

Emits parallel rays in a specific direction, simulating infinitely distant light (e.g., sunlight). Direction is computed as the vector from `light.position` to `light.target.position`. Rotation settings have NO effect on direction. Capable of casting shadows.

**Constructor:** `new DirectionalLight( color?: ColorRepresentation, intensity?: number )`
- `color` — default `0xffffff`
- `intensity` — default `1` (in lux for physically correct mode in r160+)

**Properties (own):**
- `.isDirectionalLight` (boolean, readonly) — ALWAYS `true`
- `.shadow` (DirectionalLightShadow) — shadow configuration object
- `.target` (Object3D) — defines where the light points; MUST be added to the scene if you change its position

**Intensity units:** In r160+, DirectionalLight intensity is measured in **lux** when using physically correct lighting. A value of `1` corresponds to approximately 1 lux. Outdoor sunlight is typically 50,000–100,000 lux.

**Critical requirement:** The `.target` object MUST be added to `scene` if you want to reposition it:
```js
const dirLight = new THREE.DirectionalLight( 0xffffff, 1 );
dirLight.position.set( 5, 10, 7.5 );
scene.add( dirLight );
scene.add( dirLight.target ); // REQUIRED to update target position
dirLight.target.position.set( 0, 0, 0 );
```

### 1.5 PointLight

Emits light from a single point in ALL directions. Simulates a bare lightbulb. Capable of casting shadows (via cubemap, 6 render passes — extremely expensive).

**Constructor:** `new PointLight( color?: ColorRepresentation, intensity?: number, distance?: number, decay?: number )`
- `color` — default `0xffffff`
- `intensity` — default `1` (in **candela** for physically correct mode in r160+)
- `distance` — default `0` (infinite range, inverse-square falloff)
- `decay` — default `2` (physically correct inverse-square law)

**Properties (own):**
- `.decay` (number) — dimming rate over distance; `2` = physically correct
- `.distance` (number) — maximum range; `0` = no limit (inverse-square law applies)
- `.isPointLight` (boolean, readonly) — ALWAYS `true`
- `.power` (number) — luminous power in **lumens**; related to intensity by `power = intensity * 4 * Math.PI`
- `.shadow` (PointLightShadow) — shadow configuration

**Distance behavior:** When `distance === 0`, light follows inverse-square falloff infinitely. When `distance > 0`, the light attenuates smoothly to zero at the cutoff distance — this behavior is NOT physically correct but provides artistic control.

### 1.6 SpotLight

Emits from a single point in a cone that widens with distance. Capable of casting shadows.

**Constructor:** `new SpotLight( color?: ColorRepresentation, intensity?: number, distance?: number, angle?: number, penumbra?: number, decay?: number )`
- `color` — default `0xffffff`
- `intensity` — default `1` (in **candela** for physically correct mode)
- `distance` — default `0` (infinite range)
- `angle` — cone half-angle, ALWAYS capped at `Math.PI / 2`, default `Math.PI / 3`
- `penumbra` — soft edge attenuation `[0, 1]`, default `0` (sharp edge)
- `decay` — default `2`

**Properties (own):**
- `.angle` (number) — maximum cone spread angle
- `.decay` (number) — distance-based dimming rate
- `.distance` (number) — light range cutoff
- `.isSpotLight` (boolean, readonly) — ALWAYS `true`
- `.map` (Texture | null) — optional cookie texture modulating light color; REQUIRES `castShadow = true`
- `.penumbra` (number) — cone edge softness `[0, 1]`
- `.power` (number) — luminous power in **lumens**
- `.shadow` (SpotLightShadow) — shadow configuration
- `.target` (Object3D) — direction target; MUST be added to scene

**Anti-pattern:** Setting `penumbra = 0` produces harsh, unrealistic cone edges. ALWAYS use `penumbra >= 0.1` for realistic spotlights.

### 1.7 RectAreaLight

Emits light uniformly across the face of a rectangular plane. Simulates bright windows, strip lighting, or frosted panel lights. CANNOT cast shadows. Works ONLY with PBR materials (MeshStandardMaterial, MeshPhysicalMaterial).

**Constructor:** `new RectAreaLight( color?: ColorRepresentation, intensity?: number, width?: number, height?: number )`
- `color` — default `0xffffff`
- `intensity` — default `1`
- `width` — default `10`
- `height` — default `10`

**Properties (own):**
- `.width` (number) — horizontal span
- `.height` (number) — vertical span
- `.power` (number) — luminous power in lumens
- `.isRectAreaLight` (boolean, readonly) — ALWAYS `true`

**Critical requirements:**
- For WebGLRenderer: MUST call `RectAreaLightUniformsLib.init()` before use
- For WebGPURenderer: MUST use `RectAreaLightTexturesLib` instead
- ALWAYS position and orient using `.position.set()` and `.lookAt()`

```js
import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';
import { RectAreaLightHelper } from 'three/addons/helpers/RectAreaLightHelper.js';

RectAreaLightUniformsLib.init(); // MUST call before creating RectAreaLight

const rectLight = new THREE.RectAreaLight( 0xffffff, 5, 4, 10 );
rectLight.position.set( 5, 5, 0 );
rectLight.lookAt( 0, 0, 0 );
scene.add( rectLight );
scene.add( new RectAreaLightHelper( rectLight ) );
```

### 1.8 LightProbe

An alternative lighting approach that stores information about light passing through 3D space using **spherical harmonics** rather than emitting light directly. Useful for augmented reality and environment-based ambient lighting.

**Constructor:** `new LightProbe( sh?: SphericalHarmonics3, intensity?: number )`
- `sh` — spherical harmonics object encoding lighting data
- `intensity` — default `1`

**Properties (own):**
- `.isLightProbe` (boolean, readonly) — ALWAYS `true`

Three.js currently implements diffuse light probes, functioning equivalently to irradiance environment maps. Generate probe data using `LightProbeGenerator`:
```js
import { LightProbeGenerator } from 'three/addons/lights/LightProbeGenerator.js';
const probe = LightProbeGenerator.fromCubeTexture( cubeTexture );
scene.add( probe );
```

### 1.9 Physically Correct Lighting in r160+

In Three.js r155, the property `renderer.useLegacyLights` was deprecated and later removed. As of r160+, ALL lighting uses physically based units by default:
- **DirectionalLight**: intensity in **lux** (lumens per square meter)
- **PointLight / SpotLight**: intensity in **candela** (lumens per steradian)
- **RectAreaLight**: intensity in **nits** (candela per square meter)

The `.power` property on PointLight, SpotLight, and RectAreaLight provides luminous power in **lumens** as an alternative to setting intensity directly.

### 1.10 Light Helpers

Every shadow-capable light type (and HemisphereLight) has a corresponding helper for debugging:

| Helper | Constructor | Purpose |
|--------|------------|---------|
| `DirectionalLightHelper` | `new DirectionalLightHelper( light, size?, color? )` | Visualizes direction as a plane icon |
| `SpotLightHelper` | `new SpotLightHelper( light, color? )` | Visualizes cone shape |
| `PointLightHelper` | `new PointLightHelper( light, sphereSize?, color? )` | Draws a wireframe sphere |
| `HemisphereLightHelper` | `new HemisphereLightHelper( light, size, color? )` | Shows hemisphere orientation |
| `RectAreaLightHelper` | `new RectAreaLightHelper( light )` | Outlines the rectangular emitting face |
| `LightProbeHelper` | `new LightProbeHelper( probe, size )` | Visualizes spherical harmonics |

**Critical:** ALWAYS call `helper.update()` after changing light properties if the helper does not auto-update.

### 1.11 Light Performance Budget

- **AmbientLight / HemisphereLight**: negligible cost — these apply a uniform color and do NOT contribute to per-fragment calculations significantly
- **DirectionalLight**: moderate cost per light; each adds a pass in the fragment shader
- **PointLight / SpotLight**: higher cost; distance attenuation calculations per fragment
- **RectAreaLight**: most expensive analytic light; uses LTC (Linearly Transformed Cosine) area integration
- **PointLight with shadows**: EXTREME cost — renders 6 shadow maps (one per cubemap face)

**Performance guidelines:**
- Keep total real-time lights under **4-5** for WebGL on mobile devices
- Desktop can handle **8-16** lights depending on scene complexity
- ALWAYS prefer baked lighting for static environments over real-time lights
- Use `TiledLighting` (new in recent r170+ builds) for scenes with many lights — clusters lights into screen-space tiles for efficient evaluation

---

## 2. Shadow System

### 2.1 Enabling Shadows

Three.js shadows require three explicit opt-in steps:

```js
// Step 1: Enable shadow maps on the renderer
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// Step 2: Enable shadow casting on the light
directionalLight.castShadow = true;

// Step 3: Enable per-object shadow behavior
mesh.castShadow = true;    // object casts shadows
ground.receiveShadow = true; // object receives shadows
```

NEVER forget any of these three steps — missing even one results in no shadows appearing.

### 2.2 Shadow Map Types

The `renderer.shadowMap.type` property controls shadow quality and performance:

| Type | Constant | Quality | Performance | Notes |
|------|----------|---------|-------------|-------|
| Basic | `THREE.BasicShadowMap` | Low — hard edges, aliased | Fastest | No filtering; useful for debugging |
| PCF | `THREE.PCFShadowMap` | Medium — slightly softened | Moderate | Default. Percentage-Closer Filtering with fixed kernel |
| PCF Soft | `THREE.PCFSoftShadowMap` | High — soft penumbra | Slower | Variable kernel based on `.radius`; IGNORES `shadow.radius` property |
| VSM | `THREE.VSMShadowMap` | Soft — Gaussian blur | Moderate | Variance Shadow Maps; supports `.radius` and `.blurSamples`; can exhibit light bleeding |

**Note:** `PCFSoftShadowMap` was the default "soft" option. As of r160+, the default type is `PCFShadowMap`. VSM provides true soft shadows with configurable blur but can produce light-bleeding artifacts on thin geometry.

### 2.3 LightShadow Base Properties

All shadow-capable lights share these properties via the `LightShadow` base class:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.autoUpdate` | boolean | `true` | Automatic shadow updates each frame; set `false` for static scenes |
| `.bias` | number | `0` | Depth comparison offset to reduce shadow acne; typical range `-0.001` to `0.001` |
| `.normalBias` | number | `0` | Offsets shadow queries along surface normals; helps with acne on curved surfaces |
| `.mapSize` | Vector2 | `(512, 512)` | Shadow map resolution; MUST be power-of-two values |
| `.radius` | number | `1` | Blur radius for soft shadows (VSMShadowMap); values > 1 create softer edges |
| `.blurSamples` | number | `8` | Number of blur samples (VSMShadowMap only) |
| `.intensity` | number | `1` | Shadow darkness; range `[0, 1]` where `0` = no shadow, `1` = fully opaque shadow |
| `.map` | WebGLRenderTarget | `null` | Auto-generated depth texture; readonly |
| `.camera` | Camera | varies | The virtual camera used to render the shadow map |

**Methods:**
- `.clone()` / `.copy( source )` — duplication
- `.getFrustum()` — retrieves the shadow camera frustum for culling
- `.updateMatrices( light )` — recalculates shadow camera matrices
- `.dispose()` — releases GPU resources
- `.toJSON()` — serialization

### 2.4 DirectionalLight Shadows

Uses an **orthographic** camera for the shadow map. The frustum MUST be manually sized to encompass the area where shadows should appear.

```js
const dirLight = new THREE.DirectionalLight( 0xffffff, 1 );
dirLight.castShadow = true;

// Shadow map resolution
dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;

// Orthographic frustum — MUST size manually
dirLight.shadow.camera.near = 0.5;
dirLight.shadow.camera.far = 500;
dirLight.shadow.camera.left = -50;
dirLight.shadow.camera.right = 50;
dirLight.shadow.camera.top = 50;
dirLight.shadow.camera.bottom = -50;

// Anti-acne
dirLight.shadow.bias = -0.0001;
dirLight.shadow.normalBias = 0.02;
```

**DirectionalLightShadow** specific property:
- `.isDirectionalLightShadow` (boolean, readonly) — ALWAYS `true`

**Anti-pattern:** NEVER leave the shadow camera frustum at default size — it is typically too small or too large, producing clipped or low-resolution shadows. ALWAYS visualize with `CameraHelper`:

```js
const helper = new THREE.CameraHelper( dirLight.shadow.camera );
scene.add( helper );
```

**Cascaded Shadow Maps (CSM):** Three.js does not have built-in CSM, but the community addon `CSM` from `three/addons/csm/CSM.js` splits the view frustum into cascades for large outdoor scenes.

### 2.5 SpotLight Shadows

Uses a **perspective** camera that is automatically configured from the SpotLight's `.angle` and `.distance`.

**SpotLightShadow** specific properties:
- `.aspect` (number) — texture aspect ratio, default `1`
- `.focus` (number) — adjusts shadow camera FOV relative to spotlight FOV, range `[0, 1]`, default `1`
- `.isSpotLightShadow` (boolean, readonly) — ALWAYS `true`

The shadow camera frustum auto-updates based on the spotlight cone, so manual frustum configuration is typically unnecessary.

### 2.6 PointLight Shadows

Uses a **cubemap** (6 perspective cameras, one per face). This is the most expensive shadow type — rendering 6 complete shadow maps per frame.

**PointLightShadow** specific property:
- `.isPointLightShadow` (boolean, readonly) — ALWAYS `true`

**Performance warning:** A single PointLight with shadows is equivalent to rendering the shadow scene **6 times**. NEVER use more than 1-2 shadow-casting PointLights simultaneously. Prefer SpotLights with shadows as a cheaper alternative.

### 2.7 Shadow Artifacts and Fixes

**Shadow Acne:** Striped self-shadowing artifacts caused by depth precision issues.
- Fix: Increase `.bias` (e.g., `-0.0001` to `-0.005`). Start small and increase gradually.
- Fix: Increase `.normalBias` (e.g., `0.02` to `0.1`) for curved surfaces.

**Peter Panning:** Shadows detach from objects and appear to float. Caused by excessive `.bias`.
- Fix: Reduce `.bias` and use `.normalBias` instead.
- Fix: Ensure shadow map resolution is high enough.

**Shadow Swimming:** Shadow edges shimmer when the camera moves. Caused by shadow map texels mapping to different world positions each frame.
- Fix: Snap the shadow camera position to texel-aligned positions.
- Fix: Increase shadow map resolution.

**Transparent/Alpha-tested materials:** By default, shadows treat all geometry as opaque.
- For alpha-tested materials: set `material.alphaTest` and use `customDepthMaterial` with the same alpha map.
- For transparent materials: transparent objects do NOT cast proper shadows by default; use screen-space techniques or baked shadows.

### 2.8 ContactShadow (Drei)

React Three Fiber's Drei library provides `<ContactShadows>` as an alternative to real-time shadow maps. It renders shadows onto a ground plane using a multi-sample blur technique:

```jsx
<ContactShadows position={[0, 0, 0]} opacity={0.5} scale={10} blur={1.5} far={1} />
```

This produces high-quality contact shadows at a fraction of the cost of shadow maps, but works ONLY for ground-plane shadows and does NOT project onto arbitrary geometry.

---

## 3. Animation System

### 3.1 Architecture Overview

Three.js uses a keyframe-based animation system with three core components:

```
AnimationClip (data: keyframe tracks)
    └── AnimationAction (playback controller)
          └── AnimationMixer (master scheduler per object)
```

### 3.2 AnimationMixer

The master playback controller. ALWAYS create one mixer per animated root object.

**Constructor:** `new AnimationMixer( rootObject: Object3D )`

**Properties:**
- `.time` (number) — global mixer time in seconds, default `0`
- `.timeScale` (number) — global time multiplier; `0` pauses ALL actions

**Methods:**
- `.clipAction( clip: AnimationClip, optionalRoot?: Object3D, blendMode?: number ): AnimationAction` — returns (or creates) an action for the given clip
- `.existingAction( clip: AnimationClip, optionalRoot?: Object3D ): AnimationAction | null` — returns previously created action or `null`
- `.getRoot(): Object3D` — returns the mixer's root object
- `.setTime( timeInSeconds: number ): AnimationMixer` — sets global time and updates all actions
- `.stopAllAction(): AnimationMixer` — deactivates all scheduled actions
- `.update( deltaTimeInSeconds: number ): AnimationMixer` — advances the mixer; call in render loop with `clock.getDelta()`
- `.uncacheAction( clip, optionalRoot? ): void` — deallocates action
- `.uncacheClip( clip ): void` — deallocates clip data
- `.uncacheRoot( root ): void` — deallocates root object data

**Events (via addEventListener):**
- `'finished'` — fired when an action completes (only with `LoopOnce` + `clampWhenFinished = true`)
- `'loop'` — fired when an action loops

**Blend modes** (second argument to `clipAction`):
- `THREE.NormalAnimationBlendMode` — standard blending
- `THREE.AdditiveAnimationBlendMode` — additive blending (layered on top of base animation)

### 3.3 AnimationAction

Controls playback of a single animation clip. Created ONLY via `mixer.clipAction()` — NEVER instantiate directly.

**Constructor (internal):** `new AnimationAction( mixer, clip, localRoot?, blendMode? )`

**Properties:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.blendMode` | number | `NormalAnimationBlendMode` | How this action blends with others |
| `.clampWhenFinished` | boolean | `false` | If `true`, pauses at last frame when finished |
| `.enabled` | boolean | `true` | If `false`, action is disabled without reset |
| `.loop` | number | `THREE.LoopRepeat` | Looping mode |
| `.paused` | boolean | `false` | If `true`, freezes playback |
| `.repetitions` | number | `Infinity` | Loop count (ignored for `LoopOnce`) |
| `.time` | number | `0` | Local playback time in seconds |
| `.timeScale` | number | `1` | Speed multiplier; `0` pauses, negative reverses |
| `.weight` | number | `1` | Blend influence `[0, 1]` |
| `.zeroSlopeAtEnd` | boolean | `true` | Smooths interpolation at loop end |
| `.zeroSlopeAtStart` | boolean | `true` | Smooths interpolation at loop start |

**Loop modes:**
- `THREE.LoopOnce` — plays once and stops
- `THREE.LoopRepeat` — loops `repetitions` times, restarting from beginning
- `THREE.LoopPingPong` — plays forward then backward, `repetitions` times

**Playback methods:**
- `.play(): AnimationAction` — starts playback
- `.stop(): AnimationAction` — stops and resets
- `.reset(): AnimationAction` — resets to initial state
- `.startAt( time: number ): AnimationAction` — delays start to a specific mixer time

**Fading and transition methods:**
- `.fadeIn( duration: number ): AnimationAction` — fades weight from `0` to `1`
- `.fadeOut( duration: number ): AnimationAction` — fades weight from `1` to `0`
- `.crossFadeFrom( fadeOutAction: AnimationAction, duration: number, warp: boolean ): AnimationAction` — crossfade from another action
- `.crossFadeTo( fadeInAction: AnimationAction, duration: number, warp: boolean ): AnimationAction` — crossfade to another action
- `.stopFading(): AnimationAction` — cancel any active fade

**Speed and timing methods:**
- `.halt( duration: number ): AnimationAction` — decelerates timeScale to `0` over duration
- `.warp( startTimeScale: number, endTimeScale: number, duration: number ): AnimationAction` — smoothly transitions playback speed
- `.stopWarping(): AnimationAction` — cancel any active warp
- `.setDuration( duration: number ): AnimationAction` — adjusts timeScale so one loop takes exactly `duration` seconds
- `.setEffectiveTimeScale( timeScale: number ): AnimationAction`
- `.setEffectiveWeight( weight: number ): AnimationAction`
- `.setLoop( mode: number, repetitions: number ): AnimationAction`
- `.syncWith( action: AnimationAction ): AnimationAction` — synchronizes this action's time with another

**Query methods:**
- `.isRunning(): boolean` — ALWAYS returns `true` only if actively playing
- `.isScheduled(): boolean` — returns `true` if `.play()` was called
- `.getClip(): AnimationClip`
- `.getMixer(): AnimationMixer`
- `.getRoot(): Object3D`
- `.getEffectiveTimeScale(): number`
- `.getEffectiveWeight(): number`

### 3.4 AnimationClip

A reusable set of keyframe tracks. Typically loaded from GLTF files rather than created manually.

**Constructor:** `new AnimationClip( name?: string, duration?: number, tracks?: KeyframeTrack[], blendMode?: number )`
- `name` — default `''`
- `duration` — default `-1` (auto-calculate from tracks)
- `tracks` — array of KeyframeTrack objects
- `blendMode` — animation combination mode

**Properties:**
- `.blendMode` — blend mode enum
- `.duration` (number) — length in seconds
- `.name` (string) — identifier (GLTF clips use names from the file)
- `.tracks` (KeyframeTrack[]) — keyframe data
- `.userData` (Object) — custom metadata
- `.uuid` (string, readonly) — unique ID

**Instance methods:**
- `.clone(): AnimationClip`
- `.optimize(): AnimationClip` — removes redundant sequential keys
- `.resetDuration(): AnimationClip` — updates duration to longest track
- `.toJSON(): Object`
- `.trim(): AnimationClip` — crops all tracks to clip duration
- `.validate(): boolean` — returns `true` if all tracks are valid

**Static methods:**
- `AnimationClip.CreateClipsFromMorphTargetSequences( name, morphTargetSequence, fps, noLoop )` — generates clips from morph target patterns
- `AnimationClip.CreateFromMorphTargetSequence( name, morphTargetSequence, fps, noLoop )` — single clip from morph targets
- `AnimationClip.findByName( objectOrClipArray, name ): AnimationClip` — lookup by name
- `AnimationClip.parse( json ): AnimationClip` — deserialize from JSON
- `AnimationClip.toJSON( clip ): Object` — serialize to JSON

### 3.5 KeyframeTrack Types

**Base constructor:** `new KeyframeTrack( name: string, times: number[], values: number[], interpolation?: number )`

Specialized track types:
- `VectorKeyframeTrack` — for position, scale (Vector3 values)
- `QuaternionKeyframeTrack` — for rotation (Quaternion values, uses spherical linear interpolation)
- `NumberKeyframeTrack` — for opacity, intensity (scalar values)
- `BooleanKeyframeTrack` — for visibility toggles (discrete only)
- `ColorKeyframeTrack` — for color values
- `StringKeyframeTrack` — for string values (discrete only)

**Interpolation modes:**
- `THREE.InterpolateDiscrete` — step function, no interpolation
- `THREE.InterpolateLinear` — linear interpolation (default)
- `THREE.InterpolateSmooth` — cubic spline interpolation
- Bezier interpolation via factory method, requiring `inTangents` and `outTangents` in track settings

**PropertyBinding path format:**
- `"meshName.position"` — animate position of named mesh
- `"meshName.material.opacity"` — animate material opacity
- `"meshName.morphTargetInfluences[0]"` — animate specific morph target
- `"boneName.quaternion"` — animate bone rotation

**Track utility methods:**
- `.optimize()` — removes redundant sequential keys
- `.scale( timeScale )` — scales all time values
- `.shift( timeOffset )` — offsets all keyframes
- `.trim( startTime, endTime )` — removes keyframes outside range
- `.clone()` — creates a copy
- `.validate()` — checks validity

### 3.6 Clock

**Constructor:** `new Clock( autoStart?: boolean )` — autoStart defaults to `true` (deprecated since r183)

**Properties:**
- `.autoStart` (boolean) — auto-start on first `getDelta()` call
- `.elapsedTime` (number) — total accumulated time in seconds
- `.oldTime` (number) — last method call timestamp
- `.running` (boolean) — active status, default `true`
- `.startTime` (number) — when `start()` was last called

**Methods:**
- `.getDelta(): number` — time since last call in seconds
- `.getElapsedTime(): number` — total elapsed time in seconds
- `.start(): void` — initiates timing
- `.stop(): void` — halts without resetting

**Usage in animation loop:**
```js
const clock = new THREE.Clock();
function animate() {
    const delta = clock.getDelta();
    mixer.update( delta );
    renderer.render( scene, camera );
}
renderer.setAnimationLoop( animate );
```

### 3.7 GLTF Animation Workflow

GLTF files are the standard format for animated 3D content in Three.js. Loading and playing animations:

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const loader = new GLTFLoader();
loader.load( 'model.glb', ( gltf ) => {
    scene.add( gltf.scene );

    const mixer = new THREE.AnimationMixer( gltf.scene );

    // gltf.animations is an array of AnimationClip objects
    gltf.animations.forEach( ( clip ) => {
        mixer.clipAction( clip ).play();
    });
});
```

### 3.8 Crossfade Pattern (Character State Machine)

```js
const idleAction = mixer.clipAction( idleClip );
const walkAction = mixer.clipAction( walkClip );
const runAction = mixer.clipAction( runClip );

// Start with idle
idleAction.play();

function switchToWalk() {
    walkAction.reset();
    walkAction.setEffectiveTimeScale( 1 );
    walkAction.setEffectiveWeight( 1 );
    walkAction.crossFadeFrom( idleAction, 0.5, true ); // warp = true
    walkAction.play();
}
```

### 3.9 Additive Animation Blending

Additive animations layer on top of a base animation. Useful for adding breathing, damage reactions, or aim offsets:

```js
const baseAction = mixer.clipAction( baseClip );
const additiveAction = mixer.clipAction( additiveClip, undefined, THREE.AdditiveAnimationBlendMode );

baseAction.play();
additiveAction.play();
additiveAction.setEffectiveWeight( 0.5 ); // blend strength
```

### 3.10 Morph Target Animation

Morph targets (blend shapes) animate between vertex positions. Controlled via `mesh.morphTargetInfluences[]`:

```js
// Manual morph target animation
mesh.morphTargetInfluences[ 0 ] = Math.sin( elapsed ) * 0.5 + 0.5;

// Or via GLTF animation clips that target morphTargetInfluences
const morphAction = mixer.clipAction( morphClip );
morphAction.play();
```

---

## 4. Post-Processing

### 4.1 Three.js Built-in EffectComposer

The built-in post-processing pipeline uses `EffectComposer` from addons.

**Constructor:** `new EffectComposer( renderer: WebGLRenderer, renderTarget?: WebGLRenderTarget )`

**Properties:**
- `.passes` (Pass[]) — ordered array of post-processing passes
- `.readBuffer` (WebGLRenderTarget) — internal read buffer
- `.writeBuffer` (WebGLRenderTarget) — internal write buffer
- `.renderToScreen` (boolean) — default `true`; final pass renders to screen
- `.renderer` (WebGLRenderer) — the renderer instance

**Methods:**
- `.addPass( pass: Pass ): void` — appends a pass
- `.insertPass( pass: Pass, index: number ): void` — inserts at position
- `.removePass( pass: Pass ): void` — removes a pass
- `.render( deltaTime?: number ): void` — executes all enabled passes
- `.setSize( width: number, height: number ): void` — resizes buffers and passes
- `.setPixelRatio( pixelRatio: number ): void` — configures DPR
- `.swapBuffers(): void` — exchanges read/write buffers
- `.reset( renderTarget?: WebGLRenderTarget ): void` — restores internal state
- `.dispose(): void` — frees GPU resources

**Standard pipeline setup:**
```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );   // ALWAYS first
// ... add effect passes here ...
composer.addPass( new OutputPass() );                    // ALWAYS last (handles tone mapping + color space)

function animate() {
    composer.render(); // replaces renderer.render( scene, camera )
}
```

**Resize handling:**
```js
window.addEventListener( 'resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize( window.innerWidth, window.innerHeight );
    composer.setSize( window.innerWidth, window.innerHeight ); // MUST also resize composer
});
```

### 4.2 Available Pass Types (Three.js addons)

| Pass | Import Path | Constructor | Purpose |
|------|-------------|-------------|---------|
| `RenderPass` | `postprocessing/RenderPass.js` | `new RenderPass( scene, camera, overrideMaterial?, clearColor?, clearAlpha? )` | Beauty pass; ALWAYS first |
| `OutputPass` | `postprocessing/OutputPass.js` | `new OutputPass()` | Tone mapping + color space; ALWAYS last |
| `UnrealBloomPass` | `postprocessing/UnrealBloomPass.js` | `new UnrealBloomPass( resolution: Vector2, strength?, radius?, threshold? )` | HDR bloom |
| `SSAOPass` | `postprocessing/SSAOPass.js` | `new SSAOPass( scene, camera, width?, height?, kernelSize? )` | Screen-space ambient occlusion |
| `GTAOPass` | `postprocessing/GTAOPass.js` | `new GTAOPass( scene, camera, width?, height? )` | Ground truth AO (better quality, higher cost than SSAO) |
| `SAOPass` | `postprocessing/SAOPass.js` | `new SAOPass( scene, camera )` | Scalable ambient obscurance |
| `OutlinePass` | `postprocessing/OutlinePass.js` | `new OutlinePass( resolution: Vector2, scene, camera, selectedObjects? )` | Object outlines |
| `SMAAPass` | `postprocessing/SMAAPass.js` | `new SMAAPass( width, height )` | Subpixel morphological AA |
| `FXAAPass` | `postprocessing/FXAAPass.js` | `new FXAAPass()` | Fast approximate AA |
| `ShaderPass` | `postprocessing/ShaderPass.js` | `new ShaderPass( shader, textureID? )` | Custom GLSL shader effect |
| `BokehPass` | `postprocessing/BokehPass.js` | `new BokehPass( scene, camera, params )` | Depth of field |
| `FilmPass` | `postprocessing/FilmPass.js` | `new FilmPass( intensity?, grayscale? )` | Film grain / scanlines |
| `GlitchPass` | `postprocessing/GlitchPass.js` | `new GlitchPass( dtSize? )` | Digital glitch effect |
| `HalftonePass` | `postprocessing/HalftonePass.js` | `new HalftonePass( width, height, params )` | Halftone dot pattern |
| `DotScreenPass` | `postprocessing/DotScreenPass.js` | `new DotScreenPass( center?, angle?, scale? )` | Dot screen overlay |
| `AfterimagePass` | `postprocessing/AfterimagePass.js` | `new AfterimagePass( damp? )` | Motion trails |
| `SSRPass` | `postprocessing/SSRPass.js` | `new SSRPass( params )` | Screen-space reflections |
| `SSAARenderPass` | `postprocessing/SSAARenderPass.js` | `new SSAARenderPass( scene, camera )` | Super-sampling AA |
| `TAARenderPass` | `postprocessing/TAARenderPass.js` | `new TAARenderPass( scene, camera )` | Temporal AA |
| `RenderPixelatedPass` | `postprocessing/RenderPixelatedPass.js` | `new RenderPixelatedPass( pixelSize, scene, camera )` | Pixelation |
| `MaskPass` | `postprocessing/MaskPass.js` | `new MaskPass( scene, camera )` | Stencil masking |
| `ClearMaskPass` | `postprocessing/ClearMaskPass.js` | `new ClearMaskPass()` | Clears stencil mask |
| `ClearPass` | `postprocessing/ClearPass.js` | `new ClearPass( clearColor?, clearAlpha? )` | Clears buffer |
| `TexturePass` | `postprocessing/TexturePass.js` | `new TexturePass( map, opacity? )` | Renders a texture |
| `CubeTexturePass` | `postprocessing/CubeTexturePass.js` | `new CubeTexturePass( camera, envMap?, opacity? )` | Renders cubemap background |
| `SavePass` | `postprocessing/SavePass.js` | `new SavePass( renderTarget? )` | Saves current buffer |
| `LUTPass` | `postprocessing/LUTPass.js` | `new LUTPass( params )` | Color LUT grading |
| `RenderTransitionPass` | `postprocessing/RenderTransitionPass.js` | `new RenderTransitionPass( sceneA, cameraA, sceneB, cameraB )` | Scene transition |

### 4.3 UnrealBloomPass Tuning Guide

```js
const bloomPass = new UnrealBloomPass(
    new THREE.Vector2( window.innerWidth, window.innerHeight ),
    1.5,  // strength — bloom intensity
    0.4,  // radius — bloom spread [0, 1]
    0.85  // threshold — luminance cutoff
);
```

| Parameter | Range | Effect |
|-----------|-------|--------|
| `strength` | `0` – `3+` | Higher = more intense glow; `1.5` is a good starting point |
| `radius` | `0` – `1` | Higher = wider spread; `0.4` is balanced |
| `threshold` | `0` – `1` | Lower = more objects bloom; `0.85` limits to very bright areas |

**Requirement:** Tone mapping MUST be enabled on the renderer for bloom to function correctly.

**Selective bloom anti-pattern:** Applying bloom to specific objects only is NOT natively supported. The common workaround uses a second scene + composer layer or uses the `layers` system with multiple render passes.

### 4.4 SSAOPass / GTAOPass Configuration

**SSAOPass** (basic, faster):
```js
const ssaoPass = new SSAOPass( scene, camera, width, height );
ssaoPass.kernelRadius = 8;    // how wide AO spreads (default: 8)
ssaoPass.minDistance = 0.005;  // minimum distance for AO
ssaoPass.maxDistance = 0.1;    // maximum distance for AO
```

**GTAOPass** (better quality, slower):
```js
const gtaoPass = new GTAOPass( scene, camera, width, height );
// Access gtaoMap property for the computed AO texture
// Supports blend intensity, denoise settings, and scene clip box
```

ALWAYS prefer GTAOPass over SSAOPass for production quality. SSAOPass is suitable for prototyping.

### 4.5 OutlinePass Configuration

```js
const outlinePass = new OutlinePass(
    new THREE.Vector2( window.innerWidth, window.innerHeight ),
    scene, camera
);
outlinePass.selectedObjects = [ mesh1, mesh2 ]; // objects to outline
outlinePass.edgeStrength = 3;       // edge intensity (default: 3)
outlinePass.edgeGlow = 0;           // animated glow (default: 0)
outlinePass.edgeThickness = 1;      // outline width (default: 1)
outlinePass.visibleEdgeColor.set( 0xffffff ); // visible edge color
outlinePass.hiddenEdgeColor.set( 0x190a05 );  // occluded edge color
outlinePass.pulsePeriod = 0;        // pulse animation timing (0 = off)
outlinePass.downSampleRatio = 2;    // resolution relative to main pass
```

### 4.6 Custom ShaderPass

```js
const myShader = {
    uniforms: {
        tDiffuse: { value: null },    // ALWAYS include — receives the read buffer
        amount: { value: 0.5 }
    },
    vertexShader: `
        varying vec2 vUv;
        void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
        }
    `,
    fragmentShader: `
        uniform sampler2D tDiffuse;
        uniform float amount;
        varying vec2 vUv;
        void main() {
            vec4 color = texture2D( tDiffuse, vUv );
            gl_FragColor = mix( color, vec4( 1.0 - color.rgb, color.a ), amount );
        }
    `
};

const customPass = new ShaderPass( myShader );
composer.insertPass( customPass, 1 ); // after RenderPass
```

**The `textureID` parameter:** Defaults to `'tDiffuse'`. If your shader uses a different uniform name for the input texture, pass it as the second argument to `ShaderPass`.

### 4.7 pmndrs/postprocessing Library

The `pmndrs/postprocessing` library is a community alternative to Three.js's built-in postprocessing. Key architectural differences:

- **Effect merging:** Multiple effects are merged into a single shader pass, reducing draw calls
- **Single full-screen triangle:** Uses one triangle instead of a quad (slightly more efficient)
- **Separate EffectComposer:** Uses its own `EffectComposer`, `EffectPass`, and `RenderPass`
- **React Three Fiber integration:** Available via `@react-three/postprocessing`

**Key effects:**
- `BloomEffect` — configurable bloom with mipmaps
- `SMAAEffect` — subpixel morphological AA
- `SSAOEffect` — screen-space ambient occlusion
- `DepthOfFieldEffect` — bokeh depth-of-field
- `ToneMappingEffect` — various tone mapping operators
- `VignetteEffect` — screen edge darkening
- `ChromaticAberrationEffect` — color fringing
- `NoiseEffect` — film grain
- `GodRaysEffect` — volumetric light scattering

**CRITICAL:** NEVER mix Three.js built-in `EffectComposer` with `pmndrs/postprocessing` `EffectComposer`. They are incompatible and will produce rendering artifacts or crashes.

### 4.8 WebGPU PostProcessing

Three.js WebGPU renderer uses a separate node-based post-processing system via `PostProcessing` class:

```js
import { PostProcessing } from 'three/addons/tsl/display/PostProcessing.js';

const postProcessing = new PostProcessing( renderer );
// Uses TSL (Three Shading Language) node graphs for effects
```

This is entirely separate from both the built-in WebGL `EffectComposer` and the `pmndrs/postprocessing` library. NEVER attempt to use WebGL post-processing passes with the WebGPU renderer.

---

## 5. Physics Integration

### 5.1 cannon-es

`cannon-es` is a maintained fork of cannon.js, providing rigid body physics in pure JavaScript.

**World setup:**
```js
import * as CANNON from 'cannon-es';

const world = new CANNON.World();
world.gravity.set( 0, -9.82, 0 );
world.broadphase = new CANNON.SAPBroadphase( world ); // recommended for most scenes
world.solver.iterations = 10; // default; increase for stability
world.allowSleep = true; // enable sleeping for performance
```

**Broadphase types:**
- `NaiveBroadphase` — O(n^2), tests all pairs; simple but slow for many bodies
- `SAPBroadphase` (Sweep and Prune) — efficient for most scenes; ALWAYS prefer this
- `GridBroadphase` — spatial hash grid; good for uniform distributions

**Body creation:**
```js
const body = new CANNON.Body({
    mass: 5,          // kg; 0 = static body
    position: new CANNON.Vec3( 0, 10, 0 ),
    shape: new CANNON.Box( new CANNON.Vec3( 1, 1, 1 ) ), // half-extents
    linearDamping: 0.01,
    angularDamping: 0.01
});
world.addBody( body );
```

**Body types:**
- `CANNON.Body.DYNAMIC` — affected by forces and collisions (mass > 0)
- `CANNON.Body.STATIC` — immovable, infinite mass (mass = 0)
- `CANNON.Body.KINEMATIC` — moved programmatically, affects dynamic bodies

**Shape types:**
| Shape | Constructor | Notes |
|-------|------------|-------|
| `Box` | `new CANNON.Box( halfExtents: Vec3 )` | Axis-aligned box |
| `Sphere` | `new CANNON.Sphere( radius: number )` | Cheapest collision shape |
| `Cylinder` | `new CANNON.Cylinder( radiusTop, radiusBottom, height, numSegments )` | Cylinder |
| `Plane` | `new CANNON.Plane()` | Infinite plane (use for ground) |
| `ConvexPolyhedron` | `new CANNON.ConvexPolyhedron({ vertices, faces })` | Custom convex hull |
| `Trimesh` | `new CANNON.Trimesh( vertices, indices )` | Triangle mesh; STATIC only |
| `Heightfield` | `new CANNON.Heightfield( data, { elementSize } )` | Terrain heightmap |
| `Particle` | `new CANNON.Particle()` | Point particle |

**Materials and contacts:**
```js
const groundMat = new CANNON.Material( 'ground' );
const ballMat = new CANNON.Material( 'ball' );
const contactMaterial = new CANNON.ContactMaterial( groundMat, ballMat, {
    friction: 0.4,
    restitution: 0.6   // bounciness
});
world.addContactMaterial( contactMaterial );
```

**Constraints:**
- `PointToPointConstraint( bodyA, pivotA, bodyB, pivotB )` — ball joint
- `DistanceConstraint( bodyA, bodyB, distance )` — fixed distance
- `HingeConstraint( bodyA, bodyB, { pivotA, axisA, pivotB, axisB } )` — hinge joint
- `LockConstraint( bodyA, bodyB )` — locks bodies together
- `ConeTwistConstraint( bodyA, bodyB, options )` — cone twist (ragdoll joints)
- `Spring( bodyA, bodyB, options )` — damped spring (not technically a constraint)

**Events:**
- `body.addEventListener( 'collide', callback )` — on collision
- `body.addEventListener( 'sleep', callback )` — body went to sleep
- `body.addEventListener( 'wakeup', callback )` — body woke up

**Performance tips:**
- ALWAYS enable `world.allowSleep = true` — sleeping bodies skip simulation
- Use `SAPBroadphase` for scenes with > 20 bodies
- Minimize `Trimesh` usage — convex decomposition is ALWAYS faster
- Use `world.step( fixedTimeStep, deltaTime, maxSubSteps )` for deterministic simulation

### 5.2 Rapier

Rapier is a Rust-based physics engine compiled to WASM. Significantly faster than cannon-es for large scenes.

**WASM initialization (async — MUST await):**
```js
import RAPIER from '@dimforge/rapier3d-compat';

await RAPIER.init(); // MUST await before using ANY Rapier API

const gravity = { x: 0.0, y: -9.81, z: 0.0 };
const world = new RAPIER.World( gravity );
```

**RigidBody creation (builder pattern):**
```js
// Dynamic body
const rigidBodyDesc = RAPIER.RigidBodyDesc.dynamic()
    .setTranslation( 0, 10, 0 )
    .setLinvel( 0, 0, 0 )          // linear velocity
    .setAngvel( { x: 0, y: 0, z: 0 } ) // angular velocity
    .setLinearDamping( 0.01 )
    .setAngularDamping( 0.01 )
    .setCcdEnabled( true );        // continuous collision detection
const rigidBody = world.createRigidBody( rigidBodyDesc );

// Static body
const staticDesc = RAPIER.RigidBodyDesc.fixed()
    .setTranslation( 0, 0, 0 );
const staticBody = world.createRigidBody( staticDesc );

// Kinematic body (position-based)
const kinematicDesc = RAPIER.RigidBodyDesc.kinematicPositionBased();
const kinematicBody = world.createRigidBody( kinematicDesc );

// Kinematic body (velocity-based)
const kinematicVelDesc = RAPIER.RigidBodyDesc.kinematicVelocityBased();
```

**Collider creation (builder pattern):**
```js
// Attach collider to rigid body
const colliderDesc = RAPIER.ColliderDesc.cuboid( 1, 1, 1 ) // half-extents
    .setRestitution( 0.5 )
    .setFriction( 0.7 );
world.createCollider( colliderDesc, rigidBody );
```

**Collider shapes:**
| Shape | Constructor | Notes |
|-------|------------|-------|
| `ColliderDesc.cuboid( hx, hy, hz )` | Box with half-extents | Most common |
| `ColliderDesc.ball( radius )` | Sphere | Cheapest shape |
| `ColliderDesc.capsule( halfHeight, radius )` | Capsule | Good for characters |
| `ColliderDesc.cylinder( halfHeight, radius )` | Cylinder | |
| `ColliderDesc.cone( halfHeight, radius )` | Cone | |
| `ColliderDesc.convexHull( vertices )` | Convex hull | From Float32Array |
| `ColliderDesc.trimesh( vertices, indices )` | Triangle mesh | STATIC only recommended |
| `ColliderDesc.heightfield( nrows, ncols, heights, scale )` | Terrain | |
| `ColliderDesc.roundCuboid( hx, hy, hz, borderRadius )` | Rounded box | |

**Simulation step and queries:**
```js
// Fixed timestep
world.step();

// Ray casting
const ray = new RAPIER.Ray( { x: 0, y: 10, z: 0 }, { x: 0, y: -1, z: 0 } );
const hit = world.castRay( ray, 100, true );
if ( hit ) {
    const hitPoint = ray.pointAt( hit.timeOfImpact );
}

// Shape intersection queries
const shape = new RAPIER.Cuboid( 1, 1, 1 );
world.intersectionsWithShape( position, rotation, shape, ( collider ) => {
    // called for each intersecting collider
    return true; // continue iteration
});
```

**Collision events:**
```js
const eventQueue = new RAPIER.EventQueue( true );
world.step( eventQueue );

eventQueue.drainCollisionEvents( ( handle1, handle2, started ) => {
    // started = true for collision start, false for collision end
});

eventQueue.drainContactForceEvents( ( event ) => {
    const totalForce = event.totalForceMagnitude();
});
```

**Debug rendering:**
```js
const { vertices, colors } = world.debugRender();
// vertices: Float32Array of line segment endpoints
// colors: Float32Array of RGBA colors per vertex
// Render as THREE.LineSegments with BufferGeometry
```

### 5.3 Three.js Sync Pattern

Both engines require manual synchronization between physics bodies and Three.js meshes:

```js
// cannon-es sync
function updatePhysics( deltaTime ) {
    world.step( 1 / 60, deltaTime, 3 );
    mesh.position.copy( body.position );
    mesh.quaternion.copy( body.quaternion );
}

// Rapier sync
function updatePhysics() {
    world.step();
    const position = rigidBody.translation();
    const rotation = rigidBody.rotation();
    mesh.position.set( position.x, position.y, position.z );
    mesh.quaternion.set( rotation.x, rotation.y, rotation.z, rotation.w );
}
```

### 5.4 Performance Comparison

| Feature | cannon-es | Rapier |
|---------|-----------|--------|
| Language | JavaScript | Rust → WASM |
| Initialization | Synchronous | Async (MUST await `init()`) |
| Performance | ~1000 bodies real-time | ~10,000+ bodies real-time |
| Bundle size | ~100KB | ~300-600KB (WASM) |
| CCD | Limited | Full support |
| Deterministic | No | Yes (cross-platform) |
| Debug rendering | Manual | Built-in `world.debugRender()` |
| API style | Constructor-based | Builder pattern |
| R3F integration | `@react-three/cannon` | `@react-three/rapier` |

**Recommendation:** Use cannon-es for simple scenes (< 100 bodies) and prototyping. Use Rapier for production applications requiring performance, determinism, or CCD.

---

## 6. Audio System

### 6.1 Architecture Overview

Three.js audio is built on the Web Audio API with three core classes:

```
EventDispatcher
  └── Object3D
        ├── AudioListener (receiver — attach to camera)
        ├── Audio (non-positional — background music)
        └── PositionalAudio (3D spatial — attached to objects)
```

### 6.2 AudioListener

The virtual listener for all audio in the scene. ALWAYS create exactly one listener and attach it to the camera.

**Constructor:** `new AudioListener()`

**Properties (readonly):**
- `.context` (AudioContext) — the native Web Audio API context
- `.gain` (GainNode) — master volume control
- `.filter` (AudioNode | null) — optional audio filter
- `.timeDelta` (number) — time delta for audio operations

**Methods:**
- `.getMasterVolume(): number` — get current master volume
- `.setMasterVolume( value: number ): AudioListener` — set master volume
- `.getFilter(): AudioNode` — get current filter
- `.setFilter( filter: AudioNode ): AudioListener` — apply a filter
- `.removeFilter(): AudioListener` — clear the filter
- `.getInput(): GainNode` — returns the listener's input node

**Setup pattern:**
```js
const listener = new THREE.AudioListener();
camera.add( listener ); // ALWAYS add to camera — listener position = camera position
```

### 6.3 Audio (Non-Positional)

Global audio source — volume is the same regardless of listener position. Use for background music and UI sounds.

**Constructor:** `new Audio( listener: AudioListener )`

**Properties:**
- `.buffer` (AudioBuffer, readonly) — the loaded audio data
- `.context` (AudioContext, readonly) — Web Audio context
- `.gain` (GainNode, readonly) — volume control node
- `.isPlaying` (boolean, readonly) — playback state
- `.source` (AudioBufferSourceNode, readonly) — the audio source node
- `.autoplay` (boolean) — auto-play when buffer is set; default `false`
- `.loop` (boolean) — loop playback; default `false`
- `.loopStart` (number) — loop start time in seconds
- `.loopEnd` (number) — loop end time in seconds
- `.offset` (number) — playback start offset in seconds
- `.playbackRate` (number) — speed multiplier; default `1`
- `.detune` (number) — pitch shift in cents

**Methods:**
- `.play(): Audio` — start playback
- `.pause(): Audio` — pause playback
- `.stop(): Audio` — stop and reset
- `.setBuffer( buffer: AudioBuffer ): Audio` — set audio data from AudioLoader
- `.setMediaElementSource( mediaElement: HTMLMediaElement ): Audio` — use HTML5 audio/video as source
- `.setMediaStreamSource( mediaStream: MediaStream ): Audio` — use live stream (microphone)
- `.setNodeSource( audioNode: AudioNode ): Audio` — use custom Web Audio node
- `.setVolume( value: number ): Audio` — set volume `[0, 1]`
- `.getVolume(): number` — get current volume
- `.setPlaybackRate( value: number ): Audio` — set speed
- `.getPlaybackRate(): number` — get current speed
- `.setDetune( value: number ): Audio` — set pitch shift
- `.getDetune(): number` — get current pitch shift
- `.setFilters( filters: AudioNode[] ): Audio` — apply filter chain
- `.getFilters(): AudioNode[]` — get current filters
- `.setFilter( filter: AudioNode ): Audio` — set single filter
- `.getFilter(): AudioNode` — get current filter
- `.connect(): Audio` — connect to destination
- `.disconnect(): Audio` — disconnect from destination

### 6.4 PositionalAudio (3D Spatial)

Spatial audio source with distance-based volume rolloff and directional cones. Position determined by the Object3D it is attached to.

**Constructor:** `new PositionalAudio( listener: AudioListener )`

**Properties (own):**
- `.panner` (PannerNode, readonly) — manages 3D spatial behavior

**Methods (in addition to inherited Audio methods):**
- `.getDistanceModel(): string` — returns `'linear'`, `'inverse'`, or `'exponential'`
- `.setDistanceModel( value: string ): PositionalAudio` — sets the distance attenuation algorithm
- `.getRefDistance(): number` — reference distance where attenuation starts
- `.setRefDistance( value: number ): PositionalAudio` — set reference distance
- `.getMaxDistance(): number` — maximum distance (relevant for `'linear'` model)
- `.setMaxDistance( value: number ): PositionalAudio` — set max distance
- `.getRolloffFactor(): number` — rate of volume decrease
- `.setRolloffFactor( value: number ): PositionalAudio` — set rolloff rate
- `.setDirectionalCone( coneInnerAngle: number, coneOuterAngle: number, coneOuterGain: number ): PositionalAudio` — defines a directional audio cone

**Distance models:**

| Model | Formula | Use Case |
|-------|---------|----------|
| `'inverse'` | `refDistance / (refDistance + rolloffFactor * (distance - refDistance))` | Default; realistic falloff |
| `'linear'` | `1 - rolloffFactor * (distance - refDistance) / (maxDistance - refDistance)` | Predictable, artist-friendly |
| `'exponential'` | `(distance / refDistance) ^ -rolloffFactor` | Dramatic falloff |

**Setup pattern:**
```js
const sound = new THREE.PositionalAudio( listener );
audioLoader.load( 'sound.ogg', ( buffer ) => {
    sound.setBuffer( buffer );
    sound.setRefDistance( 20 );
    sound.setRolloffFactor( 1 );
    sound.setDistanceModel( 'inverse' );
    sound.setLoop( true );
    sound.setVolume( 0.5 );
});
mesh.add( sound ); // attach to a mesh — sound position follows mesh
```

### 6.5 AudioLoader

Loads audio buffers asynchronously via FileLoader internally.

**Constructor:** `new AudioLoader( manager?: LoadingManager )`

**Methods:**
- `.load( url: string, onLoad?: Function, onProgress?: Function, onError?: Function ): void`

The `onLoad` callback receives an `AudioBuffer` object.

```js
const audioLoader = new THREE.AudioLoader();
audioLoader.load( 'music.mp3', ( buffer ) => {
    sound.setBuffer( buffer );
    sound.play();
});
```

### 6.6 AudioAnalyser

Provides real-time frequency analysis for audio visualization.

**Constructor:** `new AudioAnalyser( audio: Audio, fftSize?: number )`
- `audio` — the Audio or PositionalAudio to analyze
- `fftSize` — FFT window size, default `2048`; MUST be a power of 2

**Methods:**
- `.getFrequencyData(): Uint8Array` — frequency domain data (0-255 per bin); array length = `fftSize / 2`
- `.getAverageFrequency(): number` — mean of all frequency values

**Usage for visualization:**
```js
const analyser = new THREE.AudioAnalyser( sound, 256 );

function animate() {
    const data = analyser.getFrequencyData();
    const avg = analyser.getAverageFrequency();
    // Use data to drive visual elements
    mesh.scale.y = avg / 128; // scale mesh based on volume
}
```

### 6.7 Browser Autoplay Policy

Modern browsers REQUIRE user interaction (click, tap, keypress) before audio can play. This is a hard requirement enforced by Chrome, Firefox, and Safari.

**ALWAYS use this pattern:**
```js
document.addEventListener( 'click', () => {
    if ( listener.context.state === 'suspended' ) {
        listener.context.resume();
    }
    sound.play();
}, { once: true } );
```

**Anti-patterns:**
- NEVER set `autoplay = true` and expect it to work without user interaction
- NEVER ignore the `AudioContext.state === 'suspended'` check
- NEVER create AudioContext before user interaction on iOS Safari

---

## Summary Table: Feature Coverage

| Feature | Shadow Support | Performance Cost | Key Limitation |
|---------|---------------|-----------------|----------------|
| AmbientLight | No | Negligible | No directionality |
| HemisphereLight | No | Negligible | No shadows |
| DirectionalLight | Yes (orthographic) | Moderate | Manual frustum sizing |
| PointLight | Yes (cubemap, 6 faces) | Extreme | 6x shadow map cost |
| SpotLight | Yes (perspective) | Moderate | Cone angle limit PI/2 |
| RectAreaLight | No | High | PBR materials only |
| LightProbe | No | Low | Diffuse only |
| Animation System | N/A | Varies | Manual mixer.update() |
| Post-Processing | N/A | Per-pass | Resize handling required |
| cannon-es Physics | N/A | ~1000 bodies | No CCD |
| Rapier Physics | N/A | ~10000+ bodies | Async WASM init |
| Audio (non-positional) | N/A | Low | Autoplay policy |
| PositionalAudio | N/A | Low | Autoplay policy |

---

> End of vooronderzoek. All information cross-referenced against official Three.js documentation (r160+) and GitHub source at `github.com/mrdoob/three.js`.
