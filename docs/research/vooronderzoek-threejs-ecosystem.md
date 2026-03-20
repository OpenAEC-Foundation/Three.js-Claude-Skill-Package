# Vooronderzoek: Three.js Ecosystem

> Deep research document covering React Three Fiber, Drei, WebGPU/TSL, WebXR, IFC/BIM, and Controls.
> All information cross-referenced against official documentation.
> Three.js r160+, React Three Fiber 8.x/9.x, Drei latest.

---

## 1. React Three Fiber (R3F) Core

**Source:** https://r3f.docs.pmnd.rs/

React Three Fiber is a React renderer for Three.js. It translates JSX into Three.js scene graph operations with zero overhead compared to imperative Three.js code. R3F version 8 pairs with React 18; R3F version 9 pairs with React 19.

### 1.1 Canvas Component

The `<Canvas>` component is the root of every R3F scene. It creates a WebGL context, scene, camera, and render loop automatically.

**All Canvas Props:**

| Prop | Type | Default | Purpose |
|------|------|---------|---------|
| `children` | `React.ReactNode` | — | Scene content |
| `fallback` | `React.ReactNode` | — | DOM fallback during WebGL init |
| `gl` | `Renderer props \| (canvas) => Renderer` | `{}` | WebGL renderer configuration or factory |
| `camera` | `Camera props \| THREE.Camera` | `{ fov: 75, near: 0.1, far: 1000, position: [0, 0, 5] }` | Default perspective camera |
| `scene` | `Scene props \| THREE.Scene` | `{}` | Scene configuration or pre-existing scene |
| `shadows` | `boolean \| ShadowMapType` | `false` | Enable shadow maps; pass string for shadow type |
| `raycaster` | `Raycaster props` | `{}` | Raycaster configuration for pointer events |
| `frameloop` | `"always" \| "demand" \| "never"` | `"always"` | Render loop strategy |
| `resize` | `ResizeOptions` | `{ scroll: true, debounce: { scroll: 50, resize: 0 } }` | Canvas resize behavior |
| `orthographic` | `boolean` | `false` | Use OrthographicCamera instead of PerspectiveCamera |
| `dpr` | `number \| [min, max]` | `[1, 2]` | Device pixel ratio or clamped range |
| `legacy` | `boolean` | `false` | Disables automatic color management |
| `linear` | `boolean` | `false` | Switches to linear color space (no sRGB conversion) |
| `flat` | `boolean` | `false` | Disables tone mapping |
| `events` | `EventManager` | R3F default | Custom event manager factory |
| `eventSource` | `HTMLElement \| React.RefObject` | `gl.domElement.parentNode` | DOM element that captures pointer events |
| `eventPrefix` | `string` | `"offset"` | Event coordinate prefix (offset, client, page, layer, screen) |
| `onCreated` | `(state: RootState) => void` | — | Callback after Canvas initialization |
| `onPointerMissed` | `(event: PointerEvent) => void` | — | Fires when click hits no mesh |

**frameloop modes:**
- `"always"` — ALWAYS renders every frame via requestAnimationFrame.
- `"demand"` — ONLY renders when `invalidate()` is called. Use for static scenes to save GPU.
- `"never"` — NEVER renders automatically. Caller MUST invoke `advance(timestamp)` manually.

### 1.2 JSX-to-Three.js Mapping Rules

R3F uses a deterministic mapping convention:

1. **Lowercase JSX elements map to Three.js classes.** `<mesh />` creates `new THREE.Mesh()`. `<meshStandardMaterial />` creates `new THREE.MeshStandardMaterial()`.
2. **`args` passes constructor arguments as an array.** `<sphereGeometry args={[1, 32, 32]} />` becomes `new THREE.SphereGeometry(1, 32, 32)`. When `args` changes, the object is destroyed and recreated.
3. **`attach` binds the object to a parent property.** `<meshStandardMaterial attach="material" />` sets `parent.material = this`. Geometries auto-attach to `"geometry"`, materials to `"material"`.
4. **Nested attach uses dash notation.** `attach="shadow-camera"` sets `parent.shadow.camera = this`. Array indexing: `attach="material-0"`.
5. **Functional attach.** `attach={(parent, self) => { parent.add(self); return () => parent.remove(self); }}` for custom bind/unbind logic.
6. **Properties with `.set()` accept shorthand.** `position={[1, 2, 3]}` calls `object.position.set(1, 2, 3)`. `color="hotpink"` calls `object.color.set("hotpink")`.
7. **SetScalar shorthand.** `scale={2}` calls `object.scale.setScalar(2)`.
8. **Dash-case pierces nested properties.** `rotation-x={Math.PI}` sets `object.rotation.x = Math.PI`. `material-uniforms-resolution-value={[512, 512]}` traverses nested objects.

### 1.3 Primitives and extend()

**Primitives** insert pre-existing Three.js objects into the declarative tree:
```jsx
<primitive object={existingMesh} position={[10, 0, 0]} />
```
NEVER add the same object instance to the tree multiple times. Primitives do NOT auto-dispose; the caller MUST manage lifecycle.

**extend()** registers custom Three.js classes as JSX elements:
```jsx
import { extend } from '@react-three/fiber'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
extend({ OrbitControls })
// Now usable as <orbitControls />
```
The JSX element name is the camelCase version of the registered key.

### 1.4 Hooks

#### useFrame

```typescript
useFrame((state: RootState, delta: number, xrFrame?: XRFrame) => void, priority?: number)
```

Subscribes a callback to the render loop. Executes every frame.

**state object properties:**

| Property | Type | Description |
|----------|------|-------------|
| `gl` | `THREE.WebGLRenderer` | The renderer instance |
| `scene` | `THREE.Scene` | The scene |
| `camera` | `THREE.Camera` | The active camera |
| `raycaster` | `THREE.Raycaster` | Default raycaster |
| `pointer` | `THREE.Vector2` | Normalized pointer coordinates (-1 to +1) |
| `mouse` | `THREE.Vector2` | DEPRECATED — use `pointer` |
| `clock` | `THREE.Clock` | Running system clock |
| `size` | `{ width, height, top, left }` | Canvas dimensions in pixels |
| `viewport` | `{ width, height, initialDpr, dpr, factor, distance, aspect, getCurrentViewport() }` | Camera-relative viewport metrics |
| `linear` | `boolean` | Whether linear color space is active |
| `flat` | `boolean` | Whether tone mapping is disabled |
| `legacy` | `boolean` | Whether color management is disabled |
| `frameloop` | `string` | Current frameloop mode |
| `performance` | `{ current, min, max, debounce, regress() }` | Adaptive performance state |
| `xr` | `{ connect, disconnect }` | XR session management |
| `set` | `function` | Mutate state directly |
| `get` | `function` | Read state non-reactively |
| `invalidate` | `function` | Request render in demand mode |
| `advance` | `function` | Advance one tick in never mode |
| `setSize` | `function` | Resize canvas |
| `setDpr` | `function` | Set pixel ratio |
| `setFrameloop` | `function` | Change render mode at runtime |
| `setEvents` | `function` | Reconfigure event layer |
| `events` | `{ connected, handlers, connect, disconnect }` | Event system state |

