# threejs-impl-physics — Methods Reference

## cannon-es API

### CANNON.World

| Method / Property | Signature | Description |
|-------------------|-----------|-------------|
| `constructor` | `new CANNON.World()` | Creates a physics world |
| `.gravity` | `Vec3` | World gravity vector; default `(0, 0, 0)` |
| `.broadphase` | `Broadphase` | Collision broadphase algorithm |
| `.solver` | `Solver` | Constraint solver; `.iterations` controls accuracy |
| `.allowSleep` | `boolean` | Enable body sleeping; default `false` |
| `.step()` | `(fixedTimeStep: number, timeSinceLastCalled?: number, maxSubSteps?: number) => void` | Advance simulation; ALWAYS use three-argument form |
| `.addBody()` | `(body: Body) => void` | Add a rigid body to the world |
| `.removeBody()` | `(body: Body) => void` | Remove a rigid body from the world |
| `.addConstraint()` | `(constraint: Constraint) => void` | Add a constraint |
| `.removeConstraint()` | `(constraint: Constraint) => void` | Remove a constraint |
| `.addContactMaterial()` | `(cm: ContactMaterial) => void` | Register a contact material pair |
| `.removeContactMaterial()` | `(cm: ContactMaterial) => void` | Remove a contact material pair |

### CANNON.Body

| Method / Property | Signature | Description |
|-------------------|-----------|-------------|
| `constructor` | `new CANNON.Body(options: { mass, position?, shape?, material?, linearDamping?, angularDamping?, type?, fixedRotation?, collisionFilterGroup?, collisionFilterMask? })` | Creates a rigid body |
| `.position` | `Vec3` | Body position in world space |
| `.quaternion` | `Quaternion` | Body orientation |
| `.velocity` | `Vec3` | Linear velocity |
| `.angularVelocity` | `Vec3` | Angular velocity |
| `.mass` | `number` | Body mass in kg; `0` = static |
| `.type` | `number` | `DYNAMIC`, `STATIC`, or `KINEMATIC` |
| `.material` | `Material \| null` | Physics material for contact calculations |
| `.linearDamping` | `number` | Linear velocity damping `[0, 1]` |
| `.angularDamping` | `number` | Angular velocity damping `[0, 1]` |
| `.fixedRotation` | `boolean` | Lock rotation; default `false` |
| `.sleepState` | `number` | `AWAKE`, `SLEEPY`, or `SLEEPING` |
| `.addShape()` | `(shape: Shape, offset?: Vec3, orientation?: Quaternion) => Body` | Add a collision shape |
| `.removeShape()` | `(shape: Shape) => Body` | Remove a collision shape |
| `.applyForce()` | `(force: Vec3, worldPoint?: Vec3) => void` | Apply force at world point |
| `.applyImpulse()` | `(impulse: Vec3, worldPoint?: Vec3) => void` | Apply instant impulse |
| `.applyLocalForce()` | `(force: Vec3, localPoint?: Vec3) => void` | Apply force in local space |
| `.applyLocalImpulse()` | `(impulse: Vec3, localPoint?: Vec3) => void` | Apply impulse in local space |
| `.applyTorque()` | `(torque: Vec3) => void` | Apply rotational torque |
| `.sleep()` | `() => void` | Force body to sleep |
| `.wakeUp()` | `() => void` | Force body awake |
| `.addEventListener()` | `(type: string, listener: Function) => void` | Listen for `'collide'`, `'sleep'`, `'wakeup'` |
| `.removeEventListener()` | `(type: string, listener: Function) => void` | Remove event listener |

### CANNON.Vec3

