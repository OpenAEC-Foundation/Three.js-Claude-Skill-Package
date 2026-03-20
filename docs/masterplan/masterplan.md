# Three.js Skill Package — Definitive Masterplan

## Status

Phase 3 complete. Finalized from raw masterplan after vooronderzoek review.
Date: 2026-03-20

---

## Decisions Made During Refinement

| # | Decision | Rationale |
|---|----------|-----------|
| D-06 | **Added** `threejs-syntax-shaders` | Custom GLSL shaders (ShaderMaterial, RawShaderMaterial, uniforms, ShaderChunk, onBeforeCompile) is a large topic that would exceed 500 lines in materials skill |
| D-07 | **Split** `threejs-impl-react-three-fiber` into R3F + Drei | R3F core (Canvas, hooks, events) is substantial; Drei has 50+ components across 10 categories |
| D-08 | **Added** `threejs-impl-audio` | AudioListener, Audio, PositionalAudio — small but self-contained API surface |
| D-09 | **Added** `threejs-impl-xr` | WebXR/VR/AR with controllers, hand tracking, hit testing — growing importance |
| D-10 | **Added** `threejs-core-raycaster` | Raycaster has own constructor, params, intersection format, InstancedMesh support, layers — enough for standalone skill |

**Result**: 19 raw skills → **24 definitive skills** (0 merges, 5 additions, 0 removals).

---

## Definitive Skill Inventory (24 skills)

### threejs-core/ (4 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `threejs-core-scene-graph` | Object3D hierarchy, Scene, Group, Mesh. Parent-child ops, traversal, layers, visibility, fog, userData, matrix update cycle | Object3D, Scene, Group, Mesh, Layers, Fog, FogExp2 | vooronderzoek-core §1 | M | None |
| `threejs-core-renderer` | WebGLRenderer init, render loop, resize, pixel ratio, tone mapping (7 types), color management, shadow map config, render targets, clipping planes, compileAsync, dispose | WebGLRenderer, WebGLRenderTarget, WebGLCubeRenderTarget | vooronderzoek-core §2 | L | None |
| `threejs-core-math` | Vector3, Matrix4, Quaternion, Euler, Color, MathUtils, Box3, Sphere, Plane, Ray. Coordinate system (Y-up), transform operations | Vector2/3/4, Matrix3/4, Quaternion, Euler, Color, MathUtils, Box3, Sphere | vooronderzoek-core §5 | M | None |
| `threejs-core-raycaster` | Raycaster constructor, params, setFromCamera, intersectObject/intersectObjects, intersection format, InstancedMesh picking, layers filtering, mouse picking pattern | Raycaster, Intersection | vooronderzoek-core §4 | S | core-scene-graph |

### threejs-syntax/ (5 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `threejs-syntax-geometries` | BufferGeometry, BufferAttribute, 21 built-in geometries, InstancedMesh, indexed vs non-indexed, custom geometry, morph targets, groups, disposal | BufferGeometry, BufferAttribute, InstancedMesh, all geometry classes | vooronderzoek-geom-mat §1-3 | L | core-scene-graph, core-math |
| `threejs-syntax-materials` | Material base, PBR (Standard/Physical), all 15+ material types, texture maps (25+ types), color space rules, disposal | All Material subclasses, Texture, TextureLoader | vooronderzoek-geom-mat §4-5 | L | core-scene-graph, core-renderer |
| `threejs-syntax-shaders` | ShaderMaterial, RawShaderMaterial, GLSL uniforms/attributes, ShaderChunk, onBeforeCompile, defines, GLSL3 | ShaderMaterial, RawShaderMaterial, ShaderChunk | vooronderzoek-geom-mat §6 | M | syntax-materials |
| `threejs-syntax-loaders` | GLTFLoader + DRACOLoader + KTX2Loader, TextureLoader, FBXLoader, OBJLoader, RGBELoader, LoadingManager, progress | GLTFLoader, DRACOLoader, KTX2Loader, TextureLoader, FBXLoader, LoadingManager | vooronderzoek-geom-mat §loaders, b1-syntax §3 | M | core-scene-graph |
| `threejs-syntax-controls` | OrbitControls (27 props), MapControls, FlyControls, PointerLockControls, TransformControls, lifecycle, disposal | OrbitControls, MapControls, FlyControls, PointerLockControls, TransformControls | vooronderzoek-ecosystem §6 | S | core-renderer |

### threejs-impl/ (10 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `threejs-impl-lighting` | 7 light types, intensity units (physically correct), PMREMGenerator IBL, HDR env maps, scene.environment, light helpers, performance budget | AmbientLight, DirectionalLight, PointLight, SpotLight, HemisphereLight, RectAreaLight, LightProbe, PMREMGenerator | vooronderzoek-features §1 | M | syntax-materials |
| `threejs-impl-shadows` | 4 shadow map types, per-light shadow config, frustum sizing, bias/normalBias, shadow artifacts (acne, peter panning), CameraHelper debug | LightShadow, DirectionalLightShadow, SpotLightShadow, PointLightShadow, CameraHelper | vooronderzoek-features §2 | M | impl-lighting, core-renderer |
| `threejs-impl-animation` | AnimationMixer, AnimationClip, AnimationAction, KeyframeTrack types, crossfade, additive blending, Clock, GLTF skeletal animation | AnimationMixer, AnimationClip, AnimationAction, KeyframeTrack, Clock | vooronderzoek-features §3 | M | syntax-loaders |
| `threejs-impl-post-processing` | Built-in EffectComposer (27+ passes), pmndrs/postprocessing (21+ effects), custom passes, NEVER mix libraries | EffectComposer, RenderPass, UnrealBloomPass, SSAOPass, pmndrs EffectPass | vooronderzoek-features §4 | L | core-renderer |
| `threejs-impl-physics` | cannon-es (World, Body, Shape, Material, Constraints) and Rapier (WASM, RigidBody, Collider, ray queries). Sync patterns, comparison | cannon-es, @dimforge/rapier3d, World, Body, RigidBody, Collider | vooronderzoek-features §5 | L | core-scene-graph, core-math |
| `threejs-impl-react-three-fiber` | Canvas (17 props), useFrame, useThree, useLoader, JSX mapping, event system, extend(), portals, frameloop, performance | Canvas, useFrame, useThree, useLoader, useGraph | vooronderzoek-ecosystem §1 | L | core-scene-graph |
| `threejs-impl-drei` | 150+ Drei components: controls, environment, text/HTML, materials, loaders (useGLTF, useTexture), staging, performance, shapes, abstractions | @react-three/drei (all component categories) | vooronderzoek-ecosystem §2 | L | impl-react-three-fiber |
| `threejs-impl-webgpu` | WebGPURenderer init, browser detection, node materials, TSL (type system, math, geometry, texture, lighting, compute), migration from WebGL | WebGPURenderer, NodeMaterial types, TSL functions, ComputeNode | vooronderzoek-ecosystem §3 | L | core-renderer, syntax-materials |
| `threejs-impl-audio` | AudioListener on camera, Audio (non-positional), PositionalAudio (3D spatial), AudioLoader, AudioAnalyser, distance models, autoplay policy | AudioListener, Audio, PositionalAudio, AudioLoader, AudioAnalyser | vooronderzoek-features §6 | S | core-scene-graph |
| `threejs-impl-xr` | WebXRManager, VRButton/ARButton, XR controllers/hands, hit testing, teleportation, session types, XR render loop | WebXRManager, VRButton, ARButton, XRControllerModelFactory | vooronderzoek-ecosystem §4 | M | core-renderer |

