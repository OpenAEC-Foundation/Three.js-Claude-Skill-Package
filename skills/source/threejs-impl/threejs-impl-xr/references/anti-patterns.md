# threejs-impl-xr — Anti-Patterns

## Anti-Pattern 1: Using requestAnimationFrame for XR

### Wrong

```javascript
renderer.xr.enabled = true;

function animate() {
  requestAnimationFrame(animate);  // BREAKS in XR
  renderer.render(scene, camera);
}
animate();
```

### Why It Fails

`requestAnimationFrame` stops firing when a WebXR session is active. The WebXR API uses its own frame loop synchronized to the headset's refresh rate. The scene will freeze the moment the user enters VR/AR.

### Correct

```javascript
renderer.xr.enabled = true;

renderer.setAnimationLoop((time, frame) => {
  renderer.render(scene, camera);
});
```

`setAnimationLoop` works for BOTH XR and non-XR rendering. ALWAYS use it when XR is enabled, even if the app also runs in non-XR mode.

---

## Anti-Pattern 2: Moving the Camera Directly in VR

### Wrong

```javascript
// Trying to teleport by moving the camera
camera.position.set(5, 0, 10);
```

### Why It Fails

The WebXR API overwrites the camera's position and rotation EVERY frame based on the headset's physical pose. Any direct camera manipulation is immediately overridden. The user sees no movement.

### Correct

```javascript
const cameraRig = new THREE.Group();
cameraRig.add(camera);
scene.add(cameraRig);

// Move the rig — camera position is relative to rig
cameraRig.position.set(5, 0, 10);
```

ALWAYS use a camera rig group. The WebXR API positions the camera relative to its parent, so moving the parent moves the entire viewpoint.

---

## Anti-Pattern 3: Heavy Post-Processing in VR

### Wrong

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { SSAOPass } from 'three/addons/postprocessing/SSAOPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new SSAOPass(scene, camera));
composer.addPass(new UnrealBloomPass());

renderer.setAnimationLoop(() => {
  composer.render();  // Runs ALL passes for EACH eye
});
```

### Why It Fails

In stereo VR, each post-processing pass runs TWICE (once per eye). SSAO + Bloom + other passes can easily exceed the frame budget. At 90fps, each frame has only 11.1ms. Dropped frames cause motion sickness.

### Correct

```javascript
// Minimize or eliminate post-processing in VR
renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);  // Direct render, no composer
});

// If post-processing is essential, use ONLY lightweight passes
// and monitor frame timing with renderer.info
```

NEVER use SSAO, screen-space reflections, or multi-pass bloom in VR. If visual effects are needed, bake them into textures or use material-based alternatives.

---

## Anti-Pattern 4: Forgetting to Add Controllers to Scene

### Wrong

```javascript
const controller = renderer.xr.getController(0);
controller.addEventListener('select', onSelect);
// Controller exists but is never added to the scene
```

### Why It Fails

`getController()` returns a `THREE.Group` that MUST be added to the scene graph. Without `scene.add(controller)`, the controller's `matrixWorld` is never updated. Event handlers fire, but `controller.position` and `controller.matrixWorld` contain stale identity matrices. Raycasting from the controller produces incorrect results.

### Correct

```javascript
const controller = renderer.xr.getController(0);
controller.addEventListener('select', onSelect);
scene.add(controller);  // MUST add to scene

// Same for grip and hand spaces
const grip = renderer.xr.getControllerGrip(0);
scene.add(grip);

