# threejs-agents-scene-builder — Anti-Patterns

## Lighting Anti-Patterns

### AP-L01: Using AmbientLight as the Only Light Source

**Wrong:** Adding only `AmbientLight` to a scene.
**Why:** AmbientLight illuminates all surfaces equally from all directions. The result is a flat, dimensionless scene with no shadows, no depth cues, and no visual interest.
**Fix:** ALWAYS combine AmbientLight with at least one directional, point, or spot light to create depth through light-dark contrast.

### AP-L02: Too Many Shadow-Casting Lights

**Wrong:** Enabling `castShadow = true` on every light in the scene.
**Why:** Each shadow-casting light renders the scene from its perspective into a shadow map. PointLight shadows render 6 times (cubemap). Three PointLights with shadows = 18 extra render passes per frame.
**Fix:** NEVER exceed 2-3 shadow-casting lights. Prefer DirectionalLight and SpotLight shadows over PointLight shadows. Use ContactShadows or AccumulativeShadows for soft ground shadows without real-time shadow maps.

### AP-L03: Forgetting to Add Light Target to Scene

**Wrong:** Changing `directionalLight.target.position` without adding the target to the scene.
**Why:** The target is an Object3D. If it is not in the scene graph, its world matrix is NEVER updated, so the light direction does not change.
**Fix:** ALWAYS call `scene.add(directionalLight.target)` when repositioning the target.

### AP-L04: Not Sizing DirectionalLight Shadow Frustum

**Wrong:** Using default shadow camera frustum bounds for DirectionalLight.
**Why:** The default orthographic frustum is typically too large (wasting shadow map resolution) or too small (clipping shadows). This produces low-quality or missing shadows.
**Fix:** ALWAYS manually set `shadow.camera.left/right/top/bottom/near/far` to tightly fit the area where shadows must appear. Use `CameraHelper` to visualize the frustum during development.

### AP-L05: Using RectAreaLight Without Initialization

**Wrong:** Creating a `RectAreaLight` without calling `RectAreaLightUniformsLib.init()`.
**Why:** RectAreaLight requires precomputed LTC (Linearly Transformed Cosine) lookup tables. Without initialization, the light produces incorrect or missing illumination.
**Fix:** ALWAYS call `RectAreaLightUniformsLib.init()` once before creating any RectAreaLight (WebGL). For WebGPU, use `RectAreaLightTexturesLib` instead.

---

## Material Anti-Patterns

### AP-M01: Using MeshPhysicalMaterial Everywhere

**Wrong:** Defaulting to `MeshPhysicalMaterial` for all objects.
**Why:** MeshPhysicalMaterial is the most expensive built-in material. It computes clearcoat, sheen, transmission, iridescence, and other effects even when not needed, wasting GPU cycles.
**Fix:** ALWAYS use `MeshStandardMaterial` as the default. Only upgrade to MeshPhysicalMaterial for objects that specifically need clearcoat, transmission, sheen, or other extended PBR features.

### AP-M02: Not Disposing Materials and Textures

**Wrong:** Removing objects from the scene without calling `.dispose()` on their geometry, material, and textures.
**Why:** GPU resources (buffers, textures, shader programs) are NEVER automatically freed when objects are removed from the scene graph. This causes memory leaks that degrade performance over time.
**Fix:** ALWAYS dispose geometry, material, and all textures when permanently removing objects. In R3F, auto-disposal handles this unless `dispose={null}` is set.

### AP-M03: Ignoring Environment Maps for PBR Materials

**Wrong:** Using MeshStandardMaterial or MeshPhysicalMaterial without setting `scene.environment`.
**Why:** PBR materials rely on environment maps for accurate reflections and ambient lighting. Without an environment map, metallic surfaces appear black and rough surfaces lack ambient contribution.
**Fix:** ALWAYS set `scene.environment` to a PMREM-processed HDR texture when using PBR materials. In R3F, use Drei's `<Environment>` component.

---

## Camera Anti-Patterns

### AP-C01: Setting Near Plane to Zero

**Wrong:** `new PerspectiveCamera(75, aspect, 0, 1000)`.
**Why:** A near plane of 0 causes division-by-zero in the projection matrix, making the depth buffer completely non-functional. Z-fighting occurs across the entire scene.
**Fix:** ALWAYS set `near` to the largest value that does not clip visible content. Typical minimum: `0.01` for small scenes, `0.1` for medium, `1` for large.

### AP-C02: Excessive Near/Far Ratio

**Wrong:** `new PerspectiveCamera(75, aspect, 0.001, 100000)`.
**Why:** Depth buffer precision is distributed logarithmically between near and far. A ratio of 1:100,000,000 means almost all precision is consumed near the camera, causing z-fighting on distant objects.
**Fix:** ALWAYS minimize the near/far ratio. Ideal: < 1:10,000. For extreme ranges (CAD/BIM), use `logarithmicDepthBuffer: true` on the renderer.

### AP-C03: Forgetting updateProjectionMatrix

**Wrong:** Changing `camera.fov` or `camera.aspect` without calling `camera.updateProjectionMatrix()`.
**Why:** The projection matrix is computed from fov/aspect/near/far and cached. Changing properties without updating the matrix has no visual effect.
**Fix:** ALWAYS call `camera.updateProjectionMatrix()` after modifying any camera frustum property.