### threejs-errors/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `threejs-errors-performance` | Memory leaks (dispose patterns), draw call optimization, InstancedMesh, LOD, frustum culling, texture compression, renderer.info, Stats.js | dispose(), InstancedMesh, LOD, Stats, renderer.info | vooronderzoek-core §6, vooronderzoek-geom-mat §anti-patterns | M | core-renderer, syntax-geometries |
| `threejs-errors-rendering` | Black screen, invisible objects, wrong colors (color space), z-fighting, shadow artifacts, needsUpdate missing, WebGL context lost, material side issues | Common error scenarios and fix patterns | all vooronderzoek anti-pattern sections | M | core-renderer, syntax-materials |

### threejs-agents/ (2 skills)

| Name | Scope | Key APIs | Research Input | Complexity | Dependencies |
|------|-------|----------|----------------|------------|-------------|
| `threejs-agents-scene-builder` | Decision tree for scene composition: lighting recipe, material selection, camera setup, post-processing pipeline, environment | All core + syntax + impl patterns | all vooronderzoek | L | ALL other skills |
| `threejs-agents-model-optimizer` | GLTF optimization pipeline: Draco compression, texture resize/compress (KTX2), mesh simplification, LOD generation, bundle analysis | GLTFLoader, DRACOLoader, KTX2Loader, gltf-transform | vooronderzoek-geom-mat, vooronderzoek-features | M | syntax-loaders, errors-performance |

---

## Batch Execution Plan (DEFINITIVE)

| Batch | Skills | Count | Dependencies | Notes |
|-------|--------|-------|-------------|-------|
| 1 | `core-scene-graph`, `core-renderer`, `core-math` | 3 | None | Foundation layer |
| 2 | `core-raycaster`, `syntax-geometries`, `syntax-materials` | 3 | Batch 1 | Core completion + primary syntax |
| 3 | `syntax-shaders`, `syntax-loaders`, `syntax-controls` | 3 | Batch 1-2 | Remaining syntax |
| 4 | `impl-lighting`, `impl-shadows`, `impl-animation` | 3 | Batch 2-3 | Core rendering features |
| 5 | `impl-post-processing`, `impl-physics`, `impl-audio` | 3 | Batch 1-3 | Advanced features |
| 6 | `impl-react-three-fiber`, `impl-drei`, `impl-xr` | 3 | Batch 1-3 | Ecosystem integrations |
| 7 | `impl-webgpu`, `impl-ifc-viewer`, `errors-performance` | 3 | Batch 1-3 | Specialized + errors |
| 8 | `errors-rendering`, `agents-scene-builder`, `agents-model-optimizer` | 3 | ALL above | Errors + agents last |

**Total**: 24 skills across 8 batches.

---

## Per-Skill Agent Prompts

### Constants

```
PROJECT_ROOT = C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package
RESEARCH_DIR = C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\docs\research
REQUIREMENTS_FILE = C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\REQUIREMENTS.md
REFERENCE_SKILL = C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md
```

---

### Batch 1

#### Prompt: threejs-core-scene-graph

```
## Task: Create the threejs-core-scene-graph skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-core\threejs-core-scene-graph\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete Object3D, Scene, Group, Mesh API)
3. references/examples.md (working code examples)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: threejs-core-scene-graph
description: >
  Use when creating Three.js scenes, adding objects, managing parent-child
  hierarchies, or organizing 3D content. Prevents the common mistake of
  using add() instead of attach() for reparenting, forgetting
  matrixAutoUpdate, or missing dispose() calls. Covers Object3D, Scene,
  Group, Mesh, Layers, Fog, traversal, coordinate conversion.
  Keywords: scene graph, Object3D, add, remove, traverse, layers, fog,
  parent, children, visibility, Three.js scene setup.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Object3D: ALL transform properties (position, rotation, quaternion, scale, matrix), hierarchy props (parent, children), rendering props (visible, layers, castShadow, receiveShadow, renderOrder, frustumCulled), identity props (name, id, uuid, userData)
- Hierarchy methods: add(), remove(), attach(), clear(), removeFromParent() — with edge cases
- Traversal: traverse(), traverseVisible(), traverseAncestors(), getObjectByName/Id/Property
- Matrix system: matrixAutoUpdate, updateMatrix(), updateMatrixWorld(), updateWorldMatrix()
- Coordinate conversion: worldToLocal(), localToWorld() — mutation warning
- Rotation: lookAt() (camera vs non-camera behavior), rotateOnAxis/WorldAxis
- Scene class: background, environment, fog, overrideMaterial, environmentIntensity/Rotation
- Fog: Fog vs FogExp2, material.fog property
- Group: semantic container, no extra functionality
- Mesh: geometry + material, morphTargetInfluences, multi-material with groups
- Layers: bitmask system, enable/disable/toggle/test, selective rendering

### Research Sections to Read
From vooronderzoek-threejs-core.md:
- Section 1: Scene Graph Deep Dive (complete section)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be verified against vooronderzoek research
- Include version annotations for r160+ specific features
- Include Critical Warnings section with NEVER rules
- Import style: import { Scene, Group, Mesh } from 'three'
```

#### Prompt: threejs-core-renderer

```
## Task: Create the threejs-core-renderer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-core\threejs-core-renderer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete WebGLRenderer API)
3. references/examples.md (working code examples)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: threejs-core-renderer
description: >
  Use when initializing a Three.js renderer, setting up the render loop,
  handling window resize, configuring tone mapping, or managing color
  spaces. Prevents the common mistake of not capping pixel ratio, using
  wrong color space, or forgetting to enable shadow maps. Covers
  WebGLRenderer, render targets, tone mapping, color management.
  Keywords: WebGLRenderer, render loop, setSize, setPixelRatio,
  toneMapping, outputColorSpace, shadowMap, WebGLRenderTarget, dispose,
  setAnimationLoop, compileAsync.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- WebGLRenderer constructor: ALL 12 parameters with defaults and usage guidance
- Key properties: domElement, shadowMap (enabled, type), toneMapping (7 types), toneMappingExposure, outputColorSpace, autoClear, sortObjects, clippingPlanes, localClippingEnabled, info
- Color management: outputColorSpace = SRGBColorSpace, texture.colorSpace rules, physically correct lights
- Tone mapping decision tree: when to use ACESFilmic vs AgX vs Neutral vs None
- Core methods: render(), setSize(), setPixelRatio(), setAnimationLoop(), setClearColor(), clear(), dispose()
- Render targets: WebGLRenderTarget, WebGLCubeRenderTarget, setRenderTarget(), readRenderTargetPixels()
- Shader compilation: compile(), compileAsync()
- Viewport/scissor: setViewport(), setScissor(), setScissorTest()
- Clipping planes: global and per-material (localClippingEnabled)
- Standard initialization pattern with best practices
- Resize handling pattern (window resize + pixel ratio cap at 2x)
- WebGPURenderer awareness: mention existence, point to impl-webgpu skill

### Research Sections to Read
From vooronderzoek-threejs-core.md:
- Section 2: Renderer Deep Dive (complete section)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be verified against vooronderzoek research
- Include version annotations for r160+ specific features
- Include Critical Warnings section with NEVER rules
- Import style: import { WebGLRenderer, SRGBColorSpace, ACESFilmicToneMapping } from 'three'
```