| Method | Signature | Description |
|--------|-----------|-------------|
| `constructor` | `new CANNON.Vec3(x?: number, y?: number, z?: number)` | 3D vector |
| `.set()` | `(x: number, y: number, z: number) => Vec3` | Set components |
| `.copy()` | `(source: Vec3) => Vec3` | Copy from another Vec3 |
| `.vadd()` | `(v: Vec3, target?: Vec3) => Vec3` | Vector addition |
| `.vsub()` | `(v: Vec3, target?: Vec3) => Vec3` | Vector subtraction |
| `.scale()` | `(scalar: number, target?: Vec3) => Vec3` | Scalar multiply |
| `.dot()` | `(v: Vec3) => number` | Dot product |
| `.cross()` | `(v: Vec3, target?: Vec3) => Vec3` | Cross product |
| `.length()` | `() => number` | Vector magnitude |
| `.normalize()` | `() => Vec3` | Normalize in place |
| `.distanceTo()` | `(v: Vec3) => number` | Distance to another Vec3 |

### CANNON Shape Constructors

| Shape | Constructor |
|-------|------------|
| `Box` | `new CANNON.Box(halfExtents: Vec3)` |
| `Sphere` | `new CANNON.Sphere(radius: number)` |
| `Cylinder` | `new CANNON.Cylinder(radiusTop: number, radiusBottom: number, height: number, numSegments: number)` |
| `Plane` | `new CANNON.Plane()` |
| `ConvexPolyhedron` | `new CANNON.ConvexPolyhedron({ vertices: Vec3[], faces: number[][] })` |
| `Trimesh` | `new CANNON.Trimesh(vertices: number[], indices: number[])` |
| `Heightfield` | `new CANNON.Heightfield(data: number[][], options: { elementSize: number })` |
| `Particle` | `new CANNON.Particle()` |

### CANNON.Material and CANNON.ContactMaterial

| Constructor | Signature |
|-------------|-----------|
| `Material` | `new CANNON.Material(name?: string)` |
| `ContactMaterial` | `new CANNON.ContactMaterial(m1: Material, m2: Material, options: { friction?: number, restitution?: number, contactEquationStiffness?: number, contactEquationRelaxation?: number })` |

### CANNON Constraint Constructors

| Constraint | Constructor |
|-----------|------------|
| `PointToPointConstraint` | `new CANNON.PointToPointConstraint(bodyA: Body, pivotA: Vec3, bodyB: Body, pivotB: Vec3, maxForce?: number)` |
| `DistanceConstraint` | `new CANNON.DistanceConstraint(bodyA: Body, bodyB: Body, distance?: number, maxForce?: number)` |
| `HingeConstraint` | `new CANNON.HingeConstraint(bodyA: Body, bodyB: Body, options: { pivotA: Vec3, axisA: Vec3, pivotB: Vec3, axisB: Vec3, maxForce?: number })` |
| `LockConstraint` | `new CANNON.LockConstraint(bodyA: Body, bodyB: Body, options?: { maxForce?: number })` |
| `ConeTwistConstraint` | `new CANNON.ConeTwistConstraint(bodyA: Body, bodyB: Body, options: { pivotA: Vec3, axisA: Vec3, pivotB: Vec3, axisB: Vec3, angle?: number, twistAngle?: number })` |
| `Spring` | `new CANNON.Spring(bodyA: Body, bodyB: Body, options: { restLength?: number, stiffness?: number, damping?: number, localAnchorA?: Vec3, localAnchorB?: Vec3 })` |

### CANNON Broadphase Types

| Type | Constructor | Complexity |
|------|------------|------------|
| `NaiveBroadphase` | `new CANNON.NaiveBroadphase()` | O(n^2) — test all pairs |
| `SAPBroadphase` | `new CANNON.SAPBroadphase(world)` | O(n log n) — sweep and prune |
| `GridBroadphase` | `new CANNON.GridBroadphase(options)` | O(n) — spatial hash grid |

---

## Rapier API

### RAPIER Module

| Method | Signature | Description |
|--------|-----------|-------------|
| `init()` | `() => Promise<void>` | Initialize WASM module; MUST await before ANY other API call |

### RAPIER.World

