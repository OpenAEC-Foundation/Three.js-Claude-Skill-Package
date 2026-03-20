# B1 Research: Three.js Core Concepts — Landscape Mapping

> Research date: 2026-03-20
> Sources: Official Three.js documentation (threejs.org/docs), r160+
> Scope: Scene Graph, Renderer, Camera, Math Utilities

---

## 1. Scene Graph Architecture

### 1.1 Class Hierarchy Overview

Three.js uses a tree-based scene graph where every visible and invisible object descends from `Object3D`. The inheritance chain is:

```
EventDispatcher
  └── Object3D
        ├── Scene
        ├── Group
        ├── Mesh
        │     ├── SkinnedMesh
        │     └── InstancedMesh
        ├── Line
        │     ├── LineLoop
        │     └── LineSegments
        ├── Points
        ├── Sprite
        ├── Bone
        ├── Camera (abstract)
        │     ├── PerspectiveCamera
        │     ├── OrthographicCamera
        │     ├── ArrayCamera
        │     ├── CubeCamera
        │     └── StereoCamera
        └── Light (abstract)
              ├── AmbientLight
              ├── DirectionalLight
              ├── PointLight
              ├── SpotLight
              ├── HemisphereLight
              ├── RectAreaLight
              └── LightProbe
```

Every node in this tree is an `Object3D`. The `Scene` itself is an `Object3D`, which means it can be nested, transformed, and traversed like any other node.

### 1.2 Object3D — The Foundation Class

`Object3D` extends `EventDispatcher` and is the base class for ALL 3D objects in Three.js. It provides:

- **Transform system** (position, rotation, scale, matrices)
- **Parent-child hierarchy** (scene graph relationships)
- **Visibility and rendering control** (layers, frustum culling, render order)
- **Shadow configuration** (castShadow, receiveShadow)
- **User data storage** (userData object for application-specific data)

#### Constructor

```javascript
new THREE.Object3D()
```

No parameters. Creates an object at origin (0,0,0) with identity rotation and unit scale.

#### Transform Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `position` | `Vector3` | `(0, 0, 0)` | Local position relative to parent |
| `rotation` | `Euler` | `(0, 0, 0, 'XYZ')` | Local rotation as Euler angles (radians) |
| `quaternion` | `Quaternion` | `(0, 0, 0, 1)` | Local rotation as quaternion — synced with `rotation` |
| `scale` | `Vector3` | `(1, 1, 1)` | Local scale factors |
| `matrix` | `Matrix4` | Identity | Local transform matrix (composed from position/rotation/scale) |
| `matrixWorld` | `Matrix4` | Identity | World transform matrix (includes parent transforms) |
| `matrixAutoUpdate` | `Boolean` | `true` | When true, matrix is auto-recomputed from position/rotation/scale |
| `up` | `Vector3` | `(0, 1, 0)` | Up direction used by `lookAt()` |

**Critical pitfall — rotation vs quaternion sync:** The `rotation` (Euler) and `quaternion` properties are internally synchronized. Setting one ALWAYS updates the other automatically. NEVER manipulate both independently in the same frame — this leads to overwrite bugs.

**Critical pitfall — matrixAutoUpdate:** When `matrixAutoUpdate` is `true` (default), Three.js recomputes `matrix` from `position`, `rotation`, and `scale` every frame. If you set `matrix` directly, you MUST set `matrixAutoUpdate = false` first, otherwise your manual matrix will be overwritten.

#### Hierarchy Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `parent` | `Object3D \| null` | `null` | Parent in scene graph |
| `children` | `Array<Object3D>` | `[]` | Direct children |

#### Rendering Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `visible` | `Boolean` | `true` | Whether this object is rendered |
| `frustumCulled` | `Boolean` | `true` | Whether renderer skips objects outside camera frustum |
| `renderOrder` | `Number` | `0` | Override default render sorting |
| `layers` | `Layers` | Layer 0 | Bitmask for selective rendering (camera only sees objects on matching layers) |
| `castShadow` | `Boolean` | `false` | Whether object casts shadows |
| `receiveShadow` | `Boolean` | `false` | Whether object receives shadows |

#### Identity and Data

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `name` | `String` | `""` | Human-readable name for `getObjectByName()` |
| `id` | `Number` | Auto-increment | Internal unique ID |
| `uuid` | `String` | Auto-generated | Universally unique identifier |
| `userData` | `Object` | `{}` | Arbitrary user data (NEVER overwritten by Three.js) |

### 1.3 Scene Graph Methods

#### Adding and Removing Children

```javascript
// Add one or more objects as children
parent.add(child1, child2, child3): Object3D

// Remove specific children
parent.remove(child1, child2): Object3D

// Remove this object from its parent
child.removeFromParent(): Object3D

// Remove ALL children
parent.clear(): Object3D

// Attach child while preserving its world transform
// (re-parents the object without moving it visually)
newParent.attach(child): Object3D
```

**Critical pitfall — add() vs attach():** `add()` places the child in the parent's local space (child's world transform changes). `attach()` re-parents while PRESERVING the child's world transform by adjusting its local transform. Use `attach()` when transferring objects between groups without visual jumping.

**Critical pitfall — add() side effect:** Calling `parent.add(child)` ALWAYS removes `child` from its current parent first. An Object3D can only have ONE parent.

#### Traversal Methods

```javascript
// Execute callback on this object and ALL descendants (depth-first)
object.traverse(callback: (child: Object3D) => void): void

// Same as traverse, but skip invisible objects (visible === false)
object.traverseVisible(callback: (child: Object3D) => void): void

// Execute callback on all ancestors up to root
object.traverseAncestors(callback: (ancestor: Object3D) => void): void
```

**Performance note:** `traverse()` visits EVERY descendant. For large scenes (10,000+ objects), use `layers` or manual filtering to avoid full traversal on every frame.

#### Object Lookup

```javascript
// Find by name (first match, recursive)
scene.getObjectByName(name: String): Object3D | undefined

// Find by id (first match, recursive)
scene.getObjectById(id: Number): Object3D | undefined

// Find by arbitrary property
scene.getObjectByProperty(propName: String, value: any): Object3D | undefined
```

