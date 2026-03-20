# threejs-agents-scene-builder — Scene Recipes

## Recipe A: Product Viewer

### Scene Configuration

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Camera | PerspectiveCamera, fov: 40 | Minimal distortion for product |
| Controls | OrbitControls, damping, no pan, clamped polar | User orbits product only |
| Lighting | Three-point studio setup | Professional product photography look |
| Material | MeshStandardMaterial or MeshPhysicalMaterial | PBR realism |
| Environment | HDR studio preset | Clean reflections |
| Shadows | ContactShadows (Drei) or AccumulativeShadows | Soft ground shadow |
| Post-processing | Optional subtle bloom + FXAA | Clean output |
| Background | Solid color or gradient | Focus on product |
| Ground | Invisible ground plane for shadows | Shadow catcher |

### Imperative Setup Steps

```
1. WebGLRenderer({ antialias: true, alpha: true })
2. renderer.toneMapping = ACESFilmicToneMapping
3. renderer.toneMappingExposure = 1.0
4. PerspectiveCamera(40, aspect, 0.01, 100)
5. camera.position.set(0, 1, 3)
6. Load HDR environment → PMREMGenerator → scene.environment
7. SpotLight key (left, 300cd) + SpotLight fill (right, 100cd) + SpotLight rim (back, 200cd)
8. AmbientLight(0xffffff, 0.15)
9. Load GLTF model → center with Box3 → add to scene
10. OrbitControls: enableDamping, maxPolarAngle=PI/2, minDistance=1, maxDistance=10, autoRotate
11. Ground plane for shadow reception (or ContactShadows in R3F)
12. Resize handler for camera + renderer
```

### R3F Setup Structure

```
<Canvas camera={{ fov: 40, position: [0, 1, 3] }} shadows>
  <Environment preset="studio" />
  <Stage preset="rembrandt" intensity={0.5} shadows="contact">
    <Model />
  </Stage>
  <OrbitControls enableDamping autoRotate enablePan={false} maxPolarAngle={Math.PI/2} />
</Canvas>
```

---

## Recipe B: Architectural Walkthrough

### Scene Configuration

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Camera | PerspectiveCamera, fov: 65 | Natural FOV for interiors |
| Controls | PointerLockControls or OrbitControls | First-person or orbit navigation |
| Lighting | HDR environment + DirectionalLight (sun) + PointLights (fixtures) | Realistic indoor/outdoor |
| Material | MeshStandardMaterial with texture maps | Realistic surfaces |
| Environment | Architectural HDR (apartment or lobby preset) | Accurate reflections |
| Shadows | PCFSoftShadowMap, DirectionalLight shadow | Realistic sun shadows |
| Post-processing | GTAO + subtle bloom + SMAA | Depth and realism |
| Background | HDR environment as background | Visible through windows |

### Setup Steps

```
1. WebGLRenderer({ antialias: true })
2. renderer.shadowMap.enabled = true
3. renderer.shadowMap.type = PCFSoftShadowMap
4. renderer.toneMapping = ACESFilmicToneMapping
5. PerspectiveCamera(65, aspect, 0.1, 500)
6. camera.position.set(0, 1.6, 5) — eye height
7. HDR environment for IBL and background
8. DirectionalLight for sun: position high, shadow frustum sized to building footprint
9. PointLights for interior fixtures (no shadows or max 1-2 with shadows)
10. Load GLTF architectural model
11. Enable castShadow/receiveShadow per mesh
12. PointerLockControls for walkthrough OR OrbitControls with target at (0, 1.6, 0)
13. EffectComposer: RenderPass + GTAOPass + UnrealBloomPass(0.3) + SMAAPass + OutputPass
14. Collision detection for walkthrough mode (raycaster downward for ground)
```

---

## Recipe C: Game Level / Interactive 3D

### Scene Configuration

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Camera | PerspectiveCamera, fov: 55-70 | Game-appropriate FOV |
| Controls | Custom (keyboard + mouse) or PointerLockControls | Game input |
| Lighting | DirectionalLight (sun) + AmbientLight + PointLights (effects) | Game lighting |
| Material | Mix of Standard and Lambert | Balance quality vs performance |
| Environment | Procedural sky or skybox | Game world backdrop |
| Shadows | PCFSoftShadowMap, 1 DirectionalLight shadow | Performance budget |
| Post-processing | SSAO + bloom (for effects) + FXAA | Game visual quality |
| Physics | Rapier (via @dimforge/rapier3d-compat) | Collision and dynamics |

### Setup Steps

```
1. WebGLRenderer({ antialias: false }) — use FXAA post-process instead
2. renderer.shadowMap.enabled = true, type = PCFSoftShadowMap
3. PerspectiveCamera(60, aspect, 0.1, 1000)
4. DirectionalLight for sun with optimized shadow frustum
5. AmbientLight or HemisphereLight for fill
6. Load level geometry (GLTF)
7. Set matrixAutoUpdate = false on static objects
8. InstancedMesh for repeated objects (trees, rocks, props)
9. Physics world initialization (Rapier)
10. Player controller with collision
11. EffectComposer: RenderPass + SSAOPass + UnrealBloomPass + FXAAPass + OutputPass
12. Game loop via requestAnimationFrame with fixed timestep for physics
```

---

## Recipe D: Data Visualization

