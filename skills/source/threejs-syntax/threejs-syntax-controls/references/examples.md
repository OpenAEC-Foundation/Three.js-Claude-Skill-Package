# threejs-syntax-controls -- Examples

## Example 1: OrbitControls with Damping

Standard setup for inspecting a 3D model with smooth camera movement.

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

// Scene setup
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(5, 3, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Controls -- ALWAYS pass renderer.domElement
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.minDistance = 2;
controls.maxDistance = 20;
controls.maxPolarAngle = Math.PI / 2; // Prevent camera going below ground
controls.target.set(0, 1, 0);         // Look at object center

// Add a mesh
const geometry = new THREE.BoxGeometry();
const material = new THREE.MeshStandardMaterial({ color: 0x0077ff });
const cube = new THREE.Mesh(geometry, material);
cube.position.set(0, 1, 0);
scene.add(cube);

// Light
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

// Animation loop -- MUST call controls.update() when enableDamping is true
function animate() {
  requestAnimationFrame(animate);
  controls.update(); // REQUIRED for damping
  renderer.render(scene, camera);
}
animate();

// Cleanup on page unload
window.addEventListener('beforeunload', () => {
  controls.dispose();
  renderer.dispose();
});
```

---

## Example 2: PointerLockControls for First-Person

First-person camera with WASD movement. Pointer lock activates on click.

```javascript
import * as THREE from 'three';
import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 1.6, 5); // Eye height

const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new PointerLockControls(camera, renderer.domElement);

// MUST activate from user gesture
document.addEventListener('click', () => {
  controls.lock();
});

controls.addEventListener('lock', () => {
  console.log('Pointer locked -- use mouse to look around');
});

controls.addEventListener('unlock', () => {
  console.log('Pointer unlocked');
});

// Movement state
const velocity = new THREE.Vector3();
const direction = new THREE.Vector3();
const moveState = { forward: false, backward: false, left: false, right: false };

document.addEventListener('keydown', (event) => {
  switch (event.code) {
    case 'KeyW': moveState.forward = true; break;
    case 'KeyS': moveState.backward = true; break;
    case 'KeyA': moveState.left = true; break;
    case 'KeyD': moveState.right = true; break;
  }
});

document.addEventListener('keyup', (event) => {
  switch (event.code) {
    case 'KeyW': moveState.forward = false; break;
    case 'KeyS': moveState.backward = false; break;
    case 'KeyA': moveState.left = false; break;
    case 'KeyD': moveState.right = false; break;
  }
});

const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();

  if (controls.isLocked) {
    // Deceleration
    velocity.x -= velocity.x * 10.0 * delta;
    velocity.z -= velocity.z * 10.0 * delta;

    // Direction from input
    direction.z = Number(moveState.forward) - Number(moveState.backward);
    direction.x = Number(moveState.right) - Number(moveState.left);
    direction.normalize();

    const speed = 5.0;
    if (moveState.forward || moveState.backward) velocity.z -= direction.z * speed * delta;
    if (moveState.left || moveState.right) velocity.x -= direction.x * speed * delta;

    controls.moveRight(-velocity.x * delta);
    controls.moveForward(-velocity.z * delta);
  }

  renderer.render(scene, camera);
}
animate();
```

---

## Example 3: TransformControls with OrbitControls

Editor setup where you can orbit the scene AND move/rotate/scale objects with a gizmo.

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { TransformControls } from 'three/addons/controls/TransformControls.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(5, 3, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Orbit controls for camera
const orbitControls = new OrbitControls(camera, renderer.domElement);
orbitControls.enableDamping = true;

// Transform controls for object manipulation
const transformControls = new TransformControls(camera, renderer.domElement);
scene.add(transformControls); // MUST add to scene

// CRITICAL: Disable orbit while dragging the gizmo
transformControls.addEventListener('dragging-changed', (event) => {
  orbitControls.enabled = !event.value;
});

// Create an object to manipulate
const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0xff4444 })
);
scene.add(cube);
transformControls.attach(cube);

// Keyboard shortcuts for mode switching
document.addEventListener('keydown', (event) => {
  switch (event.key) {
    case 'g': transformControls.setMode('translate'); break;
    case 'r': transformControls.setMode('rotate'); break;
    case 's': transformControls.setMode('scale'); break;
    case 'x': transformControls.showX = !transformControls.showX; break;
    case 'y': transformControls.showY = !transformControls.showY; break;
    case 'z': transformControls.showZ = !transformControls.showZ; break;
    case ' ': // Toggle world/local space
      transformControls.setSpace(
        transformControls.space === 'world' ? 'local' : 'world'
      );
      break;
  }
});

// Snapping with Ctrl held
transformControls.addEventListener('objectChange', () => {
  // React to object transform changes
  console.log('Object moved to:', cube.position.toArray());
});

scene.add(new THREE.AmbientLight(0xffffff, 0.5));
scene.add(new THREE.DirectionalLight(0xffffff, 1).translateZ(5));

function animate() {
  requestAnimationFrame(animate);
  orbitControls.update();
  renderer.render(scene, camera);
}
animate();

// Cleanup
function cleanup() {
  transformControls.dispose();
  orbitControls.dispose();
  renderer.dispose();
}
```

---

## Example 4: FlyControls for Free Flight

Unrestricted camera movement with WASD and mouse.

```javascript
import * as THREE from 'three';
import { FlyControls } from 'three/addons/controls/FlyControls.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 5000);
camera.position.set(0, 50, 200);

const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new FlyControls(camera, renderer.domElement);
controls.movementSpeed = 50;
controls.rollSpeed = 0.5;
controls.dragToLook = true; // Only rotate when dragging mouse

// Add some geometry to fly around
for (let i = 0; i < 200; i++) {
  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(10, 10, 10),
    new THREE.MeshNormalMaterial()
  );
  mesh.position.set(
    Math.random() * 1000 - 500,
    Math.random() * 1000 - 500,
    Math.random() * 1000 - 500
  );
  scene.add(mesh);
}

const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  controls.update(delta); // MUST pass delta -- NEVER omit
  renderer.render(scene, camera);
}
animate();
```

---

## Example 5: MapControls for Top-Down View

Map-like navigation for architectural or strategy views.

```javascript
import * as THREE from 'three';
import { MapControls } from 'three/addons/controls/MapControls.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 50, 50);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const controls = new MapControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;
controls.maxPolarAngle = Math.PI / 2.5; // Prevent camera going below horizon
controls.minDistance = 10;
controls.maxDistance = 200;

// Ground plane
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(200, 200),
  new THREE.MeshStandardMaterial({ color: 0x228B22 })
);
ground.rotation.x = -Math.PI / 2;
scene.add(ground);

// Grid helper
scene.add(new THREE.GridHelper(200, 40, 0x000000, 0x444444));

// Buildings
for (let i = 0; i < 20; i++) {
  const height = Math.random() * 10 + 2;
  const building = new THREE.Mesh(
    new THREE.BoxGeometry(4, height, 4),
    new THREE.MeshStandardMaterial({ color: 0x888888 })
  );
  building.position.set(
    Math.random() * 100 - 50,
    height / 2,
    Math.random() * 100 - 50
  );
  scene.add(building);
}

scene.add(new THREE.AmbientLight(0xffffff, 0.6));
const sun = new THREE.DirectionalLight(0xffffff, 0.8);
sun.position.set(50, 100, 50);
scene.add(sun);

function animate() {
  requestAnimationFrame(animate);
  controls.update(); // REQUIRED for damping
  renderer.render(scene, camera);
}
animate();
```
