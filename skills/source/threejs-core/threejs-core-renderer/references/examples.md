# threejs-core-renderer — Working Examples

> All examples use ES module imports and are verified against Three.js r160+.
> Every example is complete and runnable.

---

## Example 1: Complete Application Setup

Full initialization with renderer, camera, resize handling, and animation loop.

```javascript
import {
  WebGLRenderer, Scene, PerspectiveCamera, SRGBColorSpace,
  ACESFilmicToneMapping, PCFSoftShadowMap, BoxGeometry,
  MeshStandardMaterial, Mesh, DirectionalLight, AmbientLight
} from 'three';

// Renderer
const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.outputColorSpace = SRGBColorSpace;
renderer.toneMapping = ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// Scene
const scene = new Scene();

// Camera
const camera = new PerspectiveCamera(
  50, window.innerWidth / window.innerHeight, 0.1, 1000
);
camera.position.set(3, 3, 5);
camera.lookAt(0, 0, 0);

// Lighting
const ambientLight = new AmbientLight(0xffffff, 0.4);
scene.add(ambientLight);

const dirLight = new DirectionalLight(0xffffff, 1.5);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);

// Mesh
const geometry = new BoxGeometry(1, 1, 1);
const material = new MeshStandardMaterial({ color: 0x4488ff });
const cube = new Mesh(geometry, material);
cube.castShadow = true;
scene.add(cube);

// Resize handler
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation loop
renderer.setAnimationLoop((time) => {
  cube.rotation.y = time * 0.001;
  renderer.render(scene, camera);
});
```

---

## Example 2: Render-to-Texture with WebGLRenderTarget

Renders a scene to a texture, then uses that texture on a plane in the main scene.

```javascript
import {
  WebGLRenderer, Scene, PerspectiveCamera, WebGLRenderTarget,
  SRGBColorSpace, ACESFilmicToneMapping, BoxGeometry,
  MeshStandardMaterial, Mesh, PlaneGeometry, MeshBasicMaterial,
  AmbientLight, DirectionalLight, LinearFilter
} from 'three';

// Main renderer
const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.outputColorSpace = SRGBColorSpace;
renderer.toneMapping = ACESFilmicToneMapping;
document.body.appendChild(renderer.domElement);

// Render target (off-screen framebuffer)
const renderTarget = new WebGLRenderTarget(512, 512, {
  minFilter: LinearFilter,
  magFilter: LinearFilter,
});

// Off-screen scene (rendered to texture)
const offScene = new Scene();
const offCamera = new PerspectiveCamera(50, 1, 0.1, 100);
offCamera.position.set(0, 0, 3);
const offCube = new Mesh(
  new BoxGeometry(1, 1, 1),
  new MeshStandardMaterial({ color: 0xff4444 })
);
offScene.add(offCube);
offScene.add(new AmbientLight(0xffffff, 0.5));
offScene.add(new DirectionalLight(0xffffff, 1.0));

// Main scene (displays the render target texture)
const mainScene = new Scene();
const mainCamera = new PerspectiveCamera(
  50, window.innerWidth / window.innerHeight, 0.1, 100
);
mainCamera.position.set(0, 0, 4);

const screen = new Mesh(
  new PlaneGeometry(3, 3),
  new MeshBasicMaterial({ map: renderTarget.texture })
);
mainScene.add(screen);

// Resize
window.addEventListener('resize', () => {
  mainCamera.aspect = window.innerWidth / window.innerHeight;
  mainCamera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

// Animation loop
renderer.setAnimationLoop((time) => {
  offCube.rotation.y = time * 0.001;

  // Render to texture
  renderer.setRenderTarget(renderTarget);
  renderer.render(offScene, offCamera);

  // Render main scene to screen
  renderer.setRenderTarget(null);
  renderer.render(mainScene, mainCamera);
});
```

---

## Example 3: Orthographic Camera with Resize

Demonstrates an orthographic setup that maintains consistent world-space units.

```javascript
import {
  WebGLRenderer, Scene, OrthographicCamera, SRGBColorSpace,
  BoxGeometry, MeshNormalMaterial, Mesh, GridHelper
} from 'three';

const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.outputColorSpace = SRGBColorSpace;
document.body.appendChild(renderer.domElement);

const scene = new Scene();

// Orthographic camera showing 10 world units vertically
const frustumSize = 10;
let aspect = window.innerWidth / window.innerHeight;
const camera = new OrthographicCamera(
  -frustumSize * aspect / 2,
   frustumSize * aspect / 2,
   frustumSize / 2,
  -frustumSize / 2,
  0.1, 1000
);
camera.position.set(5, 5, 5);
camera.lookAt(0, 0, 0);

scene.add(new GridHelper(10, 10));
scene.add(new Mesh(new BoxGeometry(1, 1, 1), new MeshNormalMaterial()));

// Resize: MUST update all six frustum values
window.addEventListener('resize', () => {
  aspect = window.innerWidth / window.innerHeight;
  camera.left = -frustumSize * aspect / 2;
  camera.right = frustumSize * aspect / 2;
  camera.top = frustumSize / 2;
  camera.bottom = -frustumSize / 2;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
});
```

