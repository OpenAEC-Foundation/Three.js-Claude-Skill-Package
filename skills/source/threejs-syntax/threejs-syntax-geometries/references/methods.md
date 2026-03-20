# API Signatures Reference (Three.js Geometries r160+)

## BufferGeometry

```javascript
import { BufferGeometry } from 'three';
```

### Constructor

```javascript
new BufferGeometry()
// Creates an empty geometry. ALWAYS add attributes via setAttribute() before rendering.
```

### Properties

```javascript
geometry.id: number              // Auto-incremented unique identifier
geometry.uuid: string            // Auto-generated UUID
geometry.name: string            // Optional human-readable name (default: "")
geometry.type: string            // Class type identifier (default: "BufferGeometry")
geometry.attributes: Object      // Hash map of named BufferAttribute instances
geometry.index: BufferAttribute | null  // Optional index buffer (default: null)
geometry.morphAttributes: Object // Hash map of morph target attribute arrays (default: {})
geometry.morphTargetsRelative: boolean  // true = morph data is relative offsets (default: false)
geometry.groups: Array           // Array of { start, count, materialIndex } (default: [])
geometry.drawRange: Object       // { start: 0, count: Infinity }
geometry.boundingBox: Box3 | null       // null until computeBoundingBox() is called
geometry.boundingSphere: Sphere | null  // null until computeBoundingSphere() is called
geometry.userData: Object        // Arbitrary user data storage (default: {})
```

### Attribute Management Methods

```javascript
setAttribute(name: string, attribute: BufferAttribute): BufferGeometry
// Add or replace a named attribute. Returns this for chaining.

getAttribute(name: string): BufferAttribute | undefined
// Retrieve attribute by name.

deleteAttribute(name: string): BufferGeometry
// Remove a named attribute. Returns this for chaining.

hasAttribute(name: string): boolean
// Check if attribute exists.

setIndex(index: BufferAttribute | number[]): void
// Set the index buffer. Accepts BufferAttribute or plain array (auto-converted).
```

### Bounding Volume Methods

```javascript
computeBoundingBox(): void
// Compute and cache the axis-aligned bounding box.

computeBoundingSphere(): void
// Compute and cache the bounding sphere.
```

### Normal and Tangent Methods

```javascript
computeVertexNormals(): void
// Compute smooth vertex normals from face topology.
// ALWAYS call this after building custom geometry if lighting is needed.

computeTangents(): void
// Compute tangent vectors. REQUIRES position, normal, uv attributes AND an index buffer.
// Fails silently if any prerequisite is missing.

normalizeNormals(): void
// Normalize all normal vectors to unit length.
```

### Group Management (Multi-Material)

```javascript
addGroup(start: number, count: number, materialIndex?: number): void
// Define a render group. start/count refer to index positions (indexed) or vertex positions (non-indexed).

clearGroups(): void
// Remove all groups.
```

### Draw Range

```javascript
setDrawRange(start: number, count: number): void
// Limit which vertices/indices are rendered. Useful for progressive rendering or LOD.
```

### Transform Methods (All return `this`, modify vertex data in-place)

```javascript
translate(x: number, y: number, z: number): BufferGeometry
rotateX(radians: number): BufferGeometry
rotateY(radians: number): BufferGeometry
rotateZ(radians: number): BufferGeometry
scale(x: number, y: number, z: number): BufferGeometry
center(): BufferGeometry    // Center geometry at origin
lookAt(vector: Vector3): BufferGeometry  // Orient geometry to face a point
```

### Conversion and Serialization

```javascript
toNonIndexed(): BufferGeometry  // Create non-indexed copy (duplicates shared vertices)
clone(): BufferGeometry         // Deep copy the geometry
copy(source: BufferGeometry): BufferGeometry  // Copy attributes from source
toJSON(): Object                // Serialize to JSON
dispose(): void                 // Free GPU resources
```

---

## BufferAttribute

```javascript
import { BufferAttribute } from 'three';
```

### Constructor

```javascript
new BufferAttribute(array: TypedArray, itemSize: number, normalized?: boolean)
// array: Float32Array, Uint16Array, etc.
// itemSize: values per vertex (1=scalar, 2=UV, 3=position/normal, 4=RGBA)
// normalized: if true, integer values mapped to [0,1] or [-1,1] on GPU (default: false)
```