#### Prompt: threejs-core-math

```
## Task: Create the threejs-core-math skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-core\threejs-core-math\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete Vector3, Matrix4, Quaternion, Euler, Color, MathUtils API)
3. references/examples.md (working code examples)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: threejs-core-math
description: >
  Use when working with 3D math in Three.js: vectors, matrices,
  quaternions, rotations, colors, or coordinate transforms. Prevents the
  common mistake of mutating shared vectors, using wrong rotation order,
  or confusing Euler gimbal lock. Covers Vector3, Matrix4, Quaternion,
  Euler, Color, MathUtils, Box3, Sphere, coordinate system.
  Keywords: Vector3, Matrix4, Quaternion, Euler, Color, MathUtils,
  lerp, slerp, cross, dot, normalize, degToRad, Y-up, right-handed.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Coordinate system: right-handed, Y-up, implications for Blender/other tool imports
- Vector3: arithmetic (add, sub, multiply, divide), geometric (cross, dot, normalize, length, distanceTo), interpolation (lerp, lerpVectors), comparison (equals, manhattanLength), conversion (toArray, fromArray), mutation warning (most methods mutate in-place, use clone())
- Vector2, Vector4: key differences from Vector3
- Matrix4: column-major storage, compose/decompose, multiplication order (right-to-left), factory methods (makeRotationX, makeTranslation, makePerspective), lookAt, invert, transpose
- Quaternion: slerp, multiply (order matters!), setFromAxisAngle, setFromEuler, setFromRotationMatrix, identity, conjugate, normalize
- Euler: rotation order (XYZ default), gimbal lock at ±90° pitch, reorder(), setFromQuaternion, setFromRotationMatrix
- Color: constructor overloads (hex, string, r/g/b), setHSL, lerpHSL, convertSRGBToLinear/convertLinearToSRGB, color space management
- MathUtils: degToRad, radToDeg, clamp, lerp, inverseLerp, mapLinear, smoothstep, randFloat, randInt, generateUUID, isPowerOfTwo
- Bounding volumes: Box3 (setFromObject, containsPoint, intersectsBox), Sphere, Plane, Ray, Frustum
- Decision tree: when to use Euler vs Quaternion vs Matrix for rotation

### Research Sections to Read
From vooronderzoek-threejs-core.md:
- Section 5: Math Utilities Deep Dive (complete section)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language, not "you should" or "consider"
- All code examples must be verified against vooronderzoek research
- Include Critical Warnings section with NEVER rules (mutation, rotation order, gimbal lock)
- Import style: import { Vector3, Matrix4, Quaternion, Euler, Color, MathUtils } from 'three'
```

---

### Batch 2

#### Prompt: threejs-core-raycaster

```
## Task: Create the threejs-core-raycaster skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-core\threejs-core-raycaster\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (complete Raycaster API)
3. references/examples.md (mouse picking, hover detection, InstancedMesh picking)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: threejs-core-raycaster
description: >
  Use when implementing mouse picking, hover detection, object selection,
  or line-of-sight checks in Three.js. Prevents the common mistake of
  raycasting every mousemove frame, not using layers for filtering, or
  missing recursive flag. Covers Raycaster, setFromCamera, intersection
  format, InstancedMesh picking, layers, performance.
  Keywords: Raycaster, mouse picking, intersectObjects, click, hover,
  setFromCamera, intersection, instanceId, object selection.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Raycaster constructor: origin, direction, near, far
- Properties: ray, near, far, camera, layers, params (Mesh, Line, LOD, Points, Sprite thresholds)
- setFromCamera(coords, camera): normalized device coordinates (-1 to +1)
- intersectObject(object, recursive?, intersects?): return format
- intersectObjects(objects, recursive?, intersects?): return format
- Intersection object: point, distance, face, faceIndex, object, uv, uv1, instanceId, normal
- InstancedMesh picking: instanceId in intersection result
- Layers filtering: raycaster.layers for selective picking
- Mouse picking pattern: complete code with NDC conversion
- Hover detection pattern: onPointerMove with previous/current intersection
- Performance: throttle mousemove raycasting, use layers, avoid recursive on large scenes, bounding sphere checks
- Integration with controls (when to raycast vs when controls handle interaction)

### Research Sections to Read
From vooronderzoek-threejs-core.md:
- Section 4: Raycaster (complete section)

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples must be verified against vooronderzoek research
- Include Critical Warnings section with NEVER rules
- Import style: import { Raycaster, Vector2 } from 'three'
```

#### Prompt: threejs-syntax-geometries

```
## Task: Create the threejs-syntax-geometries skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-syntax\threejs-syntax-geometries\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (BufferGeometry, BufferAttribute, InstancedMesh API)
3. references/examples.md (custom geometry, instanced mesh, extrude)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: threejs-syntax-geometries
description: >
  Use when creating 3D shapes, custom geometry, or instanced meshes in
  Three.js. Prevents the common mistake of forgetting dispose(), not
  setting needsUpdate on BufferAttribute, or using non-indexed geometry
  when indexed is better. Covers BufferGeometry, BufferAttribute, all
  21 built-in geometries, InstancedMesh, ExtrudeGeometry, custom geometry.
  Keywords: BufferGeometry, BoxGeometry, SphereGeometry, PlaneGeometry,
  InstancedMesh, BufferAttribute, geometry, shape, extrude, custom mesh.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- BufferGeometry: all properties, setAttribute/getAttribute/deleteAttribute, setIndex, groups (addGroup for multi-material), drawRange, compute methods, dispose
- BufferAttribute: constructor, typed variants (Float32, Uint16, etc.), itemSize, needsUpdate, usage hints, accessor methods (getX/setX), InterleavedBufferAttribute
- Indexed vs non-indexed: trade-offs, when to use each, toNonIndexed()
- ALL 21 built-in geometry classes with constructor signatures
- ExtrudeGeometry options in detail (depth, bevel, extrudePath)
- Shape class for 2D paths (moveTo, lineTo, bezierCurveTo, holes)
- Custom geometry: step-by-step with position, normal, uv attributes
- Morph targets: morphAttributes, morphTargetInfluences
- InstancedMesh: constructor, setMatrixAt/getMatrixAt, setColorAt, instanceMatrix.needsUpdate, InstancedBufferAttribute for custom per-instance data
- Disposal: geometry.dispose(), when to reuse vs dispose

### Research Sections to Read
From vooronderzoek-threejs-geometry-materials.md:
- Section 1: BufferGeometry Deep Dive
- Section 2: Built-in Geometries Catalog
- Section 3: InstancedMesh

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples must be verified against vooronderzoek research
- Include Critical Warnings section
- Import style: import { BufferGeometry, Float32BufferAttribute, BoxGeometry, InstancedMesh } from 'three'
```

#### Prompt: threejs-syntax-materials

