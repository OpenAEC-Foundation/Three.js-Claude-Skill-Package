# threejs-impl-lighting — Examples Reference

> Lighting recipes for common scenarios. All examples use ES module imports and Three.js r160+.

---

## Example 1: Outdoor Sunlight Scene

Simulates natural daylight with a sun, sky hemisphere, and HDR environment.

```javascript
import * as THREE from 'three';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

// Renderer with tone mapping for physically correct values
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 0.8;

const scene = new THREE.Scene();

// 1. Hemisphere light for sky/ground ambient
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x362d1b, 0.5);
scene.add(hemi);

// 2. Directional light as sun (high lux for physically correct)
const sun = new THREE.DirectionalLight(0xfff4e6, 3);
sun.position.set(50, 100, 75);
scene.add(sun);
scene.add(sun.target); // REQUIRED — target defaults to (0,0,0)

// 3. HDR environment for reflections on PBR materials
const pmrem = new THREE.PMREMGenerator(renderer);
new RGBELoader().load('outdoor_field.hdr', (hdrTexture) => {
  const envMap = pmrem.fromEquirectangular(hdrTexture);
  scene.environment = envMap.texture;
  scene.background = envMap.texture;
  scene.backgroundBlurriness = 0.05; // Slight blur for depth
  hdrTexture.dispose();
  pmrem.dispose();
});
```

**Key points:**
- HemisphereLight provides gradient ambient (blue sky to brown ground)
- DirectionalLight simulates parallel sunlight rays
- HDR environment handles PBR reflections automatically
- Tone mapping REQUIRED to prevent blown-out whites

---

## Example 2: Indoor Room Lighting

Multiple light sources simulating a residential room with ceiling light and table lamp.

```javascript
import * as THREE from 'three';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;

const scene = new THREE.Scene();

// 1. Low ambient for base illumination (simulates light bounce)
const ambient = new THREE.AmbientLight(0xfff5e6, 0.15);
scene.add(ambient);

// 2. Ceiling light — SpotLight pointing downward
const ceiling = new THREE.SpotLight(0xffeedd, 800);
ceiling.position.set(0, 3.5, 0);
ceiling.angle = Math.PI / 4;
ceiling.penumbra = 0.5;       // Soft cone edges
ceiling.decay = 2;
ceiling.distance = 10;
ceiling.target.position.set(0, 0, 0);
scene.add(ceiling);
scene.add(ceiling.target);

// 3. Table lamp — warm PointLight with limited range
const lamp = new THREE.PointLight(0xffaa44, 400, 5, 2);
lamp.position.set(-2, 1.2, 1);
scene.add(lamp);

// 4. Window light (optional) — DirectionalLight for sunlight through window
const windowLight = new THREE.DirectionalLight(0xfff8f0, 1.5);
windowLight.position.set(-5, 4, -2);
scene.add(windowLight);
scene.add(windowLight.target);
```

**Key points:**
- Low ambient simulates indirect light bounce
- SpotLight for focused overhead fixture with soft penumbra
- PointLight for omnidirectional table lamp with distance falloff
- Warm color temperatures (0xffeedd, 0xffaa44) for cozy interior feel

---

## Example 3: Studio / Product Photography

Three-point lighting setup for showcasing 3D models.

```javascript
import * as THREE from 'three';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.2;

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);

// 1. Key light — main illumination from upper-left
const keyLight = new THREE.DirectionalLight(0xffffff, 4);
keyLight.position.set(-3, 5, 5);
scene.add(keyLight);
scene.add(keyLight.target);

// 2. Fill light — softer, from opposite side to reduce harsh shadows
const fillLight = new THREE.DirectionalLight(0x8888ff, 1.5);
fillLight.position.set(3, 3, 3);
scene.add(fillLight);
scene.add(fillLight.target);

// 3. Rim/back light — highlights edges from behind
const rimLight = new THREE.DirectionalLight(0xffddcc, 2);
rimLight.position.set(0, 3, -5);
scene.add(rimLight);
scene.add(rimLight.target);

// 4. Environment map for PBR reflections (studio HDRI)
const pmrem = new THREE.PMREMGenerator(renderer);
new RGBELoader().load('studio_small.hdr', (hdrTexture) => {
  const envMap = pmrem.fromEquirectangular(hdrTexture);
  scene.environment = envMap.texture;
  scene.environmentIntensity = 0.5; // Subtle IBL, let directional lights dominate
  hdrTexture.dispose();
  pmrem.dispose();
});
```

