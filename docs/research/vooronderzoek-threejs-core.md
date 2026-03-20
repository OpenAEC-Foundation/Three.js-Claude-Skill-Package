# Vooronderzoek: Three.js Core Concepts

> Deep research document for the Three.js Skill Package.
> Covers scene graph, renderer, cameras, raycaster, math utilities, and disposal patterns.
> All API references verified against official Three.js documentation (r160+).
> English only. Deterministic language (ALWAYS/NEVER).

---

## 1. Scene Graph Deep Dive

### 1.1 Object3D — The Foundation of Every 3D Object

Every visible and invisible entity in a Three.js scene inherits from `Object3D`. It is the base class for `Mesh`, `Group`, `Light`, `Camera`, `Bone`, `Sprite`, `Line`, `Points`, and `Scene` itself. Understanding Object3D is ALWAYS required before working with any Three.js object.

**Source:** https://threejs.org/docs/#api/en/core/Object3D

#### Complete Property Catalog

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `uuid` | `string` | auto-generated | Read-only unique identifier (RFC 4122 v4) |
| `name` | `string` | `""` | Human-readable name. NEVER relied upon for lookup in production; use `getObjectByName()` for convenience only |
| `type` | `string` | `"Object3D"` | Read-only type string |
| `parent` | `Object3D \| null` | `null` | Parent in the scene graph. Set automatically by `add()`/`remove()` |
| `children` | `Object3D[]` | `[]` | Direct children array. NEVER modify directly; ALWAYS use `add()`/`remove()` |
| `position` | `Vector3` | `(0, 0, 0)` | Local position relative to parent |
| `rotation` | `Euler` | `(0, 0, 0, 'XYZ')` | Local rotation. Linked to `quaternion` — modifying one ALWAYS updates the other |
| `quaternion` | `Quaternion` | `(0, 0, 0, 1)` | Local rotation as quaternion. Linked to `rotation` |
| `scale` | `Vector3` | `(1, 1, 1)` | Local scale. Non-uniform scale can cause issues with normals |
| `up` | `Vector3` | `(0, 1, 0)` | Up direction for `lookAt()`. ALWAYS Y-up by default |
| `visible` | `boolean` | `true` | When `false`, object and all descendants are skipped during rendering. Traversal methods still visit invisible objects unless `traverseVisible()` is used |
| `castShadow` | `boolean` | `false` | Whether this object casts shadows |
| `receiveShadow` | `boolean` | `false` | Whether this object receives shadows |
| `frustumCulled` | `boolean` | `true` | Whether the renderer skips objects outside the camera frustum. Set to `false` for objects that must ALWAYS render (e.g., skyboxes, large particle systems) |
| `renderOrder` | `number` | `0` | Overrides default rendering order. Higher values render later. Useful for transparency sorting |
| `layers` | `Layers` | layer 0 enabled | 32-layer bitmask system for selective rendering and raycasting |
| `userData` | `object` | `{}` | Dictionary for custom application data. NEVER store Three.js objects here without manual disposal |
| `matrix` | `Matrix4` | identity | Local transform matrix (position + rotation + scale combined) |
| `matrixWorld` | `Matrix4` | identity | World (global) transform matrix |
| `matrixAutoUpdate` | `boolean` | `true` | When `true`, the renderer calls `updateMatrix()` automatically before each frame |
| `matrixWorldAutoUpdate` | `boolean` | `true` | When `true`, the renderer calls `updateMatrixWorld()` automatically |
| `matrixWorldNeedsUpdate` | `boolean` | `false` | Forces world matrix recalculation on next `updateMatrixWorld()` call |
| `modelViewMatrix` | `Matrix4` | — | Computed per frame: `camera.matrixWorldInverse * object.matrixWorld`. Used in shaders |
| `normalMatrix` | `Matrix3` | — | Inverse-transpose of the upper-left 3x3 of `modelViewMatrix`. Used for correct lighting |

#### Hierarchy Methods

```typescript
add(object: Object3D, ...args: Object3D[]): this
```
Adds one or more children. If the object already has a parent, it is ALWAYS removed from the old parent first. Returns `this` for chaining. Fires the `added` event on the child.

```typescript
remove(object: Object3D, ...args: Object3D[]): this
```
Removes one or more children. Fires the `removed` event on the child. NEVER throws if the object is not a child; silently does nothing.

```typescript
removeFromParent(): this
```
Removes this object from its parent. Equivalent to `this.parent.remove(this)`. Safe to call when `parent` is `null`.

```typescript
clear(): this
```
Removes ALL children. ALWAYS prefer `clear()` over iterating `children` for bulk removal to avoid index-shifting bugs.

```typescript
attach(object: Object3D): this
```
Adds the object as a child while preserving its world transform. This is critical when reparenting objects that must maintain their visual position. Internally computes the new local transform from the world transform. ALWAYS use `attach()` instead of `add()` when you need world-position preservation.

**Edge case:** When calling `add()` during a `traverse()` callback, the newly added object MAY be visited in the same traversal. NEVER modify the children array during traversal; collect objects first, then modify after traversal completes.

#### Traversal Methods

```typescript
traverse(callback: (object: Object3D) => void): void
```
Depth-first traversal of this object and ALL descendants. The callback receives each object. Performance: O(n) where n is total descendant count.

```typescript
traverseVisible(callback: (object: Object3D) => void): void
```
Same as `traverse()` but skips objects where `visible === false` and all their descendants.

```typescript
traverseAncestors(callback: (object: Object3D) => void): void
```
Walks UP the tree from this object's parent to the root. Does NOT include the object itself.

```typescript
getObjectByName(name: string): Object3D | undefined
getObjectById(id: number): Object3D | undefined
getObjectByProperty(name: string, value: any): Object3D | undefined
```
Recursive search through descendants. Returns the first match or `undefined`. Performance: O(n) — for frequent lookups, ALWAYS cache the reference.

#### Matrix Update Cycle

The matrix update cycle is one of the most misunderstood aspects of Three.js:

1. **Automatic mode** (`matrixAutoUpdate = true`, default): Before each render, the renderer calls `updateMatrixWorld()` on the scene, which recursively:
   - Calls `updateMatrix()` on each object (recomputes `matrix` from `position`, `rotation`, `scale`)
   - Multiplies `parent.matrixWorld * matrix` to get `matrixWorld`

