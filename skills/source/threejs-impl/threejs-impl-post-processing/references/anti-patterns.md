# anti-patterns.md -- threejs-impl-post-processing

> What NOT to do with Three.js post-processing, with explanations of WHY.

---

## Anti-Pattern 1: Missing RenderPass

**WRONG:**
```js
const composer = new EffectComposer( renderer );
composer.addPass( new UnrealBloomPass( resolution, 1.5, 0.4, 0.85 ) );
composer.addPass( new OutputPass() );
```

**WHY:** Without `RenderPass` as the first pass, the bloom pass receives an EMPTY buffer. No scene content is rendered into the post-processing pipeline. The result is a completely black screen.

**CORRECT:**
```js
const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) ); // ALWAYS first
composer.addPass( new UnrealBloomPass( resolution, 1.5, 0.4, 0.85 ) );
composer.addPass( new OutputPass() );
```

---

## Anti-Pattern 2: Missing OutputPass

**WRONG:**
```js
const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( new UnrealBloomPass( resolution, 1.5, 0.4, 0.85 ) );
// No OutputPass
```

**WHY:** `OutputPass` applies tone mapping and color space conversion. Without it, colors appear washed out, overly bright, or in the wrong color space (linear instead of sRGB). This is especially visible with HDR effects like bloom.

**CORRECT:**
```js
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( new UnrealBloomPass( resolution, 1.5, 0.4, 0.85 ) );
composer.addPass( new OutputPass() ); // ALWAYS last
```

---

## Anti-Pattern 3: Mixing Two EffectComposer Libraries

**WRONG:**
```js
// Importing from BOTH libraries
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { BloomEffect, EffectPass } from 'postprocessing';

const composer = new EffectComposer( renderer ); // Three.js built-in
composer.addPass( new EffectPass( camera, new BloomEffect() ) ); // pmndrs effect
```

**WHY:** The built-in `EffectComposer` and `pmndrs/postprocessing` `EffectComposer` use incompatible internal buffer formats and rendering pipelines. Mixing passes from one library into the other produces rendering artifacts, black screens, or crashes. They are two completely separate systems.

**CORRECT:** Choose ONE library and use it exclusively:

```js
// Option A: Built-in only
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

// Option B: pmndrs only
import { EffectComposer, RenderPass, EffectPass, BloomEffect } from 'postprocessing';
```

---

## Anti-Pattern 4: Forgetting to Resize the Composer

**WRONG:**
```js
window.addEventListener( 'resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize( window.innerWidth, window.innerHeight );
  // Missing: composer.setSize()
} );
```

**WHY:** The `EffectComposer` maintains its own internal render targets at a fixed resolution. If you resize only the renderer, the internal buffers remain at the OLD size. This causes blurry, stretched, or misaligned post-processing effects.

**CORRECT:**
```js
window.addEventListener( 'resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize( window.innerWidth, window.innerHeight );
  composer.setSize( window.innerWidth, window.innerHeight ); // ALWAYS resize both
} );
```

---

## Anti-Pattern 5: Calling renderer.render() AND composer.render()

**WRONG:**
```js
function animate() {
  requestAnimationFrame( animate );
  renderer.render( scene, camera ); // renders scene directly to screen
  composer.render();                 // renders scene AGAIN through post-processing
}
```

**WHY:** The `RenderPass` inside the composer already calls `renderer.render()` internally. Calling `renderer.render()` before `composer.render()` renders the scene twice: once directly to the screen (without effects) and once through the post-processing pipeline. This wastes GPU resources and can cause flickering.

**CORRECT:**
```js
function animate() {
  requestAnimationFrame( animate );
  composer.render(); // This is ALL you need
}
```

---

## Anti-Pattern 6: Wrong Pass Order (AA Before Effects)

**WRONG:**
```js
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( new SMAAPass( width, height ) );      // AA FIRST
composer.addPass( new UnrealBloomPass( res, 1.5, 0.4, 0.85 ) ); // Bloom AFTER AA
composer.addPass( new OutputPass() );
```

