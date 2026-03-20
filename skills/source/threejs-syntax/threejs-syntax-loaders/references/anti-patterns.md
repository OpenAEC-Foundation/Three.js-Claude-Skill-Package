# threejs-syntax-loaders — Anti-Patterns

## Anti-Pattern 1: Loading Assets in the Render Loop

**WRONG:**
```javascript
function animate() {
  requestAnimationFrame(animate);
  // WRONG: creates a new load request every frame (60+ times per second)
  loader.load('model.glb', (gltf) => {
    scene.add(gltf.scene);
  });
  renderer.render(scene, camera);
}
```

**WHY:** Each frame spawns a new HTTP request and parse operation. This floods the network, consumes unbounded memory, and adds duplicate objects to the scene every frame.

**CORRECT:**
```javascript
// Load ONCE during initialization
const gltf = await gltfLoader.loadAsync('model.glb');
scene.add(gltf.scene);

// Render loop does NOT load anything
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
```

---

## Anti-Pattern 2: Missing DRACOLoader Decoder Path

**WRONG:**
```javascript
const dracoLoader = new DRACOLoader();
// Forgot to call setDecoderPath()
gltfLoader.setDRACOLoader(dracoLoader);
const gltf = await gltfLoader.loadAsync('compressed.glb');
// Runtime error: decoder WASM not found
```

**WHY:** Draco decompression requires WASM files (`draco_decoder.wasm`, `draco_wasm_wrapper.js`). Without the correct path, loading silently fails or throws a cryptic error.

**CORRECT:**
```javascript
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
gltfLoader.setDRACOLoader(dracoLoader);
```

---

## Anti-Pattern 3: Not Disposing Loaded Models

**WRONG:**
```javascript
// Remove model from scene but forget to dispose GPU resources
scene.remove(model);
// GPU memory leak: geometry buffers, shader programs, and textures remain allocated
```

**WHY:** Removing an object from the scene does NOT free GPU memory. Geometry, materials, and textures MUST be explicitly disposed. Repeated load-remove cycles without disposal cause the application to run out of GPU memory and crash.

**CORRECT:**
```javascript
function disposeModel(model) {
  model.traverse((child) => {
    if (child.isMesh) {
      child.geometry.dispose();
      if (Array.isArray(child.material)) {
        child.material.forEach((mat) => {
          Object.values(mat).forEach((val) => {
            if (val && typeof val === 'object' && 'dispose' in val) {
              val.dispose();
            }
          });
          mat.dispose();
        });
      } else {
        Object.values(child.material).forEach((val) => {
          if (val && typeof val === 'object' && 'dispose' in val) {
            val.dispose();
          }
        });
        child.material.dispose();
      }
    }
  });
  model.removeFromParent();
}
```

---

## Anti-Pattern 4: No Error Handling on loadAsync

**WRONG:**
```javascript
// No try/catch -- if the file does not exist, the promise rejects silently
const gltf = await gltfLoader.loadAsync('missing-model.glb');
scene.add(gltf.scene);
// Unhandled promise rejection -- may crash the application
```

**WHY:** Network failures, 404 responses, and corrupt files cause unhandled promise rejections. In strict environments this crashes the application. Without error handling, there is no feedback about what went wrong.

**CORRECT:**
```javascript
try {
  const gltf = await gltfLoader.loadAsync('model.glb');
  scene.add(gltf.scene);
} catch (error) {
  console.error('Failed to load model:', error);
  // Show fallback geometry or user-facing error message
}
```

---

## Anti-Pattern 5: Wrong Color Space on Data Textures

**WRONG:**
```javascript
const normalMap = await loader.loadAsync('normal.jpg');
normalMap.colorSpace = THREE.SRGBColorSpace; // WRONG for data textures
```

**WHY:** Normal maps, roughness maps, metalness maps, AO maps, and other non-color textures store mathematical data, not visible colors. Setting `SRGBColorSpace` applies gamma correction that corrupts the data values, causing incorrect lighting, broken normals, or washed-out PBR rendering.

