# threejs-syntax-shaders -- Examples

## Example 1: Basic Custom ShaderMaterial

A minimal ShaderMaterial with a time-based color animation.

```javascript
import * as THREE from 'three';

const material = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0.0 },
    uColor: { value: new THREE.Color(0x3399ff) },
  },
  vertexShader: `
    varying vec2 vUv;

    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform float uTime;
    uniform vec3 uColor;
    varying vec2 vUv;

    void main() {
      float pulse = sin(uTime * 3.0) * 0.5 + 0.5;
      vec3 color = mix(uColor, vec3(1.0), pulse * vUv.y);
      gl_FragColor = vec4(color, 1.0);
    }
  `,
});

const mesh = new THREE.Mesh(new THREE.PlaneGeometry(2, 2), material);

// Animation loop
const clock = new THREE.Clock();
function animate() {
  material.uniforms.uTime.value = clock.getElapsedTime();
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

---

## Example 2: Vertex Displacement with Texture

Displaces vertices along their normals using a height map texture.

```javascript
import * as THREE from 'three';

const loader = new THREE.TextureLoader();
const heightMap = loader.load('/textures/heightmap.png');

const material = new THREE.ShaderMaterial({
  uniforms: {
    uHeightMap: { value: heightMap },
    uDisplacement: { value: 0.5 },
  },
  vertexShader: `
    uniform sampler2D uHeightMap;
    uniform float uDisplacement;
    varying vec2 vUv;
    varying float vHeight;

    void main() {
      vUv = uv;
      float height = texture2D(uHeightMap, uv).r;
      vHeight = height;
      vec3 displaced = position + normal * height * uDisplacement;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(displaced, 1.0);
    }
  `,
  fragmentShader: `
    varying vec2 vUv;
    varying float vHeight;

    void main() {
      vec3 low = vec3(0.0, 0.2, 0.8);
      vec3 high = vec3(1.0, 0.9, 0.3);
      vec3 color = mix(low, high, vHeight);
      gl_FragColor = vec4(color, 1.0);
    }
  `,
});

const geometry = new THREE.PlaneGeometry(10, 10, 256, 256);
const mesh = new THREE.Mesh(geometry, material);
```

---

## Example 3: onBeforeCompile -- Patching MeshStandardMaterial

Adds a wave vertex displacement to a standard PBR material while preserving all lighting and shadow behavior.

```javascript
import * as THREE from 'three';

const material = new THREE.MeshStandardMaterial({
  color: 0x2194ce,
  roughness: 0.4,
  metalness: 0.1,
});

material.onBeforeCompile = (shader) => {
  // Add custom uniforms
  shader.uniforms.uTime = { value: 0.0 };
  shader.uniforms.uAmplitude = { value: 0.3 };

  // Inject uniform declaration at top of vertex shader
  shader.vertexShader = `
    uniform float uTime;
    uniform float uAmplitude;
  ` + shader.vertexShader;

  // Replace begin_vertex chunk to add displacement
  shader.vertexShader = shader.vertexShader.replace(
    '#include <begin_vertex>',
    `
    #include <begin_vertex>
    float wave = sin(transformed.x * 4.0 + uTime * 2.0) *
                 cos(transformed.z * 4.0 + uTime * 1.5);
    transformed.y += wave * uAmplitude;
    `
  );

  // Store shader reference for uniform updates
  material.userData.shader = shader;
};

// ALWAYS override customProgramCacheKey with onBeforeCompile
material.customProgramCacheKey = () => 'wavy-standard';

// Animation loop
const clock = new THREE.Clock();
function animate() {
  if (material.userData.shader) {
    material.userData.shader.uniforms.uTime.value = clock.getElapsedTime();
  }
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
animate();
```

---

## Example 4: RawShaderMaterial with GLSL3

A RawShaderMaterial using GLSL 3.0 ES syntax for full control.

```javascript
import * as THREE from 'three';

const material = new THREE.RawShaderMaterial({
  glslVersion: THREE.GLSL3,
  uniforms: {
    uProjectionMatrix: { value: new THREE.Matrix4() },
    uModelViewMatrix: { value: new THREE.Matrix4() },
    uTime: { value: 0.0 },
  },
  vertexShader: `
    precision highp float;

    in vec3 position;
    in vec2 uv;

    uniform mat4 uProjectionMatrix;
    uniform mat4 uModelViewMatrix;
    uniform float uTime;

    out vec2 vUv;
    out float vWave;

    void main() {
      vUv = uv;
      vec3 pos = position;
      pos.z = sin(pos.x * 5.0 + uTime) * 0.2;
      vWave = pos.z;
      gl_Position = uProjectionMatrix * uModelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    precision highp float;

    in vec2 vUv;
    in float vWave;

    out vec4 fragColor;

    void main() {
      vec3 color = vec3(vUv, 0.5 + vWave);
      fragColor = vec4(color, 1.0);
    }
  `,
});

// With RawShaderMaterial, you MUST manually update the matrix uniforms
const mesh = new THREE.Mesh(new THREE.PlaneGeometry(4, 4, 64, 64), material);

function animate() {
  mesh.updateMatrixWorld();
  material.uniforms.uModelViewMatrix.value.multiplyMatrices(
    camera.matrixWorldInverse,
    mesh.matrixWorld
  );
  material.uniforms.uProjectionMatrix.value.copy(camera.projectionMatrix);
  material.uniforms.uTime.value = performance.now() / 1000;
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

---

## Example 5: ShaderMaterial with Fog and Lights Support

Using ShaderChunk includes to integrate with Three.js fog and lighting systems.

```javascript
import * as THREE from 'three';

const material = new THREE.ShaderMaterial({
  uniforms: {
    ...THREE.UniformsLib.fog,
    uColor: { value: new THREE.Color(0xff6600) },
  },
  vertexShader: `
    #include <common>
    #include <fog_pars_vertex>

    varying vec3 vNormal;

    void main() {
      vNormal = normalize(normalMatrix * normal);
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
      gl_Position = projectionMatrix * mvPosition;
      #include <fog_vertex>
    }
  `,
  fragmentShader: `
    #include <common>
    #include <fog_pars_fragment>

    uniform vec3 uColor;
    varying vec3 vNormal;

    void main() {
      float light = dot(vNormal, normalize(vec3(1.0, 1.0, 1.0)));
      light = clamp(light, 0.2, 1.0);
      gl_FragColor = vec4(uColor * light, 1.0);
      #include <fog_fragment>
    }
  `,
  fog: true,  // REQUIRED to enable fog uniform injection
});
```

---

## Official Sources

- https://threejs.org/docs/#api/en/materials/ShaderMaterial
- https://threejs.org/docs/#api/en/materials/RawShaderMaterial
- https://threejs.org/examples/#webgl_shader
- https://threejs.org/examples/#webgl_shader_lava
