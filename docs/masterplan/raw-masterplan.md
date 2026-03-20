# Three.js Skill Package — Raw Masterplan (B1)

## Status

Phase B1: Raw Masterplan — landscape mapped, preliminary skill inventory defined.
Date: 2026-03-20

---

## 1. Technology Landscape Summary

### 1.1 Three.js Core (r160+)

Three.js uses a tree-based scene graph with `Object3D` as the universal base class. The inheritance chain covers Scene, Group, Mesh, Camera, and Light families. Key subsystems:

- **Scene Graph**: Object3D hierarchy with parent-child relationships, traversal, layers, and coordinate conversion
- **Renderer**: WebGLRenderer (primary, WebGL 2.0) and WebGPURenderer (experimental, async init, TSL node materials)
- **Camera**: PerspectiveCamera and OrthographicCamera with frustum configuration
- **Math**: Vector2/3/4, Matrix3/4, Quaternion, Euler, Color, MathUtils — right-handed Y-up coordinate system
- **Color Management**: r160+ defaults to SRGBColorSpace, physically correct lights enabled by default

### 1.2 Geometry & Materials

- **BufferGeometry**: Sole geometry base class (legacy Geometry removed in r125). 20+ built-in geometry types.
- **BufferAttribute**: Typed arrays for GPU data. InstancedMesh for batch rendering (single draw call, thousands of objects).
- **Materials**: 15+ material types. PBR via MeshStandardMaterial (metalness/roughness) and MeshPhysicalMaterial (clearcoat, transmission, sheen, iridescence, anisotropy).
- **ShaderMaterial/RawShaderMaterial**: Custom GLSL shaders with uniforms.
- **Textures**: 12+ map types (diffuse, normal, roughness, metalness, AO, emissive, displacement, etc.)

### 1.3 Loaders & Controls

- **Loaders**: Built-in (TextureLoader, CubeTextureLoader, FileLoader) + 18 addon loaders. GLTF is the preferred 3D format. DRACOLoader for compression, KTX2Loader for GPU textures.
- **Controls**: 9 control types. OrbitControls (most common), MapControls, FlyControls, PointerLockControls, TransformControls.

### 1.4 Lighting & Shadows

- **Lights**: 7 types (Ambient, Hemisphere, Directional, Point, Spot, RectArea, IESSpot). Shadow support on Directional, Point, Spot.
- **Shadows**: PCFSoftShadowMap recommended. Shadow camera frustum tuning critical. PointLight shadows = 6 shadow maps (expensive).
- **IBL**: PMREMGenerator for environment maps. scene.environment auto-applies to all PBR materials.

### 1.5 Animation

- **AnimationMixer/AnimationClip/AnimationAction**: Layered architecture. 6 KeyframeTrack types.
- **Crossfade, blending, additive animation**: Full state machine support.
- **GLTF skeletal animation**: Primary workflow for character animation.

### 1.6 Post-Processing

- **Three.js Built-in**: EffectComposer with 16 pass types (Bloom, SSAO, Outline, SMAA, etc.)
- **pmndrs/postprocessing**: Higher-performance alternative with auto effect merging. 21+ effects. NEVER mix with built-in EffectComposer.

### 1.7 Physics

- **cannon-es**: Pure JS, MIT, good for <200 bodies. Mutable API.
- **Rapier**: Rust→WASM, Apache-2.0, excellent for 1000+ bodies. Descriptor pattern. Cross-platform deterministic.
- **react-three-rapier**: R3F integration with declarative components, collision events, joints, instanced physics.

### 1.8 React Three Fiber (R3F)

- **@react-three/fiber v8/v9**: React renderer for Three.js. JSX→Three.js mapping. Core hooks: useFrame, useThree, useLoader.
- **@react-three/drei**: 50+ helper components (controls, environment, text, materials, loaders, performance).
- **Event system**: Pointer events on 3D objects (onClick, onPointerOver, etc.)

### 1.9 WebGPU & TSL

- **WebGPURenderer**: Async init, Chrome/Edge 113+, Safari 18+. API mostly compatible with WebGLRenderer.
- **Node Materials**: 10 node material types replacing classic materials. Composable node graph.
- **TSL**: JavaScript-based shader authoring. Math, vector, texture, lighting, storage nodes.
- **Compute Shaders**: GPGPU via ComputeNode and StorageBufferNode.

### 1.10 IFC/BIM

- **web-ifc**: Low-level WASM parser (active).
- **web-ifc-three (IFCLoader)**: DEPRECATED.
- **@thatopen/components v3.x**: Modern successor, component-based, AGPL-3.0 license. Spatial tree, property extraction, dimensioning, section planes.

---

## 2. Preliminary Skill Inventory (19 skills)

### 2.1 Core Skills (3)

