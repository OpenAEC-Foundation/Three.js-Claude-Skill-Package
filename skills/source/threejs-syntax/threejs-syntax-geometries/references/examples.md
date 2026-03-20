# Working Code Examples (Three.js Geometries r160+)

## Example 1: Custom Quad Geometry with Position, Normal, and UV

```javascript
import * as THREE from 'three';

// Create an empty BufferGeometry
const geometry = new THREE.BufferGeometry();

// Step 1: Define 4 vertex positions (3 floats each)
const positions = new Float32Array([
  -1, -1, 0,   // vertex 0 (bottom-left)
   1, -1, 0,   // vertex 1 (bottom-right)
   1,  1, 0,   // vertex 2 (top-right)
  -1,  1, 0    // vertex 3 (top-left)
]);
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

// Step 2: Define indices for two triangles forming a quad
const indices = new Uint16Array([0, 1, 2, 0, 2, 3]);
geometry.setIndex(new THREE.BufferAttribute(indices, 1));

// Step 3: Define UV coordinates (2 floats each)
const uvs = new Float32Array([
  0, 0,   // vertex 0
  1, 0,   // vertex 1
  1, 1,   // vertex 2
  0, 1    // vertex 3
]);
geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));

// Step 4: Compute normals automatically from face topology
geometry.computeVertexNormals();

// Step 5: Compute bounding sphere for frustum culling
geometry.computeBoundingSphere();

// Step 6: Create mesh
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
```

---

## Example 2: InstancedMesh — 1000 Random Cubes

```javascript
import * as THREE from 'three';

const geometry = new THREE.BoxGeometry(0.5, 0.5, 0.5);
const material = new THREE.MeshStandardMaterial({ color: 0xffffff });
const count = 1000;

const instancedMesh = new THREE.InstancedMesh(geometry, material, count);

// Use a dummy Object3D to compose transform matrices
const dummy = new THREE.Object3D();
const color = new THREE.Color();

for (let i = 0; i < count; i++) {
  // Set position
  dummy.position.set(
    (Math.random() - 0.5) * 50,
    (Math.random() - 0.5) * 50,
    (Math.random() - 0.5) * 50
  );

  // Set rotation
  dummy.rotation.set(
    Math.random() * Math.PI,
    Math.random() * Math.PI,
    0
  );

  // Set scale
  dummy.scale.setScalar(0.5 + Math.random() * 1.5);

  // CRITICAL: updateMatrix() computes the local matrix from position/rotation/scale
  dummy.updateMatrix();
  instancedMesh.setMatrixAt(i, dummy.matrix);

  // Set per-instance color
  color.setHSL(Math.random(), 0.8, 0.5);
  instancedMesh.setColorAt(i, color);
}

// CRITICAL: ALWAYS set needsUpdate after writing matrices/colors
instancedMesh.instanceMatrix.needsUpdate = true;
instancedMesh.instanceColor.needsUpdate = true;

scene.add(instancedMesh);
```

---

## Example 3: ExtrudeGeometry — L-Shaped Profile with Hole

```javascript
import * as THREE from 'three';

// Define the L-shaped outline
const shape = new THREE.Shape();
shape.moveTo(0, 0);
shape.lineTo(3, 0);
shape.lineTo(3, 1);
shape.lineTo(1, 1);
shape.lineTo(1, 3);
shape.lineTo(0, 3);
shape.lineTo(0, 0); // Close the shape

// Add a circular hole
const holePath = new THREE.Path();
holePath.absarc(0.5, 2, 0.3, 0, Math.PI * 2, false);
shape.holes.push(holePath);

// Extrude with bevel
const extrudeSettings = {
  depth: 2,
  bevelEnabled: true,
  bevelThickness: 0.1,
  bevelSize: 0.1,
  bevelOffset: 0,
  bevelSegments: 3,
  curveSegments: 12,
  steps: 1
};

const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
const material = new THREE.MeshStandardMaterial({ color: 0x888888 });
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
```

---

## Example 4: Extrude Along a 3D Path (extrudePath)

```javascript
import * as THREE from 'three';

// Define a circular cross-section shape
const circleShape = new THREE.Shape();
circleShape.absarc(0, 0, 0.3, 0, Math.PI * 2, false);

// Define a 3D curve to extrude along
const curve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(-5, 0, 0),
  new THREE.Vector3(-2, 3, 2),
  new THREE.Vector3(2, -3, -2),
  new THREE.Vector3(5, 0, 0)
]);

const extrudeSettings = {
  steps: 100,
  bevelEnabled: false,
  extrudePath: curve
};

const geometry = new THREE.ExtrudeGeometry(circleShape, extrudeSettings);
const material = new THREE.MeshStandardMaterial({ color: 0xff6600, side: THREE.DoubleSide });
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
```

---

## Example 5: Dynamic Geometry — Animated Vertex Positions

```javascript
import * as THREE from 'three';

// Create a plane with enough segments to deform
const geometry = new THREE.PlaneGeometry(10, 10, 64, 64);

// CRITICAL: Set usage hint BEFORE first render for dynamic data
const positionAttr = geometry.getAttribute('position');
positionAttr.usage = THREE.DynamicDrawUsage;

const material = new THREE.MeshStandardMaterial({
  color: 0x0088ff,
  wireframe: true
});
const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);

// Animation loop — wave effect
function animate(time) {
  const positions = geometry.getAttribute('position');
  const t = time * 0.001;

  for (let i = 0; i < positions.count; i++) {
    const x = positions.getX(i);
    const y = positions.getY(i);

    // Compute new Z based on wave function
    const z = Math.sin(x * 0.5 + t) * Math.cos(y * 0.5 + t) * 1.5;
    positions.setZ(i, z);
  }

  // CRITICAL: ALWAYS set needsUpdate after modifying attribute data
  positions.needsUpdate = true;

  // Recompute normals for correct lighting
  geometry.computeVertexNormals();

  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

requestAnimationFrame(animate);
```

---

## Example 6: Multi-Material with Groups

```javascript
import * as THREE from 'three';

const geometry = new THREE.BoxGeometry(2, 2, 2);

// BoxGeometry already has groups defined (one per face pair).
// Clear them and define custom groups:
geometry.clearGroups();

// Each face of a box = 6 indices (2 triangles * 3 vertices)
// Box has 6 faces = 36 total indices
geometry.addGroup(0, 12, 0);   // Front + back faces -> material 0
geometry.addGroup(12, 12, 1);  // Top + bottom faces -> material 1
geometry.addGroup(24, 12, 2);  // Left + right faces -> material 2

const materials = [
  new THREE.MeshStandardMaterial({ color: 0xff0000 }), // Red
  new THREE.MeshStandardMaterial({ color: 0x00ff00 }), // Green
  new THREE.MeshStandardMaterial({ color: 0x0000ff })  // Blue
];

const mesh = new THREE.Mesh(geometry, materials);
scene.add(mesh);
```

---

## Example 7: EdgesGeometry for Architectural Outlines

```javascript
import * as THREE from 'three';

// Create a box
const boxGeometry = new THREE.BoxGeometry(2, 3, 1);
const boxMaterial = new THREE.MeshStandardMaterial({ color: 0xcccccc });
const boxMesh = new THREE.Mesh(boxGeometry, boxMaterial);
scene.add(boxMesh);

// Create clean edges (only edges where angle > 1 degree)
const edgesGeometry = new THREE.EdgesGeometry(boxGeometry, 1);
const edgesMaterial = new THREE.LineBasicMaterial({ color: 0x000000 });
const edgesMesh = new THREE.LineSegments(edgesGeometry, edgesMaterial);
scene.add(edgesMesh);
```
