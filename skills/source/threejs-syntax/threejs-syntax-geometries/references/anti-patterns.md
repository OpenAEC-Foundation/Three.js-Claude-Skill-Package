# Anti-Patterns (Three.js Geometries r160+)

## 1. Forgetting needsUpdate After Modifying BufferAttribute

```javascript
// WRONG: GPU buffer will NOT update — changes are invisible
const positions = geometry.getAttribute('position');
positions.setXYZ(0, 5, 0, 0);
// Nothing happens on screen

// CORRECT: ALWAYS set needsUpdate = true after modifying attribute data
const positions = geometry.getAttribute('position');
positions.setXYZ(0, 5, 0, 0);
positions.needsUpdate = true;
```

**WHY**: Three.js does NOT automatically detect changes to typed arrays. The `needsUpdate` flag tells the renderer to re-upload the data to the GPU on the next frame.

---

## 2. Forgetting instanceMatrix.needsUpdate on InstancedMesh

```javascript
// WRONG: All instances render at the origin (identity matrix)
const mesh = new THREE.InstancedMesh(geometry, material, 100);
const dummy = new THREE.Object3D();
for (let i = 0; i < 100; i++) {
  dummy.position.set(i * 2, 0, 0);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
// Forgot: mesh.instanceMatrix.needsUpdate = true;

// CORRECT: ALWAYS set needsUpdate after all setMatrixAt calls
mesh.instanceMatrix.needsUpdate = true;
```

**WHY**: `setMatrixAt` writes to a CPU-side buffer. Without `needsUpdate = true`, the GPU never receives the updated matrices.

---

## 3. Using In-Place Transform Methods in Animation Loops

```javascript
// WRONG: geometry.translate modifies vertex data permanently — each frame accumulates
function animate() {
  geometry.translate(0, 0.01, 0); // Vertices drift further every frame!
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

// CORRECT: Use the mesh transform for runtime animation
function animate() {
  mesh.position.y += 0.01;
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

**WHY**: `geometry.translate()`, `geometry.rotateX()`, `geometry.scale()`, and similar methods permanently alter the vertex position data in the BufferAttribute. They are intended for one-time geometry alignment, NEVER for animation.

---

## 4. Not Disposing Geometry When Removing from Scene

```javascript
// WRONG: GPU memory leak — vertex buffers stay allocated
scene.remove(mesh);
mesh = null; // JavaScript GC frees the JS object, but GPU buffers remain

// CORRECT: ALWAYS dispose geometry (and material) when removing permanently
scene.remove(mesh);
mesh.geometry.dispose();
mesh.material.dispose();
mesh = null;
```

**WHY**: The JavaScript garbage collector only frees JavaScript objects. GPU-side vertex buffers, index buffers, and shader programs are NOT freed automatically. You MUST call `dispose()` explicitly.

---

## 5. Not Computing Normals on Custom Geometry

```javascript
// WRONG: MeshStandardMaterial renders completely black
const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
geometry.setIndex(indices);
const mesh = new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ color: 0xff0000 }));

// CORRECT: ALWAYS compute normals for lit materials
const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
geometry.setIndex(indices);
geometry.computeVertexNormals(); // Required for lighting calculations
const mesh = new THREE.Mesh(geometry, new THREE.MeshStandardMaterial({ color: 0xff0000 }));
```

**WHY**: Lit materials (MeshStandardMaterial, MeshPhongMaterial, etc.) require normal vectors to calculate light interaction. Without normals, every surface normal defaults to zero, and all lighting calculations produce black.

---

## 6. Using StaticDrawUsage for Frequently Updated Attributes

```javascript
// WRONG: Default usage hint tells GPU driver the data is static
const positions = new THREE.Float32BufferAttribute(data, 3);
// positions.usage = THREE.StaticDrawUsage (default)
// Then updating every frame — GPU driver may use slow upload path

// CORRECT: Set DynamicDrawUsage BEFORE the first render
const positions = new THREE.Float32BufferAttribute(data, 3);
positions.usage = THREE.DynamicDrawUsage; // MUST be set before first render
geometry.setAttribute('position', positions);
```

**WHY**: GPU drivers use the usage hint to decide where to allocate the buffer. `StaticDrawUsage` places the buffer in GPU-optimized memory that is slow to update. `DynamicDrawUsage` uses a buffer that is fast to update from the CPU. Changing usage after the first render has no effect on some drivers.

---

## 7. Trying to Resize InstancedMesh After Creation

```javascript
// WRONG: count is read-only after construction
const mesh = new THREE.InstancedMesh(geometry, material, 100);
mesh.count = 200; // This does NOT allocate more buffer space!