**Pitfall:** These methods perform recursive search. They return `undefined` (not `null`) if no match is found. ALWAYS check the return value before accessing properties.

#### Matrix Methods

```javascript
// Recompute local matrix from position/rotation/scale
object.updateMatrix(): void

// Recompute world matrix (and optionally children)
object.updateMatrixWorld(force: Boolean = false): void

// Selective update: control parent and children updates independently
object.updateWorldMatrix(updateParents: Boolean, updateChildren: Boolean): void
```

**When to call these manually:** The renderer calls `updateMatrixWorld()` automatically before each render. You only need manual calls when you need accurate world transforms BETWEEN renders (e.g., for physics or raycasting outside the render loop).

#### Coordinate Conversion

```javascript
// Convert world-space point to this object's local space
object.worldToLocal(vector: Vector3): Vector3

// Convert local-space point to world space
object.localToWorld(vector: Vector3): Vector3
```

**Pitfall:** Both methods MUTATE the input vector and return it. ALWAYS clone first if you need to preserve the original: `object.worldToLocal(point.clone())`.

#### Rotation and Translation

```javascript
// Look at a world-space point (rotates object)
object.lookAt(x: Number | Vector3, y?: Number, z?: Number): void

// Rotate around local or world axis
object.rotateOnAxis(axis: Vector3, angle: Number): Object3D
object.rotateOnWorldAxis(axis: Vector3, angle: Number): Object3D

// Convenience rotation (radians)
object.rotateX(angle: Number): Object3D
object.rotateY(angle: Number): Object3D
object.rotateZ(angle: Number): Object3D

// Translate along local axis
object.translateOnAxis(axis: Vector3, distance: Number): Object3D
object.translateX(distance: Number): Object3D
object.translateY(distance: Number): Object3D
object.translateZ(distance: Number): Object3D
```

**Pitfall — lookAt for non-cameras:** For meshes and other non-camera objects, `lookAt()` rotates the object so its local +Z axis points TOWARD the target. For cameras, the local -Z axis points toward the target. This inversion catches many developers off guard.

### 1.4 Scene Class

`Scene` extends `Object3D` and serves as the root container for all renderable content.

#### Constructor

```javascript
new THREE.Scene()
```

#### Scene-Specific Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `background` | `Color \| Texture \| CubeTexture \| null` | `null` | Scene background |
| `environment` | `Texture \| null` | `null` | Environment map for IBL (image-based lighting) |
| `fog` | `Fog \| FogExp2 \| null` | `null` | Fog effect |
| `backgroundBlurriness` | `Number` | `0` | Blur strength for background texture (0-1) |
| `backgroundIntensity` | `Number` | `1` | Background brightness multiplier |
| `backgroundRotation` | `Euler` | `(0,0,0)` | Background rotation |
| `environmentIntensity` | `Number` | `1` | Environment map brightness multiplier |
| `environmentRotation` | `Euler` | `(0,0,0)` | Environment map rotation |
| `overrideMaterial` | `Material \| null` | `null` | Forces ALL objects to use this material (useful for debugging) |
| `isScene` | `Boolean` | `true` | Type check flag |

**Version note (r160+):** `backgroundBlurriness`, `backgroundIntensity`, `backgroundRotation`, `environmentIntensity`, and `environmentRotation` are relatively recent additions. They provide fine-grained control over IBL without needing custom shaders.

### 1.5 Group Class

`Group` extends `Object3D` with no additional functionality. It exists purely as a semantic container for organizing scene hierarchies.

```javascript
const group = new THREE.Group();
group.add(mesh1, mesh2, mesh3);
scene.add(group);

// Transform all children together
group.position.set(0, 5, 0);
group.rotation.y = Math.PI / 4;
```

Properties: `isGroup: true`, `type: "Group"`.

### 1.6 Mesh Class

`Mesh` extends `Object3D` and combines a `BufferGeometry` with a `Material` to create a renderable object.

#### Constructor

```javascript
new THREE.Mesh(geometry?: BufferGeometry, material?: Material | Material[])
```

#### Mesh-Specific Properties

| Property | Type | Description |
|----------|------|-------------|
| `geometry` | `BufferGeometry` | Shape definition |
| `material` | `Material \| Material[]` | Surface appearance (single or multi-material) |
| `isMesh` | `Boolean` | Type check flag |
| `morphTargetInfluences` | `Float32Array` | Morph target weights |
| `morphTargetDictionary` | `Object` | Named morph target mapping |

#### Key Methods

```javascript
// Raycasting (used by Raycaster internally)
mesh.raycast(raycaster: Raycaster, intersects: Array): void

// Update morph targets from geometry
mesh.updateMorphTargets(): void

// Get vertex position (considering morph targets)
mesh.getVertexPosition(index: Number, target: Vector3): Vector3
```

**Multi-material pitfall:** When using `Material[]`, each material maps to a geometry group defined by `geometry.addGroup(start, count, materialIndex)`. Forgetting to define groups results in only the first material being applied.

---

## 2. Renderer System

### 2.1 WebGLRenderer

The primary renderer for Three.js. Uses WebGL 2.0 (with WebGL 1.0 fallback removed in r160+).

#### Constructor

```javascript
new THREE.WebGLRenderer(parameters?: {
  canvas?: HTMLCanvasElement,
  context?: WebGLRenderingContext,
  alpha?: boolean,           // default: false — transparent background
  antialias?: boolean,       // default: false — MSAA
  premultipliedAlpha?: boolean, // default: true
  preserveDrawingBuffer?: boolean, // default: false — needed for screenshots
  depth?: boolean,           // default: true — depth buffer
  stencil?: boolean,         // default: true — stencil buffer
  powerPreference?: string,  // "default" | "high-performance" | "low-power"
  failIfMajorPerformanceCaveat?: boolean, // default: false
  logarithmicDepthBuffer?: boolean, // default: false — fixes z-fighting for large scenes
  precision?: string,        // "highp" | "mediump" | "lowp" — default: "highp"
  reverseDepthBuffer?: boolean // default: false — r160+ feature for better depth precision
})
```

**Critical pitfall — antialias:** MUST be set at construction time. It CANNOT be changed after the renderer is created. If you need to toggle AA, you must dispose and recreate the renderer.

