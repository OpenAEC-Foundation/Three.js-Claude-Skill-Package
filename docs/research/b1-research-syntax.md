# B1 Research: Three.js Syntax — Geometries, Materials, Loaders, Controls

> Phase B1 raw masterplan research. Covers the four syntax-layer skill domains.
> Sources: Official Three.js documentation (threejs.org/docs), r160+.
> Date: 2025-03-20

---

## 1. Geometries

### 1.1 BufferGeometry — The Universal Base Class

Since r125, the legacy `Geometry` class has been removed entirely. **BufferGeometry** is the sole geometry base class. It stores vertex data in GPU-friendly typed arrays via `BufferAttribute` objects.

**Constructor:**
```javascript
new BufferGeometry()
```
Creates an empty geometry. All data MUST be added via `setAttribute()`.

**Core properties:**

| Property | Type | Description |
|----------|------|-------------|
| `attributes` | Object | Map of named `BufferAttribute` instances (`position`, `normal`, `uv`, `color`) |
| `index` | BufferAttribute \| null | Optional index buffer for indexed geometry |
| `morphAttributes` | Object | Map of morph target attribute arrays |
| `groups` | Array | `{ start, count, materialIndex }` entries for multi-material rendering |
| `drawRange` | Object | `{ start: 0, count: Infinity }` — limits which vertices are rendered |
| `boundingBox` | Box3 \| null | Lazy-computed axis-aligned bounding box |
| `boundingSphere` | Sphere \| null | Lazy-computed bounding sphere |
| `userData` | Object | Arbitrary user data |
| `uuid` | String | Auto-generated unique ID |
| `name` | String | Optional human-readable name |

**Key methods — Attribute Management:**

| Method | Signature | Returns | Purpose |
|--------|-----------|---------|---------|
| `setAttribute` | `(name: string, attribute: BufferAttribute)` | `this` | Add or replace a named attribute |
| `getAttribute` | `(name: string)` | `BufferAttribute \| undefined` | Retrieve attribute by name |
| `deleteAttribute` | `(name: string)` | `this` | Remove an attribute |
| `hasAttribute` | `(name: string)` | `boolean` | Check attribute existence |
| `setIndex` | `(index: BufferAttribute \| Array)` | `this` | Set index buffer for indexed geometry |
| `getIndex` | `()` | `BufferAttribute \| null` | Retrieve index buffer |

**Key methods — Transform:**

| Method | Signature | Purpose |
|--------|-----------|---------|
| `applyMatrix4` | `(matrix: Matrix4)` | Transform position, normal, tangent attributes |
| `applyQuaternion` | `(q: Quaternion)` | Rotate all geometry data |
| `rotateX/Y/Z` | `(radians: number)` | Convenience rotation around axis |
| `scale` | `(x, y, z)` | Scale geometry data |
| `translate` | `(x, y, z)` | Move geometry data |
| `center` | `()` | Center geometry at origin |
| `lookAt` | `(vector: Vector3)` | Orient geometry toward a point |

**Key methods — Compute:**

| Method | Purpose |
|--------|---------|
| `computeVertexNormals()` | Calculate smooth normals from face data (requires `position`) |
| `normalizeNormals()` | Re-normalize normal vectors (call after non-uniform scaling) |
| `computeTangents()` | Compute tangent/bitangent (requires `position`, `normal`, `uv`) |
| `computeBoundingBox()` | Populate `boundingBox` property |
| `computeBoundingSphere()` | Populate `boundingSphere` property |

**Key methods — Conversion & Lifecycle:**

| Method | Purpose |
|--------|---------|
| `clone()` | Deep copy |
| `copy(source)` | Copy from another BufferGeometry |
| `toNonIndexed()` | Expand indexed geometry to non-indexed |
| `setFromPoints(points: Vector3[])` | Quick creation from point array |
| `toJSON()` | Serialize to JSON |
| `dispose()` | Free GPU memory. Fires `'dispose'` event. |

**Groups (multi-material):**

```javascript
geometry.addGroup(0, 100, 0);   // vertices 0-99 use material[0]
geometry.addGroup(100, 50, 1);  // vertices 100-149 use material[1]
geometry.clearGroups();
geometry.setDrawRange(0, 100);  // render only first 100 vertices
```

### 1.2 BufferAttribute

Stores per-vertex data in typed arrays. Every geometry attribute is a BufferAttribute.

**Constructor:**
```javascript
new BufferAttribute(array: TypedArray, itemSize: number, normalized?: boolean)
```

- `array` — Float32Array, Uint16Array, etc.
- `itemSize` — components per vertex: 3 for positions/normals, 2 for UVs, 4 for colors with alpha
- `normalized` — if true, integer data maps to [-1, 1] or [0, 1] range

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `array` | TypedArray | Raw data |
| `count` | number | Read-only: `array.length / itemSize` |
| `itemSize` | number | Components per vertex |
| `needsUpdate` | boolean | Set `true` to upload changed data to GPU |
| `usage` | GLenum | `StaticDrawUsage` (default), `DynamicDrawUsage`, `StreamDrawUsage` |
| `name` | string | Optional name |
| `normalized` | boolean | Whether integer data is normalized |

**Accessor methods:**
- `getX(index)`, `getY(index)`, `getZ(index)`, `getW(index)` — read components
- `setX(index, x)`, `setY(index, y)`, `setZ(index, z)`, `setW(index, w)` — write single component
- `setXY(index, x, y)`, `setXYZ(index, x, y, z)`, `setXYZW(index, x, y, z, w)` — write multiple
- `getComponent(index, component)`, `setComponent(index, component, value)` — generic access
- `applyMatrix3(m)`, `applyMatrix4(m)`, `applyNormalMatrix(m)` — transform in-place
- `copyAt(index1, source, index2)` — copy single vertex from another attribute
- `setUsage(usage)` — set GPU usage hint

