# threejs-impl-drei — Anti-Patterns

## AP-01: Missing Suspense Around Loader Hooks

**NEVER** use `useGLTF`, `useTexture`, `useFBX`, `useKTX2`, or `useFont` without wrapping the consuming component in `<Suspense>`.

```jsx
// WRONG — crashes the entire React tree
function App() {
  return (
    <Canvas>
      <Model />
    </Canvas>
  )
}

function Model() {
  const { nodes } = useGLTF('/model.glb') // Suspense throw with no boundary
  return <mesh geometry={nodes.Body.geometry} />
}
```

```jsx
// CORRECT — Suspense catches the loading promise
function App() {
  return (
    <Canvas>
      <Suspense fallback={null}>
        <Model />
      </Suspense>
    </Canvas>
  )
}
```

**Why:** Drei loader hooks use React Suspense internally. Without a `<Suspense>` boundary, React has no way to handle the thrown promise and the entire component tree unmounts with an error.

---

## AP-02: Forgetting makeDefault on Controls

**NEVER** omit `makeDefault` on your primary camera controls.

```jsx
// WRONG — controls work but pointer events on meshes break
<OrbitControls />
```

```jsx
// CORRECT — integrates with R3F's event system
<OrbitControls makeDefault />
```

**Why:** Without `makeDefault`, R3F does not know about the controls. The event system uses the default camera for raycasting, but the controls may update a different camera reference. This causes pointer events (`onClick`, `onPointerOver`) to use stale camera data, resulting in incorrect hit detection.

---

## AP-03: Invalid Environment Preset Name

**NEVER** use a preset name that is not in the valid list.

```jsx
// WRONG — silently fails or throws, no environment loaded
<Environment preset="outdoor" />
<Environment preset="hdri" />
<Environment preset="default" />
```

```jsx
// CORRECT — use ONLY these exact preset names
<Environment preset="city" />
// Valid: apartment, city, dawn, forest, lobby, night, park, studio, sunset, warehouse
```

**Why:** Drei downloads preset HDR files from a CDN. Invalid preset names result in a 404 network error. The component fails silently or throws, leaving the scene without environment lighting. There is no fallback mechanism.

---

## AP-04: Not Preloading Critical Assets

**NEVER** rely solely on component-mount loading for assets visible on first render.

```jsx
// WRONG — model loads only when component mounts, causing visible pop-in
function Model() {
  const { nodes } = useGLTF('/hero-model.glb')
  return <mesh geometry={nodes.Body.geometry} />
}
```

```jsx
// CORRECT — preload at module scope, asset is ready before mount
function Model() {
  const { nodes } = useGLTF('/hero-model.glb')
  return <mesh geometry={nodes.Body.geometry} />
}
useGLTF.preload('/hero-model.glb')  // starts loading immediately at import time
```

**Why:** Without `.preload()`, the asset download begins only when the component first renders. For hero models or textures that are visible immediately, this causes a jarring flash of empty content followed by sudden appearance. Preloading moves the network request to module evaluation time, overlapping with other initialization work.

---

## AP-05: Using Individual Meshes Instead of Instances

**NEVER** render hundreds of identical meshes as separate `<mesh>` elements.

```jsx
// WRONG — 1000 draw calls, one per mesh
{positions.map((pos, i) => (
  <mesh key={i} position={pos}>
    <boxGeometry args={[0.5, 0.5, 0.5]} />
    <meshStandardMaterial color="orange" />
  </mesh>
))}
```

```jsx
// CORRECT — 1 draw call for all 1000 instances
<Instances limit={1000} range={1000}>
  <boxGeometry args={[0.5, 0.5, 0.5]} />
  <meshStandardMaterial color="orange" />
  {positions.map((pos, i) => (
    <Instance key={i} position={pos} color="orange" />
  ))}
</Instances>
```

**Why:** Each separate `<mesh>` generates its own draw call. At 100+ meshes, draw call overhead dominates frame time and FPS drops dramatically. `<Instances>` uses GPU instancing to render all copies in a single draw call, reducing CPU overhead by orders of magnitude.

---

## AP-06: Recreating Geometry and Material on Every Render

**NEVER** create Three.js objects inline without memoization.

```jsx
// WRONG — new geometry and material every render cycle
function MyMesh() {
  return (
    <mesh>
      <bufferGeometry>
        <bufferAttribute
          attach="attributes-position"
          array={new Float32Array(computeVertices())}  // new array every render
          count={vertexCount}
          itemSize={3}
        />
      </bufferGeometry>
      <meshStandardMaterial color="red" />
    </mesh>
  )
}
```

