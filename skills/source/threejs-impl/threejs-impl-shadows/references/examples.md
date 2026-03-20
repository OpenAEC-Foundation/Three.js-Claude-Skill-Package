# threejs-impl-shadows -- Examples

## Example 1: Basic DirectionalLight Shadow Setup

The most common shadow scenario: a single directional light casting shadows onto a ground plane.

```js
import * as THREE from 'three';

// Scene setup
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.set(0, 5, 10);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);

// Step 1: Enable shadows on the renderer
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// Step 2: Create and configure light with shadows
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 7.5);
dirLight.castShadow = true;

// Configure shadow map resolution
dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;

// Configure orthographic frustum -- sized to scene
dirLight.shadow.camera.near = 0.5;
dirLight.shadow.camera.far = 50;
dirLight.shadow.camera.left = -10;
dirLight.shadow.camera.right = 10;
dirLight.shadow.camera.top = 10;
dirLight.shadow.camera.bottom = -10;

// Bias tuning
dirLight.shadow.bias = -0.0001;
dirLight.shadow.normalBias = 0.02;

scene.add(dirLight);
scene.add(dirLight.target);

// Step 3: Create objects with shadow flags
const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x00ff00 })
);
cube.position.y = 0.5;
cube.castShadow = true;
scene.add(cube);

const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20),
  new THREE.MeshStandardMaterial({ color: 0x808080 })
);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);

// Add ambient light for fill
scene.add(new THREE.AmbientLight(0x404040, 0.5));

// Debug helper -- REMOVE in production
const shadowHelper = new THREE.CameraHelper(dirLight.shadow.camera);
scene.add(shadowHelper);

function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

---

## Example 2: SpotLight Shadow with Soft Penumbra

SpotLight shadows auto-configure the perspective frustum from the light's cone angle.

```js
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

const scene = new THREE.Scene();

// SpotLight with shadows
const spotLight = new THREE.SpotLight(0xffffff, 50);
spotLight.position.set(0, 8, 4);
spotLight.angle = Math.PI / 6;
spotLight.penumbra = 0.3;
spotLight.decay = 2;
spotLight.distance = 30;
spotLight.castShadow = true;

spotLight.shadow.mapSize.width = 1024;
spotLight.shadow.mapSize.height = 1024;
spotLight.shadow.bias = -0.0002;
spotLight.shadow.camera.near = 1;
spotLight.shadow.camera.far = 30;

scene.add(spotLight);
scene.add(spotLight.target);

// Shadow-casting sphere
const sphere = new THREE.Mesh(
  new THREE.SphereGeometry(0.5, 32, 32),
  new THREE.MeshStandardMaterial({ color: 0xff4444 })
);
sphere.position.set(0, 1, 0);
sphere.castShadow = true;
scene.add(sphere);

// Ground plane receiving shadows
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(15, 15),
  new THREE.MeshStandardMaterial({ color: 0xcccccc })
);
ground.rotation.x = -Math.PI / 2;
ground.receiveShadow = true;
scene.add(ground);
```

---

## Example 3: PointLight Shadow (Cubemap -- Expensive)

PointLight shadows render 6 cubemap faces. Use sparingly.

```js
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

const scene = new THREE.Scene();

// PointLight -- use smaller mapSize to offset 6x cost
const pointLight = new THREE.PointLight(0xffaa33, 50, 20);
pointLight.position.set(0, 3, 0);
pointLight.castShadow = true;

pointLight.shadow.mapSize.width = 512;
pointLight.shadow.mapSize.height = 512;
pointLight.shadow.camera.near = 0.1;
pointLight.shadow.camera.far = 20;
pointLight.shadow.bias = -0.002;

scene.add(pointLight);

// Multiple objects casting shadows in all directions
const positions = [
  [-2, 0.5, -2], [2, 0.5, -2], [-2, 0.5, 2], [2, 0.5, 2]
];

positions.forEach(([x, y, z]) => {
  const box = new THREE.Mesh(
    new THREE.BoxGeometry(0.8, 1, 0.8),
    new THREE.MeshStandardMaterial({ color: 0x4488ff })
  );
  box.position.set(x, y, z);
  box.castShadow = true;
  box.receiveShadow = true;
  scene.add(box);
});

// Room walls and floor receiving shadows
const floorGeo = new THREE.PlaneGeometry(10, 10);
const wallMat = new THREE.MeshStandardMaterial({ color: 0x999999, side: THREE.DoubleSide });

const floor = new THREE.Mesh(floorGeo, wallMat);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

const ceiling = new THREE.Mesh(floorGeo, wallMat);
ceiling.rotation.x = Math.PI / 2;
ceiling.position.y = 6;
ceiling.receiveShadow = true;
scene.add(ceiling);
```

---

## Example 4: Alpha-Tested Shadow (Tree Leaves)

Custom depth material enables correct shadows for alpha-tested geometry.

```js
import * as THREE from 'three';

const loader = new THREE.TextureLoader();
const leafTexture = loader.load('leaf-diffuse.png');
const leafAlpha = loader.load('leaf-alpha.png');

// Visible material with alpha test
const leafMaterial = new THREE.MeshStandardMaterial({
  map: leafTexture,
  alphaMap: leafAlpha,
  alphaTest: 0.5,
  side: THREE.DoubleSide,
});

// Custom depth material -- MUST match alphaTest and alphaMap
const leafDepthMaterial = new THREE.MeshDepthMaterial({
  depthPacking: THREE.RGBADepthPacking,
  map: leafTexture,
  alphaMap: leafAlpha,
  alphaTest: 0.5,
});

const leafMesh = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  leafMaterial
);
leafMesh.castShadow = true;
leafMesh.customDepthMaterial = leafDepthMaterial;
scene.add(leafMesh);
```

---

## Example 5: Static Shadow Optimization

For scenes where lights and casters are stationary, render shadows once and stop.

```js
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// Disable automatic shadow updates
renderer.shadowMap.autoUpdate = false;

const scene = new THREE.Scene();

// ... set up lights, shadow casters, receivers ...

// Render shadows once after scene is fully loaded
renderer.shadowMap.needsUpdate = true;

function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();

// When something moves later, trigger a single shadow update:
function onObjectMoved() {
  renderer.shadowMap.needsUpdate = true;
}
```

---

## Example 6: VSM Soft Shadows with Configurable Blur

VSM shadow maps support `radius` and `blurSamples` for artistic control.

```js
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.VSMShadowMap;

const scene = new THREE.Scene();

const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;

dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;
dirLight.shadow.camera.left = -15;
dirLight.shadow.camera.right = 15;
dirLight.shadow.camera.top = 15;
dirLight.shadow.camera.bottom = -15;

// VSM-specific: blur radius and sample count
dirLight.shadow.radius = 4;        // larger = softer
dirLight.shadow.blurSamples = 16;  // more = smoother blur

scene.add(dirLight);
```

**Warning**: VSM can produce light bleeding artifacts. If shadows leak through thin geometry, switch to `PCFSoftShadowMap`.
