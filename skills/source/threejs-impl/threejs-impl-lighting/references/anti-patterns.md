# threejs-impl-lighting — Anti-Patterns Reference

> Common lighting mistakes and their fixes. Every anti-pattern includes WHY it fails and the correct approach.

---

## AP-1: AmbientLight as Sole Light Source

**Wrong:**
```javascript
const ambient = new THREE.AmbientLight(0xffffff, 1.0);
scene.add(ambient);
// No other lights — scene looks flat and dimensionless
```

**Why it fails:** AmbientLight applies uniform color to all surfaces regardless of orientation. Without directional or point lights, there are NO shadows, NO highlights, and NO depth cues. Every surface appears equally lit.

**Correct:**
```javascript
const ambient = new THREE.AmbientLight(0x404040, 0.3);
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(5, 10, 7);
scene.add(ambient);
scene.add(dirLight);
```

ALWAYS combine AmbientLight with at least one directional, point, or spot light.

---

## AP-2: Forgetting to Add light.target to the Scene

**Wrong:**
```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(10, 20, 10);
dirLight.target.position.set(5, 0, 5); // Target repositioned but NOT in scene
scene.add(dirLight);
// Light still points at (0,0,0) — target position is ignored
```

**Why it fails:** The `target` is an `Object3D` whose world matrix must be updated by the renderer. If the target is not added to the scene graph, its world position is NEVER computed, so the light direction remains unchanged.

**Correct:**
```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 2);
dirLight.position.set(10, 20, 10);
scene.add(dirLight);
scene.add(dirLight.target); // MUST add to scene
dirLight.target.position.set(5, 0, 5); // Now this takes effect
```

This applies to BOTH `DirectionalLight.target` and `SpotLight.target`.

---

## AP-3: Using RectAreaLight Without Init

**Wrong:**
```javascript
const rectLight = new THREE.RectAreaLight(0xffffff, 5, 4, 10);
rectLight.position.set(0, 5, 0);
rectLight.lookAt(0, 0, 0);
scene.add(rectLight);
// RectAreaLight renders incorrectly or not at all
```

**Why it fails:** `RectAreaLight` requires precomputed LTC (Linearly Transformed Cosine) lookup textures that are NOT loaded by default. Without initialization, the shader lacks the data needed to evaluate area light integrals.

**Correct:**
```javascript
import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';

RectAreaLightUniformsLib.init(); // MUST call once before creating any RectAreaLight

const rectLight = new THREE.RectAreaLight(0xffffff, 5, 4, 10);
rectLight.position.set(0, 5, 0);
rectLight.lookAt(0, 0, 0);
scene.add(rectLight);
```

For WebGPURenderer, use `RectAreaLightTexturesLib` instead.

---

## AP-4: RectAreaLight with Non-PBR Materials

**Wrong:**
```javascript
RectAreaLightUniformsLib.init();
const rectLight = new THREE.RectAreaLight(0xffffff, 5, 4, 10);
scene.add(rectLight);

const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(),
  new THREE.MeshPhongMaterial({ color: 0xff0000 }) // NOT PBR
);
scene.add(mesh);
// RectAreaLight has NO effect on MeshPhongMaterial
```

**Why it fails:** `RectAreaLight` ONLY works with `MeshStandardMaterial` and `MeshPhysicalMaterial`. The LTC evaluation shader is only injected into PBR material programs.

**Correct:**
```javascript
const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(),
  new THREE.MeshStandardMaterial({ color: 0xff0000 })
);
```

---

## AP-5: Too Many Shadow-Casting PointLights

**Wrong:**
```javascript
for (let i = 0; i < 5; i++) {
  const light = new THREE.PointLight(0xffffff, 500, 15, 2);
  light.castShadow = true;
  light.position.set(i * 3, 3, 0);
  scene.add(light);
}
// 5 PointLights x 6 shadow passes = 30 shadow map renders per frame
```

**Why it fails:** Each shadow-casting `PointLight` renders the scene from 6 directions (cubemap faces). Five such lights produce 30 shadow render passes per frame, causing severe frame rate drops.

**Correct:**
```javascript
// Use SpotLights instead — only 1 shadow pass each
for (let i = 0; i < 5; i++) {
  const light = new THREE.SpotLight(0xffffff, 500, 15, Math.PI / 4, 0.3, 2);
  light.castShadow = true;
  light.position.set(i * 3, 3, 0);
  scene.add(light);
  scene.add(light.target);
}
```

