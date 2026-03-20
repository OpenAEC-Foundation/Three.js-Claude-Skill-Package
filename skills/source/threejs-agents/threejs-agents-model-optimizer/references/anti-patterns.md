# anti-patterns.md -- threejs-agents-model-optimizer

## Anti-Pattern 1: Mixing Draco and Meshopt Compression

**WRONG:**
```bash
# Compress with Draco first
npx gltf-transform draco model.glb draco-model.glb

# Then try to add meshopt on top
gltfpack -i draco-model.glb -o final.glb -cc
```

**WHY:** Draco and meshopt are mutually exclusive mesh compression formats. They use different GLTF extensions (`KHR_draco_mesh_compression` vs `EXT_meshopt_compression`). Applying both corrupts the file or silently drops one format. The loader will fail or produce garbage geometry.

**CORRECT:**
```bash
# Choose ONE compression format
npx gltf-transform draco model.glb output.glb
# OR
gltfpack -i model.glb -o output.glb -cc
```

---

## Anti-Pattern 2: Optimizing Without Assessment

**WRONG:**
```bash
# Just blindly apply everything
npx gltf-transform optimize model.glb output.glb --compress draco --texture-compress ktx2
# "It's smaller, ship it!"
```

**WHY:** Without knowing the original polygon count, texture sizes, and file size breakdown, you cannot make informed decisions. A model with 500 triangles and 40MB of textures needs texture optimization, not mesh simplification. Blindly simplifying wastes time and may degrade quality on the wrong axis.

**CORRECT:**
```bash
# ALWAYS assess first
npx gltf-transform inspect model.glb
# Read the output, then decide which optimizations to apply
```

---

## Anti-Pattern 3: ETC1S for Normal Maps

**WRONG:**
```bash
npx gltf-transform ktx2 model.glb output.glb \
  --filter "normalTexture=etc1s"
```

**WHY:** ETC1S is a lossy format optimized for perceptual color quality. Normal maps contain mathematical direction data, not perceptual color. ETC1S compression introduces block artifacts that cause visible lighting errors: faceted shading, banding on smooth surfaces, and sparkle artifacts.

**CORRECT:**
```bash
# ALWAYS use UASTC for normal maps
npx gltf-transform ktx2 model.glb output.glb \
  --filter "normalTexture=uastc" \
  --filter "baseColorTexture=etc1s"
```

---

## Anti-Pattern 4: Joining Animated Meshes

**WRONG:**
```bash
# Model has animated doors and windows
npx gltf-transform join building.glb joined.glb
# Now doors and windows are merged into walls -- animations break
```

**WHY:** The `join` command merges meshes that share materials into single meshes. If those meshes have independent animations (skeletal or morph target), joining them destroys the animation targets. The merged mesh cannot be animated per-part.

**CORRECT:**
```bash
# Flatten and dedup, but skip join for animated models
npx gltf-transform dedup building.glb step1.glb
npx gltf-transform flatten step1.glb step2.glb
# Do NOT join if parts are independently animated
npx gltf-transform weld step2.glb output.glb
```

---

## Anti-Pattern 5: Draco Compression in Blender Export

**WRONG:**
```
Blender Export Settings:
  Format: glTF Binary (.glb)
  Compression: ✓ Draco  <-- enabling this
```

**WHY:** Blender's built-in Draco encoder uses an older version and produces suboptimal compression ratios. More importantly, it prevents further pipeline optimization -- gltf-transform cannot modify Draco-compressed meshes without first decompressing them, which adds complexity and may introduce rounding errors.

**CORRECT:**
```
Blender Export Settings:
  Format: glTF Binary (.glb)
  Compression: ✗ None  <-- keep this off

# Then compress with gltf-transform
npx gltf-transform draco exported.glb final.glb
```

---

## Anti-Pattern 6: Over-Simplification Without Visual Check

**WRONG:**
```bash
# Simplify to 5% -- maximum compression!
npx gltf-transform simplify character.glb output.glb \
  --ratio 0.05 --error 0.02
# Ship without looking at it
```

**WHY:** Extreme simplification (below 10% of original) destroys silhouettes, UV mapping, and surface detail. Characters lose fingers, architectural models lose window frames, mechanical parts lose functional geometry. The `--error` tolerance alone cannot prevent all visual artifacts because it measures geometric deviation, not perceptual quality.

**CORRECT:**
```bash
# Simplify conservatively
npx gltf-transform simplify character.glb output.glb \
  --ratio 0.5 --error 0.001

# ALWAYS load the result in a Three.js viewer and compare side-by-side
# If more reduction is needed, decrease ratio incrementally (0.4, 0.3, ...)
```