2. **Manual mode** (`matrixAutoUpdate = false`): You MUST call `updateMatrix()` yourself after changing position/rotation/scale. Set this for static objects to improve performance — the renderer skips the matrix recomputation.

3. **Force update**: `updateMatrixWorld(true)` forces the entire subtree to recalculate regardless of flags.

4. **`updateWorldMatrix(updateParents, updateChildren)`**: More granular control — updates this object's world matrix, optionally updating parents first (to ensure correct chain) and/or children after.

**Anti-pattern:** Reading `matrixWorld` values without first calling `updateWorldMatrix(true, false)` after modifying position/rotation/scale in the same frame. The world matrix is stale until the next render or explicit update.

```javascript
// CORRECT: manual matrix workflow
object.matrixAutoUpdate = false;
object.matrix.compose(position, quaternion, scale);
object.matrixWorldNeedsUpdate = true;

// CORRECT: read world position mid-frame
object.position.set(10, 0, 0);
object.updateWorldMatrix(true, false); // ensure parents are current
const worldPos = new THREE.Vector3();
object.getWorldPosition(worldPos);     // now correct
```

#### World-Space Query Methods

```typescript
getWorldPosition(target: Vector3): Vector3
getWorldQuaternion(target: Quaternion): Quaternion
getWorldScale(target: Vector3): Vector3
getWorldDirection(target: Vector3): Vector3
```
All four methods ALWAYS write to the `target` parameter (to avoid garbage collection pressure from creating new objects). They call `updateWorldMatrix(true, false)` internally to ensure correctness.

#### Transformation Methods

```typescript
lookAt(x: number, y: number, z: number): void
lookAt(vector: Vector3): void
```
Rotates the object to face a world-space point. For cameras, this points the negative-Z axis toward the target. For other objects, it points the positive-Z axis toward the target. ALWAYS call this after setting position.

```typescript
rotateOnAxis(axis: Vector3, angle: number): this          // local-space rotation
rotateOnWorldAxis(axis: Vector3, angle: number): this      // world-space rotation
rotateX(rad: number): this                                  // shorthand for rotateOnAxis(xAxis, rad)
rotateY(rad: number): this
rotateZ(rad: number): this
translateOnAxis(axis: Vector3, distance: number): this     // local-space translation
translateX(distance: number): this
translateY(distance: number): this
translateZ(distance: number): this
```

#### Layers System

The Layers system uses a 32-bit bitmask allowing objects to belong to any combination of layers 0-31:

```typescript
// Layers API
layers.set(channel: number): void         // Enable ONLY this channel (disables all others)
layers.enable(channel: number): void      // Enable a channel without affecting others
layers.enableAll(): void                  // Enable all 32 channels
layers.toggle(channel: number): void      // Toggle a channel
layers.disable(channel: number): void     // Disable a channel
layers.disableAll(): void                 // Disable all channels
layers.isEnabled(channel: number): boolean // Check if channel is enabled
layers.test(layers: Layers): boolean      // Bitwise AND test against another Layers object
```

**Source:** https://threejs.org/docs/#api/en/core/Layers

Rendering visibility is determined by `camera.layers.test(object.layers)`. If the test fails (no overlapping layers), the object is NOT rendered. Raycasters also respect layers: `raycaster.layers.test(object.layers)`.

**Use cases:**
- Layer 0: default visible objects
- Layer 1: helpers/gizmos (disable on camera for production)
- Layer 2: bloom pass objects (selective bloom with separate render pass)
- Layer 3-31: custom application layers (e.g., collision groups, selection groups)

#### userData Conventions

`userData` is a plain object for application-specific data. Common patterns:

```javascript
mesh.userData.selectable = true;
mesh.userData.category = 'wall';
mesh.userData.originalMaterial = mesh.material; // for hover effects
```

**Anti-pattern:** NEVER store circular references or DOM elements in `userData` without explicit cleanup. NEVER assume `userData` survives `clone()` deep-cloning — it performs a JSON-style shallow copy.

### 1.2 Scene

**Source:** https://threejs.org/docs/#api/en/scenes/Scene

`Scene` extends `Object3D` and serves as the root container for all renderable objects.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `background` | `Color \| Texture \| CubeTexture \| null` | `null` | Scene background. `Color` for solid, `Texture` for 2D image, `CubeTexture` for skybox |
| `environment` | `Texture \| null` | `null` | Environment map applied to ALL `MeshStandardMaterial`/`MeshPhysicalMaterial` objects that do not have their own `envMap`. Used for image-based lighting (IBL) |
| `fog` | `Fog \| FogExp2 \| null` | `null` | Fog effect applied to the scene |
| `overrideMaterial` | `Material \| null` | `null` | Forces ALL objects to use this material. Useful for depth-only passes or debug rendering |
| `backgroundBlurriness` | `number` | `0` | Blur amount for background environment maps (0-1) |
| `backgroundIntensity` | `number` | `1` | Intensity multiplier for the background |
| `backgroundRotation` | `Euler` | `(0,0,0)` | Rotation applied to the background |
| `environmentIntensity` | `number` | `1` | Intensity multiplier for the environment map |
| `environmentRotation` | `Euler` | `(0,0,0)` | Rotation applied to the environment map |

### 1.3 Fog

**Source:** https://threejs.org/docs/#api/en/scenes/Fog

**Fog (Linear):**
```typescript
new Fog(color: Color | string | number, near?: number, far?: number)
```
- `color`: Fog color
- `near`: Distance where fog starts (default: `1`)
- `far`: Distance where fog is fully opaque (default: `1000`)
- Linear interpolation between near and far

**FogExp2 (Exponential):**
```typescript
new FogExp2(color: Color | string | number, density?: number)
```
- `color`: Fog color
- `density`: Exponential density coefficient (default: `0.00025`)
- More physically accurate atmospheric scattering simulation

**Material interaction:** Every material has a `fog` boolean property (default: `true`). When `fog` is `false`, that material ignores scene fog entirely. `ShaderMaterial` and `RawShaderMaterial` have `fog: false` by default — you MUST set it to `true` and include fog shader chunks manually.

---

## 2. Renderer Deep Dive

### 2.1 WebGLRenderer

**Source:** https://threejs.org/docs/#api/en/renderers/WebGLRenderer

#### Constructor Parameters