### Scene Configuration

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Camera | OrthographicCamera or PerspectiveCamera(50) | Depends on 2D vs 3D data |
| Controls | OrbitControls (3D) or none (2D) | Data exploration |
| Lighting | AmbientLight + DirectionalLight | Clear, even illumination |
| Material | MeshBasicMaterial or MeshLambertMaterial | Performance, color accuracy |
| Environment | None or solid background | Clean data presentation |
| Shadows | None | Unnecessary for data |
| Post-processing | FXAA only | Clean edges |
| Text | Drei `<Text>` (SDF) for labels | Crisp at any scale |

### Setup Steps

```
1. WebGLRenderer({ antialias: true })
2. renderer.toneMapping = NoToneMapping — CRITICAL for accurate data colors
3. OrthographicCamera (2D) or PerspectiveCamera(50) for 3D
4. scene.background = solid color
5. AmbientLight(0xffffff, 0.6) + DirectionalLight(0xffffff, 0.8)
6. Generate geometry from data (bars, lines, points, surfaces)
7. MeshBasicMaterial with data-mapped colors (no lighting interference)
8. Drei <Text> or HTML overlays for labels and axes
9. OrbitControls with constrained angles for 3D; none for 2D
10. Resize handler
```

**Critical rule:** ALWAYS use `NoToneMapping` for data visualization — tone mapping distorts colors and breaks data-color mappings.

---

## Recipe E: Portfolio / Hero Section

### Scene Configuration

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Camera | PerspectiveCamera, fov: 50 | Balanced perspective |
| Controls | None or scroll-driven (Drei ScrollControls) | Passive viewing or scroll interaction |
| Lighting | HDR environment + dramatic DirectionalLight | Visual impact |
| Material | MeshPhysicalMaterial (glass, metallic effects) | Premium look |
| Environment | HDR with blurred background | Atmospheric |
| Shadows | ContactShadows | Subtle grounding |
| Post-processing | Bloom + color grading + FXAA | Cinematic quality |
| Animation | useFrame rotation, Float, or GSAP | Continuous motion |

### R3F Setup Structure

```
<Canvas camera={{ fov: 50, position: [0, 0, 5] }}>
  <Environment preset="city" background blur={0.5} />
  <Float speed={2} rotationIntensity={0.5} floatIntensity={0.5}>
    <Model />
  </Float>
  <ContactShadows position={[0, -1, 0]} opacity={0.4} blur={2} />
  <EffectComposer>
    <Bloom luminanceThreshold={0.8} intensity={0.5} />
  </EffectComposer>
</Canvas>
```

---

## Recipe F: CAD / BIM / IFC Viewer

### Scene Configuration

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Camera | OrthographicCamera | No perspective distortion for measurements |
| Controls | OrbitControls with screenSpacePanning | Standard CAD navigation |
| Lighting | AmbientLight + 2 DirectionalLights (opposing) | Even illumination, no harsh shadows |
| Material | MeshLambertMaterial or MeshStandardMaterial | Performance for large models |
| Environment | None or neutral HDR | Clean technical view |
| Shadows | None or minimal | Performance priority |
| Post-processing | OutlinePass (selection) + FXAA | Selection highlighting |
| Special | Layers for visibility toggling | IFC categories on different layers |

### Setup Steps

```
1. WebGLRenderer({ antialias: true, logarithmicDepthBuffer: true }) — CRITICAL for CAD z-range
2. OrthographicCamera sized to model bounds
3. AmbientLight(0xffffff, 0.6)
4. DirectionalLight from top-right (0.8) + DirectionalLight from bottom-left (0.3)
5. Load IFC via web-ifc-three or GLTF export
6. Assign layers per IFC category (walls=1, floors=2, etc.)
7. OrbitControls with enableDamping, screenSpacePanning=true
8. Raycaster for element picking → OutlinePass for selection highlight
9. Fit camera to model bounds on load using Box3
10. Section planes via clipping planes (renderer.clippingPlanes)
```

**Critical rule:** ALWAYS enable `logarithmicDepthBuffer` for CAD/BIM scenes — standard depth buffer causes z-fighting at architectural scales.

---

## Recipe G: AR / XR Experience

### Scene Configuration

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Camera | Managed by XR session | XR provides camera automatically |
| Controls | XR controller input | Hand tracking or controllers |
| Lighting | LightProbe from camera feed + DirectionalLight | Match real-world lighting |
| Material | MeshStandardMaterial | PBR for realism |
| Environment | Real-world camera feed (AR) or virtual HDR (VR) | Depends on XR mode |
| Shadows | ContactShadows on detected planes | Ground virtual objects |
| Post-processing | Minimal or none | Performance critical for XR frame rate |

### Setup Steps

```
1. WebGLRenderer({ antialias: true, alpha: true }) — alpha for AR passthrough
2. renderer.xr.enabled = true
3. Add XRButton (ARButton or VRButton from addons)
4. scene.environment from XR light estimation (AR) or HDR (VR)
5. Lightweight geometry and materials (mobile GPU)
6. Hit-test for AR placement
7. XR controller events for interaction
8. ALWAYS maintain 72-90 FPS — NEVER add heavy post-processing in XR
```

**Critical rule:** NEVER drop below 72 FPS in XR — this causes motion sickness. Budget lighting, shadows, and post-processing accordingly.