// CORRECT: Create with the maximum count, then control visible count
const maxCount = 1000;
const mesh = new THREE.InstancedMesh(geometry, material, maxCount);
mesh.count = 100; // Only render first 100 instances (read property is writable for this purpose)
// Later, increase up to maxCount:
mesh.count = 500; // Works because buffer was allocated for 1000
```

**WHY**: The `instanceMatrix` buffer is allocated once in the constructor based on `count`. You CANNOT grow it. Pre-allocate the maximum expected count and set `mesh.count` to control how many are rendered.

---

## 8. Assuming boundingBox/boundingSphere Are Auto-Computed

```javascript
// WRONG: boundingBox is null — causes crash or incorrect behavior
const geometry = new THREE.BufferGeometry();
geometry.setAttribute('position', new THREE.Float32BufferAttribute(data, 3));
console.log(geometry.boundingBox.min); // TypeError: Cannot read property 'min' of null

// CORRECT: ALWAYS call compute methods explicitly
geometry.computeBoundingBox();
geometry.computeBoundingSphere();
console.log(geometry.boundingBox.min); // Works
```

**WHY**: Bounding volumes are `null` by default and are NOT automatically computed. The renderer calls `computeBoundingSphere()` internally for frustum culling, but `boundingBox` is NEVER computed automatically. If your code needs bounding data, ALWAYS call the compute methods yourself.

---

## 9. Using Indexed Geometry for Flat Shading Without Duplicating Normals

```javascript
// WRONG: Shared vertices = shared normals = smooth shading (not flat)
const geometry = new THREE.BoxGeometry(1, 1, 1);
// BoxGeometry is already non-indexed with separate normals per face.
// But custom indexed geometry shares vertices, forcing smooth normals.

// CORRECT option A: Convert to non-indexed for true flat shading
const flatGeometry = indexedGeometry.toNonIndexed();
flatGeometry.computeVertexNormals(); // Now each face gets independent normals

// CORRECT option B: Use material flatShading property
const material = new THREE.MeshStandardMaterial({
  color: 0x888888,
  flatShading: true // Forces flat shading in the shader
});
```

**WHY**: Indexed geometry shares vertices between triangles, which means normals are interpolated (smooth). For true flat shading with indexed geometry, either convert to non-indexed or use `flatShading: true` on the material.

---

## 10. Creating Thousands of Individual Meshes Instead of InstancedMesh

```javascript
// WRONG: 10,000 draw calls — extremely slow
for (let i = 0; i < 10000; i++) {
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.set(Math.random() * 100, 0, Math.random() * 100);
  scene.add(mesh);
}

// CORRECT: Single draw call with InstancedMesh
const instancedMesh = new THREE.InstancedMesh(geometry, material, 10000);
const dummy = new THREE.Object3D();
for (let i = 0; i < 10000; i++) {
  dummy.position.set(Math.random() * 100, 0, Math.random() * 100);
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);
}
instancedMesh.instanceMatrix.needsUpdate = true;
scene.add(instancedMesh);
```

**WHY**: Each individual `Mesh` triggers a separate draw call. At 10,000+ meshes, the CPU overhead of issuing draw calls dominates frame time. `InstancedMesh` renders all instances in a single draw call using GPU instancing, which is orders of magnitude faster.

---

## 11. Forgetting to Call updateMatrix() Before setMatrixAt()

```javascript
// WRONG: dummy.matrix is still the identity matrix
const dummy = new THREE.Object3D();
dummy.position.set(5, 0, 0);
mesh.setMatrixAt(0, dummy.matrix); // Writes identity matrix, NOT the translated position

// CORRECT: ALWAYS call updateMatrix() after changing position/rotation/scale
const dummy = new THREE.Object3D();
dummy.position.set(5, 0, 0);
dummy.updateMatrix(); // Composes position/rotation/scale into dummy.matrix
mesh.setMatrixAt(0, dummy.matrix);
```

**WHY**: `Object3D.matrix` is NOT automatically updated when you change `position`, `rotation`, or `scale`. You MUST call `updateMatrix()` to compose these into the matrix before passing it to `setMatrixAt()`.
