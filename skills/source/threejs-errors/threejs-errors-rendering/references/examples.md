# threejs-errors-rendering — Examples

## Example 1: Complete Black Screen Diagnosis

```javascript
import * as THREE from 'three';

// Step 1: Create renderer with correct size
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.body.appendChild(renderer.domElement); // ALWAYS add to DOM

// Step 2: Create scene with light
const scene = new THREE.Scene();
scene.add(new THREE.AmbientLight(0xffffff, 0.4));
const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
dirLight.position.set(5, 10, 7);
scene.add(dirLight);

// Step 3: Camera positioned away from origin
const camera = new THREE.PerspectiveCamera(
  75, window.innerWidth / window.innerHeight, 0.1, 1000
);
camera.position.set(0, 2, 5);
camera.lookAt(0, 0, 0);

// Step 4: Add visible object
const mesh = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x00ff00 })
);
scene.add(mesh);

// Step 5: Render loop
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

## Example 2: Color Space Correct Setup

```javascript
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer();
renderer.outputColorSpace = THREE.SRGBColorSpace; // default in r160+
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;

const textureLoader = new THREE.TextureLoader();

// Color texture: ALWAYS SRGBColorSpace
const diffuseMap = textureLoader.load('albedo.png');
diffuseMap.colorSpace = THREE.SRGBColorSpace;

// Normal map: NEVER SRGBColorSpace
const normalMap = textureLoader.load('normal.png');
// normalMap.colorSpace remains LinearSRGBColorSpace (default)

// Roughness map: NEVER SRGBColorSpace
const roughnessMap = textureLoader.load('roughness.png');
// roughnessMap.colorSpace remains LinearSRGBColorSpace (default)

const material = new THREE.MeshStandardMaterial({
  map: diffuseMap,
  normalMap: normalMap,
  roughnessMap: roughnessMap,
  roughness: 1.0,
  metalness: 0.0
});
```

## Example 3: Z-Fighting Fix with Polygon Offset

```javascript
import * as THREE from 'three';

// Ground plane
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(10, 10),
  new THREE.MeshStandardMaterial({ color: 0x808080 })
);
ground.rotation.x = -Math.PI / 2;

// Decal on ground -- uses polygon offset to avoid z-fighting
const decal = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  new THREE.MeshStandardMaterial({
    color: 0xff0000,
    polygonOffset: true,
    polygonOffsetFactor: -1,
    polygonOffsetUnits: -1
  })
);
decal.rotation.x = -Math.PI / 2;
decal.position.y = 0.001; // tiny offset as additional safety

scene.add(ground);
scene.add(decal);
```

## Example 4: Transparent Object Ordering

```javascript
import * as THREE from 'three';

// Background opaque object
const wall = new THREE.Mesh(
  new THREE.PlaneGeometry(5, 5),
  new THREE.MeshStandardMaterial({ color: 0x444444 })
);
wall.position.z = -2;
wall.renderOrder = 0;
scene.add(wall);

// Transparent glass panel
const glass = new THREE.Mesh(
  new THREE.PlaneGeometry(3, 3),
  new THREE.MeshStandardMaterial({
    color: 0x88ccff,
    transparent: true,
    opacity: 0.3,
    depthWrite: false, // prevents depth conflicts with other transparent objects
    side: THREE.DoubleSide
  })
);
glass.renderOrder = 1;
scene.add(glass);

// Front transparent panel
const frontGlass = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 2),
  new THREE.MeshStandardMaterial({
    color: 0xff8888,
    transparent: true,
    opacity: 0.5,
    depthWrite: false,
    side: THREE.DoubleSide
  })
);
frontGlass.position.z = 1;
frontGlass.renderOrder = 2;
scene.add(frontGlass);
```

## Example 5: WebGL Context Loss Recovery

```javascript
import * as THREE from 'three';

let renderer, scene, camera, animationId;

function initScene() {
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.set(0, 2, 5);

  scene.add(new THREE.AmbientLight(0xffffff, 0.5));
  scene.add(new THREE.Mesh(
    new THREE.BoxGeometry(),
    new THREE.MeshStandardMaterial({ color: 0x00ff00 })
  ));
}

function init() {
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  // ALWAYS add context loss handlers
  renderer.domElement.addEventListener('webglcontextlost', (event) => {
    event.preventDefault();
    cancelAnimationFrame(animationId);
    console.warn('WebGL context lost. Waiting for restoration...');
  }, false);

  renderer.domElement.addEventListener('webglcontextrestored', () => {
    console.log('WebGL context restored. Re-initializing...');
    initScene();
    animate();
  }, false);

  initScene();
  animate();
}

function animate() {
  animationId = requestAnimationFrame(animate);
  renderer.render(scene, camera);
}

init();
```

## Example 6: Resize Handler with updateProjectionMatrix

```javascript
import * as THREE from 'three';

const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);

window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix(); // NEVER forget this after changing camera properties
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
});
```

## Example 7: Foliage with alphaTest (Not Transparency)

```javascript
import * as THREE from 'three';

const textureLoader = new THREE.TextureLoader();
const leafTexture = textureLoader.load('leaf.png');
leafTexture.colorSpace = THREE.SRGBColorSpace;

// Use alphaTest for hard-edge alpha, NOT transparent: true
const leafMaterial = new THREE.MeshStandardMaterial({
  map: leafTexture,
  alphaTest: 0.5,
  side: THREE.DoubleSide
  // transparent: true is NOT needed and would cause sorting issues
});

const leaf = new THREE.Mesh(new THREE.PlaneGeometry(1, 1), leafMaterial);
scene.add(leaf);
```

## Example 8: Debugging Invisible Object

```javascript
// Diagnostic function to find why an object is invisible
function diagnoseInvisible(object, camera) {
  // Check visibility chain
  let node = object;
  while (node) {
    if (!node.visible) {
      console.error(`Hidden: ${node.name || node.type} has visible=false`);
      return;
    }
    node = node.parent;
  }

  // Check layers
  if (!camera.layers.test(object.layers)) {
    console.error('Layer mismatch: camera and object share no layers');
    return;
  }

  // Check material
  if (object.material) {
    if (!object.material.visible) {
      console.error('Material.visible is false');
      return;
    }
    if (object.material.opacity === 0 && object.material.transparent) {
      console.error('Material is fully transparent (opacity=0)');
      return;
    }
  }

  // Check scale
  const s = object.scale;
  if (s.x === 0 || s.y === 0 || s.z === 0) {
    console.error(`Scale is zero: (${s.x}, ${s.y}, ${s.z})`);
    return;
  }

  // Check position relative to camera
  const distance = camera.position.distanceTo(object.position);
  if (distance < camera.near || distance > camera.far) {
    console.error(`Object at distance ${distance}, camera near=${camera.near} far=${camera.far}`);
    return;
  }

  console.log('No obvious issue found. Check geometry and frustum culling.');
}
```
