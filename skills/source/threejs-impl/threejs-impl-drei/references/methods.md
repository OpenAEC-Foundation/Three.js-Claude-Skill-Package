# threejs-impl-drei — Methods Reference

## Controls

### OrbitControls

```tsx
<OrbitControls
  makeDefault?: boolean          // ALWAYS set true for primary controls
  enableDamping?: boolean        // default: true
  dampingFactor?: number         // default: 0.05
  enableZoom?: boolean           // default: true
  enableRotate?: boolean         // default: true
  enablePan?: boolean            // default: true
  minDistance?: number           // minimum zoom distance
  maxDistance?: number           // maximum zoom distance
  minPolarAngle?: number         // minimum vertical angle (radians)
  maxPolarAngle?: number         // maximum vertical angle (radians)
  minAzimuthAngle?: number       // minimum horizontal angle
  maxAzimuthAngle?: number       // maximum horizontal angle
  target?: [x, y, z]            // orbit target point
  onChange?: (e: Event) => void  // fires on camera change
  onStart?: (e: Event) => void
  onEnd?: (e: Event) => void
/>
```

### CameraControls

```tsx
<CameraControls
  makeDefault?: boolean
  minDistance?: number
  maxDistance?: number
  minPolarAngle?: number
  maxPolarAngle?: number
  smoothTime?: number            // transition duration in seconds
  draggingSmoothTime?: number
  azimuthRotateSpeed?: number
  polarRotateSpeed?: number
  dollySpeed?: number
  truckSpeed?: number
/>
```

Imperative methods via ref:
- `ref.current.setLookAt(posX, posY, posZ, targetX, targetY, targetZ, enableTransition)`
- `ref.current.dolly(distance, enableTransition)`
- `ref.current.truck(x, y, enableTransition)`
- `ref.current.rotate(azimuth, polar, enableTransition)`
- `ref.current.fitToBox(box3OrObject, enableTransition, options)`
- `ref.current.setPosition(x, y, z, enableTransition)`
- `ref.current.setTarget(x, y, z, enableTransition)`

### ScrollControls / useScroll

```tsx
<ScrollControls
  pages?: number          // number of scroll pages (default: 1)
  distance?: number       // scroll factor (default: 1)
  damping?: number        // scroll smoothing (default: 0.25)
  horizontal?: boolean    // horizontal scroll (default: false)
  enabled?: boolean       // enable/disable (default: true)
  infinite?: boolean      // infinite scroll (default: false)
  eps?: number            // scroll threshold (default: 0.00001)
>
  {children}
</ScrollControls>
```

```tsx
const scroll = useScroll()
scroll.offset    // normalized scroll position 0..1
scroll.delta     // scroll speed
scroll.visible(from, range)  // visibility within range
scroll.range(from, range)    // clamped progress within range
scroll.curve(from, range)    // bell curve within range
```

### PresentationControls

```tsx
<PresentationControls
  global?: boolean            // respond to events globally
  cursor?: boolean            // show grab cursor
  snap?: boolean | object     // snap back to initial rotation
  speed?: number              // rotation speed (default: 1)
  zoom?: number               // zoom speed (default: 1)
  rotation?: [x, y, z]        // initial rotation
  polar?: [min, max]          // vertical rotation limits
  azimuth?: [min, max]        // horizontal rotation limits
  config?: SpringConfig       // react-spring config
/>
```

### TransformControls

```tsx
<TransformControls
  object?: THREE.Object3D     // target object (or wrap children)
  mode?: 'translate' | 'rotate' | 'scale'
  space?: 'world' | 'local'
  showX?: boolean
  showY?: boolean
  showZ?: boolean
  size?: number               // gizmo size
  onObjectChange?: () => void
/>
```

### KeyboardControls / useKeyboardControls

```tsx
<KeyboardControls
  map={[
    { name: 'forward', keys: ['ArrowUp', 'KeyW'] },
    { name: 'backward', keys: ['ArrowDown', 'KeyS'] },
    { name: 'jump', keys: ['Space'] },
  ]}
>
  {children}
</KeyboardControls>
```

