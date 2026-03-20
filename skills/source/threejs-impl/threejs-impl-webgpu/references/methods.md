# API Signatures Reference (Three.js WebGPU)

## WebGPURenderer

```javascript
import * as THREE from 'three/webgpu';

const renderer = new THREE.WebGPURenderer({
  canvas: HTMLCanvasElement,        // optional, target canvas
  antialias: boolean,               // default: false
  alpha: boolean,                   // default: false, transparent background
  depth: boolean,                   // default: true
  stencil: boolean,                 // default: false
  powerPreference: string,          // 'high-performance' | 'low-power'
  forceWebGL: boolean,              // default: false, force WebGL2 backend
});
```

### Instance Methods

```javascript
// Initialization — MUST await before rendering
await renderer.init(): Promise<void>

// Rendering
renderer.render(scene: Scene, camera: Camera): void
renderer.setAnimationLoop(callback: (time: DOMHighResTimeStamp) => void): void

// Sizing
renderer.setSize(width: number, height: number, updateStyle?: boolean): void
renderer.setPixelRatio(ratio: number): void
renderer.getSize(target: Vector2): Vector2

// Compute (WebGPU only)
await renderer.computeAsync(computeNode: ComputeNode): Promise<void>

// Disposal
renderer.dispose(): void
```

### Instance Properties

```javascript
renderer.domElement: HTMLCanvasElement  // the canvas element
renderer.info: Object                   // render statistics
renderer.toneMapping: ToneMapping       // default: NoToneMapping
renderer.toneMappingExposure: number    // default: 1
renderer.outputColorSpace: string       // default: SRGBColorSpace
renderer.shadowMap.enabled: boolean     // default: false
renderer.shadowMap.type: ShadowMapType  // default: PCFShadowMap
```

---

## WebGPU Feature Detection

```javascript
import { WebGPU } from 'three/webgpu';

WebGPU.isAvailable(): boolean
// Returns true if the browser supports WebGPU.
// Returns false if WebGPU is unavailable (use WebGLRenderer fallback).
```

---

## NodeMaterial Base Class

All node materials extend `NodeMaterial`. These properties are available on every node material type.

```javascript
// Common input nodes (all optional, accept TSL nodes)
material.colorNode: Node | null         // vec4 — base color
material.opacityNode: Node | null       // float — opacity
material.normalNode: Node | null        // vec3 — normal override
material.emissiveNode: Node | null      // color — emissive
material.positionNode: Node | null      // vec3 — vertex displacement
material.fragmentNode: Node | null      // vec4 — full fragment replacement
material.vertexNode: Node | null        // vec4 — full vertex replacement
material.outputNode: Node | null        // vec4 — final output override
material.aoNode: Node | null            // float — ambient occlusion
material.alphaTestNode: Node | null     // float — alpha test threshold
material.depthNode: Node | null         // float — custom depth
material.castShadowNode: Node | null    // vec4 — shadow casting
```

---

## MeshStandardNodeMaterial

```javascript
import { MeshStandardNodeMaterial } from 'three/webgpu';

const material = new MeshStandardNodeMaterial({
  color: 0xffffff,           // base color (classic prop, still works)
  metalness: 0.0,            // classic prop
  roughness: 1.0,            // classic prop
});

// Node inputs (override classic props)
material.metalnessNode: Node | null     // float
material.roughnessNode: Node | null     // float
material.envNode: Node | null           // color — environment map
material.lightsNode: Node | null        // lighting model override
```

---

## MeshPhysicalNodeMaterial

Extends `MeshStandardNodeMaterial` with additional physical inputs:

```javascript
import { MeshPhysicalNodeMaterial } from 'three/webgpu';

material.clearcoatNode: Node | null
material.clearcoatRoughnessNode: Node | null
material.clearcoatNormalNode: Node | null
material.sheenNode: Node | null
material.iridescenceNode: Node | null
material.iridescenceIORNode: Node | null
material.iridescenceThicknessNode: Node | null
material.specularIntensityNode: Node | null
material.specularColorNode: Node | null
material.iorNode: Node | null
material.transmissionNode: Node | null
material.thicknessNode: Node | null
material.attenuationDistanceNode: Node | null
material.attenuationColorNode: Node | null
material.dispersionNode: Node | null
material.anisotropyNode: Node | null
```

---

## TSL Core Functions

### Type Constructors

```javascript
import {
  float, int, uint, bool,
  vec2, vec3, vec4,
  ivec2, ivec3, ivec4,
  uvec2, uvec3, uvec4,
  mat2, mat3, mat4,
  color
} from 'three/tsl';

float(1.0): FloatNode
vec3(1.0, 0.0, 0.0): Vec3Node
color(0xff0000): ColorNode
```

### Variable Management

```javascript
import { uniform, toVar, toConst, varying, vertexStage, attribute } from 'three/tsl';

uniform(initialValue): UniformNode
// .onRenderUpdate((frame) => newValue)
// .onFrameUpdate((frame) => newValue)
// .onObjectUpdate((frame) => newValue)

toVar(node): VarNode           // reusable shader variable
toConst(node): ConstNode       // inline constant
varying(node): VaryingNode     // vertex-to-fragment interpolation
vertexStage(node): Node        // force vertex shader execution
attribute(name, type): AttributeNode  // buffer attribute access
```

### Math Library

