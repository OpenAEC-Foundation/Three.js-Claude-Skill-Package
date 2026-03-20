# Material Types & API Reference (Three.js r160+)

## Base Material

All materials extend `Material`. These methods and properties are available on every material type.

```typescript
class Material {
  // Properties
  side: number;                    // FrontSide | BackSide | DoubleSide
  transparent: boolean;            // Enable alpha blending
  opacity: number;                 // 0-1 (requires transparent: true)
  depthWrite: boolean;             // Write to depth buffer
  depthTest: boolean;              // Test against depth buffer
  depthFunc: number;               // LessEqualDepth (default)
  blending: number;                // NormalBlending | AdditiveBlending | SubtractiveBlending | MultiplyBlending | CustomBlending | NoBlending
  blendSrc: number;                // Source blend factor (CustomBlending)
  blendDst: number;                // Destination blend factor (CustomBlending)
  blendEquation: number;           // AddEquation (default)
  alphaTest: number;               // Discard fragments below this alpha (0-1)
  alphaToCoverage: boolean;        // Alpha-to-coverage (MSAA only)
  clippingPlanes: Plane[] | null;  // Clipping planes
  clipIntersection: boolean;       // Clip ALL planes intersection vs union
  clipShadows: boolean;            // Apply clipping to shadows
  colorWrite: boolean;             // Write color
  stencilWrite: boolean;           // Enable stencil buffer writing
  stencilFunc: number;             // Stencil comparison function
  stencilRef: number;              // Stencil reference value
  stencilWriteMask: number;        // Stencil write bitmask
  stencilFuncMask: number;         // Stencil function bitmask
  stencilFail: number;             // Stencil fail operation
  stencilZFail: number;            // Stencil depth fail operation
  stencilZPass: number;            // Stencil depth pass operation
  polygonOffset: boolean;          // Enable polygon offset (decals, coplanar)
  polygonOffsetFactor: number;     // Polygon offset factor
  polygonOffsetUnits: number;      // Polygon offset units
  visible: boolean;                // Render this material
  toneMapped: boolean;             // Apply renderer tone mapping
  needsUpdate: boolean;            // Trigger shader recompilation
  version: number;                 // Auto-incremented on changes

  // Methods
  clone(): Material;
  copy(source: Material): Material;
  dispose(): void;
  onBeforeCompile(shader: Object, renderer: WebGLRenderer): void;
  setValues(values: Object): void;
  toJSON(): Object;
}
```

---

## MeshBasicMaterial

No lighting calculations. Renders flat color or texture.

```typescript
new MeshBasicMaterial({
  color: Color | number;            // Default: 0xffffff
  map: Texture | null;              // Diffuse texture
  wireframe: boolean;               // Default: false
  wireframeLinewidth: number;       // Default: 1
  combine: number;                  // MultiplyOperation | MixOperation | AddOperation
  reflectivity: number;             // 0-1, env map reflectivity
  envMap: Texture | null;           // Environment map
  fog: boolean;                     // Default: true
  alphaMap: Texture | null;         // Alpha transparency map
  aoMap: Texture | null;            // Ambient occlusion map (requires uv2)
  aoMapIntensity: number;           // Default: 1.0
  lightMap: Texture | null;         // Baked light map (requires uv2)
  lightMapIntensity: number;        // Default: 1.0
});
```

---

## MeshLambertMaterial

Diffuse-only Lambertian shading. Cheapest lit material -- no specular highlights.

```typescript
new MeshLambertMaterial({
  color: Color | number;            // Default: 0xffffff
  emissive: Color | number;         // Default: 0x000000
  emissiveIntensity: number;        // Default: 1.0
  emissiveMap: Texture | null;
  map: Texture | null;
  bumpMap: Texture | null;
  bumpScale: number;                // Default: 1
  normalMap: Texture | null;
  normalMapType: number;            // TangentSpaceNormalMap (default)
  normalScale: Vector2;             // Default: (1, 1)
  displacementMap: Texture | null;
  displacementScale: number;        // Default: 1
  displacementBias: number;         // Default: 0
  alphaMap: Texture | null;
  envMap: Texture | null;
  combine: number;                  // MultiplyOperation (default)
  reflectivity: number;             // Default: 1
  fog: boolean;                     // Default: true
  wireframe: boolean;               // Default: false
});
```

---

## MeshPhongMaterial

Blinn-Phong model with specular highlights. Cheaper than PBR but not physically correct.

