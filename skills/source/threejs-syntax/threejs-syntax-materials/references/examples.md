# Working Code Examples (Three.js r160+ Materials)

## Example 1: PBR Material with Texture Maps

Complete setup of a physically-based material with diffuse, normal, roughness, metalness, and AO maps.

```javascript
import * as THREE from 'three';

const loader = new THREE.TextureLoader();

// Load all texture maps
const diffuseMap = loader.load('textures/brick_diffuse.jpg');
const normalMap = loader.load('textures/brick_normal.jpg');
const roughnessMap = loader.load('textures/brick_roughness.jpg');
const aoMap = loader.load('textures/brick_ao.jpg');

// CRITICAL: Set correct color spaces
diffuseMap.colorSpace = THREE.SRGBColorSpace;  // Diffuse = SRGB
// normalMap, roughnessMap, aoMap stay at NoColorSpace (default) -- NEVER change these

// Enable tiling
diffuseMap.wrapS = diffuseMap.wrapT = THREE.RepeatWrapping;
normalMap.wrapS = normalMap.wrapT = THREE.RepeatWrapping;
roughnessMap.wrapS = roughnessMap.wrapT = THREE.RepeatWrapping;
aoMap.wrapS = aoMap.wrapT = THREE.RepeatWrapping;

diffuseMap.repeat.set(2, 2);
normalMap.repeat.set(2, 2);
roughnessMap.repeat.set(2, 2);
aoMap.repeat.set(2, 2);

const material = new THREE.MeshStandardMaterial({
  map: diffuseMap,
  normalMap: normalMap,
  normalScale: new THREE.Vector2(1.0, 1.0),
  roughnessMap: roughnessMap,
  roughness: 1.0,         // Let roughnessMap control per-pixel
  metalnessMap: null,
  metalness: 0.0,         // Brick is dielectric
  aoMap: aoMap,
  aoMapIntensity: 1.0
});

// Geometry MUST have uv2 attribute for aoMap
const geometry = new THREE.BoxGeometry(2, 2, 2);
geometry.setAttribute('uv2', geometry.getAttribute('uv'));

const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
```

---

## Example 2: Glass Material with MeshPhysicalMaterial

Physically-based glass using transmission and thickness.

```javascript
import * as THREE from 'three';

const glassMaterial = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  metalness: 0.0,
  roughness: 0.0,
  transmission: 1.0,        // Fully transparent (physically-based)
  thickness: 0.5,           // Volume thickness for refraction
  ior: 1.5,                 // Glass index of refraction
  specularIntensity: 1.0,
  specularColor: 0xffffff,
  envMapIntensity: 1.0,
  transparent: true,        // Required for transmission to render correctly
  side: THREE.DoubleSide    // See both faces of glass
});

const glassPane = new THREE.Mesh(
  new THREE.PlaneGeometry(2, 3),
  glassMaterial
);
scene.add(glassPane);
```

**Note**: Transmission requires the renderer to capture the scene behind the object. This adds a render pass and has a GPU cost. NEVER use transmission on many small objects -- use simple `transparent: true` with `opacity` instead for non-physical transparency.

---

## Example 3: HDR Environment Map for PBR

Load an HDR environment map and use it for both lighting and reflections.

```javascript
import * as THREE from 'three';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;

const rgbeLoader = new RGBELoader();
rgbeLoader.load('environment.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = texture;   // PBR environment lighting for ALL materials
  scene.background = texture;    // Optional: visible HDR background
});

// Materials automatically pick up scene.environment -- no need to set envMap per material
const chrome = new THREE.MeshStandardMaterial({
  color: 0xffffff,
  metalness: 1.0,
  roughness: 0.0    // Mirror-like chrome
});

const plastic = new THREE.MeshStandardMaterial({
  color: 0xff4444,
  metalness: 0.0,
  roughness: 0.4    // Slightly glossy plastic
});
```

---

## Example 4: Multi-Material on a Single Mesh

Apply different materials to different faces of a geometry using material groups.

```javascript
import * as THREE from 'three';

// BoxGeometry has 6 groups (one per face), each with materialIndex 0-5
const geometry = new THREE.BoxGeometry(1, 1, 1);

const materials = [
  new THREE.MeshStandardMaterial({ color: 0xff0000 }), // +X face (right)
  new THREE.MeshStandardMaterial({ color: 0x00ff00 }), // -X face (left)
  new THREE.MeshStandardMaterial({ color: 0x0000ff }), // +Y face (top)
  new THREE.MeshStandardMaterial({ color: 0xffff00 }), // -Y face (bottom)
  new THREE.MeshStandardMaterial({ color: 0xff00ff }), // +Z face (front)
  new THREE.MeshStandardMaterial({ color: 0x00ffff })  // -Z face (back)
];

const cube = new THREE.Mesh(geometry, materials);
scene.add(cube);

// For custom geometry, define groups manually:
// geometry.addGroup(startIndex, count, materialIndex);
```

---

## Example 5: Proper Material Disposal

Correct cleanup pattern to prevent GPU memory leaks.

```javascript
import * as THREE from 'three';

function disposeObject(object) {
  // Traverse the entire hierarchy
  object.traverse((child) => {
    if (child.isMesh) {
      // Dispose geometry
      if (child.geometry) {
        child.geometry.dispose();
      }

      // Dispose material(s)
      if (Array.isArray(child.material)) {
        child.material.forEach((mat) => disposeMaterial(mat));
      } else if (child.material) {
        disposeMaterial(child.material);
      }
    }
  });

  // Remove from parent
  if (object.parent) {
    object.parent.remove(object);
  }
}

function disposeMaterial(material) {
  // Dispose all texture properties
  for (const key of Object.keys(material)) {
    const value = material[key];
    if (value && value.isTexture) {
      value.dispose();
    }
  }
  material.dispose();
}

// Usage: ALWAYS call when removing objects from scene
disposeObject(myModel);
```

---

## Example 6: Texture Tiling and Anisotropic Filtering

Floor texture with proper repeat, wrapping, and anisotropic filtering for oblique viewing angles.

```javascript
import * as THREE from 'three';

const loader = new THREE.TextureLoader();
const floorTexture = loader.load('textures/wood_floor.jpg');

// Color space for diffuse textures
floorTexture.colorSpace = THREE.SRGBColorSpace;

// MUST set wrapping mode BEFORE setting repeat
floorTexture.wrapS = THREE.RepeatWrapping;
floorTexture.wrapT = THREE.RepeatWrapping;
floorTexture.repeat.set(10, 10);

// Anisotropic filtering -- essential for floor/ground textures
floorTexture.anisotropy = renderer.capabilities.getMaxAnisotropy();

const floorMaterial = new THREE.MeshStandardMaterial({
  map: floorTexture,
  roughness: 0.8,
  metalness: 0.0
});

const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(50, 50),
  floorMaterial
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);
```
