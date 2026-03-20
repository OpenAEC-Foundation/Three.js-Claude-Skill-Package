# threejs-impl-animation — Examples

> Working code examples for the Three.js Animation System (r160+).

---

## Example 1: Load and Play All GLTF Animations

The most common animation workflow: load a GLTF model and play all embedded clips.

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 1.5, 3);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const clock = new THREE.Clock();
let mixer;

const loader = new GLTFLoader();
loader.load('character.glb', (gltf) => {
  scene.add(gltf.scene);

  mixer = new THREE.AnimationMixer(gltf.scene);

  // Play every animation clip from the GLTF file
  gltf.animations.forEach((clip) => {
    mixer.clipAction(clip).play();
  });
});

function animate() {
  const delta = clock.getDelta();
  if (mixer) mixer.update(delta);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

---

## Example 2: Play a Single Named Clip

Select and play a specific animation by name from the GLTF file.

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const clock = new THREE.Clock();
let mixer;

const loader = new GLTFLoader();
loader.load('character.glb', (gltf) => {
  scene.add(gltf.scene);

  mixer = new THREE.AnimationMixer(gltf.scene);

  // Find a specific clip by name
  const idleClip = THREE.AnimationClip.findByName(gltf.animations, 'Idle');
  if (idleClip) {
    const action = mixer.clipAction(idleClip);
    action.play();
  }
});

function animate() {
  const delta = clock.getDelta();
  if (mixer) mixer.update(delta);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

---

## Example 3: Crossfade Character State Machine

Smooth transitions between idle, walk, and run states using crossfade.

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const clock = new THREE.Clock();
let mixer;
const actions = {};
let currentAction;

const loader = new GLTFLoader();
loader.load('character.glb', (gltf) => {
  scene.add(gltf.scene);

  mixer = new THREE.AnimationMixer(gltf.scene);

  // Cache all actions by clip name
  gltf.animations.forEach((clip) => {
    actions[clip.name] = mixer.clipAction(clip);
  });

  // Start with idle
  currentAction = actions['Idle'];
  currentAction.play();
});

function switchAction(toName, duration = 0.5) {
  if (!actions[toName] || actions[toName] === currentAction) return;

  const toAction = actions[toName];

  // ALWAYS reset the incoming action before crossfading
  toAction.reset();
  toAction.setEffectiveTimeScale(1);
  toAction.setEffectiveWeight(1);
  toAction.crossFadeFrom(currentAction, duration, true);
  toAction.play();

  currentAction = toAction;
}

// Usage: respond to input
document.addEventListener('keydown', (event) => {
  switch (event.key) {
    case 'w': switchAction('Walk'); break;
    case 'r': switchAction('Run'); break;
    case 'i': switchAction('Idle'); break;
  }
});

function animate() {
  const delta = clock.getDelta();
  if (mixer) mixer.update(delta);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

---

## Example 4: Additive Animation Blending

Layer an additive animation (e.g., breathing or damage reaction) on top of a base animation.

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const clock = new THREE.Clock();
let mixer;

const loader = new GLTFLoader();
loader.load('character.glb', (gltf) => {
  scene.add(gltf.scene);

  mixer = new THREE.AnimationMixer(gltf.scene);

  // Base animation: normal blending
  const idleClip = THREE.AnimationClip.findByName(gltf.animations, 'Idle');
  const baseAction = mixer.clipAction(idleClip);
  baseAction.play();

  // Additive animation: layered on top
  const breatheClip = THREE.AnimationClip.findByName(gltf.animations, 'Breathe');
  const additiveAction = mixer.clipAction(
    breatheClip,
    undefined,
    THREE.AdditiveAnimationBlendMode
  );
  additiveAction.play();
  additiveAction.setEffectiveWeight(0.5); // control blend strength
});

function animate() {
  const delta = clock.getDelta();
  if (mixer) mixer.update(delta);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

---

## Example 5: Play-Once Animation with Finished Event

Play an animation exactly once, clamp at the last frame, and detect completion.

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const clock = new THREE.Clock();
let mixer;

const loader = new GLTFLoader();
loader.load('door.glb', (gltf) => {
  scene.add(gltf.scene);

  mixer = new THREE.AnimationMixer(gltf.scene);

  const openClip = THREE.AnimationClip.findByName(gltf.animations, 'DoorOpen');
  const action = mixer.clipAction(openClip);

  // Configure play-once behavior
  action.setLoop(THREE.LoopOnce, 1);
  action.clampWhenFinished = true; // ALWAYS set for LoopOnce

  // Listen for completion
  mixer.addEventListener('finished', (event) => {
    console.log('Animation finished:', event.action.getClip().name);
  });

  action.play();
});

function animate() {
  const delta = clock.getDelta();
  if (mixer) mixer.update(delta);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

---

## Example 6: Programmatic KeyframeTrack Animation

Create an animation entirely in code without loading a GLTF file.

```js
import * as THREE from 'three';

const clock = new THREE.Clock();

// Create a simple cube
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const cube = new THREE.Mesh(geometry, material);
cube.name = 'MyCube';
scene.add(cube);

// Define keyframe tracks
const positionTrack = new THREE.VectorKeyframeTrack(
  'MyCube.position',                 // PropertyBinding path
  [0, 1, 2],                         // times in seconds
  [0, 0, 0, 0, 2, 0, 0, 0, 0]       // values: [x,y,z] at each time
);

const rotationTrack = new THREE.QuaternionKeyframeTrack(
  'MyCube.quaternion',
  [0, 1, 2],
  [
    0, 0, 0, 1,                       // identity at t=0
    0, 0.707, 0, 0.707,              // 90deg Y at t=1
    0, 0, 0, 1                        // identity at t=2
  ]
);

const opacityTrack = new THREE.NumberKeyframeTrack(
  'MyCube.material.opacity',
  [0, 1, 2],
  [1, 0.3, 1]
);

// Create clip from tracks
const clip = new THREE.AnimationClip('BounceAndSpin', 2, [
  positionTrack,
  rotationTrack,
  opacityTrack
]);

// Set up mixer and play
const mixer = new THREE.AnimationMixer(cube);
const action = mixer.clipAction(clip);
action.play();

// IMPORTANT: enable transparency for opacity animation
material.transparent = true;

function animate() {
  const delta = clock.getDelta();
  mixer.update(delta);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```

---

## Example 7: Morph Target Animation

Animate morph targets (blend shapes) loaded from GLTF.

```js
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

const clock = new THREE.Clock();
let mixer;

const loader = new GLTFLoader();
loader.load('face.glb', (gltf) => {
  scene.add(gltf.scene);

  mixer = new THREE.AnimationMixer(gltf.scene);

  // GLTF morph target animations are stored as regular clips
  // They target mesh.morphTargetInfluences[index]
  const smileClip = THREE.AnimationClip.findByName(gltf.animations, 'Smile');
  if (smileClip) {
    const action = mixer.clipAction(smileClip);
    action.play();
  }
});

// Manual morph target control (alternative to clip-based)
function manualMorphControl(mesh, elapsed) {
  if (mesh.morphTargetInfluences) {
    mesh.morphTargetInfluences[0] = Math.sin(elapsed) * 0.5 + 0.5;
  }
}

function animate() {
  const delta = clock.getDelta();
  if (mixer) mixer.update(delta);
  renderer.render(scene, camera);
}
renderer.setAnimationLoop(animate);
```