---

## Anti-Pattern 7: Forgetting Decoder Setup at Runtime

**WRONG:**
```javascript
const loader = new GLTFLoader();
// Loading a Draco-compressed model without setting up DRACOLoader
const gltf = await loader.loadAsync('model-draco.glb');
// Error: "No DRACOLoader instance provided"
```

**WHY:** Draco and KTX2 compressed assets require WASM decoders at runtime. The GLTFLoader does not include these by default. Without the decoder, loading fails with a cryptic error or silently produces empty geometry.

**CORRECT:**
```javascript
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';

const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/');
dracoLoader.preload();

const ktx2Loader = new KTX2Loader();
ktx2Loader.setTranscoderPath('/basis/');
ktx2Loader.detectSupport(renderer);

const loader = new GLTFLoader();
loader.setDRACOLoader(dracoLoader);
loader.setKTX2Loader(ktx2Loader);
```

---

## Anti-Pattern 8: Not Disposing Decoders

**WRONG:**
```javascript
function loadModel(url) {
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath('/draco/');

  const loader = new GLTFLoader();
  loader.setDRACOLoader(dracoLoader);

  return loader.loadAsync(url);
  // dracoLoader WASM instance leaks every call
}
```

**WHY:** DRACOLoader and KTX2Loader instantiate WASM modules that consume significant memory. Creating new instances per load without disposing them causes memory leaks. The WASM modules are never garbage collected.

**CORRECT:**
```javascript
// Create ONCE, reuse, dispose when done
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/draco/');
dracoLoader.preload();

const loader = new GLTFLoader();
loader.setDRACOLoader(dracoLoader);

// Load multiple models with the same loader instance
await loader.loadAsync('model1.glb');
await loader.loadAsync('model2.glb');

// Dispose when ALL loading is complete
dracoLoader.dispose();
```

---

## Anti-Pattern 9: UASTC on Mobile for Color Maps

**WRONG:**
```bash
npx gltf-transform ktx2 model.glb output.glb \
  --filter "baseColorTexture=uastc"
# Deploy to mobile web
```

**WHY:** UASTC textures are 8-16 bytes per texel in GPU VRAM. A single 2048x2048 UASTC texture uses 32-64MB of VRAM. Mobile GPUs typically have 1-4GB of shared VRAM. Using UASTC for color maps on mobile causes out-of-memory crashes, texture thrashing, and severe frame drops.

**CORRECT:**
```bash
# Use ETC1S for color maps on mobile -- 10-20x smaller VRAM footprint
npx gltf-transform ktx2 model.glb output.glb \
  --filter "baseColorTexture=etc1s" \
  --quality 128
```

---

## Anti-Pattern 10: Discarding the Original Model

**WRONG:**
```bash
# Optimize in-place, overwriting the original
npx gltf-transform optimize model.glb model.glb --compress draco
# Original is gone forever
```

**WHY:** All optimization operations are lossy. Mesh simplification removes geometry. Texture compression reduces quality. Draco quantization rounds vertex positions. If a bug is found, visual quality needs improvement, or a different optimization strategy is needed, the original model is required. Without it, re-optimization compounds losses.

**CORRECT:**
```bash
# ALWAYS write to a new file
npx gltf-transform optimize model.glb model-optimized.glb --compress draco

# Keep originals in version control or asset storage
# Use a naming convention: model.glb (original), model-web.glb (optimized)
```

---

## Anti-Pattern 11: Wrong Transform Order

**WRONG:**
```bash
# Compress BEFORE simplifying
npx gltf-transform draco model.glb step1.glb
npx gltf-transform simplify step1.glb step2.glb --ratio 0.5
# Draco decompression + re-compression introduces double quantization errors
```

**WHY:** Draco compression quantizes vertex positions. Simplifying after Draco means the simplifier works on already-quantized (less precise) data, producing worse results. Additionally, the output would need to be re-compressed, applying quantization twice and compounding precision loss.

**CORRECT:**
```bash
# ALWAYS follow this order:
# 1. dedup → 2. flatten → 3. join → 4. weld → 5. simplify
# → 6. texture optimize → 7. compress (Draco) → 8. prune → 9. quantize
npx gltf-transform dedup model.glb step1.glb
npx gltf-transform simplify step1.glb step2.glb --ratio 0.5
npx gltf-transform draco step2.glb output.glb
```
