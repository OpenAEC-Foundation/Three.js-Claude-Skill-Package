# API Signatures Reference (Three.js Raycaster)

## Raycaster

### Constructor

```typescript
new Raycaster(
  origin?: Vector3,      // Ray origin — default: (0, 0, 0)
  direction?: Vector3,   // Ray direction — MUST be normalized — default: (0, 0, -1)
  near?: number,         // Minimum intersection distance — default: 0
  far?: number           // Maximum intersection distance — default: Infinity
): Raycaster
```

### Properties

```typescript
raycaster.ray: Ray              // The underlying Ray (has .origin and .direction)
raycaster.near: number          // Minimum distance — default: 0
raycaster.far: number           // Maximum distance — default: Infinity
raycaster.camera: Camera        // REQUIRED for Sprite raycasting — set manually
raycaster.layers: Layers        // Layer mask for filtering — default: layer 0 enabled
raycaster.params: {             // Per-type intersection thresholds
  Mesh: {},
  Line: { threshold: number },  // default: 1 (world units)
  LOD: {},
  Points: { threshold: number },// default: 1 (world units)
  Sprite: {}
}
```

---

### set(origin: Vector3, direction: Vector3): void

Manually sets the ray origin and direction.

- `origin`: starting point of the ray
- `direction`: MUST be normalized (unit length). Unnormalized direction produces wrong distance values.

```javascript
import * as THREE from 'three';

const raycaster = new THREE.Raycaster();
raycaster.set(
  new THREE.Vector3(0, 10, 0),
  new THREE.Vector3(0, -1, 0)  // pointing down, already normalized
);
```

---

### setFromCamera(coords: Vector2, camera: Camera): void

Configures the ray based on normalized device coordinates and a camera.

- `coords`: `Vector2` with x in [-1, +1] (left to right) and y in [-1, +1] (bottom to top)
- `camera`: `PerspectiveCamera` creates a ray from camera origin through the NDC point; `OrthographicCamera` creates a parallel ray

```javascript
import * as THREE from 'three';

const mouse = new THREE.Vector2();
mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

const raycaster = new THREE.Raycaster();
raycaster.setFromCamera(mouse, camera);
```

**For non-fullscreen canvases:**

```javascript
const rect = renderer.domElement.getBoundingClientRect();
mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
```

---

### intersectObject(object: Object3D, recursive?: boolean, optionalTarget?: Intersection[]): Intersection[]

Tests a single Object3D (and optionally its descendants) for ray intersections.

- `object`: the Object3D to test
- `recursive` (default: `true`): when `true`, also tests all descendants in the scene graph
- `optionalTarget`: pass a reusable array to avoid allocation — the array is NOT cleared automatically, results are appended
- **Returns**: `Intersection[]` sorted by distance (nearest first)

```javascript
import * as THREE from 'three';

const raycaster = new THREE.Raycaster();
const intersections = [];
raycaster.setFromCamera(mouse, camera);

// Test a single mesh and its children
const hits = raycaster.intersectObject(group, true, intersections);
// hits === intersections (same reference)
```

---

### intersectObjects(objects: Object3D[], recursive?: boolean, optionalTarget?: Intersection[]): Intersection[]

Tests an array of Object3D instances for ray intersections.

- `objects`: array of Object3D instances to test
- `recursive` (default: `true`): when `true`, tests descendants of each object
- `optionalTarget`: pass a reusable array to avoid allocation — results are appended, NOT cleared
- **Returns**: `Intersection[]` sorted by distance (nearest first)

```javascript
import * as THREE from 'three';

const raycaster = new THREE.Raycaster();
const intersections = [];
const selectableObjects = [meshA, meshB, meshC];

raycaster.setFromCamera(mouse, camera);
raycaster.intersectObjects(selectableObjects, false, intersections);

if (intersections.length > 0) {
  console.log('Nearest hit:', intersections[0].object.name);
}

intersections.length = 0; // clear for next frame
```

---

## Intersection Object

Each element in the returned intersections array:

```typescript
interface Intersection {
  distance: number;           // Distance from ray origin to hit point (world units)
  point: Vector3;             // Hit point in world space
  face: Face | null;          // Hit face: { a: number, b: number, c: number, normal: Vector3 }
  faceIndex: number;          // Index of the hit face in geometry.index
  object: Object3D;           // Reference to the intersected Object3D
  uv?: Vector2;               // UV coordinates at intersection (requires UV attribute)
  uv1?: Vector2;              // Second UV set at intersection (if geometry has uv1)
  normal?: Vector3;           // Interpolated face normal at intersection point
  instanceId?: number;        // Instance index — ONLY present for InstancedMesh hits
}
```

### Face Object

```typescript
interface Face {
  a: number;         // vertex index A
  b: number;         // vertex index B
  c: number;         // vertex index C
  normal: Vector3;   // face normal (NOT interpolated)
}
```

---

## Layers (used for filtering)

The `raycaster.layers` property is a `Layers` instance controlling which objects the raycaster tests.

```typescript
layers.set(channel: number): void          // Enable ONLY this channel (0-31), disable all others
layers.enable(channel: number): void       // Enable this channel (additive)
layers.enableAll(): void                   // Enable all 32 channels
layers.toggle(channel: number): void       // Toggle a channel on/off
layers.disable(channel: number): void      // Disable a specific channel
layers.disableAll(): void                  // Disable all channels
layers.isEnabled(channel: number): boolean // Check if a channel is enabled
layers.test(other: Layers): boolean        // Bitwise AND test — true if any channel overlaps
```

The raycaster ONLY tests objects where `raycaster.layers.test(object.layers)` returns `true`. By default, both raycaster and objects are on layer 0, so everything is tested.

---

## Raycaster params Detail

The `params` object controls intersection sensitivity for non-mesh types:

```javascript
import * as THREE from 'three';

const raycaster = new THREE.Raycaster();

// Increase line picking sensitivity (default: 1 world unit)
raycaster.params.Line.threshold = 3;

// Increase point cloud picking sensitivity (default: 1 world unit)
raycaster.params.Points.threshold = 0.5;
```

- `Line.threshold`: maximum perpendicular distance from the ray to count as a hit on a Line/LineSegments geometry
- `Points.threshold`: maximum distance from the ray to count as a hit on a Points geometry
- Mesh objects use exact triangle intersection — no threshold needed
- Sprite objects use screen-space bounds — `raycaster.camera` MUST be set
