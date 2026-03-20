# threejs-core-renderer — Method Reference

> Complete API signatures for WebGLRenderer, PerspectiveCamera, OrthographicCamera, and related classes.
> All signatures verified against Three.js r160+ official documentation.

---

## WebGLRenderer

### Constructor

```typescript
new WebGLRenderer(parameters?: {
  canvas?: HTMLCanvasElement;
  context?: WebGLRenderingContext;
  precision?: 'highp' | 'mediump' | 'lowp';
  alpha?: boolean;
  premultipliedAlpha?: boolean;
  antialias?: boolean;
  stencil?: boolean;
  preserveDrawingBuffer?: boolean;
  powerPreference?: 'high-performance' | 'low-power' | 'default';
  failIfMajorPerformanceCaveat?: boolean;
  depth?: boolean;
  logarithmicDepthBuffer?: boolean;
})
```

### Properties

| Property | Type | Default | Mutable | Description |
|----------|------|---------|---------|-------------|
| `domElement` | `HTMLCanvasElement` | -- | Read-only | The canvas element |
| `autoClear` | `boolean` | `true` | Yes | Auto-clear before render |
| `autoClearColor` | `boolean` | `true` | Yes | Auto-clear color buffer |
| `autoClearDepth` | `boolean` | `true` | Yes | Auto-clear depth buffer |
| `autoClearStencil` | `boolean` | `true` | Yes | Auto-clear stencil buffer |
| `sortObjects` | `boolean` | `true` | Yes | Auto-sort draw order |
| `clippingPlanes` | `Plane[]` | `[]` | Yes | Global clipping planes |
| `localClippingEnabled` | `boolean` | `false` | Yes | Enable per-material clipping |
| `outputColorSpace` | `string` | `SRGBColorSpace` | Yes | Output color space |
| `toneMapping` | `ToneMapping` | `NoToneMapping` | Yes | Tone mapping algorithm |
| `toneMappingExposure` | `number` | `1` | Yes | Tone mapping exposure |
| `shadowMap` | `WebGLShadowMap` | -- | Read-only | Shadow map config object |
| `shadowMap.enabled` | `boolean` | `false` | Yes | Enable shadow mapping |
| `shadowMap.type` | `ShadowMapType` | `PCFShadowMap` | Init only | Shadow map type |
| `info` | `object` | -- | Read-only | Render statistics |
| `info.render.calls` | `number` | -- | Read-only | Draw calls per frame |
| `info.render.triangles` | `number` | -- | Read-only | Triangles per frame |
| `info.render.points` | `number` | -- | Read-only | Points per frame |
| `info.render.lines` | `number` | -- | Read-only | Lines per frame |
| `info.memory.geometries` | `number` | -- | Read-only | Cached geometries |
| `info.memory.textures` | `number` | -- | Read-only | Cached textures |
| `capabilities` | `object` | -- | Read-only | WebGL capabilities |
| `capabilities.maxTextures` | `number` | -- | Read-only | Max texture units |
| `capabilities.maxVertexTextures` | `number` | -- | Read-only | Max vertex texture units |
| `capabilities.precision` | `string` | -- | Read-only | Actual shader precision |
| `xr` | `WebXRManager` | -- | Read-only | WebXR session manager |

### Rendering Methods

```typescript
render(scene: Scene, camera: Camera): void
```
Renders the scene using the given camera. ALWAYS call inside `setAnimationLoop` callback or after `setRenderTarget`.

```typescript
setAnimationLoop(callback: ((time: DOMHighResTimeStamp) => void) | null): void
```
Sets the animation loop function. Pass `null` to stop. Handles WebXR sessions automatically. ALWAYS prefer over raw `requestAnimationFrame`.

```typescript
compile(scene: Scene, camera: Camera): Set<Material>
```
Synchronously compiles all shaders in the scene. Returns the set of compiled materials. Blocks the main thread.

```typescript
compileAsync(scene: Scene, camera: Camera): Promise<void>
```
Asynchronously compiles all shaders. Returns a Promise. ALWAYS call after asset loading, before first visible render. (r160+)

