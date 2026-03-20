# examples.md -- threejs-agents-model-optimizer

## Example 1: Full Web Optimization Pipeline (CLI)

Optimize a 50MB architectural model to under 5MB for desktop web delivery.

```bash
# Step 1: Assess the model
npx gltf-transform inspect building.glb
# Output shows: 450K triangles, 12 textures at 4096x4096, 48MB total

# Step 2: Resize textures to 2048 max
npx gltf-transform resize building.glb step2.glb \
  --width 2048 --height 2048

# Step 3: Deduplicate shared data
npx gltf-transform dedup step2.glb step3.glb

# Step 4: Flatten empty nodes
npx gltf-transform flatten step3.glb step4.glb

# Step 5: Join meshes by material
npx gltf-transform join step4.glb step5.glb

# Step 6: Weld close vertices
npx gltf-transform weld step5.glb step6.glb --tolerance 0.001

# Step 7: Simplify mesh to 50%
npx gltf-transform simplify step6.glb step7.glb \
  --ratio 0.5 --error 0.001

# Step 8: Compress textures to KTX2
npx gltf-transform ktx2 step7.glb step8.glb

# Step 9: Apply Draco compression
npx gltf-transform draco step8.glb step9.glb

# Step 10: Prune unused data
npx gltf-transform prune step9.glb building-optimized.glb

# Verify result
npx gltf-transform inspect building-optimized.glb
# Expected: ~225K triangles, KTX2 textures, ~3-5MB
```

---

## Example 2: One-Command Optimization

For quick optimization without manual steps:

```bash
npx gltf-transform optimize input.glb output.glb \
  --compress draco \
  --texture-compress ktx2
```

---

## Example 3: Mobile-Optimized Asset

Strict optimization for mobile web (target: <2MB, <50K triangles).

```bash
# Aggressive texture resize
npx gltf-transform resize model.glb step1.glb \
  --width 1024 --height 1024

# Aggressive mesh simplification
npx gltf-transform simplify step1.glb step2.glb \
  --ratio 0.25 --error 0.005

# ETC1S compression for small file size
npx gltf-transform ktx2 step2.glb step3.glb \
  --filter "baseColorTexture=etc1s" \
  --quality 128

# Draco with aggressive quantization
npx gltf-transform draco step3.glb mobile-model.glb \
  --quantize-position 11 \
  --quantize-normal 8 \
  --quantize-texcoord 10
```

---

## Example 4: LOD Chain Generation

Generate 4 LOD levels from a single high-poly model.

```bash
# LOD0: Full quality (just optimize, no simplification)
npx gltf-transform dedup hero.glb lod0.glb
npx gltf-transform weld lod0.glb lod0.glb --tolerance 0.0001

# LOD1: 50% triangles
npx gltf-transform simplify hero.glb lod1.glb \
  --ratio 0.5 --error 0.001

# LOD2: 25% triangles
npx gltf-transform simplify hero.glb lod2.glb \
  --ratio 0.25 --error 0.005

# LOD3: 10% triangles
npx gltf-transform simplify hero.glb lod3.glb \
  --ratio 0.10 --error 0.01

# Compress all LODs
for f in lod0.glb lod1.glb lod2.glb lod3.glb; do
  npx gltf-transform draco "$f" "compressed-$f"
done
```

### Loading LOD Chain in Three.js

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/');
dracoLoader.preload();

const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

const lod = new THREE.LOD();

const lodFiles = [
  { file: 'compressed-lod0.glb', distance: 0 },
  { file: 'compressed-lod1.glb', distance: 15 },
  { file: 'compressed-lod2.glb', distance: 40 },
  { file: 'compressed-lod3.glb', distance: 100 },
];

// Load all LOD levels
const loadPromises = lodFiles.map(({ file, distance }) =>
  gltfLoader.loadAsync(file).then((gltf) => {
    lod.addLevel(gltf.scene, distance);
  })
);

await Promise.all(loadPromises);
scene.add(lod);

// LOD updates automatically if lod.autoUpdate is true (default)
```

---

## Example 5: Node.js Scripted Pipeline

Programmatic optimization for build scripts or CI/CD.

```javascript
import { NodeIO } from '@gltf-transform/core';
import { ALL_EXTENSIONS } from '@gltf-transform/extensions';
import {
  dedup, flatten, join, weld, simplify,
  textureCompress, draco, prune, quantize
} from '@gltf-transform/functions';
import draco3d from 'draco3dgltf';

