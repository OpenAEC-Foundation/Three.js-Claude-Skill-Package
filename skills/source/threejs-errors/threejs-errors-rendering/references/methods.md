# threejs-errors-rendering — Methods Reference

## Renderer Diagnostics

```typescript
// Check renderer size
renderer.getSize(target: Vector2): Vector2

// Check render info (draw calls, triangles, memory)
renderer.info.render.calls: number
renderer.info.render.triangles: number
renderer.info.memory.geometries: number
renderer.info.memory.textures: number
renderer.info.autoReset: boolean  // set false to accumulate across frames

// Reset info counters
renderer.info.reset(): void
```

## Size and Pixel Ratio

```typescript
renderer.setSize(width: number, height: number, updateStyle?: boolean): void
// updateStyle default: true. Set false when canvas size is managed externally.

renderer.setPixelRatio(value: number): void
// ALWAYS cap: renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))

renderer.getPixelRatio(): number
```

## Color Space and Tone Mapping

```typescript
renderer.outputColorSpace: string
// Default: THREE.SRGBColorSpace (r160+)
// Set to THREE.LinearSRGBColorSpace for post-processing pipelines

renderer.toneMapping: number
// THREE.NoToneMapping | THREE.LinearToneMapping | THREE.ReinhardToneMapping
// THREE.CineonToneMapping | THREE.ACESFilmicToneMapping
// THREE.AgXToneMapping | THREE.NeutralToneMapping

renderer.toneMappingExposure: number  // Default: 1.0
```

## Texture Color Space

```typescript
texture.colorSpace: string
// THREE.SRGBColorSpace — for diffuse, emissive, color textures
// THREE.LinearSRGBColorSpace — for normal, roughness, metalness, AO, displacement
// THREE.NoColorSpace — default (no conversion)

texture.needsUpdate: boolean  // set true after changing image or properties
```

## Camera Projection

```typescript
// PerspectiveCamera
camera.fov: number
camera.aspect: number
camera.near: number
camera.far: number
camera.zoom: number
camera.updateProjectionMatrix(): void
// ALWAYS call after changing fov, aspect, near, far, or zoom

// OrthographicCamera
camera.left: number
camera.right: number
camera.top: number
camera.bottom: number
camera.updateProjectionMatrix(): void
```

## Material Visibility Properties

```typescript
material.side: number
// THREE.FrontSide (default) | THREE.BackSide | THREE.DoubleSide

material.visible: boolean        // Default: true
material.transparent: boolean    // Default: false. MUST be true for opacity < 1
material.opacity: number         // Default: 1.0. Requires transparent: true
material.alphaTest: number       // Default: 0. Fragments below threshold discarded
material.depthWrite: boolean     // Default: true. Set false for transparent objects
material.depthTest: boolean      // Default: true
material.needsUpdate: boolean    // Set true to recompile shader

material.polygonOffset: boolean          // Default: false
material.polygonOffsetFactor: number     // Default: 0
material.polygonOffsetUnits: number      // Default: 0
```

## Object Visibility Properties

```typescript
object.visible: boolean          // Default: true. Inherited by descendants
object.frustumCulled: boolean    // Default: true
object.renderOrder: number       // Default: 0. Higher renders later
object.layers: Layers            // 32-bit layer mask

layers.set(layer: number): void       // Enable ONLY this layer
layers.enable(layer: number): void    // Add a layer
layers.disable(layer: number): void   // Remove a layer
layers.test(layers: Layers): boolean  // Check overlap
```

## BufferAttribute Update

```typescript
attribute.needsUpdate: boolean   // Set true after modifying array data
attribute.usage: number          // THREE.StaticDrawUsage | THREE.DynamicDrawUsage

// InstancedMesh specific
mesh.instanceMatrix: InstancedBufferAttribute
mesh.instanceColor: InstancedBufferAttribute | null
mesh.setMatrixAt(index: number, matrix: Matrix4): void
mesh.setColorAt(index: number, color: Color): void
// ALWAYS set instanceMatrix.needsUpdate = true after setMatrixAt
```

## Geometry Bounds

```typescript
geometry.computeBoundingBox(): void
geometry.computeBoundingSphere(): void
// ALWAYS call after modifying position attribute data
// Required for correct frustum culling
```

## WebGL Context Events

```typescript
canvas.addEventListener('webglcontextlost', (event: WebGLContextEvent) => void)
canvas.addEventListener('webglcontextrestored', () => void)
// event.preventDefault() in contextlost handler allows restoration
```

## WebGLRenderer Constructor Options

```typescript
new THREE.WebGLRenderer({
  canvas?: HTMLCanvasElement,
  antialias?: boolean,           // Default: false
  alpha?: boolean,               // Default: false (transparent background)
  logarithmicDepthBuffer?: boolean, // Default: false (helps z-fighting)
  powerPreference?: string,      // 'high-performance' | 'low-power' | 'default'
  preserveDrawingBuffer?: boolean, // Default: false (needed for toDataURL)
  stencil?: boolean,             // Default: true
  depth?: boolean                // Default: true
})
```
