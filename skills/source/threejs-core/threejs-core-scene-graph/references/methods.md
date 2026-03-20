# API Signatures Reference (Three.js Scene Graph r160+)

## Object3D

The base class for all 3D objects. Every Mesh, Group, Light, Camera, and Scene inherits from Object3D.

### Constructor

```typescript
new Object3D()
```

No parameters. Creates an object at origin with identity transform.

### Transform Properties

```typescript
position: Vector3           // default: (0, 0, 0) -- local position
rotation: Euler             // default: (0, 0, 0, 'XYZ') -- linked to quaternion
quaternion: Quaternion      // default: (0, 0, 0, 1) -- linked to rotation
scale: Vector3              // default: (1, 1, 1) -- local scale
up: Vector3                 // default: (0, 1, 0) -- up direction for lookAt()
```

### Hierarchy Properties

```typescript
parent: Object3D | null     // set automatically by add()/remove()
children: Object3D[]        // NEVER modify directly
```

### Rendering Properties

```typescript
visible: boolean            // default: true
castShadow: boolean         // default: false
receiveShadow: boolean      // default: false
frustumCulled: boolean      // default: true
renderOrder: number         // default: 0
layers: Layers              // default: layer 0 enabled
```

### Identity Properties

```typescript
uuid: string                // read-only, auto-generated RFC 4122 v4
id: number                  // read-only, auto-incrementing integer
name: string                // default: ""
type: string                // read-only, e.g., "Object3D", "Mesh"
userData: object             // default: {}
```

### Matrix Properties

```typescript
matrix: Matrix4                 // local transform matrix
matrixWorld: Matrix4             // world transform matrix
matrixAutoUpdate: boolean        // default: true
matrixWorldAutoUpdate: boolean   // default: true
matrixWorldNeedsUpdate: boolean  // default: false
modelViewMatrix: Matrix4         // computed per frame by renderer
normalMatrix: Matrix3            // computed per frame by renderer
```

---

### Hierarchy Methods

```typescript
add(object: Object3D, ...args: Object3D[]): this
```
Adds one or more children. Auto-removes from previous parent. Fires `added` event on child. Returns `this` for chaining.

```typescript
remove(object: Object3D, ...args: Object3D[]): this
```
Removes one or more children. Fires `removed` event. NEVER throws if object is not a child. Returns `this`.

```typescript
removeFromParent(): this
```
Removes this object from its parent. Safe when `parent` is `null`. Returns `this`.

```typescript
clear(): this
```
Removes ALL children. ALWAYS prefer over manual iteration to avoid index-shifting bugs. Returns `this`.

```typescript
attach(object: Object3D): this
```
Adds object as child while preserving its world transform. Internally recomputes local transform from world transform. Returns `this`.

---

### Traversal Methods

```typescript
traverse(callback: (object: Object3D) => void): void
```
Depth-first traversal of this object and ALL descendants. Performance: O(n).

```typescript
traverseVisible(callback: (object: Object3D) => void): void
```
Same as `traverse()` but skips objects where `visible === false` and all their descendants.

```typescript
traverseAncestors(callback: (object: Object3D) => void): void
```
Walks UP the tree from parent to root. Does NOT include the starting object.

```typescript
getObjectByName(name: string): Object3D | undefined
```
Recursive search. Returns first match or `undefined`.

```typescript
getObjectById(id: number): Object3D | undefined
```
Recursive search by auto-generated id. Returns first match or `undefined`.

```typescript
getObjectByProperty(name: string, value: any): Object3D | undefined
```
Recursive search by arbitrary property. Returns first match or `undefined`.

---

### Transform Methods

```typescript
lookAt(x: number, y: number, z: number): void
lookAt(vector: Vector3): void
```
Rotates to face a world-space point. Cameras point negative-Z; other objects point positive-Z toward target.

```typescript
rotateOnAxis(axis: Vector3, angle: number): this
```
Rotates around a local-space axis by angle (radians).

```typescript
rotateOnWorldAxis(axis: Vector3, angle: number): this
```
Rotates around a world-space axis by angle (radians).

```typescript
rotateX(angle: number): this
rotateY(angle: number): this
rotateZ(angle: number): this
```
Shorthand for `rotateOnAxis()` on the respective axis.

```typescript
translateOnAxis(axis: Vector3, distance: number): this
```
Translates along a local-space axis.

```typescript
translateX(distance: number): this
translateY(distance: number): this
translateZ(distance: number): this
```
Shorthand for `translateOnAxis()` on the respective axis.

---

### Coordinate Conversion Methods

```typescript
localToWorld(vector: Vector3): Vector3
```
Converts vector from local space to world space. **MUTATES** the input vector. Returns the same vector.

```typescript
worldToLocal(vector: Vector3): Vector3
```
Converts vector from world space to local space. **MUTATES** the input vector. Returns the same vector.

---

### Matrix Update Methods

```typescript
updateMatrix(): void
```
Recomputes `matrix` from `position`, `rotation`, and `scale`. Called automatically when `matrixAutoUpdate` is `true`.

```typescript
updateMatrixWorld(force?: boolean): void
```
Recursively updates `matrixWorld` for this object and all descendants. When `force` is `true`, recalculates regardless of flags.

