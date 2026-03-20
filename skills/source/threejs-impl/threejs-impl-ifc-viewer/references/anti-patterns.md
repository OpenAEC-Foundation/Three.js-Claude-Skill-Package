# Anti-Patterns (IFC/BIM Viewer)

## 1. Using Deprecated web-ifc-three

```javascript
// WRONG: web-ifc-three is deprecated and unmaintained
import { IFCLoader } from 'web-ifc-three/IFCLoader';
const loader = new IFCLoader();
loader.ifcManager.setWasmPath('./');
const model = await loader.loadAsync('/model.ifc');

// CORRECT: Use web-ifc directly with manual Three.js conversion
import * as WebIFC from 'web-ifc';
const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath('./');
await ifcApi.Init();
// ... extract geometry and convert to Three.js BufferGeometry

// CORRECT: Or use @thatopen/components for a high-level approach
import * as OBC from '@thatopen/components';
const components = new OBC.Components();
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();
const model = await ifcLoader.load(data);
```

**WHY**: `web-ifc-three` has not been updated since 2023. It has unresolved bugs, lacks support for newer IFC schemas, and will NEVER receive security patches. The IFC.js project officially migrated to @thatopen/components.

---

## 2. Forgetting to Close Models (WASM Memory Leak)

```javascript
// WRONG: Model is never closed, WASM memory leaks permanently
async function loadAndRender(ifcData) {
  const modelID = ifcApi.OpenModel(ifcData);
  const walls = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
  // ... render walls
  // modelID is lost -- WASM memory is leaked until page reload
}

// CORRECT: ALWAYS close the model when done
async function loadAndRender(ifcData) {
  const modelID = ifcApi.OpenModel(ifcData);
  try {
    const walls = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCWALL);
    // ... render walls
  } finally {
    ifcApi.CloseModel(modelID);
  }
}
```

**WHY**: WASM linear memory cannot be partially freed. Every `OpenModel` allocates memory inside the WASM heap. Without `CloseModel`, that memory is permanently occupied until the browser tab is refreshed or closed.

---

## 3. Loading All Geometry at Once for Large Files

```javascript
// WRONG: Loading 200MB IFC file entirely into WASM and extracting all geometry
const modelID = ifcApi.OpenModel(hugeIfcData); // 200MB Uint8Array
const allLines = ifcApi.GetAllLines(modelID);
for (const id of allLines) {
  const mesh = ifcApi.GetFlatMesh(modelID, id); // Extracts ALL geometry at once
  // ... thousands of Three.js meshes created simultaneously
}

// CORRECT: Load by storey or element type, dispose when not visible
const storeyIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCBUILDINGSTOREY);
for (const storeyID of storeyIDs) {
  // Only load geometry for elements in this storey
  // Hide/dispose storeys that are not currently viewed
}
```

**WHY**: Large IFC files can produce millions of triangles. Loading all geometry simultaneously exhausts both WASM heap memory and GPU memory. The browser will either crash with an OOM error or become unresponsive. ALWAYS load incrementally and dispose unused geometry.

---

## 4. Skipping components.init() or components.dispose()

```javascript
// WRONG: Missing init -- subsystems are not started
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();
world.scene = new OBC.SimpleScene(components);
world.renderer = new OBC.SimpleRenderer(components, container);
// Forgot components.init() -- render loop never starts

// WRONG: Missing dispose -- memory leaks on unmount
function destroyViewer() {
  container.innerHTML = ''; // DOM cleared but Three.js objects still in memory
}

// CORRECT: ALWAYS init after setup, ALWAYS dispose on teardown
const components = new OBC.Components();
// ... setup world, scene, camera, renderer ...
components.init(); // Start render loop and subsystems

// On teardown:
components.dispose(); // Releases ALL resources
```

**WHY**: `components.init()` starts the render loop and initializes all registered subsystems. Without it, nothing renders. `components.dispose()` releases Three.js scenes, renderers, geometries, materials, textures, WASM memory, and event listeners. Without it, every viewer mount/unmount cycle leaks GPU and CPU memory.