### Size and Pixel Ratio Methods

```typescript
setSize(width: number, height: number, updateStyle?: boolean): void
```
Sets the output canvas size. `updateStyle` (default `true`) controls whether the CSS `width`/`height` style attributes are updated. Set `updateStyle: false` when the canvas is sized by CSS.

```typescript
setPixelRatio(value: number): void
```
Sets the device pixel ratio. ALWAYS pass `Math.min(window.devicePixelRatio, 2)`.

```typescript
getPixelRatio(): number
```
Returns the current pixel ratio.

```typescript
getSize(target: Vector2): Vector2
```
Returns the current output canvas size (in CSS pixels) into the target Vector2.

### Clear Methods

```typescript
setClearColor(color: Color | string | number, alpha?: number): void
```
Sets the clear color. `alpha` defaults to `1`.

```typescript
getClearColor(target: Color): Color
```
Writes the current clear color into `target`.

```typescript
getClearAlpha(): number
```
Returns the current clear alpha.

```typescript
clear(color?: boolean, depth?: boolean, stencil?: boolean): void
```
Manually clears the buffers. All parameters default to `true`.

### Render Target Methods

```typescript
setRenderTarget(
  renderTarget: WebGLRenderTarget | null,
  activeCubeFace?: number,
  activeMipmapLevel?: number
): void
```
Redirects rendering to the given render target. Pass `null` to restore the default framebuffer. `activeCubeFace` is for cube render targets (0-5). `activeMipmapLevel` is for mipmap-level rendering.

```typescript
readRenderTargetPixels(
  renderTarget: WebGLRenderTarget,
  x: number, y: number,
  width: number, height: number,
  buffer: TypedArray
): void
```
Reads pixel data from a render target into a typed array buffer. The buffer size MUST be `width * height * 4` for RGBA format.

### Viewport and Scissor Methods

```typescript
setViewport(x: number, y: number, width: number, height: number): void
setViewport(v: Vector4): void
```
Sets the viewport region. Coordinates are in pixels from the bottom-left of the canvas.

```typescript
setScissor(x: number, y: number, width: number, height: number): void
setScissor(v: Vector4): void
```
Sets the scissor region. Only pixels within this rectangle are affected by rendering.

```typescript
setScissorTest(enable: boolean): void
```
Enables or disables the scissor test.

### Context and Disposal

```typescript
getContext(): WebGLRenderingContext
```
Returns the underlying WebGL context.

```typescript
dispose(): void
```
Releases the WebGL context and all associated resources. ALWAYS call on cleanup.

---

## WebGLRenderTarget

### Constructor

```typescript
new WebGLRenderTarget(width: number, height: number, options?: {
  minFilter?: TextureFilter;       // Default: LinearFilter
  magFilter?: TextureFilter;       // Default: LinearFilter
  format?: PixelFormat;            // Default: RGBAFormat
  type?: TextureDataType;          // Default: UnsignedByteType
  stencilBuffer?: boolean;         // Default: false
  depthBuffer?: boolean;           // Default: true
  samples?: number;                // MSAA samples. Default: 0
  colorSpace?: string;             // Default: '' (NoColorSpace)
  depthTexture?: DepthTexture;     // Optional depth texture attachment
})
```

### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `texture` | `Texture` | The color attachment texture |
| `depthTexture` | `DepthTexture \| null` | Optional depth texture |
| `width` | `number` | Render target width in pixels |
| `height` | `number` | Render target height in pixels |
| `samples` | `number` | MSAA sample count |
| `scissor` | `Vector4` | Scissor rectangle |
| `scissorTest` | `boolean` | Scissor test state |
| `viewport` | `Vector4` | Viewport rectangle |

### Methods

```typescript
setSize(width: number, height: number): void
```
Resizes the render target. ALWAYS call when the output size changes.

```typescript
clone(): WebGLRenderTarget
```
Returns a copy.

```typescript
dispose(): void
```
Releases GPU resources. ALWAYS call when no longer needed.

---

## WebGLCubeRenderTarget