const hand = renderer.xr.getHand(0);
scene.add(hand);
```

ALWAYS add EVERY controller space you use to the scene (or to a camera rig group if using locomotion).

---

## Anti-Pattern 5: Wrong Reference Space for Standing VR

### Wrong

```javascript
renderer.xr.setReferenceSpaceType('local');
// User stands up — floor is at eye level
```

### Why It Fails

`'local'` places the origin at the headset's initial position (eye level). Objects placed at y=0 appear at head height, not on the floor. The user's real-world floor has no representation in the coordinate system.

### Correct

```javascript
renderer.xr.setReferenceSpaceType('local-floor');
// y=0 is at floor level
```

ALWAYS use `'local-floor'` for standing/room-scale VR. Use `'local'` ONLY for seated experiences where floor position is irrelevant.

---

## Anti-Pattern 6: Not Setting enabled Before Creating XR Button

### Wrong

```javascript
// renderer.xr.enabled is still false (default)
document.body.appendChild(VRButton.createButton(renderer));
renderer.xr.enabled = true;  // Too late
```

### Why It Fails

`VRButton.createButton()` checks `navigator.xr.isSessionSupported()` and configures the button based on the renderer's XR state. Setting `enabled` after button creation can cause the button to show incorrect state or fail to initialize the session properly.

### Correct

```javascript
renderer.xr.enabled = true;  // FIRST
renderer.xr.setReferenceSpaceType('local-floor');
document.body.appendChild(VRButton.createButton(renderer));  // AFTER
```

ALWAYS configure `renderer.xr` properties BEFORE creating VRButton or ARButton.

---

## Anti-Pattern 7: Ignoring Foveated Rendering

### Wrong

```javascript
renderer.xr.enabled = true;
// Default foveation is 0 (no foveation)
// Rendering full resolution across entire field of view
```

### Why It Fails

Without foveated rendering, the GPU renders at full resolution across the entire lens area, including the periphery where the user cannot perceive detail. This wastes GPU budget that could maintain stable frame rates.

### Correct

```javascript
renderer.xr.enabled = true;
renderer.xr.setFoveation(1.0);  // Maximum foveated rendering
```

ALWAYS set foveation to 1.0 unless visual quality in the periphery is critical. This provides the largest performance gain with minimal perceived quality loss.

---

## Anti-Pattern 8: Creating Objects Every Frame in XR

### Wrong

```javascript
renderer.setAnimationLoop((time, frame) => {
  // Allocating new objects every frame
  const direction = new THREE.Vector3(0, 0, -1);
  const matrix = new THREE.Matrix4();
  matrix.extractRotation(controller.matrixWorld);
  direction.applyMatrix4(matrix);
  renderer.render(scene, camera);
});
```

### Why It Fails

Allocating objects in the render loop causes garbage collection pauses. In VR, even a 5ms GC pause causes a visible frame drop and potential nausea. At 90fps, the ENTIRE frame budget is 11.1ms.

### Correct

```javascript
// Pre-allocate outside the loop
const direction = new THREE.Vector3();
const tempMatrix = new THREE.Matrix4();

renderer.setAnimationLoop((time, frame) => {
  direction.set(0, 0, -1);
  tempMatrix.extractRotation(controller.matrixWorld);
  direction.applyMatrix4(tempMatrix);
  renderer.render(scene, camera);
});
```

ALWAYS pre-allocate vectors, matrices, quaternions, and other math objects outside the render loop. Reuse them every frame.

---

## Anti-Pattern 9: Not Handling Session End Cleanup

### Wrong

```javascript
let hitTestSource = null;

renderer.setAnimationLoop((time, frame) => {
  if (frame && hitTestSource) {
    const results = frame.getHitTestResults(hitTestSource);
    // hitTestSource becomes invalid after session ends
    // but code still tries to use it
  }
});
```

### Why It Fails

When an XR session ends, all session-specific resources (hit test sources, reference spaces, anchors) become invalid. Using them after session end throws errors or returns empty results. Re-entering XR creates a new session with new resources.

### Correct

```javascript
let hitTestSource = null;

const session = renderer.xr.getSession();
session.addEventListener('end', () => {
  hitTestSource = null;  // Clean up session-specific state
});

renderer.setAnimationLoop((time, frame) => {
  if (frame && hitTestSource) {
    const results = frame.getHitTestResults(hitTestSource);
    // Safe — hitTestSource is nulled on session end
  }
});
```

ALWAYS listen for the `'end'` event on the XR session and reset all session-specific state.
