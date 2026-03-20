# threejs-syntax-loaders — Methods Reference

## Loader Base Class

All loaders inherit from `THREE.Loader`.

### Constructor

```javascript
new THREE.Loader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(url: string, onLoad: Function, onProgress?: Function, onError?: Function)` | `void` | Load asset with callbacks |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<any>` | Load asset with Promise |
| `setPath` | `(path: string)` | `this` | Set base URL prefix prepended to all load URLs |
| `setResourcePath` | `(path: string)` | `this` | Set path for resolving referenced resources (textures in models) |
| `setCrossOrigin` | `(value: string)` | `this` | Set CORS mode (`'anonymous'` or `'use-credentials'`) |
| `setWithCredentials` | `(value: boolean)` | `this` | Enable credentials for cross-origin requests |
| `setRequestHeader` | `(header: object)` | `this` | Set custom HTTP headers as key-value pairs |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `manager` | `LoadingManager` | The LoadingManager instance for this loader |
| `crossOrigin` | `string` | CORS mode |
| `withCredentials` | `boolean` | Credentials flag |
| `path` | `string` | Base path prefix |
| `resourcePath` | `string` | Resource resolution path |
| `requestHeader` | `object` | Custom HTTP headers |

---

## LoadingManager

### Constructor

```javascript
new THREE.LoadingManager(
  onLoad?: () => void,
  onProgress?: (url: string, itemsLoaded: number, itemsTotal: number) => void,
  onError?: (url: string) => void
)
```

### Callbacks (assignable properties)

| Callback | Signature | When Invoked |
|----------|-----------|--------------|
| `onStart` | `(url: string, itemsLoaded: number, itemsTotal: number) => void` | First item begins loading |
| `onLoad` | `() => void` | All items finished loading |
| `onProgress` | `(url: string, itemsLoaded: number, itemsTotal: number) => void` | Each individual item completes |
| `onError` | `(url: string) => void` | An item fails to load |

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `itemStart` | `(url: string)` | `void` | Manually register a loading item |
| `itemEnd` | `(url: string)` | `void` | Manually mark item as loaded |
| `itemError` | `(url: string)` | `void` | Manually mark item as failed |
| `resolveURL` | `(url: string)` | `string` | Resolve URL through any registered URL modifier |
| `setURLModifier` | `(callback: (url: string) => string)` | `this` | Register custom URL rewriting function |
| `addHandler` | `(regex: RegExp, loader: Loader)` | `this` | Auto-select loader by filename pattern |
| `removeHandler` | `(regex: RegExp)` | `this` | Remove a registered handler |

---

## GLTFLoader

### Constructor

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
new GLTFLoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(url: string, onLoad: (gltf: GLTF) => void, onProgress?: Function, onError?: Function)` | `void` | Load glTF/GLB with callbacks |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<GLTF>` | Load glTF/GLB with Promise |
| `parse` | `(data: ArrayBuffer \| string, path: string, onLoad: (gltf: GLTF) => void, onError?: Function)` | `void` | Parse raw data directly |
| `setDRACOLoader` | `(dracoLoader: DRACOLoader)` | `this` | Enable Draco geometry decompression |
| `setKTX2Loader` | `(ktx2Loader: KTX2Loader)` | `this` | Enable KTX2 texture decompression |
| `setMeshoptDecoder` | `(decoder: typeof MeshoptDecoder)` | `this` | Enable meshopt geometry decompression |
| `register` | `(plugin: (parser: GLTFParser) => GLTFLoaderPlugin)` | `this` | Register glTF extension plugin |
| `unregister` | `(plugin: (parser: GLTFParser) => GLTFLoaderPlugin)` | `this` | Unregister glTF extension plugin |

### GLTF Result Object

```typescript
interface GLTF {
  scene: THREE.Group;                    // Root scene node
  scenes: THREE.Group[];                 // All scenes in file
  cameras: THREE.Camera[];              // Embedded cameras
  animations: THREE.AnimationClip[];    // Animation clips
  asset: {
    generator: string;
    version: string;                    // glTF spec version (e.g., "2.0")
  };
  parser: GLTFParser;                    // Internal parser (advanced)
  userData: Record<string, any>;        // Custom extension data
}
```

---

## DRACOLoader

### Constructor

```javascript
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
new DRACOLoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `setDecoderPath` | `(path: string)` | `this` | Path to directory containing `draco_decoder.wasm` and `draco_wasm_wrapper.js` |
| `setDecoderConfig` | `(config: { type: 'js' \| 'wasm' })` | `this` | Decoder type -- WASM is default and significantly faster |
| `preload` | `()` | `this` | Pre-fetch decoder before first load (reduces first-load latency) |
| `dispose` | `()` | `void` | Free WASM decoder memory and terminate worker |
| `load` | `(url: string, onLoad: Function, onProgress?: Function, onError?: Function)` | `void` | Load Draco-compressed geometry |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<BufferGeometry>` | Load with Promise |

---

## KTX2Loader

### Constructor

```javascript
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
new KTX2Loader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `setTranscoderPath` | `(path: string)` | `this` | Path to Basis Universal transcoder files |
| `detectSupport` | `(renderer: WebGLRenderer)` | `this` | Detect GPU compressed texture support -- MUST call before loading |
| `load` | `(url: string, onLoad: Function, onProgress?: Function, onError?: Function)` | `void` | Load KTX2 texture |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<CompressedTexture>` | Load with Promise |
| `dispose` | `()` | `void` | Free transcoder resources |

---

## TextureLoader

### Constructor

```javascript
new THREE.TextureLoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(url: string, onLoad?: (texture: Texture) => void, onProgress?: Function, onError?: Function)` | `Texture` | Load texture -- returns Texture immediately (populated async) |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<Texture>` | Load texture with Promise |