**Typed convenience subclasses:**
- `Float32BufferAttribute` — most common for positions, normals, UVs
- `Float16BufferAttribute` — compact storage
- `Int8BufferAttribute`, `Int16BufferAttribute`, `Int32BufferAttribute`
- `Uint8BufferAttribute`, `Uint8ClampedBufferAttribute`, `Uint16BufferAttribute`, `Uint32BufferAttribute`

**InterleavedBufferAttribute:** Stores multiple attributes interleaved in a single buffer for better cache performance. Use with `InterleavedBuffer`.

### 1.3 Indexed vs Non-Indexed Geometry

**Indexed geometry** reuses vertices via an index buffer. A quad using 4 unique vertices + 6 indices (two triangles) is more memory-efficient than 6 separate vertices.

```javascript
// Indexed
const positions = new Float32Array([0,0,0, 1,0,0, 1,1,0, 0,1,0]);
const indices = new Uint16Array([0,1,2, 2,3,0]);
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setIndex(new THREE.BufferAttribute(indices, 1));

// Non-indexed: every triangle has its own 3 vertices
const nonIndexed = geometry.toNonIndexed();
```

**When to use indexed:** ALWAYS for static meshes with shared vertices (most cases). Non-indexed is needed when per-face attributes differ (flat shading, UV seams).

### 1.4 Built-in Geometry Classes

All extend `BufferGeometry`. All constructors accept numeric parameters with sensible defaults.

| Class | Constructor Parameters | Typical Use |
|-------|----------------------|-------------|
| `BoxGeometry` | `(width=1, height=1, depth=1, wSeg=1, hSeg=1, dSeg=1)` | Cubes, boxes |
| `SphereGeometry` | `(radius=1, wSeg=32, hSeg=16, phiStart, phiLen, thetaStart, thetaLen)` | Spheres, domes |
| `PlaneGeometry` | `(width=1, height=1, wSeg=1, hSeg=1)` | Ground planes, walls |
| `CylinderGeometry` | `(radiusTop=1, radiusBot=1, height=1, radSeg=32, hSeg=1, openEnded, thetaStart, thetaLen)` | Columns, pipes |
| `ConeGeometry` | `(radius=1, height=1, radSeg=32, hSeg=1, openEnded, thetaStart, thetaLen)` | Cones, spikes |
| `TorusGeometry` | `(radius=1, tube=0.4, radSeg=12, tubSeg=48, arc=2pi)` | Rings, donuts |
| `TorusKnotGeometry` | `(radius=1, tube=0.4, tubSeg=64, radSeg=8, p=2, q=3)` | Decorative knots |
| `RingGeometry` | `(innerR=0.5, outerR=1, thetaSeg=32, phiSeg=1, thetaStart, thetaLen)` | Flat rings |
| `CircleGeometry` | `(radius=1, segments=32, thetaStart, thetaLen)` | Flat circles |
| `CapsuleGeometry` | `(radius=1, length=1, capSeg=4, radSeg=8)` | Pill shapes |
| `DodecahedronGeometry` | `(radius=1, detail=0)` | 12-sided polyhedron |
| `IcosahedronGeometry` | `(radius=1, detail=0)` | 20-sided polyhedron |
| `OctahedronGeometry` | `(radius=1, detail=0)` | 8-sided polyhedron |
| `TetrahedronGeometry` | `(radius=1, detail=0)` | 4-sided polyhedron |
| `PolyhedronGeometry` | `(vertices, indices, radius=1, detail=0)` | Custom polyhedra |
| `LatheGeometry` | `(points, segments=12, phiStart=0, phiLen=2pi)` | Revolved profiles |
| `ExtrudeGeometry` | `(shapes, options)` | 2D shapes extruded to 3D |
| `ShapeGeometry` | `(shapes, curveSegments=12)` | Flat 2D shapes |
| `TubeGeometry` | `(curve, tubSeg=64, radius=1, radSeg=8, closed=false)` | Tubes along paths |
| `EdgesGeometry` | `(geometry, thresholdAngle=1)` | Edge outlines |
| `WireframeGeometry` | `(geometry)` | Wireframe lines |

**ExtrudeGeometry options object:**
```javascript
{
  depth: 1,              // extrusion depth
  bevelEnabled: true,
  bevelThickness: 0.2,
  bevelSize: 0.1,
  bevelSegments: 3,
  steps: 1,
  curveSegments: 12
}
```

### 1.5 InstancedMesh and InstancedBufferGeometry

For rendering many copies of the same geometry with different transforms/colors, use `InstancedMesh`. This is a single draw call for potentially thousands of objects.

**Constructor:**
```javascript
new InstancedMesh(geometry: BufferGeometry, material: Material, count: number)
```

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `count` | number | Number of instances |
| `instanceMatrix` | InstancedBufferAttribute | 4x4 matrix per instance (auto-created) |
| `instanceColor` | InstancedBufferAttribute \| null | Optional per-instance color |
| `morphTexture` | DataTexture \| null | Optional per-instance morph targets |

**Methods:**

| Method | Signature | Purpose |
|--------|-----------|---------|
| `setMatrixAt` | `(index, matrix: Matrix4)` | Set transform for instance |
| `getMatrixAt` | `(index, matrix: Matrix4)` | Get transform for instance |
| `setColorAt` | `(index, color: Color)` | Set color for instance |
| `getColorAt` | `(index, color: Color)` | Get color for instance |
| `setMorphAt` | `(index, mesh: Mesh)` | Set morph weights for instance |
| `getMorphAt` | `(index, mesh: Mesh)` | Get morph weights for instance |
| `computeBoundingBox` | `()` | Compute bounding box for all instances |
| `computeBoundingSphere` | `()` | Compute bounding sphere for all instances |
| `dispose` | `()` | Free GPU resources |