```typescript
new MeshPhongMaterial({
  color: Color | number;            // Default: 0xffffff
  specular: Color | number;         // Default: 0x111111, specular highlight color
  shininess: number;                // Default: 30, higher = tighter highlight
  emissive: Color | number;         // Default: 0x000000
  emissiveIntensity: number;        // Default: 1.0
  map: Texture | null;
  specularMap: Texture | null;      // Specular intensity map
  bumpMap: Texture | null;
  normalMap: Texture | null;
  displacementMap: Texture | null;
  envMap: Texture | null;
  combine: number;                  // MultiplyOperation (default)
  reflectivity: number;             // Default: 1
  fog: boolean;                     // Default: true
  flatShading: boolean;             // Default: false
  wireframe: boolean;               // Default: false
});
```

---

## MeshStandardMaterial

PBR metalness/roughness workflow. Recommended material for most use cases.

```typescript
new MeshStandardMaterial({
  color: Color | number;            // Default: 0xffffff
  roughness: number;                // 0 (mirror) to 1 (fully rough). Default: 1.0
  metalness: number;                // 0 (dielectric) to 1 (metal). Default: 0.0
  map: Texture | null;              // Diffuse/albedo
  roughnessMap: Texture | null;     // Per-pixel roughness (G channel)
  metalnessMap: Texture | null;     // Per-pixel metalness (B channel)
  normalMap: Texture | null;        // Surface normal perturbation
  normalMapType: number;            // TangentSpaceNormalMap (default)
  normalScale: Vector2;             // Default: (1, 1)
  bumpMap: Texture | null;          // Grayscale height map
  bumpScale: number;                // Default: 1.0
  displacementMap: Texture | null;  // Vertex displacement
  displacementScale: number;        // Default: 1.0
  displacementBias: number;         // Default: 0.0
  aoMap: Texture | null;            // Ambient occlusion (R channel, requires uv2)
  aoMapIntensity: number;           // Default: 1.0
  emissive: Color | number;         // Default: 0x000000
  emissiveMap: Texture | null;
  emissiveIntensity: number;        // Default: 1.0
  envMap: Texture | null;           // Environment reflection
  envMapIntensity: number;          // Default: 1.0
  alphaMap: Texture | null;         // Per-pixel transparency (R channel)
  lightMap: Texture | null;         // Baked lighting (requires uv2)
  lightMapIntensity: number;        // Default: 1.0
  flatShading: boolean;             // Default: false
  wireframe: boolean;               // Default: false
  wireframeLinewidth: number;       // Default: 1
  fog: boolean;                     // Default: true
});
```

---

## MeshPhysicalMaterial

Extends MeshStandardMaterial with ALL its properties, plus advanced PBR features.

```typescript
new MeshPhysicalMaterial({
  // ALL MeshStandardMaterial properties, plus:

  // Clearcoat
  clearcoat: number;                    // 0-1. Default: 0.0
  clearcoatRoughness: number;           // 0-1. Default: 0.0
  clearcoatMap: Texture | null;
  clearcoatRoughnessMap: Texture | null;
  clearcoatNormalMap: Texture | null;
  clearcoatNormalScale: Vector2;        // Default: (1, 1)

  // Transmission (glass)
  transmission: number;                 // 0-1. Default: 0.0
  transmissionMap: Texture | null;
  thickness: number;                    // Default: 0.0
  thicknessMap: Texture | null;
  ior: number;                          // 1.0-2.333. Default: 1.5
  attenuationDistance: number;           // Default: Infinity
  attenuationColor: Color;              // Default: white

  // Sheen (fabric)
  sheen: number;                        // 0-1. Default: 0.0
  sheenColor: Color;                    // Default: 0x000000
  sheenColorMap: Texture | null;
  sheenRoughness: number;               // 0-1. Default: 1.0
  sheenRoughnessMap: Texture | null;

  // Iridescence (thin-film)
  iridescence: number;                  // 0-1. Default: 0.0
  iridescenceIOR: number;               // Default: 1.3
  iridescenceThicknessRange: [number, number]; // Default: [100, 400] nm
  iridescenceMap: Texture | null;
  iridescenceThicknessMap: Texture | null;

  // Anisotropy
  anisotropy: number;                   // Default: 0.0
  anisotropyRotation: number;           // Radians. Default: 0.0
  anisotropyMap: Texture | null;

  // Specular
  specularIntensity: number;            // Default: 1.0
  specularIntensityMap: Texture | null;
  specularColor: Color;                 // Default: white
  specularColorMap: Texture | null;

  // Other
  reflectivity: number;                 // Default: 0.5
  dispersion: number;                   // Default: 0.0 (chromatic dispersion)
});
```

