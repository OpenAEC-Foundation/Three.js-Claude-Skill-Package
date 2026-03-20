# Working Code Examples (IFC/BIM Viewer)

## Example 1: Minimal web-ifc Viewer with Three.js

```javascript
import * as THREE from 'three';
import * as WebIFC from 'web-ifc';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

// Scene setup
const scene = new THREE.Scene();
scene.background = new THREE.Color(0xf0f0f0);
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(15, 15, 15);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new OrbitControls(camera, renderer.domElement);
scene.add(new THREE.AmbientLight(0xffffff, 0.6));
scene.add(new THREE.DirectionalLight(0xffffff, 0.8));

// Initialize web-ifc
const ifcApi = new WebIFC.IfcAPI();
ifcApi.SetWasmPath('./wasm/');  // Directory containing web-ifc.wasm
await ifcApi.Init();

// Load IFC file from fetch
const response = await fetch('/models/building.ifc');
const buffer = await response.arrayBuffer();
const ifcData = new Uint8Array(buffer);
const modelID = ifcApi.OpenModel(ifcData);

// Extract and render all geometry
const allTypes = ifcApi.GetAllTypesOfModel(modelID);
for (const typeInfo of allTypes) {
  const ids = ifcApi.GetLineIDsWithType(modelID, typeInfo.typeID);
  for (const id of ids) {
    try {
      const flatMesh = ifcApi.GetFlatMesh(modelID, id);
      for (const pg of flatMesh.geometries) {
        const meshData = ifcApi.GetPlacedGeometry(modelID, pg);
        const threeMesh = convertToThreeMesh(meshData, pg);
        threeMesh.userData.expressID = id;
        scene.add(threeMesh);
      }
    } catch {
      // Not all entities have geometry -- skip silently
    }
  }
}

ifcApi.CloseModel(modelID); // Release WASM memory

function convertToThreeMesh(meshData, pg) {
  const { vertexData, indexData } = meshData;
  const vertexCount = vertexData.length / 6;
  const positions = new Float32Array(vertexCount * 3);
  const normals = new Float32Array(vertexCount * 3);

  for (let i = 0; i < vertexCount; i++) {
    positions[i * 3]     = vertexData[i * 6];
    positions[i * 3 + 1] = vertexData[i * 6 + 1];
    positions[i * 3 + 2] = vertexData[i * 6 + 2];
    normals[i * 3]       = vertexData[i * 6 + 3];
    normals[i * 3 + 1]   = vertexData[i * 6 + 4];
    normals[i * 3 + 2]   = vertexData[i * 6 + 5];
  }

  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geometry.setAttribute('normal', new THREE.BufferAttribute(normals, 3));
  geometry.setIndex(new THREE.BufferAttribute(indexData, 1));

  const { x: r, y: g, z: b, w: a } = pg.color;
  const material = new THREE.MeshPhongMaterial({
    color: new THREE.Color(r, g, b),
    opacity: a,
    transparent: a < 1,
    side: THREE.DoubleSide,
  });

  const mesh = new THREE.Mesh(geometry, material);
  const mat = new THREE.Matrix4().fromArray(pg.flatTransformation);
  mesh.applyMatrix4(mat);
  return mesh;
}

// Render loop
function animate() {
  requestAnimationFrame(animate);
  controls.update();
  renderer.render(scene, camera);
}
animate();
```

---

## Example 2: @thatopen/components Full BIM Viewer

```javascript
import * as OBC from '@thatopen/components';

// Container element
const container = document.getElementById('viewer-container');

// Initialize components
const components = new OBC.Components();
const worlds = components.get(OBC.Worlds);
const world = worlds.create();

world.scene = new OBC.SimpleScene(components);
world.scene.setup();

world.camera = new OBC.SimpleCamera(components);
world.camera.controls.setLookAt(20, 20, 20, 0, 0, 0);

world.renderer = new OBC.SimpleRenderer(components, container);

components.init();

// Load IFC
const ifcLoader = components.get(OBC.IfcLoader);
await ifcLoader.setup();

const fileInput = document.getElementById('ifc-input');
fileInput.addEventListener('change', async (event) => {
  const file = event.target.files[0];
  if (!file) return;
  const data = new Uint8Array(await file.arrayBuffer());
  const model = await ifcLoader.load(data);
  // model is automatically added to the world scene
  console.log('Loaded model with', model.children.length, 'fragment meshes');
});

// Cleanup on page unload
window.addEventListener('beforeunload', () => {
  components.dispose();
});
```

---

## Example 3: IFC Element Picking with Raycasting

