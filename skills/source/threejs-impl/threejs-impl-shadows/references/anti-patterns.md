# threejs-impl-shadows -- Anti-Patterns

## Anti-Pattern 1: Missing Shadow Opt-In Step

**Wrong**: Enabling shadows on the light but forgetting the renderer or mesh flags.

```js
// BAD -- shadows silently fail with no error
const light = new THREE.DirectionalLight(0xffffff, 1);
light.castShadow = true;
scene.add(light);

const cube = new THREE.Mesh(geometry, material);
scene.add(cube);
// No renderer.shadowMap.enabled
// No cube.castShadow
// No ground.receiveShadow
// Result: zero shadows, zero errors
```

**Correct**: ALWAYS set all three levels -- renderer, light, and mesh.

```js
renderer.shadowMap.enabled = true;
light.castShadow = true;
cube.castShadow = true;
ground.receiveShadow = true;
```

**Why**: Three.js shadows use a three-level opt-in system. Each level independently gates shadow behavior. Missing any one level silently produces no shadows.

---

## Anti-Pattern 2: Default DirectionalLight Shadow Camera Frustum

**Wrong**: Leaving the shadow camera frustum at default values.

```js
// BAD -- default frustum is tiny (-5 to 5)
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.castShadow = true;
scene.add(dirLight);
// Shadow camera frustum: left=-5, right=5, top=5, bottom=-5
// Result: most of the scene has no shadows or extremely pixelated shadows
```

**Correct**: ALWAYS size the frustum to match your scene.

```js
dirLight.shadow.camera.left = -30;
dirLight.shadow.camera.right = 30;
dirLight.shadow.camera.top = 30;
dirLight.shadow.camera.bottom = -30;
dirLight.shadow.camera.near = 0.5;
dirLight.shadow.camera.far = 100;
```

**Why**: The default orthographic frustum is -5 to 5 on each axis. For any scene larger than a 10-unit cube, shadows are either clipped or spread across too few texels. ALWAYS use `CameraHelper` to verify.

---

## Anti-Pattern 3: Excessive PointLight Shadow Usage

**Wrong**: Multiple shadow-casting PointLights in a scene.

```js
// BAD -- 3 PointLight shadows = 18 shadow render passes per frame
for (let i = 0; i < 3; i++) {
  const pl = new THREE.PointLight(0xffffff, 1, 50);
  pl.castShadow = true;
  pl.shadow.mapSize.set(1024, 1024);
  scene.add(pl);
}
// Result: 18 shadow maps per frame (6 per PointLight), severe frame drops
```

**Correct**: Replace PointLights with SpotLights where possible.

```js
// GOOD -- SpotLight costs 1 render pass per light
const spotLight = new THREE.SpotLight(0xffffff, 1, 50, Math.PI / 3);
spotLight.castShadow = true;
scene.add(spotLight);
```

**Why**: Each PointLight shadow renders a 6-face cubemap. Three shadow-casting PointLights at 1024x1024 means 18 render passes per frame. NEVER use more than 1 shadow-casting PointLight; prefer SpotLights at 1/6th the cost.

---

## Anti-Pattern 4: Over-Correcting Bias (Peter Panning)

**Wrong**: Setting shadow bias too high to fix acne, causing shadows to detach.

```js
// BAD -- excessive bias causes peter panning
dirLight.shadow.bias = -0.05; // way too large
// Result: shadows visibly float away from objects
```

**Correct**: Use minimal bias combined with normalBias.

```js
dirLight.shadow.bias = -0.0001;   // small depth bias
dirLight.shadow.normalBias = 0.02; // normal-based offset for curved surfaces
```

**Why**: `bias` shifts the entire shadow depth comparison. Large values push shadows away from surfaces entirely. `normalBias` offsets along the surface normal, which fixes acne on curved geometry without causing peter panning. ALWAYS try `normalBias` before increasing `bias`.

---

## Anti-Pattern 5: Non-Power-of-Two Shadow Map Size

**Wrong**: Using arbitrary shadow map dimensions.

```js
// BAD -- non-power-of-two dimensions
dirLight.shadow.mapSize.width = 1000;
dirLight.shadow.mapSize.height = 1000;
// Result: GPU may pad to next power-of-two anyway, wasting memory and causing artifacts
```

**Correct**: ALWAYS use power-of-two values.

```js
dirLight.shadow.mapSize.width = 1024;
dirLight.shadow.mapSize.height = 1024;
```

**Why**: WebGL texture hardware is optimized for power-of-two dimensions (256, 512, 1024, 2048, 4096). Non-power-of-two sizes may be silently padded, causing wasted memory and potential filtering artifacts.

