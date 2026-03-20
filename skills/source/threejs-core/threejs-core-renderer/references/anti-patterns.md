# threejs-core-renderer — Anti-Patterns

> What NOT to do with WebGLRenderer and cameras.
> Each anti-pattern shows the wrong code, explains why it fails, and provides the correct alternative.

---

## Anti-Pattern 1: Uncapped Pixel Ratio

### Wrong

```javascript
renderer.setPixelRatio(window.devicePixelRatio);
```

### Why It Fails

On devices with `devicePixelRatio` of 3 or higher, this renders 9x+ more pixels than a 1x display. A 1920x1080 canvas at 3x becomes 5760x3240 — over 18 million pixels per frame. This causes severe frame drops, GPU overheating on mobile, and provides no visible quality benefit over 2x.

### Correct

```javascript
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
```

ALWAYS cap at 2. The visual difference between 2x and 3x is imperceptible, but the performance cost is massive.

---

## Anti-Pattern 2: Forgetting updateProjectionMatrix

### Wrong

```javascript
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

### Why It Fails

Modifying `camera.aspect` (or `fov`, `near`, `far`, `zoom`) does NOT automatically recompute the projection matrix. The scene continues rendering with the old aspect ratio, causing stretched or squished output.

### Correct

```javascript
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

ALWAYS call `camera.updateProjectionMatrix()` after changing any projection parameter.

---

## Anti-Pattern 3: Wrong Color Space for Data Textures

### Wrong

```javascript
import { TextureLoader, SRGBColorSpace } from 'three';

const loader = new TextureLoader();
const normalMap = loader.load('normal.png');
normalMap.colorSpace = SRGBColorSpace; // WRONG for data textures
```

### Why It Fails

Normal maps, roughness maps, metalness maps, AO maps, and displacement maps contain linear data values, not perceptual colors. Applying sRGB gamma decoding to these textures corrupts the data — normals point in wrong directions, roughness values are nonlinear, and lighting calculations produce incorrect results.

### Correct

```javascript
import { TextureLoader, SRGBColorSpace, LinearSRGBColorSpace } from 'three';

const loader = new TextureLoader();

// Color textures: SRGBColorSpace
const diffuseMap = loader.load('diffuse.png');
diffuseMap.colorSpace = SRGBColorSpace;

// Data textures: LinearSRGBColorSpace
const normalMap = loader.load('normal.png');
normalMap.colorSpace = LinearSRGBColorSpace;

const roughnessMap = loader.load('roughness.png');
roughnessMap.colorSpace = LinearSRGBColorSpace;
```

---

## Anti-Pattern 4: Changing Shadow Map Type After First Render

### Wrong

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = PCFShadowMap;

// Later, at runtime...
renderer.shadowMap.type = PCFSoftShadowMap; // triggers full shader recompilation
```

### Why It Fails

Changing `shadowMap.type` after the first render forces Three.js to recompile ALL shadow-receiving shaders. This causes a massive frame spike and temporary freeze. The shadow map type is baked into shader defines at compile time.

### Correct

```javascript
// Set shadow map type ONCE at initialization, before any render call
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = PCFSoftShadowMap;
```

ALWAYS decide the shadow map type at initialization. NEVER change it at runtime.

---

## Anti-Pattern 5: Using requestAnimationFrame Instead of setAnimationLoop

### Wrong

```javascript
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

### Why It Fails

Raw `requestAnimationFrame` does not integrate with the WebXR session lifecycle. When entering VR/AR mode, the XR runtime provides its own frame callback. Using `requestAnimationFrame` means: (1) the XR session loop and your loop compete, (2) the timestamp is wrong for XR frames, (3) you must manually manage XR session start/stop.

### Correct

```javascript
renderer.setAnimationLoop((time) => {
  renderer.render(scene, camera);
});
```

`setAnimationLoop` automatically switches between `requestAnimationFrame` (desktop) and `XRSession.requestAnimationFrame` (XR mode).

---

## Anti-Pattern 6: Forgetting to Dispose the Renderer

### Wrong

```javascript
// Component unmount or page navigation
document.body.removeChild(renderer.domElement);
// No dispose call — WebGL context leaks
```

### Why It Fails