### Constructor

```typescript
new WebGLCubeRenderTarget(size: number, options?: WebGLRenderTargetOptions)
```

Used with `CubeCamera` for dynamic environment maps. `size` is the resolution per face.

---

## PerspectiveCamera

### Constructor

```typescript
new PerspectiveCamera(
  fov?: number,    // Vertical FOV in degrees. Default: 50
  aspect?: number, // Width / height. Default: 1
  near?: number,   // Near plane. Default: 0.1
  far?: number     // Far plane. Default: 2000
)
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `fov` | `number` | `50` | Vertical field of view in degrees |
| `aspect` | `number` | `1` | Aspect ratio (width / height) |
| `near` | `number` | `0.1` | Near clipping plane |
| `far` | `number` | `2000` | Far clipping plane |
| `zoom` | `number` | `1` | Zoom factor (>1 zooms in) |
| `filmGauge` | `number` | `35` | Film gauge in mm |
| `filmOffset` | `number` | `0` | Horizontal off-center offset in mm |
| `focus` | `number` | `10` | Focus distance for stereo rendering |
| `view` | `object \| null` | `null` | Sub-frustum (set by setViewOffset) |

### Methods

```typescript
updateProjectionMatrix(): void
```
Recomputes the projection matrix. MUST call after modifying `fov`, `aspect`, `near`, `far`, or `zoom`.

```typescript
setViewOffset(
  fullWidth: number, fullHeight: number,
  x: number, y: number,
  width: number, height: number
): void
```
Sets a view sub-frustum for multi-monitor/tiled rendering.

```typescript
clearViewOffset(): void
```
Removes the view offset.

```typescript
getEffectiveFOV(): number
```
Returns the actual FOV accounting for `zoom`.

```typescript
getFilmWidth(): number
getFilmHeight(): number
```
Returns effective film dimensions in mm.

---

## OrthographicCamera

### Constructor

```typescript
new OrthographicCamera(
  left: number,
  right: number,
  top: number,
  bottom: number,
  near?: number,  // Default: 0.1
  far?: number    // Default: 2000
)
```

### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `left` | `number` | -- | Left frustum plane |
| `right` | `number` | -- | Right frustum plane |
| `top` | `number` | -- | Top frustum plane |
| `bottom` | `number` | -- | Bottom frustum plane |
| `near` | `number` | `0.1` | Near clipping plane |
| `far` | `number` | `2000` | Far clipping plane |
| `zoom` | `number` | `1` | Zoom factor (>1 zooms in) |

### Methods

```typescript
updateProjectionMatrix(): void
```
MUST call after modifying any frustum parameter or `zoom`.

```typescript
setViewOffset(
  fullWidth: number, fullHeight: number,
  x: number, y: number,
  width: number, height: number
): void
clearViewOffset(): void
```

---

## ArrayCamera

### Constructor

```typescript
new ArrayCamera(cameras?: PerspectiveCamera[])
```

Each sub-camera MUST have `camera.viewport` set to a `Vector4(x, y, width, height)` defining its render region.

---

## CubeCamera

### Constructor

```typescript
new CubeCamera(near: number, far: number, renderTarget: WebGLCubeRenderTarget)
```

### Methods

```typescript
update(renderer: WebGLRenderer, scene: Scene): void
```
Renders the scene from all 6 directions into the cube render target. This calls `renderer.render()` 6 times. NEVER call every frame for static environments.

---

## Tone Mapping Constants

```typescript
import {
  NoToneMapping,
  LinearToneMapping,
  ReinhardToneMapping,
  CineonToneMapping,
  ACESFilmicToneMapping,
  AgXToneMapping,        // r160+
  NeutralToneMapping     // r160+
} from 'three';
```

## Shadow Map Type Constants

```typescript
import {
  BasicShadowMap,
  PCFShadowMap,
  PCFSoftShadowMap,
  VSMShadowMap
} from 'three';
```

## Color Space Constants

```typescript
import {
  SRGBColorSpace,
  LinearSRGBColorSpace,
  NoColorSpace
} from 'three';
```
