# API Signatures Reference (Three.js r160+ Math)

## Vector3

```typescript
class Vector3 {
  constructor(x?: number, y?: number, z?: number)

  // Properties
  x: number
  y: number
  z: number
  readonly isVector3: true

  // Assignment
  set(x: number, y: number, z: number): this
  setScalar(scalar: number): this
  setX(x: number): this
  setY(y: number): this
  setZ(z: number): this
  setComponent(index: 0 | 1 | 2, value: number): this
  getComponent(index: 0 | 1 | 2): number
  copy(v: Vector3): this
  clone(): Vector3

  // Arithmetic
  add(v: Vector3): this
  addScalar(s: number): this
  addVectors(a: Vector3, b: Vector3): this
  addScaledVector(v: Vector3, s: number): this
  sub(v: Vector3): this
  subScalar(s: number): this
  subVectors(a: Vector3, b: Vector3): this
  multiply(v: Vector3): this
  multiplyScalar(s: number): this
  multiplyVectors(a: Vector3, b: Vector3): this
  divide(v: Vector3): this
  divideScalar(s: number): this
  negate(): this

  // Geometric
  dot(v: Vector3): number
  cross(v: Vector3): this
  crossVectors(a: Vector3, b: Vector3): this
  length(): number
  lengthSq(): number
  manhattanLength(): number
  normalize(): this
  setLength(length: number): this
  reflect(normal: Vector3): this
  angleTo(v: Vector3): number
  projectOnVector(v: Vector3): this
  projectOnPlane(planeNormal: Vector3): this
  project(camera: Camera): this
  unproject(camera: Camera): this

  // Distance
  distanceTo(v: Vector3): number
  distanceToSquared(v: Vector3): number
  manhattanDistanceTo(v: Vector3): number

  // Interpolation
  lerp(v: Vector3, alpha: number): this
  lerpVectors(v1: Vector3, v2: Vector3, alpha: number): this
  clamp(min: Vector3, max: Vector3): this
  clampLength(min: number, max: number): this
  clampScalar(minVal: number, maxVal: number): this

  // Transform
  applyMatrix3(m: Matrix3): this
  applyMatrix4(m: Matrix4): this
  applyNormalMatrix(m: Matrix3): this
  applyQuaternion(q: Quaternion): this
  applyAxisAngle(axis: Vector3, angle: number): this
  applyEuler(euler: Euler): this
  transformDirection(m: Matrix4): this

  // Conversion
  setFromMatrixPosition(m: Matrix4): this
  setFromMatrixScale(m: Matrix4): this
  setFromMatrixColumn(m: Matrix4, index: number): this
  setFromMatrix3Column(m: Matrix3, index: number): this
  setFromSpherical(s: Spherical): this
  setFromSphericalCoords(radius: number, phi: number, theta: number): this
  setFromCylindrical(c: Cylindrical): this
  setFromCylindricalCoords(radius: number, theta: number, y: number): this

  // Utility
  equals(v: Vector3): boolean
  toArray(array?: number[], offset?: number): number[]
  fromArray(array: number[], offset?: number): this
  fromBufferAttribute(attribute: BufferAttribute, index: number): this
  min(v: Vector3): this
  max(v: Vector3): this
  floor(): this
  ceil(): this
  round(): this
  roundToZero(): this
  random(): this
}
```

**Key behavior:** All methods that return `this` mutate the instance in-place. ALWAYS use `clone()` before modifying a shared vector.

---

## Vector2

```typescript
class Vector2 {
  constructor(x?: number, y?: number)

  x: number
  y: number

  // Same arithmetic/utility pattern as Vector3 but 2D only
  set(x: number, y: number): this
  add(v: Vector2): this
  sub(v: Vector2): this
  multiply(v: Vector2): this
  multiplyScalar(s: number): this
  divide(v: Vector2): this
  divideScalar(s: number): this
  dot(v: Vector2): number
  cross(v: Vector2): number  // Returns scalar (2D cross product)
  length(): number
  lengthSq(): number
  normalize(): this
  lerp(v: Vector2, alpha: number): this
  distanceTo(v: Vector2): number
  angle(): number  // Angle in radians from positive X axis
  rotateAround(center: Vector2, angle: number): this
  clone(): Vector2
  copy(v: Vector2): this
  equals(v: Vector2): boolean
  toArray(array?: number[], offset?: number): number[]
  fromArray(array: number[], offset?: number): this
}
```

