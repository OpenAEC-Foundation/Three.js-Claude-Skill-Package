# threejs-impl-xr — Examples

## Example 1: Complete VR Scene with Controllers

A fully functional VR scene with controller models, laser pointers, and object interaction.

```javascript
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';
import { XRControllerModelFactory } from 'three/addons/webxr/XRControllerModelFactory.js';

// Scene setup
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x505050);
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 100);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.xr.enabled = true;
renderer.xr.setReferenceSpaceType('local-floor');
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

// Lighting
scene.add(new THREE.HemisphereLight(0x808080, 0x606060));
const directionalLight = new THREE.DirectionalLight(0xffffff);
directionalLight.position.set(1, 1, 1);
scene.add(directionalLight);

// Floor
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(10, 10),
  new THREE.MeshStandardMaterial({ color: 0x222222 })
);
floor.rotation.x = -Math.PI / 2;
scene.add(floor);

// Interactive objects
const group = new THREE.Group();
scene.add(group);
for (let i = 0; i < 20; i++) {
  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(0.15, 0.15, 0.15),
    new THREE.MeshStandardMaterial({ color: Math.random() * 0xffffff })
  );
  mesh.position.set(
    Math.random() * 4 - 2,
    Math.random() * 2 + 0.5,
    Math.random() * 4 - 2
  );
  group.add(mesh);
}

// Controllers
const controllerModelFactory = new XRControllerModelFactory();
const raycaster = new THREE.Raycaster();
const tempMatrix = new THREE.Matrix4();

function setupController(index) {
  const controller = renderer.xr.getController(index);
  controller.addEventListener('selectstart', () => {
    tempMatrix.identity().extractRotation(controller.matrixWorld);
    raycaster.ray.origin.setFromMatrixPosition(controller.matrixWorld);
    raycaster.ray.direction.set(0, 0, -1).applyMatrix4(tempMatrix);
    const intersects = raycaster.intersectObjects(group.children, false);
    if (intersects.length > 0) {
      controller.userData.selected = intersects[0].object;
      controller.attach(intersects[0].object);
    }
  });
  controller.addEventListener('selectend', () => {
    if (controller.userData.selected) {
      group.attach(controller.userData.selected);
      controller.userData.selected = null;
    }
  });
  scene.add(controller);

  // Laser pointer
  const line = new THREE.Line(
    new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(0, 0, 0),
      new THREE.Vector3(0, 0, -5)
    ]),
    new THREE.LineBasicMaterial({ color: 0xffffff })
  );
  controller.add(line);

  // Controller model on grip space
  const grip = renderer.xr.getControllerGrip(index);
  grip.add(controllerModelFactory.createControllerModel(grip));
  scene.add(grip);

  return controller;
}

setupController(0);
setupController(1);

// MUST use setAnimationLoop
renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
});
```

---

## Example 2: AR Hit Test with Object Placement

Place objects on real-world surfaces using AR hit testing.

```javascript
import * as THREE from 'three';
import { ARButton } from 'three/addons/webxr/ARButton.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.01, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(ARButton.createButton(renderer, {
  requiredFeatures: ['hit-test']
}));

// Light
scene.add(new THREE.HemisphereLight(0xffffff, 0xbbbbff, 1));

// Reticle (placement indicator)
const reticle = new THREE.Mesh(
  new THREE.RingGeometry(0.15, 0.2, 32).rotateX(-Math.PI / 2),
  new THREE.MeshBasicMaterial()
);
reticle.matrixAutoUpdate = false;
reticle.visible = false;
scene.add(reticle);

// Place object on tap
const controller = renderer.xr.getController(0);
controller.addEventListener('select', () => {
  if (reticle.visible) {
    const mesh = new THREE.Mesh(
      new THREE.CylinderGeometry(0.05, 0.05, 0.2, 32),
      new THREE.MeshStandardMaterial({ color: 0x00ff88 })
    );
    mesh.position.setFromMatrixPosition(reticle.matrix);
    scene.add(mesh);
  }
});
scene.add(controller);

// Hit testing
let hitTestSource = null;
let hitTestSourceRequested = false;

renderer.setAnimationLoop((time, frame) => {
  if (frame) {
    const session = renderer.xr.getSession();
    const referenceSpace = renderer.xr.getReferenceSpace();

    if (!hitTestSourceRequested) {
      session.requestReferenceSpace('viewer').then((viewerSpace) => {
        session.requestHitTestSource({ space: viewerSpace }).then((source) => {
          hitTestSource = source;
        });
      });
      hitTestSourceRequested = true;

      session.addEventListener('end', () => {
        hitTestSourceRequested = false;
        hitTestSource = null;
      });
    }

    if (hitTestSource) {
      const results = frame.getHitTestResults(hitTestSource);
      if (results.length > 0) {
        const pose = results[0].getPose(referenceSpace);
        reticle.visible = true;
        reticle.matrix.fromArray(pose.transform.matrix);
      } else {
        reticle.visible = false;
      }
    }
  }

  renderer.render(scene, camera);
});
```

---

## Example 3: VR Hand Tracking with Pinch Interaction

