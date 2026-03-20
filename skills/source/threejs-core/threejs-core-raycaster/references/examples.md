# Working Code Examples (Three.js Raycaster)

## Example 1: Mouse Click Picking

Select an object by clicking on it. Uses a flat array of selectable objects and a reusable intersections array.

```javascript
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Create selectable objects
const selectableObjects = [];
const geometry = new THREE.BoxGeometry(1, 1, 1);

for (let i = 0; i < 10; i++) {
  const material = new THREE.MeshStandardMaterial({ color: 0x4488ff });
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.set((i - 5) * 2, 0, 0);
  scene.add(mesh);
  selectableObjects.push(mesh);
}

// Add light
scene.add(new THREE.DirectionalLight(0xffffff, 1));
scene.add(new THREE.AmbientLight(0x404040));

// Raycasting setup
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const intersections = [];
let selectedObject = null;

renderer.domElement.addEventListener('click', (event) => {
  // Convert mouse to NDC
  const rect = renderer.domElement.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

  raycaster.setFromCamera(mouse, camera);
  raycaster.intersectObjects(selectableObjects, false, intersections);

  // Deselect previous
  if (selectedObject) {
    selectedObject.material.color.set(0x4488ff);
    selectedObject = null;
  }

  // Select new
  if (intersections.length > 0) {
    selectedObject = intersections[0].object;
    selectedObject.material.color.set(0xff4444);
  }

  intersections.length = 0; // clear for reuse
});

function animate() {
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

---

## Example 2: Hover Detection with Pointer Move

Highlight objects on mouse hover. Uses a flag to throttle raycasting to the render loop.

```javascript
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Create hoverable objects
const hoverableObjects = [];
const geometry = new THREE.SphereGeometry(0.5, 32, 16);

for (let i = 0; i < 8; i++) {
  const material = new THREE.MeshStandardMaterial({ color: 0x22cc88 });
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.set(Math.cos(i * 0.8) * 4, 0, Math.sin(i * 0.8) * 4);
  scene.add(mesh);
  hoverableObjects.push(mesh);
}

scene.add(new THREE.DirectionalLight(0xffffff, 1));
scene.add(new THREE.AmbientLight(0x404040));

// Raycasting setup
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const intersections = [];
let hoveredObject = null;
let needsRaycast = false;

renderer.domElement.addEventListener('pointermove', (event) => {
  const rect = renderer.domElement.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
  needsRaycast = true;
});

function animate() {
  if (needsRaycast) {
    raycaster.setFromCamera(mouse, camera);
    raycaster.intersectObjects(hoverableObjects, false, intersections);

    // Unhover previous
    if (hoveredObject) {
      hoveredObject.material.color.set(0x22cc88);
      hoveredObject.material.emissive.set(0x000000);
      hoveredObject = null;
    }

    // Hover new
    if (intersections.length > 0) {
      hoveredObject = intersections[0].object;
      hoveredObject.material.color.set(0xffcc00);
      hoveredObject.material.emissive.set(0x333300);
      renderer.domElement.style.cursor = 'pointer';
    } else {
      renderer.domElement.style.cursor = 'default';
    }

    intersections.length = 0;
    needsRaycast = false;
  }

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

---

## Example 3: InstancedMesh Picking

Pick individual instances from an InstancedMesh using `instanceId`.

```javascript
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 10, 15);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Create InstancedMesh with 100 instances
const count = 100;
const geometry = new THREE.BoxGeometry(0.8, 0.8, 0.8);
const material = new THREE.MeshStandardMaterial();
const instancedMesh = new THREE.InstancedMesh(geometry, material, count);

const dummy = new THREE.Object3D();
const defaultColor = new THREE.Color(0x4488ff);

for (let i = 0; i < count; i++) {
  dummy.position.set(
    (i % 10) * 1.5 - 7.5,
    0,
    Math.floor(i / 10) * 1.5 - 7.5
  );
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);
  instancedMesh.setColorAt(i, defaultColor);
}
scene.add(instancedMesh);

scene.add(new THREE.DirectionalLight(0xffffff, 1));
scene.add(new THREE.AmbientLight(0x404040));

// Raycasting setup
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
const intersections = [];
const selectedColor = new THREE.Color(0xff4444);
let lastSelectedId = -1;

renderer.domElement.addEventListener('click', (event) => {
  const rect = renderer.domElement.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

  raycaster.setFromCamera(mouse, camera);
  raycaster.intersectObject(instancedMesh, false, intersections);

  // Deselect previous instance
  if (lastSelectedId >= 0) {
    instancedMesh.setColorAt(lastSelectedId, defaultColor);
  }

  if (intersections.length > 0 && intersections[0].instanceId !== undefined) {
    const id = intersections[0].instanceId;
    instancedMesh.setColorAt(id, selectedColor);
    lastSelectedId = id;

    // Retrieve instance transform
    const matrix = new THREE.Matrix4();
    instancedMesh.getMatrixAt(id, matrix);
    const position = new THREE.Vector3();
    position.setFromMatrixPosition(matrix);
    console.log(`Selected instance ${id} at position:`, position);
  } else {
    lastSelectedId = -1;
  }

  instancedMesh.instanceColor.needsUpdate = true;
  intersections.length = 0;
});

function animate() {
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

---

## Example 4: Layer-Based Selective Picking

Use layers to separate interactive objects from non-interactive decoration.

```javascript
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Layer assignments
const LAYER_DEFAULT = 0;
const LAYER_INTERACTIVE = 1;
const LAYER_DECORATION = 2;

// Camera must see all layers
camera.layers.enable(LAYER_INTERACTIVE);
camera.layers.enable(LAYER_DECORATION);

// Interactive objects (layer 1)
const interactiveGroup = [];
for (let i = 0; i < 5; i++) {
  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(1, 1, 1),
    new THREE.MeshStandardMaterial({ color: 0x4488ff })
  );
  mesh.position.set(i * 2 - 4, 0, 0);
  mesh.layers.set(LAYER_INTERACTIVE);
  scene.add(mesh);
  interactiveGroup.push(mesh);
}

