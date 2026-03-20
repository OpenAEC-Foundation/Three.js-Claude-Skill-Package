# Vooronderzoek: Three.js Geometry and Materials Deep Dive

> Comprehensive research document covering BufferGeometry, built-in geometries, InstancedMesh,
> the material system, textures, and custom shaders in Three.js r160+.
> Sources: Official Three.js documentation (threejs.org/docs), verified via WebFetch.
> Date: 2026-03-20

---

## 1. BufferGeometry Deep Dive

Since r125, the legacy `Geometry` class has been fully removed. `BufferGeometry` is the ONLY geometry base class in Three.js. It stores vertex data as GPU-friendly typed arrays via `BufferAttribute` objects.

### 1.1 Constructor

```javascript
new BufferGeometry()
```

Creates an empty geometry. You MUST add attributes via `setAttribute()` before the geometry can be rendered.

### 1.2 Properties — Complete Reference

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `id` | `Integer` | auto | Unique numeric identifier, auto-incremented |
| `uuid` | `String` | auto | UUID, auto-generated on creation |
| `name` | `String` | `""` | Optional human-readable name |
| `type` | `String` | `"BufferGeometry"` | Class type identifier |
| `attributes` | `Object` | `{}` | Hash map of named `BufferAttribute` instances |
| `index` | `BufferAttribute \| null` | `null` | Optional index buffer for indexed rendering |
| `morphAttributes` | `Object` | `{}` | Hash map of morph target attribute arrays |
| `morphTargetsRelative` | `Boolean` | `false` | If `true`, morph data is treated as relative offsets rather than absolute positions |
| `groups` | `Array` | `[]` | Array of `{ start, count, materialIndex }` for multi-material |
| `drawRange` | `Object` | `{ start: 0, count: Infinity }` | Controls which portion of the geometry is rendered |
| `boundingBox` | `Box3 \| null` | `null` | Cached bounding box — `null` until `computeBoundingBox()` is called |
| `boundingSphere` | `Sphere \| null` | `null` | Cached bounding sphere — `null` until `computeBoundingSphere()` is called |
| `userData` | `Object` | `{}` | Arbitrary user data storage |

### 1.3 Methods — Complete Reference

#### Attribute Management

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `setAttribute` | `(name: string, attribute: BufferAttribute): BufferGeometry` | `this` | Add or replace a named attribute |
| `getAttribute` | `(name: string): BufferAttribute \| undefined` | attribute | Retrieve attribute by name |
| `deleteAttribute` | `(name: string): BufferGeometry` | `this` | Remove a named attribute |
| `hasAttribute` | `(name: string): boolean` | boolean | Check if attribute exists |

#### Bounding Volume Computation

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `computeBoundingBox` | `(): void` | void | Compute and cache the axis-aligned bounding box |
| `computeBoundingSphere` | `(): void` | void | Compute and cache the bounding sphere |

**Anti-pattern**: NEVER assume `boundingBox` or `boundingSphere` are populated automatically. They are `null` until you explicitly call the compute method. The renderer calls `computeBoundingSphere()` for frustum culling, but if you need bounding data in your own code, ALWAYS call it yourself.

#### Normal and Tangent Computation

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `computeVertexNormals` | `(): void` | void | Compute smooth vertex normals from face topology |
| `computeTangents` | `(): void` | void | Compute tangent vectors (requires `normal` and `uv` attributes, and an index) |
| `normalizeNormals` | `(): void` | void | Normalize all normal vectors to unit length |

**Rule**: ALWAYS call `computeVertexNormals()` after building custom geometry if you need lighting. Without normals, `MeshStandardMaterial` and other lit materials will render black.

**Rule**: `computeTangents()` requires the geometry to have `position`, `normal`, `uv` attributes AND an index buffer. It will fail silently if any of these are missing.

#### Group Management (Multi-Material)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `addGroup` | `(start: number, count: number, materialIndex?: number): void` | void | Define a render group |
| `clearGroups` | `(): void` | void | Remove all groups |

Groups enable multi-material rendering. Each group maps a range of faces (by index) to a specific material in the material array passed to `Mesh`.

```javascript
// Multi-material example
geometry.addGroup(0, 6, 0);    // First 6 indices -> material[0]
geometry.addGroup(6, 6, 1);    // Next 6 indices -> material[1]
const mesh = new THREE.Mesh(geometry, [materialA, materialB]);
```

#### Draw Range

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `setDrawRange` | `(start: number, count: number): void` | void | Limit which vertices/indices are rendered |

Useful for progressive rendering or LOD systems. Only the specified range will be sent to the GPU for drawing.

#### Transform Operations (All return `this` for chaining)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `translate` | `(x: number, y: number, z: number): BufferGeometry` | `this` | Translate all vertex positions |
| `rotateX` | `(radians: number): BufferGeometry` | `this` | Rotate around X axis |
| `rotateY` | `(radians: number): BufferGeometry` | `this` | Rotate around Y axis |
| `rotateZ` | `(radians: number): BufferGeometry` | `this` | Rotate around Z axis |
| `scale` | `(x: number, y: number, z: number): BufferGeometry` | `this` | Scale vertex positions |
| `center` | `(): BufferGeometry` | `this` | Center geometry at the origin |
| `lookAt` | `(vector: Vector3): BufferGeometry` | `this` | Orient geometry to face a point |

**Important**: These methods modify the actual vertex data in-place. They are NOT equivalent to setting `mesh.position` or `mesh.rotation`. Use them for one-time geometry alignment, NEVER in an animation loop.

#### Conversion and Serialization

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `toNonIndexed` | `(): BufferGeometry` | new geometry | Create a non-indexed copy (duplicates shared vertices) |
| `clone` | `(): BufferGeometry` | new geometry | Deep copy the geometry |
| `copy` | `(source: BufferGeometry): BufferGeometry` | `this` | Copy attributes from another geometry |
| `toJSON` | `(): Object` | JSON | Serialize to JSON |
| `dispose` | `(): void` | void | Free GPU resources associated with this geometry |

### 1.4 BufferAttribute

`BufferAttribute` wraps a typed array and describes how the GPU should interpret it.

#### Constructor

```javascript
new BufferAttribute(array: TypedArray, itemSize: number, normalized?: boolean)
```

- `array` — A `Float32Array`, `Uint16Array`, etc.
- `itemSize` — Number of values per vertex (1 for scalar, 2 for UV, 3 for position/normal, 4 for color+alpha)
- `normalized` — If `true`, integer values are mapped to [0, 1] or [-1, 1] on the GPU

#### Key Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `array` | `TypedArray` | — | The underlying data |
| `itemSize` | `number` | — | Values per vertex |
| `count` | `number` | computed | `array.length / itemSize` |
| `normalized` | `boolean` | `false` | Normalize integer data |
| `usage` | `number` | `StaticDrawUsage` | GPU usage hint |
| `needsUpdate` | `boolean` | `false` | Set `true` to upload changes to GPU |
| `name` | `string` | `""` | Optional identifier |

#### Usage Hints

| Constant | Value | When to Use |
|----------|-------|-------------|
| `THREE.StaticDrawUsage` | 35044 | Data uploaded once, read many times (default, most common) |
| `THREE.DynamicDrawUsage` | 35048 | Data updated frequently (e.g., particle systems, animated vertices) |
| `THREE.StreamDrawUsage` | 35040 | Data updated every frame and read once |

