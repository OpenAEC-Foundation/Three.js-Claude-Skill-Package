# threejs-impl-physics — Anti-Patterns

## Anti-Pattern 1: Forgetting `world.step()` in the Animation Loop

**Wrong:**
```js
function animate() {
  requestAnimationFrame(animate);
  // world.step() is missing!
  mesh.position.copy(body.position);
  mesh.quaternion.copy(body.quaternion);
  renderer.render(scene, camera);
}
```

**Why it fails:** Without calling `world.step()`, the physics simulation NEVER advances. Bodies remain frozen at their initial positions regardless of gravity or applied forces.

**Correct:**
```js
function animate() {
  requestAnimationFrame(animate);
  const delta = clock.getDelta();
  world.step(1 / 60, delta, 3); // ALWAYS step the world
  mesh.position.copy(body.position);
  mesh.quaternion.copy(body.quaternion);
  renderer.render(scene, camera);
}
```

---

## Anti-Pattern 2: Using `.copy()` with Rapier Return Values

**Wrong:**
```js
// Rapier sync
const pos = rigidBody.translation();
mesh.position.copy(pos); // FAILS — pos is {x, y, z}, not a Vector3
```

**Why it fails:** Rapier `translation()` and `rotation()` return plain JavaScript objects `{x, y, z}` and `{x, y, z, w}`. Three.js `.copy()` expects objects with a `.copy()` method or matching class instances. This may silently fail or throw errors depending on the Three.js version.

**Correct:**
```js
const pos = rigidBody.translation();
const rot = rigidBody.rotation();
mesh.position.set(pos.x, pos.y, pos.z);
mesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
```

**Note:** cannon-es `Vec3` and `Quaternion` ARE compatible with Three.js `.copy()` — this anti-pattern applies ONLY to Rapier.

---

## Anti-Pattern 3: Using Rapier APIs Before `await RAPIER.init()`

**Wrong:**
```js
import RAPIER from '@dimforge/rapier3d-compat';

// Calling immediately without init
const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 }); // THROWS: WASM not loaded
```

**Why it fails:** Rapier is a Rust library compiled to WASM. The WASM binary MUST be loaded and initialized before ANY Rapier class is available. Accessing constructors before `init()` resolves causes runtime errors.

**Correct:**
```js
import RAPIER from '@dimforge/rapier3d-compat';

async function main() {
  await RAPIER.init(); // MUST await first
  const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 });
  // Now safe to use all Rapier APIs
}
main();
```

---

## Anti-Pattern 4: Using Trimesh on Dynamic Bodies

**Wrong:**
```js
// cannon-es
const body = new CANNON.Body({
  mass: 5, // dynamic
  shape: new CANNON.Trimesh(vertices, indices), // BROKEN — Trimesh is static only
});

// Rapier
const desc = RAPIER.RigidBodyDesc.dynamic();
const body = world.createRigidBody(desc);
world.createCollider(RAPIER.ColliderDesc.trimesh(vertices, indices), body); // BROKEN
```

**Why it fails:** Triangle mesh collision detection in both cannon-es and Rapier is designed ONLY for static geometry. Dynamic trimesh colliders produce incorrect collision responses, tunneling, and undefined behavior. The engines do NOT support mesh-mesh dynamic collision.

**Correct:**
```js
// Use convex hull for dynamic bodies
// cannon-es
const body = new CANNON.Body({
  mass: 5,
  shape: new CANNON.ConvexPolyhedron({ vertices: convexVerts, faces: convexFaces }),
});

// Rapier
const desc = RAPIER.ColliderDesc.convexHull(new Float32Array(flatVertices));
```

ALWAYS decompose concave dynamic geometry into convex parts using libraries like `v-hacd` or manual decomposition.

---

## Anti-Pattern 5: Wrong Half-Extents for Box Shapes

**Wrong:**
```js
// Three.js box is 2x2x2 (full size)
const mesh = new THREE.Mesh(new THREE.BoxGeometry(2, 2, 2));

// cannon-es — passing full size instead of half-extents
const shape = new CANNON.Box(new CANNON.Vec3(2, 2, 2)); // WRONG — this creates a 4x4x4 box

// Rapier — same mistake
const collider = RAPIER.ColliderDesc.cuboid(2, 2, 2); // WRONG — 4x4x4
```

**Why it fails:** Both cannon-es `Box` and Rapier `cuboid` use HALF-EXTENTS, not full dimensions. A `BoxGeometry(2, 2, 2)` has half-extents of `(1, 1, 1)`. Passing full sizes doubles the physics collider, causing objects to collide in empty space.

