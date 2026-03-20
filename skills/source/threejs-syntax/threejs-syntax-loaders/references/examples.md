# threejs-syntax-loaders — Examples

## Example 1: Load GLTF with Draco Compression and Error Handling

The most common loading pattern: GLTF model with Draco-compressed geometry.

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

// Set up Draco decoder
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
dracoLoader.preload(); // Pre-fetch WASM for faster first load

// Set up GLTF loader
const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

// Load model
try {
  const gltf = await gltfLoader.loadAsync('/models/building.glb');

  // Add to scene
  scene.add(gltf.scene);

  // Access animations if present
  if (gltf.animations.length > 0) {
    const mixer = new THREE.AnimationMixer(gltf.scene);
    gltf.animations.forEach((clip) => {
      mixer.clipAction(clip).play();
    });
  }

  // Access embedded cameras if present
  if (gltf.cameras.length > 0) {
    const embeddedCamera = gltf.cameras[0];
  }
} catch (error) {
  console.error('Failed to load GLTF model:', error);
}

// After all models are loaded, free WASM decoder memory
dracoLoader.dispose();
```

---

## Example 2: LoadingManager with Progress Tracking

Track loading progress across multiple loaders for a loading screen.

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const progressBar = document.getElementById('progress-bar');
const loadingScreen = document.getElementById('loading-screen');

const manager = new THREE.LoadingManager();

manager.onStart = (url, loaded, total) => {
  loadingScreen.style.display = 'flex';
};

manager.onProgress = (url, loaded, total) => {
  const percent = (loaded / total) * 100;
  progressBar.style.width = `${percent}%`;
};

manager.onLoad = () => {
  loadingScreen.style.display = 'none';
};

manager.onError = (url) => {
  console.error(`Failed to load: ${url}`);
};

// All loaders share the same manager
const textureLoader = new THREE.TextureLoader(manager);
const gltfLoader = new GLTFLoader(manager);
const rgbeLoader = new RGBELoader(manager);

// Load assets -- manager tracks all of them
const diffuse = textureLoader.load('/textures/diffuse.jpg');
const normal = textureLoader.load('/textures/normal.jpg');
gltfLoader.load('/models/scene.glb', (gltf) => { scene.add(gltf.scene); });
rgbeLoader.load('/env/studio.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = texture;
});
```

---

## Example 3: HDR Environment Map with RGBELoader

Load an HDR environment map for physically-based lighting and reflections.

```javascript
import * as THREE from 'three';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;

const rgbeLoader = new RGBELoader();
rgbeLoader.load('/environments/sunset.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;

  // Use as environment lighting (affects all PBR materials)
  scene.environment = texture;

  // Optionally use as visible background
  scene.background = texture;
});
```

---

## Example 4: OBJ with MTL Materials

Load a Wavefront OBJ file with its companion MTL material file.

```javascript
import { OBJLoader } from 'three/addons/loaders/OBJLoader.js';
import { MTLLoader } from 'three/addons/loaders/MTLLoader.js';

// ALWAYS load MTL first, then OBJ
const mtlLoader = new MTLLoader();
mtlLoader.setPath('/models/');

try {
  const materials = await mtlLoader.loadAsync('furniture.mtl');
  materials.preload();

  const objLoader = new OBJLoader();
  objLoader.setMaterials(materials);
  objLoader.setPath('/models/');

  const model = await objLoader.loadAsync('furniture.obj');
  scene.add(model);
} catch (error) {
  console.error('Failed to load OBJ/MTL:', error);
}
```

---

## Example 5: Full Loader Setup with KTX2 and Meshopt

Complete production loader setup supporting all compression formats.

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

// Draco decoder for compressed geometry
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');

// KTX2 transcoder for compressed textures
const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('/basis/');
ktx2Loader.detectSupport(renderer); // MUST pass renderer

// Meshopt decoder
await MeshoptDecoder.ready;

// Configure GLTF loader with all decoders
const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);
gltfLoader.setKTX2Loader(ktx2Loader);
gltfLoader.setMeshoptDecoder(MeshoptDecoder);

// Now load any GLTF/GLB -- compressed or uncompressed
try {
  const gltf = await gltfLoader.loadAsync('/models/optimized-scene.glb');
  scene.add(gltf.scene);
} catch (error) {
  console.error('Model loading failed:', error);
}

// Cleanup after all loading is complete
dracoLoader.dispose();
ktx2Loader.dispose();
```

---

## Example 6: Texture Loading with Correct Color Spaces

Load textures with proper color space assignments for PBR materials.

```javascript
import * as THREE from 'three';

const loader = new THREE.TextureLoader();

// Color textures: ALWAYS set SRGBColorSpace
const diffuseMap = await loader.loadAsync('/textures/wood_diffuse.jpg');
diffuseMap.colorSpace = THREE.SRGBColorSpace;

// Data textures: NEVER set SRGBColorSpace -- leave at default (NoColorSpace)
const normalMap = await loader.loadAsync('/textures/wood_normal.jpg');
const roughnessMap = await loader.loadAsync('/textures/wood_roughness.jpg');
const aoMap = await loader.loadAsync('/textures/wood_ao.jpg');

const material = new THREE.MeshStandardMaterial({
  map: diffuseMap,
  normalMap: normalMap,
  roughnessMap: roughnessMap,
  aoMap: aoMap,
});
```

---

## Example 7: Cubemap Skybox

Load a six-image cubemap as scene background.

```javascript
import * as THREE from 'three';

const cubeLoader = new THREE.CubeTextureLoader();
cubeLoader.setPath('/textures/skybox/');

const skybox = cubeLoader.load([
  'px.jpg', 'nx.jpg',  // right, left
  'py.jpg', 'ny.jpg',  // top, bottom
  'pz.jpg', 'nz.jpg'   // front, back
]);

scene.background = skybox;
scene.environment = skybox; // Also use for reflections
```
