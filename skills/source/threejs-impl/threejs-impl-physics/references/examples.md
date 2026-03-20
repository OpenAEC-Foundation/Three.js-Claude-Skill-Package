# threejs-impl-physics — Examples

## Example 1: cannon-es — Falling Boxes Scene

Complete scene with a static ground plane and dynamic falling boxes.

```js
import * as THREE from 'three';
import * as CANNON from 'cannon-es';

// Three.js setup
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 10, 20);
camera.lookAt(0, 0, 0);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

// Lighting
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 7.5);
dirLight.castShadow = true;
scene.add(dirLight);
scene.add(new THREE.AmbientLight(0x404040, 0.5));

// Physics world
const world = new CANNON.World();
world.gravity.set(0, -9.82, 0);
world.broadphase = new CANNON.SAPBroadphase(world);
world.allowSleep = true;

// Contact material
const defaultMat = new CANNON.Material('default');
const defaultContact = new CANNON.ContactMaterial(defaultMat, defaultMat, {
  friction: 0.3,
  restitution: 0.4,
});
world.addContactMaterial(defaultContact);

// Ground
const groundBody = new CANNON.Body({
  mass: 0, // static
  shape: new CANNON.Plane(),
  material: defaultMat,
});
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

const groundMesh = new THREE.Mesh(
  new THREE.PlaneGeometry(50, 50),
  new THREE.MeshStandardMaterial({ color: 0x808080 }),
);
groundMesh.rotation.x = -Math.PI / 2;
groundMesh.receiveShadow = true;
scene.add(groundMesh);

// Dynamic boxes
const pairs = [];
for (let i = 0; i < 20; i++) {
  const size = 0.5 + Math.random() * 0.5;
  const halfSize = size / 2;

  const body = new CANNON.Body({
    mass: 1,
    position: new CANNON.Vec3(
      (Math.random() - 0.5) * 4,
      5 + i * 2,
      (Math.random() - 0.5) * 4,
    ),
    shape: new CANNON.Box(new CANNON.Vec3(halfSize, halfSize, halfSize)),
    material: defaultMat,
  });
  world.addBody(body);

  const mesh = new THREE.Mesh(
    new THREE.BoxGeometry(size, size, size),
    new THREE.MeshStandardMaterial({ color: Math.random() * 0xffffff }),
  );
  mesh.castShadow = true;
  scene.add(mesh);

  pairs.push({ mesh, body });
}

// Animation loop
const clock = new THREE.Clock();
function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();

  world.step(1 / 60, delta, 3);
  for (const { mesh, body } of pairs) {
    mesh.position.copy(body.position);
    mesh.quaternion.copy(body.quaternion);
  }

  renderer.render(scene, camera);
}
animate();
```

---

## Example 2: Rapier — Sphere Drop with Ray Casting

Complete scene using Rapier WASM with ray-based ground detection.

```js
import * as THREE from 'three';
import RAPIER from '@dimforge/rapier3d-compat';

async function main() {
  await RAPIER.init();

  // Three.js setup
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.set(0, 8, 15);
  camera.lookAt(0, 0, 0);
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);
  scene.add(new THREE.DirectionalLight(0xffffff, 1));
  scene.add(new THREE.AmbientLight(0x404040, 0.5));

  // Physics world
  const gravity = { x: 0.0, y: -9.81, z: 0.0 };
  const world = new RAPIER.World(gravity);

  // Ground (fixed body)
  const groundDesc = RAPIER.RigidBodyDesc.fixed().setTranslation(0, -0.5, 0);
  const groundBody = world.createRigidBody(groundDesc);
  const groundCollider = RAPIER.ColliderDesc.cuboid(25, 0.5, 25)
    .setRestitution(0.3)
    .setFriction(0.8);
  world.createCollider(groundCollider, groundBody);

  const groundMesh = new THREE.Mesh(
    new THREE.BoxGeometry(50, 1, 50),
    new THREE.MeshStandardMaterial({ color: 0x808080 }),
  );
  groundMesh.position.set(0, -0.5, 0);
  scene.add(groundMesh);

  // Spheres
  const pairs = [];
  for (let i = 0; i < 50; i++) {
    const radius = 0.3 + Math.random() * 0.3;
    const bodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(
        (Math.random() - 0.5) * 6,
        5 + i * 1.5,
        (Math.random() - 0.5) * 6,
      )
      .setCcdEnabled(true);
    const rigidBody = world.createRigidBody(bodyDesc);
    const colliderDesc = RAPIER.ColliderDesc.ball(radius)
      .setRestitution(0.7)
      .setFriction(0.5);
    world.createCollider(colliderDesc, rigidBody);

    const mesh = new THREE.Mesh(
      new THREE.SphereGeometry(radius, 16, 16),
      new THREE.MeshStandardMaterial({ color: Math.random() * 0xffffff }),
    );
    scene.add(mesh);
    pairs.push({ mesh, rigidBody });
  }

  // Raycaster marker
  const markerGeo = new THREE.SphereGeometry(0.1, 8, 8);
  const markerMat = new THREE.MeshBasicMaterial({ color: 0xff0000 });
  const marker = new THREE.Mesh(markerGeo, markerMat);
  scene.add(marker);

  // Ray cast on click
  window.addEventListener('click', () => {
    const ray = new RAPIER.Ray({ x: 0, y: 20, z: 0 }, { x: 0, y: -1, z: 0 });
    const hit = world.castRay(ray, 50, true);
    if (hit) {
      const point = ray.pointAt(hit.timeOfImpact);
      marker.position.set(point.x, point.y, point.z);
    }
  });

  // Animation loop
  function animate() {
    requestAnimationFrame(animate);
    world.step();
    for (const { mesh, rigidBody } of pairs) {
      const pos = rigidBody.translation();
      const rot = rigidBody.rotation();
      mesh.position.set(pos.x, pos.y, pos.z);
      mesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
    }
    renderer.render(scene, camera);
  }
  animate();
}
main();
```

