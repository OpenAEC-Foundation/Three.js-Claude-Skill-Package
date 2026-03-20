# API Signatures Reference (IFC/BIM Viewer)

## web-ifc — IfcAPI

The central class for all low-level IFC operations. ALWAYS initialize asynchronously before use.

```typescript
class IfcAPI {
  // Initialization
  SetWasmPath(path: string): void
  Init(): Promise<void>

  // Model lifecycle
  OpenModel(data: Uint8Array, settings?: LoaderSettings): number
  CloseModel(modelID: number): void
  IsModelOpen(modelID: number): boolean

  // Geometry extraction
  GetGeometry(modelID: number, expressID: number): IfcGeometry
  GetFlatMesh(modelID: number, expressID: number): FlatMesh
  GetPlacedGeometry(modelID: number, pg: PlacedGeometry): MeshData

  // Entity access
  GetLine(modelID: number, expressID: number, flatten?: boolean): object
  GetAllLines(modelID: number): number[]
  GetLineIDsWithType(modelID: number, type: number): number[]
  GetAllTypesOfModel(modelID: number): TypeInfo[]

  // Entity modification
  WriteLine(modelID: number, lineObject: object): void
  CreateModel(settings?: LoaderSettings): number

  // Coordinate system
  GetCoordinationMatrix(modelID: number): number[]
  SetGeometryTransformation(modelID: number, transformationMatrix: number[]): void
}
```

### LoaderSettings

```typescript
interface LoaderSettings {
  COORDINATE_TO_ORIGIN?: boolean;  // Move model to origin
  USE_FAST_BOOLS?: boolean;        // Faster boolean operations (less accurate)
  CIRCLE_SEGMENTS_LOW?: number;    // Segment count for small circles (default: 5)
  CIRCLE_SEGMENTS_MEDIUM?: number; // Segment count for medium circles (default: 8)
  CIRCLE_SEGMENTS_HIGH?: number;   // Segment count for large circles (default: 12)
  BOOL_ABORT_THRESHOLD?: number;   // Abort boolean ops after N iterations
}
```

### FlatMesh

```typescript
interface FlatMesh {
  geometries: PlacedGeometry[];  // Array of geometry placements
  expressID: number;             // ExpressID of the IFC element
}
```

### PlacedGeometry

```typescript
interface PlacedGeometry {
  color: { x: number; y: number; z: number; w: number }; // RGBA (0-1)
  flatTransformation: Float64Array;  // 4x4 column-major transform matrix
  geometryExpressID: number;         // ExpressID of the geometry definition
}
```

### MeshData

```typescript
interface MeshData {
  vertexData: Float32Array;  // Interleaved [px, py, pz, nx, ny, nz, ...] per vertex
  indexData: Uint32Array;    // Triangle indices
}
```

---

## @thatopen/components — Core API

### Components (Central Manager)

```typescript
class Components {
  get<T extends Component>(ComponentClass: new (...args: any[]) => T): T
  init(): void
  dispose(): void
  readonly enabled: boolean
}
```

### Worlds

```typescript
class Worlds extends Component {
  create(): World
  list: Map<string, World>
  delete(world: World): void
}
```

### World

```typescript
interface World {
  scene: SimpleScene
  camera: SimpleCamera
  renderer: SimpleRenderer
  uuid: string
  enabled: boolean
}
```

### SimpleScene

```typescript
class SimpleScene {
  constructor(components: Components)
  setup(config?: { backgroundColor?: THREE.Color }): void
  readonly three: THREE.Scene  // Access underlying Three.js scene
  dispose(): void
}
```

### SimpleCamera

```typescript
class SimpleCamera {
  constructor(components: Components)
  readonly three: THREE.PerspectiveCamera | THREE.OrthographicCamera
  readonly controls: CameraControls  // camera-controls library instance
  dispose(): void
}
```

### SimpleRenderer

```typescript
class SimpleRenderer {
  constructor(components: Components, container: HTMLElement)
  readonly three: THREE.WebGLRenderer
  dispose(): void
}
```

### IfcLoader

```typescript
class IfcLoader extends Component {
  setup(config?: IfcLoaderConfig): Promise<void>
  load(data: Uint8Array, coordinateToOrigin?: boolean): Promise<THREE.Group>
  readonly settings: IfcLoaderSettings
}
```

### FragmentsManager

```typescript
class FragmentsManager extends Component {
  readonly list: Map<string, Fragment>
  dispose(): void
  export(group: THREE.Group): Uint8Array    // Export to .frag format
  load(data: Uint8Array): THREE.Group       // Load from .frag format
}
```

### Highlighter

```typescript
class Highlighter extends Component {
  setup(config?: { world: World }): void
  highlight(name: string, removePrevious?: boolean): Map<string, Set<number>>
  clear(name?: string): void
  readonly selection: Map<string, Map<string, Set<number>>>
}
```

### Clipper

```typescript
class Clipper extends Component {
  enabled: boolean
  create(world: World): void   // Creates a clipping plane at click position
  delete(world: World): void   // Removes last clipping plane
  deleteAll(): void            // Removes all clipping planes
  dispose(): void
}
```

---

## IFC Type Constants (web-ifc)

All constants are exported from the `web-ifc` module as numeric values:

```typescript
// Structural elements
const IFCWALL: number
const IFCWALLSTANDARDCASE: number
const IFCSLAB: number
const IFCCOLUMN: number
const IFCBEAM: number
const IFCPLATE: number
const IFCMEMBER: number
const IFCFOOTING: number
const IFCPILE: number

// Openings and furnishing
const IFCDOOR: number
const IFCWINDOW: number
const IFCFURNISHINGELEMENT: number
const IFCCOVERING: number
const IFCRAILING: number
const IFCSTAIR: number
const IFCSTAIRFLIGHT: number
const IFCRAMP: number
const IFCRAMPFLIGHT: number

// Spatial structure
const IFCPROJECT: number
const IFCSITE: number
const IFCBUILDING: number
const IFCBUILDINGSTOREY: number
const IFCSPACE: number

// Relationships
const IFCRELAGGREGATES: number
const IFCRELCONTAINEDINSPATIALSTRUCTURE: number
const IFCRELDEFINESBYPROPERTIES: number
const IFCRELDEFINESBYTYPE: number
const IFCRELASSOCIATESMATERIAL: number
const IFCRELCONNECTSPATHELEMENTS: number

// Properties
const IFCPROPERTYSET: number
const IFCPROPERTYSINGLEVALUE: number
const IFCELEMENTQUANTITY: number
const IFCQUANTITYLENGTH: number
const IFCQUANTITYAREA: number
const IFCQUANTITYVOLUME: number

// MEP (Mechanical, Electrical, Plumbing)
const IFCFLOWSEGMENT: number
const IFCFLOWTERMINAL: number
const IFCFLOWFITTING: number
const IFCDISTRIBUTIONELEMENT: number
```