**Rule**: ALWAYS set `usage` to `DynamicDrawUsage` before first render if you plan to update the attribute data every frame. Changing usage after initial upload has no effect in some drivers.

#### Accessor Methods

```javascript
getX(index: number): number
getY(index: number): number
getZ(index: number): number
getW(index: number): number
setX(index: number, x: number): this
setY(index: number, y: number): this
setZ(index: number, z: number): this
setW(index: number, w: number): this
setXY(index: number, x: number, y: number): this
setXYZ(index: number, x: number, y: number, z: number): this
setXYZW(index: number, x: number, y: number, z: number, w: number): this
```

**Rule**: After modifying values via set methods, ALWAYS set `attribute.needsUpdate = true` to trigger GPU upload.

#### Typed Array Convenience Classes

| Class | Underlying Type | Bytes per Element | Use Case |
|-------|----------------|-------------------|----------|
| `Float32BufferAttribute` | `Float32Array` | 4 | Positions, normals, UVs (most common) |
| `Float16BufferAttribute` | `Float16Array` | 2 | Memory-optimized attributes |
| `Int8BufferAttribute` | `Int8Array` | 1 | Signed byte data |
| `Int16BufferAttribute` | `Int16Array` | 2 | Signed short data |
| `Int32BufferAttribute` | `Int32Array` | 4 | Signed integer data |
| `Uint8BufferAttribute` | `Uint8Array` | 1 | Byte colors (with normalized=true) |
| `Uint8ClampedBufferAttribute` | `Uint8ClampedArray` | 1 | Clamped byte data |
| `Uint16BufferAttribute` | `Uint16Array` | 2 | Index buffers (<65536 vertices) |
| `Uint32BufferAttribute` | `Uint32Array` | 4 | Index buffers (>65536 vertices) |

### 1.5 InterleavedBuffer and InterleavedBufferAttribute

For optimal GPU cache performance, multiple attributes can be interleaved in a single buffer.

#### InterleavedBuffer Constructor

```javascript
new InterleavedBuffer(array: TypedArray, stride: number)
```

- `array` — Single typed array containing all interleaved data
- `stride` — Number of values between consecutive entries of the same attribute

#### InterleavedBufferAttribute Constructor

```javascript
new InterleavedBufferAttribute(interleavedBuffer: InterleavedBuffer, itemSize: number, offset: number, normalized?: boolean)
```

- `interleavedBuffer` — The parent `InterleavedBuffer`
- `itemSize` — Values per vertex for this attribute (3 for position, 2 for UV)
- `offset` — Starting position within each stride
- `normalized` — Whether to normalize integer values

#### Interleaved vs Separate Buffers

```javascript
// INTERLEAVED — Better GPU cache locality for large meshes
// Layout: [x,y,z, nx,ny,nz, u,v, x,y,z, nx,ny,nz, u,v, ...]
const stride = 8; // 3 + 3 + 2
const buffer = new THREE.InterleavedBuffer(new Float32Array(vertexCount * stride), stride);

geometry.setAttribute('position', new THREE.InterleavedBufferAttribute(buffer, 3, 0));
geometry.setAttribute('normal',   new THREE.InterleavedBufferAttribute(buffer, 3, 3));
geometry.setAttribute('uv',       new THREE.InterleavedBufferAttribute(buffer, 2, 6));

// SEPARATE — Better for partial updates
geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
geometry.setAttribute('normal',   new THREE.Float32BufferAttribute(normals, 3));
geometry.setAttribute('uv',       new THREE.Float32BufferAttribute(uvs, 2));
```

**Use interleaved when**: The geometry is large (10000+ vertices), all attributes are updated together or not at all, and GPU throughput is critical. Expect 10-30% improvement on large meshes.

**Use separate when**: Individual attributes are updated independently at different frequencies, or when the geometry is small.

### 1.6 Indexed vs Non-Indexed Geometry

**Indexed geometry** uses an index buffer (`geometry.index`) to reference shared vertices. A cube has 8 unique vertices but 36 indices (6 faces x 2 triangles x 3 vertices).

**Non-indexed geometry** duplicates shared vertices. The same cube would have 36 position entries.

| Aspect | Indexed | Non-Indexed |
|--------|---------|-------------|
| Memory | Lower (shared vertices) | Higher (duplicated vertices) |
| GPU post-transform cache | Better (vertex reuse) | No reuse |
| Flat shading | Requires duplicated normals or groups | Natural (each face gets own normals) |
| Morph targets | Works | Works |
| Wireframe | Clean edges | May show diagonal lines |

**Rule**: ALWAYS use indexed geometry for smooth-shaded meshes with shared vertices. Use `geometry.toNonIndexed()` only when you need per-face attributes (e.g., flat shading with unique normals per face).

### 1.7 Custom Geometry Creation — Step by Step

```javascript
const geometry = new THREE.BufferGeometry();

// Step 1: Define vertex positions (3 values per vertex)
const positions = new Float32Array([
  -1, -1,  0,   // vertex 0
   1, -1,  0,   // vertex 1
   1,  1,  0,   // vertex 2
  -1,  1,  0    // vertex 3
]);
geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

// Step 2: Define indices for two triangles forming a quad
const indices = new Uint16Array([0, 1, 2, 0, 2, 3]);
geometry.setIndex(new THREE.BufferAttribute(indices, 1));

// Step 3: Define UV coordinates (2 values per vertex)
const uvs = new Float32Array([
  0, 0,   // vertex 0
  1, 0,   // vertex 1
  1, 1,   // vertex 2
  0, 1    // vertex 3
]);
geometry.setAttribute('uv', new THREE.BufferAttribute(uvs, 2));

// Step 4: Compute normals automatically
geometry.computeVertexNormals();

// Step 5: Compute bounding volumes for frustum culling
geometry.computeBoundingSphere();
```

### 1.8 Morph Targets

Morph targets enable shape interpolation for facial expressions, blend shapes, and similar effects.

```javascript
// Define morph target positions
const morphPositions = new Float32Array([...]);
geometry.morphAttributes.position = [
  new THREE.BufferAttribute(morphPositions, 3)
];

// On the mesh, control blending
mesh.morphTargetInfluences[0] = 0.5; // 50% blend toward morph target 0

// Named morph targets (e.g., from GLTF)
mesh.morphTargetDictionary; // { "smile": 0, "frown": 1, ... }
```

When `morphTargetsRelative` is `true`, the morph data represents offsets from the base position rather than absolute positions. This is more memory-efficient for subtle deformations.

---

## 2. Built-in Geometries Catalog

All built-in geometries extend `BufferGeometry`. They generate vertex data in their constructor. Below is the complete catalog with constructor signatures and all parameters.

### 2.1 Primitive Geometries

#### BoxGeometry
```javascript
new BoxGeometry(width?: number, height?: number, depth?: number,
                widthSegments?: number, heightSegments?: number, depthSegments?: number)
// Defaults: width=1, height=1, depth=1, widthSegments=1, heightSegments=1, depthSegments=1
```

#### SphereGeometry
```javascript
new SphereGeometry(radius?: number, widthSegments?: number, heightSegments?: number,
                   phiStart?: number, phiLength?: number,
                   thetaStart?: number, thetaLength?: number)
// Defaults: radius=1, widthSegments=32, heightSegments=16,
//           phiStart=0, phiLength=2*PI, thetaStart=0, thetaLength=PI
```

