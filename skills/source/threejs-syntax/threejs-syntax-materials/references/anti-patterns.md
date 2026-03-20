# Anti-Patterns (Three.js Materials & Textures)

## 1. Wrong Color Space on Data Textures

```javascript
// WRONG: Setting SRGBColorSpace on a normal map corrupts normal vectors
const normalMap = loader.load('normal.jpg');
normalMap.colorSpace = THREE.SRGBColorSpace; // CORRUPTED lighting!

// CORRECT: Data textures ALWAYS use NoColorSpace (the default)
const normalMap = loader.load('normal.jpg');
// colorSpace stays at NoColorSpace -- do NOT change it

// CORRECT: ONLY diffuse/emissive/color textures get SRGBColorSpace
const diffuseMap = loader.load('diffuse.jpg');
diffuseMap.colorSpace = THREE.SRGBColorSpace;
```

**WHY**: The renderer performs lighting calculations in linear space. Setting `SRGBColorSpace` on data textures (normal, roughness, metalness, AO, bump, displacement, alpha) applies an incorrect gamma correction, producing washed-out or oversaturated results and completely wrong surface normals.

---

## 2. Opacity Without Transparent Flag

```javascript
// WRONG: opacity is silently ignored without transparent: true
const material = new THREE.MeshStandardMaterial({
  color: 0xff0000,
  opacity: 0.5    // Has NO visible effect!
});

// CORRECT: ALWAYS pair opacity with transparent
const material = new THREE.MeshStandardMaterial({
  color: 0xff0000,
  opacity: 0.5,
  transparent: true
});
```

**WHY**: Three.js skips alpha blending entirely unless `transparent` is explicitly `true`. The `opacity` property exists on the material but is not applied during rendering without the flag.

---

## 3. Using MeshPhysicalMaterial Everywhere

```javascript
// WRONG: Using Physical for a simple colored object
const material = new THREE.MeshPhysicalMaterial({
  color: 0xff4444,
  roughness: 0.5,
  metalness: 0.0
  // No clearcoat, transmission, sheen, iridescence, or anisotropy used
});

// CORRECT: Use MeshStandardMaterial when advanced PBR features are not needed
const material = new THREE.MeshStandardMaterial({
  color: 0xff4444,
  roughness: 0.5,
  metalness: 0.0
});
```

**WHY**: `MeshPhysicalMaterial` compiles a significantly larger shader than `MeshStandardMaterial`. Every physical material instance adds shader compilation time and GPU overhead even if the advanced features are unused. ONLY use Physical when you need clearcoat, transmission, sheen, iridescence, anisotropy, or dispersion.

---

## 4. Forgetting to Dispose Materials and Textures

```javascript
// WRONG: Removing mesh from scene without disposing GPU resources
scene.remove(mesh);
// GPU memory for material, textures, and geometry is NEVER freed!

// CORRECT: ALWAYS dispose before removing
mesh.geometry.dispose();
for (const key of Object.keys(mesh.material)) {
  const value = mesh.material[key];
  if (value && value.isTexture) {
    value.dispose();
  }
}
mesh.material.dispose();
scene.remove(mesh);
```

**WHY**: `scene.remove()` only removes the object from the scene graph. GPU-side resources (compiled shaders, texture buffers, geometry buffers) are NOT freed automatically. Without explicit `dispose()` calls, GPU memory leaks accumulate and eventually crash the application or cause severe performance degradation.

---

## 5. Repeating Textures Without RepeatWrapping

```javascript
// WRONG: Setting repeat without changing wrapping mode
const texture = loader.load('tile.jpg');
texture.repeat.set(4, 4);
// Default ClampToEdgeWrapping: texture does NOT tile, edge pixels are stretched

// CORRECT: ALWAYS set wrapping mode when using repeat
const texture = loader.load('tile.jpg');
texture.wrapS = THREE.RepeatWrapping;
texture.wrapT = THREE.RepeatWrapping;
texture.repeat.set(4, 4);
```

**WHY**: The default wrapping mode is `ClampToEdgeWrapping`, which stretches the edge pixels of the texture infinitely. `repeat` values greater than `(1, 1)` only produce visible tiling when `wrapS` and `wrapT` are set to `RepeatWrapping` or `MirroredRepeatWrapping`.

---

## 6. Setting linewidth > 1 on LineBasicMaterial