**Critical pattern:**
```javascript
const mesh = new THREE.InstancedMesh(geometry, material, 1000);
const dummy = new THREE.Object3D();

for (let i = 0; i < 1000; i++) {
  dummy.position.set(Math.random() * 100, 0, Math.random() * 100);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}
mesh.instanceMatrix.needsUpdate = true; // ALWAYS set after modifying
scene.add(mesh);
```

**InstancedBufferGeometry** extends BufferGeometry with `instanceCount` property for lower-level instanced rendering control.

**InstancedBufferAttribute** stores per-instance data with a configurable `meshPerAttribute` stride.

### 1.6 Geometry Pitfalls and Anti-Patterns

1. **Forgetting `dispose()`** — Geometry buffers are uploaded to GPU. Without dispose, memory leaks accumulate. ALWAYS call `geometry.dispose()` when removing meshes.
2. **Forgetting `needsUpdate`** — After modifying BufferAttribute data, you MUST set `attribute.needsUpdate = true`. The GPU buffer will not re-upload otherwise.
3. **Using `toNonIndexed()` unnecessarily** — Indexed geometry is more memory-efficient. Only convert when per-face attributes are required.
4. **Not calling `computeVertexNormals()`** — Custom geometry will render flat/black without normals. ALWAYS compute after setting positions.
5. **Creating geometry in the render loop** — Geometry allocation is expensive. ALWAYS create outside the loop; reuse or modify attributes instead.
6. **Wrong `itemSize`** — Position attributes MUST use itemSize 3, UV attributes MUST use itemSize 2. Mismatches cause silent rendering failures.
7. **InstancedMesh without `needsUpdate`** — After `setMatrixAt()`, you MUST set `instanceMatrix.needsUpdate = true`.

---

## 2. Materials

### 2.1 Material Hierarchy

The material system follows a class hierarchy:

```
Material (base)
├── MeshBasicMaterial      — unlit, flat color/texture
├── MeshLambertMaterial    — Lambertian (diffuse only, cheap)
├── MeshPhongMaterial      — Blinn-Phong (specular highlights, medium cost)
├── MeshStandardMaterial   — PBR metallic-roughness (recommended default)
│   └── MeshPhysicalMaterial — Extended PBR (clearcoat, transmission, sheen, iridescence)
├── MeshToonMaterial       — Cel/toon shading
├── MeshNormalMaterial     — Debug normals visualization
├── MeshDepthMaterial      — Depth-based rendering
├── MeshDistanceMaterial   — Distance-based (for point light shadows)
├── MeshMatcapMaterial     — Matcap-based shading (no lights needed)
├── ShaderMaterial         — Custom GLSL shaders with Three.js uniforms
├── RawShaderMaterial      — Custom GLSL without built-in uniforms
├── PointsMaterial         — Point cloud rendering
├── LineBasicMaterial      — Simple lines
├── LineDashedMaterial     — Dashed lines
├── SpriteMaterial         — Billboard sprites
└── ShadowMaterial        — Receives shadows only (transparent otherwise)
```

### 2.2 Material Base Class Properties

All materials inherit these from `Material`:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `side` | Constant | `FrontSide` | `FrontSide`, `BackSide`, or `DoubleSide` |
| `transparent` | boolean | `false` | Enable alpha blending |
| `opacity` | number | `1.0` | Overall opacity (requires `transparent: true` for < 1) |
| `depthTest` | boolean | `true` | Whether to test against depth buffer |
| `depthWrite` | boolean | `true` | Whether to write to depth buffer |
| `blending` | Constant | `NormalBlending` | Blending mode |
| `visible` | boolean | `true` | Whether to render |
| `needsUpdate` | boolean | `false` | Set `true` to recompile shader program |
| `fog` | boolean | `true` | Whether affected by scene fog |
| `wireframe` | boolean | `false` | Render as wireframe |
| `flatShading` | boolean | `false` | Use flat normals |
| `alphaTest` | number | `0` | Fragments with alpha below this are discarded |
| `alphaToCoverage` | boolean | `false` | Enable alpha-to-coverage (MSAA) |
| `clipIntersection` | boolean | `false` | Intersection mode for clipping planes |
| `clippingPlanes` | Plane[] | `null` | User-defined clipping planes |
| `colorWrite` | boolean | `true` | Whether to write color |
| `stencilWrite` | boolean | `false` | Whether to write stencil |
| `polygonOffset` | boolean | `false` | Enable polygon offset (z-fighting fix) |
| `polygonOffsetFactor` | number | `0` | Polygon offset factor |
| `polygonOffsetUnits` | number | `0` | Polygon offset units |

**Base class methods:**
- `clone()` — deep copy
- `copy(source)` — copy from another material
- `dispose()` — free shader program and GPU resources. Fires `'dispose'` event.
- `onBeforeCompile(shader, renderer)` — hook to modify shader before compilation
- `toJSON(meta)` — serialize to JSON

### 2.3 MeshStandardMaterial — PBR Metallic-Roughness

The recommended default material for physically-based rendering. Uses the metallic-roughness workflow, the industry standard for real-time PBR.

**Constructor:**
```javascript
new MeshStandardMaterial(parameters?: object)
```