**CORRECT:**
```javascript
// Color textures: ALWAYS set SRGBColorSpace
const diffuse = await loader.loadAsync('diffuse.jpg');
diffuse.colorSpace = THREE.SRGBColorSpace;

// Data textures: NEVER set SRGBColorSpace -- leave at default
const normalMap = await loader.loadAsync('normal.jpg');
// normalMap.colorSpace stays at NoColorSpace (default)
```

---

## Anti-Pattern 6: Not Disposing DRACOLoader After Use

**WRONG:**
```javascript
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/');
gltfLoader.setDRACOLoader(dracoLoader);

await gltfLoader.loadAsync('model1.glb');
await gltfLoader.loadAsync('model2.glb');
// WASM decoder stays in memory forever
```

**WHY:** The DRACOLoader spawns Web Workers with WASM decoders. If not disposed after loading is complete, these workers and their WASM memory remain allocated for the entire application lifetime.

**CORRECT:**
```javascript
await gltfLoader.loadAsync('model1.glb');
await gltfLoader.loadAsync('model2.glb');
// All Draco models loaded -- free decoder resources
dracoLoader.dispose();
```

---

## Anti-Pattern 7: Loading .gltf Without .bin Companion

**WRONG:**
```
/models/
  scene.gltf     <-- deployed
  // scene.bin   <-- MISSING
```

**WHY:** Separated `.gltf` files reference an external `.bin` file for mesh data. If the `.bin` file is missing, the loader fails with a network error. This is a common deployment mistake.

**CORRECT:** Use `.glb` (binary GLTF) for single-file distribution, or ensure both `.gltf` and `.bin` files are deployed together.

```javascript
// Preferred: single-file binary format
const gltf = await gltfLoader.loadAsync('scene.glb');
```

---

## Anti-Pattern 8: Loading OBJ Before MTL

**WRONG:**
```javascript
// Loading OBJ first -- materials are not available
const model = await objLoader.loadAsync('model.obj');
// Now loading MTL after the fact -- too late
const materials = await mtlLoader.loadAsync('model.mtl');
// model already has default white materials
```

**WHY:** OBJLoader assigns materials at parse time. If MTL materials are not set on the loader before calling `load()`, all meshes get default white `MeshPhongMaterial`.

**CORRECT:**
```javascript
// ALWAYS load MTL first
const materials = await mtlLoader.loadAsync('model.mtl');
materials.preload();
objLoader.setMaterials(materials);

// Then load OBJ -- materials are applied during parsing
const model = await objLoader.loadAsync('model.obj');
```

---

## Anti-Pattern 9: Forgetting KTX2Loader.detectSupport()

**WRONG:**
```javascript
const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('/basis/');
// Forgot detectSupport() -- transcoder does not know which GPU formats are available
gltfLoader.setKTX2Loader(ktx2Loader);
```

**WHY:** KTX2 textures are transcoded to GPU-native compressed formats (BC, ETC, ASTC). Without calling `detectSupport(renderer)`, the transcoder does not know which formats the GPU supports and either fails or picks a suboptimal fallback.

**CORRECT:**
```javascript
const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('/basis/');
ktx2Loader.detectSupport(renderer); // MUST pass the renderer instance
gltfLoader.setKTX2Loader(ktx2Loader);
```

---

## Anti-Pattern 10: Missing CORS Headers for Cross-Origin Assets

**WRONG:**
```javascript
// Loading from a different domain without CORS
const texture = loader.load('https://other-domain.com/texture.jpg');
// Fails silently or shows security error
```

**WHY:** Browsers block cross-origin resource loading unless the server sends proper CORS headers (`Access-Control-Allow-Origin`). Texture loads fail silently, resulting in black or missing textures with no console error in some cases.

**CORRECT:**
```javascript
// Option 1: Serve assets from the same origin
const texture = loader.load('/textures/texture.jpg');

// Option 2: Configure CORS on the loader and ensure server sends CORS headers
loader.setCrossOrigin('anonymous');
const texture = loader.load('https://cdn.example.com/texture.jpg');
// Server MUST respond with: Access-Control-Allow-Origin: *
```
