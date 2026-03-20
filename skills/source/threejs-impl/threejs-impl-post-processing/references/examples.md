# examples.md -- threejs-impl-post-processing

> Working code examples for common Three.js post-processing setups.
> All examples use ES module imports and Three.js r160+.

---

## Example 1: Bloom Effect

HDR bloom that makes bright objects glow.

```js
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

// Renderer -- tone mapping MUST be enabled for bloom
const renderer = new THREE.WebGLRenderer( { antialias: false } );
renderer.setSize( window.innerWidth, window.innerHeight );
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.0;
document.body.appendChild( renderer.domElement );

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 75, window.innerWidth / window.innerHeight, 0.1, 1000 );
camera.position.z = 5;

// Emissive material -- bloom picks up values above threshold
const geometry = new THREE.SphereGeometry( 1, 32, 32 );
const material = new THREE.MeshStandardMaterial( {
  color: 0x000000,
  emissive: 0xff6600,
  emissiveIntensity: 2.0 // above 1.0 to trigger bloom
} );
scene.add( new THREE.Mesh( geometry, material ) );
scene.add( new THREE.AmbientLight( 0xffffff, 0.2 ) );

// Post-processing pipeline
const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );       // ALWAYS first
composer.addPass( new UnrealBloomPass(
  new THREE.Vector2( window.innerWidth, window.innerHeight ),
  1.5,  // strength
  0.4,  // radius
  0.85  // threshold
) );
composer.addPass( new OutputPass() );                       // ALWAYS last

// Resize handler
window.addEventListener( 'resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize( window.innerWidth, window.innerHeight );
  composer.setSize( window.innerWidth, window.innerHeight );
} );

function animate() {
  requestAnimationFrame( animate );
  composer.render(); // NOT renderer.render()
}
animate();
```

---

## Example 2: SSAO + Bloom Combo

Multiple effects combined in a single pipeline.

```js
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { GTAOPass } from 'three/addons/postprocessing/GTAOPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const renderer = new THREE.WebGLRenderer();
renderer.setSize( window.innerWidth, window.innerHeight );
renderer.toneMapping = THREE.ACESFilmicToneMapping;

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 60, window.innerWidth / window.innerHeight, 0.1, 100 );

const composer = new EffectComposer( renderer );

// Pass 1: RenderPass (ALWAYS first)
composer.addPass( new RenderPass( scene, camera ) );

// Pass 2: GTAO for ambient occlusion (better quality than SSAOPass)
const gtaoPass = new GTAOPass( scene, camera, window.innerWidth, window.innerHeight );
composer.addPass( gtaoPass );

// Pass 3: Bloom
const bloomPass = new UnrealBloomPass(
  new THREE.Vector2( window.innerWidth, window.innerHeight ),
  0.8, 0.3, 0.9
);
composer.addPass( bloomPass );

// Pass 4: OutputPass (ALWAYS last)
composer.addPass( new OutputPass() );

function animate() {
  requestAnimationFrame( animate );
  composer.render();
}
animate();
```

---

## Example 3: Object Outlines on Hover

Highlighting objects with OutlinePass on mouse hover.