---

## Vector4

```typescript
class Vector4 {
  constructor(x?: number, y?: number, z?: number, w?: number)

  x: number
  y: number
  z: number
  w: number

  // Same arithmetic pattern as Vector3 but 4D
  set(x: number, y: number, z: number, w: number): this
  add(v: Vector4): this
  sub(v: Vector4): this
  multiplyScalar(s: number): this
  divideScalar(s: number): this
  dot(v: Vector4): number
  length(): number
  normalize(): this
  lerp(v: Vector4, alpha: number): this
  applyMatrix4(m: Matrix4): this  // Transforms as homogeneous coordinate
  clone(): Vector4
  copy(v: Vector4): this
  equals(v: Vector4): boolean
}
```

---

## Matrix4

```typescript
class Matrix4 {
  constructor()

  // Properties
  elements: number[]  // 16 floats, column-major order
  readonly isMatrix4: true

  // Assignment
  set(
    n11: number, n12: number, n13: number, n14: number,
    n21: number, n22: number, n23: number, n24: number,
    n31: number, n32: number, n33: number, n34: number,
    n41: number, n42: number, n43: number, n44: number
  ): this
  identity(): this
  copy(m: Matrix4): this
  clone(): Matrix4

  // Composition
  compose(position: Vector3, quaternion: Quaternion, scale: Vector3): this
  decompose(position: Vector3, quaternion: Quaternion, scale: Vector3): this

  // Multiplication
  multiply(m: Matrix4): this             // this = this * m (right-multiply)
  premultiply(m: Matrix4): this          // this = m * this (left-multiply)
  multiplyMatrices(a: Matrix4, b: Matrix4): this  // this = a * b
  multiplyScalar(s: number): this

  // Factory methods
  makeTranslation(x: number, y: number, z: number): this
  makeTranslation(v: Vector3): this
  makeScale(x: number, y: number, z: number): this
  makeRotationX(theta: number): this
  makeRotationY(theta: number): this
  makeRotationZ(theta: number): this
  makeRotationAxis(axis: Vector3, angle: number): this
  makeRotationFromEuler(euler: Euler): this
  makeRotationFromQuaternion(q: Quaternion): this
  makeBasis(xAxis: Vector3, yAxis: Vector3, zAxis: Vector3): this
  lookAt(eye: Vector3, target: Vector3, up: Vector3): this
  makePerspective(
    left: number, right: number, top: number, bottom: number,
    near: number, far: number
  ): this
  makeOrthographic(
    left: number, right: number, top: number, bottom: number,
    near: number, far: number
  ): this

  // Operations
  determinant(): number
  invert(): this
  transpose(): this
  extractBasis(xAxis: Vector3, yAxis: Vector3, zAxis: Vector3): this
  extractRotation(m: Matrix4): this
  setPosition(x: number, y: number, z: number): this
  setPosition(v: Vector3): this
  getMaxScaleOnAxis(): number

  // Comparison / Conversion
  equals(m: Matrix4): boolean
  toArray(array?: number[], offset?: number): number[]
  fromArray(array: number[], offset?: number): this
}
```

**Column-major layout:**
```
elements[0]  elements[4]  elements[8]   elements[12]   // m11 m12 m13 tx
elements[1]  elements[5]  elements[9]   elements[13]   // m21 m22 m23 ty
elements[2]  elements[6]  elements[10]  elements[14]   // m31 m32 m33 tz
elements[3]  elements[7]  elements[11]  elements[15]   // 0   0   0   1
```

