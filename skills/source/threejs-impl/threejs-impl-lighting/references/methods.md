# threejs-impl-lighting — Methods Reference

> Complete API signatures for all light types, helpers, and PMREMGenerator in Three.js r160+.

---

## Light Base Class

```typescript
// Abstract base — do NOT instantiate directly
class Light extends Object3D {
  color: Color;                    // Default: 0xffffff
  intensity: number;               // Default: 1
  readonly isLight: boolean;       // ALWAYS true
  dispose(): void;                 // Release GPU resources
}
```

---

## AmbientLight

```typescript
class AmbientLight extends Light {
  constructor(color?: ColorRepresentation, intensity?: number);
  // color default: 0xffffff
  // intensity default: 1 (unitless multiplier)
  readonly isAmbientLight: boolean; // ALWAYS true
}
```

---

## HemisphereLight

```typescript
class HemisphereLight extends Light {
  constructor(
    skyColor?: ColorRepresentation,     // Default: 0xffffff
    groundColor?: ColorRepresentation,  // Default: 0xffffff
    intensity?: number                  // Default: 1 (unitless)
  );
  groundColor: Color;
  readonly isHemisphereLight: boolean;  // ALWAYS true
}
```

---

## DirectionalLight

```typescript
class DirectionalLight extends Light {
  constructor(
    color?: ColorRepresentation,  // Default: 0xffffff
    intensity?: number            // Default: 1 (lux in r160+)
  );
  shadow: DirectionalLightShadow;
  target: Object3D;               // MUST add to scene for repositioning
  readonly isDirectionalLight: boolean; // ALWAYS true
}
```

**Intensity unit:** Lux (lm/m2). Outdoor sunlight = 50,000-100,000 lux.

---

## PointLight

```typescript
class PointLight extends Light {
  constructor(
    color?: ColorRepresentation,  // Default: 0xffffff
    intensity?: number,           // Default: 1 (candela in r160+)
    distance?: number,            // Default: 0 (infinite, inverse-square)
    decay?: number                // Default: 2 (physically correct)
  );
  decay: number;                  // 2 = inverse-square law
  distance: number;               // 0 = no limit
  power: number;                  // Lumens = intensity * 4 * Math.PI
  shadow: PointLightShadow;
  readonly isPointLight: boolean; // ALWAYS true
}
```

**Distance behavior:**
- `distance === 0`: Inverse-square falloff, infinite range
- `distance > 0`: Smooth attenuation to zero at cutoff (NOT physically correct, artistic control)

---

## SpotLight

```typescript
class SpotLight extends Light {
  constructor(
    color?: ColorRepresentation,  // Default: 0xffffff
    intensity?: number,           // Default: 1 (candela in r160+)
    distance?: number,            // Default: 0 (infinite)
    angle?: number,               // Default: Math.PI / 3, max Math.PI / 2
    penumbra?: number,            // Default: 0 (sharp edge), range [0, 1]
    decay?: number                // Default: 2
  );
  angle: number;                  // Cone half-angle, capped at Math.PI / 2
  decay: number;
  distance: number;
  map: Texture | null;            // Cookie texture, REQUIRES castShadow = true
  penumbra: number;               // Edge softness [0, 1]
  power: number;                  // Lumens = intensity * Math.PI
  shadow: SpotLightShadow;
  target: Object3D;               // MUST add to scene for repositioning
  readonly isSpotLight: boolean;  // ALWAYS true
}
```

---

## RectAreaLight

```typescript
class RectAreaLight extends Light {
  constructor(
    color?: ColorRepresentation,  // Default: 0xffffff
    intensity?: number,           // Default: 1 (nits = cd/m2 in r160+)
    width?: number,               // Default: 10
    height?: number               // Default: 10
  );
  width: number;
  height: number;
  power: number;                  // Lumens
  readonly isRectAreaLight: boolean; // ALWAYS true
}
```

**Requirements:**
- WebGLRenderer: MUST call `RectAreaLightUniformsLib.init()` before use
- WebGPURenderer: MUST use `RectAreaLightTexturesLib` instead
- Works ONLY with `MeshStandardMaterial` and `MeshPhysicalMaterial`
- Orient using `.position.set()` and `.lookAt()`, NOT rotation
- CANNOT cast shadows

---

## LightProbe

```typescript
class LightProbe extends Light {
  constructor(
    sh?: SphericalHarmonics3,     // Spherical harmonics data
    intensity?: number            // Default: 1
  );
  readonly isLightProbe: boolean; // ALWAYS true
}
```

### LightProbeGenerator (addon)