#### PlaneGeometry
```javascript
new PlaneGeometry(width?: number, height?: number,
                  widthSegments?: number, heightSegments?: number)
// Defaults: width=1, height=1, widthSegments=1, heightSegments=1
```

#### CylinderGeometry
```javascript
new CylinderGeometry(radiusTop?: number, radiusBottom?: number, height?: number,
                     radialSegments?: number, heightSegments?: number,
                     openEnded?: boolean, thetaStart?: number, thetaLength?: number)
// Defaults: radiusTop=1, radiusBottom=1, height=1, radialSegments=32,
//           heightSegments=1, openEnded=false, thetaStart=0, thetaLength=2*PI
```

#### ConeGeometry
```javascript
new ConeGeometry(radius?: number, height?: number,
                 radialSegments?: number, heightSegments?: number,
                 openEnded?: boolean, thetaStart?: number, thetaLength?: number)
// Defaults: radius=1, height=1, radialSegments=32, heightSegments=1,
//           openEnded=false, thetaStart=0, thetaLength=2*PI
```

#### CapsuleGeometry
```javascript
new CapsuleGeometry(radius?: number, length?: number,
                    capSegments?: number, radialSegments?: number)
// Defaults: radius=1, length=1, capSegments=4, radialSegments=8
```

#### TorusGeometry
```javascript
new TorusGeometry(radius?: number, tube?: number,
                  radialSegments?: number, tubularSegments?: number, arc?: number)
// Defaults: radius=1, tube=0.4, radialSegments=12, tubularSegments=48, arc=2*PI
```

#### TorusKnotGeometry
```javascript
new TorusKnotGeometry(radius?: number, tube?: number,
                      tubularSegments?: number, radialSegments?: number,
                      p?: number, q?: number)
// Defaults: radius=1, tube=0.4, tubularSegments=64, radialSegments=8, p=2, q=3
```

#### CircleGeometry
```javascript
new CircleGeometry(radius?: number, segments?: number,
                   thetaStart?: number, thetaLength?: number)
// Defaults: radius=1, segments=32, thetaStart=0, thetaLength=2*PI
```

#### RingGeometry
```javascript
new RingGeometry(innerRadius?: number, outerRadius?: number,
                 thetaSegments?: number, phiSegments?: number,
                 thetaStart?: number, thetaLength?: number)
// Defaults: innerRadius=0.5, outerRadius=1, thetaSegments=32,
//           phiSegments=1, thetaStart=0, thetaLength=2*PI
```

### 2.2 Polyhedron Geometries

All polyhedron geometries accept `(radius?: number, detail?: number)`. The `detail` parameter controls subdivision: 0 = base shape, higher values = more triangles (smoother approximation of a sphere).

| Geometry | Base Faces | Constructor |
|----------|-----------|-------------|
| `TetrahedronGeometry` | 4 | `new TetrahedronGeometry(radius=1, detail=0)` |
| `OctahedronGeometry` | 8 | `new OctahedronGeometry(radius=1, detail=0)` |
| `DodecahedronGeometry` | 12 | `new DodecahedronGeometry(radius=1, detail=0)` |
| `IcosahedronGeometry` | 20 | `new IcosahedronGeometry(radius=1, detail=0)` |
| `PolyhedronGeometry` | custom | `new PolyhedronGeometry(vertices, indices, radius=1, detail=0)` |

`PolyhedronGeometry` is the base class. You provide custom vertex positions and face indices, then it projects them onto a sphere and subdivides.

### 2.3 Path-Based Geometries

#### ExtrudeGeometry
```javascript
new ExtrudeGeometry(shapes: Shape | Shape[], options?: Object)
```

**Options (all with defaults):**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `curveSegments` | `number` | `12` | Points on curves |
| `steps` | `number` | `1` | Extrusion subdivision steps |
| `depth` | `number` | `1` | Extrusion depth |
| `bevelEnabled` | `boolean` | `true` | Enable beveled edges |
| `bevelThickness` | `number` | `0.2` | Bevel depth into shape |
| `bevelSize` | `number` | `bevelThickness - 0.1` | Bevel distance from outline |
| `bevelOffset` | `number` | `0` | Bevel offset from edge |
| `bevelSegments` | `number` | `3` | Bevel resolution |
| `extrudePath` | `Curve` | `undefined` | 3D path to extrude along |
| `UVGenerator` | `Object` | `WorldUVGenerator` | Custom UV generation |

#### Shape Class — 2D Path Drawing

`Shape` extends `Path` which extends `CurvePath`. It defines a 2D outline for extrusion.

```javascript
const shape = new THREE.Shape();
shape.moveTo(x, y);                                    // Move pen (start new subpath)
shape.lineTo(x, y);                                    // Straight line
shape.bezierCurveTo(cp1x, cp1y, cp2x, cp2y, x, y);   // Cubic Bezier
shape.quadraticCurveTo(cpx, cpy, x, y);                // Quadratic Bezier
shape.splineThru(points: Vector2[]);                    // Smooth spline through points
shape.arc(aX, aY, aRadius, aStartAngle, aEndAngle, aClockwise); // Relative arc
shape.absarc(aX, aY, aRadius, aStartAngle, aEndAngle, aClockwise); // Absolute arc

// Holes — Array of Path objects
shape.holes.push(holePath);
```

#### ShapeGeometry
```javascript
new ShapeGeometry(shapes: Shape | Shape[], curveSegments?: number)
// Defaults: curveSegments=12
```
Flat 2D geometry from a Shape. No extrusion.

#### LatheGeometry
```javascript
new LatheGeometry(points: Vector2[], segments?: number,
                  phiStart?: number, phiLength?: number)
// Defaults: segments=12, phiStart=0, phiLength=2*PI
```
Revolves a 2D profile around the Y axis. `points` define the profile as an array of `Vector2(x, y)` where x is the radius at height y.

#### TubeGeometry
```javascript
new TubeGeometry(path: Curve, tubularSegments?: number,
                 radius?: number, radialSegments?: number, closed?: boolean)
// Defaults: tubularSegments=64, radius=1, radialSegments=8, closed=false
```
Creates a tube along a `Curve3` (e.g., `CatmullRomCurve3`, `CubicBezierCurve3`, `LineCurve3`).

### 2.4 Utility Geometries

#### EdgesGeometry
```javascript
new EdgesGeometry(geometry: BufferGeometry, thresholdAngle?: number)
// Default: thresholdAngle=1 (degrees)
```
Generates edges only where the angle between adjacent faces exceeds `thresholdAngle`. Produces clean architectural outlines. Use with `LineSegments`.

#### WireframeGeometry
```javascript
new WireframeGeometry(geometry: BufferGeometry)
```
Generates ALL edges including interior triangle edges. Use with `LineSegments`. Results in a complete wireframe of every triangle.

**Rule**: Use `EdgesGeometry` for clean silhouettes (architectural models, CAD). Use `WireframeGeometry` only when you want to see every triangle edge (debug visualization).

---

## 3. InstancedMesh

`InstancedMesh` renders many copies of the same geometry+material with different transforms, using GPU instancing. This is dramatically faster than creating individual `Mesh` objects.

### 3.1 Constructor

```javascript
new InstancedMesh(geometry: BufferGeometry, material: Material, count: number)
```

