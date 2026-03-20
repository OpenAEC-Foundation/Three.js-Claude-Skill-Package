# Anti-Patterns (Three.js Raycaster)

## 1. Raycasting on Every mousemove

```javascript
// WRONG: Raycasts on every single mousemove event (can fire 100+ times/second)
canvas.addEventListener('mousemove', (event) => {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(objects, true); // runs hundreds of times/sec
  // ...
});

// CORRECT: Update coordinates in the event, raycast in the render loop
let needsRaycast = false;
canvas.addEventListener('pointermove', (event) => {
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
  needsRaycast = true;
});

function animate() {
  if (needsRaycast) {
    raycaster.setFromCamera(mouse, camera);
    raycaster.intersectObjects(objects, false, intersections);
    // process...
    intersections.length = 0;
    needsRaycast = false;
  }
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

**WHY**: `mousemove` fires at the browser's input rate (often 100-240Hz). Raycasting is expensive on complex scenes. Deferring to `requestAnimationFrame` limits it to the display refresh rate.

---

## 2. Raycasting Against the Entire Scene with recursive: true

```javascript
// WRONG: Traverses every object in the scene graph including lights, helpers, etc.
const hits = raycaster.intersectObjects(scene.children, true);

// CORRECT: Maintain a flat array of only the objects you want to pick
const selectableObjects = [meshA, meshB, meshC];
const hits = raycaster.intersectObjects(selectableObjects, false);
```

**WHY**: `recursive: true` on `scene.children` traverses lights, cameras, helpers, groups, and every nested child. This wastes CPU on objects that cannot be meaningfully selected. A flat array with `recursive: false` tests only the exact objects you need.

---

## 3. Allocating a New Array Every Frame

```javascript
// WRONG: Creates a new array every frame, generating garbage for the GC
function animate() {
  raycaster.setFromCamera(mouse, camera);
  const hits = raycaster.intersectObjects(objects); // new array each call
  // ...
  requestAnimationFrame(animate);
}

// CORRECT: Reuse a single array via optionalTarget
const intersections = [];
function animate() {
  raycaster.setFromCamera(mouse, camera);
  raycaster.intersectObjects(objects, false, intersections);
  // process intersections...
  intersections.length = 0; // clear for next frame
  requestAnimationFrame(animate);
}
```

**WHY**: Every call without `optionalTarget` allocates a new array. In animation loops (60fps), this creates 60 arrays per second that the garbage collector must clean up, causing micro-stutters.

---

## 4. Using window.innerWidth on Non-Fullscreen Canvas

```javascript
// WRONG: Assumes canvas fills the entire browser window
mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

// CORRECT: Use the canvas bounding rect for accurate NDC conversion
const rect = renderer.domElement.getBoundingClientRect();
mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
```

**WHY**: When the canvas does not fill the entire viewport (e.g., sidebar layouts, embedded viewers), `window.innerWidth/Height` produces incorrect NDC values. The ray will miss the intended target by the offset of the canvas within the page.

---

## 5. Forgetting to Normalize the Direction Vector

```javascript
// WRONG: Direction is not normalized — distances in results will be wrong
const direction = new THREE.Vector3(1, -2, 0.5); // length != 1
raycaster.set(origin, direction);

// CORRECT: ALWAYS normalize the direction
const direction = new THREE.Vector3(1, -2, 0.5).normalize();
raycaster.set(origin, direction);
```

**WHY**: The `distance` property in intersection results assumes a normalized direction vector. An unnormalized direction produces scaled distances that do not correspond to world units, breaking sorting and distance-based logic.

---

## 6. Missing instanceId Check for InstancedMesh

```javascript
// WRONG: Assumes instanceId is always present
const hits = raycaster.intersectObjects(mixedObjects);
if (hits.length > 0) {
  const id = hits[0].instanceId; // undefined if hit object is a regular Mesh!
  instancedMesh.setColorAt(id, color); // crash or NaN behavior
}