```typescript
new WebGLRenderer(parameters?: {
  canvas?: HTMLCanvasElement,                    // Existing canvas element. Default: creates new
  context?: WebGLRenderingContext,               // Existing WebGL context. Default: creates new
  precision?: 'highp' | 'mediump' | 'lowp',     // Shader precision. Default: 'highp'
  alpha?: boolean,                               // Transparent canvas. Default: false
  premultipliedAlpha?: boolean,                  // Default: true
  antialias?: boolean,                           // MSAA. Default: false
  stencil?: boolean,                             // Stencil buffer. Default: true
  preserveDrawingBuffer?: boolean,               // For screenshots. Default: false
  powerPreference?: 'high-performance' | 'low-power' | 'default', // GPU selection hint. Default: 'default'
  failIfMajorPerformanceCaveat?: boolean,        // Fail if software renderer. Default: false
  depth?: boolean,                               // Depth buffer. Default: true
  logarithmicDepthBuffer?: boolean               // Fix Z-fighting for large scenes. Default: false
})
```

**Critical notes:**
- `antialias` CANNOT be changed after construction. ALWAYS decide at creation time.
- `preserveDrawingBuffer: true` is required for `canvas.toDataURL()` / `canvas.toBlob()`. NEVER enable it in production rendering loops — it has a performance cost.
- `logarithmicDepthBuffer: true` solves Z-fighting in large-scale scenes (architectural, geographic) but reduces depth precision for very near objects and has a per-fragment performance cost.
- `alpha: true` is required when compositing Three.js over HTML/CSS content.

#### Key Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `domElement` | `HTMLCanvasElement` | — | The canvas element. ALWAYS append this to the DOM |
| `autoClear` | `boolean` | `true` | Clear before each render |
| `autoClearColor` | `boolean` | `true` | Clear color buffer |
| `autoClearDepth` | `boolean` | `true` | Clear depth buffer |
| `autoClearStencil` | `boolean` | `true` | Clear stencil buffer |
| `sortObjects` | `boolean` | `true` | Sort opaque (front-to-back) and transparent (back-to-front) objects |
| `clippingPlanes` | `Plane[]` | `[]` | Global clipping planes |
| `localClippingEnabled` | `boolean` | `false` | MUST set to `true` to enable per-material clipping planes |
| `outputColorSpace` | `string` | `SRGBColorSpace` | Output color space (r160+) |
| `toneMapping` | `number` | `NoToneMapping` | Tone mapping algorithm |
| `toneMappingExposure` | `number` | `1` | Exposure level for tone mapping |
| `shadowMap` | `WebGLShadowMap` | — | Shadow map configuration object |
| `info` | `object` | — | Render statistics (calls, triangles, points, lines, frame, memory) |
| `capabilities` | `object` | — | WebGL capabilities (maxTextures, maxVertexTextures, precision, etc.) |
| `xr` | `WebXRManager` | — | WebXR session manager |

#### Tone Mapping Options

| Constant | Description | Use Case |
|----------|-------------|----------|
| `NoToneMapping` | No tone mapping (default) | Non-photorealistic rendering, UI overlays |
| `LinearToneMapping` | Simple linear scaling | Basic HDR clamping |
| `ReinhardToneMapping` | Classic Reinhard operator | General-purpose, preserves color |
| `CineonToneMapping` | Cineon film stock emulation | Cinematic look |
| `ACESFilmicToneMapping` | Academy Color Encoding System | Photorealistic scenes, PBR workflows. Most popular choice |
| `AgXToneMapping` | AgX tone mapper | More accurate than ACES for saturated colors. Introduced in r160+ |
| `NeutralToneMapping` | Neutral, minimal color shift | When color accuracy is paramount |

**Recommendation:** For PBR workflows, ALWAYS use `ACESFilmicToneMapping` or `AgXToneMapping`. `AgXToneMapping` handles saturated colors better and avoids the "ACES hue shift" problem with bright blues and reds.

#### Color Management (r160+)

Three.js r160+ uses a proper color management pipeline:

- `renderer.outputColorSpace = THREE.SRGBColorSpace` (default): Final output is gamma-corrected for display
- `renderer.outputColorSpace = THREE.LinearSRGBColorSpace`: Linear output (for post-processing chains)
- `texture.colorSpace = THREE.SRGBColorSpace`: For color/albedo textures (sRGB source images)
- `texture.colorSpace = THREE.LinearSRGBColorSpace`: For data textures (normal maps, roughness, metalness, AO)

**Rule:** Color textures (diffuse, emissive) are ALWAYS `SRGBColorSpace`. Data textures (normal, roughness, metalness, displacement, AO) are ALWAYS `LinearSRGBColorSpace`. Getting this wrong produces washed-out or over-saturated renders.