- `geometry` — Shared geometry (one copy on GPU)
- `material` — Shared material (supports arrays for multi-material)
- `count` — Maximum number of instances (CANNOT be changed after creation)

### 3.2 Properties

| Property | Type | Description |
|----------|------|-------------|
| `instanceMatrix` | `InstancedBufferAttribute` | Float32Array storing 4x4 matrices (16 floats per instance) |
| `instanceColor` | `InstancedBufferAttribute \| null` | Optional RGB color per instance (created on first `setColorAt` call) |
| `count` | `number` | Number of instances. Read-only after construction. |
| `frustumCulled` | `boolean` | If `true`, entire InstancedMesh is culled as one unit |
| `boundingBox` | `Box3 \| null` | Bounding box of the base geometry (NOT instances) |
| `boundingSphere` | `Sphere \| null` | Bounding sphere of the base geometry |

### 3.3 Methods

```javascript
setMatrixAt(index: number, matrix: Matrix4): void
getMatrixAt(index: number, matrix: Matrix4): Matrix4
setColorAt(index: number, color: Color): void
getColorAt(index: number, color: Color): Color
computeBoundingBox(): void
computeBoundingSphere(): void
dispose(): void
```

### 3.4 Critical Update Pattern

```javascript
const dummy = new THREE.Object3D();
const mesh = new THREE.InstancedMesh(geometry, material, 1000);

for (let i = 0; i < 1000; i++) {
  dummy.position.set(Math.random() * 100, 0, Math.random() * 100);
  dummy.rotation.y = Math.random() * Math.PI * 2;
  dummy.scale.setScalar(0.5 + Math.random());
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
}

// CRITICAL: Without this line, instances will all render at origin
mesh.instanceMatrix.needsUpdate = true;
```

**Anti-pattern**: NEVER forget `instanceMatrix.needsUpdate = true` after calling `setMatrixAt`. The GPU buffer will NOT update without this flag. Same applies to `instanceColor.needsUpdate = true` after `setColorAt`.

### 3.5 Custom Instance Attributes

For per-instance data beyond transform and color (e.g., UV offset, animation phase):

```javascript
const phases = new Float32Array(count);
for (let i = 0; i < count; i++) phases[i] = Math.random();

const phaseAttr = new THREE.InstancedBufferAttribute(phases, 1);
mesh.geometry.setAttribute('aPhase', phaseAttr);
// Access in vertex shader: attribute float aPhase;
```

### 3.6 Frustum Culling Considerations

`InstancedMesh` is culled as a single object based on the base geometry bounding sphere. If instances are spread across a large area, the bounding sphere will be very large, and frustum culling becomes ineffective.

**Strategies**:
1. Call `mesh.computeBoundingBox()` / `mesh.computeBoundingSphere()` after setting all matrices
2. For widely spread instances, split into multiple `InstancedMesh` objects grouped by spatial region
3. Set `mesh.frustumCulled = false` only as a last resort

### 3.7 Performance Guidelines

| Instance Count | Recommendation |
|---------------|----------------|
| < 10 | Use individual `Mesh` objects |
| 10 - 100 | Either approach works; profile your specific case |
| 100 - 10000 | ALWAYS use `InstancedMesh` |
| > 10000 | Use `InstancedMesh` with spatial subdivision or `BatchedMesh` (r160+) |

---

## 4. Materials System Deep Dive

### 4.1 Base Material Properties

All materials extend `Material`. These properties are shared across every material type.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `side` | `number` | `FrontSide` | `FrontSide`, `BackSide`, or `DoubleSide` |
| `transparent` | `boolean` | `false` | Enable alpha blending |
| `opacity` | `number` | `1` | Opacity (requires `transparent: true` to take effect below 1) |
| `depthWrite` | `boolean` | `true` | Write to depth buffer |
| `depthTest` | `boolean` | `true` | Test against depth buffer |
| `depthFunc` | `number` | `LessEqualDepth` | Depth comparison function |
| `blending` | `number` | `NormalBlending` | `NoBlending`, `NormalBlending`, `AdditiveBlending`, `SubtractiveBlending`, `MultiplyBlending`, `CustomBlending` |
| `blendSrc` | `number` | `SrcAlphaFactor` | Source blend factor (for `CustomBlending`) |
| `blendDst` | `number` | `OneMinusSrcAlphaFactor` | Destination blend factor |
| `blendEquation` | `number` | `AddEquation` | Blend equation |
| `alphaTest` | `number` | `0` | Discard fragments with alpha below this value |
| `alphaToCoverage` | `boolean` | `false` | Use alpha-to-coverage (MSAA only) |
| `clippingPlanes` | `Plane[] \| null` | `null` | Array of clipping planes |
| `clipIntersection` | `boolean` | `false` | If `true`, clip where ALL planes intersect rather than union |
| `clipShadows` | `boolean` | `false` | Apply clipping to shadows |
| `colorWrite` | `boolean` | `true` | Whether to write color |
| `stencilWrite` | `boolean` | `false` | Enable stencil buffer writing |
| `stencilFunc` | `number` | `AlwaysStencilFunc` | Stencil comparison function |
| `stencilRef` | `number` | `0` | Stencil reference value |
| `stencilWriteMask` | `number` | `0xff` | Stencil write bitmask |
| `stencilFuncMask` | `number` | `0xff` | Stencil function bitmask |
| `stencilFail` | `number` | `KeepStencilOp` | Stencil fail operation |
| `stencilZFail` | `number` | `KeepStencilOp` | Stencil depth fail operation |
| `stencilZPass` | `number` | `KeepStencilOp` | Stencil depth pass operation |
| `polygonOffset` | `boolean` | `false` | Enable polygon offset (for decals, coplanar faces) |
| `polygonOffsetFactor` | `number` | `0` | Polygon offset factor |
| `polygonOffsetUnits` | `number` | `0` | Polygon offset units |
| `visible` | `boolean` | `true` | Whether to render this material |
| `toneMapped` | `boolean` | `true` | Apply renderer tone mapping |
| `needsUpdate` | `boolean` | `false` | Trigger shader recompilation |
| `version` | `number` | `0` | Incremented automatically on changes |

**Base Material Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `clone` | `(): Material` | Clone the material |
| `copy` | `(source: Material): Material` | Copy properties |
| `dispose` | `(): void` | Free GPU resources |
| `onBeforeCompile` | `(shader: Object, renderer: WebGLRenderer): void` | Hook to modify shader before compilation |
| `setValues` | `(values: Object): void` | Set multiple properties at once |
| `toJSON` | `(): Object` | Serialize |

### 4.2 Complete Material Type Catalog

#### Unlit Materials

**MeshBasicMaterial** — No lighting calculations. Renders flat color or texture.
```javascript
new MeshBasicMaterial({
  color: 0xffffff,          // Color (default: white)
  map: null,                // Texture (Texture | null)
  wireframe: false,         // Wireframe mode
  wireframeLinewidth: 1,    // Wireframe line width
  combine: MultiplyOperation, // How envMap combines with color
  reflectivity: 1,          // Environment map reflectivity
  envMap: null,             // Environment map
  fog: true,                // Affected by scene fog
  alphaMap: null,           // Alpha transparency map
  aoMap: null,              // Ambient occlusion map
  aoMapIntensity: 1.0,      // AO intensity
  lightMap: null,           // Baked light map
  lightMapIntensity: 1.0    // Light map intensity
})
```

#### Diffuse-Only Materials