**PBR core properties:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `color` | Color | `0xffffff` | Base albedo color |
| `map` | Texture | `null` | Albedo/color texture |
| `roughness` | number | `1.0` | 0 = mirror, 1 = fully matte |
| `roughnessMap` | Texture | `null` | Per-pixel roughness |
| `metalness` | number | `0.0` | 0 = dielectric, 1 = metal |
| `metalnessMap` | Texture | `null` | Per-pixel metalness |
| `normalMap` | Texture | `null` | Tangent-space normal map |
| `normalScale` | Vector2 | `(1, 1)` | Normal map intensity |
| `aoMap` | Texture | `null` | Ambient occlusion (requires second UV set) |
| `aoMapIntensity` | number | `1.0` | AO strength |
| `emissive` | Color | `0x000000` | Self-emission color |
| `emissiveMap` | Texture | `null` | Emission texture |
| `emissiveIntensity` | number | `1.0` | Emission strength |
| `displacementMap` | Texture | `null` | Vertex displacement map |
| `displacementScale` | number | `1.0` | Displacement height |
| `displacementBias` | number | `0.0` | Displacement offset |
| `envMap` | Texture | `null` | Environment/reflection map |
| `envMapIntensity` | number | `1.0` | Reflection strength |
| `alphaMap` | Texture | `null` | Alpha transparency map |

**PBR workflow explanation:**

The metallic-roughness model separates surfaces into two categories:
- **Metals** (metalness = 1.0): colored specular reflections, no diffuse term. Base `color` tints the specular.
- **Dielectrics** (metalness = 0.0): white/gray specular reflections, full diffuse color. Base `color` is the diffuse albedo.
- **Roughness** controls microsurface scattering: 0.0 = perfect mirror, 1.0 = fully diffuse/matte.

Environment maps are critical for PBR realism — they provide the image-based lighting that makes metallic surfaces look convincing.

### 2.4 MeshPhysicalMaterial — Extended PBR

Extends `MeshStandardMaterial` with advanced optical properties. More expensive to render.

**Clearcoat** — secondary clear layer on top (car paint, lacquered wood):

| Property | Type | Default |
|----------|------|---------|
| `clearcoat` | number | `0` |
| `clearcoatRoughness` | number | `0` |
| `clearcoatMap` | Texture | `null` |
| `clearcoatRoughnessMap` | Texture | `null` |
| `clearcoatNormalMap` | Texture | `null` |
| `clearcoatNormalScale` | Vector2 | `(1, 1)` |

**Transmission** — light passing through (glass, water, gems):

| Property | Type | Default |
|----------|------|---------|
| `transmission` | number | `0` |
| `transmissionMap` | Texture | `null` |
| `ior` | number | `1.5` |
| `thickness` | number | `0` |
| `thicknessMap` | Texture | `null` |
| `attenuationColor` | Color | `white` |
| `attenuationDistance` | number | `Infinity` |

**Sheen** — fabric-like soft reflection at grazing angles:

| Property | Type | Default |
|----------|------|---------|
| `sheen` | number | `0` |
| `sheenRoughness` | number | `1.0` |
| `sheenColor` | Color | `0x000000` |
| `sheenColorMap` | Texture | `null` |
| `sheenRoughnessMap` | Texture | `null` |

**Iridescence** — thin-film interference (soap bubbles, oil slicks):

| Property | Type | Default |
|----------|------|---------|
| `iridescence` | number | `0` |
| `iridescenceIOR` | number | `1.3` |
| `iridescenceMap` | Texture | `null` |
| `iridescenceThicknessRange` | Array | `[100, 400]` (nm) |
| `iridescenceThicknessMap` | Texture | `null` |

**Specular control:**

| Property | Type | Default |
|----------|------|---------|
| `specularIntensity` | number | `1.0` |
| `specularIntensityMap` | Texture | `null` |
| `specularColor` | Color | `white` |
| `specularColorMap` | Texture | `null` |

**Anisotropy** — directional reflections (brushed metal, hair):

| Property | Type | Default |
|----------|------|---------|
| `anisotropy` | number | `0` |
| `anisotropyRotation` | number | `0` |
| `anisotropyMap` | Texture | `null` |

### 2.5 ShaderMaterial and RawShaderMaterial

For fully custom rendering, ShaderMaterial lets you write GLSL vertex and fragment shaders.

**Constructor:**
```javascript
new ShaderMaterial({
  uniforms: { ... },
  vertexShader: '...',
  fragmentShader: '...',
  // optional
  defines: {},
  extensions: {},
  fog: false,
  lights: false,
  clipping: false,
  wireframe: false,
  flatShading: false,
  glslVersion: THREE.GLSL1,
  defaultAttributeValues: {}
})
```

**Uniforms — the data bridge between JS and GLSL:**

```javascript
uniforms: {
  uTime:    { value: 0.0 },                          // float
  uColor:   { value: new THREE.Color(0xff0000) },     // vec3
  uTexture: { value: textureLoader.load('tex.png') }, // sampler2D
  uMatrix:  { value: new THREE.Matrix4() },            // mat4
  uVector:  { value: new THREE.Vector3(1, 0, 0) },    // vec3
  uResolution: { value: new THREE.Vector2(800, 600) } // vec2
}
```

Update in render loop: `material.uniforms.uTime.value += delta;`

**Built-in uniforms (ShaderMaterial only, NOT RawShaderMaterial):**
- `modelMatrix`, `modelViewMatrix`, `projectionMatrix`, `viewMatrix`
- `normalMatrix` — inverse-transpose of modelViewMatrix
- `cameraPosition` — world-space camera position

**Built-in attributes (ShaderMaterial only):**
- `position` (vec3), `normal` (vec3), `uv` (vec2), `color` (vec3)

