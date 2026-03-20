# Anti-Patterns (Three.js Math)

## 1. Mutating Shared Vectors

```javascript
// WRONG: Modifying a vector that is referenced by multiple objects
const origin = new Vector3(0, 0, 0);
meshA.position.copy(origin);
meshB.position.copy(origin);

// Later: accidentally mutating the "constant"
origin.set(10, 0, 0); // origin is now (10, 0, 0)
// meshA and meshB positions are NOT affected (they were copied)
// BUT this pattern is dangerous when passing vectors to methods:

const direction = new Vector3(0, 1, 0);
const reflected = direction.reflect(wallNormal); // MUTATES direction!
// direction is now the reflected vector, not (0, 1, 0) anymore

// CORRECT: ALWAYS clone before mutation when the original must be preserved
const reflected = direction.clone().reflect(wallNormal);
```

**WHY**: Nearly all Vector3/Matrix4/Quaternion methods mutate `this` in-place and return `this` for chaining. Forgetting to `clone()` before calling a mutating method silently corrupts shared data.

---

## 2. Lerping Euler Angles Directly

```javascript
// WRONG: Linear interpolation of Euler angles
function animateRotation(mesh, startEuler, endEuler, t) {
  mesh.rotation.x = startEuler.x + (endEuler.x - startEuler.x) * t;
  mesh.rotation.y = startEuler.y + (endEuler.y - startEuler.y) * t;
  mesh.rotation.z = startEuler.z + (endEuler.z - startEuler.z) * t;
}

// CORRECT: Convert to quaternions, slerp, let auto-sync handle the rest
import { Quaternion, Euler } from 'three';

const qStart = new Quaternion().setFromEuler(startEuler);
const qEnd = new Quaternion().setFromEuler(endEuler);

function animateRotation(mesh, t) {
  mesh.quaternion.slerpQuaternions(qStart, qEnd, t);
  // mesh.rotation (Euler) auto-syncs from mesh.quaternion
}
```

**WHY**: Euler angle interpolation does NOT follow the shortest rotational path. It can produce wobbly motion, unexpected detours, and triggers gimbal lock when any axis crosses +/- 90 degrees. Quaternion `slerp()` ALWAYS takes the shortest arc.

---

## 3. Using Degrees Instead of Radians

```javascript
// WRONG: Three.js expects radians, not degrees
mesh.rotation.y = 90; // This rotates ~14.3 full turns, NOT 90 degrees!

// CORRECT: ALWAYS convert degrees to radians
import { MathUtils } from 'three';
mesh.rotation.y = MathUtils.degToRad(90); // 1.5707... radians = 90 degrees

// Also correct: use Math.PI directly
mesh.rotation.y = Math.PI / 2; // 90 degrees
```

**WHY**: All Three.js rotation methods (Euler, Quaternion.setFromAxisAngle, Matrix4.makeRotationX) use radians. Passing degree values produces wildly incorrect rotations with no error message.

---

## 4. Wrong Matrix Multiplication Order

```javascript
// WRONG: Expecting left-to-right application order
const rotate = new Matrix4().makeRotationY(Math.PI / 4);
const translate = new Matrix4().makeTranslation(10, 0, 0);

// This translates FIRST, then rotates (right-to-left!)
const result = rotate.multiply(translate);

// CORRECT: If you want rotate first, then translate:
const result = translate.clone().multiply(rotate);
// Or equivalently:
const result = rotate.clone().premultiply(translate);
```

**WHY**: Three.js uses the mathematical convention where `A.multiply(B)` computes `A * B`, which applies B first, then A. This is the opposite of reading order. Use `premultiply()` when you need left-multiplication.

---

## 5. Manually Setting Quaternion Components

```javascript
// WRONG: Setting quaternion values without understanding quaternion math
mesh.quaternion.set(0, 0.5, 0, 0.5); // What rotation is this? Unclear and likely wrong.
// Also: this is NOT normalized, which produces scaling artifacts

// CORRECT: Use semantic setter methods
mesh.quaternion.setFromAxisAngle(new Vector3(0, 1, 0), Math.PI / 2);
// OR
mesh.quaternion.setFromEuler(new Euler(0, Math.PI / 2, 0));
```

**WHY**: Quaternion components (x, y, z, w) do NOT correspond to rotation axes or angles in any intuitive way. Setting them manually almost always produces incorrect rotations and unnormalized quaternions (which cause mesh distortion).

---

## 6. Forgetting to Update World Matrix Before Reading