**MeshLambertMaterial** — Diffuse shading only (Lambertian model). No specular highlights. Cheapest lit material.
```javascript
new MeshLambertMaterial({
  color: 0xffffff,
  emissive: 0x000000,
  emissiveIntensity: 1.0,
  emissiveMap: null,
  map: null,
  bumpMap: null,
  bumpScale: 1,
  normalMap: null,
  normalMapType: TangentSpaceNormalMap,
  normalScale: new Vector2(1, 1),
  displacementMap: null,
  displacementScale: 1,
  displacementBias: 0,
  alphaMap: null,
  envMap: null,
  combine: MultiplyOperation,
  reflectivity: 1,
  fog: true,
  wireframe: false
})
```

#### Specular Highlight Materials

**MeshPhongMaterial** — Blinn-Phong model with specular highlights. Cheaper than PBR but less realistic.
```javascript
new MeshPhongMaterial({
  color: 0xffffff,
  specular: 0x111111,       // Specular highlight color
  shininess: 30,            // Specular exponent (higher = tighter highlight)
  emissive: 0x000000,
  emissiveIntensity: 1.0,
  map: null,
  specularMap: null,        // Specular intensity map
  bumpMap: null,
  normalMap: null,
  displacementMap: null,
  envMap: null,
  combine: MultiplyOperation,
  reflectivity: 1,
  fog: true,
  flatShading: false,
  wireframe: false
})
```

#### PBR Materials

**MeshStandardMaterial** — Physically-based rendering with metalness/roughness workflow. This is the recommended material for most use cases.

```javascript
new MeshStandardMaterial({
  color: 0xffffff,              // Diffuse color (default: white)
  roughness: 1.0,               // 0 = mirror, 1 = fully rough
  metalness: 0.0,               // 0 = dielectric, 1 = metal
  map: null,                    // Diffuse/albedo texture
  lightMap: null,               // Baked light map (requires uv2)
  lightMapIntensity: 1.0,       // Light map multiplier
  aoMap: null,                  // Ambient occlusion (requires uv2)
  aoMapIntensity: 1.0,          // AO multiplier
  emissive: 0x000000,           // Emissive color
  emissiveMap: null,             // Emissive texture
  emissiveIntensity: 1.0,       // Emissive multiplier
  bumpMap: null,                // Bump map (grayscale height)
  bumpScale: 1.0,               // Bump intensity
  normalMap: null,              // Normal map
  normalMapType: TangentSpaceNormalMap,  // TangentSpaceNormalMap or ObjectSpaceNormalMap
  normalScale: new Vector2(1, 1), // Normal map intensity (x, y)
  displacementMap: null,        // Vertex displacement map
  displacementScale: 1.0,       // Displacement multiplier
  displacementBias: 0.0,        // Displacement offset
  roughnessMap: null,           // Per-pixel roughness
  metalnessMap: null,           // Per-pixel metalness
  alphaMap: null,               // Alpha transparency map
  envMap: null,                 // Environment reflection map
  envMapIntensity: 1.0,         // Env map multiplier
  wireframe: false,             // Wireframe rendering
  wireframeLinewidth: 1,        // Wireframe line width
  flatShading: false,           // Flat shading mode
  fog: true                     // Affected by fog
})
```

**MeshPhysicalMaterial** — Extends `MeshStandardMaterial` with advanced PBR features. Inherits ALL standard properties plus:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `clearcoat` | `float` | `0.0` | Clear coat layer intensity (0-1) |
| `clearcoatRoughness` | `float` | `0.0` | Clear coat roughness |
| `clearcoatMap` | `Texture` | `null` | Clear coat intensity map |
| `clearcoatRoughnessMap` | `Texture` | `null` | Clear coat roughness map |
| `clearcoatNormalMap` | `Texture` | `null` | Clear coat normal map |
| `clearcoatNormalScale` | `Vector2` | `(1, 1)` | Clear coat normal intensity |
| `ior` | `float` | `1.5` | Index of refraction (1.0 - 2.333) |
| `transmission` | `float` | `0.0` | Transmission/transparency (0-1, physically-based) |
| `transmissionMap` | `Texture` | `null` | Transmission map |
| `thickness` | `float` | `0.0` | Volume thickness for transmission |
| `thicknessMap` | `Texture` | `null` | Thickness map |
| `attenuationDistance` | `float` | `Infinity` | Light attenuation distance in volume |
| `attenuationColor` | `Color` | `white` | Light attenuation tint color |
| `sheen` | `float` | `0.0` | Sheen layer intensity (fabric-like) |
| `sheenColor` | `Color` | `0x000000` | Sheen tint color |
| `sheenColorMap` | `Texture` | `null` | Sheen color map |
| `sheenRoughness` | `float` | `1.0` | Sheen roughness |
| `sheenRoughnessMap` | `Texture` | `null` | Sheen roughness map |
| `iridescence` | `float` | `0.0` | Iridescence intensity (thin-film interference) |
| `iridescenceIOR` | `float` | `1.3` | Iridescence index of refraction |
| `iridescenceThicknessRange` | `[float, float]` | `[100, 400]` | Thin-film thickness range in nanometers |
| `iridescenceMap` | `Texture` | `null` | Iridescence intensity map |
| `iridescenceThicknessMap` | `Texture` | `null` | Iridescence thickness map |
| `anisotropy` | `float` | `0.0` | Anisotropic reflection strength |
| `anisotropyRotation` | `float` | `0.0` | Anisotropy rotation in radians |
| `anisotropyMap` | `Texture` | `null` | Anisotropy direction map |
| `specularIntensity` | `float` | `1.0` | Specular layer intensity |
| `specularIntensityMap` | `Texture` | `null` | Specular intensity map |
| `specularColor` | `Color` | `white` | Specular tint color |
| `specularColorMap` | `Texture` | `null` | Specular color map |
| `reflectivity` | `float` | `0.5` | Reflectivity at normal incidence |
| `dispersion` | `float` | `0.0` | Chromatic dispersion (rainbow effect) |

**Anti-pattern**: NEVER use `MeshPhysicalMaterial` when `MeshStandardMaterial` suffices. Physical material compiles a significantly larger shader and is slower to render. Only use it when you need clearcoat, transmission, sheen, iridescence, or anisotropy.

#### Stylized Materials

**MeshToonMaterial** — Cel-shading / cartoon style with discrete light steps.
```javascript
new MeshToonMaterial({
  color: 0xffffff,
  gradientMap: null,  // Texture defining light steps (use small textures like 3x1 or 5x1)
  map: null,
  normalMap: null,
  bumpMap: null,
  displacementMap: null,
  emissive: 0x000000,
  alphaMap: null,
  wireframe: false
})
```

**Rule**: When using `MeshToonMaterial`, ALWAYS set `gradientMap.minFilter = NearestFilter` and `gradientMap.magFilter = NearestFilter` to get sharp toon shading steps. Linear filtering will blur the steps into smooth shading, defeating the purpose.

**MeshMatcapMaterial** — Uses a matcap (material capture) texture for shading. No lights needed.
```javascript
new MeshMatcapMaterial({
  color: 0xffffff,
  matcap: null,    // Matcap texture (spherical environment map baked into a 2D image)
  map: null,
  bumpMap: null,
  normalMap: null,
  displacementMap: null,
  alphaMap: null,
  flatShading: false,
  fog: true
})
```

#### Utility Materials