**delta:** Time in seconds since last frame. ALWAYS use delta for frame-rate-independent animation.

**priority system:** Callbacks execute in ascending priority order. When any callback has priority > 0, R3F disables its automatic `renderer.render()` call. The subscriber with the highest priority MUST call `state.gl.render(state.scene, state.camera)` manually. Negative priorities execute in order but do NOT disable auto-rendering.

#### useThree

```typescript
const state = useThree()
const camera = useThree((state) => state.camera) // selector avoids re-renders
```

Returns the full RootState (same object as useFrame's state). ALWAYS use a selector when only one property is needed to avoid unnecessary re-renders.

#### useLoader

```typescript
const result = useLoader(LoaderClass, url, extensions?, onProgress?)
const results = useLoader(LoaderClass, [url1, url2], extensions?)
```

Suspense-based asset loading. ALWAYS wrap the consuming component in `<Suspense fallback={...}>`.

- Assets are cached by URL — loading the same URL twice returns the cached result.
- `useLoader.preload(LoaderClass, url)` — preloads assets before component mount.
- For GLTF files, the result includes `{ nodes, materials, scene, animations }`.

#### useGraph

```typescript
const { nodes, materials } = useGraph(object3D)
```

Traverses an Object3D hierarchy and returns memoized `{ nodes, materials }` collections keyed by name. Useful for selectively accessing parts of loaded models.

### 1.5 Event System

R3F implements a pointer event system via raycasting. Events bubble through the scene graph hierarchy.

**All supported events:**

| Event | Trigger |
|-------|---------|
| `onClick` | Pointer click on mesh |
| `onContextMenu` | Right-click / context menu |
| `onDoubleClick` | Double click |
| `onPointerUp` | Pointer released |
| `onPointerDown` | Pointer pressed |
| `onPointerOver` | Pointer enters mesh (fires continuously) |
| `onPointerOut` | Pointer leaves mesh |
| `onPointerEnter` | Pointer enters mesh (fires once) |
| `onPointerLeave` | Pointer leaves mesh (fires once) |
| `onPointerMove` | Pointer moves over mesh |
| `onPointerMissed` | Click hits no mesh (Canvas-level only) |
| `onWheel` | Scroll wheel |
| `onUpdate` | Object receives new props |

**Event object properties:**

| Property | Type | Description |
|----------|------|-------------|
| `object` | `THREE.Object3D` | The mesh actually hit |
| `eventObject` | `THREE.Object3D` | The object that has the event handler |
| `point` | `THREE.Vector3` | Intersection point in world space |
| `distance` | `number` | Distance from camera to intersection |
| `uv` | `THREE.Vector2` | UV coordinates at intersection |
| `face` | `THREE.Face` | Intersected face |
| `faceIndex` | `number` | Index of the intersected face |
| `unprojectedPoint` | `THREE.Vector3` | Camera-unprojected point |
| `ray` | `THREE.Ray` | The ray used for intersection |
| `camera` | `THREE.Camera` | The active camera |
| `intersections` | `Intersection[]` | All intersected objects |
| `delta` | `number` | Pixel distance between pointerdown and pointerup |
| `sourceEvent` | `Event` | Original DOM event |
| `stopPropagation()` | `function` | Prevents bubbling; blocks events to occluded objects |

### 1.6 Disposal and Performance

- R3F ALWAYS calls `dispose()` on Three.js objects when components unmount. This frees GPU resources (geometries, materials, textures).
- Set `dispose={null}` on an element or its parent to PREVENT auto-disposal. Use this when objects are shared across components.
- ALWAYS use `useMemo` for geometries and materials created imperatively to prevent recreation on every render.
- `frameloop="demand"` with `invalidate()` is the recommended pattern for static scenes (dashboards, configurators).
- `createPortal(children, scene)` renders children into a different scene/layer without affecting the main scene graph.

---

## 2. Drei Deep Dive

**Source:** https://drei.docs.pmnd.rs/

Drei is a collection of 150+ useful helpers and abstractions for React Three Fiber. It is maintained by the pmndrs collective.

### 2.1 Controls

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `CameraControls` | Full-featured camera control (recommended) | `minDistance`, `maxDistance`, `minPolarAngle`, `maxPolarAngle`, `smoothTime` |
| `ScrollControls` | Scroll-driven scene animation | `pages`, `damping`, `distance`, `horizontal`, `enabled` |
| `PresentationControls` | Drag-to-rotate with spring physics | `snap`, `global`, `cursor`, `speed`, `zoom`, `rotation`, `polar`, `azimuth`, `config` |
| `KeyboardControls` | Keyboard state as context | `map` (key-to-name mapping array) |
| `FaceControls` | Face-tracking camera control | `camera`, `autostart`, `offsetScaleFactor` |
| `MotionPathControls` | Animate camera along a path | `curves`, `damping`, `focus`, `offset` |

### 2.2 Gizmos

| Component | Purpose |
|-----------|---------|
| `GizmoHelper` | Viewport orientation widget container |
| `PivotControls` | Interactive 3D pivot gizmo with translate/rotate/scale |
| `DragControls` | Drag objects in 3D space |
| `TransformControls` | Translate/rotate/scale gizmo (wraps Three.js TransformControls) |
| `Grid` | Infinite configurable grid plane |
| `Helper / useHelper` | Visualize light helpers, camera helpers, etc. |

### 2.3 Staging and Environment

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `Environment` | HDR environment maps | `preset` (apartment, city, dawn, forest, lobby, night, park, studio, sunset, warehouse), `files`, `path`, `background`, `blur`, `resolution` |
| `Lightformer` | Custom light shapes for Environment | `form` (circle, ring, rect), `intensity`, `color`, `position`, `scale`, `rotation` |
| `AccumulativeShadows` | Soft baked shadows | `frames`, `alphaTest`, `scale`, `position`, `opacity`, `color` |
| `ContactShadows` | Contact shadow plane | `opacity`, `scale`, `blur`, `far`, `resolution`, `color`, `frames` |
| `RandomizedLight` | Randomized light for AccumulativeShadows | `amount`, `radius`, `intensity`, `ambient`, `position`, `bias` |
| `Sky` | Procedural sky dome | `distance`, `sunPosition`, `inclination`, `azimuth`, `turbidity`, `rayleigh` |
| `Stars` | Particle starfield | `radius`, `depth`, `count`, `factor`, `saturation`, `fade`, `speed` |
| `Sparkles` | Floating sparkle particles | `count`, `size`, `speed`, `opacity`, `scale`, `color`, `noise` |
| `Cloud` | Volumetric cloud | `opacity`, `speed`, `segments`, `bounds`, `volume`, `color`, `fade` |
| `Stage` | Complete lighting/shadow setup | `preset`, `intensity`, `environment`, `shadows`, `adjustCamera` |
| `Backdrop` | Curved backdrop plane | `receiveShadow`, `floor`, `segments` |
| `CameraShake` | Camera shake effect | `maxYaw`, `maxPitch`, `maxRoll`, `yawFrequency`, `pitchFrequency`, `rollFrequency`, `intensity`, `decay` |
| `Float` | Floating animation | `speed`, `rotationIntensity`, `floatIntensity`, `floatingRange` |
| `Center` | Center children on mount | `top`, `right`, `bottom`, `left`, `front`, `back`, `precise` |
| `Resize` | Normalize children to unit size | `width`, `height`, `depth` |
| `Bounds` | Auto-fit camera to content | `fit`, `clip`, `observe`, `margin` |
| `BBAnchor` | Anchor to bounding box position | `anchor` (e.g., `[0, 1, 0]` for top) |
| `Shadow` | Simple shadow plane | `color`, `colorStop`, `opacity`, `fog` |
| `Caustics` | Caustic light patterns | `color`, `worldRadius`, `ior`, `intensity`, `backsideIOR` |
| `SpotLight` | Volumetric spot light | `color`, `intensity`, `angle`, `penumbra`, `distance`, `attenuation`, `anglePower`, `opacity` |
| `useEnvironment` | Hook to load environment maps | Same options as Environment component |
| `MatcapTexture / useMatcapTexture` | Load matcap textures by ID | `id`, `format`, `resolution` |
| `NormalTexture / useNormalTexture` | Load normal textures by ID | `id`, `repeat` |
| `ShadowAlpha` | Shadow transparency control | `opacity` |

### 2.4 Text and HTML

| Component | Purpose | Key Props |
|-----------|---------|-----------|
| `Text` | SDF text rendering (troika-three-text) | `font`, `fontSize`, `color`, `maxWidth`, `lineHeight`, `letterSpacing`, `textAlign`, `anchorX`, `anchorY`, `outlineWidth`, `outlineColor` |
| `Text3D` | Extruded 3D text geometry | `font` (JSON font), `size`, `height`, `bevelEnabled`, `bevelSize`, `bevelThickness`, `curveSegments`, `letterSpacing`, `lineHeight` |
| `Billboard` | Always faces camera | `follow`, `lockX`, `lockY`, `lockZ` |
| `Html` | DOM element in 3D space | `as`, `portal`, `transform`, `sprite`, `distanceFactor`, `center`, `occlude`, `zIndexRange`, `prepend`, `fullscreen`, `className`, `style` |
| `ScreenSpace` | Render in screen space | — |
| `ScreenSizer` | Scale to fixed screen size | `scale` |
| `Hud` | Heads-up display (separate scene) | `renderPriority` |
| `Svg` | SVG as 3D geometry | `src`, `fillMaterial`, `strokeMaterial` |
| `AsciiRenderer` | ASCII art post-processing | `characters`, `invert`, `color`, `fontSize`, `cellSize`, `resolution` |

### 2.5 Materials

| Component | Purpose |
|-----------|---------|
| `MeshReflectorMaterial` | Reflective floor/surface with blur, distortion, and resolution control |
| `MeshTransmissionMaterial` | Glass-like transmission with chromatic aberration, distortion, thickness |
| `MeshRefractionMaterial` | Refraction through geometry using environment map |
| `MeshWobbleMaterial` | Animated wobble distortion on MeshStandardMaterial |
| `MeshDistortMaterial` | Perlin noise distortion on MeshStandardMaterial |
| `MeshDiscardMaterial` | Renders nothing (useful for shadow-only objects) |
| `PointMaterial` | Rounded point material with size attenuation |
| `SoftShadows` | PCSS soft shadow implementation |
| `shaderMaterial` | Helper to create custom ShaderMaterial as JSX element |

### 2.6 Loaders

| Hook | Loader | Returns |
|------|--------|---------|
| `useGLTF(url)` | GLTFLoader | `{ nodes, materials, scene, animations }` |
| `useTexture(url)` | TextureLoader | `THREE.Texture` or `THREE.Texture[]` |
| `useFBX(url)` | FBXLoader | `THREE.Group` |
| `useKTX2(url)` | KTX2Loader | Compressed texture |
| `useFont(url)` | FontLoader | Font data for Text3D |
| `useAnimations(animations, ref)` | — | `{ actions, names, mixer, ref }` |
| `useSpriteLoader(url)` | — | Sprite sheet data |
| `useVideoTexture(url)` | — | `THREE.VideoTexture` |
| `useTrailTexture(config)` | — | Trail canvas texture |

All loader hooks are Suspense-based. ALWAYS wrap consuming components in `<Suspense>`. All support `.preload(url)` for anticipatory loading. `useGLTF.preload(url)` is the recommended pattern for critical assets.

### 2.7 Performance

| Component/Hook | Purpose |
|----------------|---------|
| `Instances` | GPU instancing for identical meshes — ALWAYS use for >100 identical objects |
| `Merged` | Merges multiple geometries into one draw call |
| `Points` | Efficient point cloud rendering |
| `Segments` | Efficient line segment rendering |
| `Detailed` | LOD (Level of Detail) — switches geometry by distance |
| `Preload` | Preloads assets on mount |
| `BakeShadows` | Bakes shadow maps once, then stops updating |
| `meshBounds` | Fast raycast using bounding box (replaces mesh raycasting) |
| `AdaptiveDpr` | Lowers DPR when performance drops |
| `AdaptiveEvents` | Reduces event frequency when performance drops |
| `Bvh` | BVH-accelerated raycasting for complex meshes |
| `PerformanceMonitor` | Monitors FPS and triggers regression callbacks |

### 2.8 Abstractions

| Component | Purpose |
|-----------|---------|
| `Edges` | Renders wireframe edges of geometry |
| `Outlines` | Screen-space outline effect |
| `Trail` | Motion trail behind moving objects |
| `MarchingCubes` | Metaball/marching cubes implementation |
| `Decal` | Project texture onto mesh surface |
| `Splat` | Gaussian splatting renderer |
| `Sampler` | Sample points on mesh surface |
| `CurveModifier` | Deform mesh along curve |
| `Clone` | Deep clone of Object3D with shared geometry/materials |
| `useIntersect` | Visibility detection via intersection observer |
| `GradientTexture` | Procedural gradient texture |
| `Image` | Texture-mapped plane with shader effects |
| `PositionalAudio` | 3D positional audio source |
| `ComputedAttribute` | Derive buffer attributes from geometry |

### 2.9 Portals

| Component | Purpose |
|-----------|---------|
| `Hud` | Separate orthographic scene overlaying the main view |
| `View` | Multiple viewports in a single canvas |
| `RenderTexture` | Render scene to texture for use as material map |
| `RenderCubeTexture` | Render to cube map texture |
| `Fisheye` | Fisheye lens distortion portal |
| `Mask` | Stencil masking between components |
| `MeshPortalMaterial` | Portal effect — renders a scene inside a mesh surface |

---

## 3. WebGPU and TSL (Three Shading Language)

**Source:** https://threejs.org/docs/, https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language

### 3.1 WebGPURenderer

WebGPURenderer is Three.js's modern renderer that targets the WebGPU API with automatic WGSL shader generation. It falls back to WebGL2 when WebGPU is unavailable.

**Async initialization pattern (REQUIRED):**
```javascript
import * as THREE from 'three/webgpu';

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // MUST await before first render
```

NEVER call `renderer.render()` before `await renderer.init()` completes.

**Browser support matrix:**
| Browser | Version | Status |
|---------|---------|--------|
| Chrome | 113+ | Full support |
| Edge | 113+ | Full support |
| Safari | 18+ | Supported |
| Firefox | Experimental | Behind flag (`dom.webgpu.enabled`) |

**Feature detection:**
```javascript
import { WebGPU } from 'three/webgpu';
if (WebGPU.isAvailable()) {
  // Use WebGPURenderer
} else {
  // Fallback to WebGLRenderer
}
```

### 3.2 Node Materials

Every classic Three.js material has a node-based equivalent for use with WebGPURenderer:

| Classic Material | Node Material |
|-----------------|---------------|
| `MeshBasicMaterial` | `MeshBasicNodeMaterial` |
| `MeshStandardMaterial` | `MeshStandardNodeMaterial` |
| `MeshPhysicalMaterial` | `MeshPhysicalNodeMaterial` |
| `MeshPhongMaterial` | `MeshPhongNodeMaterial` |
| `MeshLambertMaterial` | `MeshLambertNodeMaterial` |
| `LineBasicMaterial` | `LineBasicNodeMaterial` |
| `PointsMaterial` | `PointsNodeMaterial` |
| `SpriteMaterial` | `SpriteNodeMaterial` |
| `LineDashedMaterial` | `LineDashedNodeMaterial` |

**NodeMaterial inputs (all optional, override defaults):**

- `.colorNode` (vec4) — base color
- `.opacityNode` (float) — opacity
- `.normalNode` (vec3) — normal map replacement
- `.emissiveNode` (color) — emissive output
- `.metalnessNode` (float) — metalness (Standard/Physical)
- `.roughnessNode` (float) — roughness (Standard/Physical)
- `.envNode` (color) — environment map
- `.lightsNode` — lighting model override
- `.positionNode` (vec3) — vertex displacement
- `.fragmentNode` (vec4) — full fragment shader replacement
- `.vertexNode` (vec4) — full vertex shader replacement
- `.outputNode` (vec4) — final output override
- `.aoNode` (float) — ambient occlusion
- `.alphaTestNode` (float) — alpha test threshold
- `.depthNode` (float) — custom depth
- `.castShadowNode` (vec4) — shadow casting override

Physical-specific: `.clearcoatNode`, `.clearcoatRoughnessNode`, `.clearcoatNormalNode`, `.sheenNode`, `.iridescenceNode`, `.iridescenceIORNode`, `.iridescenceThicknessNode`, `.specularIntensityNode`, `.specularColorNode`, `.iorNode`, `.transmissionNode`, `.thicknessNode`, `.attenuationDistanceNode`, `.attenuationColorNode`, `.dispersionNode`, `.anisotropyNode`.

### 3.3 TSL (Three Shading Language)

TSL is a JavaScript-based node graph system that compiles to both GLSL (WebGL2) and WGSL (WebGPU). NEVER write raw GLSL/WGSL strings — ALWAYS use TSL functions.

**Type system:**
- Scalars: `float()`, `int()`, `uint()`, `bool()`
- Vectors: `vec2()`, `vec3()`, `vec4()`, `ivec2()`, `ivec3()`, `ivec4()`, `uvec2()`, `uvec3()`, `uvec4()`
- Matrices: `mat2()`, `mat3()`, `mat4()`
- Color: `color()`
- Conversion: `.toFloat()`, `.toVec3()`, `.toColor()`, etc.

**Variables and uniforms:**
- `uniform(value)` — GPU-side dynamic value, supports `.onRenderUpdate(fn)`, `.onFrameUpdate(fn)`, `.onObjectUpdate(fn)`
- `toVar(node)` — reusable shader variable
- `toConst(node)` — inline constant
- `varying(node)` — vertex-to-fragment interpolation
- `vertexStage(node)` — force computation in vertex shader
- `attribute(name, type)` — access buffer attributes

**Operators (all chainable):**
- Arithmetic: `.add()`, `.sub()`, `.mul()`, `.div()`, `.mod()`
- Comparison: `.equal()`, `.notEqual()`, `.lessThan()`, `.greaterThan()`, `.lessThanEqual()`, `.greaterThanEqual()`
- Logical: `.and()`, `.or()`, `.not()`, `.xor()`
- Assignment: `.assign()`, `.addAssign()`, `.subAssign()`, `.mulAssign()`, `.divAssign()`
- Bitwise: `.bitAnd()`, `.bitOr()`, `.bitXor()`, `.shiftLeft()`, `.shiftRight()`

**Math library:**
`abs()`, `acos()`, `asin()`, `atan()`, `ceil()`, `clamp()`, `cos()`, `cross()`, `degrees()`, `distance()`, `dot()`, `exp()`, `floor()`, `fract()`, `inverseSqrt()`, `length()`, `log()`, `max()`, `min()`, `mix()`, `normalize()`, `pow()`, `radians()`, `reflect()`, `refract()`, `round()`, `saturate()`, `sign()`, `sin()`, `smoothstep()`, `sqrt()`, `step()`, `tan()`, `trunc()`, `faceforward()`, `dFdx()`, `dFdy()`, `fwidth()`, `negate()`, `oneMinus()`, `reciprocal()`, `cbrt()`, `pow2()`, `pow3()`, `pow4()`

Constants: `EPSILON`, `INFINITY`, `PI`, `TWO_PI`, `HALF_PI`

**Geometry nodes:**
- Position: `positionGeometry`, `positionLocal`, `positionWorld`, `positionView`, `positionWorldDirection`, `positionViewDirection`
- Normal: `normalGeometry`, `normalLocal`, `normalView`, `normalWorld`
- Tangent: `tangentGeometry`, `tangentLocal`, `tangentView`, `tangentWorld`
- Bitangent: `bitangentGeometry`, `bitangentLocal`, `bitangentView`, `bitangentWorld`
- UV: `uv(index)` — access UV channels
- Screen: `screenUV`, `screenCoordinate`, `screenSize`
- Viewport: `viewportUV`, `viewportCoordinate`, `viewportSize`

**Camera nodes:** `cameraNear`, `cameraFar`, `cameraPosition`, `cameraProjectionMatrix`, `cameraProjectionMatrixInverse`, `cameraViewMatrix`, `cameraWorldMatrix`, `cameraNormalMatrix`

**Model nodes:** `modelViewMatrix`, `modelNormalMatrix`, `modelWorldMatrix`, `modelPosition`, `modelScale`, `modelDirection`, `modelViewPosition`, `modelWorldMatrixInverse`

**Texture operations:**
- `texture(tex, uv, level)` — sample with interpolation
- `textureLoad(tex, uv, level)` — sample without interpolation
- `textureStore(tex, uv, value)` — write to storage texture
- `textureSize(tex, level)` — get texture dimensions
- `cubeTexture(tex, uvw, level)` — sample cube map
- `texture3D(tex, uvw, level)` — sample 3D texture
- `triplanarTexture(texX, texY, texZ, scale, position, normal)` — triplanar mapping
- `textureBicubic(textureNode, strength)` — bicubic interpolation

**Animation nodes:** `time` (elapsed seconds), `deltaTime` (frame delta), `oscSine(timer)`, `oscSquare(timer)`, `oscTriangle(timer)`, `oscSawtooth(timer)`

**Randomization:** `hash(seed)` (deterministic 0-1), `range(min, max)` (attribute-based)

**Control flow:**
- `If(condition, fn).ElseIf(condition, fn).Else(fn)` — ALWAYS use capital `If`
- `Switch(value).Case(val, fn).Default(fn)` — no fallthrough
- `select(condition, trueVal, falseVal)` — ternary
- `Loop(count, ({i}) => {})` — supports nested, while-style
- `Break()`, `Continue()`, `Discard()`, `Return()`

**Function definition:**
```javascript
const myFn = Fn(([param1, param2]) => {
  return param1.add(param2);
});
```

**Post-processing nodes (WebGPU pipeline):**
`bloom()`, `dof()`, `fxaa()`, `smaa()`, `gaussianBlur()`, `ssr()`, `ssgi()`, `ao()`, `chromaticAberration()`, `film()`, `dotScreen()`, `sobel()`, `afterImage()`, `anamorphic()`, `denoise()`, `lut3D()`, `motionBlur()`, `outline()`, `rgbShift()`, `transition()`, `traa()`, `renderOutput()`

**Color operations:** `luminance()`, `saturation()`, `vibrance()`, `hue()`, `posterize()`, `grayscale()`, `sepia()`

**Blend modes:** `blendBurn()`, `blendDodge()`, `blendOverlay()`, `blendScreen()`, `blendColor()`

### 3.4 Compute Shaders

WebGPU enables general-purpose GPU compute via TSL:

```javascript
const computeNode = compute(shaderFn, count, workgroupSize);
renderer.computeAsync(computeNode);
```

**Storage:** `storage(attribute, type, count)`, `storageTexture(texture)`

**Atomic operations:** `atomicAdd()`, `atomicSub()`, `atomicMax()`, `atomicMin()`, `atomicAnd()`, `atomicOr()`, `atomicXor()`, `atomicStore()`, `atomicLoad()`

**Barriers:** `workgroupBarrier()`, `storageBarrier()`, `textureBarrier()`, `barrier()`

**Built-in IDs:** `workgroupId`, `localId`, `globalId`, `numWorkgroups`, `subgroupSize`

### 3.5 WebGL to WebGPU Migration

1. Replace `import * as THREE from 'three'` with `import * as THREE from 'three/webgpu'`
2. Replace `new WebGLRenderer()` with `new WebGPURenderer()` and add `await renderer.init()`
3. Replace classic materials with NodeMaterial equivalents (or keep classic — they auto-convert)
4. Replace EffectComposer with RenderPipeline and TSL post-processing nodes
5. Replace raw GLSL ShaderMaterial with TSL-based NodeMaterial
6. ALWAYS use `renderer.setAnimationLoop()` instead of `requestAnimationFrame` for WebGPU

---

## 4. WebXR / VR / AR

**Source:** https://threejs.org/docs/#api/en/renderers/WebXRManager

### 4.1 WebXRManager

Accessed via `renderer.xr`. Manages WebXR session lifecycle.

**Properties:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enabled` | `boolean` | `false` | Enable XR rendering |
| `isPresenting` | `boolean` | read-only | Whether an XR session is active |
| `cameraAutoUpdate` | `boolean` | `true` | Auto-update camera from XR device |

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `getSession()` | `() => XRSession \| null` | Current XR session |
| `setSessionInit(options)` | `(XRSessionInit) => void` | Configure session features |
| `setReferenceSpaceType(type)` | `(string) => void` | Set reference space |
| `getController(index)` | `(number) => Group` | Get controller target ray space |
| `getControllerGrip(index)` | `(number) => Group` | Get controller grip space |
| `getHand(index)` | `(number) => Group` | Get hand tracking group |
| `setFoveation(level)` | `(number) => void` | Set foveated rendering (0.0-1.0) |
| `getFoveation()` | `() => number` | Get current foveation |
| `getEnvironmentBlendMode()` | `() => string` | Get blend mode (opaque, additive, alpha-blend) |
| `setFramebufferScaleFactor(scale)` | `(number) => void` | Adjust render resolution |

**Reference space types:** `'viewer'`, `'local'`, `'local-floor'`, `'bounded-floor'`, `'unbounded'`

**Session types:** `'immersive-vr'`, `'immersive-ar'`, `'inline'`

### 4.2 VR Setup Pattern

```javascript
import { VRButton } from 'three/addons/webxr/VRButton.js';
import { XRControllerModelFactory } from 'three/addons/webxr/XRControllerModelFactory.js';

renderer.xr.enabled = true;
renderer.xr.setReferenceSpaceType('local-floor');
document.body.appendChild(VRButton.createButton(renderer));

// MUST use setAnimationLoop — NEVER use requestAnimationFrame for XR
renderer.setAnimationLoop((time, frame) => {
  renderer.render(scene, camera);
});
```

### 4.3 XR Controllers

```javascript
const controller0 = renderer.xr.getController(0);
const controller1 = renderer.xr.getController(1);
scene.add(controller0, controller1);

// Controller events
controller0.addEventListener('selectstart', onSelectStart);
controller0.addEventListener('selectend', onSelectEnd);
controller0.addEventListener('select', onSelect);
controller0.addEventListener('squeezestart', onSqueezeStart);
controller0.addEventListener('squeezeend', onSqueezeEnd);
controller0.addEventListener('squeeze', onSqueeze);
controller0.addEventListener('connected', (event) => {
  // event.data contains XRInputSource
});
controller0.addEventListener('disconnected', () => {});

// Controller models
const gripSpace0 = renderer.xr.getControllerGrip(0);
const factory = new XRControllerModelFactory();
gripSpace0.add(factory.createControllerModel(gripSpace0));
scene.add(gripSpace0);
```

### 4.4 Hand Tracking

```javascript
import { XRHandModelFactory } from 'three/addons/webxr/XRHandModelFactory.js';

const hand0 = renderer.xr.getHand(0);
const handFactory = new XRHandModelFactory();
hand0.add(handFactory.createHandModel(hand0, 'mesh')); // 'mesh', 'spheres', or 'boxes'
scene.add(hand0);
```

### 4.5 AR Hit Testing

```javascript
renderer.xr.setSessionInit({
  requiredFeatures: ['hit-test'],
  optionalFeatures: ['dom-overlay'],
  domOverlay: { root: document.getElementById('overlay') }
});

document.body.appendChild(ARButton.createButton(renderer, {
  requiredFeatures: ['hit-test']
}));

// In animation loop with XR frame:
renderer.setAnimationLoop((time, frame) => {
  if (frame) {
    const hitTestSource = /* obtained via frame.requestHitTestSource() */;
    const results = frame.getHitTestResults(hitTestSource);
    if (results.length > 0) {
      const hit = results[0];
      const pose = hit.getPose(referenceSpace);
      reticle.matrix.fromArray(pose.transform.matrix);
      reticle.visible = true;
    }
  }
  renderer.render(scene, camera);
});
```

### 4.6 XR Addons

| Class | Purpose |
|-------|---------|
| `VRButton` | Creates "Enter VR" button with feature detection |
| `ARButton` | Creates "Enter AR" button with feature detection |
| `XRControllerModelFactory` | Loads appropriate controller 3D model |
| `XRHandModelFactory` | Creates hand tracking visualization |
| `XRHandPrimitiveModel` | Simple geometric hand representation |
| `XREstimatedLight` | Uses AR environment lighting estimation |
| `XRPlanes` | AR plane detection visualization |
| `WebXRDepthSensing` | Access depth buffer from AR device |

### 4.7 VR Performance Requirements

- Target 72fps (Quest) or 90fps (PC VR) ALWAYS — dropped frames cause nausea.
- Set `renderer.xr.setFoveation(1.0)` for maximum foveated rendering performance.
- Use `renderer.xr.setFramebufferScaleFactor(0.75)` to reduce resolution if needed.
- NEVER use heavy post-processing in VR — it doubles the workload (one render per eye).
- Use `Instances` and `Merged` from Drei aggressively to minimize draw calls.

---

## 5. IFC/BIM Viewer

### 5.1 web-ifc (Low-Level WASM Parser)

**Source:** https://github.com/IFCjs/web-ifc

web-ifc is a low-level IFC file parser compiled to WebAssembly from C++. Current stable version: 0.77.

**Initialization (ALWAYS async):**
```javascript
import * as WebIFC from 'web-ifc';
const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath('./'); // Path to web-ifc.wasm
await ifcApi.Init();
```

**Core IfcAPI methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `Init()` | `() => Promise<void>` | Initialize WASM module |
| `SetWasmPath(path)` | `(string) => void` | Set path to .wasm file |
| `OpenModel(data, settings?)` | `(Uint8Array, object?) => number` | Load IFC data, returns modelID |
| `CloseModel(modelID)` | `(number) => void` | Release model and memory |
| `GetGeometry(modelID, expressID)` | `(number, number) => object` | Get geometry for element |
| `GetFlatMesh(modelID, expressID)` | `(number, number) => FlatMesh` | Get flattened mesh data |
| `GetPlacedGeometry(modelID, pg)` | `(number, PlacedGeometry) => MeshData` | Get positioned geometry with transform |
| `GetLine(modelID, expressID, flatten?)` | `(number, number, boolean?) => object` | Read single IFC entity |
| `GetAllLines(modelID)` | `(number) => number[]` | Get all express IDs |
| `GetLineIDsWithType(modelID, type)` | `(number, number) => number[]` | Get IDs by IFC type |
| `GetAllTypesOfModel(modelID)` | `(number) => TypeInfo[]` | List all entity types in model |

**Geometry extraction pattern:**
```javascript
const modelID = ifcApi.OpenModel(ifcData);
const allWalls = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
for (const wallID of allWalls) {
  const flatMesh = ifcApi.GetFlatMesh(modelID, wallID);
  for (const pg of flatMesh.geometries) {
    const meshData = ifcApi.GetPlacedGeometry(modelID, pg);
    // meshData.vertexData contains Float32Array (position + normal)
    // meshData.indexData contains Uint32Array (triangle indices)
    // pg.flatTransformation contains Float64Array (4x4 matrix)
    // pg.color contains { x: r, y: g, z: b, w: a }
  }
}
```

**Anti-patterns:**
- NEVER load entire IFC geometry at once for large files (>50MB). Stream or load on demand.
- NEVER forget to call `CloseModel()` — WASM memory leaks are permanent until page reload.
- NEVER use the deprecated `web-ifc-three` package — it is unmaintained.

### 5.2 @thatopen/components (That Open Engine)

**Source:** https://github.com/ThatOpen/engine_components

A higher-level BIM toolkit built on Three.js. Current version: v3.3.2.

**LICENSE: MIT** (previously AGPL-3.0 for older IFC.js versions; the @thatopen packages use MIT).

**Package structure:**
- `@thatopen/components` — core (works in browser and Node.js)
- `@thatopen/components-front` — browser-specific features (UI, advanced visualization)

**Architecture:**
```javascript
import * as OBC from '@thatopen/components';

const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();

// Scene
world.scene = new OBC.SimpleScene(components);
world.scene.setup();

// Camera
world.camera = new OBC.SimpleCamera(components);
world.camera.controls.setLookAt(10, 10, 10, 0, 0, 0);

// Renderer
world.renderer = new OBC.SimpleRenderer(components, container);

// Initialize
components.init();
```

**Key components:**

| Component | Purpose |
|-----------|---------|
| `Components` | Central manager, orchestrates all subsystems |
| `Worlds` | Multi-world environment management |
| `SimpleScene` | Three.js scene wrapper |
| `SimpleCamera` | Camera with built-in controls |
| `SimpleRenderer` | WebGL rendering to DOM |
| `IfcLoader` | IFC file loading and parsing |
| `FragmentsManager` | Efficient geometry batching via fragments |
| `Highlighter` | Element selection and highlighting |
| `Clipper` | Section plane tools |
| `Plans` | Floor plan generation |

**Fragment system:** Converts IFC geometry into optimized "fragments" — batched geometries that minimize draw calls. This is critical for large BIM models (10,000+ elements).

**Anti-patterns:**
- NEVER use the deprecated `web-ifc-three` or `openbim-components` (v1) packages.
- NEVER skip `components.init()` — all subsystems require initialization.
- ALWAYS call `components.dispose()` when unmounting to prevent memory leaks.

---

## 6. Controls Deep Dive

**Source:** https://threejs.org/docs/#examples/en/controls/

### 6.1 OrbitControls

**Import:** `import { OrbitControls } from 'three/addons/controls/OrbitControls.js'`

**Constructor:** `new OrbitControls(camera: Camera, domElement: HTMLElement)`

**All properties:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `autoRotate` | `boolean` | `false` | Auto-rotate around target |
| `autoRotateSpeed` | `number` | `2.0` | Rotation speed (degrees/sec at 60fps) |
| `cursor` | `string` | `'auto'` | CSS cursor style |
| `dampingFactor` | `number` | `0.05` | Inertia factor (0-1), requires `enableDamping` |
| `domElement` | `HTMLElement` | — | Event listener target |
| `enabled` | `boolean` | `true` | Enable/disable all interaction |
| `enableDamping` | `boolean` | `false` | Smooth inertial movement |
| `enablePan` | `boolean` | `true` | Allow panning |
| `enableRotate` | `boolean` | `true` | Allow rotation |
| `enableZoom` | `boolean` | `true` | Allow zooming |
| `keys` | `object` | `{ LEFT, UP, RIGHT, BOTTOM }` | Arrow key codes |
| `maxAzimuthAngle` | `number` | `Infinity` | Max horizontal angle (radians) |
| `maxDistance` | `number` | `Infinity` | Max dolly distance (PerspectiveCamera) |
| `maxPolarAngle` | `number` | `Math.PI` | Max vertical angle (radians) |
| `maxTargetRadius` | `number` | `Infinity` | Max distance target can move from origin |
| `maxZoom` | `number` | `Infinity` | Max zoom (OrthographicCamera) |
| `minAzimuthAngle` | `number` | `-Infinity` | Min horizontal angle (radians) |
| `minDistance` | `number` | `0` | Min dolly distance (PerspectiveCamera) |
| `minPolarAngle` | `number` | `0` | Min vertical angle (radians) |
| `minTargetRadius` | `number` | `0` | Min distance target can move from origin |
| `minZoom` | `number` | `0` | Min zoom (OrthographicCamera) |
| `mouseButtons` | `object` | `{ LEFT: ROTATE, MIDDLE: DOLLY, RIGHT: PAN }` | Mouse button mapping |
| `panSpeed` | `number` | `1.0` | Pan speed multiplier |
| `rotateSpeed` | `number` | `1.0` | Rotation speed multiplier |
| `screenSpacePanning` | `boolean` | `true` | Pan in screen plane (true) or horizontal plane (false) |
| `target` | `Vector3` | `(0, 0, 0)` | Orbit focus point |
| `touches` | `object` | `{ ONE: ROTATE, TWO: DOLLY_PAN }` | Touch gesture mapping |
| `zoomSpeed` | `number` | `1.0` | Zoom speed multiplier |
| `zoomToCursor` | `boolean` | `false` | Zoom towards cursor position |

**All methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `update(deltaTime?)` | `(number?) => boolean` | Update camera. MUST call in animation loop when `enableDamping` or `autoRotate` is true |
| `reset()` | `() => void` | Reset to saved state |
| `saveState()` | `() => void` | Save current camera state |
| `dispose()` | `() => void` | Remove all event listeners |
| `getDistance()` | `() => number` | Current distance from target |
| `getAzimuthalAngle()` | `() => number` | Horizontal angle (radians) |
| `getPolarAngle()` | `() => number` | Vertical angle (radians) |
| `listenToKeyEvents(domElement)` | `(HTMLElement) => void` | Enable keyboard controls |
| `stopListenToKeyEvents()` | `() => void` | Disable keyboard controls |

**Events:** `change` (camera moved), `start` (interaction began), `end` (interaction ended).

ALWAYS call `controls.update()` in the animation loop when `enableDamping` is true. Failing to do so causes the camera to freeze after interaction ends.

### 6.2 MapControls

**Import:** `import { MapControls } from 'three/addons/controls/MapControls.js'`

MapControls is a subclass of OrbitControls with different default mouse button mapping optimized for map-like navigation:

| Button | OrbitControls | MapControls |
|--------|--------------|-------------|
| Left | Rotate | Pan |
| Middle | Dolly | Dolly |
| Right | Pan | Rotate |

Additionally, `screenSpacePanning` defaults to `true` in MapControls. All other properties and methods are identical to OrbitControls.

### 6.3 FlyControls

**Import:** `import { FlyControls } from 'three/addons/controls/FlyControls.js'`

**Constructor:** `new FlyControls(camera: Camera, domElement: HTMLElement)`

Free-flight camera controller with six degrees of freedom.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `movementSpeed` | `number` | `1.0` | Translation speed |
| `rollSpeed` | `number` | `0.005` | Roll rotation speed |
| `dragToLook` | `boolean` | `false` | Require mouse drag to look (vs. always track mouse) |
| `autoForward` | `boolean` | `false` | Automatically move forward |

**Methods:** `update(delta)` — MUST pass delta time. `dispose()` — cleanup.

**Keyboard controls:** WASD for movement, QE for roll, RF for up/down.

### 6.4 PointerLockControls

**Import:** `import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js'`

**Constructor:** `new PointerLockControls(camera: Camera, domElement: HTMLElement)`

First-person camera using the Pointer Lock API. The mouse cursor is hidden and captured.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `isLocked` | `boolean` | read-only | Whether pointer is locked |
| `maxPolarAngle` | `number` | `Math.PI` | Max vertical look angle |
| `minPolarAngle` | `number` | `0` | Min vertical look angle |
| `pointerSpeed` | `number` | `1.0` | Mouse sensitivity |

| Method | Description |
|--------|-------------|
| `lock()` | Request pointer lock (MUST be called from user gesture) |
| `unlock()` | Exit pointer lock |
| `connect()` | Attach event listeners |
| `disconnect()` | Remove event listeners |
| `dispose()` | Full cleanup |
| `getObject()` | Returns the controlled camera |
| `getDirection(target: Vector3)` | Get look direction |
| `moveForward(distance)` | Move camera forward |
| `moveRight(distance)` | Move camera right |

**Events:** `change` (camera moved), `lock` (pointer locked), `unlock` (pointer released).

ALWAYS call `lock()` from a click handler — browsers require a user gesture for Pointer Lock API.

### 6.5 TransformControls

**Import:** `import { TransformControls } from 'three/addons/controls/TransformControls.js'`

**Constructor:** `new TransformControls(camera: Camera, domElement: HTMLElement)`

Interactive gizmo for translating, rotating, and scaling objects.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `axis` | `string \| null` | `null` | Constrain to axis ('X', 'Y', 'Z', 'XY', 'YZ', 'XZ') |
| `dragging` | `boolean` | `false` | Whether user is dragging |
| `enabled` | `boolean` | `true` | Enable/disable |
| `mode` | `string` | `'translate'` | Transform mode: 'translate', 'rotate', 'scale' |
| `object` | `Object3D` | `undefined` | Attached object |
| `showX` | `boolean` | `true` | Show X axis gizmo |
| `showY` | `boolean` | `true` | Show Y axis gizmo |
| `showZ` | `boolean` | `true` | Show Z axis gizmo |
| `size` | `number` | `1` | Gizmo scale |
| `space` | `string` | `'world'` | Coordinate space: 'world' or 'local' |
| `translationSnap` | `number \| null` | `null` | Translation snap grid |
| `rotationSnap` | `number \| null` | `null` | Rotation snap (radians) |
| `scaleSnap` | `number \| null` | `null` | Scale snap increment |

| Method | Description |
|--------|-------------|
| `attach(object)` | Attach gizmo to object |
| `detach()` | Remove gizmo from object |
| `setMode(mode)` | Set transform mode |
| `setSpace(space)` | Set coordinate space |
| `setSize(size)` | Set gizmo size |
| `setTranslationSnap(snap)` | Set translation snap |
| `setRotationSnap(snap)` | Set rotation snap |
| `setScaleSnap(snap)` | Set scale snap |
| `getRaycaster()` | Get internal raycaster |
| `getMode()` | Get current mode |
| `dispose()` | Cleanup |

**Events:** `change`, `mouseDown`, `mouseUp`, `objectChange`, `dragging-changed`.

**Critical pattern — disable OrbitControls during transform:**
```javascript
transformControls.addEventListener('dragging-changed', (event) => {
  orbitControls.enabled = !event.value;
});
```
ALWAYS disable other controls while TransformControls is dragging. Failing to do so causes conflicting camera/object movement.

### 6.6 ArcballControls

**Import:** `import { ArcballControls } from 'three/addons/controls/ArcballControls.js'`

Advanced trackball-style controls with animation states and unconstrained rotation. Unlike OrbitControls, ArcballControls has NO polar angle constraint — the camera can rotate freely in all directions. Supports focus animations and cursor-based rotation scaling.

### 6.7 TrackballControls

**Import:** `import { TrackballControls } from 'three/addons/controls/TrackballControls.js'`

Similar to OrbitControls but without the polar angle constraint. The camera can rotate past the poles. Properties include `rotateSpeed`, `zoomSpeed`, `panSpeed`, `noRotate`, `noZoom`, `noPan`, `staticMoving`, `dynamicDampingFactor`.

### 6.8 DragControls

**Import:** `import { DragControls } from 'three/addons/controls/DragControls.js'`

Drag objects in 3D space along a plane. Constructor takes an array of objects, camera, and DOM element.

### 6.9 FirstPersonControls

**Import:** `import { FirstPersonControls } from 'three/addons/controls/FirstPersonControls.js'`

Mouse-look camera control without pointer lock. Properties: `activeLook`, `autoForward`, `constrainVertical`, `heightCoef`, `heightMax`, `heightMin`, `heightSpeed`, `lookSpeed`, `lookVertical`, `mouseDragOn`, `movementSpeed`, `verticalMax`, `verticalMin`.

### 6.10 Complete Controls Import Paths

All controls are imported from the `three/addons/controls/` path (or `three/examples/jsm/controls/` for older setups):

```javascript
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { MapControls } from 'three/addons/controls/MapControls.js';
import { FlyControls } from 'three/addons/controls/FlyControls.js';
import { FirstPersonControls } from 'three/addons/controls/FirstPersonControls.js';
import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';
import { TransformControls } from 'three/addons/controls/TransformControls.js';
import { TrackballControls } from 'three/addons/controls/TrackballControls.js';
import { ArcballControls } from 'three/addons/controls/ArcballControls.js';
import { DragControls } from 'three/addons/controls/DragControls.js';
```

---

## Sources

All information in this document was cross-referenced against the following official sources:

1. React Three Fiber docs — https://r3f.docs.pmnd.rs/
2. Drei docs — https://drei.docs.pmnd.rs/
3. Three.js docs — https://threejs.org/docs/
4. Three.js TSL wiki — https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language
5. web-ifc GitHub — https://github.com/IFCjs/web-ifc
6. That Open Engine — https://github.com/ThatOpen/engine_components
7. Three.js WebXR docs — https://threejs.org/docs/#api/en/renderers/WebXRManager
