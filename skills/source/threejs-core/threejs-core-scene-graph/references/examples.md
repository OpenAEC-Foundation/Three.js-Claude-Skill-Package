# Working Code Examples (Three.js Scene Graph r160+)

## Example 1: Basic Scene Setup

A minimal scene with a mesh, camera, light, and render loop.

```javascript
import {
  Scene, PerspectiveCamera, WebGLRenderer,
  Mesh, BoxGeometry, MeshStandardMaterial,
  DirectionalLight, Color
} from 'three';

// Create scene
const scene = new Scene();
scene.background = new Color(0x1a1a2e);

// Create camera
const camera = new PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 2, 5);
camera.lookAt(0, 0, 0);

// Create light
const light = new DirectionalLight(0xffffff, 1);
light.position.set(5, 10, 5);
light.castShadow = true;
scene.add(light);

// Create mesh
const mesh = new Mesh(
  new BoxGeometry(1, 1, 1),
  new MeshStandardMaterial({ color: 0x00ff88 })
);
mesh.castShadow = true;
mesh.receiveShadow = true;
scene.add(mesh);

// Create renderer
const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

// Render loop
renderer.setAnimationLoop((time) => {
  mesh.rotation.y = time * 0.001;
  renderer.render(scene, camera);
});

// Handle resize
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

---

## Example 2: Hierarchical Object Groups

Demonstrates parent-child relationships and group transforms.

```javascript
import {
  Scene, Group, Mesh, BoxGeometry, CylinderGeometry,
  MeshStandardMaterial, Vector3
} from 'three';

const scene = new Scene();

// Create a building as a group of parts
const building = new Group();
building.name = 'building-01';

// Foundation
const foundation = new Mesh(
  new BoxGeometry(10, 0.5, 10),
  new MeshStandardMaterial({ color: 0x888888 })
);
foundation.position.y = 0.25;
foundation.name = 'foundation';

// Walls
const walls = new Mesh(
  new BoxGeometry(9, 6, 9),
  new MeshStandardMaterial({ color: 0xddccbb })
);
walls.position.y = 3.5;
walls.name = 'walls';

// Roof
const roof = new Mesh(
  new CylinderGeometry(0, 7, 3, 4),
  new MeshStandardMaterial({ color: 0xcc4444 })
);
roof.position.y = 8;
roof.rotation.y = Math.PI / 4;
roof.name = 'roof';

// Assemble -- all parts move together when building moves
building.add(foundation, walls, roof);
scene.add(building);

// Move entire building -- all children follow
building.position.set(20, 0, -15);
building.rotation.y = Math.PI / 6;

// Find parts by name
const roofRef = building.getObjectByName('roof');
if (roofRef) {
  roofRef.material.color.set(0x2244cc); // change roof color
}
```

---

## Example 3: Reparenting with attach() vs add()

Demonstrates the critical difference when moving objects between groups.

```javascript
import {
  Scene, Group, Mesh, SphereGeometry, MeshBasicMaterial, Vector3
} from 'three';

const scene = new Scene();

const arm = new Group();
arm.position.set(5, 0, 0);
scene.add(arm);

const hand = new Group();
hand.position.set(3, 0, 0); // 3 units from arm pivot
arm.add(hand);

const ball = new Mesh(
  new SphereGeometry(0.5),
  new MeshBasicMaterial({ color: 0xff0000 })
);
ball.position.set(0, 0, 0);
hand.add(ball);
// Ball world position = arm(5,0,0) + hand(3,0,0) + ball(0,0,0) = (8,0,0)

// WRONG approach: add() to scene -- ball jumps to origin
// scene.add(ball);
// Ball world position would become (0,0,0) because local transform (0,0,0) is now relative to scene

// CORRECT approach: attach() to scene -- ball stays at (8,0,0)
scene.attach(ball);
// ball.position is now (8,0,0) to preserve its world position