Translation is stored in `elements[12]`, `elements[13]`, `elements[14]`.

---

## Quaternion

```typescript
class Quaternion {
  constructor(x?: number, y?: number, z?: number, w?: number)
  // Default: (0, 0, 0, 1) = identity rotation

  // Properties
  x: number
  y: number
  z: number
  w: number
  readonly isQuaternion: true

  // Assignment
  set(x: number, y: number, z: number, w: number): this
  identity(): this
  copy(q: Quaternion): this
  clone(): Quaternion

  // Setters (from other representations)
  setFromAxisAngle(axis: Vector3, angle: number): this
  setFromEuler(euler: Euler): this
  setFromRotationMatrix(m: Matrix4): this
  setFromUnitVectors(vFrom: Vector3, vTo: Vector3): this

  // Operations
  multiply(q: Quaternion): this           // this = this * q (q applied first)
  premultiply(q: Quaternion): this        // this = q * this
  slerp(qb: Quaternion, t: number): this  // spherical linear interpolation
  slerpQuaternions(qa: Quaternion, qb: Quaternion, t: number): this
  rotateTowards(q: Quaternion, step: number): this
  conjugate(): this                       // negate x, y, z
  invert(): this                          // conjugate / lengthSq
  normalize(): this
  angleTo(q: Quaternion): number

  // Comparison
  dot(q: Quaternion): number
  length(): number
  lengthSq(): number
  equals(q: Quaternion): boolean

  // Event
  _onChange(callback: () => void): this   // internal change callback
}
```

---

## Euler

```typescript
class Euler {
  constructor(x?: number, y?: number, z?: number, order?: string)
  // Default: (0, 0, 0, 'XYZ')

  // Properties
  x: number  // rotation around X axis in radians
  y: number  // rotation around Y axis in radians
  z: number  // rotation around Z axis in radians
  order: string  // 'XYZ' | 'YXZ' | 'ZXY' | 'ZYX' | 'YZX' | 'XZY'
  readonly isEuler: true

  static DEFAULT_ORDER: 'XYZ'

  // Assignment
  set(x: number, y: number, z: number, order?: string): this
  copy(euler: Euler): this
  clone(): Euler

  // Setters
  setFromRotationMatrix(m: Matrix4, order?: string): this
  setFromQuaternion(q: Quaternion, order?: string): this
  setFromVector3(v: Vector3, order?: string): this
  reorder(newOrder: string): this  // changes order, preserves rotation

  // Comparison / Conversion
  equals(euler: Euler): boolean
  toArray(array?: any[], offset?: number): any[]  // [x, y, z, order]
  fromArray(array: any[]): this

  _onChange(callback: () => void): this
}
```

**Gimbal lock reference:**

| Order | Lock axis | Lock angle |
|-------|-----------|------------|
| `'XYZ'` | Y | +/- 90 degrees (pi/2) |
| `'YXZ'` | X | +/- 90 degrees (pi/2) |
| `'ZXY'` | X | +/- 90 degrees (pi/2) |
| `'ZYX'` | Y | +/- 90 degrees (pi/2) |
| `'YZX'` | Z | +/- 90 degrees (pi/2) |
| `'XZY'` | Z | +/- 90 degrees (pi/2) |

---

## Color