**RawShaderMaterial** differences:
- Does NOT inject built-in uniforms, attributes, or precision declarations
- You MUST declare everything manually in GLSL
- Requires explicit `precision highp float;` statement
- More verbose but provides total control

**Properties:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `defines` | Object | `{}` | `#define` directives injected into GLSL |
| `fog` | boolean | `false` | Include fog uniforms |
| `lights` | boolean | `false` | Include lighting uniforms |
| `clipping` | boolean | `false` | Include clipping plane uniforms |
| `extensions` | Object | `{}` | e.g. `{ derivatives: true }` for `OES_standard_derivatives` |
| `glslVersion` | Constant | `GLSL1` | `THREE.GLSL1` or `THREE.GLSL3` |

### 2.6 Texture System

Textures are loaded via `TextureLoader` and assigned to material map properties.

**Common texture maps:**

| Map Property | Purpose | Notes |
|-------------|---------|-------|
| `map` | Base color/albedo | sRGB color space |
| `normalMap` | Surface detail without geometry | Tangent-space, linear |
| `roughnessMap` | Per-pixel roughness | Green channel typically |
| `metalnessMap` | Per-pixel metalness | Blue channel typically |
| `aoMap` | Ambient occlusion | Requires `geometry.attributes.uv2` |
| `emissiveMap` | Self-illumination areas | sRGB color space |
| `displacementMap` | Vertex displacement | Linear, requires sufficient geometry segments |
| `alphaMap` | Transparency | Requires `transparent: true` |
| `envMap` | Environment reflections | CubeTexture or equirectangular |
| `lightMap` | Baked lighting | Requires `geometry.attributes.uv2` |
| `bumpMap` | Height-based normal perturbation | Simpler than normalMap |

### 2.7 Material Pitfalls and Anti-Patterns

1. **Forgetting `material.dispose()`** — shader programs and textures leak GPU memory. ALWAYS dispose materials when removing objects.
2. **Setting `needsUpdate = true` every frame** — This recompiles the shader program. NEVER set it in the render loop. Only set when changing structural properties (adding/removing maps, changing `defines`).
3. **Transparent objects without `transparent: true`** — Opacity/alphaMap will not work without this flag.
4. **Wrong color space** — Color/emissive textures MUST be in sRGB. Normal/roughness/metalness maps MUST be in linear. Three.js r152+ uses `texture.colorSpace = THREE.SRGBColorSpace` for color textures.
5. **AO map without UV2** — `aoMap` and `lightMap` require a second UV channel: `geometry.setAttribute('uv2', geometry.attributes.uv)`.
6. **Using MeshPhysicalMaterial everywhere** — It is significantly more expensive than MeshStandardMaterial. ONLY use when you need clearcoat/transmission/sheen/iridescence.
7. **Not setting `side: THREE.DoubleSide`** — Back faces are culled by default. For thin objects (planes, leaves), you MUST set `DoubleSide`.

---

## 3. Loaders

### 3.1 Loading Architecture

Three.js uses a `Loader` base class with a shared `LoadingManager` infrastructure.

**Loader base class** provides:
- `load(url, onLoad, onProgress, onError)` — callback-based loading
- `loadAsync(url, onProgress)` — Promise-based loading
- `setPath(path)` — base URL prefix
- `setResourcePath(path)` — resource resolution path
- `setCrossOrigin(value)` — CORS setting
- `setWithCredentials(value)` — credentials for cross-origin
- `setRequestHeader(header)` — custom HTTP headers

**LoadingManager** coordinates multiple loaders:

```javascript
const manager = new THREE.LoadingManager(
  () => console.log('All loaded'),          // onLoad
  (url, loaded, total) => console.log(`${loaded}/${total}`), // onProgress
  (url) => console.error(`Failed: ${url}`)  // onError
);
```

**Manager methods:**
- `onStart(url, itemsLoaded, itemsTotal)` — loading begins
- `onLoad()` — all items finished
- `onProgress(url, itemsLoaded, itemsTotal)` — per-item progress
- `onError(url)` — item failed
- `itemStart(url)`, `itemEnd(url)`, `itemError(url)` — manual tracking
- `resolveURL(url)` — resolve URL through modifiers
- `setURLModifier(callback)` — custom URL rewriting (useful for blob URLs, service workers)

### 3.2 Built-in Loaders

| Loader | Import | Purpose |
|--------|--------|---------|
| `TextureLoader` | `three` | Load 2D textures (PNG, JPG, WebP) |
| `CubeTextureLoader` | `three` | Load 6-face cubemap textures |
| `FileLoader` | `three` | Generic binary/text file loading |
| `ImageLoader` | `three` | Load images as HTMLImageElement |
| `ImageBitmapLoader` | `three` | Load images as ImageBitmap (offscreen) |
| `ObjectLoader` | `three` | Load Three.js JSON format |
| `AudioLoader` | `three` | Load audio files for positional audio |
| `AnimationLoader` | `three` | Load animation clips |
| `BufferGeometryLoader` | `three` | Load serialized geometry |
| `MaterialLoader` | `three` | Load serialized materials |
| `DataTextureLoader` | `three` | Load raw data textures |
| `CompressedTextureLoader` | `three` | Load compressed texture formats |

**TextureLoader usage:**

```javascript
import * as THREE from 'three';

const loader = new THREE.TextureLoader();

// Callback style
loader.load('texture.png', (texture) => {
  texture.colorSpace = THREE.SRGBColorSpace; // for color textures
  material.map = texture;
  material.needsUpdate = true;
});

// Async style
const texture = await loader.loadAsync('texture.png');
```

### 3.3 Addon Loaders (three/addons)