**MeshNormalMaterial** — Maps surface normals to RGB colors. Useful for debugging.

**MeshDepthMaterial** — Renders depth from the camera. Used internally for shadow maps.

**MeshDistanceMaterial** — Renders distance from a point light. Used internally for point light shadows.

**ShadowMaterial** — Receives shadows on a transparent surface. Ideal for ground planes that should show shadows but be invisible otherwise.
```javascript
new ShadowMaterial({
  color: 0x000000,    // Shadow color
  transparent: true,  // MUST be true
  opacity: 0.5        // Shadow darkness
})
```

#### Line and Point Materials

**LineBasicMaterial** — Solid colored lines.
```javascript
new LineBasicMaterial({ color: 0xffffff, linewidth: 1 })
// NOTE: linewidth > 1 is NOT supported on most platforms due to WebGL limitations.
// Use Line2/LineMaterial from examples for thick lines.
```

**LineDashedMaterial** — Dashed lines. Requires calling `line.computeLineDistances()`.
```javascript
new LineDashedMaterial({
  color: 0xffffff,
  dashSize: 3,      // Length of dash
  gapSize: 1,       // Length of gap
  scale: 1          // Scale of the dash/gap pattern
})
```

**PointsMaterial** — Renders points (particles).
```javascript
new PointsMaterial({
  color: 0xffffff,
  size: 1,                   // Point size in world units (or pixels if sizeAttenuation=false)
  sizeAttenuation: true,     // If true, size shrinks with distance
  map: null,                 // Sprite texture
  alphaMap: null,
  alphaTest: 0
})
```

**SpriteMaterial** — For `Sprite` objects (always-facing-camera billboards).
```javascript
new SpriteMaterial({
  color: 0xffffff,
  map: null,
  alphaMap: null,
  rotation: 0,        // Rotation in radians
  sizeAttenuation: true
})
```

### 4.3 Material Selection Guide

| Use Case | Material | Why |
|----------|----------|-----|
| UI elements, unlit scenes | `MeshBasicMaterial` | Cheapest, no light computation |
| Matte diffuse objects (low-end devices) | `MeshLambertMaterial` | Fast, no specular |
| Legacy Phong shading | `MeshPhongMaterial` | Specular highlights, not physically correct |
| General-purpose 3D | `MeshStandardMaterial` | PBR, metalness/roughness, industry standard |
| Glass, car paint, fabric, soap bubbles | `MeshPhysicalMaterial` | Advanced PBR features |
| Cartoon/anime style | `MeshToonMaterial` | Discrete lighting steps |
| Sculpting previews | `MeshMatcapMaterial` | No lights needed, great for modeling |
| Debug normals | `MeshNormalMaterial` | Visual normal debugging |
| Invisible shadow receiver | `ShadowMaterial` | Transparent shadow catcher |

---

## 5. Texture System

### 5.1 Texture Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `image` | `HTMLImageElement \| HTMLCanvasElement \| HTMLVideoElement` | — | Source image data |
| `source` | `Source` | — | Source object wrapper |
| `mapping` | `number` | `UVMapping` | How texture maps to surface |
| `channel` | `number` | `0` | UV channel (0 = uv, 1 = uv2) |
| `wrapS` | `number` | `ClampToEdgeWrapping` | Horizontal wrapping |
| `wrapT` | `number` | `ClampToEdgeWrapping` | Vertical wrapping |
| `magFilter` | `number` | `LinearFilter` | Magnification filter |
| `minFilter` | `number` | `LinearMipmapLinearFilter` | Minification filter |
| `anisotropy` | `number` | `1` | Anisotropic filtering (1 = disabled, max = `renderer.capabilities.getMaxAnisotropy()`) |
| `format` | `number` | `RGBAFormat` | Texel format |
| `internalFormat` | `string \| null` | `null` | WebGL internal format |
| `type` | `number` | `UnsignedByteType` | Data type |
| `offset` | `Vector2` | `(0, 0)` | UV offset |
| `repeat` | `Vector2` | `(1, 1)` | UV repeat |
| `rotation` | `number` | `0` | UV rotation in radians |
| `center` | `Vector2` | `(0, 0)` | Center of rotation |
| `matrixAutoUpdate` | `boolean` | `true` | Auto-update UV transform matrix |
| `matrix` | `Matrix3` | identity | UV transform matrix |
| `generateMipmaps` | `boolean` | `true` | Auto-generate mipmaps |
| `premultiplyAlpha` | `boolean` | `false` | Premultiply alpha |
| `flipY` | `boolean` | `true` | Flip vertically on upload |
| `unpackAlignment` | `number` | `4` | Texel row alignment |
| `colorSpace` | `string` | `NoColorSpace` | Color space interpretation |
| `needsUpdate` | `boolean` | `false` | Trigger GPU re-upload |
| `userData` | `Object` | `{}` | Custom data |

### 5.2 Wrapping Modes

| Constant | Description |
|----------|-------------|
| `THREE.ClampToEdgeWrapping` | Edge texels are stretched (default) |
| `THREE.RepeatWrapping` | Texture tiles/repeats |
| `THREE.MirroredRepeatWrapping` | Texture tiles with alternating mirror |

**Rule**: ALWAYS set `wrapS` and `wrapT` to `RepeatWrapping` when using `repeat` values other than (1,1). The default `ClampToEdgeWrapping` will NOT tile the texture.

### 5.3 Filter Modes

**Magnification** (texture smaller than surface area):

| Constant | Description |
|----------|-------------|
| `THREE.NearestFilter` | Pixelated / crisp (retro style) |
| `THREE.LinearFilter` | Smooth interpolation (default) |

**Minification** (texture larger than surface area):

| Constant | Description |
|----------|-------------|
| `THREE.NearestFilter` | Nearest texel, no mipmap |
| `THREE.NearestMipmapNearestFilter` | Nearest texel, nearest mipmap |
| `THREE.NearestMipmapLinearFilter` | Nearest texel, linear mipmap blend |
| `THREE.LinearFilter` | Linear texel, no mipmap |
| `THREE.LinearMipmapNearestFilter` | Linear texel, nearest mipmap |
| `THREE.LinearMipmapLinearFilter` | Trilinear filtering (default, best quality) |

### 5.4 Map Types in PBR Materials

| Map Property | Color Space | Channel(s) | Description |
|-------------|-------------|------------|-------------|
| `map` | `SRGBColorSpace` | RGB(A) | Base color / albedo |
| `emissiveMap` | `SRGBColorSpace` | RGB | Emissive glow |
| `normalMap` | `NoColorSpace` | RGB | Surface normal perturbation |
| `roughnessMap` | `NoColorSpace` | G | Per-pixel roughness (green channel) |
| `metalnessMap` | `NoColorSpace` | B | Per-pixel metalness (blue channel) |
| `aoMap` | `NoColorSpace` | R | Ambient occlusion (red channel, requires uv2) |
| `bumpMap` | `NoColorSpace` | R | Height-based surface perturbation |
| `displacementMap` | `NoColorSpace` | R | Actual vertex displacement |
| `alphaMap` | `NoColorSpace` | R | Per-pixel transparency |
| `lightMap` | `SRGBColorSpace` | RGB | Baked lighting (requires uv2) |
| `envMap` | `SRGBColorSpace` | RGB | Environment reflections |
| `clearcoatMap` | `NoColorSpace` | R | Clearcoat intensity |
| `clearcoatRoughnessMap` | `NoColorSpace` | R | Clearcoat roughness |
| `clearcoatNormalMap` | `NoColorSpace` | RGB | Clearcoat normal |
| `transmissionMap` | `NoColorSpace` | R | Transmission intensity |
| `thicknessMap` | `NoColorSpace` | R | Volume thickness |
| `sheenColorMap` | `SRGBColorSpace` | RGB | Sheen tint |
| `sheenRoughnessMap` | `NoColorSpace` | R | Sheen roughness |
| `iridescenceMap` | `NoColorSpace` | R | Iridescence intensity |
| `iridescenceThicknessMap` | `NoColorSpace` | R | Thin-film thickness |
| `anisotropyMap` | `NoColorSpace` | RG | Anisotropy direction |
| `specularIntensityMap` | `NoColorSpace` | A | Specular intensity |
| `specularColorMap` | `SRGBColorSpace` | RGB | Specular tint |

