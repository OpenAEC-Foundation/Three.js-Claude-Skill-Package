# threejs-errors-rendering — Anti-Patterns

## Anti-Pattern 1: Setting near to 0 or extremely small values

```javascript
// WRONG -- causes z-fighting across the entire scene
const camera = new THREE.PerspectiveCamera(75, aspect, 0.0001, 100000);
```

**Why it fails:** The depth buffer has limited precision (typically 24 bits). A huge far/near ratio (100000 / 0.0001 = 1 billion) means almost all precision is consumed near the camera, leaving virtually none for distant objects. This causes z-fighting everywhere beyond a few meters.

**Correct:**
```javascript
// ALWAYS keep near as large as possible and far as small as possible
const camera = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000);
// far/near ratio of 10000 is acceptable; above 100000 causes visible artifacts
```

---

## Anti-Pattern 2: Setting SRGBColorSpace on normal/data maps

```javascript
// WRONG -- corrupts normal direction data
normalMap.colorSpace = THREE.SRGBColorSpace;
roughnessMap.colorSpace = THREE.SRGBColorSpace;
```

**Why it fails:** Normal maps encode direction vectors, not colors. Applying sRGB gamma correction distorts the vector values, producing incorrect lighting. Roughness and metalness maps encode linear data -- gamma correction changes their effective values.

**Correct:**
```javascript
// Data textures: NEVER set SRGBColorSpace
// They default to LinearSRGBColorSpace, which is correct
const normalMap = textureLoader.load('normal.png');
// Leave colorSpace as default

// ONLY color textures get SRGBColorSpace:
const diffuseMap = textureLoader.load('diffuse.png');
diffuseMap.colorSpace = THREE.SRGBColorSpace;
```

---

## Anti-Pattern 3: Forgetting updateProjectionMatrix after camera changes

```javascript
// WRONG -- camera projection is stale
camera.fov = 90;
camera.near = 1;
camera.far = 500;
// Rendering continues with the OLD projection matrix
```

**Why it fails:** The projection matrix is computed once and cached. Changing `fov`, `near`, `far`, `aspect`, or `zoom` does NOT automatically recompute it. The renderer uses the stale matrix, producing incorrect perspective or clipping.

**Correct:**
```javascript
camera.fov = 90;
camera.near = 1;
camera.far = 500;
camera.updateProjectionMatrix(); // ALWAYS call after ANY camera property change
```

---

## Anti-Pattern 4: Setting material.needsUpdate = true every frame

```javascript
// WRONG -- recompiles shader program 60 times per second
function animate() {
  material.color.setHSL(time, 1, 0.5);
  material.needsUpdate = true; // NEVER do this in the render loop
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

**Why it fails:** `material.needsUpdate = true` triggers a full shader program recompilation, which is expensive (1-10ms per material). Doing this every frame causes severe frame drops. Changing `color`, `opacity`, or uniform values does NOT require `needsUpdate` -- Three.js uploads these automatically.

**Correct:**
```javascript
function animate() {
  material.color.setHSL(time, 1, 0.5); // This works WITHOUT needsUpdate
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

// ONLY set needsUpdate when changing structural properties:
material.map = newTexture;
material.needsUpdate = true; // correct -- structural change
```

---

## Anti-Pattern 5: Using transparent: true for hard-edge alpha

```javascript
// WRONG -- causes sorting issues and z-buffer artifacts
const material = new THREE.MeshStandardMaterial({
  map: fenceTexture,
  transparent: true,
  side: THREE.DoubleSide
});
```

**Why it fails:** `transparent: true` enables alpha blending, which requires correct back-to-front sorting. For hundreds of overlapping foliage or fence planes, Three.js cannot sort them correctly, producing visual glitches where background shows through in wrong order.

**Correct:**
```javascript
// For hard-edge alpha (foliage, fences, cutouts), use alphaTest
const material = new THREE.MeshStandardMaterial({
  map: fenceTexture,
  alphaTest: 0.5,  // discards fragments below threshold -- no sorting needed
  side: THREE.DoubleSide
  // NO transparent: true
});
```

---

## Anti-Pattern 6: Ignoring WebGL context loss

```javascript
// WRONG -- no recovery handler
const renderer = new THREE.WebGLRenderer();
// If context is lost, the canvas goes permanently black with no recovery
```

**Why it fails:** Mobile browsers, GPU driver crashes, and memory pressure can cause WebGL context loss at any time. Without a `webglcontextlost` event handler that calls `event.preventDefault()`, the context cannot be restored. Without a `webglcontextrestored` handler, the application never recovers.

**Correct:**
```javascript
renderer.domElement.addEventListener('webglcontextlost', (event) => {
  event.preventDefault(); // ALWAYS prevent default to enable restoration
  cancelAnimationFrame(animationId);
}, false);

renderer.domElement.addEventListener('webglcontextrestored', () => {
  initScene(); // re-create materials, textures, render targets
  animate();
}, false);
```

---

## Anti-Pattern 7: Forgetting instanceMatrix.needsUpdate

```javascript
// WRONG -- instances all render at origin
for (let i = 0; i < count; i++) {
  dummy.position.set(Math.random() * 10, 0, Math.random() * 10);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
// Missing: mesh.instanceMatrix.needsUpdate = true
scene.add(mesh);
```

**Why it fails:** `setMatrixAt` writes to a CPU-side buffer. The GPU buffer is NOT updated until `instanceMatrix.needsUpdate = true` is set. Without it, the GPU still has the initial zero-matrix data, placing all instances at the origin.

**Correct:**
```javascript
for (let i = 0; i < count; i++) {
  dummy.position.set(Math.random() * 10, 0, Math.random() * 10);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
mesh.instanceMatrix.needsUpdate = true; // ALWAYS set after setMatrixAt
scene.add(mesh);
```

---

## Anti-Pattern 8: Using logarithmicDepthBuffer with post-processing

```javascript
// WRONG -- depth reads in post-processing shaders break
const renderer = new THREE.WebGLRenderer({ logarithmicDepthBuffer: true });
const composer = new EffectComposer(renderer);
// SSAO, DOF, and other depth-dependent effects produce artifacts
```

**Why it fails:** Logarithmic depth buffer encodes depth non-linearly. Post-processing passes that read the depth buffer (SSAO, depth-of-field, fog) expect linear depth values. The mismatch produces visual artifacts.

**Correct:**
```javascript
// Choose ONE approach:
// Option A: logarithmic depth, no depth-dependent post-processing
const renderer = new THREE.WebGLRenderer({ logarithmicDepthBuffer: true });

// Option B: standard depth buffer with post-processing (preferred)
const renderer = new THREE.WebGLRenderer({ logarithmicDepthBuffer: false });
// Fix z-fighting by tightening near/far instead
camera.near = 1;
camera.far = 5000;
camera.updateProjectionMatrix();
```

---

## Anti-Pattern 9: Changing shadowMap.type after first render

```javascript
// WRONG -- causes shader cache invalidation
renderer.render(scene, camera);
renderer.shadowMap.type = THREE.VSMShadowMap; // changed after first render
```

**Why it fails:** Shadow map type is baked into compiled shader programs. Changing it after the first render forces ALL shadow-receiving materials to recompile their shaders, causing a significant frame drop.

**Correct:**
```javascript
// ALWAYS set shadow map configuration BEFORE the first render
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
// Then start rendering
animate();
```