These require separate imports from the examples/addons directory.

| Loader | Import Path | Purpose |
|--------|-------------|---------|
| `GLTFLoader` | `three/addons/loaders/GLTFLoader.js` | glTF 2.0 models (.gltf, .glb) |
| `DRACOLoader` | `three/addons/loaders/DRACOLoader.js` | Draco-compressed geometry |
| `KTX2Loader` | `three/addons/loaders/KTX2Loader.js` | KTX2 compressed textures |
| `FBXLoader` | `three/addons/loaders/FBXLoader.js` | Autodesk FBX format |
| `OBJLoader` | `three/addons/loaders/OBJLoader.js` | Wavefront OBJ format |
| `MTLLoader` | `three/addons/loaders/MTLLoader.js` | OBJ material companion |
| `PLYLoader` | `three/addons/loaders/PLYLoader.js` | Point cloud / mesh PLY |
| `STLLoader` | `three/addons/loaders/STLLoader.js` | Stereolithography format |
| `SVGLoader` | `three/addons/loaders/SVGLoader.js` | SVG vector paths |
| `EXRLoader` | `three/addons/loaders/EXRLoader.js` | HDR environment maps |
| `RGBELoader` | `three/addons/loaders/RGBELoader.js` | HDRI .hdr files |
| `IFCLoader` | `three/addons/loaders/IFCLoader.js` | BIM/IFC format |
| `3DMLoader` | `three/addons/loaders/3DMLoader.js` | Rhino 3DM format |
| `LDrawLoader` | `three/addons/loaders/LDrawLoader.js` | LEGO LDraw format |
| `MMDLoader` | `three/addons/loaders/MMDLoader.js` | MikuMikuDance format |
| `PCDLoader` | `three/addons/loaders/PCDLoader.js` | Point cloud PCD |
| `TIFFLoader` | `three/addons/loaders/TIFFLoader.js` | TIFF images |
| `VOXLoader` | `three/addons/loaders/VOXLoader.js` | MagicaVoxel format |

### 3.4 GLTFLoader — The Preferred 3D Format

glTF (GL Transmission Format) is the recommended format for Three.js. It supports meshes, materials (PBR), animations, cameras, lights, and scene hierarchy in a single file.

**Import:**
```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
```

**Constructor:** `new GLTFLoader(manager?: LoadingManager)`

**Methods:**

| Method | Signature | Purpose |
|--------|-----------|---------|
| `load` | `(url, onLoad, onProgress, onError)` | Load with callbacks |
| `loadAsync` | `(url, onProgress)` | Load with Promise |
| `parse` | `(data, path, onLoad, onError)` | Parse ArrayBuffer/JSON directly |
| `setDRACOLoader` | `(dracoLoader: DRACOLoader)` | Enable Draco decompression |
| `setKTX2Loader` | `(ktx2Loader: KTX2Loader)` | Enable KTX2 texture decompression |
| `setMeshoptDecoder` | `(decoder)` | Enable meshopt decompression |
| `register` | `(plugin)` | Register a glTF extension plugin |
| `unregister` | `(plugin)` | Unregister a plugin |

**GLTF result object structure:**

```javascript
{
  scene: THREE.Group,           // root scene node
  scenes: THREE.Group[],        // all scenes in the file
  cameras: THREE.Camera[],      // embedded cameras
  animations: THREE.AnimationClip[], // animation data
  asset: {                      // metadata
    generator: string,
    version: string
  },
  parser: GLTFParser,           // internal parser (advanced)
  userData: {}                  // custom extensions
}
```

**Draco compression setup (reduces file size by 80-90%):**

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
// Or local: dracoLoader.setDecoderPath('/draco/');

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

const gltf = await gltfLoader.loadAsync('model.glb');
scene.add(gltf.scene);
```

**KTX2 texture compression setup:**

```javascript
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('/basis/');
ktx2Loader.detectSupport(renderer);

gltfLoader.setKTX2Loader(ktx2Loader);
```

**Meshopt decoder setup:**

```javascript
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

