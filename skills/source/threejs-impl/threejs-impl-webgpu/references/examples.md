# Working Code Examples (Three.js WebGPU)

## Example 1: Minimal WebGPU Scene with Fallback

```javascript
import * as THREE from 'three/webgpu';
import { WebGPU } from 'three/webgpu';

async function init() {
  let renderer;

  if (WebGPU.isAvailable()) {
    renderer = new THREE.WebGPURenderer({ antialias: true });
    await renderer.init();
  } else {
    renderer = new THREE.WebGLRenderer({ antialias: true });
  }

  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  document.body.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(
    75, window.innerWidth / window.innerHeight, 0.1, 1000
  );
  camera.position.z = 5;

  const geometry = new THREE.BoxGeometry();
  const material = new THREE.MeshStandardNodeMaterial({ color: 0x00ff00 });
  const cube = new THREE.Mesh(geometry, material);
  scene.add(cube);

  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(5, 5, 5);
  scene.add(light);

  renderer.setAnimationLoop((time) => {
    cube.rotation.x = time * 0.001;
    cube.rotation.y = time * 0.0015;
    renderer.render(scene, camera);
  });
}

init();
```

---

## Example 2: TSL Custom Material with Animated Color

```javascript
import * as THREE from 'three/webgpu';
import { color, oscSine, time, mix, vec4 } from 'three/tsl';

async function init() {
  const renderer = new THREE.WebGPURenderer({ antialias: true });
  await renderer.init();
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.z = 3;

  // Create node material with animated color
  const material = new THREE.MeshStandardNodeMaterial();
  const colorA = color(0xff0000); // red
  const colorB = color(0x0000ff); // blue
  const t = oscSine(time);        // oscillate 0-1 over time
  material.colorNode = vec4(mix(colorA, colorB, t), 1.0);

  const sphere = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 32), material);
  scene.add(sphere);

  const light = new THREE.DirectionalLight(0xffffff, 2);
  light.position.set(3, 3, 3);
  scene.add(light);
  scene.add(new THREE.AmbientLight(0x404040));

  renderer.setAnimationLoop(() => {
    renderer.render(scene, camera);
  });
}

init();
```

---

## Example 3: Vertex Displacement with TSL

```javascript
import * as THREE from 'three/webgpu';
import { positionLocal, normalLocal, sin, time, float, vec3 } from 'three/tsl';

async function init() {
  const renderer = new THREE.WebGPURenderer({ antialias: true });
  await renderer.init();
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.z = 4;

  const material = new THREE.MeshStandardNodeMaterial({ color: 0x44aaff });

  // Displace vertices along their normals using a sine wave
  const displacement = sin(
    positionLocal.y.mul(float(4.0)).add(time.mul(float(2.0)))
  ).mul(float(0.3));

  material.positionNode = positionLocal.add(normalLocal.mul(displacement));

  const geometry = new THREE.SphereGeometry(1, 64, 64);
  const mesh = new THREE.Mesh(geometry, material);
  scene.add(mesh);

  const light = new THREE.DirectionalLight(0xffffff, 2);
  light.position.set(5, 5, 5);
  scene.add(light);
  scene.add(new THREE.AmbientLight(0x222222));

  renderer.setAnimationLoop(() => {
    renderer.render(scene, camera);
  });
}

init();
```

---

## Example 4: Compute Shader — Particle Position Update

```javascript
import * as THREE from 'three/webgpu';
import {
  compute, storage, float, vec3, Fn,
  globalId, sin, cos, time
} from 'three/tsl';

async function init() {
  const renderer = new THREE.WebGPURenderer({ antialias: true });
  await renderer.init();
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 50;

  const particleCount = 10000;

  // Create storage buffer for positions
  const positionAttribute = new THREE.StorageBufferAttribute(
    new Float32Array(particleCount * 3), 3
  );

  // Initialize positions
  for (let i = 0; i < particleCount; i++) {
    positionAttribute.setXYZ(i,
      (Math.random() - 0.5) * 40,
      (Math.random() - 0.5) * 40,
      (Math.random() - 0.5) * 40
    );
  }

  // Compute shader: update positions with circular motion
  const positionStorage = storage(positionAttribute, 'vec3', particleCount);
  const computeFn = Fn(() => {
    const idx = globalId.x;
    const pos = positionStorage.element(idx);
    const angle = time.add(float(idx).mul(float(0.01)));
    pos.x.assign(sin(angle).mul(float(20.0)));
    pos.z.assign(cos(angle).mul(float(20.0)));
  });

  const computeNode = compute(computeFn, particleCount);

  // Create points geometry
  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', positionAttribute);

  const material = new THREE.PointsNodeMaterial({
    size: 0.2,
    sizeAttenuation: true,
    color: 0x00ffaa,
  });

  const points = new THREE.Points(geometry, material);
  scene.add(points);

  renderer.setAnimationLoop(async () => {
    await renderer.computeAsync(computeNode);
    renderer.render(scene, camera);
  });
}

init();
```

---

## Example 5: WebGPU Post-Processing with Bloom

```javascript
import * as THREE from 'three/webgpu';
import { PostProcessing } from 'three/webgpu';
import { pass, bloom, renderOutput } from 'three/tsl';

async function init() {
  const renderer = new THREE.WebGPURenderer({ antialias: true });
  await renderer.init();
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  document.body.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.z = 5;

  // Emissive sphere for bloom effect
  const material = new THREE.MeshStandardNodeMaterial({
    color: 0x000000,
    emissive: 0xff6600,
    emissiveIntensity: 2,
  });

  const sphere = new THREE.Mesh(new THREE.SphereGeometry(1, 32, 32), material);
  scene.add(sphere);
  scene.add(new THREE.AmbientLight(0x111111));

  // Set up post-processing pipeline
  const postProcessing = new PostProcessing(renderer);
  const scenePass = pass(scene, camera);
  const bloomPass = bloom(scenePass, {
    strength: 1.5,
    radius: 0.4,
    threshold: 0.6,
  });
  postProcessing.outputNode = renderOutput(bloomPass);

  renderer.setAnimationLoop((time) => {
    sphere.rotation.y = time * 0.001;
    postProcessing.render();
  });
}

init();
```