---

## MeshToonMaterial

Cel-shading with discrete light steps.

```typescript
new MeshToonMaterial({
  color: Color | number;            // Default: 0xffffff
  gradientMap: Texture | null;      // Light step gradient (3x1 or 5x1 texture)
  map: Texture | null;
  normalMap: Texture | null;
  bumpMap: Texture | null;
  displacementMap: Texture | null;
  emissive: Color | number;         // Default: 0x000000
  alphaMap: Texture | null;
  wireframe: boolean;               // Default: false
});
```

**Rule**: ALWAYS set `gradientMap.minFilter = NearestFilter` and `gradientMap.magFilter = NearestFilter`.

---

## MeshMatcapMaterial

Uses a matcap texture for shading. No lights needed.

```typescript
new MeshMatcapMaterial({
  color: Color | number;            // Default: 0xffffff
  matcap: Texture | null;           // Matcap texture (spherical env baked to 2D)
  map: Texture | null;
  bumpMap: Texture | null;
  normalMap: Texture | null;
  displacementMap: Texture | null;
  alphaMap: Texture | null;
  flatShading: boolean;             // Default: false
  fog: boolean;                     // Default: true
});
```

---

## Utility Materials

### MeshNormalMaterial
Maps surface normals to RGB. Useful for debugging geometry normals.

### MeshDepthMaterial
Renders depth from camera. Used internally for shadow maps.

### MeshDistanceMaterial
Renders distance from a point light. Used internally for point light shadows.

### ShadowMaterial
Receives shadows on a transparent surface.

```typescript
new ShadowMaterial({
  color: Color | number;            // Shadow color. Default: 0x000000
  transparent: boolean;             // MUST be true
  opacity: number;                  // Shadow darkness. Default: 1.0
});
```

---

## Line and Point Materials

### LineBasicMaterial

```typescript
new LineBasicMaterial({
  color: Color | number;            // Default: 0xffffff
  linewidth: number;                // Default: 1 (>1 NOT supported on most platforms)
});
```

### LineDashedMaterial

```typescript
new LineDashedMaterial({
  color: Color | number;            // Default: 0xffffff
  dashSize: number;                 // Default: 3
  gapSize: number;                  // Default: 1
  scale: number;                    // Default: 1
});
// ALWAYS call line.computeLineDistances() after creating the Line object
```

### PointsMaterial

```typescript
new PointsMaterial({
  color: Color | number;            // Default: 0xffffff
  size: number;                     // Default: 1
  sizeAttenuation: boolean;         // Default: true (shrinks with distance)
  map: Texture | null;              // Sprite texture
  alphaMap: Texture | null;
  alphaTest: number;                // Default: 0
});
```

### SpriteMaterial

```typescript
new SpriteMaterial({
  color: Color | number;            // Default: 0xffffff
  map: Texture | null;
  alphaMap: Texture | null;
  rotation: number;                 // Radians. Default: 0
  sizeAttenuation: boolean;         // Default: true
});
```

---

## Texture Class

```typescript
class Texture {
  // Properties
  image: HTMLImageElement | HTMLCanvasElement | HTMLVideoElement;
  mapping: number;              // UVMapping (default)
  channel: number;              // UV channel (0 = uv, 1 = uv2)
  wrapS: number;                // ClampToEdgeWrapping (default)
  wrapT: number;                // ClampToEdgeWrapping (default)
  magFilter: number;            // LinearFilter (default)
  minFilter: number;            // LinearMipmapLinearFilter (default)
  anisotropy: number;           // 1 (default), max = renderer.capabilities.getMaxAnisotropy()
  format: number;               // RGBAFormat (default)
  type: number;                 // UnsignedByteType (default)
  offset: Vector2;              // (0, 0)
  repeat: Vector2;              // (1, 1)
  rotation: number;             // 0
  center: Vector2;              // (0, 0)
  generateMipmaps: boolean;     // true
  premultiplyAlpha: boolean;    // false
  flipY: boolean;               // true
  colorSpace: string;           // NoColorSpace
  needsUpdate: boolean;         // false

  // Methods
  clone(): Texture;
  copy(source: Texture): Texture;
  dispose(): void;
  transformUv(uv: Vector2): Vector2;
  toJSON(meta?: Object): Object;
}
```