```typescript
class Color {
  constructor()
  constructor(color: Color | string | number)
  constructor(r: number, g: number, b: number)

  // Properties
  r: number  // 0-1
  g: number  // 0-1
  b: number  // 0-1
  readonly isColor: true

  // Setters
  set(value: Color | string | number): this
  setScalar(scalar: number): this
  setHex(hex: number, colorSpace?: string): this
  setRGB(r: number, g: number, b: number, colorSpace?: string): this
  setHSL(h: number, s: number, l: number, colorSpace?: string): this
  setStyle(style: string, colorSpace?: string): this
  setColorName(name: string): this
  copy(color: Color): this
  clone(): Color

  // Getters
  getHex(colorSpace?: string): number
  getHexString(colorSpace?: string): string
  getHSL(target: { h: number, s: number, l: number }, colorSpace?: string): { h: number, s: number, l: number }
  getRGB(target: { r: number, g: number, b: number }, colorSpace?: string): { r: number, g: number, b: number }
  getStyle(colorSpace?: string): string

  // Color space conversion
  convertSRGBToLinear(): this
  convertLinearToSRGB(): this

  // Arithmetic
  add(color: Color): this
  addColors(color1: Color, color2: Color): this
  addScalar(s: number): this
  sub(color: Color): this
  multiply(color: Color): this
  multiplyScalar(s: number): this

  // Interpolation
  lerp(color: Color, alpha: number): this
  lerpColors(color1: Color, color2: Color, alpha: number): this
  lerpHSL(color: Color, alpha: number): this

  // Comparison / Conversion
  equals(c: Color): boolean
  toArray(array?: number[], offset?: number): number[]
  fromArray(array: number[], offset?: number): this
  toJSON(): number  // returns hex
}
```

---

## MathUtils

```typescript
namespace MathUtils {
  // Conversion
  function degToRad(degrees: number): number
  function radToDeg(radians: number): number

  // Clamping
  function clamp(value: number, min: number, max: number): number

  // Interpolation
  function lerp(x: number, y: number, t: number): number
  function inverseLerp(x: number, y: number, value: number): number
  function mapLinear(x: number, a1: number, a2: number, b1: number, b2: number): number
  function smoothstep(x: number, min: number, max: number): number
  function smootherstep(x: number, min: number, max: number): number
  function damp(x: number, y: number, lambda: number, dt: number): number
  function pingpong(x: number, length?: number): number

  // Random
  function randFloat(low: number, high: number): number
  function randFloatSpread(range: number): number
  function randInt(low: number, high: number): number
  function seededRandom(seed?: number): number

  // Power of two
  function isPowerOfTwo(value: number): boolean
  function ceilPowerOfTwo(value: number): number
  function floorPowerOfTwo(value: number): number

  // Misc
  function generateUUID(): string
  function euclideanModulo(n: number, m: number): number

  const DEG2RAD: number  // Math.PI / 180
  const RAD2DEG: number  // 180 / Math.PI
}
```

---

## Box3

```typescript
class Box3 {
  constructor(min?: Vector3, max?: Vector3)
  // Default: min = (+Infinity), max = (-Infinity) = empty box

  min: Vector3
  max: Vector3

  set(min: Vector3, max: Vector3): this
  setFromArray(array: number[]): this
  setFromBufferAttribute(attribute: BufferAttribute): this
  setFromPoints(points: Vector3[]): this
  setFromCenterAndSize(center: Vector3, size: Vector3): this
  setFromObject(object: Object3D, precise?: boolean): this
  clone(): Box3
  copy(box: Box3): this
  makeEmpty(): this
  isEmpty(): boolean

  getCenter(target: Vector3): Vector3
  getSize(target: Vector3): Vector3
  getBoundingSphere(target: Sphere): Sphere

  expandByPoint(point: Vector3): this
  expandByVector(vector: Vector3): this
  expandByScalar(scalar: number): this
  expandByObject(object: Object3D, precise?: boolean): this

  containsPoint(point: Vector3): boolean
  containsBox(box: Box3): boolean
  intersectsBox(box: Box3): boolean
  intersectsSphere(sphere: Sphere): boolean
  intersectsPlane(plane: Plane): boolean
  intersectsTriangle(triangle: Triangle): boolean

  clampPoint(point: Vector3, target: Vector3): Vector3
  distanceToPoint(point: Vector3): number
  union(box: Box3): this
  intersect(box: Box3): this
  equals(box: Box3): boolean

  applyMatrix4(matrix: Matrix4): this
  translate(offset: Vector3): this
}
```

---

## Sphere