| Method / Property | Signature | Description |
|-------------------|-----------|-------------|
| `constructor` | `new RAPIER.World(gravity: {x, y, z})` | Creates a physics world |
| `.step()` | `(eventQueue?: EventQueue) => void` | Advance simulation one fixed step |
| `.gravity` | `{x: number, y: number, z: number}` | World gravity; mutable |
| `.createRigidBody()` | `(desc: RigidBodyDesc) => RigidBody` | Create a rigid body from descriptor |
| `.removeRigidBody()` | `(body: RigidBody) => void` | Remove and destroy a rigid body |
| `.createCollider()` | `(desc: ColliderDesc, parent?: RigidBody) => Collider` | Create a collider, optionally attached to a body |
| `.removeCollider()` | `(collider: Collider, wakeUpParent: boolean) => void` | Remove and destroy a collider |
| `.getCollider()` | `(handle: number) => Collider` | Get collider by handle |
| `.getRigidBody()` | `(handle: number) => RigidBody` | Get rigid body by handle |
| `.castRay()` | `(ray: Ray, maxToi: number, solid: boolean) => RayColliderHit \| null` | Cast a ray; returns first hit |
| `.castRayAndGetNormal()` | `(ray: Ray, maxToi: number, solid: boolean) => RayColliderHit \| null` | Cast ray with surface normal |
| `.intersectionsWithShape()` | `(position: {x,y,z}, rotation: {x,y,z,w}, shape: Shape, callback: (collider) => boolean) => void` | Query shape overlaps |
| `.intersectionsWithPoint()` | `(point: {x,y,z}, callback: (collider) => boolean) => void` | Query point containment |
| `.debugRender()` | `() => { vertices: Float32Array, colors: Float32Array }` | Get debug wireframe data |
| `.free()` | `() => void` | Destroy world and release WASM memory |

### RAPIER.RigidBodyDesc (Builder)

| Method | Signature | Description |
|--------|-----------|-------------|
| `.dynamic()` | `static () => RigidBodyDesc` | Dynamic body descriptor |
| `.fixed()` | `static () => RigidBodyDesc` | Static/fixed body descriptor |
| `.kinematicPositionBased()` | `static () => RigidBodyDesc` | Kinematic position-driven body |
| `.kinematicVelocityBased()` | `static () => RigidBodyDesc` | Kinematic velocity-driven body |
| `.setTranslation()` | `(x: number, y: number, z: number) => RigidBodyDesc` | Set initial position |
| `.setRotation()` | `(rotation: {x, y, z, w}) => RigidBodyDesc` | Set initial rotation quaternion |
| `.setLinvel()` | `(x: number, y: number, z: number) => RigidBodyDesc` | Set initial linear velocity |
| `.setAngvel()` | `(angvel: {x, y, z}) => RigidBodyDesc` | Set initial angular velocity |
| `.setLinearDamping()` | `(damping: number) => RigidBodyDesc` | Set linear damping |
| `.setAngularDamping()` | `(damping: number) => RigidBodyDesc` | Set angular damping |
| `.setCcdEnabled()` | `(enabled: boolean) => RigidBodyDesc` | Enable continuous collision detection |
| `.setCanSleep()` | `(canSleep: boolean) => RigidBodyDesc` | Allow/disallow sleeping |
| `.lockTranslations()` | `() => RigidBodyDesc` | Lock all translation axes |
| `.lockRotations()` | `() => RigidBodyDesc` | Lock all rotation axes |
| `.setGravityScale()` | `(scale: number) => RigidBodyDesc` | Per-body gravity multiplier |

### RAPIER.RigidBody