await MeshoptDecoder.ready;
gltfLoader.setMeshoptDecoder(MeshoptDecoder);
```

### 3.5 Loader Pitfalls and Anti-Patterns

1. **Loading in the render loop** — NEVER call `loader.load()` inside `requestAnimationFrame`. Load once, cache the result.
2. **Not disposing loaded models** — GLTF models contain meshes, materials, and textures. You MUST traverse and dispose all: `gltf.scene.traverse(child => { if (child.isMesh) { child.geometry.dispose(); child.material.dispose(); } })`.
3. **Missing CORS headers** — Cross-origin textures/models fail silently. ALWAYS serve assets from the same origin or configure CORS.
4. **Not setting DRACOLoader decoder path** — Draco decompression requires WASM decoders. The path MUST point to a directory containing `draco_decoder.wasm`.
5. **Forgetting `loadAsync` error handling** — ALWAYS wrap in try/catch. Failed loads without error handlers crash silently.
6. **Loading .gltf without .bin companion** — Separated GLTF files need both `.gltf` and `.bin` files. Use `.glb` (binary GLTF) for single-file distribution.
7. **Not calling `dracoLoader.dispose()`** — After loading all models, dispose the DRACOLoader to free the WASM decoder.

---

## 4. Controls

### 4.1 Controls Overview

Controls are addon classes (not core Three.js) that handle camera or object manipulation via mouse/touch/keyboard input. All are in `three/addons/controls/`.

| Control | Import Path | Purpose |
|---------|-------------|---------|
| `OrbitControls` | `three/addons/controls/OrbitControls.js` | Orbit around target point |
| `MapControls` | `three/addons/controls/MapControls.js` | Top-down map navigation |
| `TrackballControls` | `three/addons/controls/TrackballControls.js` | Free trackball rotation |
| `ArcballControls` | `three/addons/controls/ArcballControls.js` | Virtual arcball rotation |
| `FlyControls` | `three/addons/controls/FlyControls.js` | 6-DOF flight |
| `FirstPersonControls` | `three/addons/controls/FirstPersonControls.js` | FPS-style camera |
| `PointerLockControls` | `three/addons/controls/PointerLockControls.js` | Mouse-locked FPS |
| `DragControls` | `three/addons/controls/DragControls.js` | Click-drag objects |
| `TransformControls` | `three/addons/controls/TransformControls.js` | Translate/rotate/scale gizmo |

### 4.2 OrbitControls — The Default Choice

The most commonly used controls. Orbits the camera around a target point with mouse/touch input.

**Constructor:**
```javascript
new OrbitControls(camera: Camera, domElement: HTMLElement)
```

**Properties — Rotation:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `autoRotate` | boolean | `false` | Auto-rotate around target |
| `autoRotateSpeed` | number | `2.0` | Auto-rotation speed |
| `enableDamping` | boolean | `false` | Smooth inertia effect |
| `dampingFactor` | number | `0.05` | Damping strength (0-1) |
| `enableRotate` | boolean | `true` | Allow rotation |
| `rotateSpeed` | number | `1.0` | Rotation speed |

**Properties — Zoom:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enableZoom` | boolean | `true` | Allow zoom |
| `zoomSpeed` | number | `1.0` | Zoom speed |
| `minDistance` | number | `0` | Min zoom distance (PerspectiveCamera) |
| `maxDistance` | number | `Infinity` | Max zoom distance (PerspectiveCamera) |
| `minZoom` | number | `0` | Min zoom (OrthographicCamera) |
| `maxZoom` | number | `Infinity` | Max zoom (OrthographicCamera) |
| `zoomToCursor` | boolean | `false` | Zoom toward cursor position |

**Properties — Pan:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enablePan` | boolean | `true` | Allow panning |
| `panSpeed` | number | `1.0` | Pan speed |
| `keyPanSpeed` | number | `7.0` | Keyboard pan speed |
| `screenSpacePanning` | boolean | `true` | Pan in screen space vs world XZ plane |

**Properties — Constraints:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `minPolarAngle` | number | `0` | Min vertical angle (0 = top) |
| `maxPolarAngle` | number | `Math.PI` | Max vertical angle (PI = bottom) |
| `minAzimuthAngle` | number | `-Infinity` | Min horizontal angle |
| `maxAzimuthAngle` | number | `Infinity` | Max horizontal angle |

**Properties — Input:**

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enabled` | boolean | `true` | Master enable/disable |
| `target` | Vector3 | `(0,0,0)` | Orbit center point |
| `mouseButtons` | Object | `{LEFT: ROTATE, MIDDLE: DOLLY, RIGHT: PAN}` | Mouse button mapping |
| `touches` | Object | `{ONE: ROTATE, TWO: DOLLY_PAN}` | Touch gesture mapping |

**Methods:**

| Method | Purpose |
|--------|---------|
| `update()` | MUST call in animation loop (especially with damping/autoRotate) |
| `dispose()` | Remove all event listeners. ALWAYS call when removing controls. |
| `saveState()` | Save current camera position/target |
| `reset()` | Restore saved state |
| `getDistance()` | Current distance from target |
| `getAzimuthalAngle()` | Current horizontal angle |
| `getPolarAngle()` | Current vertical angle |
| `listenToKeyEvents(domElement)` | Enable keyboard navigation |
| `stopListenToKeyEvents()` | Disable keyboard navigation |

**Events:**

| Event | When |
|-------|------|
| `change` | Camera position/target changed |
| `start` | User begins interaction |
| `end` | User ends interaction |

### 4.3 MapControls

Extends OrbitControls with different default input mapping optimized for top-down/map-style navigation:
- **Left mouse button**: Pan (instead of rotate)
- **Right mouse button**: Rotate (instead of pan)
- **screenSpacePanning**: `false` by default (pans on world XZ plane)

Same API as OrbitControls otherwise. Use for architectural plans, GIS, strategy game cameras.

```javascript
import { MapControls } from 'three/addons/controls/MapControls.js';
const controls = new MapControls(camera, renderer.domElement);
```

### 4.4 Other Control Types

**TrackballControls** — Free rotation without polar angle clamping. Camera can rotate past the poles (unlike OrbitControls which clamps at top/bottom). Good for model inspection.

**ArcballControls** — Virtual arcball rotation with support for multiple simultaneous gestures. More intuitive for precise 3D rotation.

**FlyControls** — 6-DOF camera flight. WASD for movement, mouse for look direction. Good for architectural walkthroughs.
- `movementSpeed` — flight speed
- `rollSpeed` — roll speed
- `dragToLook` — require mouse button to look

**FirstPersonControls** — Similar to FlyControls but with automatic mouse-look (no click required). Simpler FPS-style camera.
- `movementSpeed`, `lookSpeed`
- `lookVertical` — allow vertical look
- `activeLook` — continuous mouse look

**PointerLockControls** — Uses Pointer Lock API for true FPS camera. Mouse is captured; movement is delta-based. Best for games.
- `lock()` — request pointer lock
- `unlock()` — release pointer lock
- `isLocked` — current lock state
- Dispatches `lock`, `unlock` events

**DragControls** — Click and drag objects in the scene. Does not control the camera; controls object positions.
- Constructor: `new DragControls(objects, camera, domElement)`
- Events: `dragstart`, `drag`, `dragend`, `hoveron`, `hoveroff`
- `enabled` — enable/disable