---

## Controls Anti-Patterns

### AP-K01: Forgetting controls.update() with Damping

**Wrong:** Enabling `controls.enableDamping = true` without calling `controls.update()` in the render loop.
**Why:** Damping requires per-frame updates to interpolate camera movement. Without the update call, the camera freezes as soon as the user stops interacting.
**Fix:** ALWAYS call `controls.update()` in every animation frame when damping or autoRotate is enabled.

### AP-K02: Not Disposing Controls

**Wrong:** Creating new controls without disposing old ones.
**Why:** Controls attach event listeners to the DOM element. Without disposal, old listeners accumulate, causing memory leaks and unexpected behavior (duplicate event handling).
**Fix:** ALWAYS call `controls.dispose()` before creating replacement controls or when the component unmounts.

### AP-K03: Conflicting Controls

**Wrong:** Using OrbitControls and TransformControls simultaneously without coordination.
**Why:** Both controls respond to pointer events. Dragging a TransformControls gizmo also triggers OrbitControls rotation, producing chaotic camera movement.
**Fix:** ALWAYS disable OrbitControls when TransformControls interaction starts, re-enable on end:
```
transformControls.addEventListener('dragging-changed', (event) => {
    orbitControls.enabled = !event.value;
});
```

---

## Post-Processing Anti-Patterns

### AP-P01: Missing OutputPass

**Wrong:** Building an EffectComposer pipeline without `OutputPass` at the end.
**Why:** OutputPass handles tone mapping and color space conversion (linear → sRGB). Without it, the final image has incorrect gamma and no tone mapping, appearing washed out or oversaturated.
**Fix:** ALWAYS add `OutputPass` as the last pass in the EffectComposer pipeline.

### AP-P02: Forgetting to Resize the Composer

**Wrong:** Handling window resize by updating only the camera and renderer, not the EffectComposer.
**Why:** The composer's internal render targets maintain their own size. Without resizing, post-processing renders at the wrong resolution, producing stretched or pixelated output.
**Fix:** ALWAYS call `composer.setSize(width, height)` alongside `renderer.setSize()` in the resize handler.

### AP-P03: Adding Post-Processing to XR Scenes

**Wrong:** Using EffectComposer in WebXR mode.
**Why:** XR requires rendering to the XR session's framebuffer with strict timing constraints. EffectComposer redirects rendering to its own framebuffers, breaking the XR pipeline and causing dropped frames or black screens.
**Fix:** NEVER use standard EffectComposer in XR mode. Use XR-compatible post-processing or skip it entirely to maintain frame rate.

---

## Scene Composition Anti-Patterns

### AP-S01: Not Handling Window Resize

**Wrong:** Setting up the renderer and camera once without a resize handler.
**Why:** When the window resizes, the camera aspect ratio becomes incorrect (stretched image) and the renderer canvas size does not match the viewport (blurry or cropped output).
**Fix:** ALWAYS add a resize handler that updates: (1) `camera.aspect` + `updateProjectionMatrix()`, (2) `renderer.setSize()`, (3) `composer.setSize()` if using post-processing.

### AP-S02: Not Setting Pixel Ratio

**Wrong:** Using the default pixel ratio of 1 on high-DPI displays.
**Why:** The scene appears blurry on Retina/HiDPI screens because the canvas renders at 1x resolution while the display is 2x or 3x.
**Fix:** ALWAYS call `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))`. Cap at 2 to prevent excessive GPU load on 3x displays.

### AP-S03: Tone Mapping on Data Visualization

**Wrong:** Using ACESFilmicToneMapping or other artistic tone mapping for data visualization.
**Why:** Tone mapping remaps colors for cinematic appearance, distorting the precise color values needed for data-color mappings. A data point mapped to `#FF0000` will display as a different red after tone mapping.
**Fix:** ALWAYS use `renderer.toneMapping = THREE.NoToneMapping` for data visualization scenes where color accuracy matters.

### AP-S04: Mixing R3F and Imperative Patterns

**Wrong:** Using `useRef` to imperatively modify Three.js objects managed by R3F's reconciler alongside declarative props.
**Why:** R3F's reconciler sets properties on every render cycle. Imperative modifications made via refs are overwritten on the next React render, causing flickering or lost state.
**Fix:** ALWAYS use R3F's declarative prop system for persistent state. Use `useFrame` for per-frame imperative updates. Use refs ONLY for reading values or for properties not managed by JSX props.

### AP-S05: Loading Assets Without Suspense (R3F)

**Wrong:** Using `useLoader` or `useGLTF` without wrapping the component in `<Suspense>`.
**Why:** These hooks use React Suspense internally. Without a Suspense boundary, the component throws an unhandled promise and the entire tree crashes.
**Fix:** ALWAYS wrap components that use loader hooks in `<Suspense fallback={<LoadingIndicator />}>`.

### AP-S06: Creating Geometries and Materials in Render Functions

**Wrong:** Creating `new BoxGeometry()` or `new MeshStandardMaterial()` inside a React component body without memoization.
**Why:** React components re-render frequently. Each render creates new GPU resources without disposing the old ones, causing massive memory leaks and performance degradation.
**Fix:** In R3F, declare geometry and material as JSX children (auto-managed). In imperative code within R3F, ALWAYS use `useMemo` to create and cache Three.js objects.