---

## 5. Ignoring License Implications

```javascript
// WRONG: Using AGPL-licensed code in a proprietary application
// Old IFC.js packages (openbim-components v1) were AGPL-3.0
// Using them in closed-source software violates the license
import { Components } from 'openbim-components'; // AGPL-3.0!

// CORRECT: Use @thatopen/components v3.x (MIT licensed)
import * as OBC from '@thatopen/components'; // MIT

// CORRECT: Or use web-ifc directly (MIT licensed)
import * as WebIFC from 'web-ifc'; // MIT
```

**WHY**: AGPL-3.0 requires that ANY application using the library (including over a network) must release its complete source code under AGPL-3.0. This applies even if the AGPL code runs server-side. The @thatopen packages v3.x moved to MIT, but ALWAYS verify the license field in `package.json` of your installed version.

---

## 6. Not Setting WASM Path Before Init

```javascript
// WRONG: Init without setting WASM path -- fails to find .wasm file
const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init(); // Error: web-ifc.wasm not found

// WRONG: Setting WASM path AFTER Init
const ifcApi = new WebIFC.IfcAPI();
await ifcApi.Init();
ifcApi.SetWasmPath('./wasm/'); // Too late -- Init already failed or used wrong path

// CORRECT: Set WASM path BEFORE Init
const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath('./wasm/'); // MUST point to directory with web-ifc.wasm
await ifcApi.Init();
```

**WHY**: `SetWasmPath` tells the WASM loader where to find the `.wasm` binary. If not set before `Init()`, the loader searches relative to the page URL and will fail in most bundler configurations. The WASM file MUST be copied to the public/static assets directory.

---

## 7. Not Disposing Three.js Geometry and Materials

```javascript
// WRONG: Removing mesh from scene without disposing GPU resources
scene.remove(ifcMesh);
// geometry and material are still in GPU memory

// CORRECT: ALWAYS dispose geometry and materials when removing meshes
function removeIfcMesh(mesh, scene) {
  scene.remove(mesh);
  mesh.geometry.dispose();
  if (Array.isArray(mesh.material)) {
    mesh.material.forEach((m) => m.dispose());
  } else {
    mesh.material.dispose();
  }
}

// For groups with many children:
function disposeGroup(group, scene) {
  group.traverse((child) => {
    if (child.isMesh) {
      child.geometry.dispose();
      if (Array.isArray(child.material)) {
        child.material.forEach((m) => m.dispose());
      } else {
        child.material.dispose();
      }
    }
  });
  scene.remove(group);
}
```

**WHY**: Three.js allocates GPU buffers for each geometry and material. `scene.remove()` only detaches the object from the scene graph -- it does NOT free GPU memory. Without explicit `dispose()` calls, GPU memory usage grows with every model load/unload cycle until the tab crashes.

---

## 8. Using GetLine with flatten=true on Large Models

```javascript
// WRONG: Flattening all entities in a loop (extremely slow for large models)
const allIDs = ifcApi.GetAllLines(modelID);
for (const id of allIDs) {
  const entity = ifcApi.GetLine(modelID, id, true); // flatten=true resolves ALL references
  processEntity(entity);
}

// CORRECT: Use flatten=false by default, only flatten when you need resolved references
const allIDs = ifcApi.GetAllLines(modelID);
for (const id of allIDs) {
  const entity = ifcApi.GetLine(modelID, id, false); // Fast -- returns reference IDs
  if (needsDetails(entity)) {
    const detailed = ifcApi.GetLine(modelID, id, true); // Only flatten when needed
    processDetailedEntity(detailed);
  }
}
```

**WHY**: `flatten=true` recursively resolves all entity references, which triggers many additional WASM calls per entity. On a model with 50,000+ entities, flattening every line can take minutes. ALWAYS use `flatten=false` for bulk operations and only flatten specific entities when their resolved properties are needed.