```
## Task: Create the threejs-syntax-materials skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-syntax\threejs-syntax-materials\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (all material types, texture properties)
3. references/examples.md (PBR setup, texture loading, multi-material)
4. references/anti-patterns.md (what NOT to do)

### Reference Format
Read and follow the structure of:
C:\Users\Freek Heijting\Documents\GitHub\Tauri-2-Claude-Skill-Package\skills\source\tauri-core\tauri-core-architecture\SKILL.md

### YAML Frontmatter
---
name: threejs-syntax-materials
description: >
  Use when choosing or configuring materials, loading textures, or setting
  up PBR workflows in Three.js. Prevents the common mistake of wrong
  color space on textures, forgetting material.dispose(), or not calling
  material.needsUpdate after changes. Covers all 15+ material types, PBR
  metalness/roughness, MeshPhysicalMaterial, texture maps, color space.
  Keywords: MeshStandardMaterial, MeshPhysicalMaterial, material, texture,
  PBR, metalness, roughness, normalMap, color space, SRGBColorSpace.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Base Material properties: side, transparent, opacity, depthWrite, depthTest, blending, alphaTest, visible, wireframe, fog, clippingPlanes
- ALL material types with purpose and key properties: Basic, Lambert, Phong, Standard, Physical, Toon, Matcap, Normal, Depth, Distance, Shadow, LineBasic, LineDashed, Points, Sprite
- MeshStandardMaterial: metalness, roughness, envMap, envMapIntensity, all map types
- MeshPhysicalMaterial: clearcoat, transmission, thickness, sheen, iridescence, anisotropy, ior, specularIntensity, attenuationColor/Distance
- Material selection decision tree: when to use which material
- Texture system: all 25+ map types with which color space to use
- Color space rules: diffuse/emissive = SRGBColorSpace, data maps = NoColorSpace
- TextureLoader, wrapping, filtering, anisotropy, flipY
- Material disposal: material.dispose(), texture.dispose()
- needsUpdate flag: when and why to set it
- NOT in scope: ShaderMaterial (covered in syntax-shaders skill)

### Research Sections to Read
From vooronderzoek-threejs-geometry-materials.md:
- Section 4: Materials System Deep Dive
- Section 5: Texture System

### Quality Rules
- English only
- SKILL.md < 500 lines; heavy content goes in references/
- Use ALWAYS/NEVER deterministic language
- All code examples must be verified against vooronderzoek research
- Include Critical Warnings section
- Import style: import { MeshStandardMaterial, TextureLoader, SRGBColorSpace } from 'three'
```

---

### Batch 3

#### Prompt: threejs-syntax-shaders

```
## Task: Create the threejs-syntax-shaders skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-syntax\threejs-syntax-shaders\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (ShaderMaterial, RawShaderMaterial, uniforms, ShaderChunk)
3. references/examples.md (custom vertex/fragment shader, onBeforeCompile)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-syntax-shaders
description: >
  Use when writing custom GLSL shaders, creating ShaderMaterial or
  RawShaderMaterial, defining uniforms, or patching built-in materials
  with onBeforeCompile. Prevents the common mistake of wrong uniform
  format, missing needsUpdate on uniforms, or not including required
  defines. Covers ShaderMaterial, RawShaderMaterial, GLSL, uniforms,
  ShaderChunk, onBeforeCompile, defines, GLSL3.
  Keywords: ShaderMaterial, RawShaderMaterial, GLSL, vertex shader,
  fragment shader, uniform, varying, ShaderChunk, onBeforeCompile, custom shader.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- ShaderMaterial: constructor, vertexShader, fragmentShader, uniforms, defines, extensions, glslVersion, lights, fog, wireframe
- RawShaderMaterial: difference from ShaderMaterial (no built-in uniforms/attributes injected)
- Uniforms: { value: ... } format, all types (float, int, vec2, vec3, vec4, mat3, mat4, sampler2D, samplerCube)
- Built-in uniforms: modelMatrix, viewMatrix, projectionMatrix, modelViewMatrix, normalMatrix, cameraPosition
- Built-in attributes: position, normal, uv, uv2, color
- Built-in varyings: none by default, must declare
- ShaderChunk: accessing Three.js shader library, inserting chunks
- onBeforeCompile: patching built-in materials, shader replacement tokens, customProgramCacheKey
- defines: preprocessor defines for conditional compilation
- GLSL3 mode: glslVersion = GLSL3, syntax differences (in/out vs attribute/varying, texture vs texture2D)
- Uniform update pattern: uniform.value = x (no needsUpdate needed on uniforms, but needsUpdate on material when changing shader code)

### Research Sections to Read
From vooronderzoek-threejs-geometry-materials.md:
- Section 6: Custom Shaders

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include complete minimal vertex + fragment shader example
- Include Critical Warnings section
```

#### Prompt: threejs-syntax-loaders

```
## Task: Create the threejs-syntax-loaders skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-syntax\threejs-syntax-loaders\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (GLTFLoader, DRACOLoader, all loader APIs)
3. references/examples.md (GLTF loading, texture loading, progress tracking)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-syntax-loaders
description: >
  Use when loading 3D models (GLTF, FBX, OBJ), textures, or HDR
  environment maps in Three.js. Prevents the common mistake of not
  setting up DRACOLoader, wrong WASM path, or missing error handling.
  Covers GLTFLoader, DRACOLoader, KTX2Loader, TextureLoader, RGBELoader,
  FBXLoader, OBJLoader, LoadingManager.
  Keywords: GLTFLoader, GLTF, GLB, load model, DRACOLoader, texture,
  FBXLoader, OBJLoader, RGBELoader, HDR, LoadingManager, progress.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Loader base class: load(url, onLoad, onProgress, onError), loadAsync(url, onProgress)
- LoadingManager: onStart, onLoad, onProgress, onError — global tracking
- GLTFLoader: setup, load, result object (scene, scenes, cameras, animations, asset), dispose
- DRACOLoader: setDecoderPath, setDecoderConfig, preload, dispose — WASM decoder setup
- KTX2Loader: setTranscoderPath, detectSupport(renderer) — GPU compressed textures
- MeshoptDecoder: for meshopt-compressed GLTF
- TextureLoader: load with color space assignment
- CubeTextureLoader: skybox loading
- RGBELoader: HDR environment maps
- FBXLoader: legacy format loading
- OBJLoader + MTLLoader: OBJ format
- GLTF as preferred format: why GLTF/GLB over FBX/OBJ
- Error handling patterns
- Import paths: 'three/addons/loaders/...'

### Research Sections to Read
From vooronderzoek-threejs-geometry-materials.md: loader sections
From b1-research-syntax.md: Section 3: Loaders

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include Critical Warnings section
```

#### Prompt: threejs-syntax-controls

```
## Task: Create the threejs-syntax-controls skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-syntax\threejs-syntax-controls\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (OrbitControls full API, other controls)
3. references/examples.md (setup patterns for each control type)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-syntax-controls
description: >
  Use when adding camera controls to a Three.js scene: orbit, pan, zoom,
  fly, first-person, or object manipulation. Prevents the common mistake
  of forgetting controls.update() in the render loop, not calling
  dispose(), or mixing controls. Covers OrbitControls, MapControls,
  FlyControls, PointerLockControls, TransformControls.
  Keywords: OrbitControls, controls, camera controls, orbit, pan, zoom,
  MapControls, FlyControls, PointerLockControls, TransformControls.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- OrbitControls: ALL 27 properties, methods (update, dispose, saveState, reset, listenToKeyEvents), events (change, start, end)
- MapControls: differences from OrbitControls (screenSpacePanning, mouse mapping)
- FlyControls: movementSpeed, rollSpeed, dragToLook
- PointerLockControls: lock/unlock, isLocked, first-person camera pattern
- TransformControls: mode (translate/rotate/scale), space (world/local), showX/Y/Z, dragging-changed event, integration with OrbitControls
- ArcballControls, TrackballControls, DragControls: brief overview
- Control selection decision tree: which control for which use case
- Lifecycle: construct → attach to DOM → update in loop → dispose
- enableDamping: MUST call controls.update() every frame when enabled
- Import paths: 'three/addons/controls/...'

### Research Sections to Read
From vooronderzoek-threejs-ecosystem.md:
- Section 6: Controls Deep Dive

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include Critical Warnings section
```