```javascript
import * as THREE from 'three';
import * as WebIFC from 'web-ifc';

// Assumes scene, camera, renderer already set up
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();
const ifcMeshes = []; // Populated during IFC loading

// Store expressID on each mesh during loading
function addIfcMesh(mesh, expressID) {
  mesh.userData.expressID = expressID;
  ifcMeshes.push(mesh);
  scene.add(mesh);
}

// Highlight material
const highlightMaterial = new THREE.MeshPhongMaterial({
  color: 0xff6600,
  opacity: 0.8,
  transparent: true,
  side: THREE.DoubleSide,
});

let previousSelection = null;
let previousMaterial = null;

renderer.domElement.addEventListener('click', (event) => {
  // Restore previous selection
  if (previousSelection) {
    previousSelection.material = previousMaterial;
  }

  pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
  pointer.y = -(event.clientY / window.innerHeight) * 2 + 1;

  raycaster.setFromCamera(pointer, camera);
  const intersects = raycaster.intersectObjects(ifcMeshes);

  if (intersects.length > 0) {
    const hit = intersects[0].object;
    previousSelection = hit;
    previousMaterial = hit.material;
    hit.material = highlightMaterial;

    console.log('Selected element expressID:', hit.userData.expressID);

    // Read properties from web-ifc (if model still open)
    const entity = ifcApi.GetLine(modelID, hit.userData.expressID, true);
    console.log('Entity type:', entity.constructor.name);
    console.log('Name:', entity.Name?.value);
  }
});
```

---

## Example 4: Extracting IFC Spatial Tree

```javascript
import * as WebIFC from 'web-ifc';

function buildSpatialTree(ifcApi, modelID) {
  const tree = {};

  // Get the project (root node)
  const projectIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCPROJECT);
  if (projectIDs.length === 0) return tree;

  function getChildren(parentID) {
    const node = {
      expressID: parentID,
      entity: ifcApi.GetLine(modelID, parentID, false),
      children: [],
      elements: [],
    };
    node.name = node.entity.Name?.value || `#${parentID}`;

    // Aggregated children (IFCSITE -> IFCBUILDING -> IFCBUILDINGSTOREY)
    const aggIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCRELAGGREGATES);
    for (const aggID of aggIDs) {
      const rel = ifcApi.GetLine(modelID, aggID, false);
      if (rel.RelatingObject?.value === parentID) {
        const related = rel.RelatedObjects;
        for (const ref of related) {
          node.children.push(getChildren(ref.value));
        }
      }
    }

    // Contained elements (storey -> walls, slabs, etc.)
    const containIDs = ifcApi.GetLineIDsWithType(
      modelID, WebIFC.IFCRELCONTAINEDINSPATIALSTRUCTURE
    );
    for (const containID of containIDs) {
      const rel = ifcApi.GetLine(modelID, containID, false);
      if (rel.RelatingStructure?.value === parentID) {
        for (const ref of rel.RelatedElements) {
          node.elements.push({
            expressID: ref.value,
            entity: ifcApi.GetLine(modelID, ref.value, false),
          });
        }
      }
    }

    return node;
  }

  return getChildren(projectIDs[0]);
}

// Usage
const spatialTree = buildSpatialTree(ifcApi, modelID);
console.log(JSON.stringify(spatialTree, null, 2));
```

---

## Example 5: Loading IFC by Storey (Memory-Efficient)

```javascript
import * as WebIFC from 'web-ifc';
import * as THREE from 'three';

async function loadByStorey(ifcApi, modelID, scene) {
  const storeyIDs = ifcApi.GetLineIDsWithType(modelID, WebIFC.IFCBUILDINGSTOREY);
  const storeyGroups = new Map();

  for (const storeyID of storeyIDs) {
    const storey = ifcApi.GetLine(modelID, storeyID, false);
    const storeyName = storey.Name?.value || `Storey #${storeyID}`;
    const group = new THREE.Group();
    group.name = storeyName;
    group.userData.expressID = storeyID;

    // Find elements contained in this storey
    const containIDs = ifcApi.GetLineIDsWithType(
      modelID, WebIFC.IFCRELCONTAINEDINSPATIALSTRUCTURE
    );

    for (const containID of containIDs) {
      const rel = ifcApi.GetLine(modelID, containID, false);
      if (rel.RelatingStructure?.value !== storeyID) continue;

      for (const ref of rel.RelatedElements) {
        try {
          const flatMesh = ifcApi.GetFlatMesh(modelID, ref.value);
          for (const pg of flatMesh.geometries) {
            const meshData = ifcApi.GetPlacedGeometry(modelID, pg);
            const mesh = convertToThreeMesh(meshData, pg); // from Example 1
            mesh.userData.expressID = ref.value;
            group.add(mesh);
          }
        } catch {
          // Element has no geometry -- skip
        }
      }
    }

    scene.add(group);
    storeyGroups.set(storeyID, group);
  }

  return storeyGroups;
}

// Toggle storey visibility
function setStoreyVisible(storeyGroups, storeyID, visible) {
  const group = storeyGroups.get(storeyID);
  if (group) {
    group.visible = visible;
  }
}

// Dispose a storey to free GPU memory
function disposeStorey(storeyGroups, storeyID, scene) {
  const group = storeyGroups.get(storeyID);
  if (!group) return;

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
  storeyGroups.delete(storeyID);
}
```