```typescript
updateWorldMatrix(updateParents: boolean, updateChildren: boolean): void
```
Granular matrix update. `updateParents`: walks up to root first. `updateChildren`: recursively updates descendants after.

---

### World-Space Query Methods

```typescript
getWorldPosition(target: Vector3): Vector3
```
Writes world position to `target`. Calls `updateWorldMatrix(true, false)` internally.

```typescript
getWorldQuaternion(target: Quaternion): Quaternion
```
Writes world rotation to `target`. Calls `updateWorldMatrix(true, false)` internally.

```typescript
getWorldScale(target: Vector3): Vector3
```
Writes world scale to `target`. Calls `updateWorldMatrix(true, false)` internally.

```typescript
getWorldDirection(target: Vector3): Vector3
```
Writes world-space forward direction to `target`. For cameras: negative-Z. For others: positive-Z.

---

### Utility Methods

```typescript
clone(recursive?: boolean): Object3D
```
Returns a new Object3D with copied properties. When `recursive` is `true` (default), clones all descendants. `userData` is shallow-copied.

```typescript
copy(source: Object3D, recursive?: boolean): this
```
Copies properties from source. Returns `this`.

```typescript
toJSON(meta?: object): object
```
Serializes to JSON format.

```typescript
applyMatrix4(matrix: Matrix4): void
```
Applies a matrix transform to the object. Updates position, quaternion, and scale from the decomposed matrix.

```typescript
applyQuaternion(quaternion: Quaternion): this
```
Applies a quaternion rotation. Returns `this`.

---

## Scene

Extends `Object3D`. Root container for all renderable objects.

### Constructor

```typescript
new Scene()
```

### Properties

```typescript
background: Color | Texture | CubeTexture | null  // default: null
environment: Texture | null                         // default: null -- IBL for all PBR materials
fog: Fog | FogExp2 | null                           // default: null
overrideMaterial: Material | null                    // default: null
backgroundBlurriness: number                        // default: 0 (range 0-1)
backgroundIntensity: number                         // default: 1
backgroundRotation: Euler                           // default: (0, 0, 0)
environmentIntensity: number                        // default: 1
environmentRotation: Euler                          // default: (0, 0, 0)
```

### Type Flags

```typescript
isScene: true   // read-only type check flag
```

---

## Fog

### Constructor

```typescript
new Fog(color: Color | string | number, near?: number, far?: number)
```
- `near`: distance where fog starts (default: `1`)
- `far`: distance where fog is fully opaque (default: `1000`)
- Linear interpolation between near and far

### Properties

```typescript
isFog: true             // read-only type check flag
color: Color            // fog color
near: number            // start distance
far: number             // end distance
```

---

## FogExp2

### Constructor

```typescript
new FogExp2(color: Color | string | number, density?: number)
```
- `density`: exponential density coefficient (default: `0.00025`)

### Properties

```typescript
isFogExp2: true         // read-only type check flag
color: Color            // fog color
density: number         // exponential density
```

---

## Group

Extends `Object3D`. Semantic container with no additional properties or methods.

### Constructor

```typescript
new Group()
```

### Type Flags

```typescript
isGroup: true   // read-only type check flag
```

---

## Mesh

Extends `Object3D`. Combines geometry and material into a renderable surface.

### Constructor

```typescript
new Mesh(geometry?: BufferGeometry, material?: Material | Material[])
```
- `geometry`: defaults to empty `BufferGeometry`
- `material`: defaults to `MeshBasicMaterial` with random color. Pass an array for multi-material rendering with geometry groups.

### Properties

```typescript
isMesh: true                                    // read-only type check flag
geometry: BufferGeometry                        // the geometry
material: Material | Material[]                 // single or multi-material
morphTargetInfluences: number[] | undefined     // blend weights for morph targets (0-1)
morphTargetDictionary: object | undefined       // name-to-index mapping for morph targets
```

### Methods

```typescript
getVertexPosition(index: number, target: Vector3): Vector3
```
Returns the local-space position of the vertex at the given index. Accounts for morph targets and skinning.

```typescript
updateMorphTargets(): void
```
Rebuilds `morphTargetInfluences` and `morphTargetDictionary` from the geometry.

```typescript
raycast(raycaster: Raycaster, intersects: Intersection[]): void
```
Tests for ray intersection. Called internally by `Raycaster.intersectObject()`.

---

## Layers

32-bit bitmask system for selective rendering and raycasting.

### Constructor

```typescript
new Layers()
```
Creates a Layers object with layer 0 enabled.

### Methods

```typescript
set(channel: number): void
```
Enables ONLY this channel (0-31). Disables all others.

```typescript
enable(channel: number): void
```
Enables a channel without affecting others.

```typescript
enableAll(): void
```
Enables all 32 channels.

```typescript
toggle(channel: number): void
```
Flips a channel on/off.

```typescript
disable(channel: number): void
```
Disables a channel without affecting others.

```typescript
disableAll(): void
```
Disables all 32 channels.

```typescript
isEnabled(channel: number): boolean
```
Returns `true` if the specified channel is enabled.

```typescript
test(layers: Layers): boolean
```
Returns `true` if ANY channel overlaps between this and the given Layers (bitwise AND).

### Properties

```typescript
mask: number    // the raw 32-bit bitmask. Default: 1 (layer 0)
```
