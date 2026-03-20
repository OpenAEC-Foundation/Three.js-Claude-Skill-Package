# threejs-agents-scene-builder — Decision Trees Reference

## Lighting Recipe Decision Matrix

### Outdoor Scene Lighting

| Component | Configuration |
|-----------|--------------|
| Key light | `DirectionalLight`, intensity 1-3, position `(5, 10, 7.5)` |
| Fill light | `HemisphereLight`, sky `#87CEEB`, ground `#362907`, intensity 0.5-0.8 |
| Shadow type | `PCFSoftShadowMap` |
| Shadow map | 2048x2048, bias `-0.0001`, normalBias `0.02` |
| Shadow frustum | MUST size manually to scene bounds |
| Environment | HDR sky or Drei `<Sky>` with sun position matching DirectionalLight |

### Indoor Scene Lighting

| Component | Configuration |
|-----------|--------------|
| Key lights | 1-3 `PointLight` or `SpotLight` per room, intensity 100-300 candela |
| Fill light | `AmbientLight`, intensity 0.1-0.3 |
| Shadow type | `PCFSoftShadowMap` or `VSMShadowMap` |
| Shadow casters | Limit to 2-3 lights maximum |
| Environment | Low-intensity HDR for subtle reflections |
| Special | `RectAreaLight` for window light (REQUIRES `RectAreaLightUniformsLib.init()`) |

### Studio / Product Lighting (Three-Point Setup)

| Component | Configuration |
|-----------|--------------|
| Key light | `SpotLight`, 45 degrees left, intensity 300-500 candela, penumbra 0.3-0.5 |
| Fill light | `SpotLight`, 45 degrees right, intensity 100-200 candela, penumbra 0.5 |
| Rim/back light | `SpotLight`, behind subject, intensity 200-400 candela |
| Ambient | `AmbientLight`, intensity 0.1-0.2 |
| Shadows | Drei `<ContactShadows>` or `<AccumulativeShadows>` (R3F) |
| Environment | Drei `<Environment preset="studio">` or custom HDR |

### HDR-Only Lighting (Image-Based Lighting)

| Component | Configuration |
|-----------|--------------|
| Environment | HDR equirectangular map via `PMREMGenerator` |
| `scene.environment` | Prefiltered HDR texture |
| `scene.background` | Same HDR or blurred version (`backgroundBlurriness: 0.5`) |
| Supplemental | Optional `DirectionalLight` for sharper shadows |
| Shadows | `ContactShadows` (no real-time shadow maps needed) |

### Stylized / Toon Lighting

| Component | Configuration |
|-----------|--------------|
| Key light | `DirectionalLight`, strong intensity |
| Fill light | `HemisphereLight` with contrasting sky/ground colors |
| Shadows | `BasicShadowMap` for hard-edged cel shadows |
| Materials | `MeshToonMaterial` with step gradientMap |

---

## Material Selection Decision Matrix

### By Surface Type

| Surface | Material | Key Properties |
|---------|----------|---------------|
| Metal | MeshStandardMaterial | `metalness: 1.0`, `roughness: 0.1-0.5` |
| Plastic | MeshStandardMaterial | `metalness: 0.0`, `roughness: 0.3-0.7` |
| Wood | MeshStandardMaterial | `metalness: 0.0`, `roughness: 0.6-0.9`, diffuse + normal map |
| Glass | MeshPhysicalMaterial | `transmission: 1.0`, `roughness: 0.0`, `ior: 1.5`, `thickness: 0.5` |
| Fabric | MeshPhysicalMaterial | `sheen: 1.0`, `sheenRoughness: 0.5`, `sheenColor` |
| Car paint | MeshPhysicalMaterial | `clearcoat: 1.0`, `clearcoatRoughness: 0.1`, `metalness: 0.9` |
| Water | MeshPhysicalMaterial | `transmission: 0.9`, `roughness: 0.0` + animated normal map |
| Concrete | MeshStandardMaterial | `metalness: 0.0`, `roughness: 0.8-1.0`, normal map |
| Emissive (neon, screens) | MeshStandardMaterial | `emissive: color`, `emissiveIntensity: 2-10` |

### By Performance Tier

| Tier | Material | Cost | Use Case |
|------|----------|------|----------|
| Lowest | MeshBasicMaterial | No lighting calc | Background objects, wireframes, UI elements |
| Low | MeshLambertMaterial | Vertex lighting | Many objects, mobile, retro style |
| Medium | MeshPhongMaterial | Per-pixel, no PBR | Legacy scenes, specular highlights without PBR |
| Standard | MeshStandardMaterial | Full PBR | Default choice for realistic scenes |
| High | MeshPhysicalMaterial | Extended PBR | Glass, clearcoat, sheen, transmission |
| Custom | ShaderMaterial | Variable | Full GLSL control |