// Verify
const worldPos = new Vector3();
ball.getWorldPosition(worldPos);
console.log(worldPos); // Vector3 { x: 8, y: 0, z: 0 }
```

---

## Example 4: Layers for Selective Rendering

Using layers to separate visible objects from helpers and bloom effects.

```javascript
import {
  Scene, PerspectiveCamera, WebGLRenderer,
  Mesh, BoxGeometry, MeshStandardMaterial, MeshBasicMaterial,
  AxesHelper, GridHelper
} from 'three';

const LAYERS = {
  DEFAULT: 0,
  HELPERS: 1,
  BLOOM: 2,
};

const scene = new Scene();

// Production camera -- sees layer 0 only (default)
const prodCamera = new PerspectiveCamera(75, 16 / 9, 0.1, 1000);
prodCamera.position.set(0, 5, 10);
prodCamera.lookAt(0, 0, 0);

// Debug camera -- sees layers 0 AND 1
const debugCamera = new PerspectiveCamera(75, 16 / 9, 0.1, 1000);
debugCamera.position.set(0, 5, 10);
debugCamera.lookAt(0, 0, 0);
debugCamera.layers.enable(LAYERS.HELPERS);

// Regular mesh on layer 0 (default)
const cube = new Mesh(
  new BoxGeometry(1, 1, 1),
  new MeshStandardMaterial({ color: 0x00ff00 })
);
scene.add(cube);

// Helper on layer 1 -- invisible to prodCamera
const axes = new AxesHelper(5);
axes.layers.set(LAYERS.HELPERS); // ONLY on layer 1
scene.add(axes);

const grid = new GridHelper(20, 20);
grid.layers.set(LAYERS.HELPERS);
scene.add(grid);

// Bloom mesh on layer 2
const glowCube = new Mesh(
  new BoxGeometry(0.5, 0.5, 0.5),
  new MeshBasicMaterial({ color: 0xff8800 })
);
glowCube.layers.enable(LAYERS.BLOOM); // on layers 0 AND 2
glowCube.position.set(2, 1, 0);
scene.add(glowCube);

const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(800, 450);
document.body.appendChild(renderer.domElement);

// Toggle between cameras
let useDebug = false;

renderer.setAnimationLoop(() => {
  const camera = useDebug ? debugCamera : prodCamera;
  renderer.render(scene, camera);
});
```

---

## Example 5: Scene Fog and Environment

Setting up fog and environment-based lighting.

```javascript
import {
  Scene, PerspectiveCamera, WebGLRenderer,
  Mesh, PlaneGeometry, BoxGeometry,
  MeshStandardMaterial, MeshBasicMaterial,
  Fog, FogExp2, Color, ACESFilmicToneMapping, SRGBColorSpace
} from 'three';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const scene = new Scene();
scene.background = new Color(0xaabbcc);

// Linear fog: clear at 10 units, fully opaque at 150 units
scene.fog = new Fog(0xaabbcc, 10, 150);

// Or use exponential fog (comment out the above)
// scene.fog = new FogExp2(0xaabbcc, 0.015);

// Ground plane -- receives fog
const ground = new Mesh(
  new PlaneGeometry(200, 200),
  new MeshStandardMaterial({ color: 0x556655 })
);
ground.rotation.x = -Math.PI / 2;
scene.add(ground);

// Row of boxes fading into fog
for (let i = 0; i < 20; i++) {
  const box = new Mesh(
    new BoxGeometry(2, 2, 2),
    new MeshStandardMaterial({ color: 0xcc8844 })
  );
  box.position.set(0, 1, -i * 8);
  scene.add(box);
}

// Sky material ignores fog
const sky = new Mesh(
  new PlaneGeometry(500, 500),
  new MeshBasicMaterial({ color: 0xaabbcc, fog: false }) // fog: false
);
sky.position.set(0, 100, -200);
scene.add(sky);

// Load HDR environment for PBR lighting
const rgbeLoader = new RGBELoader();
rgbeLoader.load('environment.hdr', (texture) => {
  scene.environment = texture;
  scene.environmentIntensity = 0.8;
});

// Renderer with tone mapping
const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.toneMapping = ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.outputColorSpace = SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const camera = new PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 500);
camera.position.set(0, 5, 20);
camera.lookAt(0, 1, -50);

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
});
```
