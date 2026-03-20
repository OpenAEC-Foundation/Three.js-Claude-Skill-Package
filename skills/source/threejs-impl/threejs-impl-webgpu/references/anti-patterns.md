# Anti-Patterns (Three.js WebGPU)

## 1. Rendering Before Async Init

```javascript
// WRONG: render() called before init() resolves — throws runtime error
const renderer = new THREE.WebGPURenderer({ antialias: true });
renderer.render(scene, camera); // ERROR: renderer not initialized

// CORRECT: ALWAYS await init() before any render call
const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init();
renderer.render(scene, camera);
```

**WHY**: WebGPURenderer requires asynchronous GPU adapter and device initialization. Calling render() before init() completes causes a runtime error because the GPU device handle does not exist yet.

---

## 2. No Browser Support Check

```javascript
// WRONG: Assumes WebGPU is available everywhere
const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init();

// CORRECT: ALWAYS check availability and provide fallback
import { WebGPU } from 'three/webgpu';

let renderer;
if (WebGPU.isAvailable()) {
  renderer = new THREE.WebGPURenderer({ antialias: true });
  await renderer.init();
} else {
  renderer = new THREE.WebGLRenderer({ antialias: true });
}
```

**WHY**: WebGPU is NOT available in Firefox (without a flag), older browsers, or many mobile browsers. Without a fallback, the application crashes for a large portion of users.

---

## 3. Writing Raw GLSL with WebGPU

```javascript
// WRONG: GLSL strings are NOT supported by WebGPU backend
const material = new THREE.ShaderMaterial({
  vertexShader: `void main() { gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
  fragmentShader: `void main() { gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0); }`,
});

// CORRECT: Use TSL with NodeMaterial
import { MeshBasicNodeMaterial } from 'three/webgpu';
import { vec4, float } from 'three/tsl';

const material = new MeshBasicNodeMaterial();
material.colorNode = vec4(float(1.0), float(0.0), float(0.0), float(1.0));
```

**WHY**: WebGPU uses WGSL, not GLSL. Three.js compiles TSL to WGSL automatically. ShaderMaterial with GLSL strings only works with WebGLRenderer. ALWAYS use TSL for cross-backend compatibility.

---

## 4. Using requestAnimationFrame

```javascript
// WRONG: requestAnimationFrame does not integrate with WebGPU frame timing
function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();

// CORRECT: ALWAYS use setAnimationLoop for WebGPU
renderer.setAnimationLoop((time) => {
  renderer.render(scene, camera);
});
```

**WHY**: `renderer.setAnimationLoop()` handles WebGPU frame scheduling, XR session loops, and renderer lifecycle correctly. Using `requestAnimationFrame` bypasses these mechanisms and can cause timing issues or missed frames with WebGPU.

---

## 5. Using EffectComposer with WebGPU

```javascript
// WRONG: EffectComposer is WebGL-only
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';

const composer = new EffectComposer(renderer); // FAILS with WebGPURenderer

// CORRECT: Use PostProcessing class with TSL nodes
import { PostProcessing } from 'three/webgpu';
import { pass, bloom, renderOutput } from 'three/tsl';

const postProcessing = new PostProcessing(renderer);
const scenePass = pass(scene, camera);
postProcessing.outputNode = renderOutput(bloom(scenePass));
```

**WHY**: The `EffectComposer` and its passes use WebGL-specific render targets and GLSL shaders. They are fundamentally incompatible with WebGPURenderer. The `PostProcessing` class uses TSL nodes that compile to WGSL.

---

## 6. Importing from Wrong Module Path

```javascript
// WRONG: Standard three import does not include WebGPU classes
import * as THREE from 'three';
const renderer = new THREE.WebGPURenderer(); // undefined

// CORRECT: ALWAYS import from 'three/webgpu' for WebGPU classes
import * as THREE from 'three/webgpu';
const renderer = new THREE.WebGPURenderer({ antialias: true });
```

**WHY**: `WebGPURenderer`, node materials, and WebGPU utilities are only exported from `'three/webgpu'`. The standard `'three'` entry point does not include them.

---

## 7. Using Lowercase `if` in TSL

```javascript
// WRONG: JavaScript if does not generate shader conditionals
if (someNode.greaterThan(float(0.5))) {
  material.colorNode = color(0xff0000);
}

// CORRECT: ALWAYS use TSL's capital If for shader-level conditionals
import { If, float, color } from 'three/tsl';

If(someNode.greaterThan(float(0.5)), () => {
  material.colorNode = color(0xff0000);
}).Else(() => {
  material.colorNode = color(0x0000ff);
});
```

**WHY**: JavaScript `if` evaluates at material creation time, not per-fragment. TSL `If()` generates actual GPU conditional instructions that execute per-fragment on the GPU. Using JavaScript `if` produces a static material instead of a dynamic one.

---

## 8. Synchronous Compute Dispatch

```javascript
// WRONG: computeAsync returns a Promise — ignoring it causes race conditions
renderer.computeAsync(computeNode); // fire-and-forget
renderer.render(scene, camera);     // may render before compute finishes

// CORRECT: ALWAYS await compute before rendering dependent results
await renderer.computeAsync(computeNode);
renderer.render(scene, camera);
```

**WHY**: GPU compute operations are asynchronous. Rendering before the compute pass finishes can display stale or incomplete data. ALWAYS await `computeAsync()` when the render depends on compute results.

---

## 9. Forgetting to Make Entry Point Async

```javascript
// WRONG: Cannot use await at top level in non-module scripts
const renderer = new THREE.WebGPURenderer();
renderer.init(); // Promise ignored, init never completes

// CORRECT: Use async entry point
async function init() {
  const renderer = new THREE.WebGPURenderer({ antialias: true });
  await renderer.init();
  // ... rest of setup
}
init();
```

**WHY**: `WebGPURenderer.init()` is a Promise. Without `await`, the renderer is used before GPU initialization completes. ALWAYS wrap WebGPU setup in an `async` function or use top-level `await` in ES modules.

---

## 10. Mixing Classic and Node Material APIs Incorrectly

```javascript
// WRONG: Setting classic property after assigning node — node takes precedence
const material = new MeshStandardNodeMaterial();
material.colorNode = color(0xff0000);
material.color.set(0x00ff00); // has NO effect — colorNode overrides

// CORRECT: Use EITHER classic props OR node props, not both
// Option A: Classic props only (auto-converts for WebGPU)
const material = new MeshStandardNodeMaterial({ color: 0x00ff00 });

// Option B: Node props only (full TSL control)
const material = new MeshStandardNodeMaterial();
material.colorNode = color(0x00ff00);
```

**WHY**: When a node property (e.g., `colorNode`) is set, it completely overrides the corresponding classic property (e.g., `color`). Setting both creates confusion and the classic value is silently ignored. Choose one approach and use it consistently.