### 5.5 Color Space Rules

**Critical rule**: Getting color spaces wrong is the most common source of washed-out or over-saturated rendering.

- **Diffuse/color textures** (map, emissiveMap, sheenColorMap, specularColorMap, envMap, lightMap): ALWAYS set `texture.colorSpace = THREE.SRGBColorSpace`
- **Data textures** (normalMap, roughnessMap, metalnessMap, aoMap, bumpMap, displacementMap, alphaMap, and all non-color maps): ALWAYS leave at `NoColorSpace` (or `LinearSRGBColorSpace`)
- **HDR environment maps** loaded via `RGBELoader`: The loader sets color space automatically
- The renderer performs linear-space lighting calculations and converts to display color space at output

**Anti-pattern**: NEVER set `SRGBColorSpace` on a normal map. This will corrupt the normal vectors and cause incorrect lighting.

### 5.6 Normal Map Types

| Constant | Description |
|----------|-------------|
| `THREE.TangentSpaceNormalMap` | Most common, works with any mesh orientation (default) |
| `THREE.ObjectSpaceNormalMap` | Faster but tied to mesh orientation, breaks with animation |

**Rule**: ALWAYS use `TangentSpaceNormalMap` unless you have a specific reason for object-space (e.g., static architecture with baked normals).

### 5.7 Texture Loaders

| Loader | Format | Import |
|--------|--------|--------|
| `TextureLoader` | PNG, JPG, WebP | `three` core |
| `CubeTextureLoader` | 6x PNG/JPG for cube maps | `three` core |
| `RGBELoader` | .hdr (Radiance HDR) | `three/addons/loaders/RGBELoader` |
| `EXRLoader` | .exr (OpenEXR HDR) | `three/addons/loaders/EXRLoader` |
| `KTX2Loader` | .ktx2 (GPU compressed: BC, ETC, ASTC) | `three/addons/loaders/KTX2Loader` |
| `CompressedTextureLoader` | DDS, PVR | `three/addons/loaders/CompressedTextureLoader` |

```javascript
// Standard texture loading
const loader = new THREE.TextureLoader();
const texture = loader.load('diffuse.jpg', (tex) => {
  tex.colorSpace = THREE.SRGBColorSpace;
  tex.wrapS = THREE.RepeatWrapping;
  tex.wrapT = THREE.RepeatWrapping;
});

// HDR environment map
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
const rgbeLoader = new RGBELoader();
rgbeLoader.load('env.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = texture;  // PBR environment lighting
  scene.background = texture;   // Optional: visible background
});
```

### 5.8 flipY

- `flipY = true` (default): Correct for loaded image textures (PNG, JPG)
- `flipY = false`: Required for `WebGLRenderTarget` textures, `DataTexture`, and textures from framebuffers

**Anti-pattern**: NEVER set `flipY = true` on a render target texture. The image will be upside down.

### 5.9 Anisotropic Filtering

```javascript
texture.anisotropy = renderer.capabilities.getMaxAnisotropy(); // Usually 16
```

Anisotropic filtering improves texture quality at oblique viewing angles (e.g., textured floors). Higher values = better quality but slightly more GPU cost. ALWAYS enable for ground textures and surfaces viewed at grazing angles.

---

## 6. Custom Shaders (ShaderMaterial and RawShaderMaterial)

### 6.1 ShaderMaterial

`ShaderMaterial` provides custom GLSL shaders while automatically injecting Three.js built-in uniforms and attributes.

#### Constructor

```javascript
new ShaderMaterial({
  uniforms: { uTime: { value: 0.0 } },
  vertexShader: '...',
  fragmentShader: '...',
  defines: {},
  extensions: {},
  wireframe: false,
  wireframeLinewidth: 1,
  lights: false,
  fog: false,
  clipping: false,
  glslVersion: null,    // null for GLSL1, THREE.GLSL3 for WebGL2
  defaultAttributeValues: { color: [1, 1, 1] }
})
```

#### Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `uniforms` | `Object` | `{}` | `{ name: { value: ... } }` format |
| `uniformsGroups` | `Array` | `[]` | Uniform buffer objects (UBO) |
| `vertexShader` | `string` | — | GLSL vertex shader source |
| `fragmentShader` | `string` | — | GLSL fragment shader source |
| `defines` | `Object` | `{}` | Preprocessor `#define` directives |
| `extensions` | `Object` | `{}` | GLSL extensions to enable |
| `wireframe` | `boolean` | `false` | Wireframe rendering |
| `wireframeLinewidth` | `number` | `1` | Wireframe line width |
| `lights` | `boolean` | `false` | Pass light uniforms to shader |
| `fog` | `boolean` | `false` | Pass fog uniforms to shader |
| `clipping` | `boolean` | `false` | Enable clipping planes |
| `glslVersion` | `string \| null` | `null` | `null` for GLSL1, `THREE.GLSL3` for GLSL 3.0 ES |
| `defaultAttributeValues` | `Object` | — | Fallback values for missing attributes |

### 6.2 Built-in Uniforms (Automatically Provided by Three.js)

These uniforms are injected into `ShaderMaterial` shaders. You do NOT declare them yourself.

```glsl
// Transform matrices
uniform mat4 modelMatrix;           // Object -> World
uniform mat4 modelViewMatrix;       // Object -> Camera (modelMatrix * viewMatrix)
uniform mat4 projectionMatrix;      // Camera -> Clip
uniform mat4 viewMatrix;            // World -> Camera
uniform mat3 normalMatrix;          // Transpose inverse of modelViewMatrix (for normals)

// Camera
uniform vec3 cameraPosition;        // Camera world position

// When lights: true
uniform vec3 ambientLightColor;
// Plus structured arrays for directional, point, spot, hemisphere, and rect area lights
```

### 6.3 Built-in Attributes (Automatically Provided)

```glsl
attribute vec3 position;    // Vertex position
attribute vec3 normal;      // Vertex normal
attribute vec2 uv;          // Primary UV coordinates
attribute vec2 uv2;         // Secondary UV (for aoMap, lightMap)
attribute vec4 tangent;     // Tangent vector (if computeTangents was called)
attribute vec3 color;       // Vertex color (if geometry has color attribute)
```

### 6.4 Uniform Types