**Critical pitfall — preserveDrawingBuffer:** Set to `true` only when you need `canvas.toDataURL()` or `canvas.toBlob()` for screenshots. Keeping it `true` permanently has a performance cost.

**Version note (r160+):** `reverseDepthBuffer` is a newer feature that provides better depth precision for large-scale scenes by reversing the depth range. It requires a floating-point depth buffer.

#### Key Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `domElement` | `HTMLCanvasElement` | — | The canvas element; append to DOM |
| `shadowMap` | `WebGLShadowMap` | — | Shadow map configuration |
| `shadowMap.enabled` | `Boolean` | `false` | MUST enable for shadows |
| `shadowMap.type` | `Constant` | `PCFShadowMap` | Shadow quality: `BasicShadowMap`, `PCFShadowMap`, `PCFSoftShadowMap`, `VSMShadowMap` |
| `toneMapping` | `Constant` | `NoToneMapping` | HDR tone mapping: `LinearToneMapping`, `ReinhardToneMapping`, `CineonToneMapping`, `ACESFilmicToneMapping`, `AgXToneMapping`, `NeutralToneMapping` |
| `toneMappingExposure` | `Number` | `1` | Exposure level for tone mapping |
| `outputColorSpace` | `String` | `SRGBColorSpace` | Output color space (r160+: ALWAYS `SRGBColorSpace` for correct display) |
| `info` | `Object` | — | Render statistics (draw calls, triangles, textures, etc.) |
| `autoClear` | `Boolean` | `true` | Auto-clear before render |
| `autoClearColor` | `Boolean` | `true` | Auto-clear color buffer |
| `autoClearDepth` | `Boolean` | `true` | Auto-clear depth buffer |
| `autoClearStencil` | `Boolean` | `true` | Auto-clear stencil buffer |
| `sortObjects` | `Boolean` | `true` | Sort objects by depth before rendering |
| `clippingPlanes` | `Plane[]` | `[]` | Global clipping planes |
| `localClippingEnabled` | `Boolean` | `false` | Enable per-material clipping |

**Critical pitfall — outputColorSpace:** In r160+, `outputColorSpace` defaults to `SRGBColorSpace`. If you load textures that are already in sRGB (like photos), set `texture.colorSpace = THREE.SRGBColorSpace` to avoid double gamma correction. Normal maps and data textures should remain in `LinearSRGBColorSpace`.

#### Key Methods

```javascript
// Core render call
renderer.render(scene: Scene, camera: Camera): void

// Resize handling — ALWAYS call on window resize
renderer.setSize(width: Number, height: Number, updateStyle?: Boolean): void

// Pixel ratio — ALWAYS set for retina/HiDPI displays
renderer.setPixelRatio(value: Number): void
renderer.getPixelRatio(): Number

// Animation loop (preferred over manual requestAnimationFrame)
renderer.setAnimationLoop(callback: ((time: DOMHighResTimeStamp) => void) | null): void

// Clear control
renderer.setClearColor(color: Color | String | Number, alpha?: Number): void
renderer.clear(color?: Boolean, depth?: Boolean, stencil?: Boolean): void

// Render targets (off-screen rendering)
renderer.setRenderTarget(renderTarget: WebGLRenderTarget | null): void
renderer.getRenderTarget(): WebGLRenderTarget | null
renderer.readRenderTargetPixels(rt, x, y, w, h, buffer, activeCubeFaceIndex?): void

// Viewport and scissor
renderer.setViewport(x: Number, y: Number, width: Number, height: Number): void
renderer.setScissor(x: Number, y: Number, width: Number, height: Number): void
renderer.setScissorTest(enable: Boolean): void

// Shader compilation
renderer.compile(scene: Scene, camera: Camera): void
renderer.compileAsync(scene: Scene, camera: Camera): Promise<void>

// Cleanup — ALWAYS call when destroying renderer
renderer.dispose(): void
```

#### Standard Initialization Pattern

```javascript
// 1. Create renderer
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // Cap at 2x for performance
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// 2. Handle resize
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// 3. Animation loop
renderer.setAnimationLoop((time) => {
  // Update logic here
  renderer.render(scene, camera);
});

// 4. Cleanup
renderer.dispose();
```

**Critical pitfall — pixel ratio:** NEVER use `window.devicePixelRatio` directly without capping. On 3x or 4x displays, this quadruples or more the pixel count, causing severe performance degradation. ALWAYS cap: `Math.min(window.devicePixelRatio, 2)`.

**Critical pitfall — setAnimationLoop vs requestAnimationFrame:** `setAnimationLoop()` is required for WebXR compatibility. It also handles pause/resume automatically. ALWAYS prefer `setAnimationLoop()` over manual `requestAnimationFrame` loops.

**Critical pitfall — dispose:** Failing to call `renderer.dispose()` when destroying the renderer leaks GPU memory and WebGL contexts. Browsers limit the number of active WebGL contexts (typically 8-16). In SPAs, ALWAYS dispose on component unmount.

### 2.2 WebGPURenderer

The next-generation renderer using the WebGPU API. Available in Three.js r160+ as an experimental feature.

#### Key Differences from WebGLRenderer

| Aspect | WebGLRenderer | WebGPURenderer |
|--------|--------------|----------------|
| API | WebGL 2.0 | WebGPU |
| Initialization | Synchronous | **Asynchronous** (`await renderer.init()`) |
| Shader Language | GLSL | WGSL / TSL (Three Shading Language) |
| Material System | Standard materials | Node-based materials (`*NodeMaterial`) |
| Browser Support | Universal | Chrome 113+, Edge 113+, Firefox (behind flag), Safari (in development) |
| Compute Shaders | Not supported | Supported |
| Bindless Textures | Not supported | Supported |

#### Initialization Pattern

```javascript
import * as THREE from 'three';
import WebGPURenderer from 'three/addons/renderers/webgpu/WebGPURenderer.js';

const renderer = new WebGPURenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

// CRITICAL: WebGPURenderer requires async initialization
await renderer.init();

renderer.setAnimationLoop((time) => {
  renderer.render(scene, camera);
});
```