```tsx
const [subscribeKeys, getKeys] = useKeyboardControls()
const pressed = getKeys()       // { forward: true, backward: false, jump: false }
subscribeKeys((state) => { })   // subscribe to all changes
subscribeKeys(
  (state) => state.jump,        // selector
  (pressed) => { }              // callback when jump changes
)
```

---

## Environment

### Environment

```tsx
<Environment
  preset?: 'apartment' | 'city' | 'dawn' | 'forest' | 'lobby' |
           'night' | 'park' | 'studio' | 'sunset' | 'warehouse'
  files?: string | string[]       // HDR/EXR file path(s)
  path?: string                   // base path for files
  background?: boolean | 'only'   // show as scene background
  blur?: number                   // background blur (0..1)
  resolution?: number             // cube map resolution (default: 256)
  near?: number                   // near plane for ground projection
  far?: number                    // far plane for ground projection
  ground?: { height, radius }     // ground-projected environment
  map?: THREE.Texture             // pre-loaded texture
  frames?: number                 // render frames (Infinity = realtime)
  encoding?: THREE.TextureEncoding
/>
```

### Lightformer

```tsx
<Lightformer
  form?: 'circle' | 'ring' | 'rect'   // shape (default: 'rect')
  intensity?: number                    // light intensity
  color?: string | THREE.Color          // light color
  position?: [x, y, z]
  rotation?: [x, y, z]
  scale?: number | [w, h, d]
  target?: [x, y, z]                   // look-at target
/>
```

### useEnvironment

```tsx
const envMap = useEnvironment({
  preset?: string,
  files?: string | string[],
  path?: string,
  resolution?: number,
  extensions?: (loader: Loader) => void,
})
```

---

## Shadows

### ContactShadows

```tsx
<ContactShadows
  opacity?: number           // shadow darkness (default: 1)
  scale?: number | [w, h]   // shadow plane size (default: 10)
  blur?: number              // blur passes (default: 1)
  far?: number               // shadow distance (default: 10)
  resolution?: number        // texture resolution (default: 512)
  color?: string             // shadow color (default: '#000000')
  frames?: number            // render frames (1 = bake, Infinity = realtime)
  position?: [x, y, z]
/>
```

### AccumulativeShadows

```tsx
<AccumulativeShadows
  temporal?: boolean         // spread across frames
  frames?: number            // accumulation frames (default: 40)
  alphaTest?: number         // alpha cutoff (default: 0.75)
  scale?: number             // shadow plane size
  position?: [x, y, z]
  opacity?: number           // overall opacity
  color?: string             // shadow color
  toneMapped?: boolean
/>
```

### RandomizedLight

```tsx
<RandomizedLight
  amount?: number            // number of lights (default: 8)
  radius?: number            // jitter radius (default: 1)
  intensity?: number         // light intensity (default: Math.PI)
  ambient?: number           // ambient light fraction (default: 0.5)
  position?: [x, y, z]
  bias?: number              // shadow bias
  mapSize?: number           // shadow map resolution
  size?: number              // light area size
  near?: number
  far?: number
  castShadow?: boolean
/>
```

---

## Text and HTML

### Text

```tsx
<Text
  font?: string               // font URL (.woff, .woff2, .ttf, .otf)
  fontSize?: number            // default: 1
  color?: string               // text color
  maxWidth?: number            // word wrap width
  lineHeight?: number          // line height multiplier
  letterSpacing?: number       // letter spacing
  textAlign?: 'left' | 'right' | 'center' | 'justify'
  anchorX?: 'left' | 'center' | 'right' | number
  anchorY?: 'top' | 'top-baseline' | 'middle' | 'bottom-baseline' | 'bottom' | number
  outlineWidth?: number | string
  outlineColor?: string
  outlineOpacity?: number
  strokeWidth?: number
  strokeColor?: string
  fillOpacity?: number
  depthOffset?: number
  overflowWrap?: 'normal' | 'break-word'
  whiteSpace?: 'normal' | 'nowrap' | 'overflowWrap'
  characters?: string          // pre-render charset for SDF
  onSync?: (troika) => void    // fires after text layout
>
  {string}
</Text>
```

### Text3D