```typescript
class Sphere {
  constructor(center?: Vector3, radius?: number)

  center: Vector3
  radius: number

  set(center: Vector3, radius: number): this
  setFromPoints(points: Vector3[], optionalCenter?: Vector3): this
  clone(): Sphere
  copy(sphere: Sphere): this
  isEmpty(): boolean
  makeEmpty(): this

  containsPoint(point: Vector3): boolean
  distanceToPoint(point: Vector3): number
  intersectsSphere(sphere: Sphere): boolean
  intersectsBox(box: Box3): boolean
  intersectsPlane(plane: Plane): boolean
  clampPoint(point: Vector3, target: Vector3): Vector3
  getBoundingBox(target: Box3): Box3

  applyMatrix4(matrix: Matrix4): this
  translate(offset: Vector3): this
  expandByPoint(point: Vector3): this
  union(sphere: Sphere): this
  equals(sphere: Sphere): boolean
}
```

---

## Plane

```typescript
class Plane {
  constructor(normal?: Vector3, constant?: number)

  normal: Vector3
  constant: number

  set(normal: Vector3, constant: number): this
  setFromNormalAndCoplanarPoint(normal: Vector3, point: Vector3): this
  setFromCoplanarPoints(a: Vector3, b: Vector3, c: Vector3): this
  setComponents(x: number, y: number, z: number, w: number): this
  normalize(): this
  negate(): this
  clone(): Plane
  copy(plane: Plane): this

  distanceToPoint(point: Vector3): number
  distanceToSphere(sphere: Sphere): number
  projectPoint(point: Vector3, target: Vector3): Vector3
  intersectLine(line: Line3, target: Vector3): Vector3 | null
  intersectsLine(line: Line3): boolean
  intersectsBox(box: Box3): boolean
  intersectsSphere(sphere: Sphere): boolean
  coplanarPoint(target: Vector3): Vector3
  applyMatrix4(matrix: Matrix4, optionalNormalMatrix?: Matrix3): this
  translate(offset: Vector3): this
  equals(plane: Plane): boolean
}
```

---

## Ray

```typescript
class Ray {
  constructor(origin?: Vector3, direction?: Vector3)

  origin: Vector3
  direction: Vector3

  set(origin: Vector3, direction: Vector3): this
  clone(): Ray
  copy(ray: Ray): this

  at(t: number, target: Vector3): Vector3
  lookAt(v: Vector3): this
  recast(t: number): this

  closestPointToPoint(point: Vector3, target: Vector3): Vector3
  distanceToPoint(point: Vector3): number
  distanceSqToPoint(point: Vector3): number
  distanceSqToSegment(
    v0: Vector3, v1: Vector3,
    optionalPointOnRay?: Vector3, optionalPointOnSegment?: Vector3
  ): number

  intersectBox(box: Box3, target: Vector3): Vector3 | null
  intersectsBox(box: Box3): boolean
  intersectSphere(sphere: Sphere, target: Vector3): Vector3 | null
  intersectsSphere(sphere: Sphere): boolean
  intersectPlane(plane: Plane, target: Vector3): Vector3 | null
  intersectsPlane(plane: Plane): boolean
  intersectTriangle(
    a: Vector3, b: Vector3, c: Vector3, backfaceCulling: boolean, target: Vector3
  ): Vector3 | null

  applyMatrix4(matrix4: Matrix4): this
  equals(ray: Ray): boolean
}
```

---

## Frustum

```typescript
class Frustum {
  constructor(
    p0?: Plane, p1?: Plane, p2?: Plane,
    p3?: Plane, p4?: Plane, p5?: Plane
  )

  planes: Plane[]  // 6 planes

  set(p0: Plane, p1: Plane, p2: Plane, p3: Plane, p4: Plane, p5: Plane): this
  setFromProjectionMatrix(m: Matrix4): this
  clone(): Frustum
  copy(frustum: Frustum): this

  containsPoint(point: Vector3): boolean
  intersectsObject(object: Object3D): boolean
  intersectsSprite(sprite: Sprite): boolean
  intersectsSphere(sphere: Sphere): boolean
  intersectsBox(box: Box3): boolean
}
```