### Properties

```javascript
attribute.array: TypedArray       // The underlying data
attribute.itemSize: number        // Values per vertex
attribute.count: number           // Computed: array.length / itemSize
attribute.normalized: boolean     // Normalize integer data (default: false)
attribute.usage: number           // GPU usage hint (default: StaticDrawUsage)
attribute.needsUpdate: boolean    // Set true to trigger GPU upload (default: false)
attribute.name: string            // Optional identifier (default: "")
attribute.version: number         // Auto-incremented on needsUpdate = true
```

### Accessor Methods

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

### Other Methods

```javascript
clone(): BufferAttribute
copy(source: BufferAttribute): this
copyArray(array: TypedArray): this
copyAt(index1: number, attribute: BufferAttribute, index2: number): this
set(value: TypedArray, offset?: number): this
```

### Typed Convenience Constructors

```javascript
new Float32BufferAttribute(array: number[] | Float32Array, itemSize: number, normalized?: boolean)
new Float16BufferAttribute(array: number[] | Float16Array, itemSize: number, normalized?: boolean)
new Int8BufferAttribute(array: number[] | Int8Array, itemSize: number, normalized?: boolean)
new Int16BufferAttribute(array: number[] | Int16Array, itemSize: number, normalized?: boolean)
new Int32BufferAttribute(array: number[] | Int32Array, itemSize: number, normalized?: boolean)
new Uint8BufferAttribute(array: number[] | Uint8Array, itemSize: number, normalized?: boolean)
new Uint8ClampedBufferAttribute(array: number[] | Uint8ClampedArray, itemSize: number, normalized?: boolean)
new Uint16BufferAttribute(array: number[] | Uint16Array, itemSize: number, normalized?: boolean)
new Uint32BufferAttribute(array: number[] | Uint32Array, itemSize: number, normalized?: boolean)
```

---

## InterleavedBuffer

```javascript
import { InterleavedBuffer } from 'three';

new InterleavedBuffer(array: TypedArray, stride: number)
// array: single typed array containing all interleaved data
// stride: number of values between consecutive entries of the same attribute
```

### Properties

```javascript
buffer.array: TypedArray
buffer.stride: number
buffer.count: number      // array.length / stride
buffer.usage: number      // GPU usage hint
buffer.needsUpdate: boolean
```

---

## InterleavedBufferAttribute

```javascript
import { InterleavedBufferAttribute } from 'three';

new InterleavedBufferAttribute(
  interleavedBuffer: InterleavedBuffer,
  itemSize: number,
  offset: number,
  normalized?: boolean
)
// interleavedBuffer: parent InterleavedBuffer
// itemSize: values per vertex for this attribute
// offset: starting position within each stride
```

---

## InstancedMesh

```javascript
import { InstancedMesh } from 'three';
```

### Constructor

```javascript
new InstancedMesh(geometry: BufferGeometry, material: Material, count: number)
// geometry: shared geometry (one copy on GPU)
// material: shared material (supports arrays for multi-material)
// count: maximum number of instances (CANNOT be changed after creation)
```

### Properties

```javascript
mesh.instanceMatrix: InstancedBufferAttribute  // 4x4 matrices (16 floats per instance)
mesh.instanceColor: InstancedBufferAttribute | null  // Per-instance RGB (null until first setColorAt)
mesh.count: number          // Number of instances (read-only after construction)
mesh.frustumCulled: boolean // Entire InstancedMesh culled as one unit (default: true)
mesh.boundingBox: Box3 | null
mesh.boundingSphere: Sphere | null
```

### Methods

```javascript
setMatrixAt(index: number, matrix: Matrix4): void
// Set transform matrix for instance at index.

getMatrixAt(index: number, matrix: Matrix4): Matrix4
// Read transform matrix for instance into target Matrix4.

setColorAt(index: number, color: Color): void
// Set per-instance color. Creates instanceColor on first call.

getColorAt(index: number, color: Color): Color
// Read per-instance color into target Color.

computeBoundingBox(): void
computeBoundingSphere(): void
dispose(): void
```

---

## InstancedBufferAttribute