```tsx
<Text3D
  font={string}                // REQUIRED: JSON font URL
  size?: number                // default: 1
  height?: number              // extrusion depth (default: 0.2)
  bevelEnabled?: boolean       // default: false
  bevelSize?: number           // default: 0.02
  bevelThickness?: number      // default: 0.1
  bevelSegments?: number       // default: 1
  bevelOffset?: number         // default: 0
  curveSegments?: number       // default: 8
  letterSpacing?: number       // default: 0
  lineHeight?: number          // default: 1
>
  {string}
  <meshStandardMaterial />     // MUST provide material as child
</Text3D>
```

### Html

```tsx
<Html
  as?: string                  // wrapper element (default: 'div')
  transform?: boolean          // transform with 3D position
  sprite?: boolean             // always face camera
  distanceFactor?: number      // scale based on distance from camera
  center?: boolean             // center the element
  occlude?: boolean | Object3D[]  // hide behind 3D objects
  zIndexRange?: [number, number]  // z-index range (default: [16777271, 0])
  portal?: React.RefObject     // render into portal
  prepend?: boolean            // prepend to container
  fullscreen?: boolean         // fill entire viewport
  className?: string
  style?: CSSProperties
  castShadow?: boolean
  receiveShadow?: boolean
  wrapperClass?: string
  pointerEvents?: string
/>
```

---

## Materials

### MeshTransmissionMaterial

```tsx
<MeshTransmissionMaterial
  transmission?: number        // transmission factor (default: 1)
  thickness?: number           // glass thickness (default: 0)
  roughness?: number           // surface roughness
  chromaticAberration?: number // color fringing (default: 0.06)
  anisotropy?: number          // anisotropic reflections
  distortion?: number          // distortion (default: 0)
  distortionScale?: number
  temporalDistortion?: number  // animated distortion
  ior?: number                 // index of refraction (default: 1.5)
  color?: string               // transmission color
  backside?: boolean           // render backside (default: false)
  backsideThickness?: number
  resolution?: number          // FBO resolution (default: 1024)
  samples?: number             // MSAA samples (default: 10)
  background?: THREE.Texture   // background texture
/>
```

### MeshReflectorMaterial

```tsx
<MeshReflectorMaterial
  blur?: [number, number]       // blur x/y (default: [0, 0])
  resolution?: number           // FBO resolution (default: 256)
  mixBlur?: number              // blur mix (default: 0)
  mixStrength?: number          // reflection strength (default: 1)
  roughness?: number
  depthScale?: number           // depth-based reflection (default: 0)
  minDepthThreshold?: number    // depth range start (default: 0.9)
  maxDepthThreshold?: number    // depth range end (default: 1)
  color?: string
  metalness?: number
  mirror?: number               // mirror factor (0..1)
/>
```

### shaderMaterial Helper

```tsx
const MyMaterial = shaderMaterial(
  uniforms: { [key: string]: any },   // uniform defaults
  vertexShader: string,                // GLSL vertex
  fragmentShader: string,              // GLSL fragment
  onInit?: (material: ShaderMaterial) => void
)
```

Returns a class. ALWAYS register with `extend({ MyMaterial })` before use in JSX.

---

## Loaders

### useGLTF

```tsx
const result = useGLTF(
  url: string | string[],
  useDraco?: boolean | string,  // enable Draco (true = CDN decoder)
  useMeshOpt?: boolean,         // enable MeshOpt
  extendLoader?: (loader: GLTFLoader) => void
)
// result: { nodes, materials, scene, animations, asset }
// nodes: Record<string, THREE.Mesh | THREE.Group> (by name)
// materials: Record<string, THREE.Material> (by name)

useGLTF.preload(url, useDraco?, useMeshOpt?)
```

### useTexture

```tsx
const texture = useTexture(url: string)
const textures = useTexture(urls: string[])
const maps = useTexture({
  map: string,
  normalMap?: string,
  roughnessMap?: string,
  metalnessMap?: string,
  aoMap?: string,
  displacementMap?: string,
  emissiveMap?: string,
})

useTexture.preload(url)
```

### useAnimations

