# threejs-syntax-controls -- Anti-Patterns

## Anti-Pattern 1: Forgetting controls.update() with Damping

**WRONG:**
```javascript
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

function animate() {
  requestAnimationFrame(animate);
  // Missing controls.update() !
  renderer.render(scene, camera);
}
```

**What happens:** The camera moves while the user drags, but snaps to a stop the instant the user releases the mouse. The damping/inertia effect NEVER activates. With `autoRotate`, the camera NEVER rotates.

**CORRECT:**
```javascript
function animate() {
  requestAnimationFrame(animate);
  controls.update(); // ALWAYS call when enableDamping or autoRotate is true
  renderer.render(scene, camera);
}
```

---

## Anti-Pattern 2: Not Calling dispose()

**WRONG:**
```javascript
function switchToFlyMode() {
  // Just replace controls without cleanup
  controls = new FlyControls(camera, renderer.domElement);
}
```

**What happens:** The old OrbitControls' event listeners (pointermove, pointerdown, pointerup, wheel, keydown, contextmenu) remain active on `renderer.domElement`. This causes:
- Memory leaks from accumulated listeners
- Ghost interactions where the old controls still respond to input
- Conflicting camera movement from two control systems

**CORRECT:**
```javascript
function switchToFlyMode() {
  controls.dispose(); // ALWAYS dispose before replacing
  controls = new FlyControls(camera, renderer.domElement);
}
```

---

## Anti-Pattern 3: Attaching to document Instead of renderer.domElement

**WRONG:**
```javascript
const controls = new OrbitControls(camera, document.body);
```

**What happens:** Controls capture pointer events on the entire page. Any UI elements (buttons, sliders, dropdowns) above or beside the canvas become unusable because the controls intercept their events. Scroll events on the page trigger camera zoom.

**CORRECT:**
```javascript
const controls = new OrbitControls(camera, renderer.domElement);
```

ALWAYS pass `renderer.domElement` so controls ONLY respond to events on the canvas.

---

## Anti-Pattern 4: Mixing Two Camera Controls Without Disabling

**WRONG:**
```javascript
const orbitControls = new OrbitControls(camera, renderer.domElement);
const flyControls = new FlyControls(camera, renderer.domElement);

function animate() {
  requestAnimationFrame(animate);
  orbitControls.update();
  flyControls.update(delta);
  renderer.render(scene, camera);
}
```

**What happens:** Both controls fight over the camera's position and rotation every frame. The camera jitters, teleports, or moves unpredictably. Each control overwrites what the other set.

**CORRECT:**
```javascript
let activeControls = orbitControls;
flyControls.enabled = false; // or don't create until needed

function switchControls(type) {
  activeControls.dispose();
  if (type === 'fly') {
    activeControls = new FlyControls(camera, renderer.domElement);
  } else {
    activeControls = new OrbitControls(camera, renderer.domElement);
  }
}
```

NEVER have two camera controls active simultaneously on the same camera. ALWAYS dispose or disable one before enabling another.

---

## Anti-Pattern 5: Not Disabling OrbitControls During TransformControls Drag

**WRONG:**
```javascript
const orbitControls = new OrbitControls(camera, renderer.domElement);
const transformControls = new TransformControls(camera, renderer.domElement);
scene.add(transformControls);
transformControls.attach(cube);
// No dragging-changed listener!
```

**What happens:** When the user drags a TransformControls gizmo handle, OrbitControls also responds to the drag. The camera orbits while the object moves, making precise positioning impossible.

**CORRECT:**
```javascript
transformControls.addEventListener('dragging-changed', (event) => {
  orbitControls.enabled = !event.value;
});
```

ALWAYS listen to `dragging-changed` and disable camera controls while `event.value` is `true`.

---

## Anti-Pattern 6: Not Adding TransformControls to the Scene

**WRONG:**
```javascript
const transformControls = new TransformControls(camera, renderer.domElement);
transformControls.attach(cube);
// Forgot scene.add(transformControls) !
```

**What happens:** The gizmo handles are invisible. The TransformControls object exists and processes events, but nothing renders on screen. The user cannot see or click the translate/rotate/scale handles.

**CORRECT:**
```javascript
const transformControls = new TransformControls(camera, renderer.domElement);
scene.add(transformControls); // MUST add to scene for gizmo visibility
transformControls.attach(cube);
```

---

## Anti-Pattern 7: Calling PointerLockControls.lock() Without User Gesture

**WRONG:**
```javascript
const controls = new PointerLockControls(camera, renderer.domElement);
controls.lock(); // Called immediately on page load
```

**What happens:** The browser silently rejects the pointer lock request. The Pointer Lock API requires a user-initiated event (click, keydown). The console shows no error, but the pointer is never locked and `isLocked` remains `false`.

**CORRECT:**
```javascript
const controls = new PointerLockControls(camera, renderer.domElement);

renderer.domElement.addEventListener('click', () => {
  controls.lock(); // ALWAYS from a user gesture
});
```

---

## Anti-Pattern 8: Omitting delta in FlyControls.update()

**WRONG:**
```javascript
function animate() {
  requestAnimationFrame(animate);
  controls.update(); // Missing delta argument!
  renderer.render(scene, camera);
}
```

**What happens:** `FlyControls.update()` receives `undefined` as delta, resulting in `NaN` or zero movement. The camera either freezes completely or moves at an unpredictable speed depending on internal fallback behavior.

**CORRECT:**
```javascript
const clock = new THREE.Clock();

function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  controls.update(delta); // ALWAYS pass delta for FlyControls
  renderer.render(scene, camera);
}
```

---

## Anti-Pattern 9: Setting target Without Calling update()

**WRONG:**
```javascript
controls.target.set(10, 0, 5);
// Camera still looks at (0, 0, 0) until next user interaction
```

**What happens:** Changing `target` only sets the value. The camera does NOT re-orient until `update()` is called. If `enableDamping` is false and the render loop does not call `update()`, the camera appears stuck at the old target.

**CORRECT:**
```javascript
controls.target.set(10, 0, 5);
controls.update(); // ALWAYS call after programmatic target changes
```

---

## Anti-Pattern 10: Using OrbitControls with Orthographic Camera Zoom Properties on Perspective Camera

**WRONG:**
```javascript
const camera = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000);
const controls = new OrbitControls(camera, renderer.domElement);
controls.minZoom = 0.5;  // Has NO effect on PerspectiveCamera
controls.maxZoom = 5.0;  // Has NO effect on PerspectiveCamera
```

**What happens:** `minZoom` and `maxZoom` ONLY affect `OrthographicCamera.zoom`. For `PerspectiveCamera`, these properties are silently ignored. The camera zooms without any limits.

**CORRECT:**
```javascript
// For PerspectiveCamera -- use distance limits
controls.minDistance = 2;
controls.maxDistance = 50;

// For OrthographicCamera -- use zoom limits
// controls.minZoom = 0.5;
// controls.maxZoom = 5.0;
```

ALWAYS use `minDistance`/`maxDistance` for PerspectiveCamera and `minZoom`/`maxZoom` for OrthographicCamera. NEVER mix them up.
