# threejs-errors-performance — Methods Reference

## Disposal Methods

### BufferGeometry.dispose()

```typescript
dispose(): void
```

Frees GPU vertex and index buffers. Fires the `'dispose'` event. After calling, the geometry CANNOT be used for rendering until re-uploaded.

**Source**: https://threejs.org/docs/#api/en/core/BufferGeometry.dispose

---

### Material.dispose()

```typescript
dispose(): void
```

Frees the compiled shader program and associated GPU resources. Fires the `'dispose'` event. Applies to ALL material types (`MeshStandardMaterial`, `MeshPhysicalMaterial`, `ShaderMaterial`, etc.).

**Source**: https://threejs.org/docs/#api/en/materials/Material.dispose

---

### Texture.dispose()

```typescript
dispose(): void
```

Frees GPU texture memory. Fires the `'dispose'` event. Applies to `Texture`, `CanvasTexture`, `VideoTexture`, `DataTexture`, `CubeTexture`, `CompressedTexture`, and all other texture types.

**Source**: https://threejs.org/docs/#api/en/textures/Texture.dispose

---

### WebGLRenderTarget.dispose()

```typescript
dispose(): void
```

Frees the framebuffer and associated texture(s). ALWAYS call before creating a replacement render target.

**Source**: https://threejs.org/docs/#api/en/renderers/WebGLRenderTarget.dispose

---

### WebGLRenderer.dispose()

```typescript
dispose(): void
```

Destroys the WebGL context and releases ALL GPU resources managed by the renderer. The canvas DOM element is NOT removed — remove it manually with `renderer.domElement.remove()`. NEVER call `renderer.render()` after `dispose()`.

**Source**: https://threejs.org/docs/#api/en/renderers/WebGLRenderer.dispose

---

### Controls.dispose()

```typescript
dispose(): void
```

Removes all DOM event listeners registered by the controls instance (`OrbitControls`, `MapControls`, `FlyControls`, etc.). ALWAYS call on cleanup to prevent event listener leaks.

---

## Profiling Methods and Properties

### renderer.info

```typescript
renderer.info: {
  render: {
    calls: number;       // draw calls this frame
    triangles: number;   // triangles rendered this frame
    points: number;      // points rendered this frame
    lines: number;       // lines rendered this frame
    frame: number;       // total frames rendered
  };
  memory: {
    geometries: number;  // geometries currently on GPU
    textures: number;    // textures currently on GPU
  };
  programs: WebGLProgram[] | null; // compiled shader programs
}
```

**Rule**: `renderer.info.render` resets each frame. `renderer.info.memory` is cumulative — growing values indicate a leak.

**Source**: https://threejs.org/docs/#api/en/renderers/WebGLRenderer.info

---

### renderer.info.reset()

```typescript
reset(): void
```

Resets the render statistics (`calls`, `triangles`, `points`, `lines`, `frame`). Does NOT affect memory counts.

---

## InstancedMesh Methods

### InstancedMesh Constructor

```typescript
new InstancedMesh(
  geometry: BufferGeometry,
  material: Material | Material[],
  count: number
)
```

- `count` — Maximum number of instances. CANNOT be changed after creation.
- Creates an internal `InstancedBufferAttribute` for instance matrices (16 floats per instance).

**Source**: https://threejs.org/docs/#api/en/objects/InstancedMesh

---

### setMatrixAt / getMatrixAt

```typescript
setMatrixAt(index: number, matrix: Matrix4): void
getMatrixAt(index: number, matrix: Matrix4): Matrix4
```

Sets or retrieves the 4x4 transform matrix for instance at `index`. ALWAYS set `mesh.instanceMatrix.needsUpdate = true` after calling `setMatrixAt`.

---

### setColorAt / getColorAt

```typescript
setColorAt(index: number, color: Color): void
getColorAt(index: number, color: Color): Color
```

Sets or retrieves per-instance color. The `instanceColor` attribute is created on the first `setColorAt` call. ALWAYS set `mesh.instanceColor.needsUpdate = true` after calling `setColorAt`.

---

## LOD Methods

### LOD Constructor

```typescript
new LOD()
```

**Source**: https://threejs.org/docs/#api/en/objects/LOD

---

### addLevel

```typescript
addLevel(object: Object3D, distance?: number, hysteresis?: number): this
```

- `object` — The mesh to display at this level
- `distance` — Minimum distance from camera for this level to activate (default: `0`)
- `hysteresis` — Threshold to prevent rapid switching between levels (default: `0`)

---

### update

```typescript
update(camera: Camera): void
```

ALWAYS call `lod.update(camera)` in the animation loop. This selects the appropriate level based on camera distance.

---

## BufferGeometryUtils

### mergeGeometries

```typescript
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';

mergeGeometries(
  geometries: BufferGeometry[],
  useGroups?: boolean
): BufferGeometry | null
```

- `geometries` — Array of geometries to merge (MUST have compatible attributes)
- `useGroups` — If `true`, creates material groups for multi-material support
- Returns `null` if geometries are incompatible

**Source**: https://threejs.org/docs/#examples/en/utils/BufferGeometryUtils.mergeGeometries

---

## Object3D Matrix Properties

### matrixAutoUpdate

```typescript
matrixAutoUpdate: boolean // default: true
```

When `true`, the renderer recomputes the local matrix from `position`, `rotation`, `scale` every frame. Set to `false` for static objects to skip this computation. After setting to `false`, ALWAYS call `object.updateMatrix()` once to compute the final matrix.

**Source**: https://threejs.org/docs/#api/en/core/Object3D.matrixAutoUpdate