```js
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { OutlinePass } from 'three/addons/postprocessing/OutlinePass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const renderer = new THREE.WebGLRenderer( { antialias: true } );
renderer.setSize( window.innerWidth, window.innerHeight );

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 60, window.innerWidth / window.innerHeight, 0.1, 100 );
camera.position.set( 0, 2, 5 );

// Create some objects
const objects = [];
for ( let i = 0; i < 5; i++ ) {
  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry( 1, 1, 1 ),
    new THREE.MeshStandardMaterial( { color: Math.random() * 0xffffff } )
  );
  mesh.position.x = ( i - 2 ) * 2;
  scene.add( mesh );
  objects.push( mesh );
}
scene.add( new THREE.DirectionalLight( 0xffffff, 1 ) );
scene.add( new THREE.AmbientLight( 0x404040 ) );

// Post-processing
const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );

const outlinePass = new OutlinePass(
  new THREE.Vector2( window.innerWidth, window.innerHeight ),
  scene, camera
);
outlinePass.edgeStrength = 5;
outlinePass.edgeGlow = 0.5;
outlinePass.edgeThickness = 2;
outlinePass.visibleEdgeColor.set( 0x00ff00 );
composer.addPass( outlinePass );
composer.addPass( new OutputPass() );

// Raycaster for hover detection
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

window.addEventListener( 'mousemove', ( event ) => {
  mouse.x = ( event.clientX / window.innerWidth ) * 2 - 1;
  mouse.y = -( event.clientY / window.innerHeight ) * 2 + 1;
  raycaster.setFromCamera( mouse, camera );
  const intersects = raycaster.intersectObjects( objects );
  outlinePass.selectedObjects = intersects.length > 0 ? [ intersects[ 0 ].object ] : [];
} );

function animate() {
  requestAnimationFrame( animate );
  composer.render();
}
animate();
```

---

## Example 4: Custom ShaderPass (Vignette)

Writing a custom post-processing effect with ShaderPass.

```js
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const vignetteShader = {
  uniforms: {
    tDiffuse: { value: null },      // ALWAYS include -- receives read buffer
    darkness: { value: 1.5 },
    offset: { value: 1.0 }
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
    }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform float darkness;
    uniform float offset;
    varying vec2 vUv;
    void main() {
      vec4 texel = texture2D( tDiffuse, vUv );
      vec2 uv = ( vUv - vec2( 0.5 ) ) * vec2( offset );
      float vignette = clamp( 1.0 - dot( uv, uv ), 0.0, 1.0 );
      gl_FragColor = vec4( texel.rgb * mix( 1.0, vignette, darkness ), texel.a );
    }
  `
};

// Setup (scene, camera, renderer assumed created)
const composer = new EffectComposer( renderer );
composer.addPass( new RenderPass( scene, camera ) );

const vignettePass = new ShaderPass( vignetteShader );
composer.addPass( vignettePass );

composer.addPass( new OutputPass() );

// Update uniforms at runtime
vignettePass.uniforms.darkness.value = 2.0;

function animate() {
  requestAnimationFrame( animate );
  composer.render();
}
animate();
```

---

## Example 5: pmndrs/postprocessing with Effect Merging

Using the pmndrs library for better performance with multiple effects.

```js
import * as THREE from 'three';
import { EffectComposer, EffectPass, RenderPass, BloomEffect,
         SMAAEffect, VignetteEffect, SMAAPreset } from 'postprocessing';

const renderer = new THREE.WebGLRenderer( {
  powerPreference: 'high-performance',
  antialias: false,     // SMAA handles AA
  stencil: false,
  depth: false
} );
renderer.setSize( window.innerWidth, window.innerHeight );

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera( 60, window.innerWidth / window.innerHeight, 0.1, 100 );

// pmndrs EffectComposer (NOT Three.js built-in)
const composer = new EffectComposer( renderer, {
  multisampling: 0 // disable MSAA since we use SMAA
} );

// RenderPass (pmndrs version)
composer.addPass( new RenderPass( scene, camera ) );

// EffectPass merges ALL effects into ONE shader pass
composer.addPass( new EffectPass( camera,
  new BloomEffect( {
    luminanceThreshold: 0.8,
    luminanceSmoothing: 0.075,
    mipmapBlur: true,
    intensity: 1.5
  } ),
  new VignetteEffect( {
    darkness: 0.5
  } ),
  new SMAAEffect( {
    preset: SMAAPreset.HIGH
  } )
) );

// No OutputPass needed -- pmndrs handles tone mapping internally

window.addEventListener( 'resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  const width = window.innerWidth;
  const height = window.innerHeight;
  renderer.setSize( width, height );
  composer.setSize( width, height );
} );

function animate() {
  requestAnimationFrame( animate );
  composer.render();
}
animate();
```