---

## Anti-Pattern 6: Forgetting to Add Light Target to Scene

**Wrong**: Moving a DirectionalLight target without adding it to the scene.

```js
// BAD -- target position change has no effect
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(10, 10, 10);
dirLight.target.position.set(5, 0, 5); // target NOT in scene
scene.add(dirLight);
// Result: light still points at (0, 0, 0), shadow frustum is misaligned
```

**Correct**: ALWAYS add the target to the scene.

```js
scene.add(dirLight);
scene.add(dirLight.target);
dirLight.target.position.set(5, 0, 5); // NOW takes effect
```

**Why**: `Object3D.position` updates only take effect when the object is part of the scene graph so its world matrix gets computed. Without `scene.add(dirLight.target)`, the target's world position is never updated and the light direction remains default.

---

## Anti-Pattern 7: Shadow on Transparent Objects Without customDepthMaterial

**Wrong**: Expecting transparent or alpha-tested materials to cast correct shadows automatically.

```js
// BAD -- shadow ignores alpha channel entirely
const leafMaterial = new THREE.MeshStandardMaterial({
  map: leafTexture,
  alphaMap: alphaTexture,
  alphaTest: 0.5,
  transparent: true,
});

leafMesh.material = leafMaterial;
leafMesh.castShadow = true;
// Result: shadow is a solid rectangle ignoring the alpha cutout
```

**Correct**: Assign a matching `customDepthMaterial`.

```js
leafMesh.customDepthMaterial = new THREE.MeshDepthMaterial({
  depthPacking: THREE.RGBADepthPacking,
  map: leafTexture,
  alphaMap: alphaTexture,
  alphaTest: 0.5,
});
```

**Why**: The shadow depth pass uses a separate material (default `MeshDepthMaterial`) that knows nothing about your visible material's alpha settings. You MUST provide a `customDepthMaterial` with matching `alphaMap` and `alphaTest` values.

---

## Anti-Pattern 8: Changing Shadow Map Type at Runtime

**Wrong**: Switching `renderer.shadowMap.type` after shadows have been created.

```js
// BAD -- switching type after initial render
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
// ... render a few frames ...
renderer.shadowMap.type = THREE.VSMShadowMap;
// Result: corrupted shadow maps, visual glitches
```

**Correct**: Set shadow map type BEFORE any rendering occurs, or dispose all shadow maps first.

```js
// Set BEFORE first render
renderer.shadowMap.type = THREE.VSMShadowMap;
renderer.shadowMap.enabled = true;
```

**Why**: Shadow map textures are allocated with internal format and filtering settings specific to the shadow map type. Changing the type does not re-allocate existing shadow maps. ALWAYS set the type before the first render or manually call `.dispose()` on all light shadows before switching.

---

## Anti-Pattern 9: Shadow Map Resolution Too Large on Mobile

**Wrong**: Using desktop-grade shadow maps on mobile devices.

```js
// BAD -- 4096x4096 on mobile kills performance
dirLight.shadow.mapSize.width = 4096;
dirLight.shadow.mapSize.height = 4096;
// Result: GPU memory exhaustion, frame rate collapse on mobile
```

**Correct**: Scale shadow map size to device capability.

```js
const isMobile = /Mobi|Android/i.test(navigator.userAgent);
const shadowSize = isMobile ? 1024 : 2048;
dirLight.shadow.mapSize.width = shadowSize;
dirLight.shadow.mapSize.height = shadowSize;
```

**Why**: A 4096x4096 shadow map consumes 64MB of GPU memory (single channel, 32-bit float). Mobile GPUs have limited VRAM and fill rate. NEVER exceed 2048x2048 on mobile; prefer 1024x1024.

---

## Anti-Pattern 10: Not Using Static Shadow Optimization

**Wrong**: Re-rendering shadow maps every frame when nothing moves.

```js
// BAD -- shadow maps re-render every frame even though scene is static
renderer.shadowMap.enabled = true;
// autoUpdate defaults to true -- shadow maps rebuild every frame
```

**Correct**: Disable auto-update for static scenes.

```js
renderer.shadowMap.autoUpdate = false;
renderer.shadowMap.needsUpdate = true; // render once

// Later, if something moves:
function onSceneChanged() {
  renderer.shadowMap.needsUpdate = true;
}
```

**Why**: Shadow map rendering is one of the most expensive operations in a Three.js scene. For static scenes (architectural visualization, product displays), shadow maps NEVER change. Disabling `autoUpdate` eliminates per-frame shadow passes entirely.