**Critical pitfall — async init:** Unlike WebGLRenderer, WebGPURenderer MUST call `await renderer.init()` before any rendering. Calling `render()` before `init()` completes will throw an error or produce no output.

#### Node-Based Materials (TSL)

WebGPURenderer introduces the Three Shading Language (TSL) and node-based material variants:

- `MeshStandardNodeMaterial`
- `MeshPhysicalNodeMaterial`
- `MeshBasicNodeMaterial`
- `MeshPhongNodeMaterial`
- `MeshLambertNodeMaterial`
- `LineBasicNodeMaterial`
- `PointsNodeMaterial`
- `SpriteNodeMaterial`

These materials use a node graph system instead of fixed shader parameters, enabling procedural and dynamic material effects at the shader level.

#### Related WebGPU Classes

- `WGSLNodeBuilder` — Compiles node graphs to WGSL shaders
- `WGSLNodeFunction` — WGSL function nodes
- `WGSLNodeParser` — WGSL code parser
- `WebGPUTimestampQueryPool` — GPU performance timing

#### Render Targets

Both renderers support off-screen rendering via render targets:

```javascript
// WebGL
const rt = new THREE.WebGLRenderTarget(512, 512);
renderer.setRenderTarget(rt);
renderer.render(scene, camera);
renderer.setRenderTarget(null); // Back to screen

// The texture is available at rt.texture
```

Render targets enable: post-processing pipelines, shadow maps, environment map generation, portal effects, and multi-pass rendering.

---

## 3. Camera System

### 3.1 Base Camera Class

`Camera` extends `Object3D` and adds projection-specific properties. It is abstract — NEVER instantiate directly.

#### Camera-Specific Properties

| Property | Type | Description |
|----------|------|-------------|
| `matrixWorldInverse` | `Matrix4` | Inverse of the camera's world matrix (view matrix) |
| `projectionMatrix` | `Matrix4` | The projection matrix |
| `projectionMatrixInverse` | `Matrix4` | Inverse of the projection matrix |
| `isCamera` | `Boolean` | Type check flag |
| `layers` | `Layers` | Which layers this camera renders |
| `coordinateSystem` | `Constant` | `WebGLCoordinateSystem` or `WebGPUCoordinateSystem` |

#### Key Method

```javascript
camera.getWorldDirection(target: Vector3): Vector3
```

Returns the direction the camera is looking in world space. The camera looks along its local -Z axis.

### 3.2 PerspectiveCamera

The most commonly used camera. Mimics human eye perspective with objects appearing smaller at greater distances.

#### Constructor

```javascript
new THREE.PerspectiveCamera(
  fov?: Number,    // Vertical field of view in degrees — default: 50
  aspect?: Number, // Width / height ratio — default: 1
  near?: Number,   // Near clipping plane — default: 0.1
  far?: Number     // Far clipping plane — default: 2000
)
```

#### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `fov` | `Number` | `50` | Vertical field of view (degrees, NOT radians) |
| `aspect` | `Number` | `1` | Aspect ratio (width / height) |
| `near` | `Number` | `0.1` | Near clipping plane distance |
| `far` | `Number` | `2000` | Far clipping plane distance |
| `zoom` | `Number` | `1` | Zoom factor (>1 zooms in) |
| `filmGauge` | `Number` | `35` | Film size in mm (for focal length calculations) |
| `filmOffset` | `Number` | `0` | Horizontal off-center offset |
| `focus` | `Number` | `10` | Focus distance for stereo/DOF effects |
| `view` | `Object \| null` | `null` | View offset (set by `setViewOffset()`) |
| `isPerspectiveCamera` | `Boolean` | `true` | Type check flag |

#### Key Methods

```javascript
// MUST call after changing fov, aspect, near, far, or zoom
camera.updateProjectionMatrix(): void

// Focal length (relates to filmGauge)
camera.setFocalLength(focalLength: Number): void
camera.getFocalLength(): Number

// Effective FOV (accounts for zoom)
camera.getEffectiveFOV(): Number

// Film dimensions
camera.getFilmWidth(): Number
camera.getFilmHeight(): Number

// Multi-window/multi-monitor rendering
camera.setViewOffset(fullWidth, fullHeight, offsetX, offsetY, width, height): void
camera.clearViewOffset(): void
```

**Critical pitfall — updateProjectionMatrix:** After changing ANY camera parameter (`fov`, `aspect`, `near`, `far`, `zoom`), you MUST call `camera.updateProjectionMatrix()`. Forgetting this is one of the most common Three.js bugs — the camera continues using the old projection until this method is called.

**Critical pitfall — near/far ratio:** The depth buffer precision depends on the `near/far` ratio. A ratio greater than 1000:1 (e.g., near=0.01, far=10000) causes z-fighting artifacts. ALWAYS keep `near` as large as possible and `far` as small as possible. For large scenes, consider `logarithmicDepthBuffer: true` in the renderer.

**Critical pitfall — fov units:** `fov` is in DEGREES, not radians. This is an exception to the general Three.js convention where most angles are in radians.

#### Standard Setup

```javascript
const camera = new THREE.PerspectiveCamera(
  75,                                      // 75 degree FOV
  window.innerWidth / window.innerHeight,  // Aspect ratio
  0.1,                                     // Near plane
  1000                                     // Far plane
);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0);

// On window resize:
camera.aspect = window.innerWidth / window.innerHeight;
camera.updateProjectionMatrix();
```

### 3.3 OrthographicCamera

Parallel projection camera where object size is independent of distance. Used for 2D overlays, isometric views, shadow map rendering, and technical/architectural visualization.

#### Constructor

```javascript
new THREE.OrthographicCamera(
  left: Number,    // Left frustum boundary
  right: Number,   // Right frustum boundary
  top: Number,     // Top frustum boundary
  bottom: Number,  // Bottom frustum boundary
  near?: Number,   // Near clipping plane — default: 0.1
  far?: Number     // Far clipping plane — default: 2000
)
```

#### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `left` | `Number` | — | Left boundary |
| `right` | `Number` | — | Right boundary |
| `top` | `Number` | — | Top boundary |
| `bottom` | `Number` | — | Bottom boundary |
| `near` | `Number` | `0.1` | Near clipping plane |
| `far` | `Number` | `2000` | Far clipping plane |
| `zoom` | `Number` | `1` | Zoom factor |
| `view` | `Object \| null` | `null` | View offset |
| `isOrthographicCamera` | `Boolean` | `true` | Type check flag |

#### Methods

Same pattern as PerspectiveCamera: `updateProjectionMatrix()`, `setViewOffset()`, `clearViewOffset()`, `clone()`, `copy()`, `toJSON()`.

**Critical pitfall — updateProjectionMatrix:** Same rule as PerspectiveCamera. MUST call after changing left/right/top/bottom/near/far/zoom.

#### Standard Setup

```javascript
const frustumSize = 10;
const aspect = window.innerWidth / window.innerHeight;
const camera = new THREE.OrthographicCamera(
  frustumSize * aspect / -2,  // left
  frustumSize * aspect / 2,   // right
  frustumSize / 2,             // top
  frustumSize / -2,            // bottom
  0.1,                         // near
  1000                         // far
);

// On resize:
const newAspect = window.innerWidth / window.innerHeight;
camera.left = frustumSize * newAspect / -2;
camera.right = frustumSize * newAspect / 2;
camera.top = frustumSize / 2;
camera.bottom = frustumSize / -2;
camera.updateProjectionMatrix();
```

### 3.4 Other Camera Types

| Camera | Purpose |
|--------|---------|
| `ArrayCamera` | Array of sub-cameras for multi-view rendering (VR, split-screen) |
| `CubeCamera` | Six PerspectiveCamera array for cube map generation (reflections, environment maps) |
| `StereoCamera` | Dual camera for stereoscopic rendering (VR) |

---

## 4. Math Utilities

### 4.1 Coordinate System

Three.js uses a **right-handed coordinate system** with **Y-up**:

- **+X** = right
- **+Y** = up
- **+Z** = toward the viewer (out of the screen)

This is critical when importing models from other software (e.g., Blender uses Z-up by default). The GLTFLoader handles this conversion automatically.

All angles in the math library are in **radians** unless explicitly noted (PerspectiveCamera.fov is the notable exception — it uses degrees).

### 4.2 Vector3

The most frequently used math class. Represents a 3D point or direction.

#### Constructor

```javascript
new THREE.Vector3(x?: Number, y?: Number, z?: Number)
// Defaults: (0, 0, 0)
```

#### Key Methods (Grouped by Category)

**Setting values:**
```javascript
v.set(x, y, z): Vector3
v.setScalar(s): Vector3         // Sets x=y=z=s
v.setX(x): Vector3
v.setY(y): Vector3
v.setZ(z): Vector3
v.setComponent(index, value): Vector3  // 0=x, 1=y, 2=z
v.copy(source): Vector3
v.clone(): Vector3
```

**Arithmetic (mutating — returns `this`):**
```javascript
v.add(other): Vector3
v.addScalar(s): Vector3
v.addVectors(a, b): Vector3           // v = a + b
v.addScaledVector(other, s): Vector3  // v += other * s
v.sub(other): Vector3
v.subScalar(s): Vector3
v.subVectors(a, b): Vector3           // v = a - b
v.multiply(other): Vector3            // Component-wise
v.multiplyScalar(s): Vector3
v.multiplyVectors(a, b): Vector3
v.divide(other): Vector3
v.divideScalar(s): Vector3
v.negate(): Vector3                   // v = -v
```

**Vector operations:**
```javascript
v.dot(other): Number                  // Dot product
v.cross(other): Vector3               // Cross product (mutates v)
v.crossVectors(a, b): Vector3         // v = a x b
v.angleTo(other): Number              // Angle in radians
v.normalize(): Vector3                // Make unit length
v.setLength(length): Vector3
v.length(): Number
v.lengthSq(): Number                  // Faster than length() — avoids sqrt
v.distanceTo(other): Number
v.distanceToSquared(other): Number    // Faster — avoids sqrt
v.manhattanDistanceTo(other): Number
v.manhattanLength(): Number
```

**Interpolation:**
```javascript
v.lerp(target, t): Vector3            // Linear interpolation (t: 0-1)
v.lerpVectors(a, b, t): Vector3       // v = lerp(a, b, t)
```

**Transformation:**
```javascript
v.applyMatrix3(m): Vector3
v.applyMatrix4(m): Vector3            // Applies full 4x4 transform (including translation)
v.applyNormalMatrix(m): Vector3       // For normal vectors (Matrix3)
v.applyQuaternion(q): Vector3
v.applyEuler(euler): Vector3
v.applyAxisAngle(axis, angle): Vector3
```

**Projection:**
```javascript
v.project(camera): Vector3            // World → NDC (normalized device coordinates)
v.unproject(camera): Vector3           // NDC → world
v.projectOnVector(target): Vector3
v.projectOnPlane(planeNormal): Vector3
v.reflect(normal): Vector3
```

**Utility:**
```javascript
v.clamp(min, max): Vector3            // Clamp each component (min/max are Vector3)
v.clampScalar(min, max): Vector3
v.clampLength(min, max): Vector3
v.floor(): Vector3
v.ceil(): Vector3
v.round(): Vector3
v.roundToZero(): Vector3
v.min(other): Vector3                 // Component-wise min
v.max(other): Vector3                 // Component-wise max
v.equals(other): Boolean
v.fromArray(array, offset?): Vector3
v.toArray(array?, offset?): Array
v.fromBufferAttribute(attr, index): Vector3
v.random(): Vector3                   // Random values 0-1
v.randomDirection(): Vector3           // Random unit vector
```

**Critical pitfall — mutability:** Almost ALL Vector3 methods MUTATE the instance and return `this` for chaining. This is a performance optimization to avoid garbage collection pressure, but it catches developers used to immutable math libraries. ALWAYS use `.clone()` before operations if you need to preserve the original.

**Performance tip:** Use `lengthSq()` and `distanceToSquared()` instead of `length()` and `distanceTo()` when comparing distances. The squared versions avoid an expensive `Math.sqrt()` call.

### 4.3 Matrix4

