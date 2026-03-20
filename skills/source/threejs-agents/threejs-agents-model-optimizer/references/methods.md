# methods.md -- threejs-agents-model-optimizer

## gltf-transform CLI Commands

### inspect

```bash
npx gltf-transform inspect <input>
```

Returns mesh count, triangle count, texture dimensions, accessor count, file size breakdown. ALWAYS run before optimizing.

### dedup

```bash
npx gltf-transform dedup <input> <output>
```

Removes duplicate accessors, textures, and materials. Lossless operation. ALWAYS run first in any pipeline.

### flatten

```bash
npx gltf-transform flatten <input> <output>
```

Flattens the node hierarchy by applying parent transforms to children. Removes empty/intermediate nodes. Reduces draw call overhead.

### join

```bash
npx gltf-transform join <input> <output>
```

Merges meshes that share the same material into single draw calls. Reduces CPU overhead from draw call submission.

**NEVER** join meshes that need independent animation or interaction -- joining makes them inseparable.

### weld

```bash
npx gltf-transform weld <input> <output> [--tolerance <float>]
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--tolerance` | 0.0001 | Vertex merge distance threshold |

Merges vertices within the specified tolerance. Reduces vertex count without changing geometry shape.

### simplify

```bash
npx gltf-transform simplify <input> <output> --ratio <float> --error <float>
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--ratio` | 0.5 | Target ratio of original triangle count (0.0 - 1.0) |
| `--error` | 0.001 | Maximum allowed geometric error |
| `--lock-border` | false | Prevent simplification of mesh boundaries |

Uses meshoptimizer's simplification algorithm. The `--error` parameter is the maximum geometric deviation normalized to the mesh bounding box.

### draco

```bash
npx gltf-transform draco <input> <output> [options]
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--quantize-position` | 14 | Position quantization bits (1-16) |
| `--quantize-normal` | 10 | Normal quantization bits (1-16) |
| `--quantize-texcoord` | 12 | Texture coordinate quantization bits (1-16) |
| `--quantize-color` | 8 | Vertex color quantization bits (1-16) |
| `--quantize-generic` | 12 | Generic attribute quantization bits (1-16) |

Applies Draco mesh compression. Requires DRACOLoader at runtime in Three.js.

### ktx2

```bash
npx gltf-transform ktx2 <input> <output> [options]
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--slots` | all | Comma-separated texture slots to compress |
| `--filter` | auto | Per-slot compression: `etc1s` or `uastc` |
| `--quality` | 128 | ETC1S quality (1-255) |
| `--effort` | 0 | UASTC effort level (0-5) |

Compresses textures to KTX2 format using Basis Universal. Requires KTX2Loader at runtime.

### quantize

```bash
npx gltf-transform quantize <input> <output>
```

Reduces attribute precision (e.g., float32 to int16) without Draco. Smaller file size, no runtime decoder needed.

### prune

```bash
npx gltf-transform prune <input> <output>
```

Removes unreferenced accessors, textures, materials, and meshes. ALWAYS run as the last step before compression.

### resize

```bash
npx gltf-transform resize <input> <output> --width <int> --height <int>
```

Resizes all textures that exceed the specified dimensions. Maintains aspect ratio.

### optimize (combined)

```bash
npx gltf-transform optimize <input> <output> [--compress draco|meshopt] [--texture-compress ktx2|webp]
```

Convenience command that runs dedup, flatten, join, weld, prune, and optional compression in a single pass.

---

## Three.js Loader API

### DRACOLoader

```javascript
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `setDecoderPath` | `(path: string): this` | Path to Draco WASM decoder files |
| `setDecoderConfig` | `(config: object): this` | Decoder configuration |
| `preload` | `(): this` | Begin loading the decoder module |
| `dispose` | `(): void` | Release decoder resources |

### KTX2Loader

```javascript
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `setTranscoderPath` | `(path: string): this` | Path to Basis Universal transcoder WASM |
| `detectSupport` | `(renderer: WebGLRenderer): this` | Detect GPU compressed format support |
| `dispose` | `(): void` | Release transcoder resources |

### GLTFLoader Integration

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `setDRACOLoader` | `(loader: DRACOLoader): this` | Enable Draco decompression |
| `setKTX2Loader` | `(loader: KTX2Loader): this` | Enable KTX2 texture decoding |
| `setMeshoptDecoder` | `(decoder: object): this` | Enable meshopt decompression |
| `load` | `(url, onLoad, onProgress, onError): void` | Load GLTF/GLB file |
| `loadAsync` | `(url, onProgress): Promise<GLTF>` | Promise-based loading |

### THREE.LOD

```javascript
const lod = new THREE.LOD();
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `addLevel` | `(object: Object3D, distance: number, hysteresis?: number): this` | Add a LOD level |
| `getCurrentLevel` | `(): number` | Get current active level index |
| `getObjectForDistance` | `(distance: number): Object3D` | Get object at distance |
| `update` | `(camera: Camera): void` | Update LOD selection based on camera |
| `autoUpdate` | `boolean` (property) | Auto-update in render loop (default: true) |

The `hysteresis` parameter (r160+) prevents flickering at LOD boundaries by requiring objects to move beyond the threshold by this fraction before switching.

---

## gltf-transform Node.js API

### Core Classes

```javascript
import { NodeIO, WebIO, Document } from '@gltf-transform/core';
```

| Class | Description |
|-------|-------------|
| `NodeIO` | Read/write GLTF in Node.js (filesystem + network) |
| `WebIO` | Read/write GLTF in browser (fetch-based) |
| `Document` | In-memory GLTF document graph |

### Transform Functions

```javascript
import {
  dedup, flatten, join, weld, simplify,
  prune, quantize, draco, meshopt,
  textureCompress, textureResize
} from '@gltf-transform/functions';
```

ALWAYS call `document.transform()` with transforms in dependency order:

```javascript
await document.transform(
  dedup(),           // 1. Remove duplicates
  flatten(),         // 2. Flatten hierarchy
  join(),            // 3. Merge meshes
  weld(options),     // 4. Merge vertices
  simplify(options), // 5. Reduce triangles
  textureResize({ size: [2048, 2048] }),  // 6. Resize textures
  textureCompress({ targetFormat: 'ktx2' }), // 7. Compress textures
  draco(),           // 8. Compress mesh
  prune(),           // 9. Remove unused
  quantize(),        // 10. Reduce precision
);
```