```javascript
import {
  abs, acos, asin, atan, ceil, clamp, cos, cross, degrees,
  distance, dot, exp, floor, fract, inverseSqrt, length, log,
  max, min, mix, normalize, pow, radians, reflect, refract,
  round, saturate, sign, sin, smoothstep, sqrt, step, tan, trunc,
  faceforward, dFdx, dFdy, fwidth,
  negate, oneMinus, reciprocal, cbrt, pow2, pow3, pow4
} from 'three/tsl';

// Constants
import { EPSILON, INFINITY, PI, TWO_PI, HALF_PI } from 'three/tsl';
```

### Geometry Nodes

```javascript
import {
  positionGeometry, positionLocal, positionWorld, positionView,
  positionWorldDirection, positionViewDirection,
  normalGeometry, normalLocal, normalView, normalWorld,
  tangentGeometry, tangentLocal, tangentView, tangentWorld,
  bitangentGeometry, bitangentLocal, bitangentView, bitangentWorld,
  uv,
  screenUV, screenCoordinate, screenSize,
  viewportUV, viewportCoordinate, viewportSize
} from 'three/tsl';

uv(0): UVNode  // UV channel 0 (default)
uv(1): UVNode  // UV channel 1
```

### Camera Nodes

```javascript
import {
  cameraNear, cameraFar, cameraPosition,
  cameraProjectionMatrix, cameraProjectionMatrixInverse,
  cameraViewMatrix, cameraWorldMatrix, cameraNormalMatrix
} from 'three/tsl';
```

### Model Nodes

```javascript
import {
  modelViewMatrix, modelNormalMatrix, modelWorldMatrix,
  modelPosition, modelScale, modelDirection,
  modelViewPosition, modelWorldMatrixInverse
} from 'three/tsl';
```

### Texture Functions

```javascript
import { texture, textureLoad, textureStore, textureSize, cubeTexture, triplanarTexture, textureBicubic } from 'three/tsl';

texture(tex, uv?, level?): TextureNode
textureLoad(tex, uv, level?): TextureNode
textureStore(tex, uv, value): StorageTextureNode
textureSize(tex, level?): Vec2Node
cubeTexture(tex, uvw?, level?): CubeTextureNode
triplanarTexture(texX, texY, texZ, scale?, position?, normal?): Node
textureBicubic(textureNode, strength?): Node
```

### Animation Nodes

```javascript
import { time, deltaTime, oscSine, oscSquare, oscTriangle, oscSawtooth } from 'three/tsl';

time: FloatNode                    // elapsed seconds
deltaTime: FloatNode               // frame delta seconds
oscSine(timer?): FloatNode         // sine oscillation 0-1
oscSquare(timer?): FloatNode       // square wave 0-1
oscTriangle(timer?): FloatNode     // triangle wave 0-1
oscSawtooth(timer?): FloatNode     // sawtooth wave 0-1
```

### Randomization

```javascript
import { hash, range } from 'three/tsl';

hash(seed): FloatNode              // deterministic 0-1 hash
range(min, max): FloatNode         // attribute-based random range
```

### Control Flow

```javascript
import { If, select, Loop, Break, Continue, Discard, Return, Switch, Fn } from 'three/tsl';

If(condition, thenFn).ElseIf(condition, thenFn).Else(elseFn): void
select(condition, trueVal, falseVal): Node
Loop(count, ({ i }) => { }): void
Switch(value).Case(val, fn).Default(fn): void
Break(): void
Continue(): void
Discard(): void
Return(): void

const myFn = Fn(([param1, param2]) => {
  return param1.add(param2);
});
```

### Color Operations

```javascript
import { luminance, saturation, vibrance, hue, posterize, grayscale, sepia } from 'three/tsl';
```

### Blend Modes

```javascript
import { blendBurn, blendDodge, blendOverlay, blendScreen, blendColor } from 'three/tsl';
```

---

## Compute Shader API

```javascript
import { compute, storage, storageTexture } from 'three/tsl';

compute(shaderFn: Fn, count: number, workgroupSize?: number[]): ComputeNode
storage(attribute, type, count): StorageBufferNode
storageTexture(texture): StorageTextureNode

// Dispatch
await renderer.computeAsync(computeNode): Promise<void>
```

### Atomic Operations

```javascript
import {
  atomicAdd, atomicSub, atomicMax, atomicMin,
  atomicAnd, atomicOr, atomicXor,
  atomicStore, atomicLoad
} from 'three/tsl';
```

### Barriers

```javascript
import { workgroupBarrier, storageBarrier, textureBarrier, barrier } from 'three/tsl';
```

### Built-in IDs

```javascript
import { workgroupId, localId, globalId, numWorkgroups, subgroupSize } from 'three/tsl';
```

---

## PostProcessing Class

```javascript
import { PostProcessing } from 'three/webgpu';
import { bloom, fxaa, renderOutput } from 'three/tsl';

const postProcessing = new PostProcessing(renderer: WebGPURenderer);
postProcessing.outputNode = Node;  // assign TSL post-processing chain

// Available post-processing nodes:
import {
  bloom, dof, fxaa, smaa, gaussianBlur, ssr, ssgi, ao,
  chromaticAberration, film, dotScreen, sobel, afterImage,
  anamorphic, denoise, lut3D, motionBlur, outline, rgbShift,
  transition, traa, renderOutput
} from 'three/tsl';
```
