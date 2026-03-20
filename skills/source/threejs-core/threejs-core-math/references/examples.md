# Working Code Examples (Three.js Math)

## Example 1: Vector3 Operations -- Position, Distance, and Direction

```javascript
import { Vector3 } from 'three';

// Create positions
const playerPos = new Vector3(10, 0, 5);
const enemyPos = new Vector3(20, 0, 15);

// Calculate distance between two points
const distance = playerPos.distanceTo(enemyPos);
console.log(distance); // ~14.14

// Calculate direction from player to enemy (normalized)
const direction = new Vector3().subVectors(enemyPos, playerPos).normalize();
console.log(direction); // approximately (0.707, 0, 0.707)

// Move player toward enemy by 2 units
const moveSpeed = 2;
const movement = direction.clone().multiplyScalar(moveSpeed);
playerPos.add(movement);
// playerPos is now approximately (11.41, 0, 6.41)

// Project a point onto a vector (closest point on a line)
const lineDir = new Vector3(1, 0, 0); // X axis
const point = new Vector3(3, 4, 0);
const projected = point.clone().projectOnVector(lineDir);
console.log(projected); // (3, 0, 0)

// Reflect a velocity vector off a wall
const velocity = new Vector3(1, 0, -1);
const wallNormal = new Vector3(0, 0, 1);
const reflected = velocity.clone().reflect(wallNormal);
console.log(reflected); // (1, 0, 1)

// Linear interpolation between two positions (50% blend)
const midpoint = new Vector3().lerpVectors(playerPos, enemyPos, 0.5);

// Convert to/from arrays (useful for BufferGeometry)
const arr = playerPos.toArray(); // [x, y, z]
const restored = new Vector3().fromArray(arr);
```

---

## Example 2: Matrix4 -- Compose, Decompose, and Custom Transforms

```javascript
import { Vector3, Quaternion, Matrix4, MathUtils } from 'three';

// Build a transformation matrix from position, rotation, scale
const position = new Vector3(5, 10, 0);
const quaternion = new Quaternion().setFromAxisAngle(
  new Vector3(0, 1, 0),           // Y axis
  MathUtils.degToRad(45)          // 45 degrees
);
const scale = new Vector3(2, 2, 2);

const matrix = new Matrix4();
matrix.compose(position, quaternion, scale);

// Decompose back to components
const outPos = new Vector3();
const outQuat = new Quaternion();
const outScale = new Vector3();
matrix.decompose(outPos, outQuat, outScale);
// outPos = (5, 10, 0), outScale = (2, 2, 2)

// Chain transformations: rotate then translate
// Remember: right-to-left order. multiply(M2) means M2 first, then this.
const rotMatrix = new Matrix4().makeRotationY(MathUtils.degToRad(90));
const transMatrix = new Matrix4().makeTranslation(10, 0, 0);
const combined = transMatrix.clone().multiply(rotMatrix);
// Result: first rotates 90 deg around Y, then translates 10 on X

// Transform a point with a matrix
const point = new Vector3(1, 0, 0);
point.applyMatrix4(combined);
// Point is rotated then translated

// Extract translation from a model's world matrix
const worldPos = new Vector3().setFromMatrixPosition(mesh.matrixWorld);

// Invert a matrix (useful for world-to-local conversion)
const inverseWorld = mesh.matrixWorld.clone().invert();
const localPoint = worldPos.clone().applyMatrix4(inverseWorld);
```

---

## Example 3: Quaternion Rotation -- Slerp Animation and Compound Rotations