---

## CubeTextureLoader

### Constructor

```javascript
new THREE.CubeTextureLoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(urls: string[6], onLoad?: Function, onProgress?: Function, onError?: Function)` | `CubeTexture` | Load 6 images as cube map. Order: px, nx, py, ny, pz, nz |
| `loadAsync` | `(urls: string[6], onProgress?: Function)` | `Promise<CubeTexture>` | Load with Promise |
| `setPath` | `(path: string)` | `this` | Base path prepended to each URL |

---

## RGBELoader

### Constructor

```javascript
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
new RGBELoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(url: string, onLoad?: (texture: DataTexture) => void, onProgress?: Function, onError?: Function)` | `void` | Load .hdr file |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<DataTexture>` | Load with Promise |
| `setDataType` | `(type: number)` | `this` | Set texture data type (e.g., `THREE.HalfFloatType`) |

---

## FBXLoader

### Constructor

```javascript
import { FBXLoader } from 'three/addons/loaders/FBXLoader.js';
new FBXLoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(url: string, onLoad?: (group: Group) => void, onProgress?: Function, onError?: Function)` | `void` | Load FBX model |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<Group>` | Load with Promise |
| `parse` | `(data: ArrayBuffer, path: string)` | `Group` | Parse raw FBX data |

---

## OBJLoader

### Constructor

```javascript
import { OBJLoader } from 'three/addons/loaders/OBJLoader.js';
new OBJLoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(url: string, onLoad?: (group: Group) => void, onProgress?: Function, onError?: Function)` | `void` | Load OBJ file |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<Group>` | Load with Promise |
| `parse` | `(data: string)` | `Group` | Parse OBJ text data |
| `setMaterials` | `(materials: MTLLoader.MaterialCreator)` | `this` | Assign materials loaded from MTL file |

---

## MTLLoader

### Constructor

```javascript
import { MTLLoader } from 'three/addons/loaders/MTLLoader.js';
new MTLLoader(manager?: THREE.LoadingManager)
```

### Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `load` | `(url: string, onLoad?: (creator: MaterialCreator) => void, onProgress?: Function, onError?: Function)` | `void` | Load MTL material file |
| `loadAsync` | `(url: string, onProgress?: Function)` | `Promise<MaterialCreator>` | Load with Promise |
| `parse` | `(data: string, path: string)` | `MaterialCreator` | Parse MTL text data |
| `setMaterialOptions` | `(options: object)` | `this` | Set material creation options |

### MaterialCreator Methods

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `preload` | `()` | `void` | Create all materials from parsed data |
| `getAsArray` | `()` | `Material[]` | Get all materials as array |
| `create` | `(name: string)` | `Material` | Create a specific material by name |