// Decoration objects (layer 2) — NOT pickable
for (let i = 0; i < 20; i++) {
  const mesh = new THREE.Mesh(
    new THREE.SphereGeometry(0.2, 8, 8),
    new THREE.MeshStandardMaterial({ color: 0x888888 })
  );
  mesh.position.set(
    (Math.random() - 0.5) * 12,
    (Math.random() - 0.5) * 6,
    (Math.random() - 0.5) * 6
  );
  mesh.layers.set(LAYER_DECORATION);
  scene.add(mesh);
}

scene.add(new THREE.DirectionalLight(0xffffff, 1));
scene.add(new THREE.AmbientLight(0x404040));

// Raycaster — ONLY tests layer 1 (interactive)
const raycaster = new THREE.Raycaster();
raycaster.layers.set(LAYER_INTERACTIVE);

const mouse = new THREE.Vector2();
const intersections = [];

renderer.domElement.addEventListener('click', (event) => {
  const rect = renderer.domElement.getBoundingClientRect();
  mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

  raycaster.setFromCamera(mouse, camera);
  // Can safely test scene.children — decoration is filtered by layers
  raycaster.intersectObjects(scene.children, false, intersections);

  if (intersections.length > 0) {
    console.log('Hit interactive object:', intersections[0].object);
  }

  intersections.length = 0;
});

function animate() {
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

---

## Example 5: Downward Raycast for Ground Snapping

Cast a ray downward from an object to snap it to terrain height.

```javascript
import * as THREE from 'three';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 10, 15);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Create terrain (uneven plane)
const terrainGeometry = new THREE.PlaneGeometry(20, 20, 32, 32);
terrainGeometry.rotateX(-Math.PI / 2);
const vertices = terrainGeometry.attributes.position;
for (let i = 0; i < vertices.count; i++) {
  const x = vertices.getX(i);
  const z = vertices.getZ(i);
  vertices.setY(i, Math.sin(x * 0.5) * Math.cos(z * 0.5) * 2);
}
terrainGeometry.computeVertexNormals();
const terrain = new THREE.Mesh(
  terrainGeometry,
  new THREE.MeshStandardMaterial({ color: 0x44aa44 })
);
scene.add(terrain);

// Object to snap to ground
const character = new THREE.Mesh(
  new THREE.CapsuleGeometry(0.3, 1, 8, 16),
  new THREE.MeshStandardMaterial({ color: 0xff8800 })
);
scene.add(character);

scene.add(new THREE.DirectionalLight(0xffffff, 1));
scene.add(new THREE.AmbientLight(0x404040));

// Ground-snapping raycaster
const downRay = new THREE.Raycaster();
const downDirection = new THREE.Vector3(0, -1, 0);
const rayOrigin = new THREE.Vector3();
const intersections = [];

function snapToGround(object) {
  // Cast ray from above the object downward
  rayOrigin.copy(object.position);
  rayOrigin.y = 50; // start high above
  downRay.set(rayOrigin, downDirection);
  downRay.intersectObject(terrain, false, intersections);

  if (intersections.length > 0) {
    object.position.y = intersections[0].point.y + 0.8; // offset for capsule height
  }

  intersections.length = 0;
}

// Animate character moving along X axis, snapping to terrain
let time = 0;
function animate() {
  time += 0.01;
  character.position.x = Math.sin(time) * 8;
  character.position.z = Math.cos(time * 0.7) * 8;
  snapToGround(character);

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```