```javascript
// WRONG: Reading world position without updating matrices first
mesh.position.set(5, 0, 0);
const worldPos = new Vector3().setFromMatrixPosition(mesh.matrixWorld);
// worldPos may still be (0, 0, 0) if matrices have not been updated!

// CORRECT: ALWAYS call updateMatrixWorld before reading world-space values
mesh.updateMatrixWorld(true); // force update entire hierarchy
const worldPos = new Vector3().setFromMatrixPosition(mesh.matrixWorld);

// Also correct: use getWorldPosition helper
const worldPos = new Vector3();
mesh.getWorldPosition(worldPos);
// getWorldPosition calls updateWorldMatrix internally
```

**WHY**: Three.js defers matrix updates to the render loop for performance. If you read `matrixWorld` outside the render cycle (e.g., after setting position but before render), it contains stale data. Either call `updateMatrixWorld(true)` or use the `getWorld*` helper methods.

---

## 7. Using Color.lerp for Hue Transitions

```javascript
// WRONG: RGB lerp between colors with different hues
const red = new Color(0xff0000);
const blue = new Color(0x0000ff);
const mid = red.clone().lerp(blue, 0.5);
// Result: (0.5, 0, 0.5) = dark purple/muddy magenta

// CORRECT: Use lerpHSL for perceptually smooth hue transitions
const mid = red.clone().lerpHSL(blue, 0.5);
// Result: transitions through the hue wheel (green/cyan region)
```

**WHY**: `lerp()` interpolates R, G, B channels independently, which produces desaturated, muddy intermediate colors when hues differ significantly. `lerpHSL()` interpolates in HSL space, preserving saturation and producing vibrant transitions.

---

## 8. Creating Vectors/Matrices Inside Animation Loops

```javascript
// WRONG: Allocating new objects every frame causes GC pressure
function animate() {
  const direction = new Vector3(0, 0, -1); // NEW object every frame
  direction.applyQuaternion(camera.quaternion);
  player.position.add(direction.multiplyScalar(speed));
  requestAnimationFrame(animate);
}

// CORRECT: Reuse pre-allocated objects
const _direction = new Vector3(); // allocate once, outside the loop

function animate() {
  _direction.set(0, 0, -1);
  _direction.applyQuaternion(camera.quaternion);
  player.position.add(_direction.multiplyScalar(speed));
  requestAnimationFrame(animate);
}
```

**WHY**: Creating temporary Vector3/Matrix4/Quaternion objects in a 60fps loop generates thousands of short-lived objects per second. This triggers frequent garbage collection pauses, causing visible frame drops (jank). ALWAYS pre-allocate reusable temporary objects outside the loop.

---

## 9. Ignoring Gimbal Lock with Euler Rotations

```javascript
// WRONG: Using Euler angles for a flight simulator camera
// When pitch reaches 90 degrees, yaw and roll merge -- gimbal lock!
camera.rotation.order = 'XYZ';
camera.rotation.x += pitchInput;  // pitch
camera.rotation.y += yawInput;    // yaw
camera.rotation.z += rollInput;   // roll
// At rotation.x = Math.PI/2, changing y and z produce the same rotation

// CORRECT: Use quaternion incremental rotation
const pitchQ = new Quaternion().setFromAxisAngle(
  new Vector3(1, 0, 0), pitchInput
);
const yawQ = new Quaternion().setFromAxisAngle(
  new Vector3(0, 1, 0), yawInput
);
const rollQ = new Quaternion().setFromAxisAngle(
  new Vector3(0, 0, 1), rollInput
);

camera.quaternion.multiply(yawQ);
camera.quaternion.multiply(pitchQ);
camera.quaternion.multiply(rollQ);
camera.quaternion.normalize(); // ALWAYS normalize after compound multiplies
```

**WHY**: Euler angles have an inherent singularity (gimbal lock) when the middle axis rotation reaches +/- 90 degrees. For any application requiring free 3D rotation (flight sims, space games, 6DOF controllers), ALWAYS use quaternions with incremental multiplication.

---

## 10. Not Normalizing Quaternions After Repeated Multiplication

```javascript
// WRONG: Compound quaternion operations without normalization
function update() {
  const delta = new Quaternion().setFromAxisAngle(axis, smallAngle);
  mesh.quaternion.multiply(delta);
  // After hundreds of frames, floating-point drift makes the quaternion
  // non-unit, causing mesh scaling/shearing artifacts
}

// CORRECT: Normalize periodically or after compound operations
function update() {
  const delta = new Quaternion().setFromAxisAngle(axis, smallAngle);
  mesh.quaternion.multiply(delta);
  mesh.quaternion.normalize(); // prevents drift accumulation
}
```

**WHY**: Quaternions must be unit-length (length = 1) to represent pure rotations. Floating-point arithmetic introduces tiny errors on each multiply. Over hundreds of frames, these accumulate and the quaternion drifts from unit length, causing visible mesh distortion. ALWAYS normalize after repeated multiplications.
