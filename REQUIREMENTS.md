# Three.js Skill Package — REQUIREMENTS

## Quality Guarantees

Every skill in this package MUST meet the following criteria:

### 1. Format-correct
- SKILL.md < 500 lines
- Valid YAML frontmatter with `name` and `description`
- `name`: kebab-case, max 64 characters, prefix `threejs-`
- `description`: max 1024 characters, includes trigger keywords
- `references/` directory with: methods.md, examples.md, anti-patterns.md

### 2. Code-accurate
- All JavaScript/TypeScript examples MUST be valid and runnable
- Import statements MUST use ES module syntax (`import { Scene } from 'three'`)
- API calls MUST match Three.js r160+ signatures
- Shader code (GLSL/WGSL) MUST be syntactically valid
- React Three Fiber JSX MUST follow R3F 8.x conventions

### 3. Version-explicit
- Target: Three.js r160+ (ES modules, MIT license)
- React Three Fiber: 8.x
- Drei: latest compatible with R3F 8.x
- WebGPU Renderer: Three.js built-in (experimental)
- Note breaking changes between Three.js revisions where applicable

### 4. Anti-pattern-free
- Every skill MUST document known mistakes in references/anti-patterns.md
- Common memory leak patterns documented
- Performance anti-patterns documented
- Disposal patterns for textures, geometries, materials documented

### 5. Deterministic
- Use ALWAYS/NEVER language
- No hedging words: "might", "consider", "often", "usually"
- Decision trees for conditional logic

### 6. Self-contained
- Each skill works independently without requiring other skills
- All necessary context included within the skill
- Cross-references to related skills are informational only

### 7. English-only
- All skill content in English
- Claude reads English, responds in any language
- No Dutch, German, or other languages in skill files

## Per-Category Requirements

### Core Skills
- MUST cover the complete scene graph hierarchy (Scene > Object3D > Mesh/Light/Camera)
- MUST include renderer initialization for both WebGL and WebGPU
- MUST document the Three.js math utilities (Vector3, Matrix4, Quaternion, Euler)

### Syntax Skills
- MUST include complete BufferGeometry attribute catalogs
- MUST show both basic and PBR material configurations
- MUST document all built-in loader types (GLTF, FBX, OBJ, texture loaders)
- MUST cover OrbitControls, FlyControls, PointerLockControls

### Implementation Skills
- MUST produce complete, runnable scene setups
- MUST include realistic 3D examples (not just toy scenes)
- MUST document feature-specific API patterns and shader configurations

### Error Skills
- MUST include real error scenarios with reproduction steps
- MUST provide fix patterns for every documented error
- MUST cover performance profiling with Three.js stats and Chrome DevTools

### Agent Skills
- MUST include decision trees for scene composition
- MUST validate generated code against Three.js API contracts
