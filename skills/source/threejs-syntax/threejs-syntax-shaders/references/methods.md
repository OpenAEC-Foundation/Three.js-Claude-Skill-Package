# threejs-syntax-shaders -- Methods Reference

## ShaderMaterial Constructor

```javascript
new THREE.ShaderMaterial(parameters?: {
  uniforms?: { [name: string]: { value: any } },
  uniformsGroups?: THREE.UniformsGroup[],
  vertexShader?: string,
  fragmentShader?: string,
  defines?: { [name: string]: string | number },
  extensions?: {
    clipCullDistance?: boolean,
    multiDraw?: boolean,
  },
  wireframe?: boolean,
  wireframeLinewidth?: number,
  lights?: boolean,
  fog?: boolean,
  clipping?: boolean,
  glslVersion?: null | typeof THREE.GLSL3,
  defaultAttributeValues?: { [name: string]: number[] },
  // Inherited from Material:
  side?: THREE.Side,
  transparent?: boolean,
  opacity?: number,
  depthTest?: boolean,
  depthWrite?: boolean,
  blending?: THREE.Blending,
})
```

---

## RawShaderMaterial Constructor

```javascript
new THREE.RawShaderMaterial(parameters?: {
  // Same parameters as ShaderMaterial
  // Difference: NO built-in uniforms, attributes, or precision injected
  uniforms?: { [name: string]: { value: any } },
  vertexShader?: string,
  fragmentShader?: string,
  defines?: { [name: string]: string | number },
  glslVersion?: null | typeof THREE.GLSL3,
  // ...all Material base properties
})
```

---

## ShaderMaterial Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `uniforms` | `Object` | `{}` | Map of `{ name: { value: any } }` |
| `uniformsGroups` | `THREE.UniformsGroup[]` | `[]` | Uniform buffer objects |
| `vertexShader` | `string` | default vertex shader | GLSL vertex shader source |
| `fragmentShader` | `string` | default fragment shader | GLSL fragment shader source |
| `defines` | `Object` | `{}` | Preprocessor defines |
| `extensions` | `Object` | `{}` | WebGL extensions to enable |
| `wireframe` | `boolean` | `false` | Render as wireframe |
| `wireframeLinewidth` | `number` | `1` | Line width (limited to 1 on most platforms) |
| `lights` | `boolean` | `false` | If `true`, passes light data as uniforms |
| `fog` | `boolean` | `false` | If `true`, passes fog data as uniforms |
| `clipping` | `boolean` | `false` | If `true`, enables clipping planes |
| `glslVersion` | `string \| null` | `null` | `null` = GLSL1, `THREE.GLSL3` = GLSL 3.0 ES |
| `defaultAttributeValues` | `Object` | `{ color: [1,1,1], uv: [0,0], uv2: [0,0] }` | Fallback values for missing geometry attributes |
| `isShaderMaterial` | `boolean` | `true` | Read-only type flag |

---

## ShaderMaterial Methods (Inherited from Material)

| Method | Signature | Returns | Description |
|--------|-----------|---------|-------------|
| `clone` | `(): ShaderMaterial` | new instance | Deep clone the material |
| `copy` | `(source: ShaderMaterial): this` | `this` | Copy properties from another material |
| `dispose` | `(): void` | void | Free GPU resources |
| `toJSON` | `(meta?: object): object` | JSON | Serialize to JSON |
| `onBeforeCompile` | `(shader: object, renderer: WebGLRenderer): void` | void | Callback before shader compilation |
| `customProgramCacheKey` | `(): string` | string | Return custom key for shader program caching |

---

## Built-in Uniforms (Injected by ShaderMaterial)

### Transform Matrices

| Uniform | GLSL Type | Description |
|---------|-----------|-------------|
| `modelMatrix` | `mat4` | Object-to-world transform |
| `modelViewMatrix` | `mat4` | Object-to-camera (`viewMatrix * modelMatrix`) |
| `projectionMatrix` | `mat4` | Camera-to-clip projection |
| `viewMatrix` | `mat4` | World-to-camera transform |
| `normalMatrix` | `mat3` | Transpose inverse of `modelViewMatrix` |

### Camera

| Uniform | GLSL Type | Description |
|---------|-----------|-------------|
| `cameraPosition` | `vec3` | Camera position in world space |

### Lighting (when `lights: true`)

| Uniform | GLSL Type | Description |
|---------|-----------|-------------|
| `ambientLightColor` | `vec3` | Combined ambient light color |
| `directionalLights` | struct array | Direction, color for each directional light |
| `pointLights` | struct array | Position, color, distance, decay for each point light |
| `spotLights` | struct array | Position, direction, color, distance, decay, angle, penumbra |
| `hemisphereLights` | struct array | Sky color, ground color, direction |

---

## Built-in Attributes (Injected by ShaderMaterial)

| Attribute | GLSL Type | Source |
|-----------|-----------|--------|
| `position` | `vec3` | `geometry.attributes.position` |
| `normal` | `vec3` | `geometry.attributes.normal` |
| `uv` | `vec2` | `geometry.attributes.uv` |
| `uv2` | `vec2` | `geometry.attributes.uv2` |
| `tangent` | `vec4` | `geometry.attributes.tangent` (if present) |
| `color` | `vec3` | `geometry.attributes.color` (if present) |

---

## ShaderChunk API

```javascript
// Access any internal shader chunk by name
THREE.ShaderChunk['common']          // Returns string with source code
THREE.ShaderChunk['fog_pars_vertex'] // Returns fog vertex pars source
```

### Usage in GLSL (ShaderMaterial only)

```glsl
#include <common>
#include <fog_pars_vertex>
```

---

## onBeforeCompile Callback Signature

```javascript
material.onBeforeCompile = (shader: {
  uniforms: { [name: string]: { value: any } },
  vertexShader: string,
  fragmentShader: string,
  defines: { [name: string]: string | number },
}, renderer: THREE.WebGLRenderer) => void;
```

### customProgramCacheKey Signature

```javascript
material.customProgramCacheKey = (): string => {
  return 'unique-key-for-this-variant';
};
```

---

## UniformsLib

Three.js groups commonly-used uniforms into libraries:

```javascript
THREE.UniformsLib.common    // diffuse map, opacity, alphaMap, UV transform
THREE.UniformsLib.envmap    // environment map uniforms
THREE.UniformsLib.normalmap // normal map uniforms
THREE.UniformsLib.fog       // fogColor, fogNear, fogFar, fogDensity
THREE.UniformsLib.lights    // all light type uniforms
```

### UniformsUtils

```javascript
// Merge multiple uniform objects (deep clone)
const merged = THREE.UniformsUtils.merge([
  THREE.UniformsLib.common,
  THREE.UniformsLib.lights,
  { uCustom: { value: 1.0 } },
]);

// Clone uniforms (deep copy of values)
const cloned = THREE.UniformsUtils.clone(someUniforms);
```

---

## Official Sources

- https://threejs.org/docs/#api/en/materials/ShaderMaterial
- https://threejs.org/docs/#api/en/materials/RawShaderMaterial
- https://threejs.org/docs/#api/en/renderers/shaders/ShaderChunk
- https://threejs.org/docs/#api/en/renderers/shaders/UniformsLib
- https://threejs.org/docs/#api/en/renderers/shaders/UniformsUtils