**Correct:**
```js
const width = 2, height = 2, depth = 2;
const mesh = new THREE.Mesh(new THREE.BoxGeometry(width, height, depth));

// cannon-es — half-extents
const shape = new CANNON.Box(new CANNON.Vec3(width / 2, height / 2, depth / 2));

// Rapier — half-extents
const collider = RAPIER.ColliderDesc.cuboid(width / 2, height / 2, depth / 2);
```

---

## Anti-Pattern 6: Variable Timestep Physics

**Wrong:**
```js
function animate() {
  const delta = clock.getDelta();
  world.step(delta); // WRONG — variable timestep causes non-deterministic simulation
}
```

**Why it fails:** Using `delta` directly as the timestep produces different simulation results at different frame rates. Objects may tunnel through walls at low FPS or behave differently on fast vs slow machines.

**Correct (cannon-es):**
```js
function animate() {
  const delta = clock.getDelta();
  world.step(1 / 60, delta, 3); // fixed step with interpolation
}
```

**Correct (Rapier):**
```js
// Rapier uses a fixed internal timestep by default (1/60)
world.step(); // ALWAYS call without arguments for consistent behavior
```

ALWAYS use a fixed timestep. cannon-es achieves this via the three-argument `world.step()`. Rapier uses a fixed timestep internally by default.

---

## Anti-Pattern 7: Not Assigning Materials to Bodies

**Wrong:**
```js
const groundMat = new CANNON.Material('ground');
const ballMat = new CANNON.Material('ball');
const contact = new CANNON.ContactMaterial(groundMat, ballMat, {
  friction: 0.4,
  restitution: 0.6,
});
world.addContactMaterial(contact);

// Materials never assigned to bodies — ContactMaterial has NO effect
const ground = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
const ball = new CANNON.Body({ mass: 1, shape: new CANNON.Sphere(1) });
```

**Why it fails:** Creating a `ContactMaterial` only defines rules for interactions between two `Material` instances. Unless the `Material` is explicitly assigned to each `Body` via the `.material` property, the engine uses default material settings and the `ContactMaterial` is ignored.

**Correct:**
```js
ground.material = groundMat;
ball.material = ballMat;
```

---

## Anti-Pattern 8: Not Disposing Rapier WASM Resources

**Wrong:**
```js
// Switching scenes, removing physics
function cleanupPhysics() {
  // Just setting world to null — WASM memory leaks
  world = null;
}
```

**Why it fails:** Rapier objects live in WASM linear memory, which is NOT managed by JavaScript's garbage collector. Failing to call `.free()` on the world and event queue causes permanent memory leaks that persist until page reload.

**Correct:**
```js
function cleanupPhysics() {
  if (eventQueue) {
    eventQueue.free();
    eventQueue = null;
  }
  if (world) {
    world.free();
    world = null;
  }
}
```

ALWAYS call `.free()` on Rapier `World` and `EventQueue` objects when they are no longer needed.

---

## Anti-Pattern 9: Ignoring Sleep for Static Scenes

**Wrong:**
```js
const world = new CANNON.World();
world.gravity.set(0, -9.82, 0);
// allowSleep defaults to false — ALL bodies simulated every frame
```

**Why it fails:** Without sleep, every body in the world is simulated every physics step, even if it has come to rest. In a scene with hundreds of stacked boxes that have settled, this wastes the majority of the physics budget on bodies that are not moving.

**Correct:**
```js
const world = new CANNON.World();
world.gravity.set(0, -9.82, 0);
world.allowSleep = true; // bodies at rest stop being simulated
```

ALWAYS enable sleep in cannon-es. Rapier enables sleep by default.

---

## Anti-Pattern 10: Creating Physics Ground with Wrong Orientation

**Wrong (cannon-es):**
```js
const ground = new CANNON.Body({
  mass: 0,
  shape: new CANNON.Plane(),
  // No rotation — Plane faces +Z by default, not +Y
});
world.addBody(ground);
```

**Why it fails:** `CANNON.Plane` faces the local +Z axis by default. Without rotating it, the ground plane is vertical (like a wall), and objects fall through the intended ground position.

**Correct:**
```js
const ground = new CANNON.Body({
  mass: 0,
  shape: new CANNON.Plane(),
});
ground.quaternion.setFromEuler(-Math.PI / 2, 0, 0); // rotate to face +Y (up)
world.addBody(ground);
```

ALWAYS rotate `CANNON.Plane` by `-Math.PI / 2` around X to create a horizontal ground.
