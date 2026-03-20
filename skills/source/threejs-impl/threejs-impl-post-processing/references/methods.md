# methods.md -- threejs-impl-post-processing

> Complete API signatures for Three.js post-processing classes.
> All imports from `three/addons/postprocessing/` unless otherwise noted.

---

## EffectComposer

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
```

### Constructor

```ts
new EffectComposer( renderer: WebGLRenderer, renderTarget?: WebGLRenderTarget )
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.passes` | `Pass[]` | `[]` | Ordered array of post-processing passes |
| `.readBuffer` | `WebGLRenderTarget` | auto | Internal read buffer |
| `.writeBuffer` | `WebGLRenderTarget` | auto | Internal write buffer |
| `.renderToScreen` | `boolean` | `true` | Final pass renders to screen |
| `.renderer` | `WebGLRenderer` | -- | The renderer instance |

### Methods

```ts
addPass( pass: Pass ): void
insertPass( pass: Pass, index: number ): void
removePass( pass: Pass ): void
render( deltaTime?: number ): void
setSize( width: number, height: number ): void
setPixelRatio( pixelRatio: number ): void
swapBuffers(): void
reset( renderTarget?: WebGLRenderTarget ): void
dispose(): void
```

---

## Pass (Base Class)

All passes extend the abstract `Pass` class.

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.enabled` | `boolean` | `true` | Whether this pass executes |
| `.needsSwap` | `boolean` | `true` | Whether to swap read/write buffers after rendering |
| `.clear` | `boolean` | `false` | Whether to clear the buffer before rendering |
| `.renderToScreen` | `boolean` | `false` | Whether this pass renders directly to screen |

### Methods

```ts
setSize( width: number, height: number ): void
dispose(): void
```

---

## RenderPass

```ts
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';

new RenderPass(
  scene: Scene,
  camera: Camera,
  overrideMaterial?: Material,
  clearColor?: Color,
  clearAlpha?: number
)
```

| Property | Type | Description |
|----------|------|-------------|
| `.scene` | `Scene` | The scene to render |
| `.camera` | `Camera` | The camera to render from |
| `.overrideMaterial` | `Material \| null` | Override material for all objects |
| `.clearColor` | `Color \| null` | Clear color override |
| `.clearAlpha` | `number` | Clear alpha override |
| `.clearDepth` | `boolean` | Whether to clear depth buffer |

---

## OutputPass

```ts
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

new OutputPass()
```

Applies the renderer's tone mapping and output color space conversion. ALWAYS the last pass.

---

## UnrealBloomPass

```ts
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

new UnrealBloomPass(
  resolution: Vector2,
  strength?: number,   // default 1
  radius?: number,     // default 0
  threshold?: number   // default 0
)
```

| Property | Type | Range | Description |
|----------|------|-------|-------------|
| `.strength` | `number` | `0` -- `3+` | Bloom intensity |
| `.radius` | `number` | `0` -- `1` | Bloom spread width |
| `.threshold` | `number` | `0` -- `1` | Luminance cutoff for bloom |
| `.resolution` | `Vector2` | -- | Resolution of bloom buffers |

---

## SSAOPass

```ts
import { SSAOPass } from 'three/addons/postprocessing/SSAOPass.js';

new SSAOPass(
  scene: Scene,
  camera: Camera,
  width?: number,
  height?: number,
  kernelSize?: number
)
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.kernelRadius` | `number` | `8` | AO spread radius |
| `.minDistance` | `number` | `0.005` | Minimum distance for AO |
| `.maxDistance` | `number` | `0.1` | Maximum distance for AO |
| `.output` | `number` | `SSAOPass.OUTPUT.Default` | Output mode |

Output modes: `SSAOPass.OUTPUT.Default`, `SSAOPass.OUTPUT.SSAO`, `SSAOPass.OUTPUT.Blur`, `SSAOPass.OUTPUT.Depth`, `SSAOPass.OUTPUT.Normal`.

---

## GTAOPass

```ts
import { GTAOPass } from 'three/addons/postprocessing/GTAOPass.js';

new GTAOPass(
  scene: Scene,
  camera: Camera,
  width?: number,
  height?: number
)
```

Higher quality than SSAOPass. Supports blend intensity and denoise settings.

---

## OutlinePass

```ts
import { OutlinePass } from 'three/addons/postprocessing/OutlinePass.js';

new OutlinePass(
  resolution: Vector2,
  scene: Scene,
  camera: Camera,
  selectedObjects?: Object3D[]
)
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.selectedObjects` | `Object3D[]` | `[]` | Objects to outline |
| `.edgeStrength` | `number` | `3` | Edge intensity |
| `.edgeGlow` | `number` | `0` | Animated glow amount |
| `.edgeThickness` | `number` | `1` | Outline width |
| `.visibleEdgeColor` | `Color` | `0xffffff` | Visible edge color |
| `.hiddenEdgeColor` | `Color` | `0x190a05` | Occluded edge color |
| `.pulsePeriod` | `number` | `0` | Pulse animation period (0 = off) |
| `.downSampleRatio` | `number` | `2` | Resolution downsampling |

