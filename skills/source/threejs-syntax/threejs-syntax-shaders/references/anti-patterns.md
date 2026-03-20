# threejs-syntax-shaders -- Anti-Patterns

## Anti-Pattern 1: Bare Uniform Values

**WRONG:**
```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    uTime: 0.0,                    // WRONG -- bare value
    uColor: new THREE.Color(),     // WRONG -- bare value
  },
});
```

**CORRECT:**
```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0.0 },                    // ALWAYS wrap in { value: }
    uColor: { value: new THREE.Color() },      // ALWAYS wrap in { value: }
  },
});
```

**Why:** Three.js expects every uniform to be an object with a `value` property. Bare values are silently ignored -- the uniform is never sent to the GPU, and the shader uses uninitialized data.

---

## Anti-Pattern 2: Replacing Uniform Objects Instead of Mutating .value

**WRONG:**
```javascript
// In animation loop
material.uniforms.uTime = { value: performance.now() };  // WRONG -- replaces object
```

**CORRECT:**
```javascript
// In animation loop
material.uniforms.uTime.value = performance.now();  // ALWAYS mutate .value
```

**Why:** Three.js caches internal references to uniform objects. Replacing the object breaks the reference, so the GPU never receives the updated value.

---

## Anti-Pattern 3: Declaring Built-in Uniforms in ShaderMaterial

**WRONG:**
```glsl
// In a ShaderMaterial vertex shader
uniform mat4 modelViewMatrix;   // WRONG -- already injected
uniform mat4 projectionMatrix;  // WRONG -- already injected
attribute vec3 position;        // WRONG -- already injected
```

**CORRECT:**
```glsl
// In a ShaderMaterial vertex shader -- just use them directly
void main() {
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
```

**Why:** `ShaderMaterial` automatically prepends built-in uniform and attribute declarations. Redeclaring them causes a GLSL compilation error ("identifier already declared"). This ONLY applies to `ShaderMaterial` -- `RawShaderMaterial` requires explicit declarations.

---

## Anti-Pattern 4: Forgetting customProgramCacheKey with onBeforeCompile

**WRONG:**
```javascript
material.onBeforeCompile = (shader) => {
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    '#include <begin_vertex>\ntransformed.y += 1.0;'
  );
};
// No customProgramCacheKey -- Three.js may reuse a cached unpatched program
```

**CORRECT:**
```javascript
material.onBeforeCompile = (shader) => {
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    '#include <begin_vertex>\ntransformed.y += 1.0;'
  );
};
material.customProgramCacheKey = () => 'my-displaced-material';
```

**Why:** Three.js caches compiled shader programs by material type and parameters. Without a unique cache key, Two materials of the same type may share a cached program, causing your `onBeforeCompile` modifications to silently disappear on the second instance.

---

## Anti-Pattern 5: Using GLSL1 Syntax with glslVersion: GLSL3

**WRONG:**
```javascript
const material = new THREE.ShaderMaterial({
  glslVersion: THREE.GLSL3,
  fragmentShader: `
    varying vec2 vUv;              // WRONG -- 'varying' is not valid in GLSL3
    void main() {
      gl_FragColor = vec4(1.0);    // WRONG -- gl_FragColor does not exist in GLSL3
    }
  `,
});
```

**CORRECT:**
```javascript
const material = new THREE.ShaderMaterial({
  glslVersion: THREE.GLSL3,
  fragmentShader: `
    precision highp float;
    in vec2 vUv;                   // 'varying' becomes 'in' in fragment shader
    out vec4 fragColor;            // Declare output explicitly

    void main() {
      fragColor = vec4(1.0);      // Write to declared output
    }
  `,
});
```

**Why:** GLSL 3.0 ES removes `attribute`, `varying`, `gl_FragColor`, and `texture2D()`. Using them causes compilation errors. ALWAYS use `in`/`out`, a declared output variable, and `texture()`.

---

## Anti-Pattern 6: Forgetting precision in RawShaderMaterial

**WRONG:**
```javascript
const material = new THREE.RawShaderMaterial({
  vertexShader: `
    attribute vec3 position;      // WRONG -- no precision declaration
    void main() {
      gl_Position = vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    void main() {                 // WRONG -- no precision declaration
      gl_FragColor = vec4(1.0);
    }
  `,
});
```

**CORRECT:**
```javascript
const material = new THREE.RawShaderMaterial({
  vertexShader: `
    precision highp float;
    attribute vec3 position;
    void main() {
      gl_Position = vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    precision highp float;
    void main() {
      gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});
```

**Why:** `RawShaderMaterial` injects NOTHING -- no precision qualifier, no uniforms, no attributes. The fragment shader requires an explicit precision declaration on mobile GPUs and many desktop drivers. Omitting it causes a GLSL compilation error.

---

## Anti-Pattern 7: Using #include Chunks in RawShaderMaterial

**WRONG:**
```javascript
const material = new THREE.RawShaderMaterial({
  vertexShader: `
    precision highp float;
    #include <common>              // WRONG -- not processed in RawShaderMaterial
    attribute vec3 position;
    void main() {
      gl_Position = vec4(position, 1.0);
    }
  `,
});
```

**CORRECT (use ShaderMaterial instead):**
```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: `
    #include <common>              // Works in ShaderMaterial
    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
});
```

**Or manually inline the chunk:**
```javascript
const material = new THREE.RawShaderMaterial({
  vertexShader: `
    precision highp float;
    ${THREE.ShaderChunk.common}
    attribute vec3 position;
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
});
```

**Why:** `RawShaderMaterial` does NOT process `#include` directives. The string `#include <common>` is passed literally to the GLSL compiler, which does not understand it, causing a compilation error.

---

## Anti-Pattern 8: Setting needsUpdate on Uniform Changes

**WRONG:**
```javascript
material.uniforms.uTime.value = 1.0;
material.needsUpdate = true;          // WRONG -- unnecessary and expensive
```

**CORRECT:**
```javascript
material.uniforms.uTime.value = 1.0;  // Just update the value -- no needsUpdate needed
```

**Why:** Uniform values are sent to the GPU every frame automatically. Setting `material.needsUpdate = true` triggers a full shader recompilation, which is extremely expensive (causes a frame stutter). ONLY set `needsUpdate = true` when changing `defines`, `vertexShader`, `fragmentShader`, `lights`, `fog`, or `clipping`.

---

## Anti-Pattern 9: Storing onBeforeCompile Shader Reference Without Null Check

**WRONG:**
```javascript
material.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = { value: 0 };
  material.userData.shader = shader;
};

// In animation loop -- crashes if shader has not compiled yet
material.userData.shader.uniforms.uTime.value = clock.getElapsedTime();
```

**CORRECT:**
```javascript
material.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = { value: 0 };
  material.userData.shader = shader;
};

// In animation loop -- ALWAYS check existence
if (material.userData.shader) {
  material.userData.shader.uniforms.uTime.value = clock.getElapsedTime();
}
```

**Why:** The `onBeforeCompile` callback runs lazily on first render. Before the first `renderer.render()` call, `material.userData.shader` is `undefined`. Additionally, setting `material.needsUpdate = true` can recreate the shader object, invalidating old references. ALWAYS guard access with a null check.

---

## Official Sources

- https://threejs.org/docs/#api/en/materials/ShaderMaterial
- https://threejs.org/docs/#api/en/materials/RawShaderMaterial
- https://threejs.org/docs/#api/en/renderers/shaders/ShaderChunk