**TransformControls** — 3D gizmo for translating, rotating, or scaling objects. Like Blender/Unity transform gizmos.
- `attach(object)` — attach gizmo to object
- `detach()` — remove gizmo
- `setMode('translate' | 'rotate' | 'scale')` — switch mode
- `setSpace('world' | 'local')` — coordinate space
- `setSize(number)` — gizmo visual size
- Events: `change`, `dragging-changed`, `objectChange`

### 4.5 Controls Lifecycle Pattern

```javascript
// Setup
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;
controls.dampingFactor = 0.05;

// Animation loop — MUST call update()
function animate() {
  requestAnimationFrame(animate);
  controls.update(); // required for damping and autoRotate
  renderer.render(scene, camera);
}

// Cleanup — MUST call dispose()
function cleanup() {
  controls.dispose();
}
```

### 4.6 Controls Pitfalls and Anti-Patterns

1. **Forgetting `controls.update()` in the animation loop** — Damping and autoRotate will not work without calling `update()` every frame.
2. **Forgetting `controls.dispose()`** — Event listeners leak. ALWAYS dispose when destroying the scene.
3. **Using OrbitControls with TransformControls simultaneously** — When dragging a TransformControls gizmo, OrbitControls receives the same mouse events. ALWAYS disable OrbitControls during TransformControls dragging:
   ```javascript
   transformControls.addEventListener('dragging-changed', (event) => {
     orbitControls.enabled = !event.value;
   });
   ```
4. **Not setting `controls.target`** — OrbitControls orbits around `(0,0,0)` by default. ALWAYS set the target to the point of interest:
   ```javascript
   controls.target.set(x, y, z);
   controls.update();
   ```
5. **Attaching controls to wrong DOM element** — ALWAYS pass `renderer.domElement`, not `document` or `document.body`. Using document causes event conflicts with UI elements.
6. **PointerLockControls without user gesture** — Browsers require a user interaction (click) before allowing pointer lock. ALWAYS trigger `controls.lock()` from a click handler.

---

## 5. Import Path Reference

### Core (from `three` package)
```javascript
import * as THREE from 'three';
import { BufferGeometry, BufferAttribute, Float32BufferAttribute } from 'three';
import { InstancedMesh, InstancedBufferGeometry, InstancedBufferAttribute } from 'three';
import { MeshStandardMaterial, MeshPhysicalMaterial, ShaderMaterial, RawShaderMaterial } from 'three';
import { TextureLoader, CubeTextureLoader, LoadingManager } from 'three';
import { BoxGeometry, SphereGeometry, PlaneGeometry, CylinderGeometry } from 'three';
```

### Addons (from `three/addons/` or `three/examples/jsm/`)
```javascript
// Loaders
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { FBXLoader } from 'three/addons/loaders/FBXLoader.js';
import { OBJLoader } from 'three/addons/loaders/OBJLoader.js';
import { STLLoader } from 'three/addons/loaders/STLLoader.js';
import { EXRLoader } from 'three/addons/loaders/EXRLoader.js';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';

// Controls
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { MapControls } from 'three/addons/controls/MapControls.js';
import { FlyControls } from 'three/addons/controls/FlyControls.js';
import { FirstPersonControls } from 'three/addons/controls/FirstPersonControls.js';
import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';
import { TransformControls } from 'three/addons/controls/TransformControls.js';
import { DragControls } from 'three/addons/controls/DragControls.js';
import { TrackballControls } from 'three/addons/controls/TrackballControls.js';
import { ArcballControls } from 'three/addons/controls/ArcballControls.js';
```

---

## 6. Skill Scope Mapping

Based on this research, the four syntax skills should cover:

### Skill: threejs-syntax-geometries
- BufferGeometry as the sole base class (legacy Geometry removed)
- BufferAttribute and typed subclasses
- All 20 built-in geometry classes with constructor parameters
- Indexed vs non-indexed geometry
- InstancedMesh and InstancedBufferGeometry for batch rendering
- Custom geometry creation from typed arrays
- Memory management: dispose() pattern
- Groups for multi-material rendering

### Skill: threejs-syntax-materials
- Material class hierarchy (15+ material types)
- PBR workflow: MeshStandardMaterial (metalness/roughness)
- MeshPhysicalMaterial advanced properties (clearcoat, transmission, sheen, iridescence, anisotropy)
- ShaderMaterial and RawShaderMaterial for custom GLSL
- Uniform system and built-in uniforms/attributes
- Texture map types and their purposes (12+ map types)
- Color space management (sRGB vs linear)
- Material disposal and needsUpdate semantics

### Skill: threejs-syntax-loaders
- Loader base class and LoadingManager architecture
- Built-in loaders (TextureLoader, CubeTextureLoader, etc.)
- GLTFLoader as the preferred 3D format loader
- DRACOLoader, KTX2Loader, MeshoptDecoder for compression
- GLTF result object structure (scene, animations, cameras)
- Addon loaders (FBX, OBJ, STL, PLY, EXR, RGBE, IFC, 3DM)
- Error handling and progress tracking
- Model disposal traversal pattern

### Skill: threejs-syntax-controls
- OrbitControls (full API: rotation, zoom, pan, constraints, damping)
- MapControls (differences from OrbitControls for top-down navigation)
- FlyControls / FirstPersonControls / PointerLockControls for FPS-style
- TransformControls for object manipulation gizmos
- DragControls for object dragging
- Controls lifecycle: constructor, update() in loop, dispose() on cleanup
- OrbitControls + TransformControls coordination pattern