#### Shadow Map Configuration

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap; // or BasicShadowMap, PCFShadowMap, VSMShadowMap
```

| Type | Quality | Performance | Notes |
|------|---------|-------------|-------|
| `BasicShadowMap` | Low (hard edges) | Fastest | No filtering |
| `PCFShadowMap` | Medium | Medium | Default. Percentage-Closer Filtering |
| `PCFSoftShadowMap` | High (soft edges) | Slower | Bilinear PCF. Most popular |
| `VSMShadowMap` | High (very soft) | Slowest | Variance Shadow Maps. Can have light bleeding artifacts |

**Anti-pattern:** NEVER change `shadowMap.type` after the first render — it requires shader recompilation. Set it once at initialization.

#### Render Targets (Off-Screen Rendering)

```typescript
new WebGLRenderTarget(width: number, height: number, options?: {
  minFilter?: number,         // Default: LinearFilter
  magFilter?: number,         // Default: LinearFilter
  format?: number,            // Default: RGBAFormat
  type?: number,              // Default: UnsignedByteType (use HalfFloatType for HDR)
  stencilBuffer?: boolean,    // Default: false
  depthBuffer?: boolean,      // Default: true
  samples?: number,           // MSAA samples. Default: 0
  colorSpace?: string,        // Default: '' (NoColorSpace)
  depthTexture?: DepthTexture // Attach depth texture for shadow/depth reads
})
```

Usage pattern:
```javascript
const renderTarget = new THREE.WebGLRenderTarget(1024, 1024);
renderer.setRenderTarget(renderTarget);
renderer.render(scene, camera);
renderer.setRenderTarget(null); // restore default framebuffer
// renderTarget.texture is now usable as a regular Texture
```

`WebGLCubeRenderTarget` is used with `CubeCamera` for dynamic environment maps.

#### Key Methods

```typescript
render(scene: Scene, camera: Camera): void
setSize(width: number, height: number, updateStyle?: boolean): void  // updateStyle default: true
setPixelRatio(value: number): void                    // ALWAYS use window.devicePixelRatio
setClearColor(color: Color | string | number, alpha?: number): void
getClearColor(target: Color): Color
getClearAlpha(): number
clear(color?: boolean, depth?: boolean, stencil?: boolean): void
setRenderTarget(renderTarget: WebGLRenderTarget | null, activeCubeFace?: number, activeMipmapLevel?: number): void
setAnimationLoop(callback: ((time: DOMHighResTimeStamp) => void) | null): void
compile(scene: Scene, camera: Camera): Set<Material>            // synchronous shader compilation
compileAsync(scene: Scene, camera: Camera): Promise<void>       // async shader compilation (r160+)
dispose(): void                                                  // ALWAYS call on cleanup
getContext(): WebGLRenderingContext
getPixelRatio(): number
getSize(target: Vector2): Vector2
setViewport(x: number, y: number, width: number, height: number): void
setScissor(x: number, y: number, width: number, height: number): void
setScissorTest(boolean: boolean): void
readRenderTargetPixels(renderTarget, x, y, width, height, buffer): void
```

**`setAnimationLoop` vs `requestAnimationFrame`:** ALWAYS use `renderer.setAnimationLoop()` in Three.js r160+. It handles WebXR sessions automatically and provides the correct timestamp. Using raw `requestAnimationFrame` bypasses XR and requires manual frame management.

**`compileAsync`:** Pre-compiles all shaders in the scene asynchronously. Call this after loading assets but before the first visible render to prevent frame drops from just-in-time shader compilation. Returns a Promise that resolves when all shaders are compiled.

#### Clipping Planes

Global clipping:
```javascript
renderer.clippingPlanes = [new THREE.Plane(new THREE.Vector3(0, -1, 0), 1)];
```

Per-material clipping (MUST enable `localClippingEnabled`):
```javascript
renderer.localClippingEnabled = true;
material.clippingPlanes = [plane];
material.clipIntersection = false; // false = union (default), true = intersection
material.clipShadows = true;       // also clip shadow geometry
```

---

## 3. Camera System

### 3.1 PerspectiveCamera

**Source:** https://threejs.org/docs/#api/en/cameras/PerspectiveCamera

```typescript
new PerspectiveCamera(
  fov?: number,     // Vertical field of view in degrees. Default: 50
  aspect?: number,  // Width / height ratio. Default: 1
  near?: number,    // Near clipping plane. Default: 0.1
  far?: number      // Far clipping plane. Default: 2000
)
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `fov` | `number` | `50` | Vertical FOV in degrees (NOT radians) |
| `aspect` | `number` | `1` | Aspect ratio (width / height) |
| `near` | `number` | `0.1` | Near clipping plane distance |
| `far` | `number` | `2000` | Far clipping plane distance |
| `zoom` | `number` | `1` | Zoom factor. Values > 1 zoom in, < 1 zoom out |
| `filmGauge` | `number` | `35` | Film gauge in mm. Used with `filmOffset` for camera effects |
| `filmOffset` | `number` | `0` | Horizontal off-center offset in mm |
| `focus` | `number` | `10` | Object distance for stereo rendering |
| `view` | `object \| null` | `null` | View sub-frustum (set by `setViewOffset()`) |

**Critical rule:** After modifying `fov`, `aspect`, `near`, `far`, or `zoom`, you MUST call `camera.updateProjectionMatrix()`. Forgetting this is the number-one camera bug.

```javascript
// CORRECT: resize handler
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

**Methods:**

```typescript
updateProjectionMatrix(): void      // MUST call after changing fov/aspect/near/far/zoom
setViewOffset(fullWidth: number, fullHeight: number, x: number, y: number, width: number, height: number): void
clearViewOffset(): void
getEffectiveFOV(): number           // Returns actual FOV accounting for zoom
getFilmWidth(): number              // Effective film width in mm
getFilmHeight(): number             // Effective film height in mm
toJSON(meta?: object): object
```

**FOV conventions:** Three.js uses VERTICAL FOV. Human vision is approximately 60-75 degrees vertical. Standard values: 50-75 for general 3D, 90+ for FPS-style views, 10-20 for telephoto effects.

**Near/far best practices:**
- ALWAYS keep the ratio `far/near` as small as possible (under 10,000) to maximize depth buffer precision
- NEVER use `near: 0` — it causes Z-fighting everywhere
- For architectural scenes: `near: 0.1, far: 1000` (meters)
- For geographic scenes: use `logarithmicDepthBuffer: true` on the renderer

### 3.2 OrthographicCamera

**Source:** https://threejs.org/docs/#api/en/cameras/OrthographicCamera

```typescript
new OrthographicCamera(
  left: number,     // Left plane of frustum
  right: number,    // Right plane of frustum
  top: number,      // Top plane of frustum
  bottom: number,   // Bottom plane of frustum
  near?: number,    // Default: 0.1
  far?: number      // Default: 2000
)
```

No perspective distortion. Objects appear the same size regardless of distance. ALWAYS used for: 2D overlays, isometric views, technical/CAD views, shadow camera frustums.

```javascript
// 2D-style setup that matches screen pixels
const camera = new THREE.OrthographicCamera(
  -window.innerWidth / 2,   // left
  window.innerWidth / 2,    // right
  window.innerHeight / 2,   // top
  -window.innerHeight / 2,  // bottom
  0.1, 1000
);