```jsx
// CORRECT — memoize computed data
function MyMesh() {
  const vertices = useMemo(() => new Float32Array(computeVertices()), [])
  return (
    <mesh>
      <bufferGeometry>
        <bufferAttribute
          attach="attributes-position"
          array={vertices}
          count={vertexCount}
          itemSize={3}
        />
      </bufferGeometry>
      <meshStandardMaterial color="red" />
    </mesh>
  )
}
```

**Why:** R3F disposes and recreates Three.js objects when their constructor arguments (`args`) change. Creating new arrays or objects inline causes R3F to detect a change every frame, triggering GPU resource deallocation and reallocation. This causes massive GC pressure and frame drops.

---

## AP-07: Using Html Without occlude or distanceFactor

**NEVER** use `<Html>` in a 3D scene without considering occlusion and scaling.

```jsx
// WRONG — HTML label floats above everything, same size regardless of distance
<Html>
  <div>Label</div>
</Html>
```

```jsx
// CORRECT — label scales with distance and hides behind objects
<Html
  transform
  distanceFactor={10}
  occlude
  center
>
  <div>Label</div>
</Html>
```

**Why:** Without `occlude`, HTML elements render on top of ALL 3D content, breaking spatial perception. Without `distanceFactor`, labels remain the same pixel size regardless of camera distance, making them look disconnected from the 3D scene. Without `transform`, the HTML element does not participate in 3D positioning.

---

## AP-08: Using ContactShadows with frames={Infinity} Unnecessarily

**NEVER** use real-time ContactShadows when objects are static.

```jsx
// WRONG — re-renders shadow every frame for a static scene
<ContactShadows
  opacity={0.5}
  scale={10}
  blur={2}
  resolution={1024}
/>
```

```jsx
// CORRECT — bake shadow once for static scenes
<ContactShadows
  opacity={0.5}
  scale={10}
  blur={2}
  resolution={1024}
  frames={1}           // render once and stop
/>
```

**Why:** `ContactShadows` defaults to rendering every frame (`frames={Infinity}`). For static scenes where objects do not move, this wastes GPU resources re-rendering an identical shadow map 60 times per second. Setting `frames={1}` bakes the shadow on mount and stops updating, saving significant GPU time.

---

## AP-09: TransformControls Conflicting with OrbitControls

**NEVER** use TransformControls and OrbitControls without preventing event conflict.

```jsx
// WRONG — dragging the gizmo also orbits the camera
<OrbitControls makeDefault />
<TransformControls object={meshRef.current} />
```

```jsx
// CORRECT — disable orbit controls while dragging the gizmo
function Scene() {
  const orbitRef = useRef()
  return (
    <>
      <OrbitControls ref={orbitRef} makeDefault />
      <TransformControls
        object={meshRef.current}
        onMouseDown={() => (orbitRef.current.enabled = false)}
        onMouseUp={() => (orbitRef.current.enabled = true)}
      />
    </>
  )
}
```

**Why:** Both controls listen to the same pointer events. When dragging a TransformControls gizmo handle, the OrbitControls also receives the drag event and rotates the camera simultaneously. Disabling OrbitControls during TransformControls interaction prevents this conflict.

---

## AP-10: Mounting the Same useGLTF Scene Object Multiple Times

**NEVER** use `<primitive object={gltf.scene} />` in multiple components simultaneously.

```jsx
// WRONG — same object reference mounted twice, second instance steals it from the first
function App() {
  return (
    <>
      <Model position={[0, 0, 0]} />
      <Model position={[5, 0, 0]} />
    </>
  )
}

function Model({ position }) {
  const gltf = useGLTF('/model.glb')
  return <primitive object={gltf.scene} position={position} />
}
```

```jsx
// CORRECT — clone the scene for each instance, or use individual nodes
function Model({ position }) {
  const { nodes, materials } = useGLTF('/model.glb')
  return (
    <group position={position}>
      <mesh geometry={nodes.Body.geometry} material={materials.Metal} />
    </group>
  )
}

// ALTERNATIVE — use Clone from Drei
import { Clone } from '@react-three/drei'

function Model({ position }) {
  const { scene } = useGLTF('/model.glb')
  return <Clone object={scene} position={position} />
}
```

**Why:** A Three.js Object3D can only have one parent. When you mount the same `scene` object in two locations, the second `<primitive>` reparents it, removing it from the first. Use `<Clone>` to create a deep copy with shared geometry and materials, or access individual `nodes` and create separate mesh elements.
