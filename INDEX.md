# Three.js Skill Package — Skill Index

> 24 deterministic skills for Three.js 3D web development with Claude.
> Each skill is self-contained, English-only, and follows ALWAYS/NEVER deterministic language.

---

## Skills by Category

### Core (4 skills) — Foundation knowledge

| # | Skill | Description |
|---|-------|-------------|
| 1 | threejs-core-scene-graph | Object3D hierarchy, Scene, Group, Mesh, Layers, Fog, traversal, coordinate conversion, matrix system. |
| 2 | threejs-core-renderer | WebGLRenderer setup, render loop, resize, pixel ratio, tone mapping (7 types), color management, shadow maps, render targets, clipping planes. |
| 3 | threejs-core-math | Vector3, Matrix4, Quaternion, Euler, Color, MathUtils, Box3, Sphere, Ray. Right-handed Y-up coordinate system. |
| 4 | threejs-core-raycaster | Raycaster for mouse picking, hover detection, object selection, InstancedMesh picking, layers filtering. |

---

### Syntax (5 skills) — How to define 3D objects

| # | Skill | Description |
|---|-------|-------------|
| 5 | threejs-syntax-geometries | BufferGeometry, BufferAttribute, 21 built-in geometries, InstancedMesh, custom geometry, morph targets. |
| 6 | threejs-syntax-materials | All 15+ material types, PBR (Standard/Physical), textures (25+ map types), color space rules, disposal. |
| 7 | threejs-syntax-shaders | ShaderMaterial, RawShaderMaterial, GLSL uniforms, ShaderChunk, onBeforeCompile, defines, GLSL3. |
| 8 | threejs-syntax-loaders | GLTFLoader, DRACOLoader, KTX2Loader, TextureLoader, RGBELoader, FBXLoader, OBJLoader, LoadingManager. |
| 9 | threejs-syntax-controls | OrbitControls (27 props), MapControls, FlyControls, PointerLockControls, TransformControls. |

---

### Implementation (10 skills) — Feature recipes

| # | Skill | Description |
|---|-------|-------------|
| 10 | threejs-impl-lighting | 7 light types, physically correct intensity, PMREMGenerator IBL, HDR environment maps, helpers. |
| 11 | threejs-impl-shadows | Shadow map types (PCF/VSM), per-light config, bias tuning, shadow artifacts diagnosis, CameraHelper. |
| 12 | threejs-impl-animation | AnimationMixer, AnimationClip, AnimationAction, crossfade, additive blending, GLTF skeletal animation. |
| 13 | threejs-impl-post-processing | EffectComposer (27+ passes), pmndrs/postprocessing (21+ effects), custom ShaderPass. NEVER mix libraries. |
| 14 | threejs-impl-physics | cannon-es and Rapier integration, rigid bodies, colliders, constraints, sync patterns, performance comparison. |
| 15 | threejs-impl-react-three-fiber | R3F Canvas, useFrame, useThree, useLoader, JSX mapping, event system, extend(), performance patterns. |
| 16 | threejs-impl-drei | 150+ Drei components: controls, environment, text, materials, loaders, staging, performance helpers. |
| 17 | threejs-impl-webgpu | WebGPURenderer (EXPERIMENTAL), node materials, TSL, compute shaders, WebGL fallback, migration guide. |
| 18 | threejs-impl-ifc-viewer | web-ifc (MIT) and @thatopen/components (AGPL-3.0), IFC loading, spatial tree, property extraction. |
| 19 | threejs-impl-audio | AudioListener, Audio, PositionalAudio, AudioAnalyser, distance models, browser autoplay policy. |
| 20 | threejs-impl-xr | WebXR VR/AR, VRButton/ARButton, controllers, hand tracking, hit testing, teleportation. |

---

### Errors (2 skills) — Diagnosis and anti-patterns

| # | Skill | Description |
|---|-------|-------------|
| 21 | threejs-errors-performance | Memory leaks, dispose patterns, draw call optimization, InstancedMesh, LOD, renderer.info, Stats.js. |
| 22 | threejs-errors-rendering | Black screen, invisible objects, wrong colors, z-fighting, shadow artifacts, WebGL context loss, needsUpdate. |

---

### Agents (2 skills) — Intelligent orchestration

| # | Skill | Description |
|---|-------|-------------|
| 23 | threejs-agents-scene-builder | Decision trees for scene composition: lighting, materials, camera, controls, post-processing, environment. |
| 24 | threejs-agents-model-optimizer | GLTF optimization pipeline: Draco, KTX2, mesh simplification, LOD generation, gltf-transform. |

---

## Skill Dependencies

```
Layer 0 (no deps):    core-scene-graph, core-renderer, core-math
Layer 1 (→ core):     core-raycaster, syntax-geometries, syntax-materials, syntax-loaders, syntax-controls
Layer 2 (→ syntax):   syntax-shaders, impl-lighting, impl-shadows, impl-animation, impl-post-processing
Layer 3 (→ syntax):   impl-physics, impl-react-three-fiber, impl-webgpu, impl-ifc-viewer, impl-audio, impl-xr
Layer 4 (→ impl):     impl-drei, errors-performance, errors-rendering
Layer 5 (→ all):      agents-scene-builder, agents-model-optimizer
```

---

## How to Use This Package

1. **Building a 3D scene from scratch:** Start with `threejs-agents-scene-builder` (#23) — it routes to the correct skills via decision trees.
2. **Setting up a specific feature:** Go directly to the relevant `impl/` skill (#10-20).
3. **Looking up API patterns:** Use the `syntax/` skills (#5-9) for complete API catalogs.
4. **Debugging rendering issues:** Use `threejs-errors-rendering` (#22) for visual problems, `threejs-errors-performance` (#21) for FPS/memory issues.
5. **Optimizing 3D models:** Use `threejs-agents-model-optimizer` (#24) for the optimization pipeline.

---

**Version:** 1.0 — 24 skills, validated and complete.