// Resize handler — MUST update all six frustum values + call updateProjectionMatrix
window.addEventListener('resize', () => {
  camera.left = -window.innerWidth / 2;
  camera.right = window.innerWidth / 2;
  camera.top = window.innerHeight / 2;
  camera.bottom = -window.innerHeight / 2;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

`zoom` works here too — default `1`, higher values zoom in by uniformly shrinking the frustum.

### 3.3 ArrayCamera

**Source:** https://threejs.org/docs/#api/en/cameras/ArrayCamera

```typescript
new ArrayCamera(cameras?: PerspectiveCamera[])
```

Renders multiple viewports in a single render call. Each sub-camera has its own viewport defined via `camera.viewport = new Vector4(x, y, width, height)`. Used for split-screen, VR stereo, and multi-view rendering.

### 3.4 CubeCamera

**Source:** https://threejs.org/docs/#api/en/cameras/CubeCamera

```typescript
new CubeCamera(near: number, far: number, renderTarget: WebGLCubeRenderTarget)
```

Renders the scene from all 6 directions into a cube render target. Used for dynamic environment maps and real-time reflections.

```javascript
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256);
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);

// In animation loop (expensive — do not call every frame unless needed)
cubeCamera.update(renderer, scene);
material.envMap = cubeRenderTarget.texture;
```

**Performance:** `cubeCamera.update()` renders the scene 6 times. NEVER call every frame for static environments. Use a lower resolution (128-512) and update only when the environment changes.

### 3.5 Camera Layers for Selective Rendering

Cameras have a `layers` property. Only objects whose layers overlap with the camera's layers are rendered:

```javascript
camera.layers.enable(1);        // camera now sees layer 0 AND layer 1
helperMesh.layers.set(1);       // helper is ONLY on layer 1
productionCamera.layers.set(0); // production camera sees ONLY layer 0
```

---

## 4. Raycaster

**Source:** https://threejs.org/docs/#api/en/core/Raycaster

### 4.1 Constructor and Properties

```typescript
new Raycaster(
  origin?: Vector3,      // Default: (0, 0, 0)
  direction?: Vector3,   // Default: (0, 0, -1) — MUST be normalized
  near?: number,         // Default: 0
  far?: number           // Default: Infinity
)
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `ray` | `Ray` | — | The underlying Ray object (origin + direction) |
| `near` | `number` | `0` | Minimum intersection distance |
| `far` | `number` | `Infinity` | Maximum intersection distance |
| `camera` | `Camera` | — | Required for raycasting against `Sprite` objects |
| `layers` | `Layers` | layer 0 | Layer mask — only objects on matching layers are tested |
| `params` | `object` | see below | Per-type intersection parameters |

```javascript
raycaster.params = {
  Mesh: {},
  Line: { threshold: 1 },    // Distance threshold for line hits (world units)
  LOD: {},
  Points: { threshold: 1 },  // Distance threshold for point hits (world units)
  Sprite: {}
};
```

### 4.2 Core Methods

```typescript
set(origin: Vector3, direction: Vector3): void
```
Manually sets the ray origin and direction. Direction MUST be normalized.

```typescript
setFromCamera(coords: Vector2, camera: Camera): void
```
Sets the ray from normalized device coordinates (-1 to +1 on both axes) and a camera. This is the standard mouse-picking method.

```javascript
const mouse = new THREE.Vector2();
// Convert DOM coordinates to NDC
mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
raycaster.setFromCamera(mouse, camera);
```

```typescript
intersectObject(object: Object3D, recursive?: boolean, optionalTarget?: Intersection[]): Intersection[]
intersectObjects(objects: Object3D[], recursive?: boolean, optionalTarget?: Intersection[]): Intersection[]
```

- `recursive` (default: `true`): When `true`, tests against all descendants
- `optionalTarget`: Reusable array to avoid allocation. ALWAYS pass a reusable array in animation loops
- Results are ALWAYS sorted by distance (nearest first)

### 4.3 Intersection Object Format

Each intersection in the returned array:

```typescript
interface Intersection {
  distance: number;           // Distance from ray origin to hit point
  point: Vector3;             // Hit point in world space
  face: Face | null;          // Hit face ({a, b, c} vertex indices + normal)
  faceIndex: number;          // Index of the hit face in the geometry
  object: Object3D;           // The intersected object
  uv?: Vector2;               // UV coordinates at intersection point
  uv1?: Vector2;              // Second UV set (if available)
  normal?: Vector3;           // Interpolated surface normal at hit point
  instanceId?: number;        // Instance index for InstancedMesh
}
```

### 4.4 Performance Optimization

1. **Layer filtering:** Set `raycaster.layers` to test only relevant layers. Objects on non-matching layers are skipped entirely.
2. **`recursive: false`:** Pass `false` when you know the exact objects to test (flat array of meshes) to avoid tree traversal.
3. **Bounding sphere pre-check:** Three.js automatically tests bounding spheres before testing triangles. Ensure `geometry.boundingSphere` is computed (call `geometry.computeBoundingSphere()` if needed).
4. **Reusable array:** Pass `optionalTarget` to avoid array allocation per frame.
5. **Throttle raycasting:** For mouse move events, NEVER raycast on every `mousemove`. Use `requestAnimationFrame` or throttle to 30-60fps.

### 4.5 InstancedMesh Raycasting

When raycasting against `InstancedMesh`, the returned intersection includes `instanceId`:

```javascript
const intersects = raycaster.intersectObject(instancedMesh);
if (intersects.length > 0) {
  const instanceId = intersects[0].instanceId;
  // Use instanceId to identify which instance was hit
  const matrix = new THREE.Matrix4();
  instancedMesh.getMatrixAt(instanceId, matrix);
}
```

### 4.6 Common Patterns

**Mouse picking:**
```javascript
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const intersections = [];

canvas.addEventListener('click', (event) => {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);
  raycaster.intersectObjects(selectableObjects, false, intersections);
  if (intersections.length > 0) {
    handleSelection(intersections[0].object);
  }
  intersections.length = 0; // clear for reuse
});
```

---

## 5. Math Utilities Deep Dive

### 5.1 Vector3

**Source:** https://threejs.org/docs/#api/en/math/Vector3

Vector3 is the most-used math class. All methods return `this` (for chaining) unless documented otherwise.

**Arithmetic:**
- `add(v)`, `addScalar(s)`, `addVectors(a, b)` — addition
- `sub(v)`, `subScalar(s)`, `subVectors(a, b)` — subtraction
- `multiply(v)`, `multiplyScalar(s)`, `multiplyVectors(a, b)` — component-wise multiplication
- `divide(v)`, `divideScalar(s)` — component-wise division
- `negate()` — negates all components

**Geometric:**
- `dot(v): number` — dot product
- `cross(v)`, `crossVectors(a, b)` — cross product (modifies `this`)
- `length(): number`, `lengthSq(): number` — magnitude
- `normalize()` — sets length to 1
- `setLength(length)` — scales to specific length
- `reflect(normal)` — reflects vector off a surface
- `project(camera)` / `unproject(camera)` — project to/from NDC

**Distance:**
- `distanceTo(v): number`, `distanceToSquared(v): number` — distance between points
- `manhattanDistance(v): number`, `manhattanLength(): number` — L1 distance

**Interpolation:**
- `lerp(v, alpha)` — linear interpolation
- `lerpVectors(v1, v2, t)` — lerp between two vectors, stores result in `this`
- `clamp(min, max)` — clamp component-wise
- `clampLength(min, max)` — clamp vector length
- `clampScalar(min, max)` — clamp all components to scalar range

**Transform:**
- `applyMatrix3(m)`, `applyMatrix4(m)` — matrix transformation
- `applyQuaternion(q)` — quaternion rotation
- `applyAxisAngle(axis, angle)` — rotate around axis
- `applyEuler(euler)` — apply Euler rotation

**Conversion:**
- `setFromMatrixPosition(m)` — extract translation from Matrix4
- `setFromMatrixScale(m)` — extract scale from Matrix4
- `setFromMatrixColumn(m, index)` — extract column vector
- `setFromSpherical(s)` / `setFromSphericalCoords(r, phi, theta)` — spherical to Cartesian
- `setFromCylindrical(c)` / `setFromCylindricalCoords(r, theta, y)` — cylindrical to Cartesian

**Utility:**
- `set(x, y, z)`, `copy(v)`, `clone()` — assignment
- `equals(v): boolean` — exact equality
- `toArray(array?, offset?)`, `fromArray(array, offset?)` — array conversion
- `min(v)`, `max(v)` — component-wise min/max
- `floor()`, `ceil()`, `round()`, `roundToZero()` — rounding
- `random()` — random values in [0, 1) per component

### 5.2 Matrix4

**Source:** https://threejs.org/docs/#api/en/math/Matrix4

Column-major storage (WebGL convention). The `elements` array of 16 floats:

```
elements[0]  elements[4]  elements[8]   elements[12]   // column 0, 1, 2, 3
elements[1]  elements[5]  elements[9]   elements[13]   // (when read as rows)
elements[2]  elements[6]  elements[10]  elements[14]
elements[3]  elements[7]  elements[11]  elements[15]
```

Translation is stored in elements `[12, 13, 14]` (last column).

**Composition:**
- `compose(position, quaternion, scale)` — build TRS matrix
- `decompose(position, quaternion, scale)` — extract TRS components

**Multiplication:**
- `multiply(m)` — `this = this * m` (right-multiply). In Three.js, transformations are applied right-to-left: `M1.multiply(M2)` means M2 is applied first, then M1
- `premultiply(m)` — `this = m * this` (left-multiply)
- `multiplyMatrices(a, b)` — `this = a * b`

**Factory methods:**
- `makeTranslation(x, y, z)`, `makeScale(x, y, z)`
- `makeRotationX(theta)`, `makeRotationY(theta)`, `makeRotationZ(theta)`
- `makeRotationAxis(axis, angle)`, `makeRotationFromEuler(euler)`, `makeRotationFromQuaternion(q)`
- `lookAt(eye, target, up)` — view matrix
- `makeBasis(xAxis, yAxis, zAxis)` — basis matrix

**Operations:**
- `determinant(): number`, `invert()`, `transpose()`
- `extractBasis(x, y, z)`, `extractRotation(m)`
- `setPosition(v | x, y, z)` — set translation component only
- `identity()`, `clone()`, `copy(m)`, `equals(m): boolean`
- `toArray(array?, offset?)`, `fromArray(array, offset?)`

### 5.3 Quaternion

**Source:** https://threejs.org/docs/#api/en/math/Quaternion

Quaternions represent rotations without gimbal lock. ALWAYS prefer quaternions over Euler angles for interpolated animations and compound rotations.

```typescript
new Quaternion(x?: number, y?: number, z?: number, w?: number) // Default: (0, 0, 0, 1) = identity
```

**Setters:**
- `setFromAxisAngle(axis: Vector3, angle: number)` — rotation around axis
- `setFromEuler(euler: Euler)` — convert from Euler
- `setFromRotationMatrix(m: Matrix4)` — extract rotation from matrix
- `setFromUnitVectors(vFrom: Vector3, vTo: Vector3)` — rotation that maps vFrom to vTo

**Operations:**
- `multiply(q)` — `this = this * q`. Rotation order: q is applied first, then `this`
- `premultiply(q)` — `this = q * this`
- `slerp(qb, t)` — spherical linear interpolation (smooth rotation blending)
- `slerpQuaternions(qa, qb, t)` — slerp between two quaternions, stores in `this`
- `rotateTowards(q, step)` — rotate toward target by at most `step` radians
- `conjugate()` — negate x, y, z (for unit quaternions, conjugate = inverse)
- `invert()` — full inverse (conjugate / lengthSq)
- `normalize()` — normalize to unit quaternion
- `dot(q): number`, `length(): number`, `lengthSq(): number`
- `identity()` — set to (0, 0, 0, 1)
- `equals(q): boolean`, `clone()`, `copy(q)`

**Anti-pattern:** NEVER manually set `x`, `y`, `z`, `w` values unless you understand quaternion math. ALWAYS use `setFromAxisAngle()`, `setFromEuler()`, or `slerp()`.

### 5.4 Euler

**Source:** https://threejs.org/docs/#api/en/math/Euler

```typescript
new Euler(x?: number, y?: number, z?: number, order?: string)
// Default: (0, 0, 0, 'XYZ')
```

Rotation orders: `'XYZ'`, `'YXZ'`, `'ZXY'`, `'ZYX'`, `'YZX'`, `'XZY'`

**Gimbal lock:** Occurs when two axes align, losing one degree of rotational freedom. In `'XYZ'` order, gimbal lock occurs when Y rotation = +/- 90 degrees (pi/2 radians). Symptoms: unexpected snapping, rotation around the wrong axis.

**Methods:**
- `setFromRotationMatrix(m: Matrix4): this`
- `setFromQuaternion(q: Quaternion, order?: string): this`
- `setFromVector3(v: Vector3, order?: string): this`
- `reorder(newOrder: string): this` — change order while preserving the rotation
- `equals(euler: Euler): boolean`
- `toArray(array?, offset?): [x, y, z, order]`

**Rule:** For animations that interpolate rotation, ALWAYS convert to Quaternion first, slerp, then convert back if needed. NEVER lerp Euler angles directly — it produces incorrect paths and can trigger gimbal lock.

### 5.5 Color

**Source:** https://threejs.org/docs/#api/en/math/Color

```typescript
// Constructor overloads
new Color()                                    // white (1, 1, 1)
new Color(0xff0000)                           // hex integer
new Color('red')                              // CSS color name
new Color('rgb(255, 0, 0)')                   // CSS rgb string
new Color('#ff0000')                          // CSS hex string
new Color('hsl(0, 100%, 50%)')                // CSS hsl string
new Color(1.0, 0.0, 0.0)                     // RGB floats (0-1)
new Color(existingColor)                      // copy
```

Properties: `r`, `g`, `b` (all `number`, range 0-1).

**Setters:** `set(value)`, `setHex(hex)`, `setRGB(r, g, b)`, `setHSL(h, s, l)`, `setStyle(cssString)`, `setColorName(name)`

**Getters:** `getHex(): number`, `getHexString(): string`, `getHSL(target): {h, s, l}`, `getRGB(target): {r, g, b}`, `getStyle(): string`

**Interpolation:**
- `lerp(color, alpha)` — linear RGB interpolation
- `lerpHSL(color, alpha)` — HSL interpolation (better for perceptually uniform transitions)
- `lerpColors(color1, color2, alpha)` — static lerp between two colors

**Color space conversion:**
- `convertSRGBToLinear()` — sRGB to linear (for shader input)
- `convertLinearToSRGB()` — linear to sRGB (for display)

**Anti-pattern:** NEVER use `lerp()` for smooth color transitions through different hues — it produces muddy intermediate colors. ALWAYS use `lerpHSL()` for hue-shifting animations.

### 5.6 MathUtils

**Source:** https://threejs.org/docs/#api/en/math/MathUtils

Static utility methods (NEVER instantiate — access via `THREE.MathUtils.method()`):

| Method | Signature | Description |
|--------|-----------|-------------|
| `degToRad` | `(degrees: number): number` | Convert degrees to radians |
| `radToDeg` | `(radians: number): number` | Convert radians to degrees |
| `clamp` | `(value: number, min: number, max: number): number` | Clamp value to range |
| `lerp` | `(x: number, y: number, t: number): number` | Linear interpolation |
| `inverseLerp` | `(x: number, y: number, value: number): number` | Inverse of lerp (find t) |
| `mapLinear` | `(x: number, a1: number, a2: number, b1: number, b2: number): number` | Map from range [a1,a2] to [b1,b2] |
| `smoothstep` | `(x: number, min: number, max: number): number` | Hermite smoothstep (ease in-out) |
| `smootherstep` | `(x: number, min: number, max: number): number` | Ken Perlin's improved smoothstep |
| `randFloat` | `(low: number, high: number): number` | Random float in range |
| `randFloatSpread` | `(range: number): number` | Random float in [-range/2, range/2] |
| `randInt` | `(low: number, high: number): number` | Random integer in range |
| `generateUUID` | `(): string` | Generate RFC 4122 v4 UUID |
| `seededRandom` | `(seed?: number): number` | Seeded pseudo-random number |
| `pingpong` | `(x: number, length?: number): number` | Ping-pong value (0 to length and back) |
| `damp` | `(x: number, y: number, lambda: number, dt: number): number` | Frame-rate-independent damping |
| `euclideanModulo` | `(n: number, m: number): number` | Always-positive modulo |
| `isPowerOfTwo` | `(value: number): boolean` | Check power of two |
| `ceilPowerOfTwo` | `(value: number): number` | Round up to next power of two |
| `floorPowerOfTwo` | `(value: number): number` | Round down to previous power of two |

`damp()` is particularly useful for smooth camera follow and UI animations — it produces exponential decay that is frame-rate independent, unlike naive lerp in an animation loop.

### 5.7 Bounding Volumes

**Box3:**
```typescript
new Box3(min?: Vector3, max?: Vector3) // Default: +Infinity min, -Infinity max (empty)
```
Key methods: `setFromObject(object)`, `expandByPoint(point)`, `containsPoint(point): boolean`, `intersectsBox(box): boolean`, `intersectsSphere(sphere): boolean`, `getCenter(target): Vector3`, `getSize(target): Vector3`, `getBoundingSphere(target): Sphere`, `clampPoint(point, target): Vector3`, `distanceToPoint(point): number`, `union(box)`, `intersect(box)`.

**Sphere:**
```typescript
new Sphere(center?: Vector3, radius?: number)
```
Key methods: `containsPoint(point): boolean`, `intersectsBox(box): boolean`, `intersectsSphere(sphere): boolean`, `distanceToPoint(point): number`, `clampPoint(point, target): Vector3`.

**Plane:**
```typescript
new Plane(normal?: Vector3, constant?: number)
```
Key methods: `distanceToPoint(point): number`, `projectPoint(point, target): Vector3`, `intersectLine(line, target): Vector3 | null`.

**Ray:**
```typescript
new Ray(origin?: Vector3, direction?: Vector3)
```
Key methods: `intersectBox(box, target): Vector3 | null`, `intersectSphere(sphere, target): Vector3 | null`, `intersectPlane(plane, target): Vector3 | null`, `distanceToPoint(point): number`.

**Frustum:**
```typescript
new Frustum(...planes: Plane[]) // 6 planes
```
Key methods: `setFromProjectionMatrix(m: Matrix4)`, `containsPoint(point): boolean`, `intersectsObject(object): boolean`, `intersectsBox(box): boolean`, `intersectsSphere(sphere): boolean`.

### 5.8 Coordinate System

Three.js uses a **right-handed coordinate system** with **Y-up**:
- **X**: right
- **Y**: up
- **Z**: toward the viewer (out of the screen)

**Implication for imported models:**
- Blender uses Z-up. The glTF exporter handles conversion automatically.
- FBX files may arrive with Z-up. `FBXLoader` converts automatically.
- OBJ files have no standard. Check and rotate if needed.
- IFC files use Z-up. ALWAYS apply a -90 degree X rotation or use a loader that converts.

NEVER assume imported models match Three.js conventions — ALWAYS verify orientation after loading.

---

## 6. Disposal and Memory Management

### 6.1 What Requires Disposal

Three.js allocates GPU resources (buffers, textures, programs) that are NOT automatically garbage collected by JavaScript. The following object types MUST be manually disposed:

| Object Type | Method | GPU Resource Released |
|-------------|--------|---------------------|
| `BufferGeometry` | `geometry.dispose()` | Vertex/index buffers |
| `Material` (all types) | `material.dispose()` | Shader programs, uniforms |
| `Texture` (all types) | `texture.dispose()` | GPU texture memory |
| `WebGLRenderTarget` | `renderTarget.dispose()` | Framebuffer, textures |
| `WebGLRenderer` | `renderer.dispose()` | Entire WebGL context |
| `PMREMGenerator` | `pmremGenerator.dispose()` | Prefiltered env maps |
| Controls (all types) | `controls.dispose()` | Event listeners |

### 6.2 Disposal Patterns

**Single object disposal:**
```javascript
mesh.geometry.dispose();
mesh.material.dispose();
if (mesh.material.map) mesh.material.map.dispose();
if (mesh.material.normalMap) mesh.material.normalMap.dispose();
// ... repeat for all texture properties
```

**Complete material disposal (all texture maps):**
```javascript
function disposeMaterial(material) {
  const textureProperties = [
    'map', 'lightMap', 'bumpMap', 'normalMap', 'specularMap',
    'envMap', 'alphaMap', 'aoMap', 'displacementMap',
    'emissiveMap', 'gradientMap', 'metalnessMap', 'roughnessMap',
    'clearcoatMap', 'clearcoatNormalMap', 'clearcoatRoughnessMap',
    'transmissionMap', 'thicknessMap', 'sheenColorMap', 'sheenRoughnessMap'
  ];
  for (const prop of textureProperties) {
    if (material[prop]) material[prop].dispose();
  }
  material.dispose();
}
```

**Scene cleanup (full traversal):**
```javascript
function disposeScene(scene) {
  scene.traverse((object) => {
    if (object.geometry) {
      object.geometry.dispose();
    }
    if (object.material) {
      if (Array.isArray(object.material)) {
        object.material.forEach(disposeMaterial);
      } else {
        disposeMaterial(object.material);
      }
    }
  });
  scene.clear();
}
```

### 6.3 Event-Based Disposal

Geometries, materials, and textures fire a `dispose` event when `.dispose()` is called:

```javascript
texture.addEventListener('dispose', () => {
  console.log('Texture GPU resources released');
  // Clean up any associated application state
});
```

The renderer listens to these events to clean up internal caches (shader programs, buffer references). This is how Three.js knows to release GPU resources.

### 6.4 When to Dispose vs. When to Reuse

**ALWAYS dispose when:**
- Removing objects permanently from the scene
- Switching between completely different scenes
- Unloading loaded models
- Application/component unmount (React, Vue, etc.)

**NEVER dispose when:**
- Temporarily hiding objects (use `visible = false`)
- Objects will be re-added to the scene later
- Multiple meshes share the same geometry/material (dispose only when ALL users are done)

### 6.5 Common Memory Leak Patterns

1. **Forgetting to dispose textures on material replacement:**
   ```javascript
   // LEAK: old texture stays in GPU memory
   material.map = newTexture;

   // CORRECT:
   if (material.map) material.map.dispose();
   material.map = newTexture;
   ```

2. **Creating geometry/material in animation loops:**
   ```javascript
   // LEAK: new geometry every frame, never disposed
   function animate() {
     mesh.geometry = new THREE.BoxGeometry(1, 1, 1); // NEVER do this
   }
   ```

3. **Not disposing render targets:**
   ```javascript
   // LEAK: old render target GPU memory not freed
   const newTarget = new THREE.WebGLRenderTarget(w, h);
   // MUST dispose old target first
   ```

4. **Orphaned event listeners from controls:**
   ```javascript
   // LEAK: event listeners not removed
   // ALWAYS call controls.dispose() on cleanup
   ```

5. **Shared material/geometry disposal race condition:**
   ```javascript
   // BUG: disposing shared geometry kills it for all meshes
   const sharedGeo = new THREE.BoxGeometry(1, 1, 1);
   const mesh1 = new THREE.Mesh(sharedGeo, mat);
   const mesh2 = new THREE.Mesh(sharedGeo, mat);
   // NEVER dispose sharedGeo until BOTH meshes are removed
   ```

### 6.6 Renderer Disposal

```javascript
renderer.dispose();
// After this, the WebGL context is lost
// NEVER call render() after dispose()
// The canvas DOM element is NOT removed — remove it manually if needed
renderer.domElement.remove();
```

### 6.7 Monitoring Memory

Use `renderer.info` to monitor GPU resource allocation:

```javascript
console.log(renderer.info.memory);
// { geometries: number, textures: number }
console.log(renderer.info.render);
// { calls: number, triangles: number, points: number, lines: number, frame: number }
```

If `geometries` or `textures` count grows continuously, you have a memory leak. ALWAYS monitor these values during development.

---

## Sources

All API references in this document were verified against the official Three.js documentation:

- Object3D: https://threejs.org/docs/#api/en/core/Object3D
- Scene: https://threejs.org/docs/#api/en/scenes/Scene
- Fog: https://threejs.org/docs/#api/en/scenes/Fog
- WebGLRenderer: https://threejs.org/docs/#api/en/renderers/WebGLRenderer
- WebGLRenderTarget: https://threejs.org/docs/#api/en/renderers/WebGLRenderTarget
- PerspectiveCamera: https://threejs.org/docs/#api/en/cameras/PerspectiveCamera
- OrthographicCamera: https://threejs.org/docs/#api/en/cameras/OrthographicCamera
- ArrayCamera: https://threejs.org/docs/#api/en/cameras/ArrayCamera
- CubeCamera: https://threejs.org/docs/#api/en/cameras/CubeCamera
- Raycaster: https://threejs.org/docs/#api/en/core/Raycaster
- Layers: https://threejs.org/docs/#api/en/core/Layers
- Vector3: https://threejs.org/docs/#api/en/math/Vector3
- Matrix4: https://threejs.org/docs/#api/en/math/Matrix4
- Quaternion: https://threejs.org/docs/#api/en/math/Quaternion
- Euler: https://threejs.org/docs/#api/en/math/Euler
- Color: https://threejs.org/docs/#api/en/math/Color
- MathUtils: https://threejs.org/docs/#api/en/math/MathUtils
- Box3: https://threejs.org/docs/#api/en/math/Box3
- Disposal: https://threejs.org/docs/#manual/en/introduction/How-to-dispose-of-objects

Target Three.js version: r160+ (ES modules, MIT license).