```tsx
const { actions, names, mixer, ref } = useAnimations(
  clips: THREE.AnimationClip[],
  root?: React.RefObject<THREE.Object3D>
)
// actions: Record<string, THREE.AnimationAction | null>
// names: string[]
// mixer: THREE.AnimationMixer
// ref: React.RefObject (pass as ref to root group)
```

### useVideoTexture

```tsx
const texture = useVideoTexture(
  src: string | MediaStream,
  props?: {
    unsuspend?: 'canplay' | 'canplaythrough' | 'loadedmetadata'
    start?: boolean
    crossOrigin?: string
    muted?: boolean
    loop?: boolean
    playsInline?: boolean
  }
)
```

---

## Performance

### Instances / Instance

```tsx
<Instances
  limit?: number              // max instances (default: 1000)
  range?: number              // visible instances (default: limit)
>
  <boxGeometry />             // shared geometry
  <meshStandardMaterial />    // shared material
  <Instance
    position?: [x, y, z]
    rotation?: [x, y, z]
    scale?: number | [x, y, z]
    color?: string | THREE.Color
    onClick?: (e) => void
  />
</Instances>
```

### Merged

```tsx
<Merged
  meshes: THREE.Mesh[] | { [key: string]: THREE.Mesh }
>
  {(MeshComponent1, MeshComponent2, ...) => ReactNode}
  // or with object: {(meshes) => <meshes.Chair />}
</Merged>
```

### Detailed (LOD)

```tsx
<Detailed distances={[0, 50, 100]}>
  <HighPolyMesh />   // shown 0-50 units
  <MidPolyMesh />    // shown 50-100 units
  <LowPolyMesh />    // shown 100+ units
</Detailed>
```

### PerformanceMonitor

```tsx
<PerformanceMonitor
  ms?: number                  // sample window in ms (default: 200)
  iterations?: number          // iterations before action (default: 10)
  threshold?: number           // FPS threshold (default: 0.75)
  onIncline?: () => void       // FPS improving
  onDecline?: () => void       // FPS dropping
  onFallback?: () => void      // after N flipflops
  flipflops?: number           // max flipflops before fallback (default: Infinity)
  factor?: number              // current performance factor (0..1)
  onChange?: (api) => void     // fires on every sample
/>
```

---

## Staging

### Center

```tsx
<Center
  top?: boolean        // align top
  right?: boolean      // align right
  bottom?: boolean     // align bottom
  left?: boolean       // align left
  front?: boolean      // align front
  back?: boolean       // align back
  precise?: boolean    // use precise bounding box
  onCentered?: (props: { container, width, height, depth }) => void
/>
```

### Float

```tsx
<Float
  speed?: number                // animation speed (default: 1)
  rotationIntensity?: number    // rotation amount (default: 1)
  floatIntensity?: number       // float amount (default: 1)
  floatingRange?: [min, max]    // y-axis range (default: [-0.1, 0.1])
>
  {children}
</Float>
```

### Bounds

```tsx
<Bounds
  fit?: boolean          // auto-fit on mount (default: false)
  clip?: boolean         // auto-clip near/far (default: false)
  observe?: boolean      // watch for content changes (default: false)
  margin?: number        // extra padding (default: 1.2)
  maxDuration?: number   // animation duration
  interpolateFunc?: fn   // custom interpolation
>
  {children}
</Bounds>
```

Imperative: `ref.current.refresh().clip().fit()`

---

## Portals

### View

```tsx
<View
  index?: number              // render order
  frames?: number             // render frames (Infinity = continuous)
  track: React.RefObject      // DOM element to track
>
  {children}
</View>
```

### RenderTexture

```tsx
<RenderTexture
  width?: number
  height?: number
  frames?: number         // Infinity = continuous
  stencilBuffer?: boolean
  depthBuffer?: boolean
  generateMipmaps?: boolean
>
  {scene content}
</RenderTexture>
```

### MeshPortalMaterial

```tsx
<MeshPortalMaterial
  blend?: number          // 0 = no portal, 1 = full portal
  resolution?: number
  blur?: number
  worldUnits?: boolean
  eventPriority?: number
>
  {scene content rendered inside mesh}
</MeshPortalMaterial>
```