---

### Batch 4

#### Prompt: threejs-impl-lighting

```
## Task: Create the threejs-impl-lighting skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-lighting\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (all light types, PMREMGenerator, helpers)
3. references/examples.md (lighting setups: outdoor, indoor, studio, product)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-impl-lighting
description: >
  Use when setting up lighting in a Three.js scene: ambient, directional,
  point, spot, area, or environment lighting. Prevents the common mistake
  of too many shadow-casting lights, wrong intensity units, or missing
  environment maps for PBR. Covers all 7 light types, PMREMGenerator,
  HDR environment maps, physically correct intensity, helpers.
  Keywords: lighting, AmbientLight, DirectionalLight, PointLight,
  SpotLight, RectAreaLight, HemisphereLight, environment map, HDR, IBL,
  PMREMGenerator, scene.environment.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- All 7 light types: constructor signatures, all properties, intensity units
- Physically correct intensity in r160+ (candela, lux, nits)
- PMREMGenerator: fromEquirectangular, fromScene, fromCubemap, dispose
- HDR environment map workflow: RGBELoader → PMREMGenerator → scene.environment
- scene.environment vs scene.background: auto-applied to PBR materials
- Light helpers: DirectionalLightHelper, SpotLightHelper, PointLightHelper, HemisphereLightHelper
- RectAreaLight: requires RectAreaLightUniformsLib, RectAreaLightHelper
- LightProbe: ambient from environment
- Performance budget: recommended max shadow-casting lights
- Lighting recipes: outdoor (sun + sky), indoor (point + ambient), studio (3-point), product (HDRI)
- NOT in scope: shadow configuration (covered in impl-shadows)

### Research Sections to Read
From vooronderzoek-threejs-features.md:
- Section 1: Lighting System

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include Critical Warnings section
```

#### Prompt: threejs-impl-shadows

```
## Task: Create the threejs-impl-shadows skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-shadows\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (LightShadow API, shadow map types)
3. references/examples.md (shadow setup for directional, spot, point)
4. references/anti-patterns.md (shadow artifacts and fixes)

### YAML Frontmatter
---
name: threejs-impl-shadows
description: >
  Use when enabling shadows in a Three.js scene or debugging shadow
  artifacts like acne, peter panning, or low resolution. Prevents the
  common mistake of wrong shadow camera frustum, too many PointLight
  shadows, or missing castShadow/receiveShadow flags. Covers shadow map
  types, per-light config, bias tuning, CameraHelper debugging.
  Keywords: shadows, shadowMap, shadow acne, peter panning, bias,
  normalBias, castShadow, receiveShadow, PCFSoftShadowMap, shadow camera.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Three-step shadow opt-in: renderer.shadowMap.enabled, light.castShadow, mesh.castShadow/receiveShadow
- Shadow map types: Basic, PCF, PCFSoft, VSM — quality/performance comparison
- renderer.shadowMap.type configuration
- Per-light shadow: mapSize (power-of-two), camera frustum, bias, normalBias, radius, blurSamples, intensity
- DirectionalLight shadows: orthographic frustum sizing for the scene
- SpotLight shadows: auto-configured perspective from angle
- PointLight shadows: 6 cubemap faces, extreme cost — NEVER more than 1-2
- Shadow debugging: CameraHelper to visualize shadow camera frustum
- Shadow artifacts: acne (fix: bias), peter panning (fix: smaller bias), swimming (fix: larger mapSize), light bleeding (VSM issue)
- Transparent/alpha-tested material shadows: customDepthMaterial, alphaTest
- ContactShadows alternative from Drei (mention, point to impl-drei)

### Research Sections to Read
From vooronderzoek-threejs-features.md:
- Section 2: Shadow System

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include shadow artifact diagnosis flowchart
```

#### Prompt: threejs-impl-animation

```
## Task: Create the threejs-impl-animation skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-animation\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (AnimationMixer, AnimationAction, KeyframeTrack API)
3. references/examples.md (GLTF animation, crossfade, additive blending)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-impl-animation
description: >
  Use when playing animations, crossfading between animation states,
  or loading skeletal animations from GLTF in Three.js. Prevents the
  common mistake of not calling mixer.update(delta) every frame,
  wrong crossfade setup, or missing Clock. Covers AnimationMixer,
  AnimationClip, AnimationAction, KeyframeTrack, crossfade, blending.
  Keywords: animation, AnimationMixer, AnimationClip, AnimationAction,
  crossfade, skeletal, GLTF animation, keyframe, blend, Clock.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- AnimationMixer: constructor, clipAction, update(delta), time, timeScale, events (finished, loop), stopAllAction, uncacheClip/Root
- AnimationAction: play, stop, reset, fadeIn, fadeOut, crossFadeFrom, crossFadeTo, halt, warp — ALL properties (loop, weight, timeScale, blendMode, clampWhenFinished, paused, repetitions)
- AnimationClip: constructor, findByName, duration, tracks
- KeyframeTrack types: Vector, Quaternion, Number, Boolean, Color, String — interpolation modes
- PropertyBinding path format
- Clock: getDelta, getElapsedTime
- GLTF animation workflow: gltf.animations → mixer.clipAction → play
- Crossfade pattern for character state machines
- Additive animation blending (AdditiveAnimationBlendMode)
- Morph target animation
- Loop types: LoopOnce, LoopRepeat, LoopPingPong

### Research Sections to Read
From vooronderzoek-threejs-features.md:
- Section 3: Animation System

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include Critical Warnings section
```

---

### Batch 5

#### Prompt: threejs-impl-post-processing

```
## Task: Create the threejs-impl-post-processing skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-post-processing\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (EffectComposer, all passes, pmndrs effects)
3. references/examples.md (bloom, SSAO, outline, custom pass)
4. references/anti-patterns.md (what NOT to do — NEVER mix libraries)

### YAML Frontmatter
---
name: threejs-impl-post-processing
description: >
  Use when adding post-processing effects like bloom, SSAO, depth of
  field, or anti-aliasing to a Three.js scene. Prevents the common
  mistake of mixing Three.js EffectComposer with pmndrs/postprocessing,
  wrong pass order, or missing RenderPass. Covers EffectComposer,
  27+ passes, pmndrs/postprocessing, custom ShaderPass.
  Keywords: post-processing, EffectComposer, bloom, UnrealBloomPass,
  SSAO, outline, SMAA, FXAA, postprocessing, effects, render pass.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Three.js EffectComposer: setup, addPass, render, setSize, pass ordering rules
- RenderPass: ALWAYS first in chain
- ALL 27+ pass types with import paths and key constructor params
- UnrealBloomPass: strength, radius, threshold tuning
- SSAOPass/GTAOPass: kernelRadius, minDistance, maxDistance
- OutlinePass: selectedObjects, edgeStrength
- SMAA/FXAA: antialiasing as last pass
- Custom ShaderPass: writing fragment shaders
- pmndrs/postprocessing: architecture (effect merging), EffectComposer, EffectPass, RenderPass, key effects (Bloom, SSAO, DOF, Vignette, ToneMapping)
- NEVER mix Three.js built-in and pmndrs — incompatible architectures
- Resize handling for both libraries
- WebGPU PostProcessing class (mention, different from both)

### Research Sections to Read
From vooronderzoek-threejs-features.md:
- Section 4: Post-Processing

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
```