| # | Name | Scope | API Surface | Complexity |
|---|------|-------|-------------|------------|
| 1 | `threejs-core-scene-graph` | Object3D hierarchy, Scene, Group, Mesh. Parent-child operations, traversal, layers, coordinate conversion, visibility. | Object3D, Scene, Group, Mesh, Layers | M |
| 2 | `threejs-core-renderer` | WebGLRenderer initialization, render loop, resize, pixel ratio, tone mapping, color space, shadow map config. WebGPURenderer awareness. | WebGLRenderer, WebGPURenderer, WebGLRenderTarget | L |
| 3 | `threejs-core-math` | Vector3, Matrix4, Quaternion, Euler, Color, MathUtils. Coordinate system, common operations, transform pitfalls. | Vector2/3/4, Matrix3/4, Quaternion, Euler, Color, MathUtils | M |

### 2.2 Syntax Skills (4)

| # | Name | Scope | API Surface | Complexity |
|---|------|-------|-------------|------------|
| 4 | `threejs-syntax-geometries` | BufferGeometry, BufferAttribute, built-in geometries (20+ types), InstancedMesh, indexed vs non-indexed, custom geometry creation, disposal. | BufferGeometry, BufferAttribute, InstancedMesh, all built-in geometry classes | L |
| 5 | `threejs-syntax-materials` | Material hierarchy, PBR (Standard/Physical), textures and maps, ShaderMaterial, uniforms, color space rules, disposal. | MeshBasicMaterial, MeshStandardMaterial, MeshPhysicalMaterial, ShaderMaterial, TextureLoader | L |
| 6 | `threejs-syntax-loaders` | GLTF/GLB loading, DRACOLoader, KTX2Loader, TextureLoader, FBXLoader, OBJLoader, LoadingManager, progress tracking. | GLTFLoader, DRACOLoader, KTX2Loader, TextureLoader, FBXLoader, OBJLoader, LoadingManager | M |
| 7 | `threejs-syntax-controls` | OrbitControls, MapControls, FlyControls, PointerLockControls, TransformControls. Lifecycle, update loop, disposal. | OrbitControls, MapControls, FlyControls, PointerLockControls, TransformControls | S |

### 2.3 Implementation Skills (8)

| # | Name | Scope | API Surface | Complexity |
|---|------|-------|-------------|------------|
| 8 | `threejs-impl-lighting` | All 7 light types, IBL with PMREMGenerator, HDR environment maps, scene.environment, light helpers. | AmbientLight, DirectionalLight, PointLight, SpotLight, HemisphereLight, RectAreaLight, PMREMGenerator | M |
| 9 | `threejs-impl-shadows` | Shadow map types, per-light shadow config, shadow camera frustum, bias tuning, shadow acne/peter panning, CameraHelper for debugging. | LightShadow, DirectionalLightShadow, SpotLightShadow, PointLightShadow, CameraHelper | M |
| 10 | `threejs-impl-post-processing` | EffectComposer (built-in), RenderPass, Bloom, SSAO, Outline, SMAA. pmndrs/postprocessing library. Custom shader passes. | EffectComposer, RenderPass, UnrealBloomPass, SSAOPass, OutlinePass, SMAAPass, pmndrs EffectPass | L |
| 11 | `threejs-impl-animation` | AnimationMixer, AnimationClip, AnimationAction, KeyframeTrack types, crossfade, blending, GLTF skeletal animation, Clock. | AnimationMixer, AnimationClip, AnimationAction, KeyframeTrack, Clock | M |
| 12 | `threejs-impl-physics` | cannon-es and Rapier integration. Rigid bodies, colliders, constraints/joints, sync patterns, performance comparison. | cannon-es World/Body/Shape, Rapier World/RigidBody/Collider | L |
| 13 | `threejs-impl-react-three-fiber` | Canvas, useFrame, useThree, useLoader, JSX mapping, event system, Drei components, performance patterns, R3F + Rapier. | Canvas, useFrame, useThree, useLoader, Drei components, @react-three/rapier | L |
| 14 | `threejs-impl-webgpu` | WebGPURenderer setup, browser detection/fallback, node materials, TSL basics, compute shaders, migration from WebGL. | WebGPURenderer, MeshStandardNodeMaterial, TSL functions, ComputeNode | L |
| 15 | `threejs-impl-ifc-viewer` | @thatopen/components v3.x setup, IFC loading, spatial tree, property extraction, element picking. License warning (AGPL-3.0). | @thatopen/components, web-ifc | M |

### 2.4 Error Skills (2)

| # | Name | Scope | API Surface | Complexity |
|---|------|-------|-------------|------------|
| 16 | `threejs-errors-performance` | Memory leaks (dispose patterns), draw call optimization, InstancedMesh, LOD, frustum culling, texture compression, render budget. Stats.js, renderer.info. | dispose(), InstancedMesh, LOD, Stats, renderer.info | M |
| 17 | `threejs-errors-rendering` | Black screen, invisible objects, wrong colors (color space), z-fighting, shadow artifacts, missing updates (needsUpdate), WebGL context lost. | Common error scenarios and fix patterns | M |

### 2.5 Agent Skills (2)