Browsers have a hard limit on active WebGL contexts (typically 8-16). Without `dispose()`, the WebGL context, all compiled shader programs, all GPU textures, and all framebuffers remain allocated. In single-page applications that create/destroy renderers, this causes "Too many active WebGL contexts" errors.

### Correct

```javascript
renderer.setAnimationLoop(null);
renderer.dispose();
renderer.domElement.remove();
```

ALWAYS call `dispose()` before removing the renderer. Stop the animation loop first.

---

## Anti-Pattern 7: preserveDrawingBuffer in Production

### Wrong

```javascript
const renderer = new WebGLRenderer({
  antialias: true,
  preserveDrawingBuffer: true, // "just in case"
});
```

### Why It Fails

`preserveDrawingBuffer: true` prevents the browser from using efficient buffer-swap techniques. Instead of swapping front/back buffers (near-instant), the browser must copy the buffer contents. This adds overhead to every single frame, even when no screenshot is ever taken.

### Correct

```javascript
// Production renderer — no preserveDrawingBuffer
const renderer = new WebGLRenderer({ antialias: true });

// When a screenshot IS needed, use a one-time render approach:
function takeScreenshot() {
  renderer.render(scene, camera);
  return renderer.domElement.toDataURL('image/png');
  // Works because toDataURL is called immediately after render,
  // before the buffer can be cleared
}
```

If `toDataURL` must work at any time (not just immediately after render), create a separate renderer with `preserveDrawingBuffer: true` for screenshot purposes only.

---

## Anti-Pattern 8: Setting near to Zero

### Wrong

```javascript
const camera = new PerspectiveCamera(50, aspect, 0, 10000);
```

### Why It Fails

The depth buffer precision is distributed logarithmically between `near` and `far`. Setting `near: 0` (or very close to 0, like `0.0001`) concentrates almost all depth precision in the first few centimeters. Objects beyond that suffer extreme Z-fighting — flickering surfaces where two faces compete for the same depth value.

### Correct

```javascript
// General 3D scenes
const camera = new PerspectiveCamera(50, aspect, 0.1, 1000);

// Large-scale scenes (architecture, geography)
const camera = new PerspectiveCamera(50, aspect, 0.1, 50000);
// AND enable logarithmic depth on the renderer:
const renderer = new WebGLRenderer({ logarithmicDepthBuffer: true });
```

ALWAYS keep the `far / near` ratio under 10,000. For scenes that need a wider range, use `logarithmicDepthBuffer: true`.

---

## Anti-Pattern 9: Multi-Pass Rendering with autoClear Enabled

### Wrong

```javascript
// Rendering two passes (e.g., split-screen or overlay)
renderer.render(sceneBackground, camera);  // first pass
renderer.render(sceneOverlay, camera);     // second pass — CLEARS the first!
```

### Why It Fails

`autoClear` defaults to `true`. Each `render()` call clears the color, depth, and stencil buffers before drawing. The second render erases everything from the first.

### Correct

```javascript
renderer.autoClear = false;

renderer.clear(); // explicit clear once
renderer.render(sceneBackground, camera);

renderer.clearDepth(); // clear only depth for overlay
renderer.render(sceneOverlay, camera);
```

ALWAYS set `autoClear = false` when doing multi-pass rendering. Manage clears explicitly.

---

## Anti-Pattern 10: Not Resizing Render Targets

### Wrong

```javascript
const renderTarget = new WebGLRenderTarget(1024, 1024);

window.addEventListener('resize', () => {
  renderer.setSize(window.innerWidth, window.innerHeight);
  // renderTarget is still 1024x1024 — resolution mismatch
});
```

### Why It Fails

If the render target is used for full-screen effects (like post-processing), it must match the output resolution. A fixed-size render target produces blurry or pixelated results when the window changes size.

### Correct

```javascript
const renderTarget = new WebGLRenderTarget(
  window.innerWidth * Math.min(window.devicePixelRatio, 2),
  window.innerHeight * Math.min(window.devicePixelRatio, 2)
);

window.addEventListener('resize', () => {
  const pixelRatio = Math.min(window.devicePixelRatio, 2);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderTarget.setSize(
    window.innerWidth * pixelRatio,
    window.innerHeight * pixelRatio
  );
});
```

ALWAYS resize render targets alongside the renderer when they are used for full-screen effects.