#### Prompt: threejs-impl-physics

```
## Task: Create the threejs-impl-physics skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-physics\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (cannon-es API, Rapier API)
3. references/examples.md (falling boxes, vehicle, ragdoll)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-impl-physics
description: >
  Use when adding physics simulation to a Three.js scene: rigid bodies,
  collisions, constraints, or raycasting. Prevents the common mistake
  of wrong Vec3/Vector3 conversion, missing world.step(), or choosing
  the wrong engine. Covers cannon-es and Rapier with Three.js sync
  patterns, performance comparison.
  Keywords: physics, cannon-es, Rapier, rigid body, collider, collision,
  gravity, simulation, WASM, constraint, joint.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Engine selection decision tree: cannon-es (<200 bodies, pure JS) vs Rapier (1000+, WASM, deterministic)
- cannon-es: World, Body (mass, type, shape, position, velocity, damping), ALL 8 shape types, Material, ContactMaterial, ALL constraint types, events, broadphase types, sleep, step
- Rapier: async WASM init, RigidBodyDesc builder, ColliderDesc (all shapes), World.step, ray casting, shape queries, collision events, CCD, debug render
- Three.js sync pattern: copy position/quaternion from physics to mesh each frame
- cannon-es Vec3 → Three.js Vector3 conversion (.copy() works)
- Performance: sleeping, broadphase, substeps, body count limits
- react-three-rapier: mention existence, point to impl-drei/impl-react-three-fiber

### Research Sections to Read
From vooronderzoek-threejs-features.md:
- Section 5: Physics Integration

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
```

#### Prompt: threejs-impl-audio

```
## Task: Create the threejs-impl-audio skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-audio\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (AudioListener, Audio, PositionalAudio, AudioAnalyser API)
3. references/examples.md (background music, 3D spatial audio, visualization)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-impl-audio
description: >
  Use when adding sound to a Three.js scene: background music, 3D
  spatial audio, or audio visualization. Prevents the common mistake
  of ignoring browser autoplay policy, not attaching AudioListener to
  camera, or wrong distance model. Covers AudioListener, Audio,
  PositionalAudio, AudioAnalyser.
  Keywords: audio, sound, AudioListener, PositionalAudio, 3D audio,
  spatial audio, music, AudioAnalyser, Web Audio API.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- AudioListener: attach to camera, one per scene, context property
- Audio: non-positional (background music), setBuffer, play, pause, stop, loop, volume, playbackRate
- PositionalAudio: 3D spatial with distance rolloff, refDistance, maxDistance, rolloffFactor, distanceModel (linear, inverse, exponential), setDirectionalCone
- AudioLoader: load audio files
- AudioAnalyser: getFrequencyData, getAverageFrequency — for visualization
- Browser autoplay policy: MUST resume AudioContext after user interaction
- Setup pattern: listener → loader → audio → play on user gesture
- Distance model decision tree

### Research Sections to Read
From vooronderzoek-threejs-features.md:
- Section 6: Audio System

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
```

---

### Batch 6

#### Prompt: threejs-impl-react-three-fiber

```
## Task: Create the threejs-impl-react-three-fiber skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-react-three-fiber\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (Canvas, hooks, event system, extend)
3. references/examples.md (basic scene, interactive objects, model loading)
4. references/anti-patterns.md (what NOT to do — NEVER mix imperative + declarative)

### YAML Frontmatter
---
name: threejs-impl-react-three-fiber
description: >
  Use when building 3D scenes with React using React Three Fiber (R3F).
  Prevents the common mistake of mixing imperative Three.js code with
  R3F's declarative model, creating objects inside useFrame, or
  forgetting Suspense for loaders. Covers Canvas, useFrame, useThree,
  useLoader, JSX mapping, event system, performance patterns.
  Keywords: React Three Fiber, R3F, Canvas, useFrame, useThree,
  useLoader, @react-three/fiber, 3D React, declarative Three.js.
license: MIT
compatibility: "Designed for Claude Code. Requires @react-three/fiber 8.x+, React 18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Canvas: ALL props (camera, shadows, gl, dpr, frameloop, events, orthographic, flat, linear, resize)
- JSX → Three.js mapping: lowercase elements, args prop, attach prop, nested properties
- useFrame: state object, delta, priority system, NEVER create objects inside useFrame
- useThree: accessing gl, scene, camera, size, viewport, mouse, raycaster — selector pattern for performance
- useLoader: Suspense-based, caching, preloading
- useGraph: traversing loaded models
- Event system: 12 event types, event object properties (point, distance, face, object)
- extend(): registering custom classes as JSX elements
- Primitives: <primitive object={...} /> for imperative objects
- Performance: useMemo for geometries/materials, dispose={null}, frameloop="demand" + invalidate()
- Portals: createPortal for multi-scene rendering
- R3F auto-disposal on unmount
- NOT in scope: Drei components (covered in impl-drei)

### Research Sections to Read
From vooronderzoek-threejs-ecosystem.md:
- Section 1: React Three Fiber Core

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
```

#### Prompt: threejs-impl-drei

```
## Task: Create the threejs-impl-drei skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-drei\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (key Drei components by category with props)
3. references/examples.md (Environment, OrbitControls, useGLTF, Html, Text)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-impl-drei
description: >
  Use when using Drei helper components with React Three Fiber:
  controls, environment maps, text rendering, HTML overlays, or
  performance helpers. Prevents the common mistake of not wrapping
  useGLTF in Suspense, using wrong Environment preset name, or
  forgetting makeDefault on controls. Covers 150+ Drei components
  across controls, environment, text, materials, loaders, performance.
  Keywords: Drei, @react-three/drei, OrbitControls, Environment,
  useGLTF, useTexture, Html, Text, Stage, ContactShadows, Instances.
license: MIT
compatibility: "Designed for Claude Code. Requires @react-three/drei, @react-three/fiber 8.x+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Controls: OrbitControls (makeDefault), MapControls, PresentationControls, ScrollControls, TransformControls, DragControls
- Environment: Environment (presets list), Lightformer, AccumulativeShadows, ContactShadows, Sky, Stars
- Text: Text (SDF), Text3D (extruded), Html (DOM in 3D), Billboard, Hud
- Materials: MeshReflectorMaterial, MeshTransmissionMaterial, MeshRefractionMaterial, shaderMaterial helper
- Loaders: useGLTF (draco support), useTexture, useFBX, useKTX2, useAnimations — ALL with Suspense
- Staging: Center, Float, Stage, Bounds, BBAnchor
- Performance: Instances, Merged, Detailed (LOD), Preload, BakeShadows, AdaptiveDpr, PerformanceMonitor
- Shapes: Box, Sphere, Plane, RoundedBox, etc.
- Abstractions: Edges, Outlines, Trail, Splat (Gaussian splatting), Decal
- Gizmos: GizmoHelper, GizmoViewport, Grid, Helper
- Component selection guide: which Drei component for which task

### Research Sections to Read
From vooronderzoek-threejs-ecosystem.md:
- Section 2: Drei Deep Dive

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
```