4x4 transformation matrix. Used internally for all object transforms and camera projections.

#### Constructor

```javascript
new THREE.Matrix4()  // Creates identity matrix
```

#### Storage Format

Elements are stored in **column-major order** as a `Float32Array` of 16 elements:

```
elements = [n11, n21, n31, n41, n12, n22, n32, n42, n13, n23, n33, n43, n14, n24, n34, n44]
```

However, `set()` takes parameters in **row-major order** for readability:

```javascript
matrix.set(
  n11, n12, n13, n14,  // Row 1
  n21, n22, n23, n24,  // Row 2
  n31, n32, n33, n34,  // Row 3
  n41, n42, n43, n44   // Row 4
)
```

**Critical pitfall — element order:** The `elements` array is column-major (OpenGL convention), but `set()` accepts row-major. Mixing these up produces incorrect transforms.

#### Key Methods

**Composition and decomposition:**
```javascript
m.compose(position: Vector3, quaternion: Quaternion, scale: Vector3): Matrix4
m.decompose(position: Vector3, quaternion: Quaternion, scale: Vector3): Matrix4
```

**Factory methods (create specific transform matrices):**
```javascript
m.makeTranslation(x, y, z): Matrix4
m.makeRotationX(theta): Matrix4           // Radians
m.makeRotationY(theta): Matrix4
m.makeRotationZ(theta): Matrix4
m.makeRotationAxis(axis: Vector3, angle: Number): Matrix4
m.makeRotationFromEuler(euler: Euler): Matrix4
m.makeRotationFromQuaternion(q: Quaternion): Matrix4
m.makeScale(x, y, z): Matrix4
m.makeShear(xy, xz, yx, yz, zx, zy): Matrix4
m.makePerspective(left, right, top, bottom, near, far): Matrix4
m.makeOrthographic(left, right, top, bottom, near, far): Matrix4
m.makeBasis(xAxis, yAxis, zAxis): Matrix4
m.lookAt(eye: Vector3, target: Vector3, up: Vector3): Matrix4
```

**Multiplication:**
```javascript
m.multiply(other): Matrix4               // m = m * other
m.premultiply(other): Matrix4            // m = other * m
m.multiplyMatrices(a, b): Matrix4        // m = a * b
m.multiplyScalar(s): Matrix4
```

**Other operations:**
```javascript
m.identity(): Matrix4
m.invert(): Matrix4
m.transpose(): Matrix4
m.determinant(): Number
m.extractBasis(xAxis, yAxis, zAxis): Matrix4
m.extractRotation(source: Matrix4): Matrix4
m.setPosition(v: Vector3 | x, y?, z?): Matrix4
m.scale(v: Vector3): Matrix4
m.clone(): Matrix4
m.copy(source): Matrix4
m.equals(other): Boolean
m.fromArray(array, offset?): Matrix4
m.toArray(array?, offset?): Array
```

**Critical pitfall — multiply order:** Matrix multiplication is NOT commutative. `A * B !== B * A`. In Three.js, `m.multiply(other)` computes `m = m * other` (right-multiply), while `m.premultiply(other)` computes `m = other * m` (left-multiply). Getting this order wrong is a common source of incorrect transformations.

### 4.4 Quaternion

Represents rotation without gimbal lock. Preferred over Euler angles for animation interpolation.

#### Constructor

```javascript
new THREE.Quaternion(x?: Number, y?: Number, z?: Number, w?: Number)
// Defaults: (0, 0, 0, 1) — identity rotation
```

#### Key Methods

**Setting rotation:**
```javascript
q.set(x, y, z, w): Quaternion
q.setFromAxisAngle(axis: Vector3, angle: Number): Quaternion
q.setFromEuler(euler: Euler): Quaternion
q.setFromRotationMatrix(m: Matrix4): Quaternion
q.setFromUnitVectors(from: Vector3, to: Vector3): Quaternion  // Rotation from one direction to another
q.identity(): Quaternion               // Reset to no rotation
q.random(): Quaternion                 // Random rotation
```

**Operations:**
```javascript
q.multiply(other): Quaternion           // q = q * other (combine rotations)
q.premultiply(other): Quaternion        // q = other * q
q.multiplyQuaternions(a, b): Quaternion // q = a * b
q.conjugate(): Quaternion               // Negate x, y, z
q.invert(): Quaternion                  // Inverse rotation
q.normalize(): Quaternion               // Ensure unit quaternion
q.dot(other): Number
q.angleTo(other): Number               // Angle between rotations (radians)
q.rotateTowards(target, step): Quaternion  // Step toward target rotation
```

**Interpolation:**
```javascript
q.slerp(target, t): Quaternion                      // Spherical linear interpolation (0-1)
q.slerpQuaternions(a, b, t): Quaternion              // q = slerp(a, b, t)
```

**Utility:**
```javascript
q.length(): Number
q.lengthSq(): Number
q.clone(): Quaternion
q.copy(source): Quaternion
q.equals(other): Boolean
q.fromArray(array, offset?): Quaternion
q.toArray(array?, offset?): Array
```

**Why quaternions over Euler:** Euler angles suffer from gimbal lock (loss of a rotation degree of freedom when two axes align at 90 degrees). Quaternions NEVER have this problem. ALWAYS use quaternions for smooth rotation interpolation (animations, camera orbits). Use Euler angles only for human-readable rotation setup (e.g., `rotation.y = Math.PI / 4`).

### 4.5 Euler

Represents rotation as three angles around the X, Y, and Z axes.

#### Constructor

```javascript
new THREE.Euler(x?: Number, y?: Number, z?: Number, order?: String)
// Defaults: (0, 0, 0, 'XYZ')
```

#### Rotation Orders

The order in which rotations are applied changes the result:

| Order | Application Sequence |
|-------|---------------------|
| `'XYZ'` | X then Y then Z (default) |
| `'YXZ'` | Y then X then Z (common for FPS cameras) |
| `'ZXY'` | Z then X then Y |
| `'ZYX'` | Z then Y then X |
| `'YZX'` | Y then Z then X |
| `'XZY'` | X then Z then Y |

#### Key Methods