NEVER use more than 1-2 shadow-casting PointLights. Prefer SpotLights with shadows.

---

## AP-6: Physically Correct Intensity Without Tone Mapping

**Wrong:**
```javascript
const sun = new THREE.DirectionalLight(0xffffff, 50000); // 50k lux
scene.add(sun);
// renderer.toneMapping is NoToneMapping (default)
// Scene is completely blown out to white
```

**Why it fails:** In r160+, intensity values represent real physical units. A DirectionalLight at 50,000 lux produces values far above the displayable [0,1] range. Without tone mapping, these values clamp to white.

**Correct:**
```javascript
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 0.5;

const sun = new THREE.DirectionalLight(0xffffff, 50000);
scene.add(sun);
```

ALWAYS set `renderer.toneMapping` when using physically correct intensity values.

---

## AP-7: Not Disposing PMREMGenerator

**Wrong:**
```javascript
const pmrem = new THREE.PMREMGenerator(renderer);
new RGBELoader().load('env.hdr', (texture) => {
  const envMap = pmrem.fromEquirectangular(texture);
  scene.environment = envMap.texture;
  texture.dispose();
  // pmrem is never disposed — GPU memory leak
});
```

**Why it fails:** `PMREMGenerator` allocates render targets and framebuffers on the GPU. Without calling `.dispose()`, these resources are NEVER freed, leading to GPU memory leaks that accumulate when loading multiple environments.

**Correct:**
```javascript
const pmrem = new THREE.PMREMGenerator(renderer);
new RGBELoader().load('env.hdr', (texture) => {
  const envMap = pmrem.fromEquirectangular(texture);
  scene.environment = envMap.texture;
  texture.dispose();
  pmrem.dispose(); // ALWAYS dispose after generating env maps
});
```

---

## AP-8: Setting SpotLight penumbra to 0

**Wrong:**
```javascript
const spot = new THREE.SpotLight(0xffffff, 1000);
spot.angle = Math.PI / 6;
spot.penumbra = 0; // Default — produces harsh, unrealistic cone edge
scene.add(spot);
```

**Why it fails:** A penumbra of 0 creates a perfectly sharp cone boundary, which looks artificial and jarring. Real-world spotlights ALWAYS have some edge softening.

**Correct:**
```javascript
spot.penumbra = 0.3; // Soft, realistic edge falloff
```

ALWAYS use `penumbra >= 0.1` for realistic spotlights.

---

## AP-9: Confusing scene.environment and scene.background

**Wrong:**
```javascript
scene.background = envMap.texture;
// Expects PBR reflections, but only set background — no IBL lighting
```

**Why it fails:** `scene.background` ONLY controls the visible backdrop. It does NOT affect PBR material reflections or ambient lighting. For IBL, you MUST set `scene.environment`.

**Correct:**
```javascript
scene.environment = envMap.texture;  // IBL reflections + ambient
scene.background = envMap.texture;   // Visible backdrop (optional)
```

`scene.environment` drives PBR lighting. `scene.background` is purely visual.

---

## AP-10: Orienting RectAreaLight with rotation Instead of lookAt

**Wrong:**
```javascript
const rectLight = new THREE.RectAreaLight(0xffffff, 5, 4, 10);
rectLight.position.set(0, 5, 0);
rectLight.rotation.x = -Math.PI / 2; // Confusing, error-prone
scene.add(rectLight);
```

**Why it fails:** RectAreaLight emits from its local Z-axis. Setting rotation manually requires understanding the local coordinate system and is error-prone with multiple rotations.

**Correct:**
```javascript
const rectLight = new THREE.RectAreaLight(0xffffff, 5, 4, 10);
rectLight.position.set(0, 5, 0);
rectLight.lookAt(0, 0, 0); // Clear, intuitive — light faces the target point
scene.add(rectLight);
```

ALWAYS use `.lookAt()` to orient RectAreaLights.

---

## AP-11: Forgetting to Update Light Helpers

**Wrong:**
```javascript
const helper = new THREE.DirectionalLightHelper(dirLight, 5);
scene.add(helper);

// Later, change light color or position...
dirLight.color.set(0xff0000);
dirLight.position.set(10, 20, 10);
// Helper still shows old state
```

**Why it fails:** Some light helpers do NOT auto-update when light properties change. The helper caches its visual representation at creation time.

**Correct:**
```javascript
dirLight.color.set(0xff0000);
dirLight.position.set(10, 20, 10);
helper.update(); // ALWAYS call after changing light properties
```