---

## Example 3: cannon-es — Hinge Constraint (Door)

A door attached to a frame with a hinge constraint.

```js
import * as THREE from 'three';
import * as CANNON from 'cannon-es';

const world = new CANNON.World();
world.gravity.set(0, -9.82, 0);
world.broadphase = new CANNON.SAPBroadphase(world);

// Frame (static)
const frameBody = new CANNON.Body({ mass: 0 });
frameBody.addShape(new CANNON.Box(new CANNON.Vec3(0.1, 1.5, 0.1)));
frameBody.position.set(-1, 1.5, 0);
world.addBody(frameBody);

// Door (dynamic)
const doorBody = new CANNON.Body({
  mass: 5,
  shape: new CANNON.Box(new CANNON.Vec3(1, 1.5, 0.05)),
  position: new CANNON.Vec3(0, 1.5, 0),
});
world.addBody(doorBody);

// Hinge constraint
const hinge = new CANNON.HingeConstraint(frameBody, doorBody, {
  pivotA: new CANNON.Vec3(0.1, 0, 0),
  axisA: new CANNON.Vec3(0, 1, 0),
  pivotB: new CANNON.Vec3(-1, 0, 0),
  axisB: new CANNON.Vec3(0, 1, 0),
});
world.addConstraint(hinge);

// Three.js meshes (create scene, camera, renderer as in Example 1)
const doorMesh = new THREE.Mesh(
  new THREE.BoxGeometry(2, 3, 0.1),
  new THREE.MeshStandardMaterial({ color: 0x8b4513 }),
);
scene.add(doorMesh);

// Apply force to open door
doorBody.applyImpulse(new CANNON.Vec3(0, 0, 10), new CANNON.Vec3(1, 0, 0));

// Sync in animation loop
function updatePhysics(delta) {
  world.step(1 / 60, delta, 3);
  doorMesh.position.copy(doorBody.position);
  doorMesh.quaternion.copy(doorBody.quaternion);
}
```

---

## Example 4: Rapier — Debug Rendering

Visualize Rapier collision shapes as wireframe overlays.

```js
import * as THREE from 'three';
import RAPIER from '@dimforge/rapier3d-compat';

async function main() {
  await RAPIER.init();
  const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 });

  // ... create bodies and colliders ...

  // Debug line mesh
  let debugMesh = null;

  function updateDebugRender() {
    const { vertices, colors } = world.debugRender();

    if (debugMesh) {
      scene.remove(debugMesh);
      debugMesh.geometry.dispose();
      debugMesh.material.dispose();
    }

    const geometry = new THREE.BufferGeometry();
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
    geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 4));
    const material = new THREE.LineBasicMaterial({ vertexColors: true, depthTest: false });
    debugMesh = new THREE.LineSegments(geometry, material);
    debugMesh.renderOrder = 999;
    scene.add(debugMesh);
  }

  function animate() {
    requestAnimationFrame(animate);
    world.step();
    updateDebugRender();
    renderer.render(scene, camera);
  }
  animate();
}
main();
```

---

## Example 5: React Three Fiber with @react-three/rapier

Declarative physics with R3F.

```jsx
import { Canvas } from '@react-three/fiber';
import { Physics, RigidBody, CuboidCollider } from '@react-three/rapier';

function Ground() {
  return (
    <RigidBody type="fixed">
      <CuboidCollider args={[25, 0.5, 25]} position={[0, -0.5, 0]} />
      <mesh position={[0, -0.5, 0]}>
        <boxGeometry args={[50, 1, 50]} />
        <meshStandardMaterial color="#808080" />
      </mesh>
    </RigidBody>
  );
}

function FallingBox({ position }) {
  return (
    <RigidBody restitution={0.5} friction={0.7}>
      <mesh position={position} castShadow>
        <boxGeometry args={[1, 1, 1]} />
        <meshStandardMaterial color="orange" />
      </mesh>
    </RigidBody>
  );
}

export default function App() {
  return (
    <Canvas shadows camera={{ position: [0, 10, 20], fov: 75 }}>
      <ambientLight intensity={0.5} />
      <directionalLight position={[5, 10, 5]} castShadow />
      <Physics gravity={[0, -9.81, 0]} debug>
        <Ground />
        {Array.from({ length: 10 }, (_, i) => (
          <FallingBox key={i} position={[(Math.random() - 0.5) * 4, 5 + i * 2, 0]} />
        ))}
      </Physics>
    </Canvas>
  );
}
```