```javascript
import { Quaternion, Vector3, MathUtils } from 'three';

// Create rotation from axis + angle
const q1 = new Quaternion().setFromAxisAngle(
  new Vector3(0, 1, 0),          // Y axis
  MathUtils.degToRad(0)          // Starting rotation
);

const q2 = new Quaternion().setFromAxisAngle(
  new Vector3(0, 1, 0),          // Y axis
  MathUtils.degToRad(180)        // Target rotation
);

// Smooth rotation interpolation in animation loop
function animate() {
  const t = (Math.sin(Date.now() * 0.001) + 1) / 2; // oscillate 0-1
  mesh.quaternion.slerpQuaternions(q1, q2, t);
}

// Combine two rotations (order matters!)
const pitchUp = new Quaternion().setFromAxisAngle(
  new Vector3(1, 0, 0),          // X axis
  MathUtils.degToRad(-30)        // pitch up 30 degrees
);
const yawRight = new Quaternion().setFromAxisAngle(
  new Vector3(0, 1, 0),          // Y axis
  MathUtils.degToRad(45)         // yaw right 45 degrees
);

// Apply yaw first, then pitch: result = pitch * yaw
const combined = pitchUp.clone().multiply(yawRight);
mesh.quaternion.copy(combined);

// Find rotation that maps one direction to another
const fromDir = new Vector3(0, 0, 1);  // forward
const toDir = new Vector3(1, 0, 0);    // right
const rotation = new Quaternion().setFromUnitVectors(fromDir, toDir);
// rotation now represents a 90-degree Y rotation

// Gradually rotate toward a target (useful for turrets, cameras)
const maxStepRadians = MathUtils.degToRad(2); // 2 degrees per frame
mesh.quaternion.rotateTowards(targetQuaternion, maxStepRadians);
```

---

## Example 4: Color Operations and Color Space Management

```javascript
import { Color, MeshStandardMaterial } from 'three';

// Various constructor forms
const red = new Color(0xff0000);
const green = new Color('green');
const blue = new Color(0, 0, 1);
const coral = new Color('#ff7f50');

// HSL-based color creation
const hslColor = new Color();
hslColor.setHSL(0.6, 1.0, 0.5); // bright blue via HSL

// Smooth color transition through the hue wheel
const startColor = new Color(0xff0000); // red
const endColor = new Color(0x0000ff);   // blue

// WRONG way: lerp produces muddy brown in the middle
// const muddy = startColor.clone().lerp(endColor, 0.5);

// CORRECT way: lerpHSL goes through the hue wheel
const vibrant = startColor.clone().lerpHSL(endColor, 0.5);
// Result: goes through green/cyan on the way from red to blue

// Read HSL values
const hsl = {};
red.getHSL(hsl);
console.log(hsl); // { h: 0, s: 1, l: 0.5 }

// Color space management for physically correct rendering
const textureColor = new Color(0x808080);
textureColor.convertSRGBToLinear(); // ALWAYS do this for manual color input
                                     // when renderer.outputColorSpace = SRGBColorSpace

// Apply to material
const material = new MeshStandardMaterial({
  color: new Color(0x44aa88),
});

// Dynamically change color
material.color.setHex(0xff4444);
material.color.multiplyScalar(0.5); // darken by 50%
```

---

## Example 5: Bounding Volumes -- Collision Detection and Frustum Culling

```javascript
import {
  Box3, Sphere, Vector3, Frustum, Matrix4,
  PerspectiveCamera, Mesh, BoxGeometry, MeshBasicMaterial
} from 'three';

// Compute bounding box from a mesh
const mesh = new Mesh(
  new BoxGeometry(2, 3, 4),
  new MeshBasicMaterial()
);
mesh.position.set(5, 0, 0);
mesh.updateMatrixWorld(true); // ALWAYS update before computing bounds

const box = new Box3().setFromObject(mesh);
// box.min ~ (4, -1.5, -2), box.max ~ (6, 1.5, 2)

// Get center and size
const center = new Vector3();
const size = new Vector3();
box.getCenter(center);  // (5, 0, 0)
box.getSize(size);      // (2, 3, 4)

// Point containment test
const testPoint = new Vector3(5, 0, 0);
console.log(box.containsPoint(testPoint)); // true

// Box-box intersection (AABB collision detection)
const otherBox = new Box3(
  new Vector3(3, -1, -1),
  new Vector3(5, 1, 1)
);
console.log(box.intersectsBox(otherBox)); // true

// Bounding sphere from box
const sphere = new Sphere();
box.getBoundingSphere(sphere);

// Frustum culling: check if object is visible to camera
const camera = new PerspectiveCamera(75, 16/9, 0.1, 1000);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0);
camera.updateMatrixWorld(true);

const frustum = new Frustum();
const projScreenMatrix = new Matrix4();
projScreenMatrix.multiplyMatrices(
  camera.projectionMatrix,
  camera.matrixWorldInverse
);
frustum.setFromProjectionMatrix(projScreenMatrix);

// Check if a point or box is within the camera view
console.log(frustum.containsPoint(new Vector3(0, 0, 0))); // true if visible
console.log(frustum.intersectsBox(box)); // true if any part is visible
```