**WHY:** Anti-aliasing smooths edges in the current buffer. If applied BEFORE bloom, the bloom pass introduces new aliased edges that are NOT anti-aliased. ALWAYS apply anti-aliasing AFTER all effect passes.

**CORRECT:**
```js
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( new UnrealBloomPass( res, 1.5, 0.4, 0.85 ) ); // Effects FIRST
composer.addPass( new SMAAPass( width, height ) );                // AA AFTER effects
composer.addPass( new OutputPass() );
```

---

## Anti-Pattern 7: Missing tDiffuse Uniform in Custom ShaderPass

**WRONG:**
```js
const myShader = {
  uniforms: {
    amount: { value: 0.5 }
    // Missing tDiffuse uniform
  },
  vertexShader: `...`,
  fragmentShader: `
    uniform float amount;
    varying vec2 vUv;
    void main() {
      gl_FragColor = vec4( amount, amount, amount, 1.0 );
    }
  `
};
const pass = new ShaderPass( myShader );
```

**WHY:** `ShaderPass` writes the read buffer texture into the uniform named by `textureID` (default `'tDiffuse'`). If that uniform does not exist, the shader receives no input from the previous pass and cannot process the scene image. The result is a solid color or garbage output.

**CORRECT:**
```js
const myShader = {
  uniforms: {
    tDiffuse: { value: null }, // ALWAYS include this
    amount: { value: 0.5 }
  },
  vertexShader: `...`,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform float amount;
    varying vec2 vUv;
    void main() {
      vec4 color = texture2D( tDiffuse, vUv );
      gl_FragColor = mix( color, vec4( 1.0 - color.rgb, color.a ), amount );
    }
  `
};
```

---

## Anti-Pattern 8: Using WebGL Passes with WebGPU Renderer

**WRONG:**
```js
import { WebGPURenderer } from 'three/addons/renderers/webgpu/WebGPURenderer.js';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';

const renderer = new WebGPURenderer();
const composer = new EffectComposer( renderer ); // INCOMPATIBLE
```

**WHY:** The built-in `EffectComposer` is designed exclusively for `WebGLRenderer`. It uses WebGL-specific render targets, framebuffers, and shader compilation. The WebGPU renderer has its own node-based `PostProcessing` class that uses TSL (Three Shading Language).

**CORRECT:**
```js
import { WebGPURenderer } from 'three/addons/renderers/webgpu/WebGPURenderer.js';
import { PostProcessing } from 'three/addons/tsl/display/PostProcessing.js';

const renderer = new WebGPURenderer();
const postProcessing = new PostProcessing( renderer );
```

---

## Anti-Pattern 9: Bloom Without Tone Mapping

**WRONG:**
```js
const renderer = new THREE.WebGLRenderer();
// renderer.toneMapping is THREE.NoToneMapping (default)

const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );
composer.addPass( new UnrealBloomPass( res, 1.5, 0.4, 0.85 ) );
composer.addPass( new OutputPass() );
```

**WHY:** Bloom relies on HDR values (colors above 1.0) to determine which areas glow. Without tone mapping, these HDR values are clamped to [0, 1] before bloom can detect them. The bloom effect appears extremely faint or invisible.

**CORRECT:**
```js
const renderer = new THREE.WebGLRenderer();
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
```

---

## Anti-Pattern 10: Not Disposing Post-Processing Resources

**WRONG:**
```js
// Switching scenes or removing post-processing
composer = new EffectComposer( renderer ); // old composer leaked
```

**WHY:** Each `EffectComposer` creates internal WebGL render targets and each pass may allocate additional GPU resources. Creating a new composer without disposing the old one leaks GPU memory. Over time, this causes performance degradation or crashes.

**CORRECT:**
```js
// Clean up before replacing
composer.dispose(); // frees all internal render targets and passes
composer = new EffectComposer( renderer );
```