---

## SMAAPass

```ts
import { SMAAPass } from 'three/addons/postprocessing/SMAAPass.js';

new SMAAPass( width: number, height: number )
```

Subpixel morphological anti-aliasing. Moderate quality and cost.

---

## FXAAPass

```ts
import { FXAAPass } from 'three/addons/postprocessing/FXAAPass.js';

new FXAAPass()
```

Fast approximate anti-aliasing. Lowest quality but cheapest. Access resolution uniform via `pass.material.uniforms[ 'resolution' ]`.

---

## ShaderPass

```ts
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';

new ShaderPass( shader: object, textureID?: string )
```

- `shader` -- object with `uniforms`, `vertexShader`, `fragmentShader`
- `textureID` -- name of the uniform that receives the read buffer (default `'tDiffuse'`)

ALWAYS include a `tDiffuse` uniform (or your custom `textureID` name) set to `{ value: null }`.

---

## BokehPass

```ts
import { BokehPass } from 'three/addons/postprocessing/BokehPass.js';

new BokehPass( scene: Scene, camera: Camera, params: {
  focus?: number,     // focus distance
  aperture?: number,  // aperture size
  maxblur?: number    // maximum blur amount
})
```

---

## FilmPass

```ts
import { FilmPass } from 'three/addons/postprocessing/FilmPass.js';

new FilmPass( intensity?: number, grayscale?: boolean )
```

---

## GlitchPass

```ts
import { GlitchPass } from 'three/addons/postprocessing/GlitchPass.js';

new GlitchPass( dtSize?: number )
```

| Property | Type | Description |
|----------|------|-------------|
| `.goWild` | `boolean` | Continuous glitch mode (default `false`) |

---

## LUTPass

```ts
import { LUTPass } from 'three/addons/postprocessing/LUTPass.js';

new LUTPass( params: { lut: DataTexture, intensity?: number } )
```

---

## MaskPass / ClearMaskPass

```ts
import { MaskPass } from 'three/addons/postprocessing/MaskPass.js';
import { ClearMaskPass } from 'three/addons/postprocessing/ClearMaskPass.js';

new MaskPass( scene: Scene, camera: Camera )
new ClearMaskPass()
```

Used for stencil-based masking. MaskPass enables the stencil, ClearMaskPass disables it.

---

## SSAARenderPass

```ts
import { SSAARenderPass } from 'three/addons/postprocessing/SSAARenderPass.js';

new SSAARenderPass( scene: Scene, camera: Camera, clearColor?: Color, clearAlpha?: number )
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `.sampleLevel` | `number` | `4` | Number of samples (2^n) |
| `.unbiased` | `boolean` | `true` | Use unbiased sampling |

---

## TAARenderPass

```ts
import { TAARenderPass } from 'three/addons/postprocessing/TAARenderPass.js';

new TAARenderPass( scene: Scene, camera: Camera, clearColor?: Color, clearAlpha?: number )
```

Extends SSAARenderPass with temporal accumulation. Only re-renders when camera moves.

---

## pmndrs/postprocessing -- Key Classes

```js
import { EffectComposer, EffectPass, RenderPass, BloomEffect,
         SMAAEffect, SSAOEffect, DepthOfFieldEffect, ToneMappingEffect,
         VignetteEffect, ChromaticAberrationEffect, NoiseEffect,
         GodRaysEffect } from 'postprocessing';
```

### EffectComposer (pmndrs)

```ts
new EffectComposer( renderer: WebGLRenderer, options?: {
  depthBuffer?: boolean,
  stencilBuffer?: boolean,
  alpha?: boolean,
  multisampling?: number,
  frameBufferType?: number
})
```

### EffectPass (pmndrs)

```ts
new EffectPass( camera: Camera, ...effects: Effect[] )
```

Merges multiple effects into a single shader pass. This is the key performance advantage over the built-in system.

### BloomEffect (pmndrs)

```ts
new BloomEffect( options?: {
  blendFunction?: BlendFunction,
  luminanceThreshold?: number,
  luminanceSmoothing?: number,
  mipmapBlur?: boolean,
  intensity?: number,
  radius?: number
})
```

### SMAAEffect (pmndrs)

```ts
new SMAAEffect( options?: {
  preset?: SMAAPreset,
  edgeDetectionMode?: EdgeDetectionMode
})
```

Presets: `SMAAPreset.LOW`, `SMAAPreset.MEDIUM`, `SMAAPreset.HIGH`, `SMAAPreset.ULTRA`.

---

## WebGPU PostProcessing

```ts
import { PostProcessing } from 'three/addons/tsl/display/PostProcessing.js';

new PostProcessing( renderer: WebGPURenderer )
```

Uses TSL (Three Shading Language) node graphs. Entirely separate API from WebGL post-processing.