```javascript
import { InstancedBufferAttribute } from 'three';

new InstancedBufferAttribute(array: TypedArray, itemSize: number, normalized?: boolean, meshPerAttribute?: number)
// meshPerAttribute: number of meshes per attribute value (default: 1)
```

---

## Shape

```javascript
import { Shape } from 'three';
```

### Constructor

```javascript
new Shape(points?: Vector2[])
// Creates a shape. If points provided, creates a shape from them.
```

### Path Drawing Methods (inherited from Path)

```javascript
shape.moveTo(x: number, y: number): this
shape.lineTo(x: number, y: number): this
shape.bezierCurveTo(cp1x: number, cp1y: number, cp2x: number, cp2y: number, x: number, y: number): this
shape.quadraticCurveTo(cpx: number, cpy: number, x: number, y: number): this
shape.splineThru(points: Vector2[]): this
shape.arc(aX: number, aY: number, aRadius: number, aStartAngle: number, aEndAngle: number, aClockwise?: boolean): this
shape.absarc(aX: number, aY: number, aRadius: number, aStartAngle: number, aEndAngle: number, aClockwise?: boolean): this
shape.absellipse(aX: number, aY: number, xRadius: number, yRadius: number, aStartAngle: number, aEndAngle: number, aClockwise?: boolean, aRotation?: number): this
```

### Shape-Specific Properties

```javascript
shape.holes: Path[]  // Array of hole paths to subtract from the shape
shape.uuid: string
```

### Shape-Specific Methods

```javascript
shape.getPointsHoles(divisions: number): Vector2[][]
shape.extractPoints(divisions: number): { shape: Vector2[], holes: Vector2[][] }
```

---

## Built-in Geometry Constructors — Complete Signatures

### Primitives

```javascript
new BoxGeometry(width?: number, height?: number, depth?: number,
                widthSegments?: number, heightSegments?: number, depthSegments?: number)

new SphereGeometry(radius?: number, widthSegments?: number, heightSegments?: number,
                   phiStart?: number, phiLength?: number, thetaStart?: number, thetaLength?: number)

new PlaneGeometry(width?: number, height?: number, widthSegments?: number, heightSegments?: number)

new CylinderGeometry(radiusTop?: number, radiusBottom?: number, height?: number,
                     radialSegments?: number, heightSegments?: number, openEnded?: boolean,
                     thetaStart?: number, thetaLength?: number)

new ConeGeometry(radius?: number, height?: number, radialSegments?: number,
                 heightSegments?: number, openEnded?: boolean, thetaStart?: number, thetaLength?: number)

new CapsuleGeometry(radius?: number, length?: number, capSegments?: number, radialSegments?: number)

new TorusGeometry(radius?: number, tube?: number, radialSegments?: number,
                  tubularSegments?: number, arc?: number)

new TorusKnotGeometry(radius?: number, tube?: number, tubularSegments?: number,
                      radialSegments?: number, p?: number, q?: number)

new CircleGeometry(radius?: number, segments?: number, thetaStart?: number, thetaLength?: number)

new RingGeometry(innerRadius?: number, outerRadius?: number, thetaSegments?: number,
                 phiSegments?: number, thetaStart?: number, thetaLength?: number)
```

### Polyhedra

```javascript
new TetrahedronGeometry(radius?: number, detail?: number)
new OctahedronGeometry(radius?: number, detail?: number)
new DodecahedronGeometry(radius?: number, detail?: number)
new IcosahedronGeometry(radius?: number, detail?: number)
new PolyhedronGeometry(vertices: number[], indices: number[], radius?: number, detail?: number)
```

### Path-Based

```javascript
new ExtrudeGeometry(shapes: Shape | Shape[], options?: ExtrudeGeometryOptions)
new ShapeGeometry(shapes: Shape | Shape[], curveSegments?: number)
new LatheGeometry(points: Vector2[], segments?: number, phiStart?: number, phiLength?: number)
new TubeGeometry(path: Curve, tubularSegments?: number, radius?: number,
                 radialSegments?: number, closed?: boolean)
```

### Utility

```javascript
new EdgesGeometry(geometry: BufferGeometry, thresholdAngle?: number)
new WireframeGeometry(geometry: BufferGeometry)
```