---

## Example 4: Split-Screen with Viewport and Scissor

Renders the same scene from two different cameras side by side.

```javascript
import {
  WebGLRenderer, Scene, PerspectiveCamera, SRGBColorSpace,
  ACESFilmicToneMapping, BoxGeometry, MeshNormalMaterial, Mesh
} from 'three';

const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.outputColorSpace = SRGBColorSpace;
renderer.toneMapping = ACESFilmicToneMapping;
renderer.autoClear = false; // MUST disable for multi-pass rendering
document.body.appendChild(renderer.domElement);

const scene = new Scene();
const cube = new Mesh(new BoxGeometry(1, 1, 1), new MeshNormalMaterial());
scene.add(cube);

// Left camera (front view)
const cameraLeft = new PerspectiveCamera(50, 1, 0.1, 100);
cameraLeft.position.set(0, 0, 5);

// Right camera (top-down view)
const cameraRight = new PerspectiveCamera(50, 1, 0.1, 100);
cameraRight.position.set(0, 5, 0);
cameraRight.lookAt(0, 0, 0);

window.addEventListener('resize', () => {
  renderer.setSize(window.innerWidth, window.innerHeight);
  // Aspect ratio is 0.5 because each viewport is half-width
  const halfAspect = (window.innerWidth / 2) / window.innerHeight;
  cameraLeft.aspect = halfAspect;
  cameraLeft.updateProjectionMatrix();
  cameraRight.aspect = halfAspect;
  cameraRight.updateProjectionMatrix();
});
// Trigger initial aspect setup
window.dispatchEvent(new Event('resize'));

renderer.setAnimationLoop((time) => {
  cube.rotation.y = time * 0.001;

  const halfWidth = Math.floor(window.innerWidth / 2);
  const height = window.innerHeight;

  renderer.clear(); // manual clear since autoClear is false

  // Left viewport
  renderer.setViewport(0, 0, halfWidth, height);
  renderer.setScissor(0, 0, halfWidth, height);
  renderer.setScissorTest(true);
  renderer.render(scene, cameraLeft);

  // Right viewport
  renderer.setViewport(halfWidth, 0, halfWidth, height);
  renderer.setScissor(halfWidth, 0, halfWidth, height);
  renderer.render(scene, cameraRight);

  renderer.setScissorTest(false);
});
```

---

## Example 5: Async Shader Compilation with Loading Screen

Pre-compiles shaders to avoid frame drops when the scene first appears.

```javascript
import {
  WebGLRenderer, Scene, PerspectiveCamera, SRGBColorSpace,
  ACESFilmicToneMapping, PCFSoftShadowMap, BoxGeometry,
  MeshStandardMaterial, Mesh, DirectionalLight, AmbientLight
} from 'three';

const renderer = new WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.outputColorSpace = SRGBColorSpace;
renderer.toneMapping = ACESFilmicToneMapping;
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const scene = new Scene();
const camera = new PerspectiveCamera(
  50, window.innerWidth / window.innerHeight, 0.1, 1000
);
camera.position.set(3, 3, 5);
camera.lookAt(0, 0, 0);

// Add objects
scene.add(new AmbientLight(0xffffff, 0.4));
const dirLight = new DirectionalLight(0xffffff, 1.5);
dirLight.position.set(5, 10, 5);
dirLight.castShadow = true;
scene.add(dirLight);

const cube = new Mesh(
  new BoxGeometry(1, 1, 1),
  new MeshStandardMaterial({ color: 0x44aa88, roughness: 0.4, metalness: 0.6 })
);
cube.castShadow = true;
scene.add(cube);

// Show loading indicator
const loadingEl = document.createElement('div');
loadingEl.textContent = 'Compiling shaders...';
loadingEl.style.cssText = 'position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);font:20px sans-serif;';
document.body.appendChild(loadingEl);

// Pre-compile shaders asynchronously, then start rendering
async function init() {
  await renderer.compileAsync(scene, camera);
  loadingEl.remove();

  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });

  renderer.setAnimationLoop((time) => {
    cube.rotation.y = time * 0.001;
    renderer.render(scene, camera);
  });
}

init();
```