async function optimizeModel(inputPath, outputPath, options = {}) {
  const {
    simplifyRatio = 0.5,
    simplifyError = 0.001,
    maxTextureSize = 2048,
    useDraco = true,
    useKTX2 = true,
  } = options;

  const io = new NodeIO().registerExtensions(ALL_EXTENSIONS);

  if (useDraco) {
    io.registerDependencies({
      'draco3d.decoder': await draco3d.createDecoderModule(),
      'draco3d.encoder': await draco3d.createEncoderModule(),
    });
  }

  const document = await io.read(inputPath);

  const transforms = [
    dedup(),
    flatten(),
    join(),
    weld({ tolerance: 0.0001 }),
  ];

  if (simplifyRatio < 1.0) {
    transforms.push(simplify({ ratio: simplifyRatio, error: simplifyError }));
  }

  if (useKTX2) {
    transforms.push(textureCompress({ targetFormat: 'ktx2' }));
  }

  if (useDraco) {
    transforms.push(draco());
  }

  transforms.push(prune(), quantize());

  await document.transform(...transforms);
  await io.write(outputPath, document);

  console.log(`Optimized: ${inputPath} -> ${outputPath}`);
}

// Usage
await optimizeModel('assets/building.glb', 'dist/building.glb', {
  simplifyRatio: 0.5,
  useDraco: true,
  useKTX2: true,
});
```

---

## Example 6: Three.js Runtime with All Decoders

Complete loader setup supporting Draco, KTX2, and Meshopt compressed models.

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

function createOptimizedLoader(renderer) {
  // Draco decoder
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
  dracoLoader.preload();

  // KTX2 transcoder
  const ktx2Loader = new KTX2Loader();
  ktx2Loader.setTranscoderPath(
    'https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/libs/basis/'
  );
  ktx2Loader.detectSupport(renderer);

  // GLTF loader with all decoders
  const gltfLoader = new GLTFLoader();
  gltfLoader.setDRACOLoader(dracoLoader);
  gltfLoader.setKTX2Loader(ktx2Loader);
  gltfLoader.setMeshoptDecoder(MeshoptDecoder);

  return {
    loader: gltfLoader,
    dispose() {
      dracoLoader.dispose();
      ktx2Loader.dispose();
    },
  };
}

// Usage
const renderer = new THREE.WebGLRenderer();
const { loader, dispose } = createOptimizedLoader(renderer);

const gltf = await loader.loadAsync('optimized-model.glb');
scene.add(gltf.scene);

// ALWAYS dispose decoders when no longer needed
dispose();
```

---

## Example 7: gltfpack Quick Optimization

One-command optimization using gltfpack (meshopt-based).

```bash
# Basic optimization with meshopt compression
gltfpack -i model.glb -o optimized.glb -cc -tc

# With simplification to 50%
gltfpack -i model.glb -o optimized.glb -cc -tc -si 0.5

# With texture power-of-two resize
gltfpack -i model.glb -o optimized.glb -cc -tc -tp

# Maximum compression
gltfpack -i model.glb -o optimized.glb -cc -tc -si 0.5 -tp
```

### Loading in Three.js

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

const loader = new GLTFLoader();
loader.setMeshoptDecoder(MeshoptDecoder);

const gltf = await loader.loadAsync('optimized.glb');
scene.add(gltf.scene);
```

---

## Example 8: Batch Optimization Script

Optimize all GLB files in a directory.

```bash
#!/bin/bash
INPUT_DIR="./assets/models"
OUTPUT_DIR="./dist/models"

mkdir -p "$OUTPUT_DIR"

for file in "$INPUT_DIR"/*.glb; do
  filename=$(basename "$file")
  echo "Optimizing: $filename"

  npx gltf-transform optimize "$file" "$OUTPUT_DIR/$filename" \
    --compress draco \
    --texture-compress ktx2

  # Report size reduction
  original=$(stat -f%z "$file" 2>/dev/null || stat -c%s "$file")
  optimized=$(stat -f%z "$OUTPUT_DIR/$filename" 2>/dev/null || stat -c%s "$OUTPUT_DIR/$filename")
  ratio=$(echo "scale=1; $optimized * 100 / $original" | bc)
  echo "  $original -> $optimized bytes (${ratio}%)"
done
```