```javascript
e.set(x, y, z, order?): Euler
e.clone(): Euler
e.copy(source): Euler
e.equals(other): Boolean
e.reorder(newOrder): Euler              // Change order, maintain orientation
e.setFromQuaternion(q, order?): Euler
e.setFromRotationMatrix(m, order?): Euler
e.setFromVector3(v, order?): Euler
e.fromArray(array): Euler
e.toArray(array?, offset?): Array
```

**Gimbal lock warning:** Gimbal lock occurs when the middle axis rotation reaches +/-90 degrees. At this point, the first and third axes become parallel, effectively losing one degree of freedom. This manifests as sudden jumps or inability to reach certain orientations. Use quaternions for any rotation that passes through these angles.

### 4.6 Color

Represents a color with RGB components in linear space (0-1 range).

#### Constructor (Multiple Forms)

```javascript
new THREE.Color()                           // White (1, 1, 1)
new THREE.Color(0xff0000)                   // Red (hex number)
new THREE.Color('#ff0000')                  // Red (hex string)
new THREE.Color('red')                      // Red (CSS color name)
new THREE.Color('rgb(255, 0, 0)')           // Red (CSS rgb)
new THREE.Color('hsl(0, 100%, 50%)')        // Red (CSS hsl)
new THREE.Color(1, 0, 0)                    // Red (r, g, b floats)
```

#### Properties

| Property | Type | Range | Description |
|----------|------|-------|-------------|
| `r` | `Number` | 0-1 | Red channel |
| `g` | `Number` | 0-1 | Green channel |
| `b` | `Number` | 0-1 | Blue channel |
| `isColor` | `Boolean` | — | Type check flag |

#### Key Methods

**Setting:**
```javascript
c.set(value): Color                     // Hex, string, or Color
c.setHex(hex): Color                    // Integer 0xRRGGBB
c.setRGB(r, g, b, colorSpace?): Color   // r160+: optional colorSpace param
c.setHSL(h, s, l, colorSpace?): Color
c.setStyle(style): Color                // CSS string
c.setColorName(name): Color             // 'red', 'blue', etc.
c.setScalar(s): Color                   // r=g=b=s
```

**Getting:**
```javascript
c.getHex(colorSpace?): Number
c.getHexString(colorSpace?): String     // Without '#' prefix
c.getHSL(target, colorSpace?): {h, s, l}
c.getRGB(target, colorSpace?): {r, g, b}
c.getStyle(colorSpace?): String         // CSS-compatible string
```

**Arithmetic:**
```javascript
c.add(other): Color
c.addColors(a, b): Color
c.addScalar(s): Color
c.sub(other): Color
c.multiply(other): Color               // Component-wise
c.multiplyScalar(s): Color
```

**Interpolation:**
```javascript
c.lerp(target, t): Color               // RGB-space interpolation
c.lerpColors(a, b, t): Color
c.lerpHSL(target, t): Color            // HSL-space interpolation (smoother for hue transitions)
```

**Color space (r160+):**
```javascript
c.convertLinearToSRGB(): Color
c.convertSRGBToLinear(): Color
```

**Utility:**
```javascript
c.offsetHSL(h, s, l): Color            // Adjust HSL values
c.equals(other): Boolean
c.clone(): Color
c.copy(source): Color
c.fromArray(array, offset?): Color
c.toArray(array?, offset?): Array
c.toJSON(): Number                      // Returns hex value
```

**Color management pitfall (r160+):** Three.js r160+ uses a color-managed workflow. Colors in materials and lights are assumed to be in **linear sRGB** space internally. When you set a color using CSS strings or hex values, they are interpreted as sRGB and converted to linear automatically. However, when setting RGB values directly with `setRGB()`, you can specify the color space: `color.setRGB(1, 0, 0, THREE.SRGBColorSpace)`. Getting this wrong causes colors to appear washed out or overly saturated.

**lerpHSL vs lerp:** Use `lerpHSL()` when interpolating between colors with different hues (e.g., red to blue). RGB-space `lerp()` produces muddy intermediate colors, while HSL-space interpolation follows the color wheel naturally.

### 4.7 MathUtils

Static utility class — no constructor needed. All methods are accessed as `THREE.MathUtils.methodName()`.

#### Complete Method List

| Method | Signature | Description |
|--------|-----------|-------------|
| `clamp` | `clamp(value, min, max): Number` | Clamp value to [min, max] range |
| `degToRad` | `degToRad(degrees): Number` | Convert degrees to radians |
| `radToDeg` | `radToDeg(radians): Number` | Convert radians to degrees |
| `lerp` | `lerp(x, y, t): Number` | Linear interpolation: `x + (y - x) * t` |
| `damp` | `damp(x, y, lambda, dt): Number` | Frame-rate independent damping |
| `inverseLerp` | `inverseLerp(x, y, value): Number` | Inverse of lerp: returns t for given value |
| `mapLinear` | `mapLinear(x, a1, a2, b1, b2): Number` | Map value from range [a1,a2] to [b1,b2] |
| `smoothstep` | `smoothstep(x, min, max): Number` | Hermite interpolation (smooth 0-1 curve) |
| `smootherstep` | `smootherstep(x, min, max): Number` | Ken Perlin's improved smoothstep |
| `pingpong` | `pingpong(x, length): Number` | Ping-pong value between 0 and length |
| `randFloat` | `randFloat(low, high): Number` | Random float in [low, high] |
| `randFloatSpread` | `randFloatSpread(range): Number` | Random float in [-range/2, range/2] |
| `randInt` | `randInt(low, high): Number` | Random integer in [low, high] |
| `seededRandom` | `seededRandom(seed?): Number` | Deterministic pseudo-random number |
| `generateUUID` | `generateUUID(): String` | RFC4122 v4 UUID |
| `isPowerOfTwo` | `isPowerOfTwo(n): Boolean` | Check if n is power of 2 |
| `ceilPowerOfTwo` | `ceilPowerOfTwo(n): Number` | Next power of 2 >= n |
| `floorPowerOfTwo` | `floorPowerOfTwo(n): Number` | Largest power of 2 <= n |
| `euclideanModulo` | `euclideanModulo(n, m): Number` | Always-positive modulo |
| `setQuaternionFromProperEuler` | `setQuaternionFromProperEuler(q, a, b, c, order): void` | Set quaternion from proper Euler angles |