#### Prompt: threejs-impl-xr

```
## Task: Create the threejs-impl-xr skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-xr\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (WebXRManager, controllers, hand tracking)
3. references/examples.md (VR scene, AR hit test, teleportation)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-impl-xr
description: >
  Use when building VR or AR experiences with Three.js using WebXR.
  Prevents the common mistake of using requestAnimationFrame instead
  of setAnimationLoop, not handling controller events, or wrong
  reference space. Covers WebXRManager, VRButton, ARButton, controllers,
  hand tracking, hit testing, teleportation.
  Keywords: VR, AR, XR, WebXR, VRButton, ARButton, immersive,
  controller, hand tracking, hit test, teleportation, headset.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- WebXRManager: enabled, isPresenting, getSession, setSessionInit, setReferenceSpaceType, setFoveation
- Session types: 'immersive-vr', 'immersive-ar', 'inline'
- VRButton.createButton(renderer), ARButton.createButton(renderer)
- XR render loop: MUST use renderer.setAnimationLoop, NEVER requestAnimationFrame
- Controllers: getController(index), getControllerGrip(index), getHand(index)
- Controller events: select, selectstart, selectend, squeeze, squeezestart, squeezeend, connected, disconnected
- XRControllerModelFactory, XRHandModelFactory
- Hand tracking: joint positions and poses
- AR hit testing: XRHitTestSource, requestHitTestSource
- Reference spaces: 'local', 'local-floor', 'bounded-floor', 'unbounded'
- Teleportation pattern
- VR performance: 72/90fps requirement, foveated rendering
- All XR addon import paths

### Research Sections to Read
From vooronderzoek-threejs-ecosystem.md:
- Section 4: WebXR/VR/AR

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
```

---

### Batch 7

#### Prompt: threejs-impl-webgpu

```
## Task: Create the threejs-impl-webgpu skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-webgpu\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (WebGPURenderer, NodeMaterial types, TSL functions)
3. references/examples.md (basic setup, TSL material, compute shader)
4. references/anti-patterns.md (what NOT to do)

### YAML Frontmatter
---
name: threejs-impl-webgpu
description: >
  Use when using the Three.js WebGPU renderer, node-based materials,
  TSL (Three Shading Language), or compute shaders. Prevents the common
  mistake of not checking browser support, missing async init, or using
  GLSL with WebGPU. Covers WebGPURenderer, node materials, TSL,
  compute shaders, WebGL fallback, migration guide.
  Keywords: WebGPU, WebGPURenderer, TSL, Three Shading Language,
  node material, compute shader, WGSL, MeshStandardNodeMaterial.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+. WebGPU: Chrome 113+, Safari 18+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- WebGPURenderer: constructor, async init, properties
- Browser detection: WebGPU.isAvailable(), fallback to WebGLRenderer
- ALL 10 NodeMaterial types and their classic equivalents
- TSL type system: float, vec2, vec3, vec4, mat3, mat4, color, texture, uniform, attribute
- TSL math/vector/texture/lighting nodes
- TSL position/normal/uv nodes
- TSL time/animation nodes
- Compute shaders: ComputeNode, StorageBufferNode, StorageTextureNode
- WebGPU PostProcessing class (separate from EffectComposer)
- Migration guide: WebGL → WebGPU step-by-step
- GLSL NOT supported under WebGPU — WGSL is native
- Mark everything as EXPERIMENTAL where appropriate

### Research Sections to Read
From vooronderzoek-threejs-ecosystem.md:
- Section 3: WebGPU & TSL

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
```

#### Prompt: threejs-impl-ifc-viewer

```
## Task: Create the threejs-impl-ifc-viewer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-impl\threejs-impl-ifc-viewer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (web-ifc API, @thatopen/components API)
3. references/examples.md (IFC loading, property extraction, element picking)
4. references/anti-patterns.md (deprecated libraries, license, memory)

### YAML Frontmatter
---
name: threejs-impl-ifc-viewer
description: >
  Use when loading and viewing IFC/BIM models in a Three.js scene.
  Prevents the common mistake of using deprecated web-ifc-three,
  ignoring AGPL-3.0 license implications, or loading large IFC files
  without streaming. Covers web-ifc (MIT), @thatopen/components
  (AGPL-3.0), IFC loading, spatial tree, property extraction.
  Keywords: IFC, BIM, web-ifc, IFC viewer, @thatopen/components,
  building model, architecture, IFC loading, spatial tree, BIM viewer.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- web-ifc (MIT): IfcAPI, OpenModel, GetGeometry, GetFlatMesh, GetLine, property reading, WASM setup
- web-ifc → Three.js geometry conversion pattern
- @thatopen/components v3.x (AGPL-3.0): Components, Worlds, IfcLoader, FragmentsManager, Highlighter
- @thatopen/components setup pattern
- IFC spatial tree navigation
- Property extraction and querying
- Element picking/highlighting
- **LICENSE WARNING: @thatopen/components is AGPL-3.0 — MUST document implications prominently**
- web-ifc-three (IFCLoader): DEPRECATED — NEVER use for new projects
- Memory management for large IFC files
- Streaming/chunking for large models

### Research Sections to Read
From vooronderzoek-threejs-ecosystem.md:
- Section 5: IFC/BIM Viewer

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- License warning MUST be in Critical Warnings section
```

#### Prompt: threejs-errors-performance

```
## Task: Create the threejs-errors-performance skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-errors\threejs-errors-performance\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (dispose API, renderer.info, Stats.js)
3. references/examples.md (dispose patterns, InstancedMesh optimization, LOD)
4. references/anti-patterns.md (memory leaks, draw call explosion, texture bloat)

### YAML Frontmatter
---
name: threejs-errors-performance
description: >
  Use when a Three.js scene has performance problems: low FPS, memory
  leaks, too many draw calls, or high GPU memory usage. Prevents the
  common mistake of forgetting dispose(), creating objects in the render
  loop, or not using InstancedMesh for repeated objects. Covers disposal
  patterns, draw call optimization, InstancedMesh, LOD, texture compression,
  renderer.info, Stats.js profiling.
  Keywords: performance, memory leak, dispose, draw calls, FPS, slow,
  optimization, InstancedMesh, LOD, renderer.info, Stats, profiling.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Memory management: dispose() on geometries, materials, textures, render targets, renderers
- Scene cleanup: traverse + conditional dispose pattern
- Common memory leak patterns (5+): creating in render loop, not disposing textures, orphaned materials, event listeners, render targets
- Draw call optimization: merge geometries, InstancedMesh, batch rendering
- InstancedMesh for repeated objects (>100 identical)
- LOD (Level of Detail): distance-based detail switching
- Texture optimization: power-of-two sizes, KTX2 compression, mipmaps, anisotropy
- renderer.info: programs, geometries, textures, render (calls, triangles, points, lines)
- Stats.js: FPS, MS, MB panels
- Chrome DevTools: Performance tab, WebGL Inspector
- Frustum culling: frustumCulled property, bounding sphere accuracy
- Object pooling pattern
- Performance budget: target metrics for different platforms

### Research Sections to Read
From vooronderzoek-threejs-core.md:
- Section 6: Disposal and Memory Management
From vooronderzoek-threejs-geometry-materials.md:
- Anti-patterns sections

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Include performance diagnosis flowchart
```

---

### Batch 8