| # | Name | Scope | API Surface | Complexity |
|---|------|-------|-------------|------------|
| 18 | `threejs-agents-scene-builder` | Decision tree for composing complete 3D scenes. Lighting recipe selection, material choice, camera setup, post-processing pipeline. | All core + syntax + impl skills | L |
| 19 | `threejs-agents-model-optimizer` | GLTF optimization pipeline: Draco compression, texture resize/compress, mesh simplification, LOD generation, bundle size analysis. | GLTFLoader, DRACOLoader, KTX2Loader, gltf-transform | M |

---

## 3. Dependency Graph

```
Layer 0 (no deps):    core-scene-graph, core-renderer, core-math
Layer 1 (→ core):     syntax-geometries, syntax-materials, syntax-loaders, syntax-controls
Layer 2 (→ syntax):   impl-lighting, impl-shadows, impl-animation, impl-post-processing
Layer 3 (→ syntax):   impl-physics, impl-react-three-fiber, impl-webgpu, impl-ifc-viewer
Layer 4 (→ impl):     errors-performance, errors-rendering
Layer 5 (→ all):      agents-scene-builder, agents-model-optimizer
```

### Specific Dependencies

| Skill | Depends On |
|-------|-----------|
| syntax-geometries | core-scene-graph, core-math |
| syntax-materials | core-scene-graph, core-renderer |
| syntax-loaders | core-scene-graph |
| syntax-controls | core-renderer |
| impl-lighting | syntax-materials |
| impl-shadows | impl-lighting, core-renderer |
| impl-animation | syntax-loaders (GLTF) |
| impl-post-processing | core-renderer |
| impl-physics | core-scene-graph, core-math |
| impl-react-three-fiber | core-scene-graph, syntax-materials, syntax-controls |
| impl-webgpu | core-renderer, syntax-materials |
| impl-ifc-viewer | syntax-loaders |
| errors-performance | core-renderer, syntax-geometries, syntax-materials |
| errors-rendering | core-renderer, syntax-materials |
| agents-scene-builder | ALL impl skills |
| agents-model-optimizer | syntax-loaders, errors-performance |

---

## 4. Preliminary Batch Plan

| Batch | Skills | Count | Layer |
|-------|--------|-------|-------|
| B5-1 | core-scene-graph, core-renderer, core-math | 3 | 0 |
| B5-2 | syntax-geometries, syntax-materials, syntax-loaders | 3 | 1 |
| B5-3 | syntax-controls, impl-lighting, impl-shadows | 3 | 1-2 |
| B5-4 | impl-animation, impl-post-processing, impl-physics | 3 | 2-3 |
| B5-5 | impl-react-three-fiber, impl-webgpu, impl-ifc-viewer | 3 | 3 |
| B5-6 | errors-performance, errors-rendering | 2 | 4 |
| B5-7 | agents-scene-builder, agents-model-optimizer | 2 | 5 |

**Total**: 19 skills across 7 batches.

---

## 5. Open Questions for B2 Research

1. **Shader depth**: Is custom GLSL/ShaderMaterial large enough for a separate skill, or does it fit within syntax-materials? (Q-003)
2. **IFC scope**: How deep should the IFC viewer skill go given the AGPL-3.0 license of @thatopen/components? (Q-002)
3. **R3F scope**: Should R3F + Drei be split into two separate skills (R3F core + Drei helpers)?
4. **MCP server**: Should the package include an MCP server for live preview or scene analysis? (Q-001)
5. **Raycasting**: Is raycasting covered within scene-graph, or does it warrant inclusion in controls or a separate skill?
6. **Audio**: Three.js has AudioListener/PositionalAudio — include or exclude?
7. **XR/VR**: Three.js has WebXR support — include or exclude?

---

## 6. Research Plan for B2 (Vooronderzoek)

Four deep research documents needed:

| Document | Sections | Min. Words |
|----------|----------|-----------|
| `vooronderzoek-threejs-core.md` | Scene graph details, renderer internals, camera system, math utilities, coordinate system, disposal patterns | 2000 |
| `vooronderzoek-threejs-geometry-materials.md` | BufferGeometry deep dive, all material types, PBR workflow, texture pipeline, ShaderMaterial, color management | 2000 |
| `vooronderzoek-threejs-features.md` | Lighting system, shadow mapping, animation system, post-processing (both libraries), physics (cannon-es + Rapier) | 2000 |
| `vooronderzoek-threejs-ecosystem.md` | React Three Fiber, Drei catalog, WebGPU/TSL, IFC/BIM, XR/VR (if included), audio (if included) | 2000 |

---

## 7. Next Steps

1. **B2**: Execute deep research (4 vooronderzoek documents, 8000+ words total)
2. **B3**: Refine masterplan based on research findings → definitive masterplan with per-skill agent prompts
3. **B4**: Topic-specific research per skill
4. **B5**: Skill creation in 7 batches
5. **B6**: Validation
6. **B7**: Publication