**damp() for smooth animations:** `MathUtils.damp(current, target, lambda, deltaTime)` provides frame-rate independent exponential decay. This is superior to `lerp()` for smooth following behavior because it produces consistent results regardless of frame rate. ALWAYS use `damp()` instead of `lerp()` in animation loops where frame rate varies.

**euclideanModulo:** JavaScript's `%` operator can return negative values for negative inputs (e.g., `-1 % 3 === -1`). `euclideanModulo(-1, 3) === 2`. ALWAYS use `euclideanModulo()` when you need wrapping behavior (e.g., looping angles).

### 4.8 Other Math Classes

For completeness, Three.js also provides:

| Class | Purpose |
|-------|---------|
| `Vector2` | 2D vector (UV coordinates, screen positions) |
| `Vector4` | 4D vector (homogeneous coordinates, shader uniforms) |
| `Matrix3` | 3x3 matrix (normal matrix, 2D transforms) |
| `Box2` | 2D axis-aligned bounding box |
| `Box3` | 3D axis-aligned bounding box |
| `Sphere` | Bounding sphere |
| `Ray` | Ray for raycasting |
| `Plane` | Mathematical plane |
| `Frustum` | Camera frustum (6 planes) |
| `Triangle` | Triangle with vertices |
| `Line3` | 3D line segment |
| `Cylindrical` | Cylindrical coordinates |
| `Spherical` | Spherical coordinates |
| `SphericalHarmonics3` | Spherical harmonics (for IBL) |
| `Interpolant` | Base class for animation interpolation |

---

## 5. Cross-Cutting Concerns and Anti-Patterns

### 5.1 Memory Management

Three.js does NOT automatically garbage-collect GPU resources. You MUST manually dispose:

```javascript
// Geometry
geometry.dispose();

// Material (and its textures)
material.dispose();
material.map?.dispose();
material.normalMap?.dispose();
material.envMap?.dispose();
// ... all texture properties

// Render targets
renderTarget.dispose();

// Renderer
renderer.dispose();
```

**Anti-pattern:** Creating new geometries or materials every frame without disposing the old ones. This leaks GPU memory and eventually crashes the browser tab.

**Correct pattern for scene cleanup:**
```javascript
scene.traverse((child) => {
  if (child.isMesh) {
    child.geometry.dispose();
    if (Array.isArray(child.material)) {
      child.material.forEach(m => m.dispose());
    } else {
      child.material.dispose();
    }
  }
});
```

### 5.2 Common Initialization Mistakes

1. **Not setting pixel ratio** — renders blurry on retina displays
2. **Not handling resize** — scene stretches or clips on window resize
3. **Not enabling shadow map** — shadows simply do not appear
4. **Setting antialias after construction** — has no effect
5. **Using PerspectiveCamera without updating aspect ratio on resize** — distorted perspective

### 5.3 Transform Pitfalls Summary

1. **Forgetting `updateProjectionMatrix()`** after changing camera properties
2. **Modifying `matrix` with `matrixAutoUpdate = true`** — changes overwritten every frame
3. **Using `rotation` and `quaternion` independently** — they auto-sync and overwrite each other
4. **Not calling `.clone()` before mutating vectors** — original value corrupted
5. **Wrong matrix multiplication order** — `multiply()` is right-multiply, not left-multiply
6. **Assuming `lookAt()` works the same for cameras and meshes** — camera looks along -Z, meshes along +Z
7. **Using Euler angles for smooth animation** — gimbal lock at 90-degree boundaries; use quaternion `slerp()` instead

### 5.4 Version-Specific Notes (r160+)

| Change | Impact |
|--------|--------|
| Color management enabled by default | `outputColorSpace = SRGBColorSpace`; textures need correct `colorSpace` |
| WebGL 1.0 support removed | Minimum WebGL 2.0 required |
| `reverseDepthBuffer` option added | Better depth precision for large scenes |
| `compileAsync()` added | Non-blocking shader compilation |
| `backgroundBlurriness`, `backgroundIntensity` on Scene | Finer control over environment backgrounds |
| `environmentIntensity`, `environmentRotation` on Scene | Independent environment map control |
| `setRGB()` and `setHSL()` accept `colorSpace` parameter | Explicit color space specification |
| WebGPURenderer stabilization | Usable (experimental) with TSL and node materials |
| `AgXToneMapping` and `NeutralToneMapping` added | New tone mapping options alongside ACES |

---

## 6. Summary Table: Core Classes at a Glance

| Class | Category | Key Purpose | Most Important Method |
|-------|----------|-------------|----------------------|
| `Object3D` | Scene Graph | Base for all 3D objects | `add()`, `traverse()` |
| `Scene` | Scene Graph | Root container | `background`, `environment` |
| `Group` | Scene Graph | Organizational container | Inherited from Object3D |
| `Mesh` | Scene Graph | Renderable geometry + material | Constructor(geometry, material) |
| `WebGLRenderer` | Renderer | GPU rendering via WebGL 2.0 | `render()`, `setAnimationLoop()` |
| `WebGPURenderer` | Renderer | GPU rendering via WebGPU | `init()` (async), `render()` |
| `PerspectiveCamera` | Camera | Perspective projection | `updateProjectionMatrix()` |
| `OrthographicCamera` | Camera | Parallel projection | `updateProjectionMatrix()` |
| `Vector3` | Math | 3D point/direction | `normalize()`, `lerp()`, `clone()` |
| `Matrix4` | Math | 4x4 transformation | `compose()`, `decompose()`, `multiply()` |
| `Quaternion` | Math | Rotation (no gimbal lock) | `slerp()`, `setFromAxisAngle()` |
| `Euler` | Math | Rotation (human-readable) | `set()`, `reorder()` |
| `Color` | Math | RGB color | `setHex()`, `lerp()`, `lerpHSL()` |
| `MathUtils` | Math | Static utilities | `clamp()`, `damp()`, `degToRad()` |

---

*End of B1 Core Research. Word count: ~4000. All information sourced from official Three.js r160+ documentation at threejs.org/docs.*