| Method | Signature | Description |
|--------|-----------|-------------|
| `.translation()` | `() => {x, y, z}` | Current position |
| `.rotation()` | `() => {x, y, z, w}` | Current rotation quaternion |
| `.linvel()` | `() => {x, y, z}` | Current linear velocity |
| `.angvel()` | `() => {x, y, z}` | Current angular velocity |
| `.setTranslation()` | `(translation: {x, y, z}, wakeUp: boolean) => void` | Set position |
| `.setRotation()` | `(rotation: {x, y, z, w}, wakeUp: boolean) => void` | Set rotation |
| `.setLinvel()` | `(vel: {x, y, z}, wakeUp: boolean) => void` | Set linear velocity |
| `.setAngvel()` | `(vel: {x, y, z}, wakeUp: boolean) => void` | Set angular velocity |
| `.applyForce()` | `(force: {x, y, z}, wakeUp: boolean) => void` | Apply force at center of mass |
| `.applyImpulse()` | `(impulse: {x, y, z}, wakeUp: boolean) => void` | Apply instant impulse |
| `.applyTorque()` | `(torque: {x, y, z}, wakeUp: boolean) => void` | Apply rotational torque |
| `.applyTorqueImpulse()` | `(impulse: {x, y, z}, wakeUp: boolean) => void` | Apply instant torque |
| `.applyForceAtPoint()` | `(force: {x,y,z}, point: {x,y,z}, wakeUp: boolean) => void` | Apply force at world point |
| `.applyImpulseAtPoint()` | `(impulse: {x,y,z}, point: {x,y,z}, wakeUp: boolean) => void` | Apply impulse at world point |
| `.isSleeping()` | `() => boolean` | Check sleep state |
| `.wakeUp()` | `() => void` | Force body awake |
| `.sleep()` | `() => void` | Force body to sleep |
| `.handle` | `number` | Unique handle for lookups |

### RAPIER.ColliderDesc (Builder)

| Method | Signature | Description |
|--------|-----------|-------------|
| `.cuboid()` | `static (hx, hy, hz) => ColliderDesc` | Box with half-extents |
| `.ball()` | `static (radius) => ColliderDesc` | Sphere |
| `.capsule()` | `static (halfHeight, radius) => ColliderDesc` | Capsule |
| `.cylinder()` | `static (halfHeight, radius) => ColliderDesc` | Cylinder |
| `.cone()` | `static (halfHeight, radius) => ColliderDesc` | Cone |
| `.convexHull()` | `static (vertices: Float32Array) => ColliderDesc \| null` | Convex hull from points |
| `.trimesh()` | `static (vertices: Float32Array, indices: Uint32Array) => ColliderDesc` | Triangle mesh |
| `.heightfield()` | `static (nrows, ncols, heights: Float32Array, scale: {x,y,z}) => ColliderDesc` | Terrain |
| `.roundCuboid()` | `static (hx, hy, hz, borderRadius) => ColliderDesc` | Rounded box |
| `.setRestitution()` | `(coeff: number) => ColliderDesc` | Set bounciness |
| `.setFriction()` | `(coeff: number) => ColliderDesc` | Set friction |
| `.setDensity()` | `(density: number) => ColliderDesc` | Set density (affects mass) |
| `.setMass()` | `(mass: number) => ColliderDesc` | Set explicit mass |
| `.setSensor()` | `(isSensor: boolean) => ColliderDesc` | Make sensor (trigger only, no physics response) |
| `.setTranslation()` | `(x, y, z) => ColliderDesc` | Offset relative to parent body |
| `.setRotation()` | `(rotation: {x,y,z,w}) => ColliderDesc` | Rotation relative to parent body |
| `.setActiveEvents()` | `(events: number) => ColliderDesc` | Enable collision events |

### RAPIER.Ray

| Constructor | Signature |
|-------------|-----------|
| `Ray` | `new RAPIER.Ray(origin: {x, y, z}, dir: {x, y, z})` |

### RAPIER.EventQueue

| Method | Signature | Description |
|--------|-----------|-------------|
| `constructor` | `new RAPIER.EventQueue(autoDrain: boolean)` | Creates event queue |
| `.drainCollisionEvents()` | `(callback: (handle1: number, handle2: number, started: boolean) => void) => void` | Process collision events |
| `.drainContactForceEvents()` | `(callback: (event: ContactForceEvent) => void) => void` | Process contact force events |
| `.free()` | `() => void` | Release WASM memory |