---

## Camera Configuration Reference

### PerspectiveCamera Settings by Use Case

| Use Case | FOV | Near | Far | Position Hint |
|----------|-----|------|-----|---------------|
| Product viewer | 35-45 | 0.01 | 100 | Distance = 2-3x object radius |
| Architectural interior | 60-75 | 0.1 | 500 | Eye height (~1.6 units) |
| Architectural exterior | 50-65 | 1 | 5000 | Elevated viewpoint |
| Game (third-person) | 50-60 | 0.1 | 1000 | Behind + above character |
| Game (first-person) | 70-90 | 0.01 | 500 | Eye height |
| Data visualization | 50-60 | 0.1 | 1000 | Above and angled |
| Cinematic | 20-35 | 0.5 | 2000 | Varies per shot |

### OrthographicCamera Settings

| Use Case | Frustum Sizing | Notes |
|----------|---------------|-------|
| 2D game | Match world units | `left/right/top/bottom` = visible area |
| CAD viewer | Fit to model bounds | Recalculate on model load |
| Isometric | Fixed ratio | `left = -aspect * d`, `right = aspect * d`, `top = d`, `bottom = -d` |
| UI overlay | Match screen pixels | `left = 0`, `right = width`, `top = height`, `bottom = 0` |

---

## Controls Configuration Reference

### OrbitControls Recommended Defaults

| Property | Product Viewer | Architectural | General |
|----------|---------------|---------------|---------|
| `enableDamping` | `true` | `true` | `true` |
| `dampingFactor` | `0.05` | `0.1` | `0.05` |
| `minDistance` | `1` | `0.5` | `0` |
| `maxDistance` | `20` | `100` | `Infinity` |
| `minPolarAngle` | `0.1` | `0` | `0` |
| `maxPolarAngle` | `Math.PI / 2` | `Math.PI` | `Math.PI` |
| `autoRotate` | `true` (slow) | `false` | `false` |
| `autoRotateSpeed` | `0.5` | N/A | N/A |
| `enablePan` | `false` | `true` | `true` |
| `target` | Object center | `(0, 1.6, 0)` | `(0, 0, 0)` |

---

## Post-Processing Pipeline Reference

### Pipeline by Visual Style

| Style | Passes (in order) |
|-------|-------------------|
| Clean / minimal | RenderPass + FXAA + OutputPass |
| Realistic | RenderPass + GTAO + UnrealBloomPass (subtle) + SMAA + OutputPass |
| Cinematic | RenderPass + GTAO + BokehPass + UnrealBloomPass + FilmPass + LUTPass + OutputPass |
| Stylized / neon | RenderPass + UnrealBloomPass (strong) + OutputPass |
| Technical / CAD | RenderPass + OutlinePass + FXAA + OutputPass |
| Game | RenderPass + SSAO + UnrealBloomPass + FXAA + OutputPass |

### UnrealBloomPass Presets

| Preset | Strength | Radius | Threshold |
|--------|----------|--------|-----------|
| Subtle glow | 0.3-0.5 | 0.4 | 0.9 |
| Standard bloom | 1.0-1.5 | 0.4 | 0.85 |
| Strong neon | 2.0-3.0 | 0.6 | 0.6 |
| Dreamy | 1.5-2.0 | 0.8 | 0.4 |

### SSAO/GTAO Presets

| Preset | Radius | Intensity | Notes |
|--------|--------|-----------|-------|
| Subtle | 0.5 | 0.5 | Minimal visible effect |
| Standard | 1.0 | 1.0 | Good general-purpose |
| Strong | 2.0 | 2.0 | Visible darkening in crevices |
| Contact-only | 0.2 | 1.5 | Tight contact shadows only |

---

## Environment Map Reference

### Drei Environment Presets

| Preset | Best For |
|--------|----------|
| `apartment` | Interior product shots, warm tones |
| `city` | Urban scenes, reflective surfaces |
| `dawn` | Warm outdoor lighting |
| `forest` | Natural, green-tinted reflections |
| `lobby` | Neutral interior, even lighting |
| `night` | Dark scenes with point light accents |
| `park` | Daylight outdoor, green reflections |
| `studio` | Product photography, clean reflections |
| `sunset` | Warm dramatic lighting |
| `warehouse` | Industrial, diffuse lighting |

### HDR Loading Pattern (Imperative)

```
1. Load HDR with RGBELoader or EXRLoader
2. Create PMREMGenerator from renderer
3. Call pmremGenerator.fromEquirectangular(hdrTexture)
4. Assign result.texture to scene.environment
5. Optionally assign to scene.background
6. Dispose the original HDR texture and PMREMGenerator
```

NEVER skip step 3 — unprocessed HDR textures produce incorrect reflections on PBR materials.