| GLSL Type | JavaScript Value |
|-----------|-----------------|
| `float` | `{ value: 1.0 }` |
| `int` | `{ value: 1 }` |
| `bool` | `{ value: true }` |
| `vec2` | `{ value: new THREE.Vector2() }` |
| `vec3` | `{ value: new THREE.Vector3() }` or `{ value: new THREE.Color() }` |
| `vec4` | `{ value: new THREE.Vector4() }` |
| `mat3` | `{ value: new THREE.Matrix3() }` |
| `mat4` | `{ value: new THREE.Matrix4() }` |
| `sampler2D` | `{ value: texture }` (a `THREE.Texture`) |
| `samplerCube` | `{ value: cubeTexture }` |
| `float[]` | `{ value: [1.0, 2.0, 3.0] }` |
| `vec3[]` | `{ value: [new THREE.Vector3(), ...] }` |

### 6.5 ShaderMaterial vs RawShaderMaterial

| Aspect | ShaderMaterial | RawShaderMaterial |
|--------|----------------|-------------------|
| Built-in uniforms | Automatically injected | NONE — you must declare everything |
| Built-in attributes | Automatically declared | NONE — you must declare everything |
| `#include <chunk>` | Supported | NOT supported |
| Precision declaration | Automatic | You MUST add `precision mediump float;` |
| Use case | Extend Three.js rendering | Full shader control, porting external shaders |
| Performance | Slight overhead from unused built-ins | Minimal shader overhead |

### 6.6 ShaderChunk — Reusing Three.js Shader Code

`ShaderChunk` is an object containing all of Three.js internal shader fragments. You can include them in `ShaderMaterial` (NOT `RawShaderMaterial`):

```glsl
// In vertex shader
#include <common>
#include <fog_pars_vertex>
#include <shadowmap_pars_vertex>

void main() {
  #include <begin_vertex>
  // ... your modifications ...
  #include <project_vertex>
  #include <fog_vertex>
  #include <shadowmap_vertex>
}
```

Common chunks:
- `<common>` — Shared constants and functions (PI, saturate, etc.)
- `<fog_pars_vertex>` / `<fog_vertex>` / `<fog_pars_fragment>` / `<fog_fragment>` — Fog support
- `<shadowmap_pars_vertex>` / `<shadowmap_vertex>` / `<shadowmap_pars_fragment>` / `<shadowmap_fragment>` — Shadow support
- `<lights_pars_begin>` — Light structure declarations
- `<normal_fragment_begin>` / `<normal_fragment_maps>` — Normal mapping
- `<color_pars_vertex>` / `<color_vertex>` / `<color_pars_fragment>` / `<color_fragment>` — Vertex color

### 6.7 onBeforeCompile — Patching Built-in Materials

The most powerful shader customization technique: modify an existing material's shader at compile time.

```javascript
const material = new THREE.MeshStandardMaterial({ color: 0xff0000 });

material.onBeforeCompile = (shader) => {
  // Add custom uniform
  shader.uniforms.uTime = { value: 0.0 };

  // Inject code into vertex shader
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `
    #include <begin_vertex>
    transformed.y += sin(transformed.x * 5.0 + uTime) * 0.5;
    `
  );

  // Store reference to update uniform later
  material.userData.shader = shader;
};

// In animation loop
if (material.userData.shader) {
  material.userData.shader.uniforms.uTime.value = clock.getElapsedTime();
}
```

**Rule**: ALWAYS use `onBeforeCompile` when you want to add effects (like vertex displacement, custom fog, or UV animation) to a standard material without rewriting the entire shader. This preserves PBR lighting, shadows, and all built-in features.

**Anti-pattern**: NEVER store a permanent reference to the `shader` parameter outside of `onBeforeCompile`. The shader object can be recreated when `material.needsUpdate = true`. Store it in `userData` and check for existence before accessing.

### 6.8 Defines — Preprocessor Directives

```javascript
const material = new THREE.ShaderMaterial({
  defines: {
    USE_FOG: '',           // #define USE_FOG
    MAX_LIGHTS: 4,         // #define MAX_LIGHTS 4
    EPSILON: '0.001'       // #define EPSILON 0.001
  }
});
```

Changing `defines` requires setting `material.needsUpdate = true` to trigger recompilation.

### 6.9 GLSL3 Mode

```javascript
const material = new THREE.ShaderMaterial({
  glslVersion: THREE.GLSL3,
  vertexShader: `
    // GLSL 3.0 ES syntax
    in vec3 position;
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    out vec3 vPosition;

    void main() {
      vPosition = position;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    precision highp float;
    in vec3 vPosition;
    out vec4 fragColor;

    void main() {
      fragColor = vec4(vPosition * 0.5 + 0.5, 1.0);
    }
  `
});
```

**Key differences in GLSL3**: `attribute` becomes `in`, `varying` becomes `in`/`out`, `texture2D()` becomes `texture()`, `gl_FragColor` is replaced by a declared `out` variable.

---

## 7. Key Anti-Patterns Summary

| Anti-Pattern | Consequence | Correct Approach |
|-------------|-------------|-----------------|
| Forgetting `needsUpdate = true` on BufferAttribute | GPU shows stale data | ALWAYS set after modifying attribute data |
| Using `MeshPhysicalMaterial` everywhere | Wasted GPU cycles on complex shader | Use `MeshStandardMaterial` unless you need physical features |
| Setting `SRGBColorSpace` on normal/roughness maps | Corrupted surface data | ONLY set SRGB on diffuse/emissive/color maps |
| Not calling `computeVertexNormals()` on custom geometry | Black or unlit surfaces | ALWAYS compute normals for lit materials |
| Creating 1000 individual Mesh objects | Draw call explosion, low FPS | Use `InstancedMesh` for >100 identical objects |
| Modifying geometry transforms in animation loop | Permanent cumulative transform | Use `mesh.position/rotation/scale` for runtime transforms |
| `linewidth > 1` on `LineBasicMaterial` | Silently ignored on most platforms | Use `Line2` + `LineMaterial` from examples |
| Not disposing textures/materials/geometries | GPU memory leak | ALWAYS call `.dispose()` when removing from scene |
| Forgetting `transparent: true` with `opacity < 1` | Opacity setting is ignored | ALWAYS pair `opacity` with `transparent: true` |
| Using `RepeatWrapping` without power-of-two textures | WebGL1 warning, broken tiling | Use power-of-two textures or WebGL2 |

---

## 8. Sources

All information in this document was cross-referenced against the official Three.js documentation:

- BufferGeometry: https://threejs.org/docs/#api/en/core/BufferGeometry
- BufferAttribute: https://threejs.org/docs/#api/en/core/BufferAttribute
- InterleavedBuffer: https://threejs.org/docs/#api/en/core/InterleavedBuffer
- InstancedMesh: https://threejs.org/docs/#api/en/objects/InstancedMesh
- Material: https://threejs.org/docs/#api/en/materials/Material
- MeshStandardMaterial: https://threejs.org/docs/#api/en/materials/MeshStandardMaterial
- MeshPhysicalMaterial: https://threejs.org/docs/#api/en/materials/MeshPhysicalMaterial
- Texture: https://threejs.org/docs/#api/en/textures/Texture
- ShaderMaterial: https://threejs.org/docs/#api/en/materials/ShaderMaterial
- RawShaderMaterial: https://threejs.org/docs/#api/en/materials/RawShaderMaterial
- ExtrudeGeometry: https://threejs.org/docs/#api/en/geometries/ExtrudeGeometry

Three.js version scope: r160+ (ES modules, MIT license).
