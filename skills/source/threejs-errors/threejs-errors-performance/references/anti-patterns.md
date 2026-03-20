# threejs-errors-performance — Anti-Patterns

## Anti-Pattern 1: Creating Objects in the Render Loop

**WRONG:**
```javascript
function animate() {
  // LEAK: new geometry and material every frame, never disposed
  const geo = new THREE.BoxGeometry(1, 1, 1);
  const mat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
  const mesh = new THREE.Mesh(geo, mat);
  scene.add(mesh);
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

**Why it fails**: Each frame allocates new GPU buffers for geometry, compiles a new shader program for the material, and adds a new mesh to the scene. GPU memory grows without bound. The scene accumulates thousands of overlapping objects.

**CORRECT:**
```javascript
const geo = new THREE.BoxGeometry(1, 1, 1);
const mat = new THREE.MeshStandardMaterial({ color: 0xff0000 });
const mesh = new THREE.Mesh(geo, mat);
scene.add(mesh);

function animate() {
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

---

## Anti-Pattern 2: Forgetting to Dispose Textures on Replacement

**WRONG:**
```javascript
// Old texture stays in GPU memory forever
material.map = newTexture;
material.needsUpdate = true;
```

**Why it fails**: The old texture's GPU memory is NEVER freed. Over time, swapping textures without disposal accumulates hundreds of MB of leaked GPU memory. `renderer.info.memory.textures` will grow continuously.

**CORRECT:**
```javascript
if (material.map) material.map.dispose();
material.map = newTexture;
material.needsUpdate = true;
```

---

## Anti-Pattern 3: Individual Meshes Instead of InstancedMesh

**WRONG:**
```javascript
// 5000 draw calls — GPU cannot batch these
for (let i = 0; i < 5000; i++) {
  const mesh = new THREE.Mesh(geometry.clone(), material.clone());
  mesh.position.set(Math.random() * 100, 0, Math.random() * 100);
  scene.add(mesh);
}
```

**Why it fails**: Each `Mesh` produces a separate draw call. The GPU driver overhead for 5000 draw calls dominates frame time. Additionally, cloning geometry and material 5000 times wastes memory — identical copies occupy GPU memory 5000 times.

**CORRECT:**
```javascript
const mesh = new THREE.InstancedMesh(geometry, material, 5000);
const dummy = new THREE.Object3D();
for (let i = 0; i < 5000; i++) {
  dummy.position.set(Math.random() * 100, 0, Math.random() * 100);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
mesh.instanceMatrix.needsUpdate = true;
scene.add(mesh); // 1 draw call
```

---

## Anti-Pattern 4: Allocating Temporary Objects in the Animation Loop

**WRONG:**
```javascript
function animate() {
  // Creates garbage every frame — triggers frequent GC pauses
  const direction = new THREE.Vector3(1, 0, 0);
  const worldPos = new THREE.Vector3();
  mesh.getWorldPosition(worldPos);
  const distance = worldPos.distanceTo(new THREE.Vector3(0, 0, 0));
  requestAnimationFrame(animate);
}
```

**Why it fails**: JavaScript garbage collection pauses cause visible frame drops (stuttering). Creating `Vector3`, `Matrix4`, `Quaternion`, `Color`, or `Box3` objects every frame generates significant GC pressure at 60 FPS (60 allocations/second per object).

**CORRECT:**
```javascript
const _direction = new THREE.Vector3();
const _worldPos = new THREE.Vector3();
const _origin = new THREE.Vector3(0, 0, 0);

function animate() {
  _direction.set(1, 0, 0);
  mesh.getWorldPosition(_worldPos);
  const distance = _worldPos.distanceTo(_origin);
  requestAnimationFrame(animate);
}
```

---

## Anti-Pattern 5: Not Disposing Controls on Cleanup

**WRONG:**
```javascript
function destroyViewer() {
  scene.clear();
  renderer.dispose();
  renderer.domElement.remove();
  // controls.dispose() is missing!
}
```

**Why it fails**: `OrbitControls`, `MapControls`, `FlyControls`, and all other control classes attach event listeners to the DOM (`mousedown`, `wheel`, `touchstart`, `pointermove`, `keydown`, etc.). Without `controls.dispose()`, these listeners persist after the renderer is destroyed, causing errors when they try to interact with a disposed renderer and leaking memory via closures.

**CORRECT:**
```javascript
function destroyViewer() {
  controls.dispose(); // ALWAYS dispose controls first
  scene.traverse((object) => {
    if (object.geometry) object.geometry.dispose();
    if (object.material) {
      if (Array.isArray(object.material)) {
        object.material.forEach(disposeMaterial);
      } else {
        disposeMaterial(object.material);
      }
    }
  });
  scene.clear();
  renderer.dispose();
  renderer.domElement.remove();
}
```

---

## Anti-Pattern 6: Using MeshPhysicalMaterial Everywhere

**WRONG:**
```javascript
// Compiles a massive shader for every object, even simple walls
const wallMaterial = new THREE.MeshPhysicalMaterial({ color: 0xcccccc });
const floorMaterial = new THREE.MeshPhysicalMaterial({ color: 0x888888 });
```

**Why it fails**: `MeshPhysicalMaterial` compiles a significantly larger shader than `MeshStandardMaterial` because it includes code paths for clearcoat, transmission, sheen, iridescence, and anisotropy — even when those features are not used. This wastes GPU cycles on every fragment.

**CORRECT:**
```javascript
// Use MeshStandardMaterial for objects that do not need physical features
const wallMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc });
const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x888888 });

// ONLY use MeshPhysicalMaterial when you need its unique features
const glassMaterial = new THREE.MeshPhysicalMaterial({
  transmission: 1.0,
  roughness: 0.0,
  ior: 1.5
});
```

---

## Anti-Pattern 7: Forgetting instanceMatrix.needsUpdate

**WRONG:**
```javascript
const mesh = new THREE.InstancedMesh(geo, mat, 100);
const dummy = new THREE.Object3D();

for (let i = 0; i < 100; i++) {
  dummy.position.set(i * 2, 0, 0);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
// Missing: mesh.instanceMatrix.needsUpdate = true;
scene.add(mesh);
// Result: all 100 instances render at (0, 0, 0) stacked on top of each other
```

**Why it fails**: `setMatrixAt` writes to the CPU-side typed array but does NOT trigger GPU upload. The GPU buffer still contains the initial identity matrices (all zeros = origin). Without `needsUpdate = true`, the data NEVER reaches the GPU.

**CORRECT:**
```javascript
for (let i = 0; i < 100; i++) {
  dummy.position.set(i * 2, 0, 0);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
mesh.instanceMatrix.needsUpdate = true; // triggers GPU upload
scene.add(mesh);
```

---

## Anti-Pattern 8: Disposing Shared Resources Too Early

**WRONG:**
```javascript
const sharedGeometry = new THREE.BoxGeometry(1, 1, 1);
const sharedMaterial = new THREE.MeshStandardMaterial({ color: 0x00ff00 });

const meshA = new THREE.Mesh(sharedGeometry, sharedMaterial);
const meshB = new THREE.Mesh(sharedGeometry, sharedMaterial);
scene.add(meshA, meshB);

// Later, remove only meshA
scene.remove(meshA);
sharedGeometry.dispose(); // BUG: meshB still uses this geometry!
sharedMaterial.dispose();  // BUG: meshB still uses this material!
// meshB now renders as broken/invisible
```

**Why it fails**: `dispose()` frees GPU resources immediately. All meshes referencing the disposed geometry or material will fail to render. There is no reference counting in Three.js — you MUST track shared resources manually.

**CORRECT:**
```javascript
scene.remove(meshA);
// Do NOT dispose shared resources until ALL consumers are removed
// Only dispose when meshB is also removed:
scene.remove(meshB);
sharedGeometry.dispose();
sharedMaterial.dispose();
```