**Key points:**
- Three-point lighting: key, fill, rim
- Key light is brightest, fill is 30-50% of key intensity
- Rim light creates edge highlights for object separation
- Environment map at reduced intensity for subtle reflections
- Dark background for product showcase contrast

---

## Example 4: Product Showcase with RectAreaLight

Soft panel lighting for e-commerce or product visualization.

```javascript
import * as THREE from 'three';
import { RectAreaLightUniformsLib } from 'three/addons/lights/RectAreaLightUniformsLib.js';
import { RectAreaLightHelper } from 'three/addons/helpers/RectAreaLightHelper.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;

const scene = new THREE.Scene();
scene.background = new THREE.Color(0xf0f0f0);

// MUST init before creating any RectAreaLight
RectAreaLightUniformsLib.init();

// 1. Large overhead softbox
const topPanel = new THREE.RectAreaLight(0xffffff, 8, 6, 6);
topPanel.position.set(0, 5, 0);
topPanel.lookAt(0, 0, 0);
scene.add(topPanel);

// 2. Left fill panel (slightly warm)
const leftPanel = new THREE.RectAreaLight(0xfff5e6, 4, 3, 4);
leftPanel.position.set(-4, 2, 2);
leftPanel.lookAt(0, 0, 0);
scene.add(leftPanel);

// 3. Right accent panel (slightly cool)
const rightPanel = new THREE.RectAreaLight(0xe6f0ff, 3, 2, 3);
rightPanel.position.set(4, 2, -1);
rightPanel.lookAt(0, 0, 0);
scene.add(rightPanel);

// Debug helpers (remove in production)
scene.add(new RectAreaLightHelper(topPanel));
scene.add(new RectAreaLightHelper(leftPanel));
scene.add(new RectAreaLightHelper(rightPanel));
```

**Key points:**
- `RectAreaLightUniformsLib.init()` MUST be called before any RectAreaLight creation
- RectAreaLights produce soft, diffused illumination like photography softboxes
- Orient using `.lookAt()` -- rotation properties are NOT intuitive for area lights
- Works ONLY with MeshStandardMaterial and MeshPhysicalMaterial
- RectAreaLight CANNOT cast shadows

---

## Example 5: Environment-Only Lighting (Minimal Setup)

The simplest high-quality lighting: HDR environment map only, no analytic lights.

```javascript
import * as THREE from 'three';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;

const scene = new THREE.Scene();

const pmrem = new THREE.PMREMGenerator(renderer);
pmrem.compileEquirectangularShader(); // Pre-compile for faster first use

new RGBELoader()
  .setDataType(THREE.HalfFloatType) // Better precision for HDR
  .load('environment.hdr', (hdrTexture) => {
    const envMap = pmrem.fromEquirectangular(hdrTexture);

    scene.environment = envMap.texture;    // Drives all PBR lighting
    scene.background = envMap.texture;     // Visible backdrop
    scene.backgroundBlurriness = 0.0;     // Sharp environment
    scene.environmentIntensity = 1.0;     // Full IBL strength
    scene.environmentRotation.set(0, Math.PI / 4, 0); // Rotate light direction

    hdrTexture.dispose();
    pmrem.dispose();
  });

// All PBR materials will automatically receive:
// - Diffuse irradiance (ambient lighting)
// - Specular reflections (mirror-like highlights)
// - No additional lights needed for many use cases
```

**Key points:**
- Environment map alone provides complete PBR lighting
- `scene.environmentRotation` rotates light direction without extra lights
- `scene.environmentIntensity` controls ambient contribution
- `HalfFloatType` preserves HDR dynamic range
- Pre-compile shader with `pmrem.compileEquirectangularShader()` to avoid first-frame stutter
- ALWAYS dispose both the source texture and PMREMGenerator