```javascript
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';
import { XRHandModelFactory } from 'three/addons/webxr/XRHandModelFactory.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 100);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
renderer.xr.setReferenceSpaceType('local-floor');

// Request hand-tracking feature
renderer.xr.setSessionInit({
  optionalFeatures: ['hand-tracking']
});

document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

scene.add(new THREE.HemisphereLight(0x808080, 0x606060));

// Hand setup
const handFactory = new XRHandModelFactory();
const spheres = [];

function setupHand(index) {
  const hand = renderer.xr.getHand(index);
  hand.add(handFactory.createHandModel(hand, 'mesh'));
  scene.add(hand);

  hand.addEventListener('pinchstart', () => {
    const sphere = new THREE.Mesh(
      new THREE.SphereGeometry(0.02, 16, 16),
      new THREE.MeshStandardMaterial({ color: 0xff4444 })
    );
    // Place sphere at index fingertip (joint 9)
    const indexTip = hand.joints['index-finger-tip'];
    if (indexTip) {
      sphere.position.copy(indexTip.position);
      scene.add(sphere);
      spheres.push(sphere);
    }
  });

  return hand;
}

setupHand(0);
setupHand(1);

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
});
```

---

## Example 4: VR Teleportation with Camera Rig

```javascript
import * as THREE from 'three';
import { VRButton } from 'three/addons/webxr/VRButton.js';
import { XRControllerModelFactory } from 'three/addons/webxr/XRControllerModelFactory.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 100);

// Camera rig — ALWAYS move the rig, NEVER the camera directly
const cameraRig = new THREE.Group();
cameraRig.add(camera);
scene.add(cameraRig);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
renderer.xr.setReferenceSpaceType('local-floor');
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer));

// Floor
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(50, 50),
  new THREE.MeshStandardMaterial({ color: 0x333333 })
);
floor.rotation.x = -Math.PI / 2;
floor.receiveShadow = true;
scene.add(floor);

// Teleport marker
const marker = new THREE.Mesh(
  new THREE.RingGeometry(0.2, 0.3, 32).rotateX(-Math.PI / 2),
  new THREE.MeshBasicMaterial({ color: 0x00ff00 })
);
marker.visible = false;
scene.add(marker);

// Teleport arc (parabolic)
const arcGeometry = new THREE.BufferGeometry();
const arcMaterial = new THREE.LineBasicMaterial({ color: 0x00ff00 });
const arcLine = new THREE.Line(arcGeometry, arcMaterial);
arcLine.visible = false;
scene.add(arcLine);

const raycaster = new THREE.Raycaster();
const tempMatrix = new THREE.Matrix4();
let teleportTarget = null;

const controller = renderer.xr.getController(0);

controller.addEventListener('selectstart', () => {
  // Show teleport arc while holding trigger
  arcLine.visible = true;
});

controller.addEventListener('selectend', () => {
  arcLine.visible = false;
  marker.visible = false;
  if (teleportTarget) {
    cameraRig.position.copy(teleportTarget);
    cameraRig.position.y = 0;
    teleportTarget = null;
  }
});

// Add controller to camera rig so it moves with teleportation
cameraRig.add(controller);

const grip = renderer.xr.getControllerGrip(0);
const factory = new XRControllerModelFactory();
grip.add(factory.createControllerModel(grip));
cameraRig.add(grip);

scene.add(new THREE.HemisphereLight(0xffffff, 0x444444));

renderer.setAnimationLoop(() => {
  // Update teleport target while trigger held
  if (arcLine.visible) {
    tempMatrix.identity().extractRotation(controller.matrixWorld);
    raycaster.ray.origin.setFromMatrixPosition(controller.matrixWorld);
    raycaster.ray.direction.set(0, 0, -1).applyMatrix4(tempMatrix);

    const intersects = raycaster.intersectObject(floor);
    if (intersects.length > 0) {
      teleportTarget = intersects[0].point;
      marker.position.copy(teleportTarget);
      marker.visible = true;
    } else {
      marker.visible = false;
      teleportTarget = null;
    }
  }

  renderer.render(scene, camera);
});
```

---

## Example 5: AR with Estimated Lighting

```javascript
import * as THREE from 'three';
import { ARButton } from 'three/addons/webxr/ARButton.js';
import { XREstimatedLight } from 'three/addons/webxr/XREstimatedLight.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.01, 20);

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(ARButton.createButton(renderer, {
  optionalFeatures: ['light-estimation']
}));

// Default lighting (used when estimation is unavailable)
const defaultLight = new THREE.HemisphereLight(0xffffff, 0xbbbbff, 1);
scene.add(defaultLight);
let defaultEnvironment = null;

// XR estimated light
const xrLight = new XREstimatedLight(renderer);

xrLight.addEventListener('estimationstart', () => {
  scene.add(xrLight);
  scene.environment = xrLight.environment;
  scene.remove(defaultLight);
});

xrLight.addEventListener('estimationend', () => {
  scene.remove(xrLight);
  scene.environment = defaultEnvironment;
  scene.add(defaultLight);
});

// Place a PBR sphere that reacts to real-world lighting
const sphere = new THREE.Mesh(
  new THREE.SphereGeometry(0.1, 32, 32),
  new THREE.MeshStandardMaterial({ metalness: 0.8, roughness: 0.2 })
);
sphere.position.set(0, 0.1, -0.5);
scene.add(sphere);

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
});
```