```javascript
// WRONG: linewidth > 1 is silently ignored on most platforms
const material = new THREE.LineBasicMaterial({
  color: 0xff0000,
  linewidth: 5    // IGNORED on Windows, macOS, most Linux (WebGL limitation)
});

// CORRECT: Use Line2 + LineMaterial from addons for thick lines
import { Line2 } from 'three/addons/lines/Line2.js';
import { LineMaterial } from 'three/addons/lines/LineMaterial.js';
import { LineGeometry } from 'three/addons/lines/LineGeometry.js';

const geometry = new LineGeometry();
geometry.setPositions([0, 0, 0, 1, 1, 0, 2, 0, 0]);

const material = new LineMaterial({
  color: 0xff0000,
  linewidth: 5,    // Works on all platforms (screen-space pixels)
  resolution: new THREE.Vector2(window.innerWidth, window.innerHeight)
});

const line = new Line2(geometry, material);
scene.add(line);
```

**WHY**: The WebGL specification does not guarantee support for `lineWidth` values greater than 1. Most implementations (ANGLE on Windows/macOS, most Linux drivers) cap it at 1. The `Line2`/`LineMaterial` addon renders lines as screen-space triangles, bypassing this limitation.

---

## 7. Setting needsUpdate Every Frame

```javascript
// WRONG: Forcing shader recompilation every frame
function animate() {
  material.needsUpdate = true;  // Recompiles shader EVERY frame -- massive waste
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

// CORRECT: Only set needsUpdate once, after the change
material.flatShading = true;
material.needsUpdate = true;  // Set ONCE after the change
// Do NOT set again until the next property change
```

**WHY**: Setting `needsUpdate = true` triggers a full shader recompilation on the GPU. This is an expensive operation (can take 10-100ms per material). Doing it every frame causes severe stuttering and wasted GPU cycles. ONLY set it once after changing a property that affects the shader variant.

---

## 8. Forgetting uv2 for aoMap and lightMap

```javascript
// WRONG: Using aoMap without uv2 attribute -- AO will not render
const material = new THREE.MeshStandardMaterial({
  aoMap: aoTexture,
  aoMapIntensity: 1.0
});
const mesh = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), material);
// aoMap is silently ignored because geometry has no uv2 attribute

// CORRECT: ALWAYS add uv2 attribute when using aoMap or lightMap
const geometry = new THREE.BoxGeometry(1, 1, 1);
geometry.setAttribute('uv2', geometry.getAttribute('uv'));
const mesh = new THREE.Mesh(geometry, material);
```

**WHY**: `aoMap` and `lightMap` sample from the second UV channel (`uv2`). If the geometry does not have a `uv2` attribute, the maps are silently ignored with no error or warning. For simple cases, copying `uv` to `uv2` works. For baked lighting, use a dedicated UV unwrap for `uv2`.

---

## 9. flipY = true on Render Target Textures

```javascript
// WRONG: Default flipY on render target texture produces upside-down image
const renderTarget = new THREE.WebGLRenderTarget(512, 512);
renderer.setRenderTarget(renderTarget);
renderer.render(scene, camera);
renderer.setRenderTarget(null);

const screenMaterial = new THREE.MeshBasicMaterial({
  map: renderTarget.texture  // Upside down if flipY is not corrected
});

// CORRECT: Render target textures have flipY = false by default (correct)
// Do NOT manually set renderTarget.texture.flipY = true
// If using DataTexture, ALWAYS set flipY = false
const dataTexture = new THREE.DataTexture(data, width, height);
dataTexture.flipY = false;
dataTexture.needsUpdate = true;
```

**WHY**: Image files store pixels top-to-bottom, so `flipY = true` (default) corrects for this. Render targets and DataTextures store pixels bottom-to-top (OpenGL convention). Setting `flipY = true` on these textures flips an already-correct image upside down.

---

## 10. Not Setting Toon Gradient Filter to Nearest

```javascript
// WRONG: Linear filtering blurs toon shading into smooth gradient
const gradientMap = loader.load('toon_gradient_3.png');
// Default filters: LinearFilter / LinearMipmapLinearFilter -- SMOOTH, not toon

const material = new THREE.MeshToonMaterial({
  color: 0xff4444,
  gradientMap: gradientMap   // Looks like MeshLambertMaterial, not toon!
});

// CORRECT: ALWAYS use NearestFilter for toon gradient maps
const gradientMap = loader.load('toon_gradient_3.png');
gradientMap.minFilter = THREE.NearestFilter;
gradientMap.magFilter = THREE.NearestFilter;

const material = new THREE.MeshToonMaterial({
  color: 0xff4444,
  gradientMap: gradientMap   // Sharp discrete shading steps
});
```

**WHY**: Toon shading works by quantizing light intensity into discrete steps using a gradient map (typically 3-5 pixels wide). Linear filtering interpolates between these steps, producing a smooth gradient that looks identical to Lambert shading. `NearestFilter` preserves the sharp step boundaries that create the cel-shaded look.