// CORRECT: Check instanceId before using it
const hits = raycaster.intersectObjects(mixedObjects, false, intersections);
if (intersections.length > 0) {
  const hit = intersections[0];
  if (hit.instanceId !== undefined) {
    // InstancedMesh hit
    instancedMesh.setColorAt(hit.instanceId, color);
    instancedMesh.instanceColor.needsUpdate = true;
  } else {
    // Regular mesh hit
    hit.object.material.color.copy(color);
  }
}
intersections.length = 0;
```

**WHY**: `instanceId` is ONLY present on intersections with `InstancedMesh`. Accessing it on a regular `Mesh` intersection returns `undefined`, which causes silent failures or NaN errors when passed to `setColorAt`.

---

## 7. Not Setting raycaster.camera for Sprite Picking

```javascript
// WRONG: Raycasting against Sprites without setting raycaster.camera
const raycaster = new THREE.Raycaster();
raycaster.setFromCamera(mouse, camera);
const hits = raycaster.intersectObjects(sprites); // incorrect results

// CORRECT: Set raycaster.camera before intersecting Sprites
const raycaster = new THREE.Raycaster();
raycaster.camera = camera; // required for Sprite screen-space bounds
raycaster.setFromCamera(mouse, camera);
const hits = raycaster.intersectObjects(sprites);
```

**WHY**: Sprites are always camera-facing billboards. The raycaster needs the camera reference to compute the sprite's screen-space bounding box. Without it, sprite intersection tests produce incorrect or no results.

---

## 8. Not Clearing optionalTarget Array

```javascript
// WRONG: Results accumulate across frames because optionalTarget is never cleared
const intersections = [];
function animate() {
  raycaster.setFromCamera(mouse, camera);
  raycaster.intersectObjects(objects, false, intersections);
  // intersections grows every frame! Contains stale results
  if (intersections.length > 0) {
    handleHit(intersections[0]); // may be a stale result from a previous frame
  }
  requestAnimationFrame(animate);
}

// CORRECT: Clear the array after processing
const intersections = [];
function animate() {
  raycaster.setFromCamera(mouse, camera);
  raycaster.intersectObjects(objects, false, intersections);
  if (intersections.length > 0) {
    handleHit(intersections[0]);
  }
  intersections.length = 0; // ALWAYS clear after processing
  requestAnimationFrame(animate);
}
```

**WHY**: The `optionalTarget` array is NOT cleared automatically by `intersectObjects` — results are appended. Without clearing, the array grows unbounded and contains stale intersection data from previous frames.

---

## 9. Using offsetX/offsetY Instead of clientX/clientY

```javascript
// WRONG: offsetX/offsetY is relative to the event target, but breaks with CSS transforms
mouse.x = (event.offsetX / canvas.width) * 2 - 1;
mouse.y = -(event.offsetY / canvas.height) * 2 + 1;

// CORRECT: Use clientX/clientY with getBoundingClientRect
const rect = renderer.domElement.getBoundingClientRect();
mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
```

**WHY**: `offsetX/offsetY` are relative to the event target element, but they do not account for CSS transforms, scaling, or devicePixelRatio differences between the canvas element size and its CSS layout size. `clientX/clientY` with `getBoundingClientRect()` is the reliable approach.

---

## 10. Forgetting needsUpdate After Changing InstancedMesh Colors

```javascript
// WRONG: Color change is not visible because GPU buffer is not updated
instancedMesh.setColorAt(id, new THREE.Color(0xff0000));
// missing: instancedMesh.instanceColor.needsUpdate = true;

// CORRECT: ALWAYS set needsUpdate after modifying instance attributes
instancedMesh.setColorAt(id, new THREE.Color(0xff0000));
instancedMesh.instanceColor.needsUpdate = true;
```

**WHY**: `setColorAt` modifies the CPU-side buffer. The GPU buffer is not updated until `needsUpdate` is set to `true`. Without it, the color change is invisible until another operation triggers a buffer upload.