#### Prompt: threejs-errors-rendering

```
## Task: Create the threejs-errors-rendering skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-errors\threejs-errors-rendering\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (relevant API for common fixes)
3. references/examples.md (fix patterns for each error)
4. references/anti-patterns.md (what causes each rendering error)

### YAML Frontmatter
---
name: threejs-errors-rendering
description: >
  Use when a Three.js scene renders incorrectly: black screen, invisible
  objects, wrong colors, z-fighting, or broken shadows. Prevents the
  common mistake of wrong color space, missing material.side, or
  forgetting updateProjectionMatrix. Covers all common rendering errors
  with diagnosis steps and fix patterns.
  Keywords: black screen, invisible, wrong color, z-fighting, shadow
  artifact, rendering error, debug, nothing visible, dark, broken.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Black screen diagnosis: camera position, near/far planes, material, light, renderer size
- Invisible objects: wrong side (FrontSide/BackSide/DoubleSide), opacity=0 without transparent=true, layers mismatch, frustumCulled with wrong bounding sphere, visible=false inherited
- Wrong colors: color space mismatch (double gamma), texture.colorSpace not set, toneMapping artifacts, vertex colors not enabled
- Z-fighting: objects at same depth, fix with polygon offset, logarithmic depth buffer, or slight position offset
- Shadow artifacts: acne, peter panning (point to impl-shadows for details)
- WebGL context lost: causes, recovery pattern (webglcontextlost/webglcontextrestored events)
- needsUpdate missing: material.needsUpdate, bufferAttribute.needsUpdate, texture.needsUpdate — when each is required
- updateProjectionMatrix: MUST call after changing camera fov, aspect, near, far, zoom
- Transparent rendering: renderOrder, depthWrite=false for transparent objects
- Common console warnings: "THREE.WebGLRenderer: Texture is not power of two", "OES_texture_float not supported"
- Debugging workflow: step-by-step checklist

### Research Sections to Read
From all vooronderzoek documents: anti-patterns sections

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Organize as diagnosis flowcharts: symptom → possible causes → fixes
```

#### Prompt: threejs-agents-scene-builder

```
## Task: Create the threejs-agents-scene-builder skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-agents\threejs-agents-scene-builder\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (decision trees for lighting, materials, cameras, post-processing)
3. references/examples.md (complete scene recipes: product viewer, architectural, game level)
4. references/anti-patterns.md (common scene composition mistakes)

### YAML Frontmatter
---
name: threejs-agents-scene-builder
description: >
  Use when composing a complete Three.js scene from scratch or when
  asked to create a 3D visualization. Provides decision trees for
  choosing lighting setup, materials, camera type, controls, and
  post-processing pipeline. Orchestrates other Three.js skills.
  Keywords: create scene, build 3D, Three.js project, scene setup,
  visualization, product viewer, 3D application, scene composition.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- Scene composition decision tree: use case → lighting + materials + camera + controls + post-processing
- Lighting recipe selection: outdoor, indoor, studio, product, stylized
- Material selection: unlit, PBR, toon, custom shader — when to use which
- Camera selection: perspective (most 3D) vs orthographic (2D/isometric/CAD)
- Controls selection: orbit (viewer), map (top-down), fly (exploration), pointer lock (FPS), transform (editor)
- Post-processing pipeline: which effects for which genre
- Environment setup: HDR for PBR, solid color for stylized, gradient for simple
- Complete scene templates: product viewer, architectural walkthrough, game level, data visualization
- Performance considerations: when to use InstancedMesh, LOD, compressed textures
- R3F vs imperative: when to use which approach
- Cross-references to all other skills for implementation details

### Research Sections to Read
All vooronderzoek documents — synthesis of all patterns

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Focus on DECISION TREES, not code — code is in other skills
```

#### Prompt: threejs-agents-model-optimizer

```
## Task: Create the threejs-agents-model-optimizer skill

### Output Directory
C:\Users\Freek Heijting\Documents\GitHub\Three.js-Claude-Skill-Package\skills\source\threejs-agents\threejs-agents-model-optimizer\

### Files to Create
1. SKILL.md (main skill file, <500 lines)
2. references/methods.md (gltf-transform, Draco, KTX2, mesh simplification)
3. references/examples.md (optimization pipeline, before/after metrics)
4. references/anti-patterns.md (over-compression, wrong format)

### YAML Frontmatter
---
name: threejs-agents-model-optimizer
description: >
  Use when optimizing 3D models for web delivery: reducing file size,
  compressing meshes, optimizing textures, or generating LOD. Provides
  a complete GLTF optimization pipeline using Draco, KTX2, mesh
  simplification, and gltf-transform.
  Keywords: optimize model, reduce file size, Draco compression, KTX2,
  gltf-transform, mesh simplification, LOD, GLTF optimize, bundle size.
license: MIT
compatibility: "Designed for Claude Code. Requires Three.js r160+."
metadata:
  author: OpenAEC-Foundation
  version: "1.0"
---

### Scope (EXACT — do not exceed)
- GLTF optimization pipeline: assessment → mesh optimization → texture optimization → compression → validation
- gltf-transform CLI and API: dedup, flatten, join, prune, quantize, draco, ktx2
- Draco mesh compression: settings, quality vs size tradeoff
- KTX2 texture compression: UASTC vs ETC1S, quality settings
- Mesh simplification: meshopt, weld, simplify
- Texture optimization: resize to power-of-two, format selection (PNG vs JPEG vs WebP vs KTX2)
- LOD generation: automatic detail levels
- Bundle size analysis: what contributes to download size
- Assessment checklist: polygon count, texture resolution, file size targets
- Target metrics: <5MB for web, <20MB for desktop, specific polygon budgets
- Tools: gltf-transform, gltfpack, Blender GLTF export settings

### Research Sections to Read
From vooronderzoek-threejs-geometry-materials.md: loader/geometry sections
From vooronderzoek-threejs-features.md: performance sections

### Quality Rules
- English only, SKILL.md < 500 lines
- ALWAYS/NEVER deterministic language
- Focus on PIPELINE and DECISION TREES
```

---

## Appendix: Skill Directory Structure

```
skills/source/
├── threejs-core/
│   ├── threejs-core-scene-graph/
│   │   ├── SKILL.md
│   │   └── references/ (methods.md, examples.md, anti-patterns.md)
│   ├── threejs-core-renderer/
│   ├── threejs-core-math/
│   └── threejs-core-raycaster/
├── threejs-syntax/
│   ├── threejs-syntax-geometries/
│   ├── threejs-syntax-materials/
│   ├── threejs-syntax-shaders/
│   ├── threejs-syntax-loaders/
│   └── threejs-syntax-controls/
├── threejs-impl/
│   ├── threejs-impl-lighting/
│   ├── threejs-impl-shadows/
│   ├── threejs-impl-animation/
│   ├── threejs-impl-post-processing/
│   ├── threejs-impl-physics/
│   ├── threejs-impl-react-three-fiber/
│   ├── threejs-impl-drei/
│   ├── threejs-impl-webgpu/
│   ├── threejs-impl-ifc-viewer/
│   ├── threejs-impl-audio/
│   └── threejs-impl-xr/
├── threejs-errors/
│   ├── threejs-errors-performance/
│   └── threejs-errors-rendering/
└── threejs-agents/
    ├── threejs-agents-scene-builder/
    └── threejs-agents-model-optimizer/
```
