# threejs-impl-xr — Methods Reference

## WebXRManager

Accessed via `renderer.xr`. Manages the complete WebXR session lifecycle.

### Properties

```typescript
renderer.xr.enabled: boolean                // Default: false. MUST set to true for XR.
renderer.xr.isPresenting: boolean            // Read-only. True when XR session is active.
renderer.xr.cameraAutoUpdate: boolean        // Default: true. Auto-updates camera from XR device pose.
```

### Methods

```typescript
// Session management
renderer.xr.getSession(): XRSession | null
renderer.xr.setSessionInit(sessionInit: XRSessionInit): void
renderer.xr.setReferenceSpaceType(type: string): void
renderer.xr.getReferenceSpace(): XRReferenceSpace | null

// Controllers
renderer.xr.getController(index: number): THREE.Group
renderer.xr.getControllerGrip(index: number): THREE.Group
renderer.xr.getHand(index: number): THREE.Group

// Performance
renderer.xr.setFoveation(foveation: number): void      // 0.0 (none) to 1.0 (max)
renderer.xr.getFoveation(): number
renderer.xr.setFramebufferScaleFactor(scale: number): void

// Environment
renderer.xr.getEnvironmentBlendMode(): string           // 'opaque' | 'additive' | 'alpha-blend'
```

---

## VRButton

```typescript
import { VRButton } from 'three/addons/webxr/VRButton.js';

VRButton.createButton(renderer: THREE.WebGLRenderer): HTMLElement
```

Returns a DOM button element that:
- Checks for WebXR `immersive-vr` support
- Shows "ENTER VR" when supported
- Shows "VR NOT SUPPORTED" when unavailable
- Handles session request/end lifecycle automatically

---

## ARButton

```typescript
import { ARButton } from 'three/addons/webxr/ARButton.js';

ARButton.createButton(
  renderer: THREE.WebGLRenderer,
  sessionInit?: XRSessionInit
): HTMLElement
```

Returns a DOM button element that:
- Checks for WebXR `immersive-ar` support
- Accepts optional `sessionInit` for required/optional features
- Handles session request/end lifecycle automatically

### XRSessionInit Structure

```typescript
interface XRSessionInit {
  requiredFeatures?: string[];    // Session fails if unavailable
  optionalFeatures?: string[];    // Session proceeds without these
  domOverlay?: { root: HTMLElement };
}
```

Common features: `'hit-test'`, `'hand-tracking'`, `'dom-overlay'`, `'anchors'`, `'plane-detection'`, `'depth-sensing'`, `'light-estimation'`.

---

## XRControllerModelFactory

```typescript
import { XRControllerModelFactory } from 'three/addons/webxr/XRControllerModelFactory.js';

const factory = new XRControllerModelFactory(gltfLoader?: GLTFLoader);
factory.createControllerModel(controllerGrip: THREE.Group): THREE.Object3D
```

- ALWAYS pass the grip space (`getControllerGrip`), NOT the target ray space
- Automatically loads the correct 3D model for the detected controller hardware
- Uses the WebXR Input Profiles library for model matching

---

## XRHandModelFactory

```typescript
import { XRHandModelFactory } from 'three/addons/webxr/XRHandModelFactory.js';

const factory = new XRHandModelFactory();
factory.createHandModel(
  hand: THREE.Group,
  profile?: 'mesh' | 'spheres' | 'boxes'
): THREE.Object3D
```

- `'mesh'` — Realistic hand mesh (default)
- `'spheres'` — Joint positions as spheres
- `'boxes'` — Joint positions as boxes

---

## XREstimatedLight

```typescript
import { XREstimatedLight } from 'three/addons/webxr/XREstimatedLight.js';

const xrLight = new XREstimatedLight(renderer: THREE.WebGLRenderer);
xrLight.environment: THREE.Texture    // Environment map for PBR materials
xrLight.lightProbe: THREE.LightProbe  // Spherical harmonics light probe
xrLight.directionalLight: THREE.DirectionalLight  // Primary directional light
```

Events:
- `'estimationstart'` — Light estimation data available
- `'estimationend'` — Light estimation lost

---

## XRPlanes

```typescript
import { XRPlanes } from 'three/addons/webxr/XRPlanes.js';

const planes = new XRPlanes(renderer: THREE.WebGLRenderer);
scene.add(planes);
```

Requires `'plane-detection'` in session features. Automatically creates mesh visualizations for detected real-world planes.

---

## renderer.setAnimationLoop

```typescript
renderer.setAnimationLoop(
  callback: ((time: DOMHighResTimeStamp, frame?: XRFrame) => void) | null
): void
```

- MUST use this instead of `requestAnimationFrame` for XR
- The `frame` parameter is an `XRFrame` when an XR session is active, `undefined` otherwise
- Pass `null` to stop the loop
- Works for both XR and non-XR rendering (safe to use unconditionally)

---

## Controller Events

All events fire on the `THREE.Group` returned by `getController()`:

```typescript
controller.addEventListener('select', (event: { target: THREE.Group }) => void): void
controller.addEventListener('selectstart', (event: { target: THREE.Group }) => void): void
controller.addEventListener('selectend', (event: { target: THREE.Group }) => void): void
controller.addEventListener('squeeze', (event: { target: THREE.Group }) => void): void
controller.addEventListener('squeezestart', (event: { target: THREE.Group }) => void): void
controller.addEventListener('squeezeend', (event: { target: THREE.Group }) => void): void
controller.addEventListener('connected', (event: { data: XRInputSource, target: THREE.Group }) => void): void
controller.addEventListener('disconnected', (event: { target: THREE.Group }) => void): void
```

### XRInputSource Properties (from `connected` event)

```typescript
event.data.handedness: 'none' | 'left' | 'right'
event.data.targetRayMode: 'gaze' | 'tracked-pointer' | 'screen'
event.data.profiles: string[]        // e.g., ['oculus-touch-v3', 'oculus-touch-v2']
event.data.gamepad: Gamepad | null   // Buttons and axes for gamepad-style input
event.data.hand: XRHand | null       // Hand joint data if hand tracking active
```

---

## Hand Tracking Events

Events fire on the `THREE.Group` returned by `getHand()`:

```typescript
hand.addEventListener('connected', (event) => void): void
hand.addEventListener('disconnected', (event) => void): void
hand.addEventListener('pinchstart', (event) => void): void
hand.addEventListener('pinchend', (event) => void): void
```

---

## WebXR Hit Test API (Browser API, used within Three.js loop)

```typescript
// Request hit test source (once per session)
const viewerSpace = await session.requestReferenceSpace('viewer');
const hitTestSource = await session.requestHitTestSource({ space: viewerSpace });

// Each frame
const results: XRHitTestResult[] = frame.getHitTestResults(hitTestSource);
const pose: XRPose = results[0].getPose(referenceSpace);
// pose.transform.matrix is a Float32Array(16)
```