```typescript
// Import: 'three/addons/lights/LightProbeGenerator.js'
class LightProbeGenerator {
  static fromCubeTexture(cubeTexture: CubeTexture): LightProbe;
  static fromCubeRenderTarget(renderer: WebGLRenderer, target: WebGLCubeRenderTarget): LightProbe;
}
```

---

## RectAreaLightUniformsLib (addon)

```typescript
// Import: 'three/addons/lights/RectAreaLightUniformsLib.js'
class RectAreaLightUniformsLib {
  static init(): void;  // MUST call once before creating any RectAreaLight
}
```

---

## PMREMGenerator

```typescript
class PMREMGenerator {
  constructor(renderer: WebGLRenderer);

  fromEquirectangular(
    equirectangular: Texture,
    renderTarget?: WebGLRenderTarget
  ): WebGLRenderTarget;
  // Returns render target with .texture for scene.environment

  fromScene(
    scene: Scene,
    sigma?: number,               // Blur sigma, default 0
    near?: number,                // Default 0.1
    far?: number                  // Default 100
  ): WebGLRenderTarget;

  fromCubemap(
    cubemap: CubeTexture,
    renderTarget?: WebGLRenderTarget
  ): WebGLRenderTarget;

  compileCubemapShader(): void;   // Pre-compile for faster first use
  compileEquirectangularShader(): void;

  dispose(): void;                // ALWAYS call after generating env maps
}
```

---

## Light Helpers

### DirectionalLightHelper

```typescript
// Import: 'three'
class DirectionalLightHelper extends Object3D {
  constructor(
    light: DirectionalLight,
    size?: number,                // Default: 1
    color?: ColorRepresentation   // Default: light.color
  );
  light: DirectionalLight;
  update(): void;                 // Call after changing light properties
  dispose(): void;
}
```

### SpotLightHelper

```typescript
// Import: 'three'
class SpotLightHelper extends Object3D {
  constructor(
    light: SpotLight,
    color?: ColorRepresentation   // Default: light.color
  );
  light: SpotLight;
  update(): void;
  dispose(): void;
}
```

### PointLightHelper

```typescript
// Import: 'three'
class PointLightHelper extends Mesh {
  constructor(
    light: PointLight,
    sphereSize?: number,          // Default: 1
    color?: ColorRepresentation   // Default: light.color
  );
  light: PointLight;
  update(): void;
  dispose(): void;
}
```

### HemisphereLightHelper

```typescript
// Import: 'three'
class HemisphereLightHelper extends Object3D {
  constructor(
    light: HemisphereLight,
    size: number,                 // REQUIRED — no default
    color?: ColorRepresentation
  );
  light: HemisphereLight;
  update(): void;
  dispose(): void;
}
```

### RectAreaLightHelper (addon)

```typescript
// Import: 'three/addons/helpers/RectAreaLightHelper.js'
class RectAreaLightHelper extends Line {
  constructor(light: RectAreaLight);
  dispose(): void;
}
```

### LightProbeHelper (addon)

```typescript
// Import: 'three/addons/helpers/LightProbeHelper.js'
class LightProbeHelper extends Mesh {
  constructor(
    lightProbe: LightProbe,
    size: number                  // REQUIRED — no default
  );
  dispose(): void;
}
```

---

## Scene Environment Properties

```typescript
class Scene extends Object3D {
  environment: Texture | null;           // IBL for all PBR materials
  background: Color | Texture | null;    // Visible backdrop
  backgroundBlurriness: number;          // 0-1, blur background env map
  backgroundIntensity: number;           // Background brightness multiplier
  environmentIntensity: number;          // IBL contribution multiplier
  environmentRotation: Euler;            // Rotate the environment map
}
```

---

## RGBELoader (addon)

```typescript
// Import: 'three/addons/loaders/RGBELoader.js'
class RGBELoader extends DataTextureLoader {
  load(
    url: string,
    onLoad?: (texture: DataTexture) => void,
    onProgress?: (event: ProgressEvent) => void,
    onError?: (event: ErrorEvent) => void
  ): DataTexture;
  loadAsync(url: string, onProgress?: Function): Promise<DataTexture>;
  setDataType(type: number): this;  // HalfFloatType recommended
}
```

## EXRLoader (addon)

```typescript
// Import: 'three/addons/loaders/EXRLoader.js'
class EXRLoader extends DataTextureLoader {
  load(
    url: string,
    onLoad?: (texture: DataTexture) => void,
    onProgress?: (event: ProgressEvent) => void,
    onError?: (event: ErrorEvent) => void
  ): DataTexture;
  loadAsync(url: string, onProgress?: Function): Promise<DataTexture>;
  setDataType(type: number): this;
}
```
